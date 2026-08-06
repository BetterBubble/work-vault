---
title: Разведка · индексация репозиториев и MCU Backend (профиль аналитика)
type: note
status: draft
created: 2026-08-03
tags:
- board
- analyst
- разведка
- индексация
permalink: tacticum/00-board/explore-analyst-indexing-2026-08-03-1
---

# Разведка: как устроена индексация и где в ней MCU Backend

**Задача:** проверить фактуру за утверждением команды аналитики «репозиторий бэкенда
(MCU Backend) не индексирован, он даже не знает текущее состояние».
**Метод:** только чтение кода и конфигов в `/Users/bubblemac/tacticum/tacticum-dev`,
`/Users/bubblemac/tacticum/helm`, `/Users/bubblemac/tacticum/platform`,
`/Users/bubblemac/tacticum/KB-Brownfield-Bootstrap` + материалы vault. Ничего не
запускалось, к прод-базе не ходил.

---

## 1. Как устроена индексация репозиториев

Механизмов **два, и они не связаны между собой**. Их постоянно путают, потому что оба
называются «индексацией».

### 1.1. Хостовая RE knowledge base (KB) — то, что видит аналитик

Это то, о чём идёт спор. Цепочка:

| Звено | Где живёт | Что делает |
|---|---|---|
| RE-пайплайн | репо `KB-Brownfield-Bootstrap`, пакет `tacticum_re` | на машине с доступом к клону гоняет фазы, кладёт артефакты в `<repo>/.tacticum/runs/<run_id>/outputs/` |
| публикация в облако | `KB-Brownfield-Bootstrap/tacticum_re/cli/main.py:475` — команда `kb-upload` (`--repo-url`, `--run-id`, `--installation-id`) | шлёт `compact/*.json` + repomix-снапшот в облако через MCP `kb_initiate_upload` / `kb_finalize_upload` |
| приём и индексация | `tacticum-dev/apps/backend/src/backend/knowledge/` | пишет строки `repos` / `kb_runs` / `kb_artefacts`, кладёт блобы в MinIO, чанкует и индексирует в **Qdrant** |
| выдача агенту | MCP-сервер `tacticum-mcp` (`https://mcp.tacticum.dev/mcp`), тулы `kb_*` | `kb_discover`, `kb_get_task_context`, `kb_verify_api_exists`, `kb_get_code_context`, … |

Сущности (`tacticum-dev/apps/backend/src/backend/knowledge/infrastructure/models.py`):

- `RepoORM` (`models.py:33`, таблица `repos`) — зарегистрированный репозиторий, привязан
  к `workspace_id`, уникален парой (workspace, url) и (workspace, slug).
- `KBRunORM` (`models.py:64`, таблица `kb_runs`) — **один прогон индексации** одного репо.
  Статусы жёстко ограничены CHECK-констрейнтом: `'uploading' | 'processing' | 'ready' |
  'failed'` (`models.py:85-88`).
- `KBArtefactORM` (`models.py:94`) — метаданные файлов в MinIO.

Момент индексации в Qdrant описан в ADR:
`tacticum-dev/docs/adr/0034-knowledge-bc-phase1-repo-minio-qdrant-scopes.md:127` —
«**Indexing trigger:** синхронно в `kb_finalize_upload` … Status → `'ready'` только после
успешной Qdrant индексации».

Почему индексация вообще двухфазная (важно для MCU): ADR-0041 строка 112 —
«`tacticum-mcp` живёт в облаке … и **не имеет доступа к приватному репозиторию клиента**.
Поэтому код-часть всегда считает сторона с доступом к репо (CI/локаль), а облако принимает
результат и индексирует». То есть чтобы проиндексировать MCU Backend, нужен клон и доступ
на стороне ИВА — облако само сходить в `git.hi-tech.org` не может.

Запуск прогонов — из веб-консоли `kbconsole` (`KB-Brownfield-Bootstrap/kbconsole/jobs.py`,
воркер `JobRunner.run_job` `jobs.py:136`, systemd-юнит `deploy/kbconsole.service`).
**Никакого cron/расписания в репозитории нет** — я искал `cron|schedule|weekly|timer` по
`kbconsole/*.py` и `deploy/*`, совпадений ноль. Работы ставятся руками через консоль.

