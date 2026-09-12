# ADR 002: Vinculação de Acesso do Prontuário a um Médico Específico

## Status
Aceito

## Contexto
A **User Story 1 da Feature 2** estabelece que o paciente precisa disponibilizar seu prontuário para um **médico específico**. O critério de aceitação principal é garantir que apenas esse médico selecionado pelo paciente consiga acessar o histórico, enquanto outros médicos não autorizados recebam acesso negado. Essa é a primeira etapa (Dia 1) focada apenas no compartilhamento simples e direto entre o paciente e um único profissional de saúde.

## Decisão
O acesso ao prontuário será controlado pelo módulo de consentimento, e não por uma tabela simplificada de vínculo:

1. **Modelo de autorização:** Usaremos a tabela `consents`, com `patient_id`, `doctor_id`, `scope`, `appointment_id`, `version`, `granted_at`, `expires_at` e `revoked_at`. Para a User Story 1, o consentimento será direcionado a um médico específico.
2. **Regra de leitura:** No endpoint `GET /api/v1/patients/{patientId}/ehr`, o backend validará o usuário autenticado, seu papel de médico, o médico autorizado e a validade do consentimento. O escopo `DOCTOR` cobre o histórico desse médico; o escopo de consulta cobre somente o `appointmentId` correspondente.
3. **Bloqueio:** Para leitura de histórico compartilhado, se o médico não tiver consentimento válido, se o consentimento estiver expirado ou revogado, ou se o escopo não cobrir a consulta solicitada, a API retornará `403 Forbidden` e registrará a tentativa negada na auditoria. No comportamento atual, um médico que possui relação de atendimento com o paciente pode consultar as próprias notas, mesmo sem consentimento geral; essa exceção está documentada como dívida de decisão e deve ser revisada antes de ampliar o compartilhamento.
4. **Proteção do conteúdo:** As evoluções serão armazenadas cifradas com AES-GCM. O paciente titular poderá consultar seu próprio prontuário, e o acesso do médico ficará condicionado às regras acima.

## Consequências
**Positivas:**
- A autorização já suporta a evolução prevista para consentimento por médico, por consulta, expiração e revogação.
- O bloqueio é reavaliado a cada leitura, evitando que um consentimento revogado permaneça válido em cache.
- A decisão mantém a autorização clínica dentro dos módulos `Consent` e `EHR`, respeitando a modularidade do backend.

**Negativas / Riscos:**
- O modelo possui mais campos que uma relação simples, mas evita uma migração estrutural quando outros médicos e escopos forem adicionados nos próximos dias.
- Todo acesso ao prontuário precisa informar o paciente e pode informar a consulta para que o escopo seja avaliado corretamente.
- A regra atual combina vínculo de atendimento, autoria da nota e consentimento; essa combinação deve ser coberta por testes de acesso permitido, médico diferente, consentimento expirado, consentimento revogado e leitura das próprias notas.
