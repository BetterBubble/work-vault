---
title: Helm — Реестр данных по сущностям (реальные колонки)
type: architecture
tags:
- helm control-tower data-registry spec
source: helm:data/data-registry.md (перенесено из gitignored data/ 2026-07-08)
permalink: tacticum/20-architecture/helm-reestr-dannykh-po-sushchnostiam-realnye-kolonki
---

# Helm — Реестр данных по сущностям

> Консолидация: аудит data/real (реальные колонки и заполненность из данных) + concept-entity-provenance.md + concept-entities.md. 30 сущностей, сгруппированы по типу сборки. Собрано 2026-07-07.

## Как читать
- Строка таблицы = один файл-источник. Колонки — РЕАЛЬНЫЕ имена из файла. Заполненность — % непустых по данным (крупные файлы — оценка по выборке).
- Тип сборки: Справочник · Прямая выгрузка · Дериват по джойну · Вычленяемая. Статус: ✓ Готово · ⚙ Уточнить · ⚠ Переделать (по provenance).
- «Ключи связи» = колонки для join к другим сущностям.

## Справочник (5)

### Продукт — ✓ Готово · _Справочник_

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| manual/reference_products.csv | product, active | product → Продукт (канонический список-измерение) | 5 строк; product 100%, active 100% | Справочник-измерение, канон 5 (CS, Largo, MCU, Mail, One), все active=true. Уже, чем product_owners_ref (там 13). Без BOM. |
| manual/product_owners_ref.csv | product, product_raw, owner_name, eol_email | product → reference_products.product; product_raw → сырое написание (Телефоны→CS); eol_email → Человек (владелец) | 13 строк; все 4 колонки 100% | Справочник владельцев. Кириллическая коллизия 'СS'/'CS' (буква С U+0421) — две записи одного CS. product_raw содержит маппинг сырых имён. owner_name свободный формат. |
| data-room/economics/product_economics_37m.csv | Продукт, ФОТ млн руб., Производственные млн руб., Итого затраты млн руб., Выручка млн руб. пайп продаж, Прибыль млн руб., Маржа, Решение | Продукт → Продукт | 10 строк; все колонки 1-8 100% | Report-style: 1 строка баннера перед заголовком (заголовок на стр.2), встроенный перенос в заголовке выручки, ниже нарратив-отчёт (не данные). 9-я хвостовая пустая колонка (ROOM='MVP'). Юнит-экономика по продукту. |
| data-room/economics/portfolio_decisions.csv | Продукт, ФОТ, Производственные, Итого затраты, Выручка, Прибыль, Маржа, Решение | Продукт → Продукт | 10 строк; все 8 колонок 100% | Report-style/рабочий лист с несколькими таблицами и нарративом (заголовок основной таблицы на стр.3). Справа — 2-я таблица (Продукт/Затраты/Доля/Решение/Комментарий), ниже под-блоки. Цифры совпадают с product_economics_37m, отличается колонкой Решение. |

### Поколение — ✓ Готово · _Справочник_

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| manual/reference_generations.csv | generation, active | generation → Поколение (справочник-измерение; ключ для blocks.generation и др.) | 6 строк; generation 100%, active 100% | Технологическая ось: 1.0, 1.5, 1.5-KMP, 1.5-RN, 2.0, 3.0, все active=true. Строки с суффиксами (не числа). Иерархия 1.5↔RN/KMP не выражена. Без BOM. |
| git/repos_manifest.csv | repo, gitlab_remote, product, generation, commits_exported, latest_commit_date | repo → Репозиторий; product → Продукт; generation → Поколение | 8 строк; product 100%, generation 100%, gitlab_remote 100% | Манифест 8 экспортированных репо: связь repo→product→generation (1.0/1.5-KMP/1.5-RN). BOM перед 'repo'. Служебный файл выгрузки. |
| manual/blocks.csv | block_id, block_name, eol_email, product, generation, cross, teams | generation → reference_generations.generation; product → reference_products.product; eol_email → Человек | 9 строк; generation 11.1% (только One 1.5), product 55.6% | Поколение заполнено лишь у 1 из 9 блоков. teams пусто 0%. Здесь Поколение — метка блока. Без BOM. |
| manual/goals.csv | goal_id, title, description, critical, product, generation, owner_hint | generation → Поколение; goal_id → directives_extra.id; product → Продукт; owner_hint → Человек | 15 строк; generation 0% (пусто), product 100%, owner_hint 86.7% | Колонка generation полностью пустая — как источник Поколения для целей не работает. Без BOM. |

### Блок — ✓ Готово · _Справочник_

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| manual/blocks.csv | block_id, block_name, eol_email, product, generation, cross, teams | block_name → directives_extra.block; eol_email → Человек (ЕОЛ); product → Продукт; generation → Поколение | 9 строк; block_id 100%, block_name 100%, eol_email 100%, product 55.6%, cross 44.4%, teams 0% | Основной справочник блоков (b_stab_gd..b_spezproekty). teams пусто 9/9 — связь блок→команда не материализована. 4 сквозных блока (cross=true) без продукта — платформа/инфра. block_name кириллица = ключ связи. Без BOM. |
| data-room/team/track_owners.csv | Контур (владелец трека / ЕОЛ-кандидат), Треки, Продукты, Gold-людей, Люди Gold | Контур → target_team.Владелец трека; Продукты → Продукт (список через ' · '); Люди Gold → Человек | 5 строк; все 5 колонок 100% | Справочник-агрегат контуров/владельцев треков (обратная сторона blocks.eol_email). Продукты и Люди Gold — денормализованные списки в ячейке. BOM в первом заголовке. |
| manual/directives_extra.csv | id, block, due, depends_on, override, override_reason, client | block → blocks.block_name; id → goals.goal_id; depends_on → directives_extra.id; client → Клиент | 15 строк; block 100%, id 100%, due 100%, client 100%, depends_on 13.3%, override 20% | Плоская таблица D01..D15, привязка директив к блокам через кириллические имена block. Без BOM. |

