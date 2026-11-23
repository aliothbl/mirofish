# SPEC-20261123-1937: Entrevistas com agentes funcionam durante a geração do relatório

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-23 19:37
**Ativada:** —
**Concluída:** —
**Pausada em:** —
**Commit final:** —
**Keywords:** interview, report-agent, env-alive, close-env, lifecycle, ipc
**Features:** relatorio, simulacao
**Branch:** —
**Programa:** —
**Workspace:** —
**Origem:** usuário em 2026-11-23 19:37 (achado da segunda execução, `report_087538cd1cfe`)
**Resumo:** O ambiente da simulação continua vivo enquanto o relatório é gerado, para que a ferramenta `interview_agents` funcione em vez de falhar sempre.

## Objetivo

O relatório só pode ser gerado com o run `completed`/`stopped` (`backend/app/api/report.py:86-117`), e as entrevistas exigem o ambiente vivo (`simulation_runner.py:1728`, `:1790`). Hoje o único jeito de terminar um run é encerrar o processo (ver SPEC-20261123-1937-run-completion-deadlock), então as entrevistas nunca funcionam pelo fluxo normal. No `report_087538cd1cfe` (2026-11-23), as 3 chamadas a `interview_agents` falharam com "模拟环境未运行或已关闭". O relatório saiu baseado só no grafo do Zep, sem a voz dos agentes, que é um dos diferenciais do MiroFish. O mesmo aconteceu no run do Lototec.

## Escopo

**DENTRO:**
- Política de ciclo de vida do ambiente após as rodadas: fica vivo até o relatório terminar, mais um tempo ocioso configurável (`SIM_ENV_IDLE_TIMEOUT`, padrão sugerido 30 min), e depois é encerrado com `close_env`.
- A API de relatório aceita run `completed` com `env_alive=true`.
- Se o ambiente não estiver vivo, o report agent é avisado antes (ferramenta `interview_agents` fora da lista ou marcada indisponível), em vez de gastar iterações com chamadas que falham.
- O relatório registra se as entrevistas foram usadas (meta ou seção de método).

**FORA:**
- Publicar o estado terminal sem esperar o processo (SPEC-20261123-1937-run-completion-deadlock, pré-requisito).
- Reabrir um ambiente já encerrado (recarregar o banco OASIS para entrevistar depois); pode virar SPEC própria.
- Chat com agentes no Step 5 (usa o mesmo IPC; só se beneficia da política).

## Invariantes

- NUNCA deixar o ambiente vivo sem prazo: todo ambiente pós-rodadas tem encerramento garantido (timeout ou `stop`).
- SEMPRE o relatório continua sendo gerado quando as entrevistas não estão disponíveis (degradação, não falha).

## Implementação

Depende da SPEC `run-completion-deadlock`. No runner, depois de `completed`, iniciar um temporizador de ociosidade renovado a cada comando IPC e a cada geração de relatório em andamento. Ao expirar, enviar `close_env`. No `report_agent`, consultar `check_env_alive` no início e ajustar as ferramentas disponíveis.

### Modelo de dados

| Entidade | Campos / mudança |
|---|---|
| `run_state.json` | `env_idle_deadline` |
| `meta.json` do relatório | `interviews_available` (bool), `interviews_used` (int) |

<!-- Alternativas: (1) gerar o relatório antes de encerrar, de forma automática — rejeitada porque o usuário pode não querer relatório. (2) Manter o ambiente vivo para sempre — rejeitada pelo custo de memória e CPU (run do Lototec: CPU a 484%). -->

## Riscos

- Ambiente ocioso consumindo RAM/CPU enquanto espera — mitigação: timeout configurável e `close_env` no `stop`.
- Entrevistas com custo de LLM extra por seção — mitigação: limite de entrevistas por relatório já existente no agente; medir no primeiro run.

## Sinais de sucesso

- Relatórios citam respostas de entrevistas com agentes; o `console_log.txt` não tem mais "Interview API call failed".

## Critério de aceite

- [ ] Teste: `POST /api/report/generate` aceita run `completed` com ambiente vivo.
- [ ] Teste: o ambiente recebe `close_env` após o timeout de ociosidade, e o timeout é renovado por comandos IPC.
- [ ] Teste: com ambiente morto, o agente não chama `interview_agents` e o `meta.json` registra `interviews_available=false`.
- [ ] Run real: relatório gerado logo após o fim das rodadas contém entrevistas bem-sucedidas no `agent_log.jsonl`. | evidence: manual @aliothbl
- [ ] Suíte do backend verde.
