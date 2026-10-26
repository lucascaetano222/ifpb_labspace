## UC01 – Cadastrar-se

- **Ator:** Visitante
- **Requisitos:** RF01, RF02, RF03
- **Pré-condição:** nenhuma conta ativa com o e-mail informado.
- **Pós-condição:** conta criada como inativa, com perfil atribuído pelo domínio e token de confirmação enviado.
- **Include:** Validar domínio institucional. **Extend:** Solicitar perfil de Administrador.

1. O Visitante acessa a tela de cadastro.
2. O sistema exibe o seletor de domínio (@academico.ifpb.edu.br ou @ifpb.edu.br).
3. O Visitante seleciona o domínio e informa nome, prefixo do e-mail e senha.
4. O sistema executa Validar domínio institucional: concatena prefixo e domínio e verifica o formato.
5. O sistema atribui o perfil: Estudante para @academico, Servidor para @ifpb (RF02).
6. O sistema cria a conta inativa e enfileira o e-mail de confirmação (RN04).
7. O sistema informa que o link de confirmação foi enviado.

- **FA1 – Solicitar perfil de Administrador (passo 3, só @ifpb.edu.br):** o Visitante marca "Solicitar perfil de Administrador"; no passo 6 o sistema cria também uma solicitação pendente na fila de aprovação (RF03). A conta funciona como Servidor até a aprovação no UC18.
- **FE1 – E-mail fora do padrão (passo 4):** o Visitante digitou um e-mail completo de outro domínio. O backend recusa com HTTP 400 e mensagem explicando que só o prefixo deve ser informado.
- **FE2 – E-mail já cadastrado (passo 4):** o sistema informa que já existe conta e oferece Recuperar senha (UC04).
- **FE3 – Senha fraca (passo 4):** o sistema mostra os requisitos mínimos de senha e mantém os demais campos preenchidos.

## UC02 – Confirmar conta

- **Atores:** Visitante; Serviço de E-mail (secundário)
- **Requisitos:** RF04
- **Pré-condição:** conta criada pelo UC01 e ainda inativa.
- **Pós-condição:** conta ativa; o usuário pode fazer login.

1. O Serviço de E-mail entrega a mensagem com o link contendo o token.
2. O Visitante clica no link.
3. O sistema valida o token (existe, não usado, dentro da validade).
4. O sistema ativa a conta e invalida o token.
5. O sistema redireciona para o login com mensagem de sucesso.

- **FE1 – Token expirado (passo 3):** o sistema informa a expiração e oferece reenviar o link, voltando ao passo 1.
- **FE2 – Token inválido ou já usado (passo 3):** o sistema mostra erro e não altera a conta.

## UC03 – Fazer login

- **Ator:** Visitante
- **Requisitos:** RF02, RNF07
- **Pré-condição:** conta ativa.
- **Pós-condição:** sessão autenticada com o perfil do usuário.

1. O Visitante informa e-mail e senha.
2. O sistema valida as credenciais.
3. O sistema cria a sessão com o perfil (Estudante, Servidor ou Administrador).
4. O sistema redireciona para o dashboard (UC05).

- **FE1 – Credenciais inválidas (passo 2):** mensagem genérica, sem revelar se o e-mail existe.
- **FE2 – Conta não confirmada (passo 2):** o sistema informa e oferece reenviar a confirmação (UC02).
- **FE3 – Conta desativada pelo Administrador (passo 2):** o sistema bloqueia o acesso e orienta procurar a coordenação.

## UC04 – Recuperar senha

- **Atores:** Visitante; Serviço de E-mail (secundário)
- **Requisitos:** RF04
- **Pré-condição:** nenhuma.
- **Pós-condição:** senha redefinida e token invalidado.

1. O Visitante clica em "Esqueci minha senha" e informa o e-mail.
2. O sistema mostra sempre a mesma mensagem de envio, exista ou não a conta.
3. Se a conta existe, o sistema gera um token e o Serviço de E-mail entrega o link.
4. O Visitante abre o link e informa a nova senha duas vezes.
5. O sistema valida o token e a senha, grava a nova senha e invalida o token.
6. O sistema redireciona para o login.

- **FE1 – Token expirado ou inválido (passo 5):** o sistema informa e oferece recomeçar no passo 1.
- **FE2 – Senhas não coincidem (passo 4):** o sistema pede nova digitação.

## UC05 – Visualizar dashboard de laboratórios

- **Ator:** Estudante (e, por herança, Servidor e Administrador)
- **Requisitos:** RF22
- **Pré-condição:** usuário autenticado.
- **Pós-condição:** nenhuma alteração de dados.

