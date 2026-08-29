# Vida Conecta

Plataforma que conecta **pacientes** a **médicos** por consulta em vídeo, com histórico clínico compartilhado entre profissionais — apenas com **consentimento explícito** do paciente.

Aplicação **pequena**: um backend monolítico modular, um frontend web e um serviço de vídeo em tempo real **desacoplado**.

---

## Contexto do negócio

Pacientes agendam consultas, realizam videochamada com o médico, consultam o próprio prontuário e recebem prescrição digital. Médicos acessam o histórico clínico compartilhado somente quando o paciente autoriza.

### Escopo

| Módulo | O que faz |
| --- | --- |
| Agendamento | Paciente escolhe médico, data e horário; confirma ou cancela consulta |
| Videochamada | Consulta ao vivo (WebRTC), isolada do restante do sistema |
| Prontuário eletrônico | Registro clínico do paciente, com storage separado e mais protegido |
| Prescrição digital | Receita gerada pelo médico e disponibilizada ao paciente |

### Fora de escopo (Entregas futuras)

Fila de espera, laudos de imagem, integração com farmácias e app mobile.

---

## Restrição especial — LGPD

Dados de saúde são **dados sensíveis**. A plataforma precisa de:

- **Controle de acesso rígido** — autenticação, papéis (paciente, médico, admin) e autorização por consulta/consentimento
- **Criptografia em trânsito** — TLS em todas as APIs e no sinalização/mídia da videochamada
- **Criptografia em repouso** — banco transacional e storage do prontuário cifrados
- **Consentimento explícito** — o paciente autoriza, por médico ou por consulta, o compartilhamento do histórico; sem consentimento válido, o médico não lê o prontuário de outro profissional

---

## SLA desejado

| Indicador | Meta |
| --- | --- |
| Disponibilidade da plataforma | **99.9%** (~8,7 h de downtime/ano) |
| Taxa de queda da videochamada | **≤ 1%** |

Uso distribuído no dia comercial, com picos no **almoço** e no **fim da tarde**. A videochamada escala de forma independente (serviço de mídia separado).

---

## Arquitetura

Monólito modular no Spring Boot: um único deploy, módulos internos (agendamento, prontuário, prescrição, consentimento, auth). O vídeo **não** passa pelo backend de negócio — só o sinal de “consulta iniciada/encerrada” e o token de sala.

### Visão geral

O frontend fala com o **backend** (API de negócio) e, na consulta, também com o **WebRTC** (mídia). Banco e prontuário só o backend acessa. **Grafana** concentra métricas e logs (disponibilidade e taxa de queda da chamada).

```mermaid
flowchart TB
  FE[Frontend<br/>React + Vite + Tailwind]

  BE[Backend<br/>Spring Boot]

  PG[(PostgreSQL<br/>usuários, agenda,<br/>consentimento, prescrição)]
  EHR[(Storage protegido<br/>prontuário cifrado)]
  SFU[WebRTC<br/>sala de videochamada]

  subgraph obs [Observabilidade]
    GRAF[Grafana]
  end

  FE -->|HTTPS / REST + JWT| BE
  FE -->|mídia da consulta| SFU

  BE --> PG
  BE --> EHR
  BE -.->|cria sala e devolve token| SFU

  BE -->|métricas e logs| GRAF
  SFU -->|taxa de queda da chamada| GRAF
```

### Módulos do backend

```mermaid
flowchart LR
  subgraph api [Spring Boot]
    AUTH[Auth e papéis]
    AG[Agendamento]
    CONS[Consentimento]
    PRONT[Prontuário]
    RX[Prescrição]
    VID[Integração vídeo]
  end

  AUTH --> AG
  AUTH --> CONS
  AUTH --> PRONT
  AUTH --> RX
  AG --> VID
  CONS --> PRONT
  PRONT --> EHR[(Storage protegido)]
  AG --> PG[(PostgreSQL)]
  CONS --> PG
  RX --> PG
  VID --> SFU[WebRTC]
```

