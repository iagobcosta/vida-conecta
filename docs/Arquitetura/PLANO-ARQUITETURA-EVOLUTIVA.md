# Plano de Arquitetura Evolutiva

## Objetivo

Evoluir o MVP do Vida Conecta sem abandonar o monólito modular atual, preservando o fluxo de agendamento, prontuário e prescrição enquanto os riscos de concorrência, privacidade, disponibilidade e integração são tratados na ordem correta.

## Estado atual

- Backend Spring Boot em monólito modular, com fronteiras verificadas pelo Spring Modulith.
- PostgreSQL concentra usuários, agenda, consentimentos, evoluções clínicas e receitas.
- Evoluções clínicas são cifradas com AES-GCM dentro de `clinical_notes.ciphertext`.
- Agendamento valida disponibilidade e sobreposição na camada de serviço, mas ainda não possui proteção absoluta contra duas transações concorrentes.
- Autorização do prontuário combina titularidade, relação de atendimento, autoria da nota e consentimento válido.
- Videochamada funcional com Jitsi Meet; o backend usa token mock para autorização da sala e registra eventos da sessão.
- Deploy de produção usa imagem imutável, Docker Compose, Nginx, observabilidade e atualização automática via SSM.

## Princípios de evolução

1. Manter o monólito modular enquanto o volume e os limites de operação não justificarem serviços separados.
2. Evoluir um contrato por vez, com migration reversível ou estratégia de compatibilidade quando possível.
3. Tratar dados clínicos como domínio de alta sensibilidade e aplicar menor privilégio por padrão.
4. Medir a necessidade de escala antes de extrair um módulo para outro processo.
5. Toda mudança estrutural deve incluir testes de contrato, segurança e operação.

## Fases

### Fase 0 — Consolidar o MVP

**Objetivo:** tornar o fluxo atual verificável e reduzir ambiguidades.

**Entregas:**

- Publicar e validar o OpenAPI como contrato da API.
- Cobrir com testes as regras de agendamento, autorização do prontuário e visibilidade de receitas.
- Definir formalmente se médico relacionado pode ler apenas suas próprias notas sem consentimento geral.
- Padronizar datas como `Instant`/`TIMESTAMPTZ` e apresentação em `America/Sao_Paulo`.
- Documentar os limites atuais do token mock e dos jobs de retenção.

**Pronto quando:** a CI valida os fluxos críticos e não há diferença entre README, ADR, OpenAPI e comportamento observado.

### Fase 1 — Integridade e privacidade transacional

**Objetivo:** fechar os riscos mais graves do domínio clínico e do agendamento.

**Entregas:**

- Adicionar proteção de banco contra sobreposição concorrente de consultas, preferencialmente com `tstzrange` e restrição de exclusão no PostgreSQL.
- Mapear conflitos de persistência para `409 Conflict` de forma consistente.
- Completar auditoria de leituras e tentativas negadas do prontuário.
- Garantir que auditoria seja append-only por permissão de banco e política de acesso.
- Revisar e testar a política de consentimento, expiração e revogação imediata.

**Pronto quando:** testes concorrentes não permitem dupla reserva e testes de autorização cobrem titular, médico autorizado, médico não autorizado, consentimento expirado e revogado.

### Fase 2 — Operação segura de dados clínicos

**Objetivo:** aproximar a persistência da arquitetura desejada para dados sensíveis.

**Entregas:**

- Separar o storage clínico do banco transacional, mantendo no PostgreSQL apenas metadados e referências.
- Usar KMS/Secrets Manager ou equivalente para chaves, com rotação e procedimento de recuperação documentado.
- Implementar exportação estruturada, anonimização, retenção e descarte conforme validação jurídica.
- Criar jobs idempotentes para retenção de auditoria e prontuário.
- Testar backup e restore incluindo a decriptação de um registro clínico.

**Pronto quando:** restore é testado periodicamente, não há segredo em repositório e os direitos de exportação, anonimização e revogação possuem evidência de execução.

### Fase 3 — Notificações resilientes e contratos externos

**Objetivo:** desacoplar efeitos secundários sem alterar a transação clínica principal.

**Entregas:**

- Introduzir eventos transacionais/outbox para notificações de agendamento, consentimento, evolução e receita.
- Processar notificações com retry, idempotência e dead-letter queue quando necessário.
- Publicar documentação OpenAPI versionada e exemplos de erro para frontend e integrações.
- Implementar validação de cadastro por e-mail como fluxo separado do convite de médicos.

**Pronto quando:** falha do provedor de e-mail não desfaz agendamento ou emissão de receita e cada evento pode ser reprocessado sem duplicação.

### Fase 4 — Operação e evolução da videochamada Jitsi

**Objetivo:** tornar a integração Jitsi Meet mais resiliente, observável e adequada à produção, mantendo a autorização da sala no backend.

**Entregas:**

- Definir entre Jitsi self-hosted e serviço público a partir de custo, região, disponibilidade e LGPD.
- Manter no backend apenas autorização da sala, emissão do token mock e eventos de início/fim.
- Adicionar métricas de entrada, duração, latência e queda de chamada.
- Testar janela de acesso, reconexão, encerramento, configuração do domínio e isolamento entre consultas.

**Pronto quando:** o Jitsi atende à meta de disponibilidade definida, possui monitoramento com alertas e o fluxo de autorização continua impedindo entradas fora da janela ou da consulta.

### Fase 5 — Escala seletiva

**Objetivo:** separar componentes somente quando houver pressão operacional comprovada.

**Gatilhos:**

- filas de notificação ou vídeo afetando o SLA da API;
- volume de leitura clínica exigindo escala independente;
- deploy do módulo de vídeo exigindo ciclo diferente do domínio transacional;
- necessidade de isolamento de segurança ou de equipe.

**Possíveis extrações:** serviço de mídia, worker de notificações, storage clínico e, posteriormente, leitura de agenda. O domínio transacional de agendamento deve ser o último candidato a ser separado, devido à necessidade de consistência forte.

## Qualidade e governança

Cada fase deve produzir:

- ADR para decisões irreversíveis ou com impacto estrutural;
- migration versionada e plano de rollback;
- testes automatizados da regra alterada;
- métricas e alertas operacionais;
- atualização do README, OpenAPI e documentação LGPD;
- registro de riscos aceitos e data de revisão.

## Priorização resumida

| Ordem | Risco tratado | Próximo resultado |
| --- | --- | --- |
| 1 | Divergência entre contrato e implementação | Documentação e testes confiáveis |
| 2 | Dupla reserva e autorização clínica | Integridade e privacidade reforçadas |
| 3 | Retenção, backup e storage clínico | Operação compatível com dados sensíveis |
| 4 | Notificações e e-mail | Efeitos secundários resilientes |
| 5 | Operação da videochamada Jitsi | Telemedicina observável e resiliente |
| 6 | Escala seletiva | Serviços separados apenas quando justificados |
