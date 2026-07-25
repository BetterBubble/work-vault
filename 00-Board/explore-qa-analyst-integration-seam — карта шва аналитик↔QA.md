---
title: explore-qa-analyst-integration-seam — карта шва аналитик↔QA
type: note
permalink: tacticum/00-board/explore-qa-analyst-integration-seam-karta-shva-analitik-qa
status: draft
role: explorer
lead: lead-qa
created: 2026-07-23
tags:
- explore
- qa
- analyst
- integration
- seam
- tc-handoff
- lead-qa
- draft
---

# Карта шва аналитик ↔ QA (TC-handoff + общие части + зависимость от ADR)

Разведка кода (read-only) для QA-профиля. Скоуп «готовый профиль» = автоматизация + интеграция с аналитиком. Явно разделено: **[ШОВ В КОДЕ]** vs **[ПРОБЕЛ]** vs **[РЕШАЕТ ADR lead-arch]**. Модель взаимодействия профилей — ADR lead-arch, НЕ наш; здесь только картирование зависимости.

Worktree: аналитик — `~/tacticum/tacticum-dev/templates/`; QA — `~/tacticum/tacticum-dev-qa-kit/templates/`.

## 1. TC-handoff — где/в каком виде аналитик отдаёт TC и что читает QA

### Сторона аналитика (производит TC)
- **Скилл `tests-authoring`** (`iva-analysis-base/ingredients/skills/tests-authoring/SKILL.md`). Выход — **markdown-СТРАТЕГИЯ тест-кейсов уровня требования/контракта**: GIVEN/WHEN/THEN, каждый TC трассируется на UC (`Covers: FT-n/UC-n`), min 3 сценария на UC (позитив/негатив/граница). Явно НЕ тест-код стэка. Артефакт — `tests.md` в постановке (см. `agents/tacticum-workflow.md:97-99,134` — «tests.md — <N тест-кейсов, переиспользовано M Allure-TC>»).
- **Чтение существующего покрытия:** `mcp__helm-analyst__requirement_tests(requirement)` — сколько автотестов уже покрывают требование, зелёные ли (Allure TestOps, **read-only**). Аналитик обязан сперва переиспользовать существующие Allure-TC, потом дописывать (`tests-authoring/SKILL.md:21-26,72,82`).
- **Публикация:** Jira-декомпозиция + линк (analysis-base manifest: «Мост helm↔Jira на MVP — только ЛИНК»). Write-канал аналитика (ADR) — `iva-atlassian-write`.

### Сторона QA (потребляет TC)
Точки входа скилла `write-autotest` (`iva-qa-autotest-base/.../write-autotest/references/input-sources.md`), три источника → нормализуются в `.tasks/work/tc-{allure_id}/input.md`:
1. **Allure TestOps** (канон): URL TC → `python -m tools.testops get <N>` (CLI модуля `tools.testops` в репо one-web). Ключ конвейера — `allure_id`.
2. **CSV `.tcs`/`.csv`** — только fallback, если CLI TestOps недоступен (`csv-parsing.md`).
3. **Свободный текст** — генерит `desc-<unix_ts>` как id.
`jira-issue-autotest/SKILL.md:41,46` — связанные TC резолвятся через интеграцию **TestOps↔Jira** (`tools.testops cases <ISSUE> --unautomated`, AQL `issue="<ISSUE>"`); «в саму Jira не ходим».

### [ПРОБЕЛ — ключевой разрыв шва]
**Формат/место TC у аналитика и у QA НЕ совпадают, и нет код-пути между ними.**
- Аналитик производит TC как **markdown-стратегию** (GIVEN/WHEN/THEN, ключ `FT-n/UC-n`) в `tests.md` + Jira-линк.
- QA потребляет **структурированные кейсы Allure TestOps** (ключ `allure_id`) через `tools.testops`, либо CSV/URL.
- `requirement_tests` — только **read** (lookup покрытия), НЕ write-канал регистрации TC в TestOps.
- **Никакой скилл/CLI не переносит** markdown-TC аналитика в TestOps как зарегистрированные кейсы. Кто регистрирует TC в TestOps (человек? импорт TestOps?) — в коде не выражено. → Это разрыв «общего артефакта» из ADR §4.

## 2. «QA ревьюит-дополняет TC» (не авторинг) — есть ли флоу

### [ШОВ В КОДЕ — частично есть]
`write-autotest/references/tc-review.md` уже реализует ревью TC:
- **Пре-чек ТК** (шаг 2.5 write-autotest) по 4 критериям: полнота / непротиворечивость / избыточность / тестируемость. Исход: `tc_review=clean|issues|blocked`.
- **Протокол расхождения ТК↔AUT** (классы 1-3): текстовая неточность / ненаблюдаемое утверждение / поведенческое расхождение → красный тест, review-файл, вопрос человеку.
- **Review-файл** `.tasks/tc-review/TC-{allure_id}.md`, адресат — владелец ТК; передача — ручной TODO.
- Метрика `tc_truth_minutes` в ledger.

### [ПРОБЕЛ]
Существующий механизм — это **находки при автоматизации** (byproduct write-autotest, срабатывает когда QA уже пишет автотест), а НЕ то, что описал созвон 23.07:
- Нет **standalone-скилла «ревью фичи-реквеста / дополнение неочевидными сценариями ДО разработки»** (созвон: «тестировщикам — скилл ревью фичи-реквеста, добавить неочевидные сценарии»). Текущий tc-review ищет дефекты в тексте ТК, не **дополняет** набор новыми сценариями на ревью.
- Выход ревью — **локальный markdown-файл**, обратной публикации в TestOps/Jira нет (передача владельцу — ручной TODO).
- Кандидаты (НЕ проектирую, только обозначаю): (а) вынести/расширить `tc-review` в отдельный ingredient ревью-дополнения; (б) шаренный скилл в analysis-лейне (раз авторинг там). Где живёт ingredient — вопрос ownership → ADR §3.

## 3. «Дырка под общие части профилей»

### Механизм уже есть [ШОВ В КОДЕ]
Композиция `depends_on` (ADR-0056/0057). Обе роли тянут `tacticum-core-base`. ADR §3: «дырка» = существующий `depends_on`, без новой сущности.

Роли: `iva-role-analyst = [tacticum-core-base, iva-analysis-base]`; `iva-role-qa = [tacticum-core-base, iva-qa-autotest-base, iva-write-base]`.

### Что общего / что напрашивается вынести
- **`tests-authoring`** (analysis-лейн) — авторит аналитик, но QA нужен для чтения/ревью. Стэк-агностичен. ADR: живёт в analysis, шарится всеми QA-ролями.
- **Общий артефакт-хендофф** — требование/US + TC (ADR §4). Сейчас двоится по формату (см. пробел §1).
- **Покрытие двоится концептуально:** аналитик — `helm-analyst.requirement_coverage`/`requirement_tests` (Allure TestOps); QA — свой `coverage-ledger` (`iva-qa-autotest-base/craft-stack/shared/coverage-ledger.template.md`, `.tasks/coverage-ledger.md`). Две модели покрытия — кандидат на согласование.

### [Мех. дублирование — флаг, не дизайн]
Лейны `iva-analysis-base`, `tacticum-core-base`, `iva-write-base` физически лежат в ОБОИХ worktree и **различаются** (`diff` manifests → DIFFERS; core-base: manifest + kb-navigation отличаются). QA-kit — форк-worktree от `feat/iva-write-base`; его копии analysis/core могут дрейфовать от dev. Это worktree-дрейф, не дизайн-дублирование, но при мердже учесть.

## 4. Зависимость от ADR lead-arch (`adr-draft-model-vzaimodeistviia-profilei-...`, статус draft/proposed)

### Что ADR УЖЕ фиксирует (черновик)
- **§3 ownership:** QA = qa-autotest-лейн (исполнение/автоматизация: скиллы + субагенты) + **ревью-дополнение TC (НЕ авторинг с нуля)**. Переиспользуемые части = процесс-лейны single-owner, реюз композицией, не копией. «Дырка» = `depends_on`.
- **§4 пайплайн:** аналитик TC → в фичер-реквест ДО разработки; QA ревью-дополнение + автоматизация; статус покрытия из Allure через `requirement_tests`. Инвариант — интеграция без разрыва на общих артефактах (требование/US, TC).
- **§2 + §6.3 MCP-скоуп QA:** пресет над `helm-analyst` = `requirement_tests` (+`requirement_coverage`), узкий; отдельный QA-MCP **НЕ заводить** (архитектурно избыточен, закрывается requirement_tests + Allure/TestOps CLI в one-web).
- **§6.1 write-канал:** аналитик публикует через `iva-atlassian-write` (mcp-atlassian, личный PAT, interim, в проде).

### Что ADR оставляет ОТКРЫТЫМ / ждём от lead-arch
1. **Формат/место TC-хендоффа НЕ специфицирован.** ADR §4 говорит «общие артефакты (требование/US, TC)», но НЕ решает разрыв markdown-стратегия (аналитик) ↔ Allure TestOps структурированные кейсы (QA). Это главная открытая зависимость QA (см. пробел §1).
2. **Где живёт ingredient «ревью-дополнение TC»** (QA-лейн vs шаренный analysis-лейн) — ADR фиксирует что QA это делает, но не где сущность. Ownership → ADR §3, не финализировано.
3. **Единый механизм сборки (Солонко, три→один)** — §6.4 ОТКРЫТО, укладку сверяет Солонко.
4. **Write-канал QA** (обратно в TestOps/Jira для findings ревью) — ADR §3 «в проработке у lead-qa, тонкий/скоупленный». Расхождение-флаг: QA-роль сейчас несёт `iva-write-base` (iva-write MCP), а ADR аналитику даёт `iva-atlassian-write` — согласовать наименование/канал.

### Решения, которые QA НЕ может принять без ADR lead-arch
- Канонический формат/место TC-хендоффа (markdown vs TestOps vs Jira) — иначе разрыв §1 не закрыть.
- Где размещается ingredient ревью-дополнения TC (ownership лейна).
- MCP-пресет QA (requirement_tests узкий) — ADR рекомендует, но статус draft.
- Есть ли у QA write-back в TestOps для findings ревью — зависит от write-модели ADR + механизма Солонко.

## 5. Открытые вопросы (к lead-qa / lead-arch / пользователю)
1. **Кто и чем регистрирует TC аналитика в Allure TestOps?** (импорт из markdown? ручной ввод? другой инструмент?) — без этого шов §1 не сомкнут в коде.
2. Ревью-дополнение TC — отдельный QA-скилл или расширение `tc-review` / шаренный analysis-скилл? (ждёт ADR §3, но нужен наш вход lead-arch'у).
3. Согласовать write-канал: `iva-write-base` (QA, текущий) vs `iva-atlassian-write` (ADR) — один канал или разные?
4. Двойное покрытие (analyst `requirement_tests` vs QA `coverage-ledger`) — сводить или оставить как есть?
5. Worktree-дрейф копий analysis/core-base между dev и qa-kit — учесть при мердж-порядке (QA-ветка стоит на `feat/iva-write-base`, не в main).

## Источники (пути)
- QA: `~/tacticum/tacticum-dev-qa-kit/templates/iva-qa-autotest-base/ingredients/skills/write-autotest/{SKILL.md, references/input-sources.md, references/tc-review.md, references/csv-parsing.md, references/testops-api.md}`; `.../jira-issue-autotest/SKILL.md`; `iva-role-qa/manifest.yaml`.
- Аналитик: `~/tacticum/tacticum-dev/templates/iva-analysis-base/{manifest.yaml, ingredients/skills/tests-authoring/SKILL.md, ingredients/agents/tacticum-workflow.md}`; `iva-role-analyst/manifest.yaml`.
- Память: [[adr-draft-model-vzaimodeistviia-profilei-analitik-qa-razrabotchik-skouping-mcp-po-roliam]] · [[reshenie-test-keisy-pishut-analitiki-qa-dopolniaet-na-reviu-2026-07-23]] · [[qa-profile-model-opis-multi-stek-model-qa-leinov]] · [[napravlenie-profili-qa-profil-iva-role-qa-aqa-toolkit-iva]]