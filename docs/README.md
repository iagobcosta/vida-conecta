# Documentação do Vida Conecta

## Organização

- [`ADR/`](ADR/): decisões de arquitetura, com contexto, decisão e consequências.
- [`Features/`](Features/): macro funcionalidades e user stories incrementais.
- [`LGPD/`](LGPD/): tratamento de dados pessoais e clínicos, retenção, incidentes e direitos do titular.
- [`Arquitetura/`](Arquitetura/): estado atual e plano de evolução técnica.

## Documentos principais

- [ADR-001 — Agendamento simples](ADR/ADR-001-Agendamento-Simples.md)
- [ADR-002 — Autorização do prontuário](ADR/ADR-002-Autorizacao-Prontuario.md)
- [ADR-003 — Disponibilização da receita](ADR/ADR-003-Disponibilizacao-Receita.md)
- [MC01 — Agendamento, prontuário e receita](Features/MC01%20-%20AgendamentoConsultas.md)
- [Conformidade LGPD](LGPD/LGPD_COMPLIANCE.md)
- [Plano de arquitetura evolutiva](Arquitetura/PLANO-ARQUITETURA-EVOLUTIVA.md)

## Documentação importante ainda necessária

O projeto ainda precisa de um **contrato de API versionado**, preferencialmente baseado no OpenAPI já exposto pelo backend. Esse documento deve registrar, por recurso:

- autenticação e papéis permitidos;
- request, response e códigos de erro;
- regras de idempotência e concorrência;
- exemplos para frontend e integrações;
- política de compatibilidade entre versões.

Também é recomendável criar uma matriz de rastreabilidade ligando User Story, endpoint, regra de negócio, teste e ADR. A ausência desses dois documentos dificulta validar se uma entrega está completa e se frontend, backend e documentação continuam alinhados.
