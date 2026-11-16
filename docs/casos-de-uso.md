# Casos de Uso – Sistema de Visualização Espacial de Laboratórios (IFPB)

> Requisitos (RF/RNF), regras de negócio (RN) e transições de status (T1–T9) citados aqui estão definidos em [spec.md](spec.md). A matriz de rastreabilidade está no fim deste documento.

## UC01 – Cadastrar-se

- **Ator:** Visitante
- **Requisitos:** RF01, RF02, RF03, RF04, RF21, RNF04
- **Pré-condição:** nenhuma.
- **Pós-condição:** conta criada como não confirmada, com perfil atribuído pelo domínio, e e-mail de confirmação enfileirado.
- **Include:** Validar domínio institucional. **Extend:** Solicitar perfil de Administrador.

1. O Visitante acessa a tela de cadastro.
2. O sistema exibe o seletor de domínio (@academico.ifpb.edu.br ou @ifpb.edu.br).
3. O Visitante seleciona o domínio e informa nome, prefixo do e-mail e senha.
4. O sistema executa Validar domínio institucional: concatena prefixo e domínio, verifica o formato e confere a senha com a política (RNF04).
5. O sistema atribui o perfil: Estudante para @academico, Servidor para @ifpb (RF02).
6. O sistema cria a conta não confirmada e enfileira o e-mail de confirmação (RF04, RF21).
7. O sistema informa que o link de confirmação foi enviado.

- **FA1 – Solicitar perfil de Administrador (passo 3, só @ifpb.edu.br):** o Visitante marca "Solicitar perfil de Administrador"; no passo 6 o sistema registra também a solicitação, que só entra na fila dos Administradores quando a conta for confirmada (UC02, RF03). A conta funciona como Servidor até a decisão no UC18.
- **FE1 – E-mail fora do padrão (passo 4):** o Visitante digitou um e-mail completo ou um `@`. O sistema recusa com HTTP 400 e mensagem explicando que só o prefixo deve ser informado.
- **FE2 – E-mail já cadastrado (passo 4):** o sistema segue para o passo 7 com a mesma mensagem de sucesso, sem criar conta. Se a conta existente já foi confirmada, enfileira para o dono do e-mail um aviso da tentativa com o link de recuperação de senha; se não foi, reenvia o link de confirmação (RF01). Assim não revela quais e-mails existem.
- **FE3 – Senha fora da política (passo 4):** o sistema mostra os requisitos mínimos de senha (RNF04) e mantém os demais campos preenchidos.

## UC02 – Confirmar conta

- **Atores:** Visitante; Serviço de E-mail (secundário)
- **Requisitos:** RF03, RF04, RF20, RF21
- **Pré-condição:** conta criada pelo UC01 e ainda não confirmada.
- **Pós-condição:** conta confirmada; o usuário pode fazer login.

1. O Serviço de E-mail entrega a mensagem com o link contendo o token.
2. O Visitante clica no link.
3. O sistema valida o token: existe, não foi usado e está dentro das 24 horas de validade (RF04).
4. O sistema confirma a conta e invalida o token. Se houver solicitação de perfil Administrador feita no cadastro, ela entra na fila como PENDENTE e os Administradores são notificados (RF03, RF20).
5. O sistema redireciona para o login com mensagem de sucesso.

- **FE1 – Token expirado (passo 3):** o sistema informa a expiração e oferece reenviar o link, voltando ao passo 1.
- **FE2 – Token inválido ou já usado (passo 3):** o sistema mostra erro e não altera a conta.

## UC03 – Fazer login

- **Ator:** Visitante
- **Requisitos:** RF02, RNF03, RNF04
- **Pré-condição:** conta confirmada e ativa.
- **Pós-condição:** sessão autenticada com o perfil do usuário.

1. O Visitante informa e-mail e senha.
2. O sistema valida as credenciais.
3. O sistema cria a sessão com o perfil (Estudante, Servidor ou Administrador).
4. O sistema redireciona para o dashboard (UC05).

