### Projeto: `Vida Conecta`
### Equipe: `Iago, Pedro, Edval, Maria`

---

### 🥇 Feature 1 — `Agendamento de consultas`

- **Problema real que ela resolve:** Pacientes têm dificuldade para encontrar um médico, identificar horários disponíveis e marcar uma consulta sem depender de trocas manuais de mensagens.
- **Critério(s) de prioridade que mais pesaram:** Valor direto para paciente e médico; frequência de uso; redução de conflitos de agenda; possibilidade de entregar uma jornada funcional e incremental em três dias úteis.
- **Em uma frase o que seria a aplicação utópica** (a versão completa, dos sonhos): O paciente encontra o profissional ideal, consulta horários em tempo real, agenda ou reagenda com facilidade e recebe lembretes e atualizações automáticas.
- **MVP comercializável em três entregas incrementais:** A Feature 1 será desenvolvida em três user stories, uma por dia útil. A primeira já entrega um agendamento simples utilizável; as seguintes refinam e finalizam o fluxo sem quebrar o que já foi entregue.

#### User Story 1 — `Agendamento simples` — Dia 1

**Como paciente, quero escolher um médico e um horário disponível para solicitar uma consulta, para conseguir realizar meu primeiro agendamento sem depender de contato manual.**

- **Entrega:** Paciente autenticado visualiza a lista de médicos, seleciona um médico, consulta seus horários previamente cadastrados e cria uma consulta em um horário livre.
- **Critério de aceitação:** Ao concluir a operação, a consulta aparece na agenda do paciente com médico, data, horário e status inicial; o mesmo horário não pode ser reservado duas vezes.
- **Valor entregue:** Primeira versão utilizável do produto, permitindo que o paciente marque uma consulta ponta a ponta.

#### User Story 2 — `Refinamento do agendamento` — Dia 2

**Como paciente, quero pesquisar e filtrar médicos e visualizar melhor os horários disponíveis, para encontrar uma opção adequada com menos esforço.**

- **Entrega:** Paciente pesquisa médico por nome, especialidade ou CRM, visualiza os horários livres dos próximos dias e recebe confirmação do agendamento criado.
- **Critério de aceitação:** A busca retorna apenas médicos disponíveis; os horários ocupados não podem ser selecionados; após o agendamento, paciente e médico conseguem visualizar os dados atualizados.
- **Incremento:** Melhora a descoberta e a escolha sem alterar o fluxo de agendamento simples entregue no Dia 1.

#### User Story 3 — `Finalização do agendamento` — Dia 3

**Como paciente ou médico, quero confirmar, cancelar ou reagendar uma consulta, para manter a agenda atualizada quando houver mudanças.**

- **Entrega:** Médico confirma a consulta; paciente ou médico visualiza o compromisso, cancela conforme sua permissão e recebe uma notificação com o novo status; o paciente pode iniciar um reagendamento.
- **Critério de aceitação:** O cancelamento libera o horário; o médico informa o motivo do cancelamento; a consulta mantém seu histórico de status e nenhuma alteração gera conflito com outro agendamento.
- **Incremento:** Completa o ciclo de vida do agendamento e aproveita as consultas e horários já criados nas duas entregas anteriores.

---

### 🥈 Feature 2 — `Prontuário eletrônico com consentimento LGPD`

- **Problema real que ela resolve:** Informações clínicas ficam dispersas ou são compartilhadas sem transparência, dificultando a continuidade do cuidado e aumentando o risco de exposição de dados sensíveis.
- **Critério(s) de prioridade que mais pesaram:** Segurança e confiança; obrigação legal relacionada à LGPD; continuidade do tratamento; possibilidade de começar com um compartilhamento controlado e evoluir para regras mais completas.
- **O "elefante" dela** (a versão completa, dos sonhos): Um prontuário longitudinal, criptografado, interoperável e auditável, com compartilhamento granular entre profissionais, consentimento versionado, revogação imediata, portabilidade e anonimização automatizada.
- **MVP comercializável em três entregas incrementais:** A Feature 2 será desenvolvida em três user stories, uma por dia útil. A primeira já permite utilizar o prontuário com um médico específico; as seguintes ampliam o compartilhamento e os controles de proteção de dados.

#### User Story 1 — `Disponibilização do prontuário para médico específico` — Dia 1

**Como paciente, quero disponibilizar meu prontuário para um médico específico, para que ele possa consultar meu histórico durante o atendimento autorizado.**

- **Entrega:** Paciente autenticado visualiza o próprio prontuário e concede acesso a um médico específico; o médico autorizado consegue consultar o histórico clínico relacionado ao paciente.
- **Critério de aceitação:** O prontuário só fica acessível ao médico selecionado; um médico não autorizado recebe acesso negado.
- **Valor entregue:** Primeira versão utilizável, permitindo o compartilhamento controlado do histórico clínico entre paciente e médico.

#### User Story 2 — `Disponibilização com consentimento para outros médicos` — Dia 2

**Como paciente, quero conceder consentimento para outros médicos, por profissional ou por consulta, para compartilhar meu histórico conforme a necessidade do tratamento.**