### 1.2. Локальный код-интеллект — то, чего у аналитика нет

Serena (LSP) и `codegraph` — индексируют **клон на машине разработчика**, к облачной KB
отношения не имеют. Правило разграничения дословно
(`tacticum-dev/templates/tacticum-development-core/ingredients/rules/local-code-intelligence.md:16-18`):

> **Boundary:** Serena answers *structure of the code as it is now*; `kb_*` answers
> *intent/architecture*. They do not overlap.

Аналитику ни то, ни другое не положено: `tacticum-core-base/manifest.yaml:28` —
«Аналитический факт-база MCP (helm-analyst, iva-read) и **serena — НЕ в core**: они в
лейнах analysis и development соответственно», а квикстарт аналитика прямо говорит
«**клон репозиториев не нужен**» (`docs/user_manuals/iva-role-analyst-profile-quickstart.md:29`).
Значит для аналитика «состояние кода» = ровно то, что лежит в облачной KB, и ничего больше.

### 1.3. Реестр проиндексированного — конфига НЕТ

Списка «что проиндексировано» в коде и конфигах **не существует**. Единственный источник —
таблицы `repos` + `kb_runs` в проде, и читается он только вызовом `kb_discover`
(`apps/backend/src/backend/knowledge/application/discover.py:34`): возвращает по одной
записи на репо — **последний ран со статусом `ready`**, в рамках workspace вызывающего
(`discover.py:80`, workspace-скоуп по ADR-0034 D3).

Профили RE-пайплайна тоже не в git: `tacticum_re/config/profile.py:11` ищет
`profiles/<id>.local.yaml`, а в репозитории лежит только `profiles/adr-domains.yaml`.
То есть какие репо гонялись — знает только VPS.

---

## 2. Список того, что проиндексировано

**Списка проиндексированного в коде нет** (см. §1.3) — есть только план и косвенные следы.

### 2.1. План индексаций (файл Глеба) — там MCU Backend ЕСТЬ

`/Users/bubblemac/tacticum-vault/90-Materials/План индексаций.xlsx`, лист `iva_svodnaya`.
Колонки: Приоритет · Тех.стек · Репозиторий · Клиент/Команда · Язык · Framework ·
Ответственный · Источник. **Колонки «проиндексирован да/нет» в файле нет** — это план, а не
статус.

Строка 20:

> `3 | Backend | https://git.hi-tech.org/ivcs/ivcs-server | IVA MCU Backend | Родионов Алексей | Трегубов Антон (?) | Доступ к агентской платформе`

То есть **MCU Backend = `ivcs-server`**, приоритет **3** (низший из трёх; у всего веба и
десктопа — 1). Рядом второй бэкенд MCU: строка 21 — `https://git.hi-tech.org/modules/media`
(IVA Media/Service, приоритет 3).

Отдельный блок строк 36–43 помечен источником «**Серверные модули (ждут разрешения на
индексацию)**» — `modules/monitoring`, `modules/conversion`, `modules/filestorage`,
**`modules/voip-signalling-gateway`**, `modules/registry`, `modules/nginx-media`,
`media/imp`, `media/imp-testing`. Приоритет у них не проставлен вовсе.

Это важно для формулировки: у «MCU Backend» два кандидата-репозитория, и они в разных
состояниях — `ivcs/ivcs-server` в общем списке с приоритетом 3, а
`modules/voip-signalling-gateway` (сигнальный шлюз MCU, с которым работает Java-профиль)
в группе «ждут разрешения».

### 2.2. Что реально гонялось — по следам в коде

Прямых упоминаний ранов в git нашлось немного, и это не полный список:

- `KB-Brownfield-Bootstrap/docs/kb-semantic-uc-dedup-report.md:4` — «Тестовые KB: **kb/5
  (iva-outlook-plugin)**, **kb/6 (load-runner)**»
- `docs/doc-variant-collapse-design.md:19` — «kb/11 Android UCIM»
- `adr/ADR-0016-product-scope-firewall.md:4` — «верифицировано на **kb/11** (kmp — мессенджер)»
- `adr/ADR-0020-scenario-taxonomy-guards.md:14,40` — «**kb/13**», блоки
  `generated-iva-mcu-*` внутри репо **iva-connect**