- **FE1 – Credenciais inválidas (passo 2):** mensagem genérica, sem revelar se o e-mail existe.
- **FE2 – Conta não confirmada (passo 2, senha correta):** o sistema informa e oferece reenviar a confirmação (UC02). Com senha errada vale o FE1.
- **FE3 – Conta desativada pelo Administrador (passo 2, senha correta):** o sistema bloqueia o acesso e orienta procurar a coordenação.
- **FE4 – Muitas tentativas (passo 2):** depois de 5 falhas em 15 minutos, o sistema recusa novas tentativas por 15 minutos com mensagem genérica (RNF04).

## UC04 – Recuperar senha

- **Atores:** Visitante; Serviço de E-mail (secundário)
- **Requisitos:** RF04, RF21, RNF04
- **Pré-condição:** nenhuma.
- **Pós-condição:** senha redefinida, token invalidado e sessões anteriores encerradas.

1. O Visitante clica em "Esqueci minha senha" e informa o e-mail.
2. O sistema mostra sempre a mesma mensagem de envio, exista ou não a conta.
3. Se a conta existe, o sistema gera um token válido por 1 hora e o Serviço de E-mail entrega o link (RF21).
4. O Visitante abre o link e informa a nova senha duas vezes.
5. O sistema valida o token e a política de senha (RNF04), grava a nova senha, invalida o token e encerra as sessões abertas da conta.
6. O sistema redireciona para o login.

- **FE1 – Token expirado ou inválido (passo 5):** o sistema informa e oferece recomeçar no passo 1.
- **FE2 – Senhas não coincidem ou fora da política (passo 5):** o sistema pede nova digitação.

## UC05 – Visualizar dashboard de laboratórios

- **Ator:** Estudante (e, por herança, Servidor e Administrador)
- **Requisitos:** RF22, RNF05
- **Pré-condição:** usuário autenticado.
- **Pós-condição:** nenhuma alteração de dados.

1. O usuário acessa a página inicial.
2. O sistema lista um card por sala ativa, com nome, bloco e contagem de equipamentos por status (cor, ícone e texto).
3. O usuário pode filtrar por bloco ou por status.
4. O usuário clica em um card e o sistema inicia o UC06.

- **FA1 – Nenhuma sala cadastrada (passo 2):** o sistema mostra estado vazio; para o Administrador, exibe atalho para Manter salas (UC13).

## UC06 – Visualizar planta da sala

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF08, RF17, RF23, RNF02, RNF05, RNF08
- **Pré-condição:** usuário autenticado; sala ativa.
- **Pós-condição:** nenhuma alteração de dados.
- **Extend:** Consultar histórico do equipamento.

1. O sistema desenha a planta na proporção real da largura e comprimento da sala (RNF02).
2. O sistema posiciona os ícones por categoria, com cor, ícone e rótulo do status atual (RNF05).
3. O usuário aplica zoom e pan.
4. O usuário passa o cursor, toca ou foca pelo teclado um ícone.
5. O sistema exibe popover com nome, categoria, periféricos, status e última ocorrência.
6. O popover mostra o histórico e apenas as ações permitidas ao perfil (matriz 1.3 do spec): registrar ocorrência, reportar falha, mover, editar observação, solicitar chamado, abrir chamado, desfazer falso alarme, editar e excluir.

- **FA1 – Consultar histórico (passo 5, extend):** o usuário clica em "Histórico"; o sistema lista mudanças de status, ocorrências, solicitações, chamados e movimentações do equipamento em ordem cronológica, inclusive em salas anteriores (RF17).
- **FA2 – Tela pequena (passo 1):** o sistema ajusta a escala à tela mantendo a proporção.
- **FA3 – Navegação por teclado (passo 4):** Tab percorre os ícones; Enter abre o popover; as setas e as teclas + e − fazem pan e zoom (RNF05).

## UC07 – Visualizar notificações

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF20
- **Pré-condição:** usuário autenticado.
- **Pós-condição:** notificações abertas marcadas como lidas.

1. O sistema mostra no cabeçalho o contador de notificações não lidas.
2. O usuário abre a lista.
3. O sistema exibe os avisos destinados ao usuário conforme a tabela da RF20.
4. O usuário clica em um aviso e o sistema abre o equipamento, a sala, a solicitação ou o chamado correspondente.
5. O sistema marca o aviso como lido.