### ЕОЛ — ⚠ Переделать · _Справочник · роль-владелец Блока (сворачивается в person_email), не отдельный узел_

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| manual/blocks.csv | block_id, block_name, eol_email, product, generation, cross, teams | eol_email → Человек (teams.person_email / владелец блока); block_name → директивы; product → Продукт | 9 строк; eol_email 100% (9/9) | Первичный источник роли ЕОЛ: поле eol_email резолвится в Человека по почте. НЕ отдельная сущность — роль-владелец блока. Без BOM. |
| manual/product_owners_ref.csv | product, product_raw, owner_name, eol_email | eol_email → Человек (владелец продукта); product → Продукт; product_raw → сырое имя | 13 строк; eol_email 100%, owner_name 100% | ЕОЛ на уровне продукта (владельцы 13 продуктов→несколько человек). Кириллическая коллизия 'СS'/'CS'. owner_name свободный формат. Без BOM. |
| data-room/team/track_owners.csv | Контур (владелец трека / ЕОЛ-кандидат), Треки, Продукты, Gold-людей, Люди Gold | Контур → target_team.Владелец трека; Люди Gold → Человек; Продукты → Продукт | 5 строк; все 5 колонок 100% | ЕОЛ выражен как «Контур (владелец трека / ЕОЛ-кандидат)» — 4-7 уникальных владельцев. Списки в ячейках (' · ', '; '). BOM в первом заголовке. |
| monitor-gd/ceo_directives.csv | id, поручение, owner_email, due, block, depends_on, override, override_reason, client, product | owner_email → Человек (ответственный); product → Продукт; client → Клиент; depends_on → ceo_directives.id | 15 строк; owner_email 87% (пусто у D14, D15), block 100%, product 100% | owner_email = ответственный за поручение (близко к ЕОЛ, но по директиве, не по блоку). Пусто у 2 директив. Без BOM. |

### Цель — ⚙ Уточнить · _Справочник · → порождает Инициативу (генезис goal)_

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| manual/goals.csv | goal_id, title, description, critical, product, generation, owner_hint | goal_id → directives_extra.id (общие D01..D15); product → Продукт; owner_hint → Человек | 15 строк; goal_id 100%, title 100%, critical 100%, product 100%, owner_hint 86.7%, generation 0% | Основной источник: 15 целей D01-D15 от ГД. generation пусто 0%; title и description дублируются; critical bool (true у D03, D06); owner_hint пуст у D14, D15. Нет OKR/метрик. Без BOM. |
| monitor-gd/ceo_directives.csv | id, поручение, owner_email, due, block, depends_on, override, override_reason, client, product | id → goals.goal_id (D01..D15, директива↔цель); owner_email → Человек; product → Продукт; client → Клиент | 15 строк; id 100%, поручение 100%, due 100%, block 100%, client 100%, product 100%, owner_email 87% | Пара к goals (те же D-коды). Директива уточняет/порождает Цель — по канону разводятся на две сущности. depends_on 13%, override 20%. Без BOM. |
| manual/directives_extra.csv | id, block, due, depends_on, override, override_reason, client | id → goals.goal_id (доп.атрибуты к целям); block → blocks.block_name; depends_on → directives_extra.id; client → Клиент | 15 строк; id 100%, block 100%, due 100%, client 100%, depends_on 13.3%, override 20% | Доп.атрибуты целей/директив D01..D15 (due, зависимости, override). override_reason на 100% коррелирует с override (оба у 3 строк). Без BOM. |
| eva/eva_tasks.csv | code, project_code, project_name, type, status, status_type, resolution, responsible, responsible_login, author, author_login, priority, story_points, epic, parent, created, updated, closed, deadline, plan_start, plan_end, tags, name | code → Задача (PK); project_code → eva_projects.code; responsible_login/author_login → Человек (email); epic/parent → eva_tasks.code | 6634 строки; code 100%, type 100%, status 100%, responsible_login 71%, epic 29%, story_points 48% | Не справочник целей — трекер задач EVA, привлекается для трассировки цель→исполнение. Тип type включает Goal среди прочих; связь с целями D01-D15 напрямую не проставлена. BOM. |

## Прямая выгрузка (16)

### Директива — ✓ Готово · _Прямая выгрузка_ · → таблица MeetingDecision

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| monitor-gd/ceo_directives.csv | id, поручение, owner_email, due, block, depends_on, override, override_reason, client, product | owner_email→Человек, product→Продукт, client→Клиент, depends_on→ceo_directives.id (self) | 15 строк; owner_email 87%, due 100%, block/client/product 100%, depends_on 13%, override/override_reason 20% | Чистый CSV, без BOM, заголовок стр.1. id=D01..D15. Зеркалит MeetingDecision 1-в-1 (встреча 2026-07-02). owner_email пуст у D14/D15 |
| manual/directives_extra.csv | id, block, due, depends_on, override, override_reason, client | id→goals.goal_id (D-коды), block→blocks.block_name, depends_on→self (через ';'), client→Клиент | 15 строк; id/block/due/client 100%, depends_on 13.3%, override/override_reason 20% | Плоская, без BOM. Доп.атрибуты к целям (те же D01..D15). depends_on только у D04 |
| monitor-gd/monitor_gd_real_brief.md | — | — | нет профиля | Бриф встречи ГД (нарратив), не таблица — источник контекста/снимка |

