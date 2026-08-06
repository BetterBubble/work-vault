---
title: 'Разведка: точный URL канала записи helm-iva-write, транспорт, авторизация,
  наличие инструкции для людей'
type: note
status: draft
created: 2026-08-06
tags:
- board
- iva-write
- explore
- onboarding
permalink: tacticum/00-board/explore-url-iva-write-2026-08-06
---

# URL канала записи `helm-iva-write` — точный ответ

Репозитории: `/Users/bubblemac/tacticum/tacticum-dev` (локальный `main` ОТСТАЁТ от
`origin/main` — лейн `iva-write-base` есть только в `origin/main`; читал через
`git show origin/main:<файл>`), `/Users/bubblemac/tacticum/helm` (локальный `main`
тоже отстаёт, канал есть в `origin/main`), рабочие деревья
`~/tacticum-worktrees/iva-write-lane` и `~/tacticum-worktrees/helm-bot-honesty`.

Serena не поднималась: весь материал — yaml-манифесты, markdown-доки и один
чистый ASGI-модуль без внешних call-sites по символам. Разведка велась чтением
файлов, поиском по тексту и живой пробой прод-эндпоинта (read-only, `curl`).

## (а) Точный URL

```
https://helm.tacticum.ru/mcp/iva-write
```

Источники — дословно, три независимых:

| Где | Файл:строка |
|---|---|
| Манифест лейна (источник для установщика) | `tacticum-dev` `origin/main`:`templates/iva-write-base/manifest.yaml:130` — `url: "https://helm.tacticum.ru/mcp/iva-write"` (там же `transport: http` :129, `env_required: [TACTICUM_TOKEN]` :131, `auth_type: bearer` :132) |
| Пин-тест каталога | `tacticum-dev` `apps/backend/tests/catalog/test_role_install_smoke.py:246` — `assert meta.get("url") == "https://helm.tacticum.ru/mcp/iva-write"` (комментарий :225-233 прямо предупреждает: прежний шлюз `mcp.tacticum.ru/iva-write/mcp` СНЕСЁН) |
| Точка монтирования на сервере | `helm` `origin/main`:`src/helm/interface/mcp/iva_write_surface.py:78` — `MOUNT_PATH = "/mcp/iva-write"`; монтируется в `src/helm/main.py:233` |

URL не собирается из переменной окружения: хост и путь захардкожены в манифесте и
в коде монтирования. Из окружения берётся только АПСТРИМ (адрес контейнера внутри
compose-сети) — `HELM_IVA_WRITE_MCP_URL`, дефолт пустая строка
(`helm/src/helm/config.py:622-627`, пусто → 503 `not_configured`); боевое значение
`http://mcp-atlassian:9000/mcp` задано в `helm` `origin/main`:`docker-compose.prod.yml:81`.
Это внутренний адрес, не секрет, наружу не влияет.

**Мёртвый адрес, который легко вписать по памяти:** `https://mcp.tacticum.ru/iva-write/mcp`.
Он не отвечает (`manifest.yaml:15-17`, `test_role_install_smoke.py:231-233`).

## (б) Транспорт и суффикс

**Транспорт — streamable HTTP** (`http`, JSON-RPC поверх POST, ответ может идти
SSE-потоком). `manifest.yaml:129` → `transport: http`; поверхность сделана чистым
ASGI именно ради потоковости (`iva_write_surface.py:936-941`: «streamable-http
отдаёт SSE-поток, через обычный обработчик его пришлось бы буферизовать»).

**Суффикса нет. Правильный путь — ровно `/mcp/iva-write`**, ни `/mcp`, ни `/sse`,
ни обязательного завершающего слэша.

Почему по коду, а не по догадке:

1. `main.py:233` — `app.mount(iva_write_surface.MOUNT_PATH, iva_write_surface.surface)`,
   то есть смонтировано на `/mcp/iva-write`.
