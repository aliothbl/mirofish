# SPEC-20261123-1640: Página de simulação não reinicia o run ao abrir ou recarregar

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-23 16:40
**Ativada:** —
**Concluída:** —
**Pausada em:** —
**Commit final:** —
**Keywords:** autostart, force, reload, step3, relatorio, ux
**Features:** web, simulacao
**Branch:** —
**Programa:** —
**Workspace:** —
**Origem:** usuário em 2026-11-23 16:40 (achado 1 da primeira execução, docs/DIAGNOSTICO-2026-11-23.md §12.2)
**Resumo:** Abrir ou recarregar `/simulation/<id>/start` passa a mostrar o run existente em vez de reiniciá-lo com `force`, e o relatório pode ser gerado para qualquer run terminado.

## Objetivo

`Step3Simulation.vue` chama `doStartSimulation()` no `onMounted` (linhas 691-695) com `force: true` (linha 401). Qualquer F5, Enter na URL ou link compartilhado encerra a simulação em andamento e tenta começar outra: em 2026-11-23 isso matou um run de 3h15 na rodada 141/144. Como o botão "gerar relatório" fica no mesmo componente, um run terminado não tem caminho na UI até o relatório sem ser reiniciado.

## Escopo

**DENTRO:**
- Ao montar, o componente consulta `run-status` e decide: sem run → mostra botão "Iniciar"; run ativo → anexa ao progresso (polling); run terminado (`completed|stopped|failed`) → mostra resultado e o botão de relatório.
- Início só por ação explícita do usuário. `force: true` só com confirmação ("já existe um run; reiniciar apaga o progresso").
- Botão de gerar relatório disponível para runs `completed` e `stopped`; para `failed`, exibir o motivo (`error` do run_state).
- A navegação a partir do Step 2 continua iniciando o run automaticamente uma única vez (fluxo atual), sem depender do `onMounted` da página.

**FORA:**
- Liberar relatório de runs `failed` (decisão do backend; ver SPEC zep-ingestion-resilience).
- Redesenho visual do Step 3.

## Invariantes

- NUNCA abrir, recarregar ou compartilhar uma URL encerra uma simulação em andamento.
- SEMPRE reiniciar um run existente exige confirmação explícita.

## Implementação

Separar "anexar ao run" de "iniciar run" no `Step3Simulation.vue`; passar a intenção de início pela navegação do Step 2 (query param ou estado de rota de uso único) em vez de inferir pelo `onMounted`.

### Modelo de dados

—

<!-- Alternativas: só tirar o force — rejeitada: o reload ainda tentaria iniciar e receberia 409, sem mostrar o run existente. -->

## Riscos

- Estado de uso único na rota pode iniciar duas vezes com navegação voltar/avançar — mitigação: consumir o parâmetro e substituir a rota (`router.replace`).

## Sinais de sucesso

- Recarregar a página durante um run longo mantém o run e o progresso na tela.

## Critério de aceite

- [ ] Com um run em andamento, recarregar `/simulation/<id>/start` não gera `POST /api/simulation/start` (verificado no log do backend) e a tela mostra o progresso atual. | evidence: manual @aliothbl
- [ ] Com um run `stopped` ou `completed`, abrir a URL mostra o botão de relatório e gera o relatório sem reiniciar o run. | evidence: manual @aliothbl
- [ ] Fluxo normal (Step 2 → Step 3) continua iniciando o run uma única vez.
- [ ] `vite build` sem erros.
