---
title: rag2-ivaonehalf-gap
type: report
permalink: tacticum/20-architecture/rag2-ivaonehalf-gap
tags:
- helm
- rag2
- iva_jira
- ivaonehalf
- corpus-gap
- ingest
---

# RAG#2: дыра корпуса IVAONEHALF в векторе iva_jira

Разведка explorer (read-only, 2026-07-17). Ничего не ингестилось/не менялось. Связано: [[rag2-corpus-map]], [[rag2-r5-rebuild-and-datamap]].

## TL;DR
- **Почему gap:** IVAONEHALF (450 задач) просто НЕ попал в scope единственного полного Jira-фетча (as_of 2026-07-10). Это НЕ content-фильтр и НЕ намеренное исключение — проект отсутствует в списке 14 проектов того прогона. Прод-манифест это прямо документирует.
- **Доингест реализуем: ДА**, аддитивно и безопасно (идемпотентный upsert, коллекция не пересоздаётся → 87k IVAONE не затрагиваются).
- **Доп-фетч нужен: ДА** — канонично нужен живой Jira-фетч через adp_emb (`fields=*all&expand=changelog`). Локальный `tasks_rich` (450) даёт только title+desc без changelog/комментариев — годится максимум как урезанный stopgap, не для паритета с IVAONE.
- Это ЗАПИСЬ в рабочий RAG#2-корпус → выполнять только по отдельному решению пользователя (как Eva-wiki), НЕ в потоке.

## 1. Диагноз gap (подтверждён фактами)
- Вектор `iva_jira__bge_m3_1024`: `project=IVAONEHALF` → **0 точек**; `project=IVAONE` → 87097. Всего 319303 по 14 проектам.
- Прод-манифест `/opt/helm/data/real/jira/jira_manifest.csv` (дословно):
  > `# IVAONEHALF: в выгрузке ОТСУТСТВУЕТ (0 точек); в scope/графе/commits есть, при необходимости довыгрузить.`
  Список проектов манифеста: VCSMOB, IVAONE, VCSWEB2, VCSWEB, IMP, IVASBC, IVAUC2, SCORE, VCSMASH, LRGWEB, IVATR, IVACS, CEO, STRAT — **IVAONEHALF нет**.
- Механика ингеста: `rag2_extract.fetch_jira_issues` (`src/helm/ingest/rag2_extract.py:831`) — постраничный REST по JQL `project={project} ORDER BY updated DESC`, `fields=*all&expand=changelog`. Гоняется ПО ОДНОМУ проекту; список проектов — из конфига `rag2_jira_projects` (`config.py:255`, дефолт пустой, задаётся на прогоне) через `rag2_harness` (`jira_projects=container`, `:520`).
- IVAONEHALF НЕ в exclude нигде: `graph.py:33 IVA_PROJECTS=("IVAONE","IVAONEHALF")`, changelog/quality-скрипты дефолтят на `IVAONE,IVAONEHALF`. То есть проект «легальный», выпал только из одного полного вектор-фетча.
- Тайминг усугубил: задачи IVAONEHALF — свежий KMP-импорт багов (`cr=2026-07-07`, лейблы `kmp_bug_import`, `kmp_import_2026_07_07`). В M1-срезе 2026-07-03 их ещё не было; полный фетч 07-10 их уже застал по дате, но проект не был в списке.

## 2. Сырьё для доингеста
- Локально `data/iva/tasks_rich/`: IVAONEHALF = **450** задач. Наполнение: `sum` у 450/450, `desc` у 446/450, есть type/status/priority/labels/links/est/spent. **`chlog` = 0/450** (changelog в tasks_rich не кладётся вообще — он в отдельном tasks_changelog, который был IVAONE-only).
- Вывод: текста (title+desc) для эмбеддинга достаточно, НО для паритета с IVAONE-точками (там чанки задача+комментарии+changelog) нужен полный фетч из Jira. `tasks_rich` не несёт комментариев/истории → индекс был бы беднее.

## 3. План доингеста IVAONEHALF (пошагово, НЕ выполнять без решения пользователя)
ЗАПИСЬ в рабочий RAG#2-корпус `iva_jira` (+ Meili-зеркало). Гейт: отдельное решение пользователя, как Eva-wiki. Не в потоке перемера.

1. Доступ: живой Jira `jira.iva.ru` через adp_emb-туннель (как в исходном полном фетче; credentials группы `browse-allprojects`).
2. Прогон rag2-харнеса в scope только IVAONEHALF: `rag2_jira_projects=IVAONEHALF` (env/override), полный режим (без incremental-даты). Пайплайн: `fetch_jira_issues(project=IVAONEHALF, fields=*all, expand=changelog)` → `jira_issue_to_record` → чанкинг+эмбеддинг (`rag2/ingest.py:238`) → `store.upsert` в `iva_jira__bge_m3_1024` + Meili тем же uid.
3. Верификация после: `count project=IVAONEHALF` > 0 (ожидаемо сотни-тысячи точек с учётом чанков changelog/комментов); точечный search по IVAONEHALF-задаче; `count project=IVAONE` не изменился (== 87097).
4. Обновить `jira_manifest.csv` (снять пометку «отсутствует», добавить строку IVAONEHALF).

## 4. Риск для существующих 87k IVAONE
**Практически нулевой.** Write-путь — идемпотентный `upsert` по стабильному uid (UUID из ключа задачи+чанк), `ensure_collection` best-effort, **без recreate/delete коллекции** (`rag2/vectorstore.py:123`, `rag2/ingest.py:297`). Новые IVAONEHALF-точки имеют другие ключи → другие uid → добавляются рядом, IVAONE-точки не перезаписываются и не удаляются. Единственная общая зона — сама коллекция (создаётся, если нет; она есть). Meili-зеркало — та же семантика по uid.

## Итог
Gap = scope-пропуск одного проекта в полном фетче (подтверждён манифестом), не баг парсинга и не фильтр. Доингест реализуем и низкориск, но требует живого Jira-фетча (adp_emb) и отдельного решения пользователя (запись в рабочий корпус).

---

## Доступность выгрузки через adp (проверка 2026-07-17, read-only)

Проверял доступность источников ЧЕРЕЗ туннели с сервера helm. На adp_emb ничего не создавал/не менял. Связано: [[iva-contour-access-helm]].

Статус туннелей (ss на helm): **8443** (Jira), **8444** (API), **8446** (Eva-wiki), **8081** (JUMP) — все LISTEN.

### A) IVAONEHALF (Jira) — ДОСТУПНО, выгрузка реализуема
- Туннель **8443 ЖИВ** (ранее FAIL): `https://127.0.0.1:8443/rest/api/2/...` отвечает (404/400 на пробных путях = сервер за туннелем поднят, не connection-refused).
- Живой Jira отвечает (проверено через iva-mcp, у него свой PAT из `/opt/iva-mcp/env`):
  - `project=IVAONEHALF` → **total = 574 задачи** (в живой Jira на 2026-07-17; локальный tasks_rich-снимок был 450 → корпус вырос на ~124).
  - `IVAONEHALF-287` с `expand=changelog` → **полный changelog отдаётся**: статус-переходы с таймстемпами (`Open→Ready for Development→In Progress→Ready for Test→Test in Progress→Test in review→Closed`), авторы, worklog, links, fix-version. Т.е. `fields=*all&expand=changelog` работает.
- **Вердикт: выгрузка IVAONEHALF отсюда реализуема, ДА.** Всё есть: туннель 8443 up + PAT рабочий + данные (574 задачи с полным changelog).
- **Что нужно:** прогнать `rag2_extract`/rag2-харнес в scope `IVAONEHALF` через 8443 (base_url jira.iva.ru, basic-auth PAT). Объём ~574 задачи → сотни-тысячи точек с учётом чанков changelog/комментов. Скрипт: канон — `rag2_harness` (полный ингест, upsert в iva_jira), НЕ `extract_jira_changelog.py` (тот только для узкого changelog-дампа EpicTask). Гейт остаётся: запись в рабочий корпус — по решению пользователя.

### B) Eva-wiki (orionpro) — НЕ ДОСТУПНО без creds
- Идентификация: **Eva-wiki = `eva-wiki.orionpro.org` (10.3.0.245)**, туннель **8446 → 10.3.0.245:443**. Это HTML-вики документации/канона ИВА (Eva DOC-000245), «источник Eva-B». Отдельно от EVA-трекера `eva.iva.ru` (тот уже выгружен CSV-дампом `data/real/eva/`, 25 проектов / 6634 задачи, as-of 2026-07-04) — не путать.
- Проба туннеля 8446: nginx/1.30.0, корень → **HTTP 302 `Location: /auth/not_authorized?next_url=/`**. Туннель UP, но приложение за ним требует аутентификацию.
- Аутентификация: **session-login (форма/cookie), НЕ basic-auth/PAT** — нужны **orionpro-креды**, которых нет (совпадает с прежним реконом: «Eva-wiki требует session-login, нужны креды»). Была отложена как вторая часть Р-4.
- **Вердикт: выгрузка Eva-wiki отсюда НЕ реализуема сейчас, НЕТ** — блокер: нет orionpro session-creds.
- **Что нужно:** (1) учётка/сессия orionpro для `eva-wiki.orionpro.org`; (2) уточнить формат обхода (HTML-страницы вики → парсер, аналогично confluence-ветке); (3) отдельный extract под HTML-вики (в текущем коде такого нет — `eva_source.py` читает CSV-дамп трекера, не вики). Гейт: доступ выдаёт пользователь.

### Строки лиду
- **IVAONEHALF:** доступно с adp — ДА (8443 up, PAT ок, 574 задачи, changelog отдаётся). Доп-фетч не нужен сверх этого — фетчим прямо этот источник. Объём ~574 задачи.
- **Eva-wiki:** доступно с adp — НЕТ. Не хватает orionpro session-creds (login, не PAT). Формат — HTML-вики (10.3.0.245), нужен отдельный парсер.