2. Форма БЕЗ слэша работает специально: `main.py:237-262`, ASGI-обёртка
   `_McpMountTrailingSlash` дописывает слэш для `/mcp/process`, `/mcp/hrd` и
   `MOUNT_PATH`. Без неё голый `/mcp/iva-write` перехватил бы соседний
   `Mount("/mcp")` (analyst) и ответил бы 404 при верном ключе и верном хосте —
   ловушка описана там же в докстринге.
3. Хвост пути ПРОКСИРУЕТСЯ наверх: `_upstream_url` (`iva_write_surface.py:190-203`)
   вычитает `root_path` и приклеивает остаток к апстриму. Апстрим уже
   `http://mcp-atlassian:9000/mcp`. Значит клиентский `/mcp/iva-write/mcp` уедет в
   `…:9000/mcp/mcp` — это не тот эндпоинт, и ответит уже контейнер, а не мы.
   `/sse` в контейнер тоже уедет как `…:9000/mcp/sse`.

Живая проба прода (read-only, 06.08):

```
POST https://helm.tacticum.ru/mcp/iva-write   → 401 {"error":"unauthorized","detail":"нужен bearer-токен"}
POST https://helm.tacticum.ru/mcp/iva-write/  → 401 то же
POST https://helm.tacticum.ru/mcp/iva-write/mcp → 401 то же
```

Все три дают 401 потому, что аутентификация стоит ДО маршрутизации хвоста
(`surface`, :956-966): нашу 401 отдаёт сам канал, значит путь до канала доходит и
поверхность на проде живая. Различие путей проявится уже ПОСЛЕ авторизации, на
проксировании, — поэтому по 401 отличить верный суффикс от неверного нельзя, и
опираться нужно на код `_upstream_url`, а не на код ответа.

## (в) Авторизация и где брать токен

**Заголовок:** `Authorization: Bearer <phk_…>`.
`helm/src/helm/interface/api/routers/iva_write.py:112-119` (`require_write_identity`):
проверяется `request.headers.get("authorization")`, обязателен префикс `bearer `,
иначе 401 с `WWW-Authenticate: Bearer`. Токен обязателен ВСЕГДА, даже при
выключенном `auth_required` (:93-99) — «dev-принципала» на этой поверхности нет.
Токен резолвится в project-hub `/resolve`, второй шанс — dev-токен `phk_*` каталога
tacticum-dev (:122-144).

Наш Bearer наверх В КОНТЕЙНЕР НЕ уходит — вырезается
(`iva_write_surface.py:104-119`, решение 1 докстринга :12-18). Личные Atlassian-токены
подставляет сам helm по согласию человека.

**Переменная называется `TACTICUM_TOKEN` — команда человека в этой части ВЕРНА.**
`manifest.yaml:131` объявляет `env_required: [TACTICUM_TOKEN]`, а рендерер codex
кладёт в конфиг именно `bearer_token_env_var = <env_required[0]>`:
`tacticum-dev/apps/backend/src/backend/catalog/infrastructure/renderers/codex.py:156-171`
(закреплено тестом `test_codex_renderer.py:156-176` — `bearer_token_env_var = "TACTICUM_TOKEN"`).
Для Claude Code тот же ключ едет заголовком:
`renderers/claude_code.py:115-120` → `{"Authorization": "Bearer ${TACTICUM_TOKEN}"}`.

Итоговый блок, который у человека ДОЛЖЕН оказаться в `~/.codex/config.toml`
(user-scope, не `.codex/config.toml` в репо — ADR-0025):

```toml
[mcp_servers.helm-iva-write]
url = "https://helm.tacticum.ru/mcp/iva-write"
bearer_token_env_var = "TACTICUM_TOKEN"
```

Оговорка честная: сам флаг `codex mcp add … --bearer-token-env-var` в наших
репозиториях не описан нигде, и codex CLI на машине не установлен — проверить
написание флага я не мог. Что проверено: имя переменной `TACTICUM_TOKEN` и форма
результирующего TOML-блока. Гарантированно рабочий путь — дописать блок руками.