### Клиент — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Client

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| crm/crm_deals.csv | taskid, стадия, создана, в_работу_до, завершена, закрыта, просрочена, автор, наименование, клиент, заказчик_юрлицо, продукт, вид_сделки, сумма, сумма_iva_с_ндс, потенциал, потенц_маржинальность, валюта, дата_перехода_в_продажу, …, пресейл, подразделение, причина_отказа, тема | клиент→Клиент (НАЗВАНИЕ+ИНН склеены), автор→Человек, продукт→Продукт | 2178 строк; клиент 99.7%, taskid 100% | BOM. Колонка клиент склеивает название+ИНН — нужен парсинг ИНН для нормализации/дедупа |
| crm/top_projects.csv | Т п/п, Заказчик, Продукт , (MCU)/(ONE)/(MAIL)/(CS)/(TERRA)/(LARGO), АСД / СУММА, Статус, Этап, Менеджер, Бюджет сделки, … Ведущий пресейл, Комментарии | Заказчик→Клиент, Менеджер/Ведущий пресейл→Человек | 40 проектов / 61 непустая строка; Заказчик 57.4% | Report-style: двухстрочная шапка (21 кол.), продуктовая плюс-матрица в колонках 3-9, строки-продолжения без Т п/п |
| data-room/support/crm_open_requests.csv | Заявка, Тип, Статус, Группа статуса, Дата создания, Возраст дней, Код клиента, Клиент, Контракт, Тариф, Продукт, Семейство продукта, Приоритет, Исполнитель, Все исполнители, Placeholder/test, Заголовок, Комментарий клиента, Решение | Код клиента→Клиент, Клиент→Клиент | 782 строки; Код клиента 100%, Клиент 100% | BOM, 2 строки баннера, заголовок стр.3. Даёт SD-код клиента для сшивки CRM↔поддержка |
| data-room/support/critical_cases.csv | Заявка, Клиент, Продукт, Статус, Возраст дней, Заголовок, Комментарий | Клиент→Клиент, Продукт→Продукт | 6 строк; Клиент 100% | Курируемая подвыборка (6 кейсов, все IVA MCU), не полный список |

### Сделка — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Deal / SalesInitiative

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| crm/crm_deals.csv | taskid, стадия, создана, в_работу_до, завершена, закрыта, просрочена, автор, клиент, продукт, сумма_iva_с_ндс, потенциал, дата_перехода_в_продажу, тема, … | taskid→Сделка (PK), клиент→Клиент, продукт→Продукт, автор→Человек | 2178 строк; taskid/стадия 100%, сумма_iva_с_ндс 100%, продукт 34.9%, клиент 99.7% | BOM. Пустые колонки: наименование, заказчик_юрлицо, вид_сделки, сумма, валюта, пресейл, подразделение (0%). Продукт 34% — блокер привязки выручки к продукту |
| crm/top_projects_pipeline.csv | N п/п, Номер сделки, Продукт, Заказчик, Продукт , (MCU)/(ONE)/(MAIL)/(CS)/(TERRA)/(LARGO)/(SBC), Бюджет сделки, Ключевой проект/нет, Статус, Менеджер, Требования, ФЛАГИ | Номер сделки→crm_deals.taskid, Заказчик→Клиент, Менеджер→Человек, Продукт→Продукт | 65 проектов / 107 непустых строк; Номер сделки 80.4%, Заказчик 60.7%, Бюджет 84.1%, Статус 62.6%, Менеджер 97.2% | Report-style: двухстрочная шапка (17 кол.), продуктовая плюс-матрица кол.5-11, строки-продолжения без N п/п. ФЛАГИ почти пуста |
| crm/top_projects.csv | Т п/п, Заказчик, Продукт , (MCU)/(ONE)/(MAIL)/(CS)/(TERRA)/(LARGO), АСД / СУММА, Статус, Этап, Менеджер, Бюджет сделки, … Комментарии | Заказчик→Клиент, Менеджер→Человек, Продукт→Продукт | 40 проектов / 61 непустая строка; Статус 86.9%, Этап 83.6%, Менеджер 85.2%, АСД/СУММА 39.3%, Бюджет сделки 0% | Report-style. Сумма лежит в АСД/СУММА, а «Бюджет сделки» пуст (100%). Много нарративных колонок |

### Контракт — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Contract

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| data-room/support/crm_open_requests.csv | Заявка, Тип, Статус, Группа статуса, Дата создания, Возраст дней, Код клиента, Клиент, Контракт, Тариф, Продукт, Семейство продукта, Приоритет, Исполнитель, Все исполнители, Placeholder/test, Заголовок, Комментарий клиента, Решение | Контракт→Контракт (172 уник.), Код клиента/Клиент→Клиент, Тариф→Тариф | 782 строки; Тариф 100%, Код клиента/Клиент 100% (колонка Контракт в fill-профиле не оценена) | BOM, заголовок стр.3. Единственный носитель контрактов поддержки. Тариф: Стандартный/Премиальный/Расширенный |
| service/sd_tasks.csv | taskid, категория, подкатегория, статус, приоритет, создана, срок, завершена, закрыта, просрочена, percent, автор, тема | taskid→Задача SD (PK); прямой колонки «контракт» НЕТ | 42277 строк; taskid/категория/статус 100% | Отдельной колонки контракта нет — связь с контрактом только через crm_open_requests. Вторичный источник. Упущены лицензионные/продажные договоры |

### Задача — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Task (+ Signal)

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| eva/eva_tasks.csv | code, project_code, project_name, type, status, status_type, resolution, responsible, responsible_login, author, author_login, priority, story_points, epic, parent, created, updated, closed, deadline, plan_start, plan_end, tags, name | code→Задача (PK), project_code→eva_projects.code, responsible_login/author_login→Человек (email), epic→Эпик, parent→Задача | 6634 строки; code 100%, epic 29%, parent 46%, story_points 48%, responsible 71%, priority 25% | BOM. epic_link дырявый (29%) → добить LLM-судьёй. Логины — корпоративные email |
| jira/jira_issues.csv | key, project, type, status, assignee, assignee_email, reporter, priority, epic_link, story_points, created, updated, duedate, labels, summary | key→Задача (PK), project→jira_manifest.project, assignee_email→Человек, epic_link→Эпик | 3071 строка (усечён ~300/проект); key/project/type/status/priority 100%, assignee_email 44.1%, epic_link 7%, story_points 0%, labels 16.6% | BOM. epic_link 7% — иерархия почти не восстановима |
| jira/ivaone/tasks_open_ivaone.csv | key, type, status, duedate, epic_link, assignee, summary | key→Задача, epic_link→Эпик, assignee→Человек (ФИО, без email) | 1409 строк; key/type/status 100%, assignee 60%, epic_link 18.1%, duedate 4.2% | BOM. Только IVAONE open. Нет колонки email — assignee только ФИО (кириллица, метка [X]) |
| service/sd_tasks.csv | taskid, категория, подкатегория, статус, приоритет, создана, срок, завершена, закрыта, просрочена, percent, автор, тема | taskid→Задача SD (PK), категория→sd_summary, автор→Человек (имя) | 42277 строк; taskid/категория/статус/приоритет 100%, тема 97.9%, срок 3.4%, завершена 4.4% | BOM. тема — многострочный HTML, нужен CSV-парсер. закрыта/просрочена — булевы (литерал False) |