- **FA1 – Marcar todas como lidas (passo 3):** o usuário zera o contador de uma vez.
- **FA2 – Item excluído (passo 4):** o sistema informa que o item foi excluído e marca o aviso como lido.

## UC08 – Registrar ocorrência

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF15, RF12, RF16, RN05
- **Pré-condição:** usuário autenticado; equipamento ativo, em qualquer status.
- **Pós-condição:** ocorrência gravada, vinculada ao equipamento e visível a todos; status não muda.
- **Extend:** Anexar foto.

1. No popover do equipamento (UC06), o usuário escolhe "Registrar ocorrência".
2. O usuário descreve a situação em texto livre e, se for o caso, escolhe o periférico afetado entre os vinculados ao equipamento (RF15).
3. O usuário confirma.
4. O sistema grava a ocorrência com autor e data, registra o log (RN01) e atualiza o popover sem recarregar a página.

- **FA1 – Anexar foto (passo 2, extend):** o usuário envia uma imagem; o sistema aplica a RN05 e mostra a miniatura.
- **FE1 – Texto vazio ou acima de 2.000 caracteres (passo 3):** o sistema exige a correção.
- **FE2 – Imagem inválida (FA1):** HTTP 413 ou 400 com o limite permitido; o texto digitado é mantido (RF12).

## UC09 – Reportar falha (Verde → Vermelho)

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF14, RF15, RF16, RF20, RF21
- **Pré-condição:** usuário autenticado; equipamento ativo.
- **Pós-condição:** equipamento em Vermelho (T1), ocorrência e log gravados, Administradores notificados.
- **Extend:** Anexar foto.

1. No popover do equipamento, o usuário escolhe "Reportar falha".
2. O sistema abre o formulário de descrição.
3. O usuário descreve o problema, escolhe o periférico afetado se for o caso, e confirma.
4. O sistema verifica a permissão e o status atual (RN03).
5. O sistema muda o status para Vermelho (T1), grava a ocorrência e o log (RN01).
6. O sistema enfileira a notificação aos Administradores (RF20, RF21).
7. O ícone na planta passa a Vermelho.

- **FA1 – Anexar foto (passo 3, extend):** igual ao FA1 do UC08.
- **FA2 – Equipamento já Vermelho ou Amarelo (passo 4):** o sistema não muda o status e grava a descrição como uma ocorrência adicional (UC08).
- **FE1 – Conflito de edição (passo 5):** outro usuário mudou o status no mesmo instante; o sistema responde HTTP 409, recarrega o estado e segue o FA2 se o equipamento já não estiver Verde (RN03).

## UC10 – Movimentar equipamento na planta

- **Ator:** Servidor (e Administrador por herança)
- **Requisitos:** RF09, RF10, RF16, RF17
- **Pré-condição:** usuário autenticado como Servidor ou Administrador; sala e equipamento ativos.
- **Pós-condição:** nova posição (e sala, se mudou) gravada; histórico, ocorrências e chamado acompanham o equipamento.

1. Na planta (UC06), o usuário ativa o modo de edição de layout.
2. O usuário arrasta o ícone para a nova posição.
3. O sistema verifica se a posição está dentro dos limites da sala.
4. O sistema grava as coordenadas em metros e o log (RN01).
5. O sistema confirma sem recarregar a página.

- **FA1 – Mover para outra sala (passo 2):** o usuário escolhe "Mover para…" e seleciona uma sala ativa; o sistema propõe uma posição livre na nova planta, que o usuário pode ajustar; o equipamento vai com seus periféricos, e o histórico e o chamado não fechado seguem com ele (RF10, RF17).
- **FA2 – Transferir periférico (passo 2):** o usuário desvincula um periférico de um equipamento e o vincula a outro; o periférico passa para a sala do novo equipamento (RF10).
- **FA3 – Mover sem arrastar (passo 2):** o usuário move o ícone pelas setas do teclado ou informa as coordenadas num formulário (RF09, RNF05).
- **FE1 – Posição fora da sala ou sobre outro ícone (passo 3):** o ícone volta à posição anterior com aviso.
- **FE2 – Estudante tenta mover (passo 1):** o modo de edição não aparece; uma requisição direta recebe HTTP 403 (RN03).
- **FE3 – Sala de destino no limite de equipamentos (FA1):** a sala não aparece como destino (RNF07).

