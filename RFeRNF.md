# Projeto: Sistema de Visualização Espacial de Laboratórios (IFPB)

## 1. DEFINIÇÃO DE PERFIS E MATRIZ DE ACESSO (RBAC)

### 1.1 Perfis de Usuário

- **ESTUDANTE:** Usuário com e-mail `@academico.ifpb.edu.br`. Foco em visualização e reporte pontual de falhas.
- **SERVIDOR:** Usuário com e-mail `@ifpb.edu.br` (perfil padrão). Pode movimentar itens, editar observações e reportar problemas estruturais.
- **ADMINISTRADOR:** Usuário com e-mail `@ifpb.edu.br` autenticado com privilégios de gestão total, abertura/fechamento de chamados e aprovação de cadastros.

### 1.2 Matriz de Permissões (RBAC)

**Legenda:** `X` Permitido · vazio = Não permitido · `R` Apenas reportar

| Funcionalidade / Ação | Estudante | Servidor | Administrador |
| :--- | :---: | :---: | :---: |
| Visualizar laboratórios, plantas e status | X | X | X |
| Registrar ocorrências (Texto livre + Foto) | X | X | X |
| Mudar status do equipamento para Vermelho (Falha) | X | X | X |
| Alterar posição de equipamentos na planta / Periféricos |  | X | X |
| Editar observações gerais da sala |  | X | X |
| Criar, editar dimensões ou excluir Salas |  |  | X |
| Abrir Chamado Técnico (Mudar p/ Amarelo) |  | R | X |
| Fechar Chamado Formalmente (Mudar p/ Verde) |  |  | X |
| Fila Geral de Chamados e Gestão de Usuários |  |  | X |

## 2. SEPARAÇÃO DE ESTADOS E CICLO DE VIDA

### 2.1 Estados do Equipamento (Visual / Físico)

- 🟢 **VERDE (Operacional):** Equipamento funcionando normalmente.
- 🟡 **AMARELO (Em Manutenção / Chamado Aberto):** Há intervenção técnica em andamento.
- 🔴 **VERMELHO (Falha Crítica):** Equipamento quebrado ou inoperante, pendente de triagem.

### 2.2 Estados do Chamado Técnico

- **ABERTO:** Chamado registrado pelo Administrador (ou reportado para triagem).
- **ASSUMIDO:** Administrador visualizou e iniciou a tratativa.
- **RESOLVIDO:** Intervenção concluída tecnicamente, aguardando validação/fechamento formal.
- **FECHADO:** Chamado encerrado formalmente pelo Administrador, retornando o equipamento ao estado Verde.

## 3. REQUISITOS FUNCIONAIS (RF) - COM CRITÉRIOS DE ACEITAÇÃO

### 3.1 Autenticação e Gestão de Usuários

- **RF01 - Cadastro por Domínio Institucional:** O sistema deve exigir seleção prévia do domínio (@academico.ifpb.edu.br ou @ifpb.edu.br) e validar o prefixo informado.
    - *Critério de Aceitação / Erro:* Se o usuário tentar inserir um e-mail completo fora do padrão selecionado, o backend deve recusar com erro HTTP 400 e mensagem explicativa.
- **RF02 - Classificação Automática:** Atribuição imediata de perfil com base no domínio.
- **RF03 - Solicitação Administrativa:** Domínios `@ifpb.edu.br` podem solicitar perfil de Administrador, caindo em fila de aprovação pendente.
- **RF04 - Confirmação e Recuperação:** Envio de token de confirmação no primeiro acesso e fluxo de redefinição de senha.
- **RF05 - Gestão de Usuários (Admin):** Painel para listar, ativar/desativar e gerenciar solicitações de administradores.

### 3.2 Cadastro de Salas e Equipamentos