1. O usuário acessa a página inicial.
2. O sistema lista um card por sala ativa, com nome, localização e contagem de equipamentos por status (Verde, Amarelo, Vermelho).
3. O usuário pode filtrar por bloco ou por status.
4. O usuário clica em um card e o sistema inicia o UC06.

- **FA1 – Nenhuma sala cadastrada (passo 2):** o sistema mostra estado vazio; para o Administrador, exibe atalho para Manter salas (UC13).

## UC06 – Visualizar planta da sala

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF08, RF23, RNF06
- **Pré-condição:** usuário autenticado; sala ativa.
- **Pós-condição:** nenhuma alteração de dados.
- **Extend:** Consultar histórico do equipamento.

1. O sistema desenha a planta na proporção real da largura e comprimento da sala.
2. O sistema posiciona os ícones por categoria, coloridos pelo status atual.
3. O usuário aplica zoom e pan.
4. O usuário passa o cursor ou toca em um ícone.
5. O sistema exibe popover com nome, categoria, periféricos, status e última ocorrência.
6. O popover mostra as ações permitidas ao perfil: reportar falha, registrar ocorrência, mover, abrir chamado.

- **FA1 – Consultar histórico (passo 5, extend):** o usuário clica em "Histórico"; o sistema lista as mudanças de status, ocorrências e chamados do equipamento em ordem cronológica, inclusive de salas anteriores (RF17).
- **FA2 – Tela pequena (passo 1):** o sistema ajusta a escala ao viewport mantendo a proporção.

## UC07 – Visualizar notificações

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF20
- **Pré-condição:** usuário autenticado.
- **Pós-condição:** notificações abertas marcadas como lidas.

1. O sistema mostra no cabeçalho o contador de notificações não lidas.
2. O usuário abre a lista.
3. O sistema exibe avisos de abertura, alteração e resolução de chamados ligados ao usuário (reportou, solicitou ou é Administrador).
4. O usuário clica em um aviso e o sistema abre o equipamento ou chamado correspondente.
5. O sistema marca o aviso como lido.

- **FA1 – Marcar todas como lidas (passo 3):** o usuário zera o contador de uma vez.

## UC08 – Registrar ocorrência

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF15, RF12, RF13
- **Pré-condição:** usuário autenticado; equipamento ativo.
- **Pós-condição:** ocorrência gravada, vinculada ao equipamento e visível a todos; status não muda.
- **Extend:** Anexar foto.

1. No popover do equipamento (UC06), o usuário escolhe "Registrar ocorrência".
2. O usuário descreve a situação em texto livre.
3. O usuário confirma.
4. O sistema grava a ocorrência com autor e data e atualiza o popover via HTMX.

- **FA1 – Anexar foto (passo 2, extend):** o usuário envia uma imagem; o sistema aplica a RN05 e mostra a miniatura.
- **FE1 – Texto vazio (passo 3):** o sistema exige a descrição.
- **FE2 – Imagem inválida (FA1):** HTTP 413 ou 400 com o limite permitido; o texto digitado é mantido.

## UC09 – Reportar falha (Verde → Vermelho)

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF14, RF15, RF16, RF20
- **Pré-condição:** usuário autenticado; equipamento em Verde.
- **Pós-condição:** equipamento em Vermelho, ocorrência e log gravados, Administradores notificados.
- **Extend:** Anexar foto.

1. No popover do equipamento, o usuário escolhe "Reportar falha".
2. O sistema abre o formulário de descrição.
3. O usuário descreve o problema e confirma.
4. O sistema verifica a permissão e o estado atual (RN03).
5. O sistema muda o status para Vermelho, grava a ocorrência e o log (RN01).
6. O sistema enfileira notificação aos Administradores (RN04).
7. O ícone na planta passa a vermelho.

- **FA1 – Anexar foto (passo 3, extend):** igual ao FA1 do UC08.
- **FA2 – Equipamento já Vermelho ou Amarelo (passo 4):** o sistema não muda o status e oferece registrar uma ocorrência adicional (UC08).
- **FE1 – Conflito de edição (passo 5):** outro usuário mudou o status no mesmo instante; o sistema recarrega o estado e informa.

## UC10 – Movimentar equipamento na planta

- **Ator:** Servidor (e Administrador por herança)
- **Requisitos:** RF09, RF10, RF16, RF17
- **Pré-condição:** usuário autenticado como Servidor ou Administrador; sala e equipamento ativos.
- **Pós-condição:** nova posição (e sala, se mudou) gravada; histórico do equipamento preservado.

1. Na planta (UC06), o usuário ativa o modo de edição de layout.
2. O usuário arrasta o ícone para a nova posição.
3. O sistema verifica se a posição está dentro dos limites da sala.
4. O sistema grava as coordenadas e o log (RN01).
5. O sistema confirma via HTMX sem recarregar a página.

