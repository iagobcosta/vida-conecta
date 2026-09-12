# ADR 001: Implementação de Agendamento Simples

## Status
Aceito

## Contexto
A **User Story 1 da Feature 1 (Agendamento simples)** exige que um paciente autenticado possa visualizar uma lista de médicos, consultar os horários livres previamente cadastrados e realizar o agendamento de uma consulta.
O critério de aceitação principal é garantir que a consulta apareça na agenda com status inicial e evitar que **o mesmo horário seja reservado duas vezes** (prevenção de *double booking*). Esta etapa é parte de um MVP que deve ser entregue no Dia 1 de desenvolvimento.

## Decisão
Para o MVP, o agendamento será persistido no PostgreSQL e validado na camada de serviço dentro de uma transação:

1. **Modelagem de dados:** A tabela `appointments` terá `patient_id`, `doctor_id`, `scheduled_at`, `duration_minutes` e `status`. O horário será armazenado como `TIMESTAMPTZ`, preservando a referência absoluta de tempo.
2. **Disponibilidade:** O serviço validará se o médico possui disponibilidade cadastrada e se não existe consulta ativa que se sobreponha ao intervalo solicitado. Consultas canceladas não bloqueiam o horário.
3. **Concorrência no MVP:** A validação de sobreposição ocorrerá imediatamente antes da persistência, e uma tentativa conflitante detectada será convertida pela API em `409 Conflict` com a mensagem de horário indisponível. O índice `idx_appointments_doctor_time` apoiará a consulta por médico e horário. O banco ainda não possui uma restrição de exclusão para blindar duas transações concorrentes.
4. **API:** A primeira entrega usará as rotas reais `GET /api/v1/doctors`, `GET /api/v1/doctors/{id}/availability`, `GET /api/v1/doctors/{id}/slots` e `POST /api/v1/appointments`.

## Consequências
**Positivas:**
- O modelo é pequeno e atende ao fluxo utilizável do Dia 1 sem criar uma entidade de reserva separada.
- O banco indexa as consultas por médico e horário, e a regra de conflito fica centralizada no serviço de agendamento.
- O uso de `TIMESTAMPTZ` evita ambiguidades de fuso no armazenamento.

**Negativas / Riscos:**
- Duas requisições concorrentes podem passar pela consulta de sobreposição antes de qualquer uma persistir. Portanto, o MVP reduz o risco, mas ainda não oferece garantia absoluta contra *double booking* em concorrência real.
- Antes de escalar horizontalmente, deve ser adicionada uma proteção transacional no banco, preferencialmente uma restrição de exclusão por intervalo (`tstzrange`) com suporte a `btree_gist`, ou um mecanismo equivalente de bloqueio.
- A API precisa manter o tratamento de `409 Conflict` para que o usuário receba uma mensagem amigável, como “Horário indisponível”.
- A interface apresenta os horários em `America/Sao_Paulo`, mas o banco mantém o instante absoluto em UTC por meio de `TIMESTAMPTZ`.
