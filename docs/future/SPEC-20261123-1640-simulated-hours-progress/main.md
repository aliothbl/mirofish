# SPEC-20261123-1640: Tempo simulado correto no progresso da simulação

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-23 16:40
**Ativada:** —
**Concluída:** —
**Pausada em:** —
**Commit final:** —
**Keywords:** simulated-hours, progresso, run-state, ui
**Features:** simulacao, web
**Branch:** —
**Programa:** —
**Workspace:** —
**Origem:** usuário em 2026-11-23 16:40 (achado da primeira execução, docs/DIAGNOSTICO-2026-11-23.md §12.2)
**Resumo:** O contador de horas simuladas (`T:0h` na UI) passa a refletir o tempo real decorrido na simulação.

## Objetivo

Durante as 144 rodadas do run de 2026-11-23, o log da UI mostrou `T:0h` o tempo todo e o `run_state.json` terminou com `simulated_hours: 0`, `twitter_simulated_hours: 0` e `reddit_simulated_hours: 0`. O script calcula `simulated_hour = (simulated_minutes // 60) % 24` (`run_parallel_simulation.py:1236`), que é a hora do dia, não o tempo acumulado, e o runner não atualiza o acumulado. O usuário perde a noção de quanto do período simulado já passou.

## Escopo

**DENTRO:**
- Registrar o tempo acumulado (rodada × minutos por rodada) no evento de início de rodada.
- `simulation_runner` atualiza `simulated_hours` por plataforma a partir desse evento.
- UI mostra horas acumuladas (e opcionalmente dia/hora do relógio simulado).

**FORA:**
- Outros campos de progresso já corretos (rodada, ações).

## Invariantes

- SEMPRE `simulated_hours` é não decrescente durante um run.

## Implementação

Incluir o acumulado no `log_round_start` dos scripts e consumir no parser de ações do runner.

### Modelo de dados

| Entidade | Campos / mudança |
|---|---|
| eventos de rodada em `actions.jsonl` | campo de horas acumuladas |

## Riscos

—

## Sinais de sucesso

- O log da UI mostra `T:` crescendo junto com as rodadas.

## Critério de aceite

- [ ] Run de 6 rodadas com 60 min/rodada termina com `simulated_hours = 6` no `run_state.json`.
- [ ] Teste do parser do runner cobre a atualização de `simulated_hours` por plataforma.
- [ ] Suíte do backend verde.
