# Intent – Sistema de Visualização Espacial de Laboratórios (IFPB)

> **Por que** este sistema existe. O *o quê* está em [spec.md](spec.md) e o *como* em [plan.md](plan.md).

## Problema

Hoje não existe uma visão única de **onde está cada equipamento** nos laboratórios do IFPB nem de **quais estão com defeito**. Falhas são percebidas por quem usa a sala, mas o aviso chega à equipe responsável de forma informal (conversa, mensagem, papel), sem registro, sem histórico e sem acompanhamento até o conserto.

## Para quem

- **Estudante:** quer saber se a sala e a máquina que vai usar estão funcionando, e avisar quando algo quebra.
- **Servidor:** organiza a sala (posição dos equipamentos, observações) e pede intervenção técnica.
- **Administrador:** recebe os problemas numa fila, abre e acompanha chamados até o fechamento, e gerencia salas, equipamentos e usuários.

## Objetivo

Oferecer uma **planta baixa interativa** de cada laboratório, em escala real, em que cada equipamento aparece com seu status (🟢 operacional, 🟡 em manutenção, 🔴 falha), e um **fluxo de chamados** rastreável do reporte ao fechamento.

## O que é sucesso

- Qualquer usuário consegue reportar uma falha em poucos cliques, direto no ícone do equipamento.
- O Administrador vê todas as pendências numa única fila, sem depender de avisos informais.
- Todo equipamento tem histórico completo (status, ocorrências, chamados, mudanças de sala), preservado mesmo após exclusão.
- O status de cada sala pode ser consultado no dashboard sem ir até o laboratório.

## Fora do escopo

- Inventário de software instalado nas máquinas.