- **RF06 - Cadastro de Laboratórios:** Inserção de largura e comprimento reais para cálculo proporcional.
- **RF07 - Cadastro de Equipamentos e Periféricos:** Registro de máquinas principais e seus componentes vinculados.
- **RF08 - Representação por Ícones:** Associação visual automática por categoria de equipamento.
- **RF09 - Posicionamento Espacial (Drag-and-Drop):** Arrastar e soltar itens na planta baixa.
- **RF10 - Modificação de Layout:** Admin e Servidor podem realocar itens entre salas ou dentro da mesma sala.
- **RF11 - Exclusão com Confirmação:** Requer modal de confirmação antes de deletar itens.
- **RF12 - Upload de Imagens:** Suporte a envio de imagens com limite técnico de 5MB e formatos JPG/PNG.
    - *Critério de Erro:* Arquivos acima de 5MB ou extensões inválidas (ex: .pdf, .exe) devem retornar erro HTTP 413/400 informando o limite.
- **RF13 - Formatação de Observações:** Campos de texto suportando formatação básica para notas de salas e equipamentos.

### 3.3 Gestão de Status, Ocorrências e Auditoria

- **RF14 - Transição de Status por Perfil:** Estudantes/Servidores podem mover de Verde para Vermelho. Apenas Administradores movem para Amarelo ou retornam para Verde.
- **RF15 - Registro de Ocorrências:** Inserção livre de texto e foto vinculada ao equipamento, visível a todos.
- **RF16 - Log Estruturado de Eventos:** Cada alteração gera um registro imutável de auditoria (Quem, Quando, De qual estado para qual estado).
- **RF17 - Histórico de Status:** Manutenção do histórico completo de vida do equipamento, preservado mesmo se o item for alterado de local.
- **RF18 - Fila de Chamados Abertos:** Tela centralizada para o Administrador gerenciar pendências institucionais.
- **RF19 - Encerramento Formal:** Etapa obrigatória de fechamento de chamado pelo administrador com parecer técnico gravado.

### 3.4 Notificações e Processamento em Segundo Plano

- **RF20 - Notificações In-App:** Geração de avisos na interface para abertura, alteração ou resolução de chamados.
- **RF21 - Fila Assíncrona (Celery/RQ):** Tarefas de disparo de notificações devem rodar em background via fila para não impactar a performance das respostas HTMX.

### 3.5 Visualização e Dashboard

- **RF22 - Dashboard de Laboratórios:** Cards informativos contendo o status geral resumido de cada sala.
- **RF23 - Detalhes da Sala e Planta Interativa:** Exibição da planta com zoom, pan e popovers interativos nos ícones de equipamentos.
- **RF24 - Edição em Lote:** Seleção múltipla de itens em uma sala para operações administrativas em massa.

## 4. EXCLUSÃO LÓGICA E RETENÇÃO DE DADOS (SOFT DELETE)

- **RNF01 - Soft Delete Obrigatório:** Nenhum registro de sala, equipamento, ocorrência ou chamado deve ser apagado fisicamente (DELETE) do banco de dados. O sistema deve utilizar uma coluna de marcação lógica (`deleted_at` / `is_active = False`).
- **RNF02 - Integridade do Histórico:** A exclusão lógica de um equipamento deve manter intactos todos os logs estruturados anteriores e o histórico de chamados para fins de auditoria interna.

## 5. REQUISITOS NÃO FUNCIONAIS (RNF)

- **RNF03 - Stack Tecnológica:** Backend em Flask, templates Jinja2, dinamismo com HTMX e interface Bootstrap 5.
- **RNF04 - Desacoplamento Patrimonial:** O sistema restringe-se ao mapeamento espacial e operacional, sem controle contábil/patrimonial.
- **RNF05 - Processamento Assíncrono:** Uso obrigatório de Celery ou RQ com Redis para tarefas pesadas e notificações.
- **RNF06 - Proporção Espacial e Responsividade:** Manutenção de proporção fiel baseada na escala real com suporte a zoom, pan e adaptação a diferentes tamanhos de tela.
- **RNF07 - Validação Robusta no Servidor:** Regras de segurança, permissões e tipos de dados validados estritamente no backend em Flask.