---
title: map-existing-vs-gap-sc12
type: report
permalink: tacticum/00-board/map-existing-vs-gap-sc12
status: draft
role: explorer
for: lead-ds
repo: ~/tacticum/tacticum-dev (read-only)
tz: scratchpad/ds-scan/figma-ds-{process-tz,scenario-1-library,scenario-2-new-screen}.md
date: 2026-07-24
tags:
- figma-ds
- explore
- sc1
- sc2
- gap-map
- lead-ds
---

# Карта existing-vs-gap — Сц.1 (библиотека) и Сц.2 (экран по макету)

Вход для плана лида: строить ТОЛЬКО реальный gap, не дублировать задеплоенное DS-командой (#132/#134). Всё сверено по репо `~/tacticum/tacticum-dev` символьно/грепом, read-only.

## EXISTING — что уже есть в репо (базовый слой обоих сценариев)

**Словарь code-bindings (ядро Сц.1/Сц.2) — ЗАДЕПЛОЕН.**
- `design-systems/iva-web/tokens.json` → `$extensions."dev.tacticum.code-bindings"`: **49 компонентов, 32 со стабильным `figma_key`** (17 null = в другом файле мультифайловой ДС).
- Поля биндинга: `name, match[]` (алиасы для резолва), `figma_key, selector` (напр. `iva-chip`), `kind, source` (путь к .component.ts), `storybook, inputs{}`.
- `design-systems/iva-web/design-system.yaml`: `version: 0.3.0, status: published, platform: web, framework_hint: react`. CHANGELOG словаря: 0.2.0 добавил code-bindings, 0.3.0 залил 32 figma_key + алиасы (Radio↔Radiobutton, Tabs↔Tab Group, Select↔Dropdown) + `figma_file_key`.
- `usage`-заметка в словаре описывает нормализацию резолва (lowercase, strip spaces/dashes).

**Навыки в веб-пакетах:**
- `iva-web-brownfield/ingredients/skills/`: `design-system-discovery`, `design-token-usage`, `ui-mockup-match` (+ angular-build/module/run/ui-testing, ivcs-libs-contract, ngrx-signals-state, nx-workspace-discipline, tests-authoring, pin-*, adr/brd/pin-authoring, bug-fix).
- `iva-web-development-base/ingredients/skills/`: DS-навыков НЕТ (только ivcs-libs-contract + angular build/test/run + nx/openapi/surgical/webrtc). **DS-способность живёт только в brownfield.**
- `iva-role-web`: своих skills НЕТ (только `ingredients/repo-configs`). Композитный профиль: manifest наследует `tacticum-ui-base` (= «способность UI: DS-discovery, токены, mockup-match, playwright») + `iva-web-development-base`.

**MCP-доставка токенов/словаря — есть** (`tacticum-mcp`: `design_list_systems / design_get_tokens / design_resolve_token / design_get_theme_tokens`), премиум-гейт ADR-0028. Это опора Сц.2 шаг 3 (резолв по словарю с сервера).

**Гайды по вебу (docs/user_manuals):** есть только `iva-kmp-figma-mapping-quickstart.md` (шаги 0-4, резолв по словарю iva-mobile). **Веб-quickstart `iva-web-figma-mapping-quickstart.md` в этом репо ОТСУТСТВУЕТ** — хотя ТЗ помечает его ✅ задеплоен. Вывод: веб-правило пилота живёт в самом репо iva-one (блок в AGENTS.md/CLAUDE.md), НЕ как профильный скилл/гайд здесь. → coordination-факт (см. риски).

---

## Сценарий 1 — наполнение библиотеки компонентов

### Требует ТЗ
1. Дизайнер публикует мастер-компонент (варианты Property=Value, auto layout, описание, publish → стабильный key).
2. Конвейер строит gap-отчёт «есть в Figma — нет в коде».
3. Разработчик+агент реализуют компонент **по анатомии репо** (web: standalone/OnPush/signals, директива `button[ivaX]` vs элемент `iva-*`, `IvaControlBase`+CVA, слоты `ivaPrefix/ivaSuffix/ivaLabel`, стили через `main.ivaGetColor()/ivaFontPreset()` — ноль hex, все варианты, Storybook + `.mdx`, spec-тест).
4. Конвейер пересобирает словарь (авто) + CI-проверка (каждый компонент/пропс существует).
5. Приёмка: полнота вариантов, витрина↔Figma числами, ноль hex, тест.

### Уже существует
- Словарь (49 комп., поля selector/source/storybook/inputs/kind) — п.4 частично покрыт (структура есть; авто-пересборка+CI НЕТ).
- gap-отчёт первый построен вручную скриптом `scripts/figma_ds_review.py` (в figma-тулинге, не в шаблонах) — ~35 сетов.
- Nx-генератор `angular-library` + гайды авторинга `libs/design-system/src/lib/docs/guides/{development,storybook}` — в репо iva-one (не в шаблонах).

### РЕАЛЬНЫЙ GAP (Сц.1)
- **[G1] Навык авторинга-конвенций web-компонента отсутствует.** Ни один скилл в веб-пакетах не описывает «анатомию компонента» (директива vs элемент, IvaControlBase, слоты, токены-не-hex, Storybook+mdx). `design-system-discovery` — это design-фаза (перечислить токены/назвать существующий lib-компонент в PIN), НЕ авторинг. Нужен новый скилл (напр. `angular-ds-component-authoring`).
- **[G2] Словарь v1 — недостающие поля.** Есть `kind`, но нет `mdx_path/host/requires/slots/import/category` (ТЗ строка 129). Влияет и на Сц.2 (чтение доки перед сборкой). Наполнение — на стороне словаря/конвейера RE, не шаблонов.
- **[G3] Авто-пересборка словаря + CI-проверка соответствия** (аналог `figma connect publish --dry-run`) — план в RE-конвейере, в репо нет. Вне зоны веб-профиля.
- Регуляризация gap-отчёта — тоже RE-конвейер (не шаблон).

---

## Сценарий 2 — новый экран по макету Figma

### Требует ТЗ
1. Предпроверка макета (auto layout? инстансы распознаются? иначе СТОП → дизайнеру).
2. Чтение структуры фрейма (Figma MCP `get_metadata` → точечно, беречь квоту ~200/день).
3. Резолв: `get_code_connect_map` → иначе имя/ключ инстанса → словарь (`design_get_tokens` → code-bindings).
4. Компонент не в словаре → СТОП по элементу (→ Сц.1 или возврат), не выдумывать.
5. Сборка из готовых (web-композиция: `<button ivaButton>`, `iva-menu` через `[ivaMenuTriggerFor]`, поля в `iva-form-field`; читать `.mdx` перед использованием) — **ТЗ ссылается на «скилл component-usage»**.
6. Токены — только именованные, ноль hex/px.
7. Приёмка: скриншот vs макет ЧИСЛАМИ (усиленный `ui-mockup-match`: pixel-diff + ΔE + расхождения размеров, допуск по образцу uiMatch); ноль самодельной разметки.

### Уже существует
- Словарь + `match`-алиасы + MCP-доставка → резолв (п.3) технически возможен.
- `ui-mockup-match` скилл ЕСТЬ, НО: **HTML-макеты only**, playwright vs Nx dev-server, диф semantic-token+DOM, **явно "NEVER pixel SSIM / pixel-diff"** и «Figma URLs/PNG out of scope». То есть текущий скилл — противоположность требованию ТЗ (числовое pixel/ΔE vs Figma).
- `design-token-usage` — резолв токенов при реализации (п.6 покрыт).

### РЕАЛЬНЫЙ GAP (Сц.2)
- **[G4] Навык `component-usage`/`angular-ds-component-usage` ОТСУТСТВУЕТ** (искал по всем templates — нет нигде). ТЗ Сц.2 шаг 5 прямо на него ссылается. Это ядро сборки экрана из словаря: резолв инстанс→selector, правила композиции Angular (директива/элемент, form-field, menu-trigger), «читать .mdx перед use», «не найден → стоп». Главный gap Сц.2.
- **[G5] `ui-mockup-match` под Figma + числа.** Расширить (или новый режим): вход = Figma-скриншот (`get_screenshot`) + node, метрики pixel-diff + ΔE + size-deltas, допуск. Прямо конфликтует с текущей философией скилла («no pixel») → это осознанное расширение, не косметика. Скилл широко зеркалирован (см. риски) — трогать аккуратно.
- **[G6] Предпроверка макета** (auto layout / распознаваемость инстансов гейт) — в скиллах нет; ТЗ: «три строчки в правило репо при тиражировании». Может войти в G4.
- **[G7] `design-system-discovery` под ось-1 (выбор ДС по поверхности + framework).** Сейчас скилл — только токены (design_list_systems/get_tokens), НЕ резолвит figma/code-bindings/компоненты; `framework_hint: react` в yaml (веб реально Angular). ТЗ (ось1): правка скилла под platform/framework_hint + фикс `.tacticum/context.yaml`. Отдельный небольшой gap.
- **[G8] Веб figma-quickstart как артефакт профиля** — в репо нет (только kmp). Тиражирование пилота iva-one → профильный гайд/скилл + маркеры в AGENTS.md. Форма — по образцу `iva-kmp-figma-mapping-quickstart.md`.

---

## Укрупнение (что в ОДИН PR)

- **PR-A «web DS component skill»:** G1 (авторинг Сц.1) + G4 (usage/сборка Сц.2) + G6 (предпроверка макета) — одна анатомия web-компонента и правила композиции, один слой знаний. Можно один скилл с двумя режимами (author/use) или два скилла в одном PR. Приземление: `iva-web-brownfield` (где живут DS-навыки) — и решить, нужен ли owner в `iva-web-development-base`/`tacticum-ui-base` под композицию role-web (см. риски).
- **PR-B «ui-mockup-match → Figma числа»:** G5 (+частично G6, если гейт вешать на вход матчинга). Отдельный PR — скилл сильно зеркалирован по стекам, риск дрейфа.
- **PR-C «design-system-discovery ось-1 + context.yaml»:** G7 + G8 (правило/quickstart профиля). Мелкий, но трогает мультизеркалимый скилл.
- G2/G3 — вне веб-профиля (словарь + RE-конвейер), в PR лида по вебу НЕ входят; зафиксировать как зависимость/передать DS-команде.

## Пересечения-риски (координация через ГД)

- **DS-навыки мультизеркалены, копии РАЗЛИЧАЮТСЯ (не байт-в-байт):** `design-system-discovery` в 7 пакетах (tacticum-ui-base, iva-web-brownfield, iva-analysis-base, iva-fr-analyst, iva-kmp-brownfield, iva-rn-brownfield, iva-brownfield-mail), `ui-mockup-match` в 5 (ui-base, web/kmp/rn/mail-brownfield). Правка G5/G7 → риск дрейфа между стеками. Координация с lead-fr (firebird-web-brownfield — свой стек) и lead-modes.
- **_mirrors.yaml (`templates/_mirrors.yaml`, ADR-0059/US#714):** пара `iva-web-development-base(owner)→iva-web-brownfield(mirror)` НЕ содержит design-system-discovery/design-token-usage/ui-mockup-match (они brownfield-only, не зеркалятся в base). НО `design-system-discovery` отдельно в паре `iva-analysis-base(owner)→iva-fr-analyst`. → Новый web-DS-usage скилл: решить owner. Если добавлять в _mirrors — новая запись; иначе brownfield-only (как текущие DS-навыки). Правка зеркалимого файла требует байт-парити (CI `check_mirror_sync.py`).
- **Композиция role-web:** `iva-role-web/manifest.yaml` наследует `tacticum-ui-base` (UI-способность) + `iva-web-development-base`. Новый скилл должен быть подключён в цепочку композиции (base или ui-base), иначе не попадёт в role-web. tacticum-ui-base = кандидат-канон для UI-generic частей; Angular-специфику логичнее в web-development-base/brownfield. Решение о размещении — за ответственной ролью.
- **iva-web-brownfield трогается напрямую** (G1/G4/G6 приземляются туда) — пересечение с lead-fr/lead-modes по этому пакету; согласовать через ГД.
- **ROLE_LANES:** термин в репо `tacticum-dev` НЕ найден (grep по CLAUDE.md/*.md/*.yaml пусто). Вероятно понятие внешней доски/оргслоя, не артефакт репо. Уточнить у ГД, что именно проверять.