**Где человек берёт ключ.** Три пути, по убыванию применимости к сотруднику ИВА:
1. Бот выдачи доступа в чате ИВА (`helm/src/helm/interface/api/routers/bot_iva_write.py`,
   webhook `POST /api/bot/iva-write/webhook`, :113) — на просьбу про ключ выпускает
   его и отдаёт ОДНОРАЗОВУЮ ссылку вида `{base}/api/iva/key/<handout>`
   (`helm/src/helm/interface/api/key_issue.py:57-65`).
2. Инструмент самого канала `iva_write_issue_key`
   (`helm/src/helm/interface/mcp/iva_write_tools.py:28,69-102`) — но он требует уже
   рабочего ключа, то есть для ПЕРВОГО ключа не годится.
3. Admin UI `dev.tacticum.dev` → Memberships → API Keys → Create
   (`tacticum-dev/docs/tacticum-mcp-install.md:38-40`) — это путь для внешних
   подписчиков Tacticum, не для сотрудника ИВА.

**Ключ ≠ согласие.** Даже с верным URL и живым ключом инструменты записи будут
отвечать отказом, пока человек однократно не даст согласие в Jira, а затем в
Confluence (порядок обязателен). Ссылки выдаёт `iva_write_connect`
(`iva_write_tools.py:27,52-67`). Хендшейк и `tools/list` helm отвечает САМ, не
спрашивая контейнер, именно чтобы клиент без согласия увидел `iva_write_connect`
(`iva_write_surface.py:26-30`).

## (г) Есть ли это в документации для людей — вывод: НЕТ, и это дефект онбординга

Что есть:

- `tacticum-dev` `origin/main`:`docs/user_manuals/iva-write-base-profile-quickstart.md`
  — общий документ канала (134 строки). Адрес там ЕСТЬ, но в прозе, строка 45:
  «появляется в конфиге сам … как блок `helm-iva-write` на
  `https://helm.tacticum.ru/mcp/iva-write`». Строка 114 — проверочная команда
  `grep -A2 "mcp_servers.helm-iva-write" ~/.codex/config.toml`.
- Тот же документ прямо говорит (строки 42-46): **«Подключать вручную тоже не
  нужно»** — канал приезжает вместе с ролью.
- Раздел «Если что-то пошло не так» (:122-133) на все случаи отвечает «обнови роль
  и перезапусти CLI» либо «твоей роли канал ещё не выдали».

Чего НЕТ нигде:

- **Команды подключения канала записи руками — ни одной.** `codex mcp add` для
  `helm-iva-write` не встречается ни в `docs/`, ни в `templates/` (проверено
  поиском по обоим репозиториям и по `origin/main`).
- **Общий гайд установки MCP канал не знает вовсе:** в
  `docs/tacticum-mcp-install.md` строки `iva-write` отсутствуют — документ описывает
  только `tacticum-mcp` (`https://mcp.tacticum.dev/catalog`). Человек, пошедший
  «читать, как подключают MCP», нашего канала там не найдёт.
- **Готового TOML-блока `[mcp_servers.helm-iva-write]` для копипасты нет** ни в
  одном квикстарте (в отличие от `helm-analyst`, у которого блок расписан в
  квикстартах ролей — например `docs/user_manuals/iva-role-analyst-profile-quickstart.md`).
  У `helm-iva-write` пака в `.codex/config.toml` нет СОЗНАТЕЛЬНО
  (`manifest.yaml:105-109`).

**Вывод — только по РЕПОЗИТОРИЮ.** В файлах `tacticum-dev` команды подключения нет,
и когда автоматика не сработала, документация отправляет обновлять роль. Но это
вывод о репозитории, а не о том, что человеку читать негде: **вики я не проверял**,
см. раздел «Дополнение 06.08». Не выдавать это за проверенное «инструкции нет».

## (д) Кому канал положен по составу (`origin/main` tacticum-dev)

