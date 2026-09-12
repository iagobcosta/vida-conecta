# ADR 003: Disponibilização da Receita Digital ao Paciente

## Status
Aceito

## Contexto
A **User Story 1 da Feature 3 (Disponibilização da receita para paciente)** exige que o médico consiga registrar uma prescrição vinculada a uma consulta e disponibilizá-la ao paciente após o atendimento.

A primeira entrega precisa ser utilizável em um dia útil, preservar a relação entre receita, consulta, médico e paciente e impedir que outros usuários consultem dados clínicos que não lhes pertencem. A consulta da receita pela conta do paciente é a User Story 2. A notificação está tecnicamente disparada no mesmo caso de uso de criação no código atual, embora seja tratada como User Story 3 para fins de evolução da experiência.

## Decisão
A receita será persistida no PostgreSQL como um recurso próprio, sempre vinculado a uma consulta existente:

1. **Modelo de dados:** A tabela `prescriptions` armazenará `patient_id`, `doctor_id`, `appointment_id` e `issued_at`. Os medicamentos serão armazenados em `prescription_items`, relacionados à receita por `prescription_id`, com `medication`, `dosage` e `instructions`.
2. **Emissão:** A API aceitará a criação somente de usuários com papel `MEDICO`. Antes de persistir, o serviço validará que a consulta existe, pertence ao médico autenticado e corresponde ao paciente informado.
3. **Endpoint de criação:** A primeira entrega usará `POST /api/v1/prescriptions`, retornando `201 Created` com a receita criada e seus itens.
4. **Visibilidade:** A receita ficará disponível ao paciente da consulta e ao médico responsável. Usuários fora dessa relação receberão `403 Forbidden`; administradores poderão consultar o recurso conforme as regras administrativas da aplicação.
5. **Consulta do recurso:** A leitura individual será feita por `GET /api/v1/prescriptions/{id}`. A listagem por usuário ficará disponível em `GET /api/v1/prescriptions` e será ordenada pela data de emissão.
6. **Evolução incremental:** A consulta pela conta e a melhoria da experiência de notificação serão tratadas nas User Stories 2 e 3, sem alterar o vínculo obrigatório com a consulta. No estado atual, a criação já chama o módulo de notificações; a separação futura deve tornar esse efeito assíncrono e resiliente.

## Consequências

**Positivas:**

- A receita tem rastreabilidade por consulta, paciente, médico e data de emissão.
- O médico não consegue emitir uma receita para uma consulta de outro profissional ou para um paciente diferente.
- A estrutura separa o cabeçalho da receita dos seus medicamentos e permite consultar o histórico do paciente.
- A primeira entrega é pequena, ponta a ponta e não depende de integração externa com farmácias ou assinatura digital qualificada.

**Negativas / Riscos:**

- A receita persistida no MVP ainda não representa uma assinatura digital qualificada nem garante aceitação automática em farmácias.
- É necessário validar a possibilidade de emitir mais de uma receita para a mesma consulta e definir esse comportamento conforme a regra de negócio.
- Os dados da prescrição são sensíveis e devem permanecer protegidos por autenticação, autorização e criptografia do ambiente de persistência.
- A implementação de notificações deve permanecer desacoplada para que uma falha de comunicação não desfaça a criação da receita.