## UC11 – Editar observações da sala

- **Ator:** Servidor (e Administrador)
- **Requisitos:** RF13, RF16
- **Pré-condição:** usuário autenticado como Servidor ou Administrador; sala ativa.
- **Pós-condição:** observação atualizada e versão anterior registrada no log (RN01).

1. Na página da sala, o usuário clica em "Editar observações".
2. O sistema abre o editor com formatação básica (negrito, itálico, listas, links).
3. O usuário edita e salva.
4. O sistema sanitiza o conteúdo, grava e registra o log com a versão anterior.

- **FA1 – Observação de equipamento (passo 1):** o mesmo fluxo vale para a observação de um equipamento, a partir do popover.
- **FE1 – Conteúdo com script ou marcação não permitida (passo 4):** o sistema remove o que não é permitido antes de gravar (RF13).
- **FE2 – Texto acima de 5.000 caracteres (passo 3):** o sistema pede para reduzir.

## UC12 – Solicitar chamado

- **Ator:** Servidor
- **Requisitos:** RF25, RF12, RF18, RF20
- **Pré-condição:** usuário autenticado como Servidor; equipamento ou sala ativos.
- **Pós-condição:** solicitação PENDENTE na fila do Administrador; o status do equipamento não muda.

1. No popover do equipamento ou na página da sala, o Servidor escolhe "Solicitar chamado".
2. O Servidor descreve o problema e pode anexar foto (RN05).
3. O sistema verifica a RN06 e grava a solicitação com estado PENDENTE.
4. O sistema enfileira a notificação aos Administradores (RF20).
5. O sistema informa que o Administrador fará a triagem; a solicitação aparece na página do item com o estado atual.

- **FA1 – Problema estrutural da sala (passo 1):** a solicitação é vinculada à sala, não a um equipamento.
- **FE1 – Já existe solicitação pendente ou chamado não fechado para o item (passo 3):** o sistema mostra o existente e oferece registrar uma ocorrência (UC08, RN06).
- **FE2 – Estudante tenta solicitar (passo 1):** a opção não aparece; uma requisição direta recebe HTTP 403 (RN03).

## UC13 – Manter salas

- **Ator:** Administrador
- **Requisitos:** RF06, RF11, RF16, RN02, RNF02
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** sala criada, alterada ou excluída logicamente.
- **Include:** Confirmar exclusão.

1. O Administrador escolhe "Nova sala".
2. Informa nome, bloco, largura e comprimento em metros e observações.
3. O sistema valida as dimensões e a unicidade do nome no bloco (RF06).
4. O sistema grava a sala, registra o log e gera a planta vazia na proporção informada.

- **FA1 – Editar (passo 1):** o Administrador altera dados ou dimensões; o sistema recalcula a escala e mantém as posições dos equipamentos em metros.
- **FA2 – Excluir (passo 1):** o sistema executa Confirmar exclusão, mostrando os equipamentos e periféricos da sala, as solicitações que serão descartadas e os chamados que serão cancelados; após confirmar, aplica a exclusão lógica em cascata (RN02, T8).
- **FE1 – Dimensões inválidas (passo 3):** o sistema rejeita zero, negativos, texto ou valores acima de 100 m.
- **FE2 – Redução que deixa equipamentos fora (FA1):** o sistema lista os itens afetados e exige reposicioná-los antes de salvar.
- **FE3 – Nome repetido no bloco (passo 3):** o sistema pede outro nome.

## UC14 – Manter equipamentos e periféricos

- **Ator:** Administrador
- **Requisitos:** RF07, RF08, RF11, RF12, RF16, RF28, RN02, RNF01
- **Pré-condição:** usuário autenticado como Administrador; sala ativa.
- **Pós-condição:** equipamento ou periférico criado, alterado ou excluído logicamente; histórico intacto.
- **Include:** Confirmar exclusão.

