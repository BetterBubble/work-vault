---
title: 'Helm — аудит данных data/real: что есть на входе + что деривится без дозапросов'
type: reference
permalink: tacticum/02-architecture/helm-audit-dannykh-data-real-chto-est-na-vkhode-chto-derivitsia-bez-dozaprosov
tags:
- helm
- control-tower
- data
- audit
- wave-1b
- competency-questions
---

Полный колоночный аудит `data/real/` (снимок 2026-07-03/04, реальные ИВА). Что есть на входе vs канон §13/3-таблицы, и что можно СФОРМИРОВАТЬ из имеющегося без внешних дозапросов. Сверять с [[control-tower-v02]] и `data/competency-questions.md`.

## Часть 1 — файлы vs входные потребности (§13)

- [reference] **git:** `git/commits.csv`(24039: repo·hash·author_email·date·branch·**jira_keys**·subject) ✅ · `merges.csv`(344, **mr_ref !NNN у 133**: +source/target branch·jira_keys·subject=заголовок) = реальный merged-MR срез · `repos_manifest`(8)/`repos_registry`(9: продукт·языки·top_committers) · `repomix/`(9 xml)·`scip/`(14, kmp-Kotlin+ivcs-Java НЕ покрыты)·`topology/`. #data
- [reference] **Jira:** `jira/jira_issues.csv`(3071: key·project·type·status·assignee(+email)·**epic_link**·SP·created·updated·duedate·labels·summary) — **epic_link заполнен 7%**, SP пуст · `jira_epics`(100) · `jira_issue_links`(**5512 blocks**-рёбер) · `active_blockers`(220) · `ivaone/`(эпики67+задачи1409, epic_link 18%) · `jira_manifest`(14: total vs exported=300/проект; STRAT/CEO/IVATR/IVACS/LRGWEB выгружены полностью). Свежесть: updated до 2026-07-03 (текущая активная работа). #data
- [reference] **CRM:** `crm/crm_deals.csv`(2178: taskid·стадия·клиент·заказчик_юрлицо·продукт·сумма_iva_с_ндс·потенциал·даты·просрочка) — **продукт 761/2178, стадии чистый энум** (Presale449·Оплачен442·ТКП170·Бюджетирование111·Знакомство243·Отменено602), колонка `сумма` пуста (сумма в `сумма_iva_с_ндс`) · `top_projects*`(pivot: заказчик×продукт×бюджет×статус×менеджер×вероятность). #data
- [reference] **Люди/штат:** `sensitive/by_person.csv`(292: person_email(232)·ФИО·**ставка_fte**·статус_отбора·gold·трек·продукт·роль·левел·подразделение — рублей НЕТ) · `data-room/team/target_team.csv`(117: ФИО·**Стек**(83)·**Левел**(117)·Роль·Должность·Track·Product·Gold·Владелец трека — CV-lite!) · `track_owners`(5) · `identity_map`(955: person_email·jira_login(пуст)·team_track(105)·git_emails·commits·jira_projects·sources). #data
- [reference] **Деньги:** `data-room/economics/product_economics_37m`(продукт→ФОТ→маржа→**Решение**)·`optimistic_scenario`·`portfolio_decisions` ✅ · `sensitive/fot_by_product.csv`(pivot, **есть рубли ФОТ по продукту**, merged-cells)·`team_detail`(pivot ПЛАН/ФАКТ-JIRA). #data
- [reference] **Поддержка/Confluence:** `service/sd_tasks.csv`(42277: категория·статус·приоритет·срок·просрочена·автор·тема — автор=displayname без email)·`sd_summary`(52) · `confluence/pages_index`(3154: space·title·updated·author·url)·`spaces`(117)·**тела только 3** (`bodies/*.json`). #data
- [reference] **Операторские входы (derived):** `manual/`(goals15·blocks9·teams29·refdata — собрано `scripts/derive_real_manual.py`)·`ceo_directives`(15)·`monitor-gd/2026-07-02/*.docx`(7 стенограмм/поручений)·`product_owners`(13). #data

## Часть 2 — что СФОРМИРУЕМ из имеющегося (без дозапросов)

- [decision] **Деривится сейчас (приоритет по ценности):** (1) **зависимости §5.4** ← `jira_issue_links`(5512)+`active_blockers` — вшить Initiative→depends_on (0 дозапросов); (2) **Т3 sales** ← `crm_deals` 761 сделка с продуктом + энум стадий→вероятность + `сумма_iva_с_ндс` + `в_работу_до`(дедлайн); (3) **полный состав команд** ← `by_person`(232 email) ⋈ `target_team`(117 +стек) ⋈ `identity_map`(955); (4) **CV-lite/fit** ← `target_team.Стек/Левел/Роль` (без CV-файла); (5) **цели расширенные** ← `ceo_directives`+Jira `STRAT`(5 эпиков)+`CEO`+докс; (6) **ФОТ по продукту** ← `fot_by_product`(спецпарсинг pivot); (7) **сигналы инцидентов** ← `sd_tasks`(42k); (8) **Т1 product-level** ← commits→repo→product(M3, готов). #derivable
- [decision] **Т2 (task→goal) — узкое место, не глубина Jira:** epic_link 7% → нужна **LLM-догадка task→goal** для задач без эпика (канон §7, не построена). Полная история Jira это НЕ чинит (epic_link всё равно 7%) → **полная история — низкий приоритет**; срез свежий (текущая работа). #wave-1b
- [decision] **PR/MR — не хардблокер:** `merges` = merged-MR срез (заголовок/автор/дата/ветка/ключ). Reviewers/approvals/открытые MR — через GitLab API; инстанс `git.hi-tech.org` достижим через VPS `adp_emb` (SSH id_ed25519_iva), **PAT создаётся**, не внешний дозапрос. #wave-1b

## Часть 3 — реально нельзя без внешнего (честно)
- [followup] **Зарплата в рублях по ФИО — нет и не будет** (оператор подтвердил); есть только FTE (by_person) + ФОТ-по-продукту (fot_by_product). → B2 (competency) не отвечаем; стоимость по ФИО — FTE-прокси. #sensitive
- [followup] **Тела Confluence** (кроме 3) → нужны для D3/D4 (RAG по докам: семантический поиск по содержимому страниц в knowledge_rag). Достаются тем же доступом к Confluence, что индекс. #wave-1b
- [followup] **SCIP kmp-Kotlin + ivcs-Java** — символьные вопросы по ним competency помечает как «честно отказать». #wave-2

## Сверка с competency-questions.md (golden-set §8.3 H)
- [reference] Это acceptance-чек-лист. Отвечаем: A1-A2,A4-A6·B3-B4,B8·C1-C2,C4-C5·D1-D2,D6. Ждёт вшивки (данные есть): **A3/D5** (`jira_issue_links`). Ждёт данных: **B2** (зп — нет), **D3/D4** (тела Confluence), C3/C6 (Т3-sales). #competency

## Отношения
- relates_to [[control-tower-v02]]
- relates_to [[2026-07-04 — Helm: реорг data + план 1a/1b (backend-only, Agent Team)]]
- relates_to [[explore-data-inventory]]
