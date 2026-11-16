# ifpb_labspace

Sistema de Visualização Espacial de Laboratórios do IFPB: planta baixa interativa de cada laboratório, com o status de cada equipamento e um fluxo de chamados técnicos do reporte ao fechamento.

## Documentação

| Documento | Conteúdo |
| :--- | :--- |
| [docs/intent.md](docs/intent.md) | **Por quê:** problema, usuários, objetivo e escopo |
| [docs/spec.md](docs/spec.md) | **O quê:** perfis, RBAC, estados e transições, regras de negócio (RN), requisitos funcionais (RF) e não funcionais (RNF) |
| [docs/casos-de-uso.md](docs/casos-de-uso.md) | Casos de uso UC01–UC24 e matriz de rastreabilidade RF → UC |
| [docs/plan.md](docs/plan.md) | **Como:** stack, decisões de implementação, entidades e etapas |

## Stack

Flask · Jinja2 · HTMX · Bootstrap 5 · RQ + Redis (detalhes em [docs/plan.md](docs/plan.md))
