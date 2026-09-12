# ADR-002: Autorização contextual do prontuário

## Status
Aceito

## Data
2026-09-12

## User story relacionada
**Feature 2 — User Story 1: Disponibilização do prontuário para médico específico (Dia 1).**

O paciente deve disponibilizar seu histórico para um médico específico, e um médico não autorizado deve receber `403 Forbidden` ao tentar ler o prontuário compartilhado.

## Contexto técnico

O prontuário é implementado nos módulos `ehr` e `consent` do monólito. O fluxo envolve:

- `ConsentController` e `ConsentService`, responsáveis por conceder, listar e revogar consentimentos;
- `EhrController` e `EhrService`, responsáveis por escrever e ler evoluções;
- `ConsentRepository` e `ClinicalNoteRepository`, para persistência;
- `ClinicalContentEncryptor`, que cifra e decifra o conteúdo clínico;
- `EhrAccessAuditRepository`, que registra eventos de leitura e escrita;
- `IdentityFacade`, que resolve paciente, médico e papel;
- `SchedulingFacade`, que verifica a relação entre paciente, médico e consulta.

No MVP, os metadados e o ciphertext das evoluções ficam no PostgreSQL. O conteúdo da evolução é armazenado em `clinical_notes.ciphertext`, acompanhado por `iv`; storage clínico separado é uma evolução futura.

## Drivers da decisão

1. Negar acesso por padrão quando não houver autorização válida.
2. Permitir evolução incremental para consentimento por médico e por consulta.
3. Reavaliar a autorização em toda leitura, sem cache de permissão clínica.
4. Registrar acessos permitidos e negados para investigação.
5. Manter a regra clínica encapsulada nas APIs dos módulos `Consent` e `EHR`.

## Decisão

### Modelo de consentimento

A autorização usa a tabela `consents`:

| Campo | Função |
| --- | --- |
| `patient_id` | Titular que concede a autorização |
| `doctor_id` | Médico autorizado |
| `scope` | `DOCTOR` ou `APPOINTMENT` |
| `appointment_id` | Consulta coberta quando o escopo é `APPOINTMENT` |
| `version` | Versão do consentimento para o par paciente/médico |
| `granted_at` | Momento da concessão |
| `expires_at` | Expiração opcional |
| `revoked_at` | Momento da revogação, quando aplicável |

O paciente concede por `POST /api/v1/consents`, consulta por `GET /api/v1/consents` e revoga por `POST /api/v1/consents/{id}/revoke`. O backend valida que somente o paciente titular pode conceder ou revogar.

### Regra de autorização de leitura

Em `GET /api/v1/patients/{patientId}/ehr`, `EhrService` aplica as seguintes decisões:

1. O paciente titular pode consultar o próprio prontuário.
2. Um administrador pode consultar conforme a regra administrativa vigente.
3. Um médico precisa ser um usuário autenticado com papel `MEDICO`.
4. Para histórico compartilhado, o médico precisa ter consentimento válido que cubra seu ID e, quando informado, o `appointmentId`.
5. O médico autor da própria nota pode visualizá-la quando a regra de relação clínica permitir.
6. Se a regra não for satisfeita, `EhrAuditService` registra `READ_DENIED` e a API retorna `403 Forbidden`.

O método `Consent.covers` implementa a cobertura: `DOCTOR` cobre o médico para qualquer consulta; `APPOINTMENT` cobre somente a consulta cujo UUID coincide. `revoked_at` e `expires_at` invalidam a autorização.

### Escrita de evolução

`POST /api/v1/patients/{patientId}/ehr` exige `MEDICO` e valida que:

- a consulta existe;
- o médico autenticado é o médico da consulta;
- o paciente da URL é o paciente da consulta;
- o conteúdo é cifrado antes de `ClinicalNoteRepository.save`.

Uma tentativa de escrita inválida gera auditoria `WRITE_DENIED` antes do `403`.

### Proteção criptográfica