Последний пункт — главная ловушка отчёта. Блоки `block-035-libs-data-access-generated-iva-mcu-fn`
и `block-036-…-generated-iva-mcu-services` (`KB-Brownfield-Bootstrap/tests/test_block_role.py:14-15`)
— это **сгенерированные клиенты MCU API внутри клиентского репо iva-connect**, а не код
самого MCU-сервера. Знание «MCU» в KB есть — но это клиентская обвязка.

Второй ложный след: **`load-runner` (kb/6) в реестре helm числится продуктом «MCU
(нагрузка)»** — `helm/data/real/git/repos_registry.csv`, строка 3. Это нагрузочный
инструмент MCU, не бэкенд.

Реестр репозиториев helm (`helm/data/real/git/repos_registry.csv`, 9 строк) даёт паспорт
самого MCU Backend: `ivcs-server`, namespace `ivcs`, продукт «MCU (IVCS server)»,
`https://git.hi-tech.org/ivcs/ivcs-server.git`, **14 260 коммитов, 56 авторов, 14 290
файлов, Java (9023 файла), последний коммит 2026-06-17**. Это большой живой Java-репозиторий.

### 2.3. Косвенное подтверждение из vault (не код)

`00-Board/Находки для Diaret по профилю аналитика (валидация 2026-07-21).md:26`:

> «**KB покрывает только клиентские репо** (ios/web/rn/kmp/mail/firebird). Бэкендов
> `disk`/`servercore` в KB НЕТ → бэкенд-контракты … код-verify'ом не проверить, только
> реестрами (`api_registry_check`/`contract_check`). … рассмотреть индексацию бэкенд-репо в KB.»

То есть 21.07 это уже было зафиксировано как находка, и 31.07 всплыло снова.

Реальный e2e-прогон FR от 21.07 (`/Users/bubblemac/tacticum/_analyst-e2e/FR-1.5-d22bead37d3-final.md`)
показывает ту же картину на живом примере: использовано **два** KB-рана (kmp `1c6eb9b6`,
filestorage `f20e773f`), а контейнер `disk` (`modules__diskstorage`) — «в KB отсутствует —
код-verify по нему не выполнялся» (строка 57), и это ушло открытым вопросом Q-2 (строка 26).

**Итог по §2: доказательства, что `ivcs/ivcs-server` проиндексирован, я не нашёл. Но и
доказательства обратного в коде нет — фактический список живёт только в проде
(`kb_discover`).** Утверждение команды аналитики согласуется со всем, что видно из кода,
но подтверждается косвенно.

---

## 3. Что агент-аналитик получает от индексации и что — без неё

### 3.1. Какие тулы опираются на KB-индекс

Все `kb_*` из `tacticum-mcp`. Ключевое устройство: **каждый тул, кроме `kb_discover`,
обязательно принимает `kb_run_id`** — идентификатор конкретного прогона конкретного репо:

| Тул | Файл | Чем ограничен |
|---|---|---|
| `kb_discover(installation_id?, pin_variant?)` | `apps/backend/src/backend/knowledge/interface/mcp/kb_discover.py:29` | единственный тул без `kb_run_id`; отдаёт `entries` — по репо на строку |
| `kb_get_task_context(kb_run_id, query_text, top_k)` | `.../kb_get_task_context.py:34` | семантический поиск в Qdrant **внутри одного рана** |
| `kb_verify_api_exists(kb_run_id, api_name\|api_names)` | `.../kb_verify_api_exists.py:29` | матч по `api_index` **одного рана** |
| `kb_get_code_context`, `kb_get_block_compact`, `kb_get_nfr`, `kb_get_coverage` | там же | все по `kb_run_id` |

### 3.2. Что возвращается, если репозитория в индексе нет — ТИХО НИЧЕГО

Это ответ на главный вопрос задания, и он однозначный.

**Ошибки «репозиторий не проиндексирован» в API не существует.** Проверил все ветки:

- `kb_discover` при отсутствии ранов возвращает `{"entries": []}` — пустой список, без
  ошибки (`kb_discover.py:47-49`: «Empty entries list if no ready runs»). Репозиторий, по
  которому раны есть, но все в статусе `uploading`/`processing`/`failed`, **из выдачи
  просто исчезает** — `discover.py:4-5` и фильтр `KBRunORM.status == "ready"`
  (`discover.py:58`). Агент не отличит «не грузили» от «упало».
