# ADR-001: Agendamento simples com prevenção de sobreposição

## Status
Aceito

## Data
2026-09-12

## User story relacionada
**Feature 1 — User Story 1: Agendamento simples (Dia 1).**

Um paciente autenticado deve selecionar um médico, consultar horários disponíveis e criar uma consulta utilizável. O sistema deve rejeitar uma reserva que conflite com outra consulta ativa do mesmo médico.

## Contexto técnico

O domínio de agendamento pertence ao módulo `scheduling` do monólito Spring Boot. O fluxo envolve:

- `AppointmentController`, que expõe a API REST;
- `AvailabilityController` e `AvailabilityService`, que validam a disponibilidade cadastrada pelo médico;
- `AppointmentService`, que aplica as regras de negócio;
- `AppointmentRepository`, que consulta e persiste `appointments` via Spring Data JPA;
- PostgreSQL, que armazena os horários como `TIMESTAMPTZ`.

A consulta possui duração variável entre 15 e 120 minutos, com padrão de 30 minutos. Portanto, a regra não pode comparar apenas igualdade de horário: deve verificar sobreposição de intervalos.

## Drivers da decisão

1. Entregar uma primeira versão ponta a ponta em um dia útil.
2. Manter a consistência do domínio dentro do módulo `scheduling`.
3. Usar o PostgreSQL como fonte de verdade da agenda.
4. Retornar um erro de negócio compreensível quando o horário estiver indisponível.
5. Preservar instantes absolutos e apresentar o horário no fuso da clínica.

## Decisão

### Modelo persistente

A entidade `Appointment` será persistida na tabela `appointments` com os campos:

| Campo | Tipo lógico | Regra |
| --- | --- | --- |
| `id` | UUID | Identificador da consulta |
| `patient_id` | UUID | FK para `users`; paciente que solicitou |
| `doctor_id` | UUID | FK para `users`; médico selecionado |
| `scheduled_at` | `TIMESTAMPTZ` | Início absoluto da consulta |
| `duration_minutes` | inteiro | Entre 15 e 120 minutos |
| `status` | enum textual | `SCHEDULED`, `CONFIRMED`, `IN_PROGRESS`, `COMPLETED` ou `CANCELLED` |
| `created_at` / `updated_at` | `TIMESTAMPTZ` | Auditoria técnica da entidade |

A disponibilidade semanal do médico fica em `doctor_availabilities`; ela não substitui o registro da consulta.

### Fluxo transacional

1. `POST /api/v1/appointments` exige o papel `PACIENTE`.
2. `AppointmentService` verifica se o médico está ativo e se o horário está no futuro.
3. `AvailabilityService.assertBookable` valida a disponibilidade e a duração solicitada.
4. `AppointmentRepository.findDoctorAppointmentsInWindow` busca consultas não canceladas em uma janela de segurança.
5. `Appointment.overlaps` compara o intervalo solicitado com `scheduled_at` e `endsAt()` das consultas existentes.
6. Em caso de conflito, o serviço lança `ConflictException`, mapeada para `409 Conflict`.
7. Sem conflito, `Appointment.schedule` cria a entidade com status `SCHEDULED`, e a transação persiste o registro.
8. Após a criação, o módulo de notificações publica o evento in-app para o médico.

### Contrato HTTP

- `GET /api/v1/doctors`: lista médicos ativos.
- `GET /api/v1/doctors/{id}/availability`: lista disponibilidades do médico.
- `GET /api/v1/doctors/{id}/slots`: calcula horários livres.
- `POST /api/v1/appointments`: cria a consulta e responde `201 Created`.
- `GET /api/v1/appointments`: lista consultas do usuário autenticado.
- `GET /api/v1/appointments/{id}`: retorna a consulta quando o usuário é participante.

A criação recebe `doctorId`, `scheduledAt` em formato ISO-8601 e, opcionalmente, `durationMinutes`.

### Concorrência aceita no MVP

A proteção atual é uma verificação de sobreposição dentro de uma transação de serviço, apoiada pelo índice `idx_appointments_doctor_time (doctor_id, scheduled_at)`. Não há atualmente uma `UNIQUE CONSTRAINT` nem uma `EXCLUDE CONSTRAINT` por intervalo.