O conteúdo clínico não é retornado diretamente do banco. `ClinicalContentEncryptor` cifra o texto e persiste ciphertext e IV; na leitura, o serviço decifra em memória para montar a resposta. A chave deve vir de configuração segura (`EHR_ENCRYPTION_KEY`) e não pode ser versionada.

## Alternativas tecnológicas consideradas

### PostgreSQL cifrado em vez de MongoDB

Mantemos os metadados, ciphertext, IV, consentimentos e auditoria no PostgreSQL porque o MVP já usa esse banco para identidade, agenda e prescrição. Isso simplifica transações, migrations, backups e consultas por paciente/médico, sem duplicar infraestrutura.

MongoDB poderia representar documentos clínicos com mais flexibilidade, especialmente para evoluções com campos variáveis. Porém, não elimina a necessidade de autorização, auditoria e gerenciamento de chaves; também acrescentaria uma segunda tecnologia de persistência ao MVP. A migração para um storage documental só deve ocorrer se o formato clínico ou o volume justificar o custo operacional.

### Autorização no módulo `Consent` em vez de regras espalhadas nos controllers

Escolhemos `ConsentService`/`ConsentFacade` como ponto de decisão para validade, escopo, expiração e revogação. Colocar verificações diretamente em cada controller seria mais rápido inicialmente, mas criaria duplicação e risco de um endpoint esquecer uma regra clínica.

Um motor externo de políticas, como OPA, também seria possível, mas é desproporcional ao tamanho atual do sistema. Deve ser reavaliado apenas quando houver múltiplos serviços independentes ou políticas compartilhadas fora do monólito.

### Cifra no campo em vez de criptografia apenas no volume

Escolhemos cifrar o conteúdo antes da persistência com AES-GCM, mantendo ciphertext e IV no registro. Criptografia de volume ou de disco continua necessária, mas sozinha não protege o dado contra leitura indevida por uma credencial com acesso ao banco. A evolução para KMS/Secrets Manager deve proteger a chave e permitir rotação controlada.

## Invariantes

- Nenhum médico acessa histórico compartilhado sem consentimento válido ou regra explícita de autoria/relação vigente.
- Consentimento revogado ou expirado não pode autorizar leitura.
- Consentimento `APPOINTMENT` nunca cobre outra consulta.
- Somente o paciente titular pode conceder ou revogar.
- Toda leitura e tentativa negada deve ter evento de auditoria conforme a cobertura atual do MVP.
- Conteúdo clínico nunca deve ser registrado em log.

## Auditoria e observabilidade

A tabela `ehr_access_audit` registra `actor_user_id`, `patient_id`, `appointment_id`, `action` e `accessed_at`. As ações relevantes são `READ`, `READ_DENIED`, `WRITE` e `WRITE_DENIED`.

A auditoria atual ainda não possui garantia física de append-only nem job de retenção. Essas capacidades ficam como evolução e devem ser protegidas por permissão de banco, retenção definida e alertas para volume anormal de acessos negados.

## Consequências

### Positivas

- O modelo já suporta o crescimento para consentimento por médico, por consulta, expiração e revogação.
- A autorização fica em um módulo dedicado e é consumida pelo EHR por API pública.
- A cifra é aplicada antes da persistência, reduzindo exposição acidental do conteúdo clínico.

### Riscos e evolução

- A regra atual também considera relação de atendimento e autoria da nota; essa exceção deve ser formalmente revisada antes de ampliar o compartilhamento.
- O PostgreSQL ainda concentra ciphertext e metadados; storage dedicado, KMS e rotação de chaves são evoluções futuras.
- Auditoria completa, append-only e retenção automática ainda precisam ser implementadas.

## Validação

Criar testes de integração para:

- paciente lendo o próprio prontuário;
- médico com consentimento `DOCTOR`;
- médico com consentimento `APPOINTMENT` correto;
- médico usando outra consulta;
- consentimento expirado;
- consentimento revogado;
- médico sem consentimento;
- escrita por médico que não pertence à consulta;
- cifra e decifra do conteúdo sem vazamento em logs;
- criação dos eventos de auditoria permitidos e negados.

