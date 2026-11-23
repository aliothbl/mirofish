# SPEC-20261123-1640: Falha pontual no envio ao Zep não invalida o run inteiro

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-23 16:40
**Ativada:** —
**Concluída:** —
**Pausada em:** —
**Commit final:** —
**Keywords:** zep, ingestao, retry, idempotencia, run-status, relatorio
**Features:** simulacao, grafo
**Branch:** —
**Programa:** —
**Workspace:** —
**Origem:** usuário em 2026-11-23 16:40 (achado 3 da primeira execução, docs/DIAGNOSTICO-2026-11-23.md §12.2)
**Resumo:** Erros transitórios de rede no envio de ações ao Zep são tratados com segurança e, quando algo se perde, o run termina com aviso em vez de `failed`, sem bloquear o relatório.

## Objetivo

Às 13:21:20 um lote de 5 ações falhou com `_ssl.c:994: The handshake operation timed out`. Por desenho (`zep_graph_memory_updater.py:536-545`) gravações não idempotentes não são reenviadas, e no `stop()` (linha 335) qualquer lote falho marca o run como `failed`. Resultado: 5 de 866 ações perdidas invalidaram um run de 3h e bloquearam o relatório, que só aceita `completed|stopped` (`api/report.py:103-117`). Foi preciso editar `run_state.json` à mão.

## Escopo

**DENTRO:**
- Classificar a falha: erro antes de a requisição chegar ao servidor (timeout de conexão/handshake TLS, DNS) é seguro para nova tentativa; erro depois do envio (timeout de leitura, 5xx) mantém a regra atual de não reenviar.
- Retry com backoff para a classe segura (até 3 tentativas).
- Novo desfecho de run "concluído com perdas" (ex.: `stopped` + `ingestion_warnings` com contagem e ids das ações perdidas), visível na UI.
- Relatório permitido nesse desfecho, com aviso no topo do relatório.

**FORA:**
- Reconciliação com o Zep para descobrir se um lote ambíguo foi gravado.
- Mudança no fluxo de ingestão do grafo inicial (`graph_builder`), que já é fail-closed com reconciliação.

## Invariantes

- NUNCA uma gravação que pode ter chegado ao servidor é reenviada automaticamente (evita duplicatas no grafo).
- SEMPRE as ações perdidas ficam registradas (quantidade e identificação) no run.

## Implementação

Separar exceções de conexão (pré-envio) das demais no `zep_graph_memory_updater`; persistir `_failed_batches` no `run_state.json`; ajustar o desfecho no `simulation_runner` e a checagem em `api/report.py`.

### Modelo de dados

| Entidade | Campos / mudança |
|---|---|
| `run_state.json` | `ingestion_warnings: {lost_actions: int, batches: [...]}` |

<!-- Alternativas: reenviar qualquer falha — rejeitada: risco de duplicar episódios no grafo (motivo original do fail-closed). -->

## Riscos

- Classificar errado um erro pós-envio como pré-envio e duplicar episódio — mitigação: lista fechada de exceções (connect/handshake) com teste para cada uma.

## Sinais de sucesso

- Um soluço de rede durante um run longo não exige intervenção manual para chegar ao relatório.

## Critério de aceite

- [ ] Teste: exceção de handshake TLS no 1º envio é reenviada e o lote chega (cliente Zep fake).
- [ ] Teste: timeout de leitura não é reenviado; o run termina `stopped` com `ingestion_warnings.lost_actions = 5`.
- [ ] `POST /api/report/generate` aceita run com perdas e o relatório traz o aviso.
- [ ] Suíte do backend verde, incluindo os testes de barreira existentes (`test_zep_report_barrier.py`, `test_zep_simulation_barrier.py`).
