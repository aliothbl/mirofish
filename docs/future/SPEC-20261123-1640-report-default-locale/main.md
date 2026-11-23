# SPEC-20261123-1640: Idioma padrão configurável e parse correto do Accept-Language

**Status:** draft
**Porte:** P
**Owner:** @aliothbl
**Criada:** 2026-11-23 16:40
**Ativada:** —
**Concluída:** —
**Pausada em:** —
**Commit final:** —
**Keywords:** locale, idioma, accept-language, relatorio, i18n
**Features:** relatorio, infra
**Branch:** —
**Programa:** —
**Workspace:** —
**Origem:** usuário em 2026-11-23 16:40 (achado 4 da primeira execução, docs/DIAGNOSTICO-2026-11-23.md §12.2)
**Resumo:** O idioma de saída deixa de cair em chinês por padrão quando o cliente não manda (ou manda num formato comum) o cabeçalho de idioma.

## Objetivo

`backend/app/utils/locale.py:30-32` usa `Accept-Language` cru e compara por igualdade: sem o cabeçalho, ou com o formato padrão de navegadores e clientes HTTP (`en-US,en;q=0.9`), cai em `zh`. Um relatório disparado pela API em 2026-11-23 saiu inteiro em chinês e precisou ser gerado de novo, com custo de LLM em dobro.

## Escopo

**DENTRO:**
- Variável `DEFAULT_LOCALE` no `.env` (padrão `en` neste fork; `.env.example` documenta).
- Parse de `Accept-Language` com qualidade e região (`en-US,en;q=0.9` → `en`; `zh-CN` → `zh`).
- Threads de background (relatório, preparo) herdam o idioma resolvido, como hoje.

**FORA:**
- Tradução de logs e mensagens internas (backlog item 23).
- Novos idiomas além de `en` e `zh`.

## Invariantes

- SEMPRE o frontend que envia `Accept-Language` explícito continua recebendo o idioma pedido.

## Implementação

Função de parse em `locale.py` + leitura de `DEFAULT_LOCALE` no `Config`.

### Modelo de dados

—

## Riscos

- Mudar o padrão de `zh` para `en` diverge do upstream — mitigação: configurável por env; decisão local deste fork.

## Sinais de sucesso

- Chamadas à API sem cabeçalho de idioma geram relatório em inglês.

## Critério de aceite

- [ ] Teste: sem cabeçalho → `DEFAULT_LOCALE`; `en-US,en;q=0.9` → `en`; `zh-CN,zh;q=0.9` → `zh`; idioma desconhecido → `DEFAULT_LOCALE`.
- [ ] `POST /api/report/generate` sem cabeçalho gera relatório em inglês com `DEFAULT_LOCALE=en`. | evidence: manual @aliothbl
- [ ] Suíte do backend verde.
