# Spec – Sistema de Visualização Espacial de Laboratórios (IFPB)

> **O quê** o sistema faz. Para o *porquê*, ver [intent.md](intent.md); para o *como* (stack, banco, bibliotecas), ver [plan.md](plan.md). Casos de uso detalhados em [casos-de-uso.md](casos-de-uso.md).
>
> **Identificadores estáveis:** RF, RNF, RN e T nunca são renumerados. Um requisito novo recebe o próximo número livre, mesmo que fique numa seção anterior; um requisito removido deixa o número vago.

## 1. PERFIS E MATRIZ DE ACESSO (RBAC)

### 1.1 Perfis de usuário

- **ESTUDANTE:** usuário com e-mail `@academico.ifpb.edu.br`. Visualiza laboratórios, registra ocorrências e reporta falhas.
- **SERVIDOR:** usuário com e-mail `@ifpb.edu.br` (perfil padrão desse domínio). Além do que o Estudante faz, movimenta equipamentos e periféricos, edita observações e solicita chamados técnicos.
- **ADMINISTRADOR:** gestão completa: salas, equipamentos, categorias, chamados, itens excluídos e usuários.

### 1.2 Como se obtém cada perfil

| Perfil | Como o usuário chega a ele |
| :--- | :--- |
| Estudante | Automaticamente, ao se cadastrar com `@academico.ifpb.edu.br` (RF02). |
| Servidor | Automaticamente, ao se cadastrar com `@ifpb.edu.br` (RF02). |
| Administrador | Um Servidor envia uma **solicitação de perfil Administrador** (RF03), no cadastro ou depois, e um Administrador a aprova (RF05). O **primeiro** Administrador é criado na implantação do sistema (ver [plan.md](plan.md)). Um Estudante não pode ser Administrador. |

Um Administrador pode ser rebaixado a Servidor por outro Administrador, respeitando a RN07.

### 1.3 Matriz de permissões

**Legenda:** `X` permitido · vazio = não permitido (requisição direta recebe HTTP 403, RN03).

| Funcionalidade / ação | Estudante | Servidor | Administrador |
| :--- | :---: | :---: | :---: |
| Visualizar dashboard, plantas, status e popovers | X | X | X |
| Consultar histórico de equipamento ativo | X | X | X |
| Ver e marcar como lidas as próprias notificações | X | X | X |
| Alterar a própria senha e encerrar a sessão | X | X | X |
| Registrar ocorrência (texto + foto) | X | X | X |
| Reportar falha (Verde → Vermelho) | X | X | X |
| Mover equipamentos e periféricos (na sala e entre salas) |  | X | X |
| Editar observações da sala |  | X | X |
| Editar observações do equipamento |  | X | X |
| Solicitar chamado técnico |  | X |  |
| Solicitar perfil de Administrador |  | X |  |
| Criar, editar e excluir salas |  |  | X |
| Criar, editar e excluir equipamentos e periféricos |  |  | X |
| Manter categorias e ícones |  |  | X |
| Editar equipamentos em lote |  |  | X |
| Acessar a fila (solicitações, falhas sem chamado e chamados) |  |  | X |
| Abrir, assumir, resolver, reabrir e cancelar chamados |  |  | X |
| Fechar chamado formalmente |  |  | X |
| Desfazer falso alarme (Vermelho → Verde) |  |  | X |
| Descartar solicitação de chamado |  |  | X |
| Ocultar ocorrência imprópria |  |  | X |
| Consultar e restaurar itens excluídos |  |  | X |
| Consultar o log de auditoria |  |  | X |
| Gerenciar usuários e decidir solicitações de Administrador |  |  | X |

O Administrador não "solicita" chamado: ele abre o chamado diretamente.

## 2. ESTADOS E CICLOS DE VIDA

O sistema separa quatro ciclos: o **status do equipamento** (físico, visível na planta), a **solicitação de chamado** (pedido de um Servidor), o **chamado técnico** (trabalho do Administrador) e a **solicitação de perfil Administrador**.

### 2.1 Status do equipamento

| Status | Significado | Indicação visual (RNF05) |
| :--- | :--- | :--- |
| 🟢 **VERDE – Operacional** | Funcionando normalmente. | Cor verde + ícone de "ok" + rótulo "Operacional" |
| 🟡 **AMARELO – Em manutenção** | Existe um chamado não fechado para o equipamento. | Cor amarela + ícone de ferramenta + rótulo "Em manutenção" |
| 🔴 **VERMELHO – Falha** | Quebrado ou inoperante, sem chamado em andamento. | Cor vermelha + ícone de "×" + rótulo "Falha" |

**Excluído** não é um status: é uma condição (RN02). O equipamento excluído some das telas operacionais e guarda o status que tinha (sempre Verde ou Vermelho, ver T8), para a restauração.

Periféricos **não têm status próprio**: a falha de um periférico é reportada no equipamento a que ele está vinculado, e a ocorrência indica, num campo opcional, qual periférico falhou (RF07, RF15).

**Quem reportou a falha** é o autor da transição T1 mais recente do equipamento. É essa pessoa que recebe os avisos do chamado e do falso alarme (RF20).

### 2.2 Transições do status do equipamento

| ID | De | Para | Quem | Condição | UC |
| :--- | :--- | :--- | :--- | :--- | :--- |
| T1 | Verde | Vermelho | Qualquer perfil | Reportar falha, com descrição obrigatória | UC09 |
| T2 | Vermelho | Verde | Administrador | Falso alarme, com justificativa obrigatória. Uma solicitação PENDENTE do equipamento continua na fila para triagem | UC21 |
| T3 | Vermelho | Amarelo | Administrador | Abrir chamado | UC15 |
| T4 | Verde | Amarelo | Administrador | Abrir chamado preventivo | UC15 |
| T5 | Amarelo | Verde | Administrador | Fechar chamado com resultado "consertado" | UC17 |
| T6 | Amarelo | Vermelho | Administrador | Fechar chamado com resultado "sem conserto" | UC17 |
| T7 | Amarelo | Status anterior à abertura (Verde ou Vermelho) | Administrador | Cancelar chamado | UC16 |
| T8 | Qualquer | Excluído | Administrador | Exclusão lógica. Se o equipamento está Amarelo, o chamado é cancelado antes, automaticamente, e o equipamento volta ao status anterior à abertura (T7); é esse status que fica guardado | UC13, UC14, UC19 |
| T9 | Excluído | Status guardado na exclusão (Verde ou Vermelho) | Administrador | Restaurar | UC22 |

Qualquer outra mudança é recusada (RN03). Não mudam o status: registrar ocorrência, mover, editar observações, solicitar chamado, descartar solicitação, assumir, resolver ou reabrir chamado. Reportar falha num equipamento que já está Vermelho ou Amarelo vira uma ocorrência (UC09 FA2). Um chamado de **sala** (problema estrutural) não muda o status de nenhum equipamento.

### 2.3 Solicitação de chamado (Servidor)

| Estado | Significado |
| :--- | :--- |
| **PENDENTE** | Criada pelo Servidor; aguarda a triagem do Administrador. |
| **CONVERTIDA** | O Administrador abriu um chamado a partir dela; o chamado guarda o vínculo. |
| **DESCARTADA** | O Administrador a recusou com justificativa, ou o item foi excluído. |

| De | Para | Quem | Condição |
| :--- | :--- | :--- | :--- |
| — | PENDENTE | Servidor | Item ativo sem solicitação pendente nem chamado não fechado (RN06) |
| PENDENTE | CONVERTIDA | Administrador | Abrir chamado para o item, a partir da solicitação ou diretamente (UC15) |
| PENDENTE | DESCARTADA | Administrador | Justificativa obrigatória (UC16) |
| PENDENTE | DESCARTADA | Sistema | O equipamento ou a sala foi excluído (RN02) |

A solicitação nunca muda o status do equipamento. Se uma solicitação for descartada e o equipamento estiver Vermelho, ele continua Vermelho até o Administrador abrir um chamado ou desfazer o falso alarme.

### 2.4 Chamado técnico

| Estado | Significado |
| :--- | :--- |
| **ABERTO** | Registrado pelo Administrador, com um responsável designado, mas a tratativa ainda não começou. |
| **ASSUMIDO** | O responsável iniciou a tratativa. Outro Administrador pode assumir no lugar dele e passa a ser o responsável. |
| **RESOLVIDO** | O responsável concluiu a intervenção e registrou a nota técnica. Falta **validar**: um Administrador (o próprio responsável ou outro) confirma, no laboratório, que o equipamento funciona. O estado separa "intervenção feita" de "funcionamento confirmado" (por exemplo, peça trocada aguardando teste). O equipamento continua Amarelo. |
| **REABERTO** | A validação falhou: o problema persiste. Volta à fila como um chamado ABERTO, com o mesmo responsável. |
| **FECHADO** | Encerrado formalmente com parecer técnico. Estado final. |
| **CANCELADO** | Encerrado sem intervenção (aberto por engano, duplicado ou item excluído). Estado final. |

| De | Para | Quem | Condição |
| :--- | :--- | :--- | :--- |
| — | ABERTO | Administrador | RN06; descrição técnica, prioridade e responsável obrigatórios |
| ABERTO ou REABERTO | ASSUMIDO | Administrador | Quem assume passa a ser o responsável |
| ASSUMIDO | RESOLVIDO | Administrador responsável | Nota técnica obrigatória |
| RESOLVIDO | FECHADO | Administrador | Parecer técnico e resultado ("consertado" ou "sem conserto") obrigatórios |
| RESOLVIDO | REABERTO | Administrador | Motivo obrigatório |
| ABERTO, ASSUMIDO ou REABERTO | CANCELADO | Administrador | Justificativa obrigatória |
| ABERTO, ASSUMIDO, RESOLVIDO ou REABERTO | CANCELADO | Sistema | O equipamento ou a sala foi excluído (RN02) |

Um chamado FECHADO ou CANCELADO não volta a ser aberto: um problema novo gera um chamado novo.

### 2.5 Solicitação de perfil Administrador

**PENDENTE** → **APROVADA** (o perfil passa a Administrador) ou **RECUSADA** (justificativa obrigatória). A solicitação pendente de um usuário desativado é recusada automaticamente. A solicitação feita no cadastro só entra na fila, e só gera aviso aos Administradores, depois que a conta é confirmada (RF04).

## 3. REGRAS DE NEGÓCIO (RN)

- **RN01 – Auditoria:** geram um registro imutável no log (RF16):
    - mudança de status do equipamento;
    - criação, edição, movimentação (posição e sala), exclusão e restauração de sala, equipamento, periférico e categoria;
    - edição de observações de sala ou de equipamento, guardando a versão anterior;
    - criação e ocultação de ocorrência;
    - criação e mudança de estado de solicitação de chamado e de chamado;
    - ativação, desativação e mudança de perfil de usuário, e decisão sobre solicitação de perfil Administrador.
- **RN02 – Exclusão lógica:** sala, equipamento, periférico, categoria, ocorrência, solicitação e chamado nunca são apagados fisicamente. Usuários não são excluídos, apenas desativados.
    - O item excluído some das telas operacionais, mas continua consultável e restaurável pelo Administrador (RF29, RF30).
    - Excluir uma sala exclui junto seus equipamentos e periféricos; excluir um equipamento exclui junto seus periféricos.
    - Logs, ocorrências e chamados do item excluído ficam intactos.
    - Solicitações pendentes do item são descartadas e chamados não fechados são cancelados automaticamente, com a justificativa "item excluído"; o equipamento volta ao status anterior ao chamado antes de ser excluído (T8).
    - Um periférico excluído sozinho pode ser restaurado; se o equipamento dele também estiver excluído, é preciso restaurá-lo antes ou vincular o periférico a outro equipamento ativo.
    - Uma categoria só pode ser excluída se nenhum item ativo a usa.
- **RN03 – Autorização e transições validadas no servidor:** toda ação é conferida no servidor contra a matriz (1.3) e as tabelas de transição (2.2 a 2.5).
    - Perfil sem permissão → HTTP 403, sem alteração.
    - Transição inválida para o estado atual, inclusive quando outro usuário mudou o estado no mesmo instante → HTTP 409, sem alteração; a interface recarrega o estado atual.
- **RN04 – Canais de comunicação:** o e-mail é usado **apenas** para assuntos da conta: confirmação de cadastro, recuperação de senha e resultado da solicitação de perfil Administrador. Avisos sobre equipamentos, solicitações e chamados são **apenas dentro do sistema** (RF20). Todo envio é assíncrono (RF21).
- **RN05 – Imagens:** apenas JPG ou PNG, verificados pelo conteúdo do arquivo e não só pela extensão, com até 5 MB. Uma foto por equipamento, por ocorrência e por solicitação. Metadados (EXIF, inclusive localização) são removidos ao gravar.
- **RN06 – Um tratamento por item:** um equipamento, ou uma sala no caso de problema estrutural, tem no máximo **uma** solicitação PENDENTE e **um** chamado não fechado (ABERTO, ASSUMIDO, RESOLVIDO ou REABERTO). Uma nova tentativa mostra o existente e oferece registrar uma ocorrência.
- **RN07 – Continuidade administrativa:** o sistema sempre tem pelo menos um Administrador ativo. Um Administrador não pode desativar nem rebaixar a si mesmo, nem desativar ou rebaixar o último Administrador ativo.

## 4. REQUISITOS FUNCIONAIS (RF) E CRITÉRIOS DE ACEITAÇÃO

### 4.1 Autenticação e contas

- **RF01 – Cadastro por domínio institucional:** o usuário escolhe o domínio (`@academico.ifpb.edu.br` ou `@ifpb.edu.br`) e informa nome, o prefixo do e-mail e a senha.
    - Prefixo com `@` ou e-mail completo → HTTP 400 com a mensagem "informe só a parte antes do @".
    - Senha fora da política (RNF04) → recusada com os requisitos mínimos; os demais campos continuam preenchidos.
    - E-mail já cadastrado → o sistema mostra a mesma mensagem de sucesso de um cadastro novo e não cria outra conta. Se a conta existente já foi confirmada, envia ao dono um e-mail avisando da tentativa, com o link de recuperação de senha; se ainda não foi, reenvia o link de confirmação. Assim o cadastro não revela quais e-mails existem, como o login e a recuperação.
    - A conta nasce **não confirmada** e só permite login depois da confirmação (RF04).
- **RF02 – Classificação automática:** o perfil é definido pelo domínio: `@academico` → Estudante; `@ifpb` → Servidor.
    - O usuário não escolhe o perfil; qualquer campo de perfil enviado na requisição é ignorado.
- **RF03 – Solicitação de perfil Administrador:** um Servidor pode pedir o perfil de Administrador no cadastro ou depois, em "Minha conta".
    - Cria uma solicitação PENDENTE (2.5) e notifica todos os Administradores (RF20). Se a solicitação foi feita no cadastro, isso só acontece depois da confirmação da conta.
    - Estudante → HTTP 403.
    - No máximo uma solicitação pendente por usuário.
    - Enquanto a solicitação estiver pendente, o usuário atua como Servidor.
    - O resultado é comunicado por e-mail e dentro do sistema (RN04).
- **RF04 – Confirmação de conta e recuperação de senha:**
    - O token de confirmação vale **24 horas**; o de recuperação, **1 hora**.
    - Todo token é de uso único; gerar um novo invalida o anterior do mesmo tipo.
    - O reenvio do link é permitido no máximo uma vez por minuto.
    - A recuperação mostra sempre a mesma mensagem, exista ou não a conta.
    - Redefinir a senha encerra todas as sessões abertas da conta.
    - Token expirado → o sistema oferece reenviar; token inválido ou usado → erro, sem alterar a conta.
- **RF05 – Gestão de usuários (Administrador):** listar, pesquisar e filtrar usuários; ativar e desativar contas; rebaixar Administrador a Servidor; aprovar ou recusar solicitações de perfil Administrador; anonimizar conta desativada a pedido do titular (RNF06).
    - A desativação encerra as sessões do usuário: a próxima requisição dele vai para o login.
    - Desativar ou rebaixar um Administrador que é responsável por chamados não fechados exige escolher antes um novo responsável entre os Administradores ativos; os chamados mantêm o estado.
    - A anonimização troca nome e e-mail por valores sem identificação e não pode ser desfeita; o log continua apontando para o mesmo usuário, agora exibido como "Usuário removido".
    - A recusa exige justificativa.
    - A RN07 é sempre respeitada.
    - Toda alteração gera log (RN01).
- **RF26 – Encerrar sessão:** o usuário sai do sistema.
    - A sessão é invalidada no servidor; qualquer requisição seguinte com ela vai para o login.
- **RF27 – Alterar a própria senha:**
    - Exige a senha atual; a nova segue a política (RNF04) e é digitada duas vezes.
    - Ao alterar, as outras sessões do usuário são encerradas.

### 4.2 Salas, equipamentos e categorias

- **RF06 – Cadastro de laboratórios:** nome, bloco, largura e comprimento reais em metros e observações.
    - Dimensões numéricas, maiores que zero e de até 100 m, com até duas casas decimais; zero, negativo ou texto → recusado.
    - O nome é único dentro do bloco, entre salas ativas.
    - Reduzir as dimensões de modo que algum equipamento fique fora da sala → recusado, listando os itens a reposicionar.
    - Uma sala nova gera a planta vazia na proporção informada.
- **RF07 – Cadastro de equipamentos e periféricos:** o equipamento tem nome, categoria, descrição, observações, foto opcional e posição na planta. O periférico (monitor, teclado, mouse…) está sempre vinculado a um único equipamento.
    - O equipamento nasce Verde.
    - O periférico não tem status próprio (2.1) e está sempre na mesma sala do equipamento a que está vinculado.
    - O equipamento usa uma categoria do tipo equipamento; o periférico, uma do tipo periférico.
    - Nome e categoria são obrigatórios.
- **RF08 – Representação por ícones:** cada item é exibido com o ícone da sua categoria.
    - Trocar a categoria troca o ícone; trocar o ícone da categoria atualiza todos os itens dela.
    - O ícone aparece sempre com o indicador de status (RNF05).
- **RF28 – Manter categorias:** o Administrador cria, edita e exclui categorias com nome, tipo (equipamento ou periférico) e ícone escolhido numa biblioteca de ícones do sistema.
    - O nome é único entre categorias ativas.
    - Uma categoria em uso por algum item ativo não pode ser excluída; o sistema lista os itens que a usam.
    - Um conjunto inicial de categorias é carregado na implantação.
- **RF09 – Posicionamento espacial (arrastar e soltar):** no modo de edição, o Servidor ou o Administrador arrasta o ícone para a nova posição.
    - A posição é gravada em metros, relativa à sala, e se mantém ao recarregar a página e em qualquer tamanho de tela.
    - Soltar fora dos limites da sala ou sobre outro ícone → o ícone volta à posição anterior com aviso.
    - O modo de edição não aparece para o Estudante; uma requisição direta recebe HTTP 403.
    - Existe uma alternativa sem arrastar: setas do teclado ou formulário de coordenadas (RNF05).
- **RF10 – Modificação de layout:** o Servidor e o Administrador movem equipamentos dentro da sala ou para **qualquer sala ativa**, e transferem periféricos de um equipamento para outro.
    - O equipamento movido leva junto os periféricos, o histórico, as ocorrências e o chamado não fechado, se houver.
    - Na sala de destino, o sistema propõe uma **posição livre** (que não fica sobre outro ícone) e o usuário pode ajustá-la; se não houver espaço livre, o usuário escolhe a posição antes de confirmar.
    - Salas excluídas não aparecem como destino, nem salas que ultrapassariam o limite de equipamentos (RNF07).
    - Um periférico transferido passa para a sala do novo equipamento.
    - Cada movimentação gera log com a sala e a posição anteriores e as novas (RN01).
- **RF11 – Exclusão com confirmação explícita:** nenhum item é excluído sem confirmação do Administrador.
    - A confirmação mostra o que será afetado: equipamentos e periféricos da sala, periféricos do equipamento, solicitações que serão descartadas e chamados que serão cancelados.
    - Desistir da confirmação não altera nada.
    - A exclusão é lógica (RN02).
- **RF12 – Upload de imagens:** segue a RN05.
    - Arquivo acima de 5 MB → HTTP 413; formato diferente de JPG ou PNG → HTTP 400. A mensagem informa o limite e o formato aceitos.
    - Em caso de erro, o texto já digitado no formulário é mantido.
    - O formulário avisa para não fotografar pessoas (RNF06).
- **RF13 – Observações formatadas:** as observações da sala e do equipamento aceitam negrito, itálico, listas e links.
    - Qualquer outra marcação, inclusive scripts, é removida antes de gravar.
    - Links só com `http` ou `https`.
    - Até 5.000 caracteres.
    - Cada edição gera log com a versão anterior (RN01).

### 4.3 Status, ocorrências, chamados e auditoria

- **RF14 – Transição de status por perfil:** o status do equipamento só muda pelas transições T1 a T9 (2.2).
    - Estudante e Servidor só fazem T1 (Verde → Vermelho); as demais são do Administrador.
    - Transição fora da tabela ou para um estado que já mudou → HTTP 409; perfil sem permissão → HTTP 403 (RN03).
    - Cada transição gera log (RN01) e as notificações da RF20.
- **RF15 – Registro de ocorrências:** texto livre, foto opcional e, opcionalmente, o periférico afetado, vinculados ao equipamento e visíveis a todos os usuários autenticados.
    - O texto é obrigatório, com até 2.000 caracteres.
    - O periférico, quando indicado, é um dos vinculados ao equipamento.
    - Pode ser registrada em qualquer status e não muda o status.
    - Mostra o nome do autor e a data; o e-mail do autor só é visível ao Administrador (RNF06).
    - O Administrador pode ocultar uma ocorrência imprópria ou com dado pessoal, com justificativa; a ocultação é uma exclusão lógica (RN02).
- **RF25 – Solicitar chamado (Servidor):** o Servidor pede intervenção técnica para um equipamento ou para a sala (problema estrutural).
    - Descrição obrigatória e foto opcional (RN05).
    - A solicitação nasce PENDENTE (2.3) e não muda o status do equipamento.
    - Já existe solicitação pendente ou chamado não fechado para o item → o sistema mostra o existente e oferece registrar uma ocorrência (RN06).
    - O Servidor acompanha o andamento pelas notificações (RF20) e vê, na página do item, o estado das próprias solicitações.
    - O Administrador converte a solicitação em chamado (RF32) ou a descarta com justificativa; o solicitante é notificado.
- **RF31 – Desfazer falso alarme:** o Administrador devolve a Verde um equipamento Vermelho (T2).
    - Justificativa obrigatória, gravada no log.
    - Uma solicitação PENDENTE do equipamento não é afetada e continua na fila.
    - Quem reportou a falha é notificado.
- **RF18 – Fila do Administrador:** tela única com três grupos: solicitações PENDENTES, equipamentos em Vermelho (por definição, sem chamado) e chamados não fechados.
    - Cada linha mostra sala, item, estado, prioridade (quando houver) e idade.
    - Filtros por sala, grupo, estado e prioridade; ordenação por idade e por prioridade.
    - Itens excluídos não aparecem.
- **RF32 – Ciclo de vida do chamado:** o Administrador abre, assume, resolve, reabre e cancela chamados seguindo a tabela 2.4.
    - Abrir exige descrição técnica, prioridade (Baixa, Média ou Alta) e um responsável entre os Administradores ativos. Se o item tem solicitação PENDENTE, ela passa a CONVERTIDA, mesmo que o chamado tenha sido aberto direto pelo popover.
    - Assumir pode ser feito pelo responsável ou por outro Administrador, que passa a ser o responsável; resolver, só pelo responsável.
    - Abrir chamado de equipamento leva-o a Amarelo (T3 ou T4) e guarda o status anterior.
    - Resolver exige nota técnica; reabrir exige motivo; cancelar exige justificativa e devolve o equipamento ao status anterior à abertura (T7).
    - No máximo um chamado não fechado por item (RN06).
    - Cada mudança gera log (RN01) e notificações (RF20).
- **RF19 – Encerramento formal:** fechar o chamado é uma etapa obrigatória, feita pelo Administrador com parecer técnico gravado.
    - Só um chamado RESOLVIDO pode ser fechado.
    - Parecer obrigatório, com pelo menos 10 caracteres, e resultado obrigatório: "consertado" (equipamento → Verde, T5) ou "sem conserto" (equipamento → Vermelho, T6, podendo depois ser excluído).
    - Chamado de sala fecha sem mudar status de equipamento.
    - FECHADO é final.
- **RF16 – Log estruturado de eventos:** cada evento da RN01 gera um registro imutável.
    - O registro tem autor, data e hora, entidade, ação, valor anterior e valor novo.
    - Não existe tela nem rota para editar ou apagar registros do log.
    - O registro é gravado na mesma transação da alteração: se uma falha, a outra também não acontece.
    - O Administrador consulta o log numa tela própria, com filtros por entidade, autor, ação e período.
- **RF17 – Histórico do equipamento:** lista, em ordem cronológica, mudanças de status, ocorrências, solicitações, chamados, movimentações (inclusive em salas anteriores) e edições.
    - Disponível a todos os usuários autenticados para equipamentos ativos.
    - Continua disponível ao Administrador depois da exclusão (RF29).
- **RF29 – Consultar itens excluídos:** o Administrador lista salas, equipamentos, periféricos, categorias e ocorrências ocultas, com quem excluiu e quando, e abre o histórico completo de cada um.
    - Pesquisa por nome e filtro por tipo.
    - Itens excluídos não aparecem para os outros perfis.
- **RF30 – Restaurar item excluído:** o Administrador devolve um item excluído às telas operacionais.
    - Restaurar uma sala restaura junto os itens que foram excluídos com ela, mas não os que já tinham sido excluídos antes.
    - Equipamento cuja sala está excluída → o Administrador restaura a sala antes ou escolhe outra sala ativa.
    - Equipamento cuja categoria está excluída → o Administrador restaura a categoria ou escolhe outra.
    - Periférico cujo equipamento está excluído → o Administrador restaura o equipamento antes ou vincula o periférico a outro equipamento ativo.
    - O status volta ao guardado na exclusão, Verde ou Vermelho (T9).
    - Se a posição antiga estiver ocupada ou fora da sala, o item recebe uma posição livre (RF10).
    - A restauração não pode ultrapassar o limite de equipamentos da sala (RNF07).
    - Chamados cancelados e solicitações descartadas na exclusão não voltam.
    - A restauração gera log (RN01).

### 4.4 Notificações e e-mails

- **RF20 – Notificações dentro do sistema:** avisos gerados pelos eventos abaixo.

    | Evento | Quem recebe |
    | :--- | :--- |
    | Falha reportada (T1) | Todos os Administradores |
    | Solicitação de chamado criada | Todos os Administradores |
    | Solicitação de chamado descartada | Solicitante |
    | Chamado aberto | Quem reportou a falha ou fez a solicitação; responsável |
    | Chamado assumido, resolvido, reaberto, cancelado ou fechado | Quem reportou a falha ou fez a solicitação; responsável |
    | Falso alarme desfeito (T2) | Quem reportou a falha |
    | Solicitação de perfil Administrador criada | Todos os Administradores |
    | Solicitação de perfil Administrador aprovada ou recusada | Solicitante (também por e-mail, RN04) |

    - Quem executou a ação não recebe aviso dela.
    - O cabeçalho mostra o contador de não lidas; abrir um aviso leva ao item e o marca como lido; existe "marcar todas como lidas".
    - Se o item do aviso foi excluído, o aviso continua na lista e o link mostra "item excluído".
- **RF21 – Envio assíncrono:** e-mails (RN04) e notificações são processados em segundo plano.
    - A resposta à ação do usuário não espera o envio.
    - Um envio que falha é tentado de novo até três vezes, e a falha final fica registrada.
    - Com o serviço de e-mail fora do ar, cadastro e recuperação de senha continuam respondendo normalmente.

### 4.5 Visualização

- **RF22 – Dashboard de laboratórios:** um card por sala ativa, com nome, bloco e contagem de equipamentos por status.
    - A contagem usa cor, ícone e texto (RNF05).
    - Filtros por bloco e por status.
    - Sem salas cadastradas → estado vazio; para o Administrador, um atalho para cadastrar sala.
- **RF23 – Planta interativa:** a planta da sala é desenhada em escala (RNF02), com zoom e pan e um popover em cada ícone.
    - Zoom e pan funcionam com mouse, toque e teclado.
    - O popover mostra nome, categoria, periféricos, status, última ocorrência visível (ocorrências ocultas não contam) e **apenas** as ações permitidas ao perfil.
    - Os ícones são alcançáveis pelo teclado (Tab) e o popover abre com Enter (RNF05).
- **RF24 – Edição em lote:** o Administrador seleciona dois ou mais equipamentos da mesma sala, por clique ou por área, e aplica uma operação: mover para outra sala (regras da RF10), alterar categoria (para outra do tipo equipamento), editar observação ou excluir.
    - O sistema mostra o resumo e pede confirmação antes de aplicar.
    - A operação é tudo ou nada: se um item falhar, nenhum é alterado, e o sistema mostra qual item impediu.
    - Um log por item (RN01).

## 5. REQUISITOS NÃO FUNCIONAIS (RNF)

- **RNF01 – Escopo:** o sistema se restringe ao mapeamento espacial e operacional dos laboratórios. Não faz controle patrimonial ou contábil nem inventário de software.
- **RNF02 – Proporção espacial e responsividade:**
    - A planta respeita a razão entre largura e comprimento reais, com erro de até 1%.
    - Posições são guardadas em metros e não dependem do tamanho da tela.
    - A planta se adapta a telas a partir de 360 px de largura sem distorcer.
- **RNF03 – Validação no servidor:** permissões, transições, formatos, tamanhos e limites são validados no servidor; a validação no navegador é só conveniência. Requisições forjadas recebem 400, 403 ou 409 conforme a RN03.
- **RNF04 – Segurança:**
    - **Política de senha:** pelo menos 8 caracteres, com letra e número, e diferente do prefixo do e-mail.
    - Senhas guardadas com hash adaptativo e sal, nunca em texto; tokens aleatórios, de uso único e guardados apenas como hash.
    - Proteção contra CSRF em todo formulário e em toda requisição que altera dados, inclusive as parciais.
    - **Limite de tentativas:** 5 logins falhos em 15 minutos para a mesma conta ou o mesmo IP bloqueiam novas tentativas por 15 minutos, com mensagem genérica.
    - **Sessão:** expira após 2 horas sem atividade; cookie de sessão inacessível a scripts, enviado só por HTTPS e restrito ao próprio site.
    - HTTPS obrigatório em produção.
    - Mensagens de erro não revelam se um e-mail está cadastrado.
- **RNF05 – Acessibilidade:** meta WCAG 2.1 nível AA.
    - O status **nunca** é indicado só pela cor: sempre há ícone ou forma e rótulo textual (2.1).
    - Contraste mínimo de 4,5:1 para texto e 3:1 para ícones.
    - Todas as ações da planta podem ser feitas pelo teclado, inclusive mover equipamento (RF09).
    - Ícones e controles têm nome acessível para leitores de tela.
- **RNF06 – Privacidade (LGPD):**
    - **Dados pessoais tratados:** nome, e-mail institucional e autoria de ações, ocorrências e fotos. **Finalidade:** operação dos laboratórios e auditoria.
    - O nome do autor é visível a usuários autenticados; o e-mail, só ao Administrador.
    - Fotos têm os metadados removidos (RN05), e o formulário de envio avisa para não fotografar pessoas. O Administrador pode ocultar ocorrência ou foto com dado pessoal (RF15).
    - **Retenção:** logs, ocorrências e chamados são mantidos enquanto o sistema operar, para auditoria. A pedido do titular, ou depois de 2 anos de conta desativada, nome e e-mail são anonimizados e a autoria passa a aparecer como "Usuário removido".
    - O sistema exibe um aviso de privacidade acessível a partir do cadastro e do rodapé.
- **RNF07 – Desempenho:**
    - A planta de uma sala com até 200 equipamentos carrega em até 2 segundos na rede do campus.
    - Reportar falha, mover, registrar ocorrência e mudar estado de chamado respondem em até 1 segundo em 95% dos casos.
    - O dashboard com até 100 salas carrega em até 2 segundos.
    - Limite de 200 equipamentos ativos por sala. Cadastro, movimentação, edição em lote e restauração que ultrapassariam o limite são recusados.
- **RNF08 – Compatibilidade:**
    - As duas últimas versões estáveis de Chrome, Firefox, Edge e Safari.
    - Computador, tablet e celular a partir de 360 px de largura.
    - Toque para zoom, pan e ações do popover; arrastar e soltar com mouse e com toque, sempre com a alternativa da RF09.
- **RNF09 – Backup e disponibilidade:**
    - Backup automático diário do banco e das imagens, guardado por 30 dias, com cópia fora do servidor da aplicação.
    - Teste de restauração a cada semestre.
    - Disponibilidade de 99% no horário de funcionamento do campus (segunda a sábado, das 7h às 22h); manutenções programadas fora desse horário.

## 6. PARÂMETROS ADOTADOS

Os valores abaixo valem para a implementação e os testes. Ficam reunidos aqui para que a coordenação possa ajustá-los num só lugar, sem mudar a estrutura do spec; no código eles ficam na configuração (ver [plan.md](plan.md)).

| Item | Valor | Onde |
| :--- | :--- | :--- |
| Validade dos tokens | 24 h (confirmação) · 1 h (recuperação) | RF04 |
| Reenvio de link | No máximo 1 por minuto | RF04 |
| Parecer técnico | Mínimo de 10 caracteres | RF19 |
| Política de senha | 8+ caracteres, letra e número | RNF04 |
| Bloqueio de login | 5 falhas em 15 min → 15 min | RNF04 |
| Expiração da sessão | 2 h sem atividade | RNF04 |
| Dimensão máxima da sala | 100 m | RF06 |
| Equipamentos por sala | 200 | RNF07 |
| Tamanho de textos | 2.000 (ocorrência) · 5.000 (observação) | RF13, RF15 |
| Prioridades de chamado | Baixa, Média, Alta | RF32 |
| Retenção e anonimização | 2 anos após desativação | RNF06 |
| Backup | Diário, 30 dias | RNF09 |
| Horário de disponibilidade | Seg–sáb, 7h–22h | RNF09 |
