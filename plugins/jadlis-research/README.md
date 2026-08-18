# jadlis-research — ресерч-стек (батч 4)

Пять скиллов, два workflow, пять MCP-серверов. Ставится выключенным
(`defaultEnabled: false`) — включается явно на батче 4, потому что тянет платные сервисы.

## Скиллы

| Команда | Что делает |
|---|---|
| `/jadlis-research:search` | веб-поиск с маршрутизацией Brave / Firecrawl по интенту |
| `/jadlis-research:full-research` | до 10 каналов: web×3 (Brave + Codex web + Grok web) + Reddit + X + HackerNews + Substack, opt-in `yandex` / `youtube` / `telegram` по роутинг-дереву → per-claim верификация (schema v2) → отчёт в vault |
| `/jadlis-research:search-paper` | научный обзор: 9 источников, snowballing, retraction-check, GRADE-синтез |
| `/jadlis-research:verif` | тройная adversarial-верификация (Codex + Fable/Opus + Grok) с арбитром |
| `/jadlis-research:keys` | ключи рельсы B, homes верификаторов, smoke-таблица PASS/FAIL |

## Каналы `full-research`

| Канал | Чем берёт | Гейт / деградация |
|---|---|---|
| `web` | Brave MCP (`llm_context` / `web_search`), Firecrawl для одной страницы, place-слой через `scripts/places-fetch.sh` | `BRAVE_API_KEY` обязателен; place-слой без `GOOGLE_PLACES_API_KEY` → Brave Place |
| `codexweb` | `codex exec` с web search (`gpt-5.6-sol`, effort high) | нет CLI или квота исчерпана → канал выключается квотным probe |
| `grokweb` | Grok CLI headless, `web_search` + `web_fetch` | нет CLI → канал выпадает |
| `reddit` | MCP `reddit` (`execute_operation`) + `reddit-alt`, no-auth-лестница Arctic Shift / PullPush через `scripts/reddit-archive.py` | всё опционально: лестница работает без ключей |
| `twitter` | Grok CLI (`x_search` живёт в подписке, не в API) | нет CLI → канал выпадает |
| `hackernews` | **свой фетчер** `scripts/hn-fetch.sh` (Algolia + Firebase, 0 кредитов, полный текст комментариев) | нужен `jq`; сломался → Brave `site:news.ycombinator.com`, дальше `sourceQuality=LOW`. **MCP для HN в плагине нет** |
| `substack` | **свой фетчер** `scripts/substack-fetch.py` (анонимный `/api/v1`, отдаёт вовлечённость 👍/💬) | нужен `uv`; сломался → Brave `site:substack.com`, дальше `sourceQuality=LOW`. **MCP для Substack в плагине нет** |
| `yandex` | `scripts/yandex-search.sh` (Yandex Search API v2, async) | `YC_SEARCH_API_KEY`; нет → канал не предлагается, принудительный выбор даёт `exit 2` + `LOW` |
| `youtube` | Brave `site:youtube.com` (основной discovery) + MCP `youtube` для точного добора + `scripts/yt-transcript.py` для транскриптов | `YOUTUBE_API_KEY` только для MCP-добора; без ключа канал живёт на Brave + транскриптах |
| `telegram` | `scripts/tg-preview.sh` (публичные `t.me/s/` превью) + Brave `site:t.me`-дорки | ключей не нужно; сиды каналов — `skills/full-research/references/telegram-seed-handles.md` |

Каналы `hackernews` и `substack` раньше ходили через MCP-серверы `hn` и `substack`
(последний — вендоренный, с venv на 100+ МБ). Оба сняты: свои фетчеры дают полный текст,
честный кэш и вовлечённость, а MCP-фоллбэка сознательно нет — при поломке фетчера канал
деградирует, а не тянет мёртвый вес.

## Ключи

**Рельса A — спрашивает Claude Code при включении плагина:** `VAULT_PATH`,
`BRAVE_API_KEY`, `FIRECRAWL_API_KEY` (+ опц. `REDDITAPIS_KEY` — резервный Reddit-MCP
`reddit-alt`; опц. `YOUTUBE_API_KEY` — MCP `youtube`). Сенситивные уезжают в
Связку ключей macOS.