Essa decisão é suficiente para o primeiro fluxo, mas não oferece garantia absoluta quando duas transações concorrentes passam pela leitura antes da persistência. Antes de escalar horizontalmente, o banco deverá adotar uma proteção de intervalo, por exemplo `tstzrange` com `EXCLUDE USING gist` e `btree_gist`, considerando apenas estados ativos.

### Tempo e fuso

O backend usa `Instant` e o PostgreSQL usa `TIMESTAMPTZ`. O instante é persistido de forma absoluta; a apresentação ao usuário usa `America/Sao_Paulo`. O serviço não deve comparar horários por strings locais.

## Alternativas tecnológicas consideradas

### PostgreSQL em vez de MongoDB

Escolhemos PostgreSQL porque o agendamento possui relacionamentos fortes com pacientes, médicos, disponibilidades e status da consulta. O modelo também depende de transações, índices por médico/horário, integridade referencial e futura proteção contra sobreposição por intervalo (`tstzrange`/`EXCLUDE`).

MongoDB seria adequado para um modelo mais flexível ou orientado a documentos, mas exigiria controlar mais regras de consistência na aplicação e não oferece a mesma aderência às constraints relacionais necessárias para a agenda. Poderá ser avaliado para dados documentais específicos, mas não é a escolha atual para o núcleo transacional.

### Consulta de sobreposição em vez de reserva de slot

Escolhemos calcular a sobreposição usando `scheduled_at`, `duration_minutes` e `Appointment.overlaps`, pois a primeira entrega aceita durações variáveis e evita criar uma entidade de slot reservado.

Uma tabela de slots pré-gerados com estado `AVAILABLE`/`BOOKED` facilitaria o bloqueio atômico, mas aumentaria o custo de geração, manutenção e alteração de horários. Pode ser considerada se a agenda passar a exigir recorrência complexa, hold temporário ou fila de espera.

### Validação na aplicação em vez de lock pessimista

O MVP usa consulta de conflitos na camada de serviço para manter baixa a complexidade. Lock pessimista (`SELECT ... FOR UPDATE`) ou exclusão por intervalo no PostgreSQL oferece garantia maior sob concorrência, mas exige uma migration e testes específicos. Essa alternativa deve ser adotada antes de escala horizontal ou alto volume de reservas.

## Invariantes

- Um paciente não pode criar consulta para si mesmo como médico.
- O médico precisa estar ativo.
- O horário precisa estar no futuro e dentro da disponibilidade.
- Consultas `CANCELLED` não bloqueiam o intervalo.
- Consultas ativas não podem se sobrepor para o mesmo médico.
- Somente o médico participante pode confirmar ou concluir.
- O motivo do cancelamento pelo médico deve ter pelo menos 10 caracteres.

## Observabilidade e falhas

- Registrar métricas de criação, conflito, cancelamento, confirmação e conclusão.
- Monitorar respostas `409` para detectar excesso de disputa por horários.
- Tratar conflito de persistência como erro de negócio, sem expor exceção SQL ao cliente.
- Garantir que a notificação não altere a decisão já persistida da consulta.

## Consequências

### Positivas

- A regra de sobreposição fica centralizada em `AppointmentService` e `Appointment`.
- O modelo é pequeno e entrega o fluxo do Dia 1 sem introduzir uma entidade de reserva temporária.
- O índice atende às consultas por médico e intervalo no MVP.

### Riscos e evolução

- A ausência de uma restrição de intervalo deixa uma janela de *double booking* sob concorrência real.
- A próxima evolução deve adicionar teste concorrente e proteção transacional no PostgreSQL.
- A API precisará manter o contrato `409 Conflict` após a mudança de mecanismo.

## Validação

A decisão deve ser coberta por testes de integração para:

- criação válida;
- médico inexistente ou desativado;
- horário fora da disponibilidade;
- intervalo sobreposto;
- consulta cancelada liberando o horário;
- tentativa concorrente de criação;
- conversão do conflito para `409`.
