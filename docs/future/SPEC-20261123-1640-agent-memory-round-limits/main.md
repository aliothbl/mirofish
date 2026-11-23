# SPEC-20261123-1640: Limites de memória dos agentes, de iterações e de rodadas

**Status:** draft
**Porte:** M
**Owner:** @aliothbl
**Criada:** 2026-11-23 16:40
**Ativada:** —
**Concluída:** —
**Pausada em:** —
**Commit final:** —
**Keywords:** memoria, token-limit, max-iteration, rodadas, contexto, custo, modelo-local
**Features:** simulacao, llm, web
**Branch:** —
**Programa:** —
**Workspace:** —
**Origem:** usuário em 2026-11-23 16:40 (achado 2 da primeira execução + backlog item 10 de docs/DIAGNOSTICO-2026-11-23.md)
**Resumo:** A memória de cada agente passa a ter janela limitada, o loop de ferramentas ganha teto e o número de rodadas fica explícito e com padrão seguro.

## Objetivo

Na primeira execução (Lototec, 19 agentes, `gpt-4o-mini`), a memória de agentes chegou a 6.041.721 tokens; o camel truncava para 128k antes de cada chamada, o que gerou 13 erros de contexto, cada chamada perto de 128k tokens pagos e cerca de 1 min de CPU por chamada só para contar tokens (CPU em 484%). O run levou 3h15 com o padrão de 144 rodadas, que a UI não deixa evidente. Sem esses limites, modelos locais com 8–16k de contexto (GTX 1070) são inviáveis.

## Escopo

**DENTRO:**
- Janela de memória por agente configurável (`SIM_AGENT_MEMORY_TOKENS`, padrão sugerido 16k), aplicada na criação dos agentes OASIS/camel em `backend/scripts/run_*_simulation.py`.
- Teto de iterações do loop de ferramentas por ação (`SIM_MAX_TOOL_ITERATIONS`, padrão sugerido 3).
- Padrão de rodadas seguro quando o usuário não escolhe (`SIM_DEFAULT_MAX_ROUNDS`, padrão sugerido 24) e o valor efetivo exibido no Step 2 antes de iniciar.
- Aviso de custo/tempo estimado no Step 2 (agentes × rodadas).
- Log por rodada do tamanho de contexto médio/máximo enviado.

**FORA:**
- Roteamento de modelo por etapa (backlog item 9, SPEC própria).
- Resumo/compactação inteligente de memória (pode vir depois; aqui é janela simples).

## Invariantes

- NUNCA uma chamada da simulação excede a janela configurada.
- SEMPRE o número de rodadas que vai rodar é visível antes do início.

## Implementação

Configurar o `token_limit`/context creator e o `max_iteration` dos agentes camel na criação do modelo (`create_model`/criação de agentes), lendo as variáveis do `.env`; Step 2 passa a enviar sempre `max_rounds`.

### Modelo de dados

| Entidade | Campos / mudança |
|---|---|
| `.env` / Config | `SIM_AGENT_MEMORY_TOKENS`, `SIM_MAX_TOOL_ITERATIONS`, `SIM_DEFAULT_MAX_ROUNDS` |

<!-- Alternativas: só reduzir rodadas — rejeitada: a memória cresce por rodada e estoura igual em runs médios ou com modelo local. -->

## Riscos

- Janela curta empobrece a coerência dos agentes — mitigação: medir com o benchmark 2 do diagnóstico (mesma entrada, janelas 8k/16k/32k) antes de fixar o padrão.
- APIs internas do camel 0.2.78 mudam o ponto de configuração — mitigação: confirmar a API na versão travada antes de implementar.

## Sinais de sucesso

- Um run de 24 rodadas com 19 agentes termina sem nenhum erro de contexto e com custo e tempo previsíveis.

## Critério de aceite

- [ ] Run de referência (Lototec, 19 agentes, 24 rodadas): zero ocorrências de `maximum context length` e de `Context truncation performed` acima da janela no `simulation.log`. | evidence: manual @aliothbl
- [ ] Contexto máximo por chamada registrado no log fica ≤ `SIM_AGENT_MEMORY_TOKENS` + prompt de sistema.
- [ ] Step 2 mostra o número de rodadas que será usado e sempre envia `max_rounds`.
- [ ] Suíte do backend verde.