- Ошибки бросаются только на два случая, и оба — про чужой/несуществующий ран:
  `kb_run_not_found` и `kb_run_not_accessible` (`interface/mcp/common.py:71-80`). Обе про
  `kb_run_id`, которого нет или который из чужого workspace, а не про репо.
- Опаснее всего `kb_verify_api_exists`: если агент возьмёт `kb_run_id` **другого** репо, он
  получит честный `{"exists": false}` / список `missing` (`kb_verify_api_exists.py:74-78`).
  Формально это «в KB такого API нет», по смыслу — «искали не там». **Отличить «операции
  нет в MCU» от «MCU не в KB» по ответу тула невозможно.**

Итого разбор формулировки из задания: это не «агент получил ошибку и не сказал», и не
«агент получил пустоту и промолчал по своей вине» — это **«платформа по дизайну не
сообщает об отсутствии репозитория в индексе»**. Единственный, кто может это заметить, —
сам агент, сверив список из `kb_discover` с нужным ему репо.

### 3.3. Что аналитику остаётся без KB

`helm-analyst` (`helm/src/helm/interface/mcp/analyst_server.py`) кода вообще не знает — он
про Jira/Confluence/публичную доку/реестры. Причём именно эти тулы ведут себя честно:

- `docs_ask` (`analyst_server.py:430`) — «Если ничего не найдено — честно сообщает
  `not_found` (не выдумывает)»;
- `api_registry_check` (`analyst_server.py:447`) — «Если операции нет — честный
  `found=false` со списком **ПРОВЕРЕННЫХ** реестров (версии + `as_of`), а не выдумка»;
- `contract_check` — аналогично по JUMP.

Контраст показательный: детерминированные реестры отдают «проверено вот это, не нашли», а
KB молча отдаёт пустоту.

Важно для MCU: **API MCU у аналитика закрыт не кодом, а реестрами OpenAPI**.
`templates/iva-analysis-fr/ingredients/skills/api-contracts-discovery/SKILL.md:22`, источник №1:
«**Реестры API MCU (ВКС-сервер)** — beta.hi-tech.org и alpha.hi-tech.org … `clients-openapi.json`
— IVA Clients API (~315 операций, v2.30.0 as_of 2026-07-21); `integration-openapi.json`
(~54); `/doc/api/bot.html`». То есть контракты MCU аналитику доступны и без индексации —
недоступно **внутреннее состояние кода** (сущности, поля, реализация).

А источник №5 того же скилла (`SKILL.md:28`) перечисляет, какие репо ожидаются в KB:
`iva/one/backend/proto`, `iva/core/link/contracts`, `iva/one/backend/api-gateway`,
`imail-mirror/servercore`. **`ivcs-server` в этом перечне нет.**

---

## 4. Что предписывают навыки, когда знания по репозиторию нет (дословно)

### 4.1. `kb-navigation` — единственный навигационный навык у аналитика

Устанавливается из `tacticum-core-base` (`manifest.yaml:76`, ingredient `kb-navigation`),
тело — `templates/tacticum-core-base/ingredients/skills/kb-navigation/SKILL.md`.

Про пустой ответ (`SKILL.md:94-111`), дословно:

> ## Empty `kb_discover` ≠ "KB unavailable"
> `kb_discover` returning `{"entries": []}` means only that THIS installation's workspace
> currently has no ready KB runs. It does NOT mean the KB is down, and it does NOT prove
> the target repo has no KB. …
> Mandatory procedure on an empty discover:
> 1. `whoami()` — list ALL your installations and their workspaces.
> 2. Pick the installation whose workspace contains KB runs for the target repo.
> 3. Re-run `kb_discover(installation_id=<that installation>)` and continue under it.
> The conclusion "this repo has no KB" is allowed ONLY after the `whoami()` sweep above
> found no installation whose workspace carries a KB run for the repo.

Про недоступность сервера (`SKILL.md:42`):

> **The Tacticum MCP server is the only KB source.** There is no local-file fallback. If
> `kb_*` tools are unavailable, STOP and ask the user to connect the MCP server — never
> read local files.

**Чего в навыке НЕТ:** предписания «если целевого репо нет в `entries` — сказать об этом
пользователю явно». Навык требует не спешить с выводом «KB нет» (антипаттерн `SKILL.md:163`),
но не требует объявить вслух установленный факт отсутствия. Дыра ровно в том месте, о
котором спорят.

### 4.2. `fr-authoring` Шаг 4 — вот здесь предписание есть, и оно жёсткое

`templates/iva-analysis-fr/ingredients/skills/fr-authoring/SKILL.md:124` — «### Шаг 4.
Проверка по коду (код-verify) — **НЕ пропускать молча**». И `SKILL.md:146-148`, дословно:

> **Деградация:** KB-рана для целевого репо нет / `kb_*` недоступны → шаг пропусти с явным
> дисклеймером в FR: «код-verify не выполнялся: нет KB для <repo>»; вопрос остаётся
> открытым `Q-n`. **Молчаливый пропуск запрещён.**

Шаблон FR содержит для этого готовое место — `SKILL.md:569-572`, раздел «П.E Что выяснено
по коду», строка-заглушка: `<дисклеймер, если KB недоступна: «код-verify не выполнялся:
нет KB для <repo>»>`.

Квикстарт роли повторяет то же как штатную ситуацию
(`docs/user_manuals/iva-role-analyst-profile-quickstart.md:202`):

> «код-verify не выполнялся: нет KB» | Штатная деградация: целевой репо не в KB. Вопрос
> остаётся в реестре; нужен evidence — **админу загрузить репо**, прогнать `/update-feature`.

Значит: **нужная норма в профиле аналитика есть и сформулирована правильно.** Отсутствие
KB для MCU Backend по канону должно давать не пустой FR, а FR с явным дисклеймером и
открытым `Q-n`. Сработало ли это в конкретном провале — по коду не установить, нужен лог.

### 4.3. `codegraph-first-navigation` и `*-local-knowledge-routing` — у аналитика их НЕТ

Проверил все установки этих навыков в `templates/`:

- `codegraph-first-navigation` — только 2 копии: `iva-kmp-brownfield` и
  `iva-kmp-development-base`. Это KMP-специфика: навык описывает PreToolUse-гард, который
  блокирует grep, «only when the repo has a valid index — `.codegraph/codegraph.db`»
  (`SKILL.md:19`), и падает в Serena+Read, если индекса нет (`SKILL.md:64-69`: «If neither
  codegraph nor ccc is set up, fall back to Serena + Read — but still NOT to raw grep»).
- `*-local-knowledge-routing` — 10 копий, все в стековых лейнах (kmp, ivcs, firebird, ios,
  ucim, web). Они маршрутизируют к **собственным файлам клонированного репозитория**
  (README, `.cursorrules`, per-module README) — например
  `iva-connect-ios-development-base/ingredients/skills/ucim-local-knowledge-routing/SKILL.md:20-35`.
  Про отсутствие облачной KB в них нет ничего: они предполагают клон на диске.

Лейны аналитика (`templates/iva-role-analyst/manifest.yaml:20-24`: `tacticum-core-base`,
`tacticum-analysis-core`, `iva-analysis-fr`, `tacticum-research-base`) **ни одного из этих
навыков не содержат**. Аналитик работает без клона, без Serena, без codegraph — только
`kb_*` и факт-база MCP.

---

## 5. Следы работ по индексации MCU

**В коде — ноль.** Проверено:

- `grep -ri mcu` по `tacticum-dev` (`*.md|*.yaml|*.json|*.py`): все попадания — про
  **клиентскую** сторону MCU. Навык `openapi-mcu-dto`
  (`templates/iva-connect-ios-development-base/ingredients/skills/openapi-mcu-dto/SKILL.md`)
  — генерация DTO из OpenAPI MCU в iOS-репо; навык `mcu-signalling-conventions`
  (`templates/iva-java-development-base/ingredients/skills/mcu-signalling-conventions/SKILL.md`)
  — про репо **`voip-signalling-gateway`** (пакеты `su.ivcs.services.signalling.gateway.*`),
  рабочая папка Java-роли — «клон Java-репозитория: MCU `voip-signalling-gateway` **или**
  `modules/diskstorage`» (`docs/user_manuals/iva-role-java-profile-quickstart.md:73`).
  То есть навыки про MCU написаны **под локальный клон и Serena**, а не под KB.
- `grep -ri mcu` по `platform` — **ни одного совпадения**.
- `ivcs-server` по `tacticum-dev` — **ни одного совпадения**. Ни в ADR, ни в CHANGELOG, ни
  в манифестах.
- В `KB-Brownfield-Bootstrap` «MCU» встречается только как имя блоков внутри чужих ранов
  (`generated-iva-mcu-*`, kb/13 iva-connect) и в фикстурах.

Осторожно с похожими именами: роль `iva-role-ivcs` работает с клоном
`git.hi-tech.org/**desktop/ivcs**` (IVA Connect Desktop, C++/Qt —
`docs/user_manuals/iva-role-ivcs-profile-quickstart.md:55`), это **не** `ivcs/ivcs-server`.
Совпадение в имени очень легко принять за след индексации MCU Backend.

**В vault следы есть — как вопросы, а не как работы:**

- `14-Canon/canon-profili …:225` (созвон 31.07): «следующий срок индексации репозитория
  бэкенда […] это больше у Димы Солонко нужно спросить […] вопрос уже не в техническом,
  Дима-то его индексирует, не сомневаюсь, это просто к чему — сейчас небольшой стоп-фактор
  части фич»
- `14-Canon/canon-profili …:244` (чат после созвона): «@Монахов Иван Есть в итоге принятое
  решение по индексации репозитория MCU Backend?» — **ответа в vault нет**
- `20-Architecture/glossary.md:124`: «**MCU Backend** — репозиторий, индексация которого не
  решена; стоп-фактор части фич»
- `00-Board/call-review-2026-07-31.md:104`: «индексация MCU Backend — НЕ наша задача, но
  она стоп-фактор для профиля аналитика»
- `00-Board/call-raw-2026-07-28-1000.md:17` (сырой транскрипт дейли, распознавание грязное,
  атрибуция реплики неточная): «…уровня обновления индекса по репозиториям, по-моему сейчас
  он раз в неделю индексируется, по расписанию… если нужно чаще — можно делать чаще…
  посмотрю более предметно». Расписания в коде нет (§1.1) — либо оно снаружи репозитория
  (cron на VPS), либо реплика про другое. **Проверять на сервере, я туда не ходил.**

---

## Чего я НЕ смог установить

1. **Проиндексирован ли `ivcs/ivcs-server` фактически.** Список живёт только в проде
   (`repos` + `kb_runs`), в git его нет by design. Ответ даёт один вызов `kb_discover` под
   нужным `installation_id` — прод-часть за тимлидом.
2. **Полный список проиндексированных репо.** По косвенным следам собрал только частично:
   iva-outlook-plugin (kb/5), load-runner (kb/6), UCIM/kmp (kb/11), iva-connect (kb/13),
   filestorage и kmp (из e2e-прогона 21.07). Это заведомо неполно и частично устарело.
3. **Есть ли расписание переиндексации.** В коде нет; реплика с дейли 28.07 про «раз в
   неделю» не подтверждена ничем, кроме грязного транскрипта.
4. **Что именно называют «MCU Backend»** — `ivcs/ivcs-server` (по «Плану индексаций», строка
   20) или `modules/voip-signalling-gateway` (с которым работает Java-роль, и который в
   группе «ждут разрешения на индексацию»). Это разные репозитории с разным статусом
   в плане, и от ответа зависит формулировка запроса Монахову.
5. **Сработала ли деградация из `fr-authoring` Шаг 4 в конкретном провале профиля.** Норма
   в навыке есть и запрещает молчаливый пропуск; выполнил ли её агент — видно только из
   лога прогона, которого пока нет. По прямому указанию не гадаю.
6. **Есть ли у аналитиков вообще доступ к workspace с нужными ранами.** Находка 21.07
   (`Находки для Diaret …:26`) отдельно говорит про тип раздаваемых ключей: membership
   `phk_*` на несколько installations ломает авто-скоуп и требует явного `installation_id`.
   Это второй, независимый способ получить пустоту вместо знания — но он про ключи, а не
   про индексацию.