### Эпик — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Epic

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| jira/jira_epics.csv | key, project, status, epic_name, updated, summary | key→Эпик (←jira_issues.epic_link), project→jira_manifest.project | 100 строк; все колонки 100% | BOM. Кросс-проектный реестр, вероятно усечён экспортом (exported=300/проект). epic_name часто дублирует summary |
| jira/ivaone/epics_ivaone.csv | key, status, duedate, assignee, assignee_email, resolutiondate, created, updated, summary | key→Эпик (←tasks_open_ivaone.epic_link), assignee_email→Человек | 67 строк; key/status/created/updated 100%, duedate 9%, assignee/assignee_email 4.5%, resolutiondate 0% | BOM. Только IVAONE-*. Пересекается с jira_epics — нужен дедуп в единое пространство |
| jira/monitor_ivaone_brief.md | — | — | нет профиля | Бриф-снимок IVAONE (нарратив/светофор), не таблица |

### Человек — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Person / PersonEmail

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| identity/identity_map.csv | person_email, ФИО, jira_login, team_track, team_product, team_role, team_status, team_dep, подразделение_gold, display_names, nicks, git_emails, git_names, commits, jira_projects, sources, conf_pages | person_email→Человек (мастер-ключ), ФИО→Человек, team_product→Продукт, jira_projects→Проект, git_emails→Коммит | 955 строк; person_email/ФИО/display_names/commits/sources/conf_pages 100%, jira_login 0%, git_emails/git_names 3.5%, team_* 11%, jira_projects 14.2% | BOM. Мастер-хаб идентичности (1 строка=человек). Мультизначные поля через ' \| '. jira_login пуст |
| manual/teams.csv | person_email, person_name, team, role, grade, manager_email, repos, jira_projects, track, product, level, stack | person_email→Человек (PK), manager_email→Человек (self, пусто), product→Продукт, jira_projects→Jira-проект | 227 строк; person_email/person_name/team 100%, role/grade/track/product/level 98.7%, jira_projects 43.6%, stack 26.9%, manager_email 0%, repos 0% | Без BOM. manager_email и repos пусты (0%). product — свободный текст, не совпадает с reference_products |
| sensitive/by_person.csv | person_email, ФИО, ставка_fte, статус_отбора, gold, трек, продукт, роль, левел, подразделение, компания | person_email→Человек, продукт→Продукт, трек→Трек | 292 строки; ФИО/ставка_fte/статус_отбора/gold/продукт 100%, person_email 79.5% | BOM. У ~21% нет email (напр. Иванова Наталья) — хрупкость ключа. Несёт FTE/gold/статус отбора |
| data-room/team/target_team.csv | ФИО, Тип, Стек, Левел, Роль, Должность, Подразделение, Track трансформации, Product трансформации, Режим участия в треке, Статус отбора, Gold, Класс Gold, Обоснование Gold, Владелец трека, Роль человека в ADP-loop | ФИО→Человек (ключ), Track→track_owners.Треки, Product→Продукт, Владелец трека→track_owners.Контур | 117 строк; ФИО/Роль/Должность/Track/Product/Gold 100%, Стек 70.9%, Класс Gold 26.5% | BOM. Только ФИО (не email) → join к Человеку по имени. Gold — бинарный 0/1 |

### Команда — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Team

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| manual/teams.csv | person_email, person_name, team, role, grade, manager_email, repos, jira_projects, track, product, level, stack | team/track (принадлежность), person_email→Человек, product→Продукт | 227 строк; team 100%, track 98.7%; team/track — сырая строка, не FK | Без BOM. Принадлежность не нормализована (нужен FK). track дублирует team |
| data-room/team/target_team.csv | ФИО, Тип, Стек, Левел, Роль, Должность, Подразделение, Track трансформации, Product трансформации, Режим участия, Статус отбора, Gold, Класс Gold, Обоснование Gold, Владелец трека, Роль в ADP-loop | Track трансформации→track_owners.Треки, Владелец трека→track_owners.Контур, Product→Продукт | 117 строк; Track/Product/Подразделение/Владелец трека 100% | BOM. Даёт трек+подразделение+владельца для нормализации команды |
| data-room/team/track_owners.csv | Контур (владелец трека / ЕОЛ-кандидат), Треки, Продукты, Gold-людей, Люди Gold | Контур→target_team.Владелец трека, Треки→target_team.Track, Продукты→Продукт (' · '), Люди Gold→Человек (ФИО ';') | 5 строк; все 100% | BOM. Справочник-агрегат контуров/треков. Продукты и Люди Gold — денормализованные списки в ячейке |

### Организация — ⚠ Переделать · _Прямая выгрузка_ · → таблицы Company + CompanyRole

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| sensitive/by_person.csv | person_email, ФИО, ставка_fte, статус_отбора, gold, трек, продукт, роль, левел, подразделение, компания | компания→Организация (работодатель, 5 юрлиц), person_email→Человек | 292 строки; ФИО 100% (колонка компания в fill-профиле не оценена, содержательные 100%) | BOM. Внутренние юрлица-работодатели (ИВА ПАО/АЙМЭЙЛ/ИВА360/НТЦ/ХАЙТЭК) |
| crm/crm_deals.csv | taskid, …, клиент, заказчик_юрлицо, продукт, сумма_iva_с_ндс, … | клиент→Организация-клиент (ИНН склеен), заказчик_юрлицо→Юрлицо (пусто) | 2178 строк; клиент 99.7%, заказчик_юрлицо 0% | BOM. Внешние клиент-юрлица с ИНН в строке клиент; отдельная колонка заказчик_юрлицо пуста |
| service/sd_tasks.csv | taskid, категория, подкатегория, статус, приоритет, создана, срок, завершена, закрыта, просрочена, percent, автор, тема | автор→Человек (имя); прямой колонки юрлица НЕТ | 42277 строк; все ключевые 100% | BOM. Отдельной колонки юрлица нет (Юр_Лица_1С — во внешнем справочнике). Нужна разметка CompanyRole (мы/клиент/партнёр) |

