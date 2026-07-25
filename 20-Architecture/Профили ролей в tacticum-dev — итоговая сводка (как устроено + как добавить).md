---
title: Профили ролей в tacticum-dev — итоговая сводка (как устроено + как добавить)
type: reference
permalink: tacticum/20-architecture/profili-rolei-v-tacticum-dev-itogovaia-svodka-kak-ustroeno-kak-dobavit
status: current
tags:
- tacticum-dev
- profiles
- roles
- agentic-dev
- qa
- designer
- techwriter
- support
- reference
---

# Профили ролей в tacticum-dev — итоговая сводка (2026-07-20)

Разбор по задаче руководителя: понять систему профилей (аналитик/разработчик-go), чтобы добавить профили **техпис, техподдержка, QA, дизайнер (mockup)**. Репо `/Users/bubblemac/tacticum/tacticum-dev` 

## Модель
- Профиль = подкаталог **`templates/<profile-id>/`** = `manifest.yaml` (схема v2, ADR-0023) + тела в `ingredients/`. Отдельного реестра нет; при мерже в `main` CI сидит манифесты в Postgres-каталог (`seed_community.py`), раздаёт через `catalog-mcp`.
- **«Роль» = стэк-профиль**, объявляющий `depends_on: [<base>...]` (композиция, ADR-0056/0057; один уровень, override профиля побеждает базу). Отдельные «role-пресеты» ОТКЛОНЕНЫ — роли через композицию.
- **9 типов ингредиентов** (ADR-0020): instruction_pack, rule_set, agent_spec, skill_spec, mcp_server_spec, command_spec, hook_spec, permission_policy, repo_config.
- **База композиции:** `tacticum-dev-base` (stack-agnostic ядро dev-воркфлоу), `tacticum-ui-base` (UI/дизайн-кластер).

## Существующие профили (`templates/`)
- Аналитик: **`iva-system-analyst`** (as-is → ТЗ), `iva-fr-analyst` (сырьё → FR). Single-tier, без конвейера coder/tester, read-only по репо, артефакт — документ.
- Разработчик: **`iva-go-backend-brownfield`** (developer-go) + web/rn/kmp/ios/mail. 4 агента (tacticum-workflow-оркестратор + coder + tester + test-runner), 8 команд (`start-task`/`run-*`/`fix-bug`), ~15 скиллов, артефакты дизайн-фазы BRD/ADR/PIN/TESTS (+MOCKUPS в UI-стеках).

## analyst vs developer-go
analyst: stack `[]`, 1 read-only агент, 1 команда, артефакт-документ, без depends_on. developer-go: stack `[go]`+optional, 4 агента-конвейер, 8 команд, 15 скиллов, код+BRD/ADR/PIN/TESTS.

## Как добавить профиль (канон — skill `profile-authoring`)
Фазы: исследование целевого репо → `<profile-id>-task.md` в корне → **апрув owner (гейт)** → US в Taiga (project_id=12) → worktree → пара coder→tester (тест-инварианты: override ==объявленный, grep-гейт чужих стеков, e2e_install golden, quickstart) → PR (мержит owner) → сид на VPS + `provision_installation` + приёмка.
**Минимум файлов:** `templates/<id>/manifest.yaml` (+depends_on на базу, только дельта-ингредиенты) + `ingredients/` + README + CHANGELOG + `tests/verify-*.ps1` + строка в `.gitattributes`.

## Статус 4 новых ролей
- **QA** 🟢 проработан: план профиля **`iva-qa`** (скиллы `test-case-authoring`+`regression-checklist`, команда `/start-testing`) + QA-автоматизатор (переиспользует tester+test-runner, `/extend-coverage`). `process-analyst-dev-qa.md`, ADR-0042/0045.
- **Дизайнер (mockup)** 🟡 частично: дизайн-кластер `tacticum-ui-base` (`design-system-discovery`, `ui-mockup-match`+playwright). НО `ui-mockup-match` = СВЕРКА рантайм-UI с макетом, НЕ генерация. Отдельного скилла «сгенерировать mockup из ТЗ» и Figma-интеграции в репо НЕТ → greenfield-дельта над `tacticum-ui-base`. Продуктовая RBAC-роль `designer` — Phase 2 (ADR-0026/0028/0046, Taiga E27 #322), defer'нута.
- **Техподдержка** 🟠 только заявлена как будущая RBAC-роль «по паттерну designer» (ADR-0028). Косвенно — ops-скилл `user-incident-investigation`.
- **Техпис** 🔴 в tacticum-dev не задокументирован (0 упоминаний). Единственный след — helm-analyst позиционируется для «аналитиков/техписов ИВА» (внешний тулсет, не профиль).

## Рекомендация по раскладке
- **QA / техпис / техподдержка** ≈ класс аналитика (single-tier, документ-артефакт) → тонкая дельта по образцу `iva-system-analyst`. QA почти готов; техпис/поддержку проектировать с нуля.
- **Дизайнер** = дельта над `tacticum-ui-base` + новый скилл mockup-генерации + опц. Figma MCP. Ждём концепт от руководителя.

## Taiga-эпики по теме (LIVE, project_id=12 `tacticum-tacticum_dev`, `project.cifragen.ru`) — сверено 2026-07-20
Taiga MCP ожил (wiki-mcp всё ещё -32602). Релевантные эпики:
- **#682 E-FRQA** — конвейер аналитик→разработчик→QA (FR-вход, report-петля, QA в процессе): New, **6/7 US в работе** ← QA активно строится.
- **#646 E-COMP** — Profile composition (tacticum-dev-base + depends_on): New, **9/11 US в работе** ← модель композиции почти готова.
- **#554 Catalog, Ingredients & Profiles**: New, 2/30.
- **#540 Design (Token Layer + Studio)**: New, 2/16, назначен Дмитрий Лебедев ← продуктовый дизайн/токены, не конвейерный профиль.
- **#560 Quality & Testing (TCMS + Gates)**: New, 0/15.
- Прочее: #602 доп-MCP навигации в brownfield-профили (0/6) · #639 E43 эффективность агентов (1/3) · #557 Feature Lifecycle & Arch Skills (9/27).
- **Эпиков «техпис» и «техподдержка» НЕТ** — эти роли в план ещё не заведены.

## Связано
- [[explore-профили-ролей-tacticum-dev]] · [[session-state]]