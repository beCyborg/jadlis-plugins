# Jadlis

Маркетплейс плагинов для передачи **Jadlis** — персональной системы управления жизнью, живущей в Obsidian-vault и управляемой из Claude Code.

Система ставится **батчами**: один батч = один вечер. Следующий батч не выдаётся, пока предыдущим реально не пользуются, — это не формальность, а единственный способ не утонуть.

## Установка

Открой Claude Code в папке будущего vault и вставь:

```
Ты — установщик системы Jadlis. Выполни ровно эти три шага и ничего сверх них.

1. Bash: CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1 claude plugin marketplace add https://github.com/beCyborg/jadlis-plugins.git
2. Bash: claude plugin install jadlis-start@jadlis
3. Скажи мне одной строкой: «Отправь /reload-plugins, потом напиши: JADLIS-BATCH 2»

Ничего не читай, не создавай и не ставь помимо этого.
```

Если Bash в приложении недоступен — те же две команды в Терминале:

```bash
CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1 claude plugin marketplace add https://github.com/beCyborg/jadlis-plugins.git
claude plugin install jadlis-start@jadlis
```

Полный HTTPS-URL и `CLAUDE_CODE_PLUGIN_PREFER_HTTPS=1` — обе защиты сразу: shorthand `owner/repo` клонируется по SSH, а SSH-ключа у нового пользователя обычно нет.

Дальше всё ведёт драйвер: `JADLIS-BATCH 2`, `JADLIS-BATCH 3`, `JADLIS-BATCH 4`.

## Плагины

| Плагин | Батч | Что даёт |
|---|---|---|
| `jadlis-start` | — | драйвер: определяет текущий шаг пробами машины, ставит следующий плагин, держит гейты |
| `jadlis-vault` | 2 | скелет папок, `CLAUDE.md`, сниппет `hide-files.css`, первый перезапуск дня |
| `jadlis-interviewer` | 3 | интервью: смысл жизни, потребности с метриками, цели квартала |
| `jadlis-research` | 4 | `search`, `full-research`, `search-paper`, `verif` + пять MCP-серверов + скилл ключей |

`jadlis-research` ставится **выключенным** (`defaultEnabled: false`): он тянет пять MCP-серверов и четыре платных сервиса. Включается явно на батче 4.

Плагины намеренно **не зависят** друг от друга. Иначе установка батча 4 подтянула бы всё сразу и гейт «не перепрыгивать» исчез бы.

## Версии

`version` в манифестах не задан — версией служит commit SHA. Каждый пуш доезжает до пользователей без ручного бампа.

## Совместимость

- macOS + десктопный Claude Code (Bash обязателен для первого шага).
- `jadlis-research` требует Claude Code ≥ 2.1.154 (`defaultEnabled`) и ≥ 2.1.193 (`renames`).

> [!WARNING]
> Имена плагинов и маркетплейса зафиксированы с первого релиза. Переименование ломает установки у тех, кто уже поставил; удаление маркетплейса удаляет и плагины. Если переименование всё же понадобится — только через `renames` в `marketplace.json`, дописывая новую запись, а не правя старую.

## Ключи

Ни один API-ключ в репозитории не лежит. Все регистрации получатель заводит свои.

Ключи разложены по двум рельсам:

- **Рельса A** — `userConfig` плагина: `VAULT_PATH`, `BRAVE_API_KEY`, `FIRECRAWL_API_KEY`. Claude Code спрашивает их при включении, сенситивные уезжают в Связку ключей macOS.
- **Рельса B** — блок `env` в `~/.claude/settings.json`: научные ключи, которые подставляются в `curl` (`PUBMED_API_KEY`, `SEMANTIC_SCHOLAR_API_KEY`, `OPENALEX_API_KEY`, …). Пишет их скилл `/jadlis-research:keys`.

Разделение вынужденное: сенситивные значения `userConfig` не долетают до обычных Bash-вызовов — они уезжают только в MCP/LSP-конфиги и хук-процессы.

`auth.json` верификаторов (Codex, Grok) — симлинки на `~/.codex/auth.json` и `~/.grok/auth.json`, создаются локально и в git не попадают.

## Лицензия

MIT — см. [LICENSE](LICENSE).