### Репозиторий — ⚙ Уточнить · _Прямая выгрузка_ · → таблица Repo

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| git/repos_manifest.csv | repo, gitlab_remote, product, generation, commits_exported, latest_commit_date | repo→Репозиторий (PK), product→Продукт | 8 строк; gitlab_remote/product/generation 100% | BOM. Манифест 8 экспортированных репо: связь repo→product + generation |
| git/repos_registry.csv | repo, namespace, продукт, gitlab_url, commits_total, authors, files, языки(файлов), first_commit, last_commit, commits_90d, branches, size_mb, top_committers | repo→Репозиторий (PK домена git), продукт→Продукт | 9 строк; продукт/gitlab_url/commits_total/first_commit/last_commit 100% | BOM. Кириллические заголовки. языки(файлов)/top_committers — многозначные строки ' ; '. Главный справочник репо |
| git/repo_activity_all.csv | repo, top_group, project_id, default_branch, last_activity, last_commit_date, last_commit_author, commit_count, contributors, branches, mr_open, mr_merged, mr_total | repo→Репозиторий, project_id→GitLab project, last_commit_author→Человек | 233 строки; top_group/project_id/commit_count/mr_open 100% | BOM. Агрегат активности по всем 233 репо. Привязка к продукту явна лишь у 8-9 |
| git/repo_activity.csv | repo, gitlab_path, project_id, default_branch, last_activity, commit_count, contributors, branches, mr_open, mr_merged, mr_total, last_commit_date, last_commit_author | repo→Репозиторий, project_id→GitLab project, last_commit_author→Человек | 8 строк; gitlab_path/project_id 100% | BOM. Курируемый short-list 8 целевых репо (gitlab_path вместо top_group) |

### Коммит — ✓ Готово · _Прямая выгрузка_ · → таблица Commit

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| git/all_commits.csv | repo, sha, date, author_name, author_email, jira_keys, subject | repo→Репозиторий, author_email→Человек, jira_keys→Задача Jira | ~216161 строк; author_email ~100%, jira_keys ~0.5% | BOM. Fill оценён по выборке. jira_keys почти всегда пусто. subject — свободный текст |
| git/commits.csv | repo, hash, author_email, author_name, date, branch, jira_keys, subject | repo→Репозиторий, author_email→Человек, jira_keys→Задача Jira | 24039 строк; author_email 100%, jira_keys 51.3%, branch 4.5% | BOM. Обогащённый срез с trackingом; jira_keys 51% (выше all_commits). Порядок колонок иной (hash) |
| git/merges.csv | repo, hash, author_email, date, mr_ref, source_branch, target_branch, jira_keys, subject | repo→Репозиторий, author_email→Человек, mr_ref→MR, jira_keys→Задача | 344 строки; author_email 100%, source/target_branch 54.4%, mr_ref 38.7%, jira_keys 11% | BOM. Merge-коммиты. Т1-ключ (коммит→задача) чистить whitelist-ом префиксов от шума CVE/ISO |

### Запрос на слияние (MR) — ⚙ Уточнить · _Прямая выгрузка_ · → таблица MergeRequest

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| git/merge_requests_all.csv | repo, mr, state, author, date, target_branch, title | repo→Репозиторий, author→Человек (GitLab-username), mr→MR (!NNN) | 1115 строк; mr/state/author/date/target_branch 100% | BOM. author — GitLab-username (не email) → join к Человеку по username. Нет source_branch. Нет reviewers/approvals (нужен GitLab PAT) |
| git/merge_requests.csv | repo, mr, state, author, date, source_branch, target_branch, title | repo→Репозиторий, author→Человек (username), mr→MR | 139 строк; date/source_branch/target_branch 100% | BOM. Срез с доп. source_branch (вероятно активные/недавние MR) |

### Экономика — ✓ Готово · _Прямая выгрузка_ · → таблица ProductEconomics

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| data-room/economics/product_economics_37m.csv | Продукт, ФОТ млн руб., Производственные млн руб., Итого затраты млн руб., Выручка млн руб.\nпайп продаж, Прибыль млн руб., Маржа, Решение, (хвост) | Продукт→Продукт | 10 строк; все колонки 1-8 100% | Пропустить строку-баннер стр.1, заголовок стр.2; перенос строки в заголовке выручки; ниже нарратив-секции. 9-я хвостовая колонка ~10% |
| data-room/economics/portfolio_decisions.csv | Продукт, ФОТ, Производственные, Итого затраты, Выручка, Прибыль, Маржа, Решение | Продукт→Продукт | 10 строк; все 100% | Report-style: несколько таблиц бок-о-бок + нарратив. Заголовок левой таблицы стр.3. Цифры совпадают с 37m, отличается колонка Решение |
| data-room/economics/optimistic_scenario.csv | Продукт, Затраты (млн), Выручка оптимистичный пайп (млн), Прибыль (млн), Маржа, Решение, (хвост) | Продукт→Продукт | 10 строк; все 100% | BOM, заголовок стр.1; ниже 3 строки нарратива пропустить. Оптимистичный сценарий выручки (другие цифры) |
| sensitive/fot_by_product.csv | Проект, Тип, янв, фев, мар, апр, май, июн, июл, авг, сен, Общий итог (ПЛАН/ФОТ 1С); правее — Тип, Проект JIRA, янв..окт, Общий итог (ФАКТ JIRA) | Проект→Продукт | ~164 строки (левая сводная); Проект 17.7%, Тип 50.6%, Общий итог 42.7% | Report-style Excel-pivot: 6 строк баннера, заголовок стр.10; несколько сводных бок-о-бок/стопкой. Низкий fill — артефакт pivot. Не приводить к плоской схеме |

