# Plan – Sistema de Visualização Espacial de Laboratórios (IFPB)

> **Como** o sistema será construído. Baseado em [spec.md](spec.md) e [casos-de-uso.md](casos-de-uso.md). Documento vivo: atualizar conforme as decisões forem tomadas.

## 1. Stack

| Camada | Tecnologia |
| :--- | :--- |
| Backend | Flask (blueprints), Flask-Login, Flask-WTF |
| Banco | SQLAlchemy + Flask-Migrate (SQLite em dev, PostgreSQL em produção) |
| Templates | Jinja2 + Bootstrap 5 |
| Interatividade | HTMX (atualizações parciais), JS leve para arrastar e soltar, zoom e pan da planta |
| Tarefas em segundo plano | RQ (Redis Queue) com Redis: e-mails e notificações (RF21) |

Bibliotecas de apoio:

| Necessidade | Biblioteca | Requisito |
| :--- | :--- | :--- |
| Hash de senha | `argon2-cffi` (Argon2id) | RNF04 |
| Limite de tentativas de login | Flask-Limiter com armazenamento no Redis, contando só as falhas (`deduct_when`) | RNF04 |
| Sanitização das observações | `nh3` (lista de tags permitidas) | RF13 |
| Validação de imagem e remoção de EXIF | Pillow | RN05, RF12 |
| Envio de e-mail | Flask-Mail, chamado só pelos workers do RQ | RN04, RF21 |

## 2. Decisões de implementação

Decisões que antes estavam no spec e pertencem ao *como*:

- **Exclusão lógica (RN02):** coluna `deleted_at` (nulo = ativo) e `deleted_by` em todas as entidades excluíveis. **Não usar `is_active`** para isso, porque o nome colide com `Usuario.ativo` (conta ativa ou desativada) e com o `UserMixin.is_active` do Flask-Login. Uma exclusão em cascata grava o mesmo `exclusao_id` em todos os itens afetados, para que a restauração da sala traga de volta só eles (RF30).
- **Consultas padrão:** um filtro global do SQLAlchemy esconde registros com `deleted_at` preenchido; as telas de itens excluídos (UC22) usam consultas explícitas.
- **Concorrência (RN03):** `Equipamento`, `Chamado` e `SolicitacaoChamado` têm a coluna `versao` (*optimistic locking* do SQLAlchemy). Conflito → HTTP 409 e a interface recarrega o fragmento pelo HTMX.
- **Transições (2.2 a 2.5 do spec):** um único módulo `app/domain/transicoes.py` define as tabelas de transição; rotas e serviços só mudam estado por ele, que também grava o `LogEvento` na mesma transação (RF16).
- **CSRF:** Flask-WTF `CSRFProtect` em toda a aplicação; o token vai num `<meta>` do layout e o HTMX o envia no cabeçalho `X-CSRFToken` (`hx-headers` no `<body>`).
- **Sessão (RNF04):** `PERMANENT_SESSION_LIFETIME` de 2 h renovado a cada requisição, `SESSION_COOKIE_HTTPONLY`, `SESSION_COOKIE_SECURE` e `SESSION_COOKIE_SAMESITE="Lax"`. Uma coluna `Usuario.sessao_versao` é incrementada ao desativar a conta, redefinir ou trocar a senha, invalidando as sessões antigas (RF04, RF05, RF27). Na troca de senha (RF27), a sessão atual é regravada com a nova versão para continuar válida.
- **Tokens (RF04):** gerados com `secrets.token_urlsafe`; o banco guarda só o hash SHA-256.
- **Imagens (RN05):** Pillow abre o arquivo para confirmar o formato real, regrava sem EXIF e salva com nome aleatório fora da pasta `static`; o acesso passa por uma rota que confere a autenticação. `MAX_CONTENT_LENGTH` de 5 MB gera o HTTP 413.
- **Fila assíncrona (RF21):** o RQ roda com `Retry(max=3)`; falhas finais vão para a `FailedJobRegistry` e são registradas no log da aplicação.
- **Exclusão de equipamento Amarelo (T8):** a mesma transação cancela o chamado, aplica T7 e só então grava `deleted_at` e `status_antes_exclusao` (sempre Verde ou Vermelho).
- **Posição livre (RF10):** o servidor procura, numa grade de 0,5 m, a primeira célula que não fica sobre outro ícone; se não houver, a interface pede que o usuário escolha a posição.
- **Parâmetros (spec, seção 6):** validade dos tokens, limites de tamanho, bloqueio de login, sessão e limite de equipamentos ficam em `config.py`, lidos de variáveis de ambiente com esses valores como padrão.
- **Anonimização (RNF06):** uma tarefa agendada no RQ (`rq-scheduler`) anonimiza, uma vez por dia, contas desativadas há mais de 2 anos; o pedido do titular é atendido pela ação do UC18. O `LogEvento` referencia o usuário pelo id, então o log não é alterado.
- **Primeiro Administrador:** comando `flask criar-admin` executado na implantação (spec, 1.2).
- **Categorias iniciais (RF28):** carregadas por uma migração de dados (Computador, Notebook, Projetor, Impressora, Roteador, Monitor, Teclado, Mouse).
- **Ícones e status (RNF05):** biblioteca Bootstrap Icons; cada status tem cor, ícone e texto (`check-circle`, `wrench`, `x-octagon`), e os ícones da planta têm `aria-label`.
- **Planta (RNF02):** posições guardadas em metros (`pos_x`, `pos_y`); a escala em pixels é calculada no frontend a partir de `largura_m` e `comprimento_m`.
- **Backup (RNF09):** `pg_dump` diário e cópia da pasta de imagens por *cron*, enviados a um servidor ou volume de backup separado do servidor da aplicação, com retenção de 30 dias.

## 3. Estrutura de pastas (proposta)

```
ifpb_labspace/
├── app/
│   ├── __init__.py          # create_app()
│   ├── models/              # entidades SQLAlchemy
│   ├── domain/              # transições de estado e regras (RN01–RN07)
│   ├── auth/                # UC01–UC04, UC20
│   ├── labs/                # salas, planta, equipamentos, categorias (UC05, UC06, UC10, UC11, UC13, UC14, UC19, UC22, UC23)
│   ├── tickets/             # ocorrências, solicitações e chamados (UC08, UC09, UC12, UC15–UC17, UC21)
│   ├── admin/               # usuários e auditoria (UC18, UC24)
│   ├── notifications/       # UC07 + tarefas assíncronas
│   ├── templates/
│   └── static/
├── migrations/
├── tests/
├── docs/
├── config.py
└── requirements.txt
```

## 4. Entidades principais

- **Usuario:** nome, email, senha_hash, perfil (ESTUDANTE, SERVIDOR, ADMIN), ativo, confirmado, sessao_versao, anonimizado_em
- **SolicitacaoAdmin:** usuario, estado (PENDENTE, APROVADA, RECUSADA), justificativa, decidido_por, decidido_em
- **Token:** usuario, tipo (CONFIRMACAO, RECUPERACAO), token_hash, expira_em, usado_em
- **Categoria:** nome, tipo (EQUIPAMENTO, PERIFERICO), icone
- **Sala:** nome, bloco, largura_m, comprimento_m, observacoes
- **Equipamento:** sala, categoria, nome, descricao, observacoes, foto, pos_x, pos_y, status (VERDE, AMARELO, VERMELHO), status_antes_exclusao, versao
- **Periferico:** equipamento, categoria, nome (sem status próprio)
- **Ocorrencia:** equipamento, periferico (opcional), autor, texto, foto, motivo_ocultacao
- **SolicitacaoChamado:** equipamento ou sala, solicitante, descricao, foto, estado (PENDENTE, CONVERTIDA, DESCARTADA), justificativa, chamado, versao
- **Chamado:** equipamento ou sala, solicitacao (opcional), reportado_por, responsavel, estado (ABERTO, ASSUMIDO, RESOLVIDO, REABERTO, FECHADO, CANCELADO), prioridade (BAIXA, MEDIA, ALTA), descricao_tecnica, nota_tecnica, parecer, resultado (CONSERTADO, SEM_CONSERTO), status_equipamento_anterior, versao
- **LogEvento:** entidade, entidade_id, autor, acao, valor_anterior, valor_novo, criado_em (imutável – RN01, RF16)
- **Notificacao:** usuario, evento, mensagem, link, lida_em

Sala, Equipamento, Periferico, Categoria, Ocorrencia, SolicitacaoChamado e Chamado têm `deleted_at`, `deleted_by` e `exclusao_id` (seção 2). Usuario não é excluído, só desativado (RN02).

## 5. Etapas de implementação

1. **Base do projeto:** `create_app`, configuração, banco, migrações, layout base com Bootstrap, CSRF e cabeçalhos de segurança.
2. **Autenticação:** UC01–UC04 e UC20 (cadastro por domínio, confirmação, login com limite de tentativas, recuperação e troca de senha, logout) e o comando `flask criar-admin`.
3. **Núcleo de domínio:** tabelas de transição, `LogEvento` e exclusão lógica, com testes unitários de todas as transições T1–T9 e dos estados de chamado e solicitação.
4. **Salas, categorias e equipamentos:** UC13, UC14, UC23 (CRUD com exclusão lógica e log).
5. **Visualização:** UC05 (dashboard) e UC06 (planta com zoom, pan, popovers e navegação por teclado).
6. **Layout:** UC10, UC11 (arrastar e soltar, mover entre salas, alternativa por teclado, observações).
7. **Ocorrências e status:** UC08, UC09, UC21.
8. **Solicitações e chamados:** UC12, UC15, UC16, UC17.
9. **Notificações e fila assíncrona:** UC07 + worker do RQ.
10. **Gestão de usuários e auditoria:** UC18, UC24 e a tarefa de anonimização.
11. **Itens excluídos e edição em lote:** UC22, UC19.
12. **Qualidade:** revisão de acessibilidade (WCAG 2.1 AA), testes de desempenho da planta com 200 equipamentos, rotina de backup e aviso de privacidade.