1. Na planta, o Administrador escolhe "Novo equipamento".
2. Informa nome, categoria (entre as cadastradas no UC23), descrição, observações e, opcionalmente, foto (RN05).
3. O sistema associa o ícone pela categoria (RF08).
4. O Administrador posiciona o ícone na planta.
5. O sistema grava o equipamento em Verde e registra o log (RN01).

- **FA1 – Cadastrar periférico (passo 1):** o Administrador seleciona um equipamento e adiciona periféricos vinculados (monitor, teclado, mouse), usando categorias do tipo periférico.
- **FA2 – Editar (passo 1):** altera dados; a mudança de categoria troca o ícone.
- **FA3 – Excluir (passo 1):** Confirmar exclusão mostra os periféricos vinculados, a solicitação que será descartada e o chamado que será cancelado; após confirmar, cancela o chamado (o equipamento volta ao status anterior a ele), descarta a solicitação e aplica a exclusão lógica, preservando logs, ocorrências e chamados (RN02, T8).
- **FE1 – Campos obrigatórios vazios (passo 2):** o sistema destaca os campos.
- **FE2 – Sala no limite de equipamentos (passo 1):** com 200 equipamentos, o sistema não permite cadastrar outro (RNF07).

## UC15 – Abrir chamado (→ Amarelo)

- **Ator:** Administrador
- **Requisitos:** RF14, RF18, RF20, RF32
- **Pré-condição:** usuário autenticado como Administrador; equipamento ou sala ativos, sem chamado não fechado (RN06). O equipamento pode estar Verde (chamado preventivo) ou Vermelho, e pode haver uma solicitação PENDENTE (UC12).
- **Pós-condição:** chamado ABERTO; equipamento em Amarelo (T3 ou T4); solicitação, se houver, CONVERTIDA; envolvidos notificados.

1. O Administrador abre o equipamento pelo popover ou um item da fila (UC16).
2. Escolhe "Abrir chamado".
3. Informa descrição técnica, prioridade (Baixa, Média ou Alta) e responsável entre os Administradores ativos.
4. O sistema cria o chamado com estado ABERTO; se o item tem solicitação PENDENTE, ela passa a CONVERTIDA, venha o Administrador da fila ou do popover.
5. O sistema muda o equipamento para Amarelo, guarda o status anterior (para um eventual cancelamento) e registra o log.
6. O sistema notifica quem reportou a falha ou fez a solicitação e o responsável (RF20).

- **FA1 – Chamado preventivo (passo 1):** com o equipamento Verde, o fluxo é o mesmo e a transição é a T4.
- **FA2 – Problema estrutural da sala (passo 1):** o chamado é vinculado à sala e o passo 5 não muda o status de nenhum equipamento.
- **FE1 – Já existe chamado não fechado para o item (passo 4):** o sistema mostra o chamado existente e não cria outro (RN06).
- **FE2 – Estado mudou no mesmo instante (passo 4):** HTTP 409; o sistema recarrega o item (RN03).

## UC16 – Gerenciar fila do Administrador

- **Ator:** Administrador
- **Requisitos:** RF18, RF20, RF25, RF32
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** solicitações e chamados com estado atualizado.

1. O Administrador acessa a fila.
2. O sistema lista três grupos: solicitações PENDENTES, equipamentos em Vermelho e chamados não fechados, com sala, item, estado, prioridade e idade.
3. O Administrador filtra e ordena.
4. O Administrador seleciona um chamado ABERTO ou REABERTO e escolhe "Assumir".
5. O sistema muda para ASSUMIDO; se quem assumiu não era o responsável, passa a ser; o sistema notifica os envolvidos.
6. Concluída a intervenção, o responsável marca como RESOLVIDO com nota técnica.
7. O sistema registra o log e notifica; o equipamento continua Amarelo até a validação no UC17.

- **FA1 – Triagem de solicitação (passo 4):** o Administrador abre um chamado a partir dela (UC15), ou a descarta com justificativa; a solicitação passa a DESCARTADA, o solicitante é notificado e o status do equipamento não muda.
- **FA2 – Equipamento em Vermelho (passo 4):** o Administrador abre um chamado (UC15) ou, se foi engano, desfaz o falso alarme (UC21).
- **FA3 – Cancelar chamado (passo 4):** para um chamado ABERTO, ASSUMIDO ou REABERTO, o Administrador informa a justificativa; o chamado passa a CANCELADO e o equipamento volta ao status anterior à abertura (T7); os envolvidos são notificados.
- **FE1 – Chamado assumido por outro Administrador (passo 5):** HTTP 409; o sistema informa e recarrega a fila (RN03).
- **FE2 – Nota técnica vazia (passo 6):** o sistema não permite marcar como RESOLVIDO.
- **FE3 – Quem tenta resolver não é o responsável (passo 6):** a opção não aparece; uma requisição direta recebe HTTP 403 (RF32).

## UC17 – Validar e fechar chamado

- **Ator:** Administrador
- **Requisitos:** RF14, RF16, RF19, RF20, RF32
- **Pré-condição:** chamado em RESOLVIDO.
- **Pós-condição:** chamado FECHADO com parecer gravado e equipamento em Verde ou Vermelho; ou chamado REABERTO.
- **Include:** Registrar parecer técnico.

1. O Administrador abre um chamado RESOLVIDO e confere o equipamento no laboratório.
2. Escolhe "Fechar chamado".
3. O sistema executa Registrar parecer técnico: o Administrador descreve o que foi feito e escolhe o resultado "consertado".
4. O sistema muda o chamado para FECHADO e o equipamento para Verde (T5).
5. O sistema registra os logs e notifica os envolvidos.

- **FA1 – Equipamento sem conserto (passo 3):** o resultado é "sem conserto"; o chamado fecha e o equipamento volta a Vermelho (T6). Se não houver como recuperá-lo, o Administrador o exclui logicamente (UC14).
- **FA2 – Problema persiste (passo 2):** o Administrador escolhe "Reabrir" e informa o motivo; o chamado passa a REABERTO, volta à fila (UC16) e o equipamento continua Amarelo.
- **FA3 – Chamado de sala (passo 4):** o chamado fecha sem mudar o status de nenhum equipamento.
- **FE1 – Parecer vazio ou com menos de 10 caracteres (passo 3):** o sistema não permite fechar.

## UC18 – Gerenciar usuários

- **Ator:** Administrador
- **Requisitos:** RF03, RF05, RF16, RF20, RNF06
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** usuários ativados, desativados, promovidos, rebaixados ou anonimizados.
- **Extend:** Decidir solicitação de Administrador.

1. O Administrador acessa o painel de usuários.
2. O sistema lista usuários com nome, e-mail, perfil, situação e data de cadastro.
3. O Administrador pesquisa ou filtra.
4. O Administrador ativa ou desativa uma conta.
5. O sistema grava a alteração e o log; uma conta desativada perde as sessões ativas, e uma solicitação de Administrador pendente dela é recusada automaticamente.

- **FA1 – Decidir solicitação de Administrador (passo 3, extend):** o Administrador abre a aba de pendências e aprova ou recusa com justificativa; na aprovação o perfil passa a Administrador; o solicitante é avisado por e-mail e dentro do sistema (RN04).
- **FA2 – Rebaixar Administrador (passo 4):** o Administrador muda o perfil de outro Administrador para Servidor.
- **FA3 – Anonimizar conta (passo 4, a pedido do titular):** para uma conta desativada, o Administrador confirma a anonimização; nome e e-mail são substituídos e a autoria passa a aparecer como "Usuário removido". A ação não pode ser desfeita (RF05, RNF06).
- **FE1 – Desativar ou rebaixar a si mesmo ou o último Administrador (passo 4):** o sistema bloqueia a ação (RN07).
- **FE2 – Administrador responsável por chamados não fechados (passo 4 ou FA2):** o sistema lista os chamados e exige escolher um novo responsável entre os Administradores ativos antes de concluir (RF05).

## UC19 – Editar equipamentos em lote

- **Ator:** Administrador
- **Requisitos:** RF10, RF11, RF16, RF24
- **Pré-condição:** usuário autenticado como Administrador; sala com dois ou mais equipamentos ativos.
- **Pós-condição:** operação aplicada a todos os itens selecionados, com um log por item.

1. Na planta, o Administrador ativa a seleção múltipla.
2. Seleciona os equipamentos por clique ou por área.
3. Escolhe a operação: mover para outra sala, alterar categoria, editar observação ou excluir.
4. O sistema mostra o resumo e pede confirmação.
5. O sistema aplica a operação em uma única transação e registra o log de cada item.

- **FA1 – Exclusão em lote (passo 3):** o resumo segue o Confirmar exclusão do UC14 para todos os itens, inclusive periféricos, solicitações e chamados afetados (RN02).
- **FA2 – Mover em lote (passo 3):** os equipamentos vão para posições livres da sala de destino, com as regras da RF10.
- **FE1 – Falha em um item (passo 5):** a transação é desfeita por inteiro e o sistema mostra qual item impediu a operação.

## UC20 – Gerenciar a própria conta

- **Ator:** Estudante (e herdeiros)
- **Requisitos:** RF03, RF26, RF27, RNF04
- **Pré-condição:** usuário autenticado.
- **Pós-condição:** senha alterada, solicitação de Administrador criada ou sessão encerrada.

1. O usuário abre "Minha conta" e escolhe "Alterar senha".
2. Informa a senha atual e a nova senha duas vezes.
3. O sistema confere a senha atual e a política (RNF04).
4. O sistema grava a nova senha, encerra as outras sessões do usuário e confirma.

- **FA1 – Solicitar perfil de Administrador (passo 1, só Servidor):** o Servidor escolhe "Solicitar perfil de Administrador"; o sistema cria a solicitação PENDENTE e notifica os Administradores (RF03).
- **FA2 – Encerrar sessão (passo 1):** o usuário clica em "Sair"; o sistema invalida a sessão e redireciona ao login (RF26).
- **FE1 – Senha atual incorreta (passo 3):** o sistema recusa sem alterar nada.
- **FE2 – Nova senha fora da política ou não coincide (passo 3):** o sistema mostra os requisitos.
- **FE3 – Solicitação já pendente ou usuário Estudante (FA1):** a opção não aparece; uma requisição direta recebe HTTP 409 ou 403 (RN03).

## UC21 – Desfazer falso alarme (Vermelho → Verde)

- **Ator:** Administrador
- **Requisitos:** RF14, RF16, RF20, RF31
- **Pré-condição:** usuário autenticado como Administrador; equipamento em Vermelho (portanto sem chamado não fechado).
- **Pós-condição:** equipamento Verde (T2), justificativa no log, quem reportou notificado. Uma solicitação PENDENTE do equipamento continua na fila (RF31).

1. Pelo popover ou pela fila (UC16 FA2), o Administrador escolhe "Marcar como falso alarme".
2. Informa a justificativa.
3. O sistema verifica permissão e estado (RN03).
4. O sistema muda o equipamento para Verde (T2) e registra o log com a justificativa.
5. O sistema notifica quem reportou a falha (RF20).

- **FE1 – Justificativa vazia (passo 2):** o sistema não permite confirmar.
- **FE2 – Equipamento já não está Vermelho (passo 3):** outro Administrador abriu um chamado ou desfez o alarme no mesmo instante; HTTP 409 e o sistema recarrega o item (RN03).

## UC22 – Consultar e restaurar itens excluídos

- **Ator:** Administrador
- **Requisitos:** RF17, RF29, RF30, RN02
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** item restaurado às telas operacionais, ou apenas consultado.

1. O Administrador acessa "Itens excluídos".
2. O sistema lista salas, equipamentos, periféricos, categorias e ocorrências ocultas, com quem excluiu e quando.
3. O Administrador pesquisa por nome ou filtra por tipo.
4. O Administrador abre um item; o sistema mostra o histórico completo (RF17).
5. O Administrador escolhe "Restaurar".
6. O sistema mostra o que será restaurado junto (itens excluídos em cascata com uma sala) e o status com que o equipamento voltará (T9).
7. O Administrador confirma; o sistema restaura e registra o log.

- **FA1 – Apenas consultar (passo 5):** o Administrador encerra sem restaurar.
- **FE1 – Sala do equipamento está excluída (passo 6):** o sistema exige restaurar a sala antes ou escolher outra sala ativa.
- **FE2 – Categoria do item está excluída (passo 6):** o sistema exige restaurar a categoria ou escolher outra.
- **FE3 – Posição antiga ocupada ou fora da sala (passo 7):** o item recebe uma posição livre e o sistema avisa.
- **FE4 – Equipamento do periférico está excluído (passo 6):** o sistema exige restaurar o equipamento antes ou vincular o periférico a outro equipamento ativo.
- **FE5 – Sala no limite de equipamentos (passo 6):** o sistema não restaura e informa o limite (RNF07).

## UC23 – Manter categorias

- **Ator:** Administrador
- **Requisitos:** RF08, RF16, RF28, RN02
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** categoria criada, alterada ou excluída logicamente.

1. O Administrador acessa "Categorias" e escolhe "Nova categoria".
2. Informa nome, tipo (equipamento ou periférico) e escolhe um ícone na biblioteca do sistema.
3. O sistema valida que o nome é único entre as categorias ativas.
4. O sistema grava a categoria e registra o log.

- **FA1 – Editar (passo 1):** a troca de ícone vale para todos os itens da categoria (RF08).
- **FA2 – Excluir (passo 1):** só é permitido se nenhum item ativo usa a categoria; a exclusão é lógica (RN02).
- **FE1 – Nome repetido (passo 3):** o sistema pede outro nome.
- **FE2 – Categoria em uso (FA2):** o sistema lista os itens que a usam e não exclui.

## UC24 – Consultar log de auditoria

- **Ator:** Administrador
- **Requisitos:** RF16
- **Pré-condição:** usuário autenticado como Administrador.
- **Pós-condição:** nenhuma alteração de dados.

1. O Administrador acessa "Auditoria".
2. O sistema lista os registros do log do mais recente para o mais antigo, com data e hora, autor, entidade, ação, valor anterior e valor novo.
3. O Administrador filtra por entidade, autor, ação ou período.
4. O Administrador abre um registro e o sistema leva ao item correspondente, ativo ou excluído.

- **FA1 – Nenhum registro no filtro (passo 3):** o sistema mostra estado vazio.
- **FE1 – Outro perfil tenta acessar (passo 1):** HTTP 403 (RN03).

---

## Matriz de rastreabilidade (RF → UC)

| RF | Casos de uso |
| :--- | :--- |
| RF01 | UC01 |
| RF02 | UC01, UC03 |
| RF03 | UC01, UC02, UC18, UC20 |
| RF04 | UC01, UC02, UC04 |
| RF05 | UC18 |
| RF06 | UC13 |
| RF07 | UC14 |
| RF08 | UC06, UC14, UC23 |
| RF09 | UC10 |
| RF10 | UC10, UC19 |
| RF11 | UC13, UC14, UC19 |
| RF12 | UC08, UC12, UC14 |
| RF13 | UC11 |
| RF14 | UC09, UC15, UC17, UC21 |
| RF15 | UC08, UC09 |
| RF16 | UC08, UC09, UC10, UC11, UC13, UC14, UC17, UC18, UC19, UC21, UC23, UC24 |
| RF17 | UC06, UC10, UC22 |
| RF18 | UC12, UC15, UC16 |
| RF19 | UC17 |
| RF20 | UC02, UC07, UC09, UC12, UC15, UC16, UC17, UC18, UC21 |
| RF21 | UC01, UC02, UC04, UC09 |
| RF22 | UC05 |
| RF23 | UC06 |
| RF24 | UC19 |
| RF25 | UC12, UC16 |
| RF26 | UC20 |
| RF27 | UC20 |
| RF28 | UC14, UC23 |
| RF29 | UC22 |
| RF30 | UC22 |
| RF31 | UC21 |
| RF32 | UC15, UC16, UC17 |