**Рельса B — блок `env` в `~/.claude/settings.json`:** `PUBMED_API_KEY`, `PUBMED_EMAIL`,
`SEMANTIC_SCHOLAR_API_KEY`, `OPENALEX_API_KEY`, `OPENALEX_MAILTO`, `CROSSREF_MAILTO`,
`UNPAYWALL_EMAIL` (+ опц. `CORE_API_KEY`, `SCITE_API_KEY`, `CONSENSUS_API_KEY`,
`YC_SEARCH_API_KEY` — opt-in канал `yandex`; `GOOGLE_PLACES_API_KEY` — place-слой
канала `web`). Пишет `/jadlis-research:keys`.

Разделение вынужденное: **сенситивный `userConfig` не долетает до обычных Bash-вызовов** —
он уезжает только в MCP/LSP-конфиги и хук-процессы. Всё, что подставляется в `curl`
внутри протоколов, обязано жить в `settings.json` → `env`.

## Внешние зависимости

| Что | Зачем | Без него |
|---|---|---|
| Codex CLI (подписка ChatGPT) | канал `codexweb`, верификатор Codex | канал выпадает; `verif` работает на двух провайдерах |
| Grok CLI (подписка Grok) | каналы `grokweb`, `twitter`, верификатор Grok | те же каналы выпадают; `verif` деградирует |
| `jq` | `scripts/hn-fetch.sh`, `scripts/places-fetch.sh` | канал `hackernews` падает на Brave; place-слой не работает |
| `uv` | шебанг `scripts/substack-fetch.py` и `scripts/yt-transcript.py` | канал `substack` падает на Brave; транскрипты YouTube недоступны |
| `yt-dlp` (опц.) | фоллбэк транскриптов в `scripts/yt-transcript.py` | при блокировке `youtube-transcript-api` транскрипт недоступен, канал цитирует по описаниям |
| `pdftotext` (poppler) | `scripts/pdf-fetch.sh` — PDF за 0 кредитов | PDF пойдут через Firecrawl, а их там режет PreToolUse-хук |
| Obsidian CLI | dedup, wikilinks, запись в дневную заметку | vault-контракт деградирует на `CLI_UNAVAILABLE`, отчёт всё равно пишется |
| `REDDITAPIS_KEY` (рельса A, опц.) | резервный MCP `reddit-alt` — точные имена сабов, не-английские запросы, live-метрики | сервер поднимается и отдаёт 401, в `/mcp` горит красным — **это нормально**; Reddit-канал идёт на основном MCP и no-auth-лестнице (Arctic Shift / PullPush) |
| `YC_SEARCH_API_KEY` (рельса B, опц.) | opt-in канал `yandex` — слой Рунета, которого нет у Brave | канал не предлагается; если выбран принудительно — `exit 2`, `sourceQuality=LOW`, workflow не падает |
| `GOOGLE_PLACES_API_KEY` (рельса B, опц.) | place-слой канала `web` для локальных тем | `exit 3 PLACES_KEY_MISSING` → слой сам уходит в `brave_place_search`. **Бюджет-кап в Google Cloud обязателен ДО первого вызова** — hard cap у API нет |
| `YOUTUBE_API_KEY` (рельса A, опц.) | MCP `youtube` — точный добор видео и метаданные (квота 10k units/день, поиск = 100 units) | сервер отдаёт ошибку и горит красным в `/mcp` — это ожидаемо; канал `youtube` идёт через Brave + локальные транскрипты |

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
`model: opus`. Каналы, верификаторы и curator идут на Opus 5.

Синтез ресерча (analyst) идёт через headless-мост `claude -p --model claude-fable-5`:
скилл передаёт заказанную модель в `args.aiModel`, а workflow возвращает
`aiModelActual` — ту, что ответила на самом деле. При падении моста analyst
доигрывается на Opus 5, и Phase C скилла правит `ai_model` во frontmatter отчёта
по `aiModelActual`, чтобы заметка не врала. Отключить мост целиком:
`fableBridge: false` + `aiModel: "claude-opus-5"`.

Верификаторы `/jadlis-research:verif`: Codex (`gpt-5.6-sol`), Claude Fable 5,
Grok (`grok-4.6`), арбитр — Fable 5.