- **Entrega:** Paciente cria consentimentos adicionais para outros médicos, define se o acesso vale para sempre ou para uma consulta específica e visualiza os consentimentos ativos.
- **Critério de aceitação:** Cada médico só consulta o prontuário quando possui um consentimento válido; consentimentos de médicos diferentes são independentes; a consulta específica limita o acesso ao atendimento correspondente.
- **Incremento:** Amplia o compartilhamento iniciado no Dia 1 sem remover a autorização já concedida ao primeiro médico.

#### User Story 3 — `Aplicação da LGPD no prontuário` — Dia 3

**Como paciente, quero controlar, acompanhar e revogar o acesso ao meu prontuário, para ter transparência e segurança sobre o uso dos meus dados de saúde.**

- **Entrega:** Paciente revoga consentimentos, consulta o histórico de acessos e o sistema mantém o prontuário cifrado e registra as tentativas de acesso realizadas pelos médicos.
- **Critério de aceitação:** A revogação bloqueia imediatamente novos acessos; cada acesso registra médico, data, consulta e resultado; o prontuário não é exposto sem autenticação e consentimento válido.
- **Incremento:** Completa os controles de privacidade sobre os compartilhamentos criados nos Dias 1 e 2.

---

### 🥉 Feature 3 — `Prescrição digital vinculada à consulta`

- **Problema real que ela resolve:** Após a consulta, o paciente pode perder a receita ou depender de documentos físicos, enquanto o médico precisa registrar e disponibilizar a prescrição separadamente.
- **Critério(s) de prioridade que mais pesaram:** Conclusão ponta a ponta da jornada clínica; benefício imediato para paciente e médico; redução de trabalho manual; possibilidade de entregar uma prescrição utilizável e evoluir a experiência em três dias úteis.
- **O "elefante" dela** (a versão completa, dos sonhos): O médico emite uma prescrição digital segura e assinada, com histórico completo, renovação controlada, validação, integração com farmácias e acesso simples pelo paciente em qualquer dispositivo.
- **MVP comercializável em três entregas incrementais:** A Feature 3 será desenvolvida em três user stories, uma por dia útil. A primeira já permite emitir e disponibilizar uma receita; as seguintes acrescentam o acesso pela conta do paciente e a comunicação da nova prescrição.

#### User Story 1 — `Disponibilização da receita para paciente` — Dia 1

**Como médico, quero criar e disponibilizar uma receita vinculada à consulta, para que o paciente tenha acesso ao tratamento prescrito após o atendimento.**

- **Entrega:** Médico autenticado seleciona uma consulta válida, informa medicamentos, dosagens e orientações e salva a prescrição associada ao paciente.
- **Critério de aceitação:** A receita é criada com médico, paciente, consulta e data de emissão; somente o paciente relacionado e o médico responsável podem acessar os dados da prescrição.
- **Valor entregue:** Primeira versão utilizável, permitindo registrar e disponibilizar uma receita digital com segurança básica.

#### User Story 2 — `Paciente consulta a prescrição na própria conta` — Dia 2

**Como paciente, quero consultar minhas receitas na própria conta, para acessar o tratamento prescrito sem depender de um documento físico ou de contato com o médico.**

- **Entrega:** Paciente autenticado visualiza suas prescrições, identifica a consulta de origem e consulta os medicamentos, dosagens, orientações e data de emissão.
- **Critério de aceitação:** O paciente só visualiza as próprias receitas; as prescrições aparecem organizadas por data; uma receita criada no Dia 1 fica disponível na conta sem duplicação.
- **Incremento:** Transforma a disponibilização criada no Dia 1 em uma experiência de acesso direto para o paciente.

#### User Story 3 — `Notificações de prescrição realizada para o paciente` — Dia 3

**Como paciente, quero receber uma notificação quando uma nova receita for emitida, para saber que a prescrição já está disponível para consulta.**

- **Entrega planejada:** Ao criar uma receita, o sistema deve gerar uma notificação in-app para o paciente, com data, médico e atalho para a prescrição; o paciente pode marcar a notificação como lida. No código atual, a notificação básica já é criada durante a emissão; este dia deve consolidar a experiência e garantir seu processamento resiliente.
- **Critério de aceitação:** A notificação é enviada apenas ao paciente da consulta; cada receita gera uma única notificação; o atalho direciona para a prescrição correta e o status de leitura é atualizado.
- **Incremento:** Melhora a descoberta da receita disponibilizada nos Dias 1 e 2 sem alterar o acesso já existente pela conta do paciente.

---

### Ficou de fora (e por quê)

| Feature descartada | Por que não entrou entre as 3 |
| --- | --- |
| E-mail para validação de cadastro | É importante para segurança e confirmação de identidade, mas pode ser adicionado depois que o fluxo principal de cadastro e acesso estiver validado. |
| Laudo de imagem | Amplia o escopo clínico e de armazenamento, mas não é necessário para validar o MVP de agendamento, prontuário e receita digital. |

---
