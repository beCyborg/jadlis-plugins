# jadlis-research — ресерч-стек (батч 4)

Четыре скилла, два workflow, пять MCP-серверов. Ставится выключенным
(`defaultEnabled: false`) — включается явно на батче 4, потому что тянет платные сервисы.

## Скиллы

| Команда | Что делает |
|---|---|
| `/jadlis-research:search` | веб-поиск с маршрутизацией Brave / Firecrawl по интенту |
| `/jadlis-research:full-research` | 7 каналов (Brave + Codex web + Grok web + Reddit + X + HN + Substack) → per-claim верификация → отчёт в vault |
| `/jadlis-research:search-paper` | научный обзор: 9 источников, snowballing, retraction-check, GRADE-синтез |
| `/jadlis-research:verif` | тройная adversarial-верификация (Codex + Fable/Opus + Grok) с арбитром |
| `/jadlis-research:keys` | ключи рельсы B, homes верификаторов, smoke-таблица PASS/FAIL |

## Ключи

**Рельса A — спрашивает Claude Code при включении плагина:** `VAULT_PATH`,
`BRAVE_API_KEY`, `FIRECRAWL_API_KEY`. Сенситивные уезжают в Связку ключей macOS.

**Рельса B — блок `env` в `~/.claude/settings.json`:** `PUBMED_API_KEY`, `PUBMED_EMAIL`,
`SEMANTIC_SCHOLAR_API_KEY`, `OPENALEX_API_KEY`, `OPENALEX_MAILTO`, `CROSSREF_MAILTO`,
`UNPAYWALL_EMAIL` (+ опц. `CORE_API_KEY`, `SCITE_API_KEY`, `CONSENSUS_API_KEY`).
Пишет `/jadlis-research:keys`.

Разделение вынужденное: **сенситивный `userConfig` не долетает до обычных Bash-вызовов** —
он уезжает только в MCP/LSP-конфиги и хук-процессы. Всё, что подставляется в `curl`
внутри протоколов, обязано жить в `settings.json` → `env`.

## Внешние зависимости

| Что | Зачем | Без него |
|---|---|---|
| Codex CLI (подписка ChatGPT) | канал `codexweb`, верификатор Codex | канал выпадает; `verif` работает на двух провайдерах |
| Grok CLI (подписка Grok) | каналы `grokweb`, `twitter`, верификатор Grok | те же каналы выпадают; `verif` деградирует |
| `uv` | substack MCP из `vendor/` | канал Substack не стартует |
| `pdftotext` (poppler) | `scripts/pdf-fetch.sh` — PDF за 0 кредитов | PDF пойдут через Firecrawl, а их там режет PreToolUse-хук |
| Obsidian CLI | dedup, wikilinks, запись в дневную заметку | vault-контракт деградирует на `CLI_UNAVAILABLE`, отчёт всё равно пишется |

## Устройство путей

- `${CLAUDE_PLUGIN_ROOT}` подставляется **в тексте SKILL.md и агентов** — там его можно писать прямо.
- В файлах, которые читаются как файлы (`protocols/`, `references/`), плейсхолдера нет:
  подстановка туда не доходит. Там пишется `{PLUGIN_ROOT}`, а значение агенту сообщает
  промпт workflow отдельной строкой.
- В JS-скриптах workflow подстановки тоже нет — скилл передаёт `pluginRoot` через `args`.
- В Bash-скриптах корень резолвится от самого скрипта: `$(cd "$(dirname "$0")/.." && pwd)`.
- Рабочие homes верификаторов живут в `${CLAUDE_PLUGIN_DATA}/verif-homes/`, а не в
  `PLUGIN_ROOT`: root меняется при каждом обновлении плагина, а homes копят сессии и кэш.
  Шаблоны (`AGENTS.md`, `config.toml`) едут в `assets/verif-homes/` и разворачиваются при
  первом запуске. `auth.json` — симлинки на `~/.codex/auth.json` и `~/.grok/auth.json`,
  в репозиторий не попадают никогда.

## Модели

Плагин **не задаёт** алиасов моделей — работает на дефолтах подписки. Агенты просят
`model: opus`. Синтез ресерча идёт через headless-мост: пробует `bridgeModel` один раз,
при недоступности падает на `claude-opus-5`, и `synthMeta.analystModel` (а с ним и
frontmatter отчёта) отражает модель, которая ответила на самом деле.
