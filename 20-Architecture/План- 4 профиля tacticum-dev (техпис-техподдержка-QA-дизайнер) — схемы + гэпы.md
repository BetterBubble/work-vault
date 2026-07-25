---
title: 'План: 4 профиля tacticum-dev (техпис/техподдержка/QA/дизайнер) — схемы + гэпы'
type: report
permalink: tacticum/20-architecture/plan-4-profilia-tacticum-dev-tekhpis-tekhpodderzhka-qa-dizainer-skhemy-gepy
status: current
tags:
- tacticum-dev
- profiles
- plan
- qa
- designer
- techwriter
- support
- profile-authoring
---

# План: 4 новых профиля в tacticum-dev (2026-07-20, на апрув пользователя)

Планер-агент по запросу руководителя. Модель профилей — см. [[Профили ролей в tacticum-dev — итоговая сводка (как устроено + как добавить)]]. Образец doc-профилей — **`templates/iva-fr-analyst`** (свежий безклоновый аналитик-документщик). **QA-автоматизатор — не профиль**, а команда `/extend-coverage` в dev-профилях.

## Профиль 1 — QA `iva-qa` 🟢 готовность высокая
- Схема: single-tier doc-профиль без клона/без depends_on, образец iva-fr-analyst. Дельта: agent `qa`, skills `test-case-authoring`+`regression-checklist`, команда `/start-testing`, MCP iva-mcp+helm-analyst(affected_systems)+tacticum-mcp. Артефакт: TC-n→UC-n + регресс-чеклист в TMS.
- Гэпы: (1) owner-решение — тест-кейсы Confluence vs Jira/Xray; (2) эталонные TC/шаблоны для few-shot; (3) зависимость от Ф3 report-экстракт (ТР-7), Taiga #682 (6/7 в работе). Дизайн готов в `process-analyst-dev-qa.md`.

## Профиль 2 — Дизайнер (mockup) 🟡 БЛОКЕР-концепт
- Схема: `depends_on:[tacticum-dev-base, tacticum-ui-base]`. Дельта greenfield: новый skill `mockup-authoring` (ГЕНЕРАЦИЯ макета из ТЗ — сейчас есть только `ui-mockup-match`=сверка), опц. Figma MCP (в профили не подключён), опц. `/start-mockup`.
- Гэпы: **⚠️ концепт дизайнера от руководителя не прислан** (артефакт? Figma? генерация-из-ТЗ vs сборка-из-DS) — блокер; корпус эталонных MOCKUPS; решение по Figma-транспорту. Не путать с продуктовой RBAC-ролью designer (ADR-0026/0028, Taiga #540, Д. Лебедев).

## Профиль 3 — Техпис `iva-tech-writer` 🔴 с нуля
- Схема: single-tier doc-профиль без клона, образец iva-fr-analyst (helm-analyst позиционирован «для техписов»). Дельта: agent `tech-writer`, skill `doc-authoring` (расщепить user-manual/release-notes/api-doc), `/start-doc`, MCP helm-analyst(docs_ask/arch_map)+iva-mcp+tacticum-mcp.
- Гэпы: (1) решение — какой артефакт (докум./релиз-ноты/API-справка); (2) корпус эталонных документов + глоссарий/стайлгайд; (3) эпика в Taiga нет.

## Профиль 4 — Техподдержка `iva-support` 🟠 с нуля, самый неопределённый
- Схема: single-tier профиль (триаж/расследование), образец логики ops-скилл `user-incident-investigation` + каркас iva-fr-analyst. Дельта: agent `support`, skills `incident-triage`+`troubleshooting-runbook`, `/start-incident`, MCP iva-mcp+helm-analyst(who_to_involve)+tacticum-mcp, опц. ssh-manager (логи, вопрос безопасности).
- Гэпы: (1) скоуп — конечные пользователи ИВА vs пользователи Tacticum-профилей (2 разных профиля); (2) корпус инцидентов/known-issues+раннбуки+SLA+шаблоны; (3) доступы к логам; (4) эпика нет.

## Очерёдность: QA → техпис → дизайнер(по концепту) → техподдержка.

## Кросс-катные гэпы
- Гейт апрува owner (`<id>-task.md` + mr.diaret@ya.ru) — обязателен, ни один не пройден.
- Эпики Taiga: QA/дизайн частично (#682/#646/#540); техпис/техподдержка — эпиков нет.
- Корпус эталонных артефактов отсутствует для всех doc-профилей (нужны few-shot образцы).
- Определение артефакта не зафиксировано (техпис/поддержка/дизайнер) — решение владельца.
- Тест-инварианты profile-authoring одинаковы (override ==объявленный, grep-гейт стеков, e2e_install golden, quickstart, .gitattributes).

## Вопросы (к пользователю/руководителю)
1. Дизайнер (руководитель, блокер): концепт — артефакт/Figma/генерация-vs-сборка?
2. Техпис: какой документ на выходе?
3. Техподдержка: скоуп (конечные ИВА vs пользователи профилей)? доступы к логам?
4. QA: тест-кейсы Confluence vs Jira/Xray?
5. Дать по 2-3 эталонных артефакта каждого типа (few-shot)?
6. Старт с QA?

## Ближайшие шаги (QA как первый, по profile-authoring)
Досье готово (process-analyst-dev-qa.md + свод) → написать `iva-qa-task.md` (образец `iva-ios-brownfield-task.md`) → **ГЕЙТ апрув owner** → US под #682 → worktree → coder/tester → сид+приёмка. Пилот «Отзыв и замена письма».

## Связано
- [[Профили ролей в tacticum-dev — итоговая сводка (как устроено + как добавить)]] · [[session-state]]
---

# Проработанный вариант профилей (заготовка к созвону с Diaret)

Предложение (не финал) по каждому профилю в формате «что умеет / стек-набор / на выходе», по образцу двух существующих наборов.

## Опора — анатомия двух наших наборов
- **Аналитик** (`iva-fr-analyst` / `iva-system-analyst`): single-tier, **1 read-only агент** (opus, tools Read/Write/Grep/Glob, без Edit по коду), skills `fr-authoring`+`api-contracts-discovery`, команды `/start-feature`+`/update-feature` (у fr) или `/prepare-tz`, MCP `helm-analyst`+`iva-mcp`+`tacticum-mcp`, stack `[]`, instruction packs (claude-md/codex-agents-md/codex-config-toml). **Выход — документ** (FR/ТЗ), публикация в Jira/Confluence через iva-mcp. Без клона репо.
- **Разработчик-go** (`iva-go-backend-brownfield`): **4 агента** (`tacticum-workflow` opus-оркестратор + `coder`/`tester`/`test-runner` sonnet), 8 команд (`start-task`/`approve-docs`/`run-implementation`/`run-coder`/`run-tester`/`run-test-runner`/`setup-code-intelligence`/`fix-bug`), ~15 skills, MCP `context7`+`serena`+`tacticum-mcp`+`helm-analyst`+`iva-read`, stack `[go]`+optional. **Выход — код + артефакты дизайн-фазы** BRD/ADR/PIN/TESTS. С клоном репо + KB-индексацией.

Итог: **QA / техпис / техподдержка → аналитик-класс** (1 агент, документ-артефакт, без клона). **Дизайнер → ближе к dev-классу** (стэк-профиль поверх `tacticum-ui-base`).

## 1) QA — `iva-qa`
- **Что умеет:** разобрать FR/постановку → выделить проверяемые пункты (UC-n/FT-n); написать тест-кейсы (позитив/негатив/границы) с трассировкой `TC-n→UC-n`; собрать чек-лист регрессии (из Impact-секции PIN + `affected_systems`); оформить вердикт (статус в Jira) и завести дефекты (баги TC→UC→FT). НЕ автоматизирует (это команда `/extend-coverage` в dev-профилях).
- **Стек:** depends_on — нет (самостоятельный, как аналитики). Агент `qa` (opus, read-only по коду). Skills: `test-case-authoring`, `regression-checklist`. Команда: `/start-testing <jira-key>`. MCP: `iva-mcp` (read FR / write кейсов+вердикта+багов), `helm-analyst` (`affected_systems`, `requirement_tests`, `related_tasks`), `tacticum-mcp`. Instruction packs как у аналитиков.
- **На выходе:** тест-кейсы `TC-n` в TMS + регресс-чеклист + Jira-вердикт и баги.
- **Готовность:** высокая (дизайн в `process-analyst-dev-qa.md`).

## 2) Техпис — `iva-tech-writer`
- **Что умеет:** собрать материал (публичная дока — `docs_ask`; Confluence/Jira-знания — `analyst_context`/`analyst_search`; C4-архитектура — `arch_map`); написать документ выбранного типа; выдержать глоссарий/стайлгайд и трассировку на требования; опубликовать на вики.
- **Стек:** depends_on — нет. Агент `tech-writer` (opus, read-only). Skills: `doc-authoring` (кандидат на расщепление: `user-manual-authoring` / `release-notes-authoring` / `api-doc-authoring`). Команда: `/start-doc <источник>`. MCP: `helm-analyst` (docs_ask/analyst_context/arch_map), `iva-mcp` (read+write вики), `tacticum-mcp` (kb_* для API-доков).
- **На выходе:** документ на Confluence/вики (тип — вопрос к обсуждению).
- **Готовность:** каркас дешёвый (копия аналитика), блок — определение артефакта + корпус образцов.

## 3) Техподдержка — `iva-support`
- **Что умеет:** принять обращение → классифицировать (тип, severity); найти похожие (`related_tasks`) и известные проблемы; локализовать (`affected_systems` — что затронуто, `who_to_involve` — кого звать); подготовить черновик ответа или эскалацию в Jira с трассировкой; (опц.) заглянуть в логи/health.
- **Стек:** depends_on — нет. Агент `support`. Skills: `incident-triage`, `troubleshooting-runbook`, опц. `known-issues-lookup`. Команда: `/start-incident <жалоба|jira-key>`. MCP: `iva-mcp` (read/write Jira), `helm-analyst` (analyst_search/related_tasks/affected_systems/who_to_involve/docs_ask), `tacticum-mcp`, опц. `ssh-manager` (логи — вопрос безопасности).
- **На выходе:** триажированный тикет / черновик ответа пользователю / эскалация в Jira.
- **Готовность:** с нуля, самый неопределённый (скоуп не задан).

## 4) Дизайнер — `iva-designer`
- **Что умеет (гипотеза, ждёт концепт):** прочитать ТЗ/FR + дизайн-систему (`design-system-discovery`, `design_*` токены); сгенерировать макет экранов (НОВЫЙ skill `mockup-authoring`); привязать к реальным токенам и свериться (`ui-mockup-match`); опц. выгрузить в Figma.
- **Стек:** depends_on: `[tacticum-dev-base, tacticum-ui-base]` (design-discovery/tokens/ui-mockup-match/playwright — из базы). Агент `designer` (или переиспользовать оркестратора). Новый skill `mockup-authoring`. Команда: `/start-mockup <ТЗ>`. MCP: `playwright` (из ui-base), `tacticum-mcp` (design_*), опц. `figma`.
- **На выходе:** MOCKUPS (+ опц. Figma-файл).
- **Готовность:** блокер — концепт руководителя (генерации макета в репо нет, только сверка).

# Вопросы к обсуждению с Diaret (созвон)
1. **Дизайнер (блокер):** артефакт на выходе — Figma-макет / код-прототип / токен-MOCKUPS? Нужна ли Figma-интеграция? Генерация «с нуля из ТЗ» или «сборка из компонентов дизайн-системы»?
2. **Техпис:** тип документа — руководство пользователя / релиз-ноты / API-справка / внутренние концепты? (определяет расщепление skills).
3. **Техподдержка:** скоуп — конечные пользователи продукта ИВА или пользователи Tacticum-профилей (внутренняя)? Нужны ли доступы к логам/мониторингу (ssh-manager)?
4. **QA:** тест-кейсы храним в Confluence или Jira/Xray? (открытый вопрос №1 процесса).
5. **Образцы:** дать по 2-3 эталонных артефакта каждого типа (тест-кейс, документ, обращение) для few-shot в skills.
6. **Порядок и эпики:** старт с QA? Завести эпики Taiga под техписа и техподдержку (сейчас их нет).
7. **Композиция:** держать QA/техпис/поддержку самостоятельными (как аналитики) или на `depends_on: [tacticum-dev-base]` ради общих kb_*/команд? (влияет на переиспользование).

# Предложения (наши рекомендации к созвону)
- **Старт с QA** — самый готовый, образец `iva-fr-analyst` свежий, эпик #682 активен; параллельно `/extend-coverage` в один dev-профиль.
- **Техпис** — вторым: каркас = копия аналитика; для MVP взять ОДИН тип документа (предлагаем **релиз-ноты из report-экстракта** — синергия с конвейером ТР-7), остальные типы — потом.
- **Техподдержка** — определить скоуп ДО проектирования; предлагаем начать с **внутренней** поддержки (пользователи Tacticum-профилей) — там уже есть ops-скилл `user-incident-investigation` как основа, меньше новых доступов.
- **Дизайнер** — держать в паузе до концепта; параллельно можно прототипировать `mockup-authoring` поверх готового `tacticum-ui-base`.
- **Общий приём:** все три doc-профиля клонируются с `iva-fr-analyst` (agent + instruction packs + iva-mcp-публикация) — экономия; отличие только в skills и команде входа.
- **Гейт:** для каждого — `<id>-task.md` на апрув Diaret до сборки (обязательная точка profile-authoring).
---

# Как реализуем + откуда профиль берёт данные (источники / MCP)

Сборка профиля = 90% готовые кусочки, 10% новое. Кусочки: агент (мозг) + skills (сценарии) + команды (вход) + **MCP (откуда данные и куда пишет)**. **Новый MCP нужен только если источника ещё нет.**

## Главный источник уже готов — `helm-analyst` (мы его сделали)
Отдаёт: Jira+Confluence (`analyst_search`/`analyst_context`/`related_tasks`), публичную доку (`docs_ask`), C4 (`arch_map`/`affected_systems`), требования+API (`requirement_coverage`/`api_registry_check`/`contract_check`), покрытие автотестами Allure (`requirement_tests`), кого звать (`who_to_involve`). Данные берёт из ингеста Jira/Confluence/Allure + Qdrant. **→ большинству профилей новый MCP не нужен, питаются отсюда.** Плюс `iva-mcp` (read/write Jira+Confluence), `tacticum-mcp` (код-контекст).

## По профилям
- **QA:** читает постановку (iva-mcp) + affected_systems/requirement_tests (helm-analyst); пишет кейсы/вердикт/баги (iva-mcp). Новый MCP — нет; опц. доучить iva-mcp писать в Xray, если TMS=Xray.
- **Техпис:** docs_ask/analyst_context/arch_map (helm-analyst) + код/API (tacticum-mcp); пишет документ (iva-mcp на вики). Новый MCP — нет.
- **Техподдержка:** обращение (iva-mcp) + related_tasks/who_to_involve (helm-analyst) + опц. логи (ssh-manager); пишет тикет/эскалацию (iva-mcp). **⚠️ ЕДИНСТВЕННЫЙ вероятный новый MCP/источник — support-KB** (база прошлых обращений + known-issues + раннбуков). Не нужен, если эти данные уже в индексируемом helm-analyst Jira; нужен, если отдельный корпус. Данные support-KB = прошлые тикеты + раннбуки + known-issues. Зависит от скоупа.
- **Дизайнер:** ТЗ (iva-mcp/helm-analyst) + токены design_* (tacticum-mcp) + опц. макеты (Figma MCP — есть в окружении, не подключён к профилям → подключить, не строить). Новый MCP — нет; недостающее = skill `mockup-authoring` (генерация) + подключение Figma.

## Что реально строим (сводка)
- Новый агент — во всех (лёгкий, копия с образца).
- Новые skills — QA 2, техпис 1-3, поддержка 2-3, дизайнер 1 (генерация макета).
- **Новый MCP/источник — почти наверняка только support-KB** (и то если данных нет в Jira). Остальное — подключение готовых (helm-analyst/iva-mcp/tacticum-mcp/figma). QA-Xray и Figma — доработка/подключение, не новый MCP.
- Тяжёлый сбор данных уже делает helm-analyst (наш).

## Доп-вопросы к Diaret (источники данных)
8. **Техподдержка:** где лежит история обращений и раннбуки — в индексируемом Jira (тогда support-KB не строим) или отдельный корпус (тогда строим источник)?
9. **QA:** если TMS=Xray — подтвердить доработку iva-mcp на запись в Xray.
10. **Дизайнер:** подключаем ли Figma MCP в профиль (транспорт/токен)?
---

# Источники данных (MCP) — что каждый умеет (подробно)

## helm-analyst (наш) — главный индекс знаний ИВА, ЧТЕНИЕ, 17 тулов
Поиск/контекст: `analyst_search`, `analyst_context`, `related_tasks`. Дока: `docs_ask` (RAG#1). C4: `arch_map`/`arch_container`/`affected_systems`. Требования: `requirement_coverage`/`nearest_spec`/`constraints`/`contradiction_check`/`gap_questions`. API-контракты: `api_registry_check` (REST), `contract_check` (JUMP). Тесты/люди: `requirement_tests` (Allure TestOps), `who_to_involve`, `effort_hint`.
Данные: ингест Jira(14 проектов)/Confluence/Allure/публичная дока/C4/API-реестры → Qdrant/Meili. → рабочая лошадь для QA/техписа/поддержки.

## iva-mcp — прямой доступ Jira+Confluence, чтение+ЗАПИСЬ
Пишет страницы/тикеты/комменты в живые системы. Отличие от helm-analyst: тот = умный индекс на чтение, iva-mcp = руки на запись. Схема: контекст из helm-analyst → публикация через iva-mcp. ⚠️ Точный список тулов iva-mcp не выверен (подключён как готовый mcp_server_spec).

## tacticum-mcp (mcp.tacticum.dev) — код репо + дизайн-система
kb_* (код целевого репо): `kb_get_task_context(query,top_k)` семантический поиск артефактов; `kb_get_code_context(...)` реальный код блока/класса по символу; `kb_verify_api_exists(api)` проверка существования API (анти-галлюцинация); `kb_get_nfr` НФР; `kb_get_coverage` doc_only/code_only/client_partial; `kb_get_block_compact` топология. (kb_discover — служебное подключение.)
design_* (дизайн-система): `design_list_systems`, `design_get_tokens`/`design_get_theme_tokens`, `design_resolve_token`.
Данные: KB-индекс репо (repomix) + каталог дизайн-систем. → dev/дизайн-профили; техпису — API-контекст; QA — проверка API.

## Прочие
context7 (дока внешних библиотек) · serena (навигация локального кода) · playwright (браузер/UI-снапшоты, сверка макета) · figma (макеты: use_figma/create_new_file/get_design_context/download_assets — есть в окружении, к профилям НЕ подключён) · ssh-manager (сервера/логи/health — для поддержки, вопрос безопасности) · iva-read (read-only срез ИВА).

## Матрица источник→профиль
- QA: helm-analyst + iva-mcp (○ tacticum-mcp).
- Техпис: helm-analyst + iva-mcp + tacticum-mcp(kb_ для API-док).
- Поддержка: helm-analyst + iva-mcp (○ ssh-manager; ⚠️ возможно новый support-KB).
- Дизайнер: tacticum-mcp(design_) + figma(подключить) + playwright + (○ helm-analyst/iva-mcp для ТЗ).
- Разработчик (образец): helm-analyst + tacticum-mcp(kb_) + context7 + serena.

Вывод: doc-профили = helm-analyst(читать) + iva-mcp(писать); tacticum-mcp где нужен код; дизайнер = design_*+figma. Новый источник строить почти наверняка только под поддержку.