### Обращение поддержки — ✓ Готово · _Прямая выгрузка_ · → таблица SD_Ticket (+ Signal)

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| service/sd_tasks.csv | taskid, категория, подкатегория, статус, приоритет, создана, срок, завершена, закрыта, просрочена, percent, автор, тема | taskid→Обращение (PK), категория/подкатегория→sd_summary, статус→справочник, автор→Человек | 42277 строк; taskid/категория/статус/приоритет 100%, тема 97.9%, срок 3.4%, завершена 4.4%, percent 29.8% | BOM. тема — многострочный HTML, нужен CSV-парсер. закрыта/просрочена — булевы (литерал False). Основной поток-сырьё для Signal/Need |
| data-room/support/crm_open_requests.csv | Заявка, Тип, Статус, Группа статуса, Дата создания, Возраст дней, Код клиента, Клиент, Контракт, Тариф, Продукт, Семейство продукта, Приоритет, Исполнитель, Все исполнители, Placeholder/test, Заголовок, Комментарий клиента, Решение | Заявка→Обращение (PK, FR./SR.*), Код клиента/Клиент→Клиент, Продукт/Семейство→Продукт, Контракт→Контракт, Исполнитель→Человек | 782 строки; Заявка/Статус/Клиент/Продукт 100%, Тип/Приоритет 85.9%, Исполнитель 86.1%, Решение 2.9% | BOM, 2 строки баннера, заголовок стр.3. FR-заявки (402) — прямые кандидаты в Need/Requirement |
| data-room/support/critical_cases.csv | Заявка, Клиент, Продукт, Статус, Возраст дней, Заголовок, Комментарий | Заявка→Обращение, Клиент→Клиент, Продукт→Продукт | 6 строк; все 100% | BOM, заголовок стр.3, футер-нарратив пропустить. Курируемые 6 острых кейсов (все IVA MCU) |
| service/sd_summary.csv | категория, статус, обращений | категория→sd_tasks.категория, статус→sd_tasks.статус | 52 строки; все 100% | BOM. Агрегат (счётчик по паре категория×статус), не сущностные строки; сумма = числу задач sd_tasks |

### Встреча + Снимок — ✓ Готово · _Прямая выгрузка_ · → таблицы Meeting / Snapshot

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| monitor-gd/monitor_gd_real_brief.md | — | — | нет профиля | Бриф-снимок встречи ГД (2026-07-03, первый, diff пуст) — нарратив, не таблица |
| jira/monitor_ivaone_brief.md | — | — | нет профиля | Бриф-снимок IVAONE — нарратив, не таблица |
| monitor-gd/ceo_directives.csv | id, поручение, owner_email, due, block, depends_on, override, override_reason, client, product | id→Директива (D01..D15) встречи 2026-07-02 | 15 строк; id/поручение/due/block/client/product 100%, owner_email 87% | Чистый CSV, без BOM. Содержимое встречи (поручения). Схема готова, но истории для динамики нет (1 встреча) |

### Документ Wiki — ⚙ Уточнить · _Прямая выгрузка_ · → WikiPage/Space (context-слой, не первокласс ORM)

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| confluence/pages_index.csv | space, space_name, page_id, title, updated, author, url | space→spaces.key, author→Человек (по имени, messy), page_id→Страница | 3154 строки; все 7 колонок 100% | BOM (utf-8-sig). Каталог полон, но ТЕЛА только 3/3154. author вида 'Unknown User (m.sudakov)' — join к Человеку ненадёжен |
| wiki/pages_index.csv | space, space_name, page_id, title, updated, author, url | space→spaces.key, author→Человек (ФИО, не email), page_id→Страница | 28990 строк; все 7 колонок 100% (fill по выборке) | BOM. Более полный каталог вики. author — ФИО кириллицей, не email |
| confluence/spaces.csv | key, name, type, pages | key→pages_index.space | 117 строк; все 4 колонки 100% | BOM. Справочник пространств. type='global' у всех (константа). У 12 пространств pages=0. Строка key='1' — тестовая |
| wiki/spaces.csv | key, name, type, pages | key→pages_index.space | 117 строк; все 4 колонки 100% | BOM. Справочник пространств вики (дублирует space_name в pages_index) |

## Дериват по джойну (3)

### Сигнал (Signal) — ✓ Готово · _Дериват по джойну_

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| jira/jira_issues.csv | key, project, type, status, assignee, assignee_email, reporter, priority, epic_link, story_points, created, updated, duedate, labels, summary | key → Задача/Issue (PK); project → jira_manifest.project; assignee_email → Человек (email); epic_link → Эпик | 3071 строк; key 100%, assignee_email 44.1%, epic_link 7%, story_points 0%, duedate 0.7% | Центральная таблица задач всех проектов, усечённый экспорт (~300/проект). Signal.source=jira, external_id=key. BOM в key, статусы/тип/priority кириллица |
| eva/eva_tasks.csv | code, project_code, project_name, type, status, status_type, resolution, responsible, responsible_login, author, author_login, priority, story_points, epic, parent, created, updated, closed, deadline, plan_start, plan_end, tags, name | code → Задача (PK, ←eva_task_links); project_code → eva_projects.code; responsible_login/author_login → Человек (email); epic/parent → eva_tasks.code | 6634 строк; code 100%, project_code 97%, responsible_login 71%, story_points 48%, epic 29% | Основная таблица трекера (Jira-подобная), 23 колонки. Даты ISO8601 +03:00. Логины — корп. email |
| git/all_commits.csv | repo, sha, date, author_name, author_email, jira_keys, subject | repo → Репозиторий; author_email → Человек (email); jira_keys → Задача Jira | ~216161 строк; author_email ~100%, jira_keys ~0.5% (оценка по выборке) | Полный экспорт коммитов. jira_keys почти всегда пуст (шум). subject — свободный текст с запятыми |
| crm/crm_deals.csv | taskid, стадия, создана, …, продукт, вид_сделки, сумма, сумма_iva_с_ндс, потенциал, валюта, …, автор, наименование, клиент, …, тема | taskid → Сделка (PK); клиент → Клиент (название+ИНН склеены); продукт → Продукт; автор → Человек (ФИО) | 2178 строк; taskid 100%, сумма_iva_с_ндс 100%, продукт 34.9%, клиент 99.7%; ряд колонок пусты 0% (наименование, вид_сделки, сумма, валюта, пресейл, подразделение) | Плоская таблица воронки CRM. Деньги в научном формате. Клиент требует парсинга ИНН |

