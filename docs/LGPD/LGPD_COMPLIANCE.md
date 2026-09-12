# Manual de Conformidade LGPD - Vida Conecta

Este documento detalha o tratamento de dados pessoais (identificáveis e sensíveis) e os procedimentos operacionais de segurança e retenção previstos para o Vida Conecta. Os itens marcados como evolução dependem de infraestrutura ou jobs ainda não presentes no MVP.

## 1. Mapeamento de Dados

O sistema lida com dois níveis de dados pessoais de pacientes:

### Dados Cadastrais (PII)
- **O que são:** Nome, CPF, Data de Nascimento, E-mail, Telefone.
- **Onde ficam:** Tabelas `users` e `patient_profiles`.
- **Sensibilidade:** Alta (Identificação Direta).
- **Proteção Aplicada:** Transmissão via HTTPS. Logs mascarados (`***.***.***-45`). Em caso de exclusão (Direito ao Esquecimento), sofrem anonimização irreversível.

### Dados Clínicos Sensíveis (Prontuários e Receitas)
- **O que são:** Evolução médica, sintomas, histórico, diagnósticos, receitas emitidas.
- **Onde ficam no MVP atual:** PostgreSQL, nas tabelas `clinical_notes`, `prescriptions` e `prescription_items`; o conteúdo das evoluções fica cifrado na coluna `clinical_notes.ciphertext`.
- **Arquitetura desejada:** Storage clínico separado e protegido, conforme o README do produto; essa migração ainda não está implementada.
- **Sensibilidade:** Crítica (Dados Médicos Art. 5º LGPD).
- **Proteção Aplicada:** Criptografia em repouso AES-256-GCM. Chave gerenciada fora do banco. Acesso restrito via RBAC e Consentimento Versionado explícito.

## 2. Fluxo de Processamento Documentado

O processamento clínico obedece a um fluxo de **Zero-Trust (Confiança Zero)** entre médicos e pacientes:

1. **Agendamento:** O paciente e o médico formam uma relação primária.
2. **Consentimento (Opt-in):** Para que um médico possa ler o histórico de prontuário eletrônico de um paciente (EHR), o paciente deve explicitamente criar um registro de Consentimento, informando um escopo (Para sempre ou para uma consulta específica) e uma data de expiração.
3. **Auditoria Contínua:** O MVP registra acessos e algumas tentativas negadas na tabela `ehr_access_audit`. A garantia física de append-only, a cobertura de todas as tentativas e a visão em tempo real para o paciente ainda precisam ser implementadas.
4. **Revogação (Opt-out):** Ao revogar o consentimento, o acesso cai de imediato. Não há cache residual.

## 3. Política de Retenção e Descarte

Conforme regulamentação do Conselho Federal de Medicina (CFM) combinada com a LGPD:

- **Auditoria de Acessos:** Meta de retenção de **5 anos**, para fins de compliance e investigação; job de retenção ainda não está implementado no MVP.
- **Prontuários Clínicos:** Meta de retenção de **20 anos** a partir do último registro, conforme política do produto; rotina automática ainda não está implementada.
- **Exclusão de Conta:** Quando um usuário invoca o direito ao esquecimento, os PII (Dados Cadastrais) são anonimizados. A política de preservar prontuários anonimizados por até 20 anos precisa ser implementada e validada juridicamente; o MVP ainda não possui os jobs de anonimização, retenção e hard-delete descritos nesta seção.

## 4. Procedimento de Backup e Restore

- **Rotina (Backups):** A base PostgreSQL (RDS ou similar em produção) deve realizar snapshots diários (incrementais) e 1 snapshot full semanal.
- **Retenção de Backups:** Backups devem ser guardados em repositório criptografado apartado por 30 dias (Cold Storage).
- **Procedimento de Restore (Disaster Recovery):** 
  1. Em caso de perda, a chave de criptografia de aplicação (`EHR_ENCRYPTION_KEY`) deve ser provisionada no ambiente novo a partir do cofre (AWS Secrets Manager / Vault).
  2. Restaurar último snapshot no banco de dados.
  3. Realizar testes de decriptação simulando um request à API restrito.
  4. Redirecionar tráfego.

## 5. Plano de Resposta a Incidentes de Segurança

Na eventualidade de um vazamento de dados, uso não autorizado ou intrusão:

1. **Contenção:** Revogar imediatamente todos os tokens JWT trocando o segredo (`JWT_SECRET`) nas variáveis de ambiente. Isso forçará re-autenticação imediata de todos os atores.
2. **Avaliação:** Investigar os logs em formato JSON e o banco de `ehr_access_audit` para identificar o vetor.
3. **Notificação (Prazo: 72 horas):** Enviar notificação aos titulares dos dados impactados (via SES/E-mail) informando: O que vazou, quais os riscos e as medidas tomadas pela clínica.
4. **Reporte:** Reportar a violação à ANPD (Autoridade Nacional de Proteção de Dados).
5. **Mitigação:** Corrigir a vulnerabilidade apontada, alterar senhas de infraestrutura e rodar *Secret Scans* no código.
