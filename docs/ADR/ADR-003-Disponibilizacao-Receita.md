# ADR-003: Disponibilização da receita digital vinculada à consulta

## Status
Aceito

## Data
2026-09-12

## User story relacionada
**Feature 3 — User Story 1: Disponibilização da receita para paciente (Dia 1).**

O médico deve emitir uma receita vinculada a uma consulta e disponibilizá-la ao paciente. A consulta da prescrição pela conta do paciente e a evolução da experiência de notificação são incrementos posteriores, embora a notificação in-app básica já seja disparada pelo caso de uso atual.

## Contexto técnico

O domínio de prescrição pertence ao módulo `prescription` e integra-se aos módulos `scheduling`, `identity` e `notification`. Os principais componentes são:

- `PrescriptionController`, que expõe criação, listagem e consulta individual;
- `PrescriptionService`, que aplica autorização e regras de vínculo;
- `PrescriptionRepository`, que persiste o cabeçalho;
- entidade `Prescription` e entidade filha `PrescriptionItem`;
- `SchedulingFacade`, que resolve a consulta;
- `IdentityFacade`, que resolve o médico e dados de apresentação;
- `NotificationFacade`, que publica a notificação `PRESCRIPTION_ISSUED`.

## Drivers da decisão

1. Entregar a primeira receita ponta a ponta em um dia útil.
2. Impedir que um médico emita receita para consulta de outro médico ou paciente diferente.
3. Preservar rastreabilidade entre receita, consulta, paciente e médico.
4. Manter medicamentos separados do cabeçalho para permitir múltiplos itens.
5. Não introduzir ainda assinatura digital qualificada ou integração com farmácias.

## Decisão

### Modelo persistente

A tabela `prescriptions` armazena:

- `id` como UUID;
- `patient_id` e `doctor_id` como FKs para `users`;
- `appointment_id` como FK obrigatória para `appointments`;
- `issued_at` como instante de emissão.

A tabela `prescription_items` armazena os itens da receita:

- `prescription_id` como FK com `ON DELETE CASCADE`;
- `medication`;
- `dosage`;
- `instructions`.

O modelo permite uma receita com múltiplos medicamentos e mantém a prescrição vinculada ao atendimento que a originou.

### Fluxo de emissão

1. `POST /api/v1/prescriptions` exige o papel `MEDICO` por `@PreAuthorize`.
2. `PrescriptionService` carrega a consulta por `SchedulingFacade`.
3. O serviço compara `appointment.doctorId()` com o usuário autenticado.
4. O serviço compara `appointment.patientId()` com `request.patientId()`.
5. Os itens do request são convertidos em `PrescriptionItem`.
6. `Prescription.issue` cria o agregado e `PrescriptionRepository.save` persiste o cabeçalho e os itens.
7. O serviço publica `PRESCRIPTION_ISSUED` para o paciente com o caminho `/receitas`.
8. A resposta retorna `201 Created` com os dados da prescrição.

Se a consulta não existir, retorna `404 Not Found`. Se o médico ou paciente não corresponderem à consulta, retorna `403 Forbidden`.

### Contrato HTTP

- `POST /api/v1/prescriptions`: emissão da receita pelo médico responsável.
- `GET /api/v1/prescriptions`: lista receitas do médico autenticado ou do paciente autenticado.
- `GET /api/v1/prescriptions/{id}`: retorna uma receita quando o usuário é paciente relacionado, médico emissor ou administrador.

A criação recebe `patientId`, `appointmentId` e uma lista de itens com medicamento, dosagem e instruções. O agregado é somente criado após a validação do vínculo com a consulta.

### Notificação

A implementação atual chama `NotificationFacade` de forma síncrona após salvar a receita. A notificação é in-app, tem tipo `PRESCRIPTION_ISSUED`, inclui a consulta de origem e aponta para `/receitas`.

A decisão mantém a notificação fora da persistência do agregado. Em uma evolução futura, o efeito deve usar outbox/evento transacional, retry e idempotência para impedir que falha de entrega afete a emissão ou gere duplicidade.

## Alternativas tecnológicas consideradas

### PostgreSQL relacional em vez de MongoDB

Escolhemos PostgreSQL porque a receita possui vínculo obrigatório com `appointments`, `users` e seus itens. Foreign keys, transações e `ON DELETE CASCADE` ajudam a preservar a integridade entre o cabeçalho e os medicamentos.

MongoDB permitiria guardar a receita e seus itens em um único documento, mas reduziria a proteção relacional já disponível no restante do sistema e introduziria uma segunda tecnologia de persistência. Só deve ser considerado se o formato dos documentos clínicos se tornar muito variável ou se houver uma necessidade comprovada de escala documental.

### Itens em tabela filha em vez de JSON no cabeçalho

Escolhemos `prescription_items` em vez de um campo JSON porque cada medicamento possui estrutura própria, validação independente e ciclo de vida dependente da receita. A tabela filha também mantém o modelo explícito para consultas e auditoria.

Um JSON seria mais simples para uma primeira gravação, mas dificultaria validação, evolução do schema, consultas por item e integridade referencial. Pode ser útil apenas para metadados não estruturados que não participem das regras centrais.

### Notificação síncrona em vez de outbox desde o início

O MVP chama `NotificationFacade` na mesma transação de emissão para reduzir componentes e entregar feedback imediato. Uma outbox transacional com worker, retry e idempotência seria mais resiliente, mas adicionaria tabela, processamento assíncrono e monitoramento.

A outbox é a alternativa recomendada quando falhas de e-mail, volume de notificações ou necessidade de reprocessamento passarem a ameaçar a confiabilidade do fluxo clínico.

### Receita própria em vez de integração imediata com farmácias

A primeira entrega persiste e disponibiliza a receita dentro da plataforma. Integração com farmácias, assinatura digital qualificada ou padrões externos exigiria requisitos regulatórios, credenciamento e contratos de integração; por isso, ficam fora desta decisão e devem possuir ADRs próprios.

## Invariantes

- Somente `MEDICO` pode criar uma receita.
- O médico autenticado deve ser o médico da consulta.
- O paciente informado deve ser o paciente da consulta.
- A receita deve possuir uma consulta existente.
- O paciente só lista e consulta as próprias receitas.
- O médico só lista e consulta receitas que emitiu.
- Usuário sem relação recebe `403 Forbidden` na consulta individual.
- Conteúdo da receita não deve ser escrito em logs.

## Segurança e observabilidade

- Auditar criação e acesso a prescrições sem registrar medicamentos em texto nos logs.
- Monitorar erros `403`, `404` e falhas de notificação.
- Medir quantidade de receitas emitidas por consulta e por médico.
- Manter autenticação JWT e autorização por recurso em todas as rotas.
- Planejar assinatura digital qualificada, validação e integração externa como ADRs separados.

## Consequências

### Positivas

- O vínculo obrigatório com `appointments` reduz emissão fora do contexto clínico.
- O agregado suporta múltiplos medicamentos e histórico por paciente/médico.
- A primeira entrega é pequena e não depende de farmácias ou de uma autoridade de assinatura externa.

### Riscos e evolução

- Não há decisão de unicidade que impeça múltiplas receitas para a mesma consulta; a regra de reemissão deve ser definida antes de impor uma constraint.
- A receita do MVP não é uma receita digital qualificada para todos os contextos regulatórios.
- A notificação síncrona pode falhar depois da persistência; outbox é a evolução recomendada.

## Validação

Criar testes de integração para:

- emissão válida pelo médico da consulta;
- tentativa de emissão por paciente;
- tentativa de emissão por outro médico;
- paciente divergente da consulta;
- consulta inexistente;
- listagem isolada por paciente e médico;
- acesso individual por usuário sem relação;
- persistência de múltiplos itens;
- geração de `PRESCRIPTION_ISSUED` sem duplicidade.