### Fluxo de uma consulta

```mermaid
sequenceDiagram
  actor Paciente
  actor Médico
  participant API as API Spring Boot
  participant PG as PostgreSQL
  participant EHR as Storage prontuário
  participant SFU as WebRTC

  Paciente->>API: Agenda consulta
  API->>PG: Persiste slot + status
  Note over Paciente,Médico: No horário da consulta
  Paciente->>API: Entrar na sala
  Médico->>API: Entrar na sala
  API->>API: Valida sessão e papéis
  API->>SFU: Cria/libera token da sala
  API-->>Paciente: Token
  API-->>Médico: Token
  Paciente->>SFU: Conecta mídia
  Médico->>SFU: Conecta mídia
  Médico->>API: Ler prontuário
  API->>PG: Consentimento válido?
  alt Consentimento explícito ok
    API->>EHR: Lê registro cifrado
    API-->>Médico: Histórico clínico
  else Sem consentimento
    API-->>Médico: Acesso negado
  end
  Médico->>API: Grava evolução + prescrição
  API->>EHR: Persiste prontuário
  API->>PG: Persiste receita
```


## Stack

| Camada | Tecnologia |
| --- | --- |
| Frontend | React, Vite, Tailwind CSS |
| Backend | Java, Spring Boot |
| Banco transacional | PostgreSQL |
| Prontuário | Storage cifrado separado (objeto ou volume dedicado; MongoDB apenas se o documento clínico exigir) |
| Vídeo | WebRTC (SFU gerenciado, ex.: LiveKit / equivalente na AWS) |
| Nuvem | AWS (HTTPS, criptografia em repouso, backups) |
| Código | GitHub | Observabilidade com Grafana

---

## Segurança (mínimo viável LGPD)

- HTTPS em toda comunicação;
- Senhas com hash (bcrypt/argon2); sessão JWT de curta duração
- RBAC: paciente só vê os próprios dados; médico só vê o que o consentimento cobre
- Consentimento versionado (quem, o quê, até quando, revogação)
- Auditoria de acesso ao prontuário (quem leu, quando, em qual consulta)
- Segredos fora do código (variáveis de ambiente / secret manager)
- Minimização: o SFU de vídeo **não** persiste conteúdo clínico

---

## Disponibilidade e picos

- Meta 99.9%: health check da API, restart automático, banco com backup e restauração testada
- Vídeo em serviço separado para o pico de almoço/fim de tarde não derrubar agendamento nem prontuário
- Observação da taxa de queda da chamada para manter ≤ 1%

---

## Estrutura prevista

```
vida-conecta/
├── README.md
├── frontend/          # React + Vite + Tailwind
└── backend/           # Spring Boot (módulos: auth, agenda, consentimento, prontuário, prescrição, vídeo)
```

---

## Disciplinas mais críticas neste projeto

- [ ] Ecossistemas de Startups
- [x] Direito Digital e LGPD
- [x] Fundamentos de Engenharia de Software
- [x] Metodologias Ágeis em Gestão de Projetos
- [ ] Desenvolvimento de Software Integrado – DevOps
- [ ] Design da Experiência do Usuário
- [x] Controle de Versão e Gerenciamento de Configuração
- [ ] Gerenciamento de Produtos
- [x] Integração e Entrega Contínua
- [x] Orquestração de Contêineres e Gerenciamento de Cluster
- [ ] Infraestrutura Automatizada
- [ ] Desenvolvimento de Software Seguro – DevSecOps
- [x] Testes Automatizados e Contínuos
- [x] Arquitetura de Microsserviços e Escalabilidade
- [x] Documentação Técnica
- [x] Computação em Nuvem
- [ ] Computação sem Servidores
- [x] Monitoramento e Análise de Logs
- [x] Tópicos Avançados em Engenharia de Software

### Plano de Desenvolvimento

- Planejamento;
- CI/MVP;
- CD/Observabilidade;
- Plano de Testes e de escala.
