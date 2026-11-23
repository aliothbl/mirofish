# SPEC-20261123-1937: Run que terminou todas as rodadas sai de `running` sozinho

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-23 19:37
**Ativada:** —
**Concluída:** —
**Pausada em:** —
**Commit final:** —
**Keywords:** run-state, completed, deadlock, wait-for-commands, no-wait, monitor
**Features:** simulacao, web
**Branch:** —
**Programa:** —
**Workspace:** —
**Origem:** usuário em 2026-11-23 19:37 (achado da segunda execução, `sim_d80bd9216b29`, 30 rodadas)
**Resumo:** Quando as duas plataformas emitem `simulation_end`, o run passa a ser publicado como terminado, em vez de ficar preso em `running` até alguém chamar o stop.

## Objetivo

No run de 2026-11-23 (`sim_d80bd9216b29`), Twitter e Reddit terminaram as 30 rodadas às 17:40:08, mas o status ficou em `running` por quase 2 horas e a tela nunca liberou o relatório. Ao terminar, `run_parallel_simulation.py` entra em modo de espera de comandos (`interview`, `batch_interview`, `close_env`) porque o runner nunca passa `--no-wait` (`run_parallel_simulation.py:1535`). Enquanto isso, `_monitor_simulation` só publica o status final depois que o processo sai (`simulation_runner.py:641`). Ao detectar `simulation_end` nas duas plataformas, o runner só registra no log "esperando processo" (`simulation_runner.py:819-828`). Resultado: impasse, que só se resolve com `POST /api/simulation/stop` manual.

## Escopo

**DENTRO:**
- Separar "rodadas concluídas" de "processo encerrado": com `simulation_end` em todas as plataformas ativas e a ingestão no Zep drenada, o run é publicado como `completed`, mesmo com o ambiente ainda vivo esperando comandos.
- Expor no estado do run se o ambiente de entrevistas continua ativo (`env_alive`) e por quanto tempo fica (política de encerramento definida na SPEC `report-interview-env-lifecycle`).
- A UI do Step 3 reconhece o novo estado terminal e libera o relatório.
- `stop` num run já `completed` encerra o ambiente sem rebaixar o status para `stopped`/`failed`.

**FORA:**
- Ordem relatório × entrevistas e encerramento automático do ambiente (SPEC-20261123-1937-report-interview-env-lifecycle).
- Reinício do run ao recarregar a página (SPEC-20261123-1640-simulation-page-autostart).

## Invariantes

- SEMPRE um run cujas plataformas ativas emitiram `simulation_end` e cuja ingestão no Zep drenou chega a um estado terminal sem ação manual.
- NUNCA publicar `completed` antes de o updater do Zep drenar (a garantia atual em `simulation_runner.py:712` continua valendo).
- NUNCA um run com processo que morreu com código ≠ 0 antes do `simulation_end` vira `completed`.

## Implementação

No parser de ações do runner, quando `_check_all_platforms_completed` for verdadeiro, disparar a mesma finalização serializada (`_finalization_lock`) que hoje roda após a saída do processo: drenar o updater do Zep e publicar `completed`. O monitor continua acompanhando o processo só para refletir `env_alive`. A saída do processo depois disso não altera o status terminal.

### Modelo de dados

| Entidade | Campos / mudança |
|---|---|
| `run_state.json` | `env_alive` (bool), `completed_at`; `runner_status=completed` pode coexistir com processo vivo |

<!-- Alternativas: (1) sempre passar `--no-wait` — rejeitada aqui porque elimina as entrevistas; a decisão sobre o ciclo de vida do ambiente fica na SPEC irmã. (2) Timeout fixo na espera de comandos — rejeitada como solução isolada porque mantém o status errado durante a espera. -->

## Riscos

- Corrida entre finalização por `simulation_end` e por saída do processo, com dois caminhos publicando estados diferentes — mitigação: os dois passam pelo `_finalization_lock`, e o primeiro estado terminal vence.
- Consumidores que tratam `completed` como "processo morto" — mitigação: revisar os usos de `RunnerStatus.COMPLETED` (API de relatório, UI) e usar `env_alive` onde importa.

## Sinais de sucesso

- Um run de 30 rodadas fica pronto para relatório segundos depois da última rodada, sem `stop` manual.

## Critério de aceite

- [ ] Teste do runner: com `simulation_end` nas duas plataformas e processo ainda vivo, `runner_status` vira `completed` após a drenagem do Zep. | verify: `cd backend && uv run --frozen python -m pytest tests -q -p no:cacheprovider -k completion`
- [ ] Teste: saída do processo depois de `completed` não altera o status terminal.
- [ ] Teste: `stop` em run `completed` encerra o ambiente e mantém `completed`.
- [ ] Run real curto (3 rodadas) chega a `completed` e a tela libera o relatório sem intervenção. | evidence: manual @aliothbl
- [ ] Suíte do backend verde.