_Технический ETL-слой (нормализованное событие) — сшивка разнородного сырья, в витринах ГД не показывается. Сырья с избытком; как нормализованные Signal-строки пока не материализовано._

### Зависимость — ✓ Готово · _Дериват по джойну_ · → граф блокировок (TaskDependency)

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| jira/jira_issue_links.csv | source_key, source_project, source_type, source_status, link_type, link_label, target_key, target_status, target_summary, source_duedate, source_summary | source_key → jira_issues.key (источник); target_key → jira_issues.key (цель); source_project → jira_manifest.project | 5512 строк; source_key/target_key/link_type/link_label 100%, source_duedate 4.4% | Рёбра графа: пара source→target с типом. link_type латиница (Blocks), link_label дублирует нижним регистром (blocks). Одна задача блокирует несколько |
| eva/eva_task_links.csv | source_code, source_project, link_type, target_code | source_code → eva_tasks.code (источник); target_code → eva_tasks.code (цель); source_project → eva_projects.code | 2332 строк; все колонки 100% | link_type ровно 2 значения поровну: affects=1166, depends_on=1166 (каждая связь продублирована в двух типах) |
| jira/active_blockers.csv | blocker_key, project, status, blocked_open_count, due, blocked_keys, summary | blocker_key → jira_issues.key; blocked_keys → jira_issues.key (список через пробел); project → jira_manifest.project | 220 строк; blocker_key/project/status/blocked_open_count/blocked_keys 100%, due 10% | blocked_keys — несколько ключей через пробел (нужен split). status кириллица (Сделать, Запланировано) |
| manual/directives_extra.csv | id, block, due, depends_on, override, override_reason, client | id → goals.goal_id (D01..D15); depends_on → directives_extra.id (self-ref, через ';'); block → blocks.block_name; client → Клиент | 15 строк; id/block/due/client 100%, depends_on 13.3%, override 20% | depends_on редкий (только D04), значения — список через ';'. При непустом override заполнен override_reason (100% коррелируют). Здесь даёт зависимости между директивами/целями |

_Граф блокировок (5512+2332 рёбра задач) — валиден и полезен. ORM Dependency — между Initiative, task-граф напрямую не мэтчится. Для витрины ГД выделять КРОСС-командные/продуктовые блокеры._

### Компонент продукта — ⚙ Уточнить · _Дериват по джойну_

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| topology-review.md | — (md-файл, не в профилях): по концепту — component_id/name, product, block-id (топология), path/профиль (node_ts/cpp_cmake/python/java), score/verdict (keep/drop/merge), evidence (repo://+scip://), trust_tier/reviewer_verdict | Компонент → Продукт (не привязан); Репозиторий → Компонент (декомпозиция, есть); Requirement → Компонент (по-компоненту) | нет профиля | Топология-блоки по 8 репо (iva-one 17, kmp 34, ivcs 11, jump 16 …). Указан в provenance трижды (3 источника) |
| topology-review.md | — (см. выше) | — (см. выше) | нет профиля | Дубль-источник в provenance («Файлы data» перечисляет topology-review.md ×3) |
| topology-review.md | — (см. выше) | — (см. выше) | нет профиля | Дубль-источник в provenance |
| IVAADP_131664812.md | — (md-файл, не в профилях) | Компонент → Продукт / архитектурный контекст ADP | нет профиля | Архитектурный разбор ADP (тело страницы), не структурированный CSV |

_Топология-блок ≠ продуктовый модуль. Достоверность инференс (trust_tier=auto-low, needs-review). Привязать продукт/владельца и поднять достоверность (нужен reviewer-approve). Ни один источник не входит в профилированный набор data/real — «нет профиля»._

## Вычленяемая (6)

### Потребность (Need) — ⚙ Уточнить · _Вычленяемая_ · → таблица NEED

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| data-room/support/crm_open_requests.csv | Заявка, Тип, Статус, Группа статуса, Дата создания, Возраст дней, Код клиента, Клиент, Контракт, Тариф, Продукт, Семейство продукта, Приоритет, Исполнитель, Все исполнители, Placeholder/test, Заголовок, Комментарий клиента, Решение | Заявка→Заявка (PK); Код клиента/Клиент→Клиент; Продукт, Семейство продукта→Продукт; Тариф→Тариф; Исполнитель→Человек | 782 строки; Заявка 100%, Тип 85.9%, Решение 2.9% | Заголовок 3 строки баннера, шапка на стр.3; тексты ссылаются на FR.*/SR.* ключи. FR-заявки (Тип=FR) — прямые кандидаты в Need |
| service/sd_tasks.csv | taskid, категория, подкатегория, статус, приоритет, создана, срок, завершена, закрыта, просрочена, percent, автор, тема | taskid→PK обращения; категория/подкатегория/статус→таксономия; автор→Человек (текстовое имя) | 42277 строк; taskid 100%, тема 97.9%, срок 3.4%, percent 29.8% | BOM; 'тема' — многострочный HTML (нужен CSV-парсер). Сырьё симптомов для кластеризации в потребность |
| IVAPROJECT_205489583.md | нет профиля | тело вики (видение PO/аналитиков) | нет профиля | Тела страниц Confluence не выгружены (3/3154) — нужны внешне для стратегических потребностей «сверху» |

_Дедуп по проблеме, не по решению. Источники расширить: проигранные сделки (стадия «Отменено» 602) + critical_cases + fr_backlog._

### Инициатива (Initiative) — ✓ Готово · _Вычленяемая_ · → таблица INITIATIVE

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| jira/jira_epics.csv | key, project, status, epic_name, updated, summary | key→Эпик (genesis-ребро); project→jira_manifest.project | 100 строк; все колонки 100% | Кросс-проектный реестр эпиков, вероятно усечён экспортом (300/проект). Эпик = кандидат-генезис Initiative |
| jira/ivaone/epics_ivaone.csv | key, status, duedate, assignee, assignee_email, resolutiondate, created, updated, summary | key→Эпик; assignee_email→Человек | 67 строк; key 100%, assignee/assignee_email 4.5%, duedate 9%, resolutiondate 0% | Срез эпиков только IVAONE; дедуп с jira_epics в единое пространство |
| manual/goals.csv | goal_id, title, description, critical, product, generation, owner_hint | goal_id→directives_extra.id; product→Продукт; owner_hint→Человек (email) | 15 строк; goal_id/product 100%, owner_hint 86.7%, generation 0% | Цели ГД D01-D15 — genesis 'а'. title/description часто дублируют друг друга |
| crm/crm_deals.csv | taskid, стадия, создана, …, автор, наименование, клиент, продукт, вид_сделки, сумма_iva_с_ндс, потенциал, дата_перехода_в_продажу, тема (+пустые) | taskid→Сделка; клиент→Клиент (склеен с ИНН); продукт→Продукт; автор→Человек | 2178 строк; taskid/стадия 100%, продукт 34.9%, сумма_iva_с_ндс 100% | 9 полностью пустых колонок; клиент несёт ИНН в строке. Сделка — genesis-источник Initiative |

_Хранить genesis_source на каждом ребре, источники не поглощать. «Золотой эпик» = сходятся цель+сделка+эпик._

### Релиз (Release) — ⚙ Уточнить · _Вычленяемая_ · → таблица RELEASE (extension, нет в ORM)

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| git/merge_requests_all.csv | repo, mr, state, author, date, target_branch, title | repo→Репозиторий; author→Человек (GitLab username, не email); mr→MR | 1115 строк; mr/state/author/date/target_branch 100% | target_branch=release/vX.Y.Z даёт ~40% релизных веток — инженерный след, не план релизов |
| git/merges.csv | repo, hash, author_email, date, mr_ref, source_branch, target_branch, jira_keys, subject | repo→Репозиторий; author_email→Человек; mr_ref→MR; jira_keys→Задача | 344 строки; author_email 100%, source/target_branch 54.4%, mr_ref 38.7%, jira_keys 11% | Merge-коммиты; branch парсится из сообщения merge не всегда |
| git/repo_activity_all.csv | repo, top_group, project_id, default_branch, last_activity, last_commit_date, last_commit_author, commit_count, contributors, branches, mr_open, mr_merged, mr_total | repo→Репозиторий; project_id→GitLab project; last_commit_author→Человек | 233 строки; все ключевые метрики 100% | Агрегат по репо; вехи/даты активности, не список версий |
| jira/jira_issues.csv | key, project, type, status, assignee, assignee_email, reporter, priority, epic_link, story_points, created, updated, duedate, labels, summary | key→Задача; project→jira_manifest; assignee_email→Человек; epic_link→Эпик | 3071 строка; key/project 100%, labels 16.6%, duedate 0.7%, story_points 0% | Метки версий только в labels (напр. on_release_is_ok); нет колонки fixVersion |

_Нужен источник fixVersion / release-plan. До выгрузки — держать релиз как веху-строку, привязанную к продукту+поколению (Поколение 1:N Релиз)._

### Требование (Requirement) — ⚙ Уточнить · _Вычленяемая_ · → таблица REQUIREMENT

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| IVAPROJECT_205489583.md | нет профиля | тело вики Confluence (PRD/требования, канонический источник) | нет профиля | Тела требований почти не выгружены (3/3154); Confluence QA-таблицы сплющены в MD, парсинг ненадёжен |
| IVAPROJECT_205489587.md | нет профиля | тело вики Confluence | нет профиля | Нужны внешне (Confluence-доступ); несёт target_generation + гипотезу эффекта |
| jira/ivaone/tasks_open_ivaone.csv | key, type, status, duedate, epic_link, assignee, summary | key→Задача; epic_link→Эпик; assignee→Человек (ФИО, без email) | 1409 строк; key/type/status 100%, assignee 60%, epic_link 18.1%, duedate 4.2% | Jira type=Feature (~341) — Requirement-уровень. Нет колонки assignee_email |
| data-room/support/crm_open_requests.csv | Заявка, Тип, Статус, Группа статуса, Дата создания, Возраст дней, Код клиента, Клиент, Контракт, Тариф, Продукт, Семейство продукта, Приоритет, Исполнитель, Все исполнители, Placeholder/test, Заголовок, Комментарий клиента, Решение | Заявка→Заявка; Клиент/Код клиента→Клиент; Продукт/Семейство продукта→Продукт; Тариф→Тариф | 782 строки; Заявка 100%, Тип 85.9%, Решение 2.9% | FR-заявки (402) — кандидаты Requirement; шапка на стр.3 |

_Разные уровни склеены (PRD / Jira-фича / FR-заявка) — свести гранулярность; канонический источник — тела вики._

### Соответствие RN/KMP (ConformanceRow) — ⚠ Переделать · _Вычленяемая_ · → таблица CONFORMANCE_ROW (трек C1)

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| models.py | нет профиля (ORM-схема, не данные) | component/requirement→Компонент/Требование; generation→Поколение (срез RN/KMP) | нет профиля / нет данных | Источник data/arch/generations/1.5/conformance_matrix_1.5.csv вне профилированного набора data/real |

_Нет данных в выгрузке + асимметрия SCIP-индексов KMP<RN. Требует внешней матрицы соответствия 1.5 и фикса индексов до показа CPO — иначе решение План А/Б поедет из-за дырки в данных._

### Сервис (Service) — ⚙ Уточнить · _Вычленяемая_ · → таблица SERVICE

| Файл | Реальные колонки | Ключи связи (join) | Заполненность | Примечания |
|---|---|---|---|---|
| eva/eva_projects.csv | code, name, total, exported | code→eva_tasks.project_code (Проект); прокси Сервис ← EVA-проект | 25 строк; code/total/exported 100%, name 96% | Справочника сервисов нет — сервис виден только косвенно через EVA-проекты. total/exported — счётчики задач |

_Завести явный справочник сервисов платформы — иначе связь инцидент→сервис (Initiative→Сервис) не построить._