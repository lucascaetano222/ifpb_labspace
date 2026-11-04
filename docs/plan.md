# Plan – Sistema de Visualização Espacial de Laboratórios (IFPB)

> **Como** o sistema será construído. Baseado em [spec.md](spec.md) e [casos-de-uso.md](casos-de-uso.md). Documento vivo: atualizar conforme as decisões forem tomadas.

## 1. Stack

| Camada | Tecnologia |
| :--- | :--- |
| Backend | Flask (blueprints), Flask-Login, Flask-WTF |
| Banco | SQLAlchemy + Flask-Migrate (SQLite em dev, PostgreSQL em produção) |
| Templates | Jinja2 + Bootstrap 5 |
| Interatividade | HTMX (atualizações parciais), JS leve para drag-and-drop, zoom e pan da planta |
| Tarefas em segundo plano | RQ (Redis Queue) com Redis (e-mails e notificações – RN04) |

## 2. Estrutura de pastas (proposta)

```
ifpb_labspace/
├── app/
│   ├── __init__.py          # create_app()
│   ├── models/              # entidades SQLAlchemy
│   ├── auth/                # UC01–UC04
│   ├── labs/                # salas, planta, equipamentos (UC05, UC06, UC10, UC11, UC13, UC14, UC19)
│   ├── tickets/             # ocorrências e chamados (UC08, UC09, UC12, UC15–UC17)
│   ├── admin/               # usuários (UC18)
│   ├── notifications/       # UC07 + tarefas assíncronas
│   ├── templates/
│   └── static/
├── migrations/
├── tests/
├── docs/
├── config.py
└── requirements.txt
```

## 3. Entidades principais

- **Usuario:** nome, email, senha_hash, perfil (ESTUDANTE, SERVIDOR, ADMIN), ativo, confirmado
- **SolicitacaoAdmin:** usuario, status (PENDENTE, APROVADA, RECUSADA), justificativa
- **Token:** usuario, tipo (CONFIRMACAO, RECUPERACAO), expira_em, usado
- **Sala:** nome, bloco, largura_m, comprimento_m, observacoes
- **Equipamento:** sala, nome, categoria, descricao, foto, pos_x, pos_y, status (VERDE, AMARELO, VERMELHO)
- **Periferico:** equipamento (pai), nome, categoria
- **Ocorrencia:** equipamento, autor, texto, foto
- **Chamado:** equipamento ou sala, solicitante, responsavel, estado (ABERTO, ASSUMIDO, RESOLVIDO, FECHADO), prioridade, parecer
- **LogEvento:** entidade, entidade_id, autor, acao, estado_anterior, estado_novo, criado_em (imutável – RN01)
- **Notificacao:** usuario, mensagem, link, lida

Todas as entidades excluíveis têm `deleted_at` (RN02). As posições ficam em metros (`pos_x`, `pos_y`), e a escala em pixels é calculada no frontend (RNF06).

## 4. Etapas de implementação

1. **Base do projeto:** `create_app`, configuração, banco, migrações, layout base com Bootstrap.
2. **Autenticação:** UC01–UC04 (cadastro por domínio, confirmação, login, recuperação de senha).
3. **Salas e equipamentos:** UC13, UC14 (CRUD com soft delete e log).
4. **Visualização:** UC05 (dashboard) e UC06 (planta com zoom, pan e popovers).
5. **Layout:** UC10, UC11 (drag-and-drop, mover entre salas, observações).
6. **Ocorrências e status:** UC08, UC09.
7. **Chamados:** UC12, UC15, UC16, UC17.
8. **Notificações e fila assíncrona:** UC07 + worker do RQ.
9. **Gestão de usuários:** UC18.
10. **Edição em lote:** UC19.

## 5. Decisões tomadas

- **Fila assíncrona: RQ** (em vez de Celery). Mais simples de aprender e configurar, e cobre o que o projeto precisa: e-mails e notificações. Em Windows, o Redis e o worker do RQ rodam via WSL ou Docker.

## 6. Decisões em aberto

- Biblioteca para zoom e pan da planta (ex.: panzoom) ou implementação própria?
- Editor de texto para observações (RF13): Markdown simples ou editor WYSIWYG leve?