- **FA1 – Mover para outra sala (passo 2):** o usuário escolhe "Mover para…" e seleciona a sala de destino; o equipamento é colocado numa posição livre da nova planta e o histórico segue com ele (RF17).
- **FA2 – Mover periférico (passo 2):** o usuário desvincula um periférico de uma máquina e o vincula a outra.
- **FE1 – Posição fora da sala (passo 3):** o ícone volta à posição anterior com aviso.
- **FE2 – Estudante tenta mover (passo 1):** o modo de edição não aparece; uma requisição direta recebe HTTP 403 (RN03).

## UC11 – Editar observações da sala

- **Ator:** Servidor (e Administrador)
- **Requisitos:** RF13, RF16
- **Pré-condição:** usuário autenticado como Servidor ou Administrador; sala ativa.
- **Pós-condição:** observação atualizada e versão anterior registrada no log.

1. Na página da sala, o usuário clica em "Editar observações".
2. O sistema abre o editor com formatação básica (negrito, itálico, listas, links).
3. O usuário edita e salva.
4. O sistema sanitiza o conteúdo, grava e registra o log.

- **FA1 – Observação de equipamento (passo 1):** o mesmo fluxo vale para a nota de um equipamento, a partir do popover.
- **FE1 – Conteúdo com script ou HTML não permitido (passo 4):** o sistema remove as tags proibidas antes de gravar.

## UC12 – Solicitar abertura de chamado

- **Ator:** Servidor
- **Requisitos:** RF14, RF18, RF20 (marcação "R" da matriz RBAC)
- **Pré-condição:** usuário autenticado como Servidor; equipamento em Vermelho ou com problema estrutural.
- **Pós-condição:** solicitação na fila do Administrador; o status do equipamento não muda.

1. No popover do equipamento ou na página da sala, o Servidor escolhe "Solicitar chamado".
2. O Servidor descreve o problema e pode anexar foto (RN05).
3. O sistema grava a solicitação com estado ABERTO para triagem.
4. O sistema enfileira notificação aos Administradores.
5. O sistema informa que o Administrador fará a triagem.

- **FA1 – Problema estrutural da sala (passo 1):** a solicitação é vinculada à sala, não a um equipamento.
- **FE1 – Já existe chamado aberto para o item (passo 3):** o sistema mostra o chamado existente e oferece adicionar uma ocorrência a ele (UC08).


## UC13 – Manter salas

- **Ator:** Administrador
- **Requisitos:** RF06, RF11, RNF01, RNF06
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** sala criada, alterada ou marcada como excluída.
- **Include:** Confirmar exclusão.

1. O Administrador escolhe "Nova sala".
2. Informa nome, bloco, largura e comprimento em metros e observações.
3. O sistema valida que as dimensões são números positivos.
4. O sistema grava a sala e gera a planta vazia na proporção informada.

- **FA1 – Editar (passo 1):** o Administrador altera dados ou dimensões; o sistema recalcula a escala e mantém as posições relativas dos equipamentos.
- **FA2 – Excluir (passo 1):** o sistema executa Confirmar exclusão, mostrando quantos equipamentos estão na sala; após confirmar, aplica soft delete (RN02).
- **FE1 – Dimensões inválidas (passo 3):** o sistema rejeita zero, negativos ou texto.
- **FE2 – Redução que deixa equipamentos fora (FA1):** o sistema lista os itens afetados e exige reposicioná-los antes de salvar.

## UC14 – Manter equipamentos e periféricos

- **Ator:** Administrador
- **Requisitos:** RF07, RF08, RF11, RF12, RNF02, RNF04
- **Pré-condição:** usuário autenticado como Administrador; sala ativa.
- **Pós-condição:** equipamento ou periférico criado, alterado ou marcado como excluído; histórico intacto.
- **Include:** Confirmar exclusão.

1. Na planta, o Administrador escolhe "Novo equipamento".
2. Informa nome, categoria, descrição e, opcionalmente, foto (RN05).
3. O sistema associa o ícone pela categoria (RF08).
4. O Administrador posiciona o ícone na planta.
5. O sistema grava o equipamento em Verde e registra o log.

- **FA1 – Cadastrar periférico (passo 1):** o Administrador seleciona uma máquina e adiciona componentes vinculados (monitor, teclado, mouse).
- **FA2 – Editar (passo 1):** altera dados; a mudança de categoria troca o ícone.
- **FA3 – Excluir (passo 1):** Confirmar exclusão avisa sobre periféricos vinculados e chamados abertos; após confirmar, soft delete preservando logs e chamados (RN02).
- **FE1 – Campos obrigatórios vazios (passo 2):** o sistema destaca os campos.