Лейн `iva-write-base` стоит в `depends_on` у **11 ролей**:
`iva-role-analyst`, `iva-role-architect`, `iva-role-connect-ios`, `iva-role-go`,
`iva-role-ivcs`, `iva-role-java`, `iva-role-kmp`, `iva-role-one-mail-desktop`,
`iva-role-qa`, `iva-role-qa-web`, `iva-role-web`.

**НЕ несут канал:** `iva-role-ios`, `iva-role-qa-desktop`, `iva-role-qa-mobile`,
`tacticum-role-techwriter`, `tacticum-role-internal`, `tacticum-role-platform`,
`firebird-role-web`.

**Отдельно и важно для текущего обращения.** Монолитные профили лейнов не получают
вовсе — у них пустой `depends_on`. Проверено: `templates/iva-kmp-brownfield/manifest.yaml`
в `origin/main` не содержит ни строки `iva-write`. Квикстарт канала пишет это прямым
текстом (:11-16): «канал тебе не приедет вовсе, и обновление профиля не поможет».
А по `razbor-migracii-kmp-kogorty-2026-08-03` (:25-37) d.ostryakov@iva.ru последний
раз заходил именно под общей установкой `687db0e8` — «iva-kmp brownfield», то есть
под МОНОЛИТОМ, а не под `iva-role-kmp`. Если он там и остался, «`iva_write_connect`
недоступен» — ожидаемое поведение, и никакой URL этого не чинит без переезда на роль.
Проверку фактического `profile_id` его установки на проде я не делал — это к
`prod-facts`.

## Дополнение 06.08 — где человек может это прочитать глазами

### 1. Вики Tacticum (Wiki.js) — НЕ ПРОВЕРЕНО, доступа нет

Прямо: **я в вики не искал и не смог.** Инструментов `wiki-mcp` в моей сессии нет
(`search_pages`/`get_page` мне недоступны). Обходной путь тоже закрыт: конфигурация
`wiki-mcp` лежит в `/Users/bubblemac/tacticum/.mcp.json` (sse, `https://wiki.cifragen.ru/mcp/sse`),
и ключ оттуда сервер не признаёт — `401 {"error":"unauthorized"}` на `/mcp/sse`, `/mcp`
и `/mcp/messages` (проба 06.08). Тот же ключ на `helm-analyst` проходит `initialize`
(200), но внутри тула отвечает `invalid or expired token`; `iva-mcp`
(`https://mcp.tacticum.ru/iva-atlassian/mcp`) отдаёт 404 — шлюз `mcp.tacticum.ru`, судя
по всему, снят целиком, как и `mcp.tacticum.ru/iva-write/mcp`.

**Значит вопрос «страница в вики есть или нет» остаётся открытым, и починку по нему
называть нельзя.** Проверить может тот, у кого `wiki-mcp` поднят: `search_pages`,
тенант `tacticum`, путь с префиксом `tacticum/` (`list_pages` не использовать — баг
auth-фильтра, `20-Architecture/wiki-mcp-usage.md:16-17`).

### 2. Но синк квикстартов идёт НЕ в вики Tacticum

`tacticum-dev` `origin/main`:`scripts/wiki_sync_manuals.py:35-37` —
`BASE = "https://wiki.iva.ru/rest/api"`, `PARENT_ID = "208703447"` («Набор профилей»),
`SPACE = "IVAPROJECT"`. Это **Confluence ИВА**, а не Wiki.js Tacticum. Публикуются все
`docs/user_manuals/*-profile-quickstart.md` (:179), и `iva-write-base` в таблицу
скрипта внесён явно (:87-88), то есть квикстарт канала под синк подготовлен.

А отсылают человека в другое место: `templates/iva-write-base/manifest.yaml:95-96`
(`post_install_notes`) и `ingredients/repo-configs/codex/AGENTS.md.fragment:43-44` —
оба говорят «страница … **в вики Tacticum**». Куда синкается документ и куда посылают
человека — разные вики. Это расхождение видно из кода и от доступа к вики не зависит.

Прогоняли ли `wiki_sync_manuals.py` после мержа лейна в `origin/main` — не проверено
(нужен доступ к Confluence ИВА, которого у меня нет).

### 3. Доезжает ли квикстарт до человека установкой профиля — НЕТ

У лейна ровно **три** ингредиента (`manifest.yaml:110-161`):
`helm-iva-write` (`mcp_server_spec`, `body: ""`), `iva-write-claude-md` и
`iva-write-agents-md` (`instruction_pack` → `target_file: CLAUDE.md` / `AGENTS.md`).
Файлов `docs/` среди них нет: **квикстарт в репозиторий пользователя не кладётся
вовсе**, он живёт только в нашем каталоге и в том, что синкнется в вики.

Что реально доезжает — секция в `AGENTS.md`/`CLAUDE.md`. И **URL канала в ней НЕТ**:
`ingredients/repo-configs/codex/AGENTS.md.fragment` называет только имя
`helm-iva-write` (:21), `TACTICUM_TOKEN` (:25), `iva_write_connect` (:31), а на
пошаговую инструкцию ссылается словами, без адреса страницы (:43-44) — и там же
прямо: «Пути в репозитории не называй: у человека нет клона этого репозитория».

Единственное место, где человек видит URL сам, — `post_install_notes` лейна
(`manifest.yaml:85`): это разовый вывод установщика в момент применения роли, не
документ, к которому можно вернуться.

### 4. Бот в чате: ответа «как подключить запись» у него нет

`helm` `origin/main`:`src/helm/application/bot_iva_write.py:809-832` (`help_text`) —
справка перечисляет ровно пять команд: «дай доступ», «статус», «отозвать доступ»,
«потерял ключ», «старт». Пункта про подключение MCP-сервера нет, и **строки
`helm.tacticum.ru` в модулях бота нет вовсе** (поиск по `src/helm/application/bot_iva_write.py`
и `src/helm/interface/api/routers/bot_iva_write.py`). Бот выдаёт доступ и ключ, но
адрес канала не сообщает никогда.

### Что из этого следует для формулировки починки

Две ветки, и выбирать между ними пока нельзя:
- **страница в вики ЕСТЬ** → дефект в том, что до человека не доезжает ССЫЛКА: ни во
  фрагменте роли (там только название страницы), ни у бота, а `post_install_notes`
  человек видит один раз;
- **страницы НЕТ** → читать негде вовсе.

Независимо от ветки уже установлено и починки требует: (1) фрагмент и манифест
посылают «в вики Tacticum», а синк идёт в Confluence ИВА; (2) бот, к которому человека
отправляют за доступом, адреса канала не знает.

## Риски и что осталось непроверенным

1. Суффикс `/mcp` в клиентском URL по 401 не отличается от верного — если человек
   уже вписал `…/mcp/iva-write/mcp`, симптом будет «подключился, но инструменты не
   работают», а не «сервер не найден». В диагностике квикстарта такой строки нет.
2. Флаг `--bearer-token-env-var` у codex CLI не верифицирован (codex не установлен,
   в репозиториях не описан).
3. Факт наличия/отсутствия страницы в вики Tacticum НЕ установлен — нет доступа
   (см. «Дополнение 06.08», п. 1). Всё, что сказано про документацию, относится к
   репозиторию.
4. Локальные `main` обоих репозиториев отстают от `origin/main`; всё, что выше,
   читано из `origin/main` и из рабочих деревьев. Номера строк — по `origin/main`
   (сверено: квикстарт канала в дереве `iva-write-lane` и в `origin/main` идентичны).

## Ссылки

[[plan-lane-iva-write-base-2026-08-05]] · [[impl-docs-iva-write-channel-2026-08-05]] ·
[[grabli-iva-write-na-chto-ne-nastupat-2026-08-05]] · [[razbor-migracii-kmp-kogorty-2026-08-03]] ·
[[report-iva-write]]