## UC15 – Abrir chamado (→ Amarelo)

- **Ator:** Administrador
- **Requisitos:** RF14, RF18, RF20
- **Pré-condição:** equipamento em Vermelho, ou solicitação vinda do UC12.
- **Pós-condição:** chamado ABERTO, equipamento em Amarelo, solicitante notificado.

1. O Administrador abre o equipamento ou a solicitação na fila (UC16).
2. Escolhe "Abrir chamado".
3. Informa descrição técnica, prioridade e responsável.
4. O sistema cria o chamado com estado ABERTO.
5. O sistema muda o equipamento para Amarelo e registra o log.
6. O sistema notifica quem reportou ou solicitou.

- **FA1 – Equipamento ainda Verde (passo 1):** o Administrador pode abrir chamado preventivo diretamente; o equipamento vai a Amarelo.
- **FE1 – Já existe chamado aberto para o equipamento (passo 4):** o sistema mostra o chamado existente e não cria outro.

## UC16 – Gerenciar fila de chamados

- **Ator:** Administrador
- **Requisitos:** RF18, RF20
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** chamados com estado atualizado (ASSUMIDO ou RESOLVIDO).

1. O Administrador acessa a fila.
2. O sistema lista solicitações e chamados não fechados, com sala, equipamento, estado, prioridade e idade.
3. O Administrador filtra e ordena.
4. O Administrador seleciona um chamado ABERTO e escolhe "Assumir".
5. O sistema muda para ASSUMIDO, registra o responsável e notifica o solicitante.
6. Concluída a intervenção, o Administrador marca como RESOLVIDO com nota técnica.
7. O sistema registra o log; o equipamento continua Amarelo até o UC17.

- **FA1 – Solicitação de Servidor sem chamado (passo 4):** o Administrador inicia o UC15 a partir dela, ou a descarta com justificativa.
- **FE1 – Chamado assumido por outro Administrador (passo 5):** o sistema informa e recarrega a fila.

## UC17 – Fechar chamado (→ Verde)

- **Ator:** Administrador
- **Requisitos:** RF14, RF19, RF16, RF20
- **Pré-condição:** chamado em RESOLVIDO.
- **Pós-condição:** chamado FECHADO, parecer gravado, equipamento em Verde.
- **Include:** Registrar parecer técnico.

1. O Administrador abre um chamado RESOLVIDO.
2. Escolhe "Fechar chamado".
3. O sistema executa Registrar parecer técnico: o Administrador descreve o que foi feito e o resultado.
4. O sistema muda o chamado para FECHADO e o equipamento para Verde.
5. O sistema registra os logs e notifica os envolvidos.

- **FA1 – Equipamento sem conserto (passo 3):** o parecer registra a inviabilidade; o Administrador pode excluir o equipamento logicamente (UC14) em vez de devolvê-lo a Verde.
- **FE1 – Parecer vazio (passo 3):** o sistema não permite fechar sem parecer.

## UC18 – Gerenciar usuários

- **Ator:** Administrador
- **Requisitos:** RF03, RF05
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** usuários ativados, desativados ou promovidos.
- **Extend:** Aprovar solicitação de Administrador.

1. O Administrador acessa o painel de usuários.
2. O sistema lista usuários com nome, e-mail, perfil, status e data de cadastro.
3. O Administrador pesquisa ou filtra.
4. O Administrador ativa ou desativa uma conta.
5. O sistema grava a alteração e o log; uma conta desativada perde a sessão ativa.

- **FA1 – Aprovar solicitação de Administrador (passo 3, extend):** o Administrador abre a aba de pendências, aprova ou recusa com justificativa; na aprovação o perfil passa a Administrador e o solicitante é notificado.
- **FE1 – Desativar a própria conta ou o último Administrador (passo 4):** o sistema bloqueia a ação.

## UC19 – Editar equipamentos em lote

- **Ator:** Administrador
- **Requisitos:** RF24, RF16
- **Pré-condição:** usuário autenticado como Administrador; sala com dois ou mais equipamentos.
- **Pós-condição:** operação aplicada a todos os itens selecionados, com um log por item.

1. Na planta, o Administrador ativa a seleção múltipla.
2. Seleciona os equipamentos por clique ou área.
3. Escolhe a operação: mover para outra sala, alterar categoria, editar observação ou excluir.
4. O sistema mostra o resumo e pede confirmação.
5. O sistema aplica a operação em uma única transação e registra o log de cada item.

- **FA1 – Exclusão em lote (passo 3):** segue a confirmação e o soft delete do UC14 para todos os itens.
- **FE1 – Falha em um item (passo 5):** a transação é desfeita por inteiro e o sistema mostra qual item impediu a operação.