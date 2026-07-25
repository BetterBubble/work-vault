---
title: 'План: ТЗ#2 режимы workflow + двухслойный гейт (lead-modes)'
type: note
permalink: tacticum/00-board/plan-modes
created: 2026-07-24
updated: 2026-07-24
status: draft
tags:
- plan
- modes
- lead-modes
- tokens
- lanes
- gate
- draft
---

# План: ТЗ#2 — режимы workflow + двухслойный гейт

**Направление:** [[napravlenie-optimizatsiia-tacticum-pod-tokeny-liogkie-rezhimy-gpt-5.6]] (lead-modes).
**Источник истины:** пакет ТЗ#2 Солонко `workflow-modes-proposal` (owner-согласован по таксономии/двухслойной схеме/сценарию research/посадке на лейны). Дизайн НЕ переоткрываем.
**Репо:** `TacticumApps/tacticum-dev` → `templates/` · **github/main = истина** (GitLab — зеркало) · **autonomy off** (PR/мерж — президент).
**Статус разведки:** карта лейнов — explorer (`explore-modes-lanes-map`, в работе); детали посадки дошлифуются его результатом.

## 1. Проблема (зачем)
`/start-task` навязывает полный конвейер (BRD→MOCKUPS→ADR→PIN→код→тесты) любой задаче. По разметке 31 реального кейса (web/kmp/iOS) ~40% недельного потока — мелкие правки, которым конвейер вреден; ещё доля — рисёрчи/рефакторинги/инфра, под которые он не спроектирован. Две жалобы-якоря: **IVAONE-12770** (рисёрч в конвейере → нерабочий код, агент сдался), **IVAONE-9806** (тривиальный зум-модификатор, 5ф/+154, раздут полным циклом). Побочно: `tacticum-mcp` ~5× токенов → команды обходят инструмент (KMP завела локальный `lite-task-workflow`).

## 2. Скоуп
**Делаем:** маршрутизацию на входе (1-й гейт), недостающие режимы-лейны, протокол смены режима по ходу (2-й слой + handoff), eval, замер токенов под GPT-5.6.
**НЕ делаем:** составы конвейеров внутри режимов (полный цикл BRD→PIN, процедуры coder/tester) не меняем. Не трогаем чужой скоуп (QA-лейны — lead-qa; ADR-модель профилей — lead-arch).

## 3. Целевая таксономия: режим = процесс-лейн (ADR-0056/57/59)
| Режим | Лейн / команда | Статус | Артефакты |
|---|---|---|---|
| Багфикс (лайт) | `tacticum-bugfix-base` / `/fix-bug` | ✅ есть (+ scope-tripwire эскалация = половина 2-го слоя) | diagnose→repro→fix→verify |
| Лайт-доработка | расширить bugfix-лейн типами refactoring-S / feature-S (из kmp-прототипа) ⚠️ развилка §6.A | новое | рабочий ордер + ок-гейт → код → verify |
| Полный | `iva-analysis-base` (`/start-task` Ф1–2) + `tacticum-development-core` (`/run-implementation` Ф3–5) | ✅ есть | BRD→MOCKUPS→ADR→PIN→TESTS |
| Рисёрч | новый `tacticum-research-base` / `/start-research` | новое (в репо НЕТ — подтверждено) | research-report + черновик ADR, без кода |
| Рефакторинг | refactoring-S покрывает лайт; кампании — отдельный лейн по спросу | отложено | план миграции → механич. правки → verify |

Первый гейт живёт в **двух местах**: `/start-task` (маршрутизация постановки) и триаж `/fix-bug` (вход dev-роли). Старые brownfield-профили — через зеркала (`_mirrors.yaml`).

## 4. Этапы (по §6 ТЗ)
- **Э0 — разведка + план (✅ done 24.07):** карта лейнов [[explore-modes-lanes-map]]; план собран. Worktree `feat/workflow-modes` поднят.
- **Э1a — скаффолд lite-ингредиента (✅ done 24.07, вариант «в» ГД):** драфт `docs/proposals/workflow-modes/`, коммит `f2f9ce7`. Отчёт [[impl-modes-lite-scaffold]].
- **Э1b — сверка с оригиналом r.yarullin (✅ done 24.07):** оригинал `lite-task-workflow` достал read-only с adp_emb (210 стр. + шаблон, `scratchpad/kmp-original/`). Драфт приведён в соответствие, коммит `86bb469`: 3 фикса (снят AFK-обход ок-гейта; диагностика на все 3 типа; нет файлов-артефактов, отчёт в чат) + восстановлены потерянные концепции (test-first, FROZEN-снапшот, гейт по тулсету, эскалация посреди работы). 9 предположений: CONFIRMED 1/4/5/6/9, CORRECTED 2/3/7/8. Генерализация — kmp-специфика в `[REPO-СЛОЙ]`. Отчёт [[impl-modes-lite-reconciled]].
  - **A/B ЗАФИКСИРОВАН → Б** (отдельный лейн `tacticum-lite-base` + отдельный скилл/команда, НЕ расширение `/fix-bug`). Доказательство: оригинал — самостоятельный скилл-«альтернатива полному циклу», сам эскалирует в `/start-task`, содержит feature-S (=change) внутри → это не часть fix-bug (инвариант restore→fix-bug).
  - ⚠️ ОТКРЫТО для живой посадки: маршрутизация входа `/fix-bug` (restore-only) vs `/lite-task` (bugfix — один из 3 типов) в 1-м гейте роли — правило написать при посадке.
- **Э1 — валидация эталона:** owner (Солонко) валидирует колонку «Эталон» в `gate-calibration-cases.md` (31 кейс). ⚠️ внешняя зависимость.
- **Э2 — research-лейн (✅ файлы готовы 24.07):** новый `tacticum-research-base` создан (`/start-research`: отчёт + ADR-черновик, без кода/клонирования; KB cross-repo; факт-инструменты опц.). Коммит `dbe976f`, 5 файлов, тесты схемы+пресетов 129/0, orphan-лейн тесты не ломает. Отчёт [[impl-modes-research-base]]. ✅ ВРЕЗКА done (коммит `65d4984`): tacticum-research-base в depends_on 7 ролей (6 dev + аналитик), ROLE_LANES обновлён, тесты 211/0. Отчёт [[impl-modes-research-wiring]]. ⏳ дифф ROLE_LANES отправлен ГД на разводку с lead-fr US#3 (shared `test_iva_role_presets.py`) — **жду GO на пуш**.
- **Э3 — ADR-first вход `/start-task`:** параметр (не режим) — конвейер от поданного ADR, сверка ADR с кодом по KB, ничего не досочиняет.
- **Э3a — живая сборка `tacticum-lite-base` (✅ done 24.07):** новый лейн (manifest + `/lite-task` + скилл `lite-task-workflow` + шаблон ордера), коммит `2ab690f`, 6 файлов, тесты 211/0. Правило маршрутизации в скилле: `/fix-bug`=узкий restore-only; `/lite-task`=широкий (bugfix/refactoring-S/feature-S); при неоднозначности гейт спрашивает; эскалация в `/start-task` по списку. Отчёт [[impl-modes-lite-base-lane]]. ✅ COMPANION + ВРЕЗКА done (GO ГД, bugfix-base = single-owner, не mirror): (1) `1102198` — bugfix-base routing «change» разведён на мелкое→/lite-task vs крупное→/start-task (restore→/fix-bug цел); (2) `3f6aa5a` — lite-base в 6 dev-ролей + ROLE_LANES (не аналитик). Тесты 211/0. Отчёт [[impl-modes-lite-wiring-companion]]. 🔧 полиш bugfix-base SKILL.md 26-31 (внутрифайловая консистентность routing) — добивается. Дифф ROLE_LANES (research+lite) отдан ГД для разводки с lead-fr US#3, пуш — бандлом через ГД.
- **Э3c — фикс по critic (🔄 24.07, GO ГД):** B1 (lite=refactoring-S/feature-S только, bugfix вон — align proposal §1.2), B2 (убрать багфикс-триггеры), M2 (локация research-артефактов no-clone), companion bugfix-base подправить. → [[impl-modes-b1b2m2-fix]].
- **Э4 — 1-й гейт (🔄 24.07, заморозка iva-analysis-base СНЯТА реш. президента, вариант «а»):** классификация в `/start-task` (аддитивно, прод-бережно) по §2/§4: достаточность→классификация→ПОДТВЕРЖДЕНИЕ, «ТЗ недостаточно»→3 вопроса; маршрут align §1.2 (bugfix→/fix-bug, refactoring-S/feature-S→/lite-task, research→/start-research). → [[impl-modes-gate1-starttask]].
- **Э5 — 2-й слой (🔄 24.07):** гейт пересмотра + handoff в `run-implementation` (§3), «агент сдался»→предложение смены режима; handoff.md разрешён как исключение к lite no-files. → [[impl-modes-gate2-runimpl]].
- **Э3c/Э4/Э5 ✅ реализованы (24.07):** фикс B1/B2/M2 (`f7c3b72`,`7ece71f`), 1-й гейт start-task (`a2ec93d`, +50/−0 аддитивно), 2-й слой run-implementation (`a11c68e`, +36/−0). Ветка = 12 коммитов, дерево чисто, объединённые тесты 211/0. Отчёты: [[impl-modes-b1b2m2-fix]] · [[impl-modes-gate1-starttask]] · [[impl-modes-gate2-runimpl]].
  - ⚠️ owner-ревью-точка (флаг ГД): 1-й гейт по букве ТЗ требует подтверждения режима ДАЖЕ для полного конвейера → добавлен шаг подтверждения к текущему авто-старту аналитика (пайплайн не деградирован).
- **Приёмка v2 (24.07):** controller [[gate-modes-v2]] = **GO** (211/0, аддитивно, 0 секретов/подписей). critic-verify [[critic-verify-modes-v2]] = B1/B2/M2 **закрыты**, 1-й гейт верен; во 2-м слое 1 MAJOR + 2 MINOR → ✅ исправлено (`7f0451e`, тимлид сверил дифф с предписаниями + тесты 211/0) [[impl-modes-critic-verify-fix]]. **Приёмка v2 ЗАКРЫТА — готово к бандлу.** Ветка = 13 коммитов, дерево чисто. Отчёт ГД отдан → бандл на push через ГД+президента.

## 4d. Очередь: ось-2 мульти-репо (cross-request lead-ds, ПОСЛЕ bundle-фикса)
Спека [[spec-axis2-workflow-for-lead-modes]] — пакет `brownfield-task-workflow` (мой single-owner; НЕ путать с `/start-task` iva-analysis-base). Требования по ТЗ Сц.4: **A** — опц. аргумент source/reference-репо в `start task`; **B** — послабление «одно дерево»→read-only source + write target (ДС письма=целевой репо); **C** — гейт Phase 3.0 реальной проверки двух деревьев. Стык с reference-скиллом lead-ds `web-to-kmp-source-reference` (в iva-kmp-development-base) — не пересекается (контракт vs механика). Не-цели: сервер не трогать, ось-1 не делать, без сверх-ТЗ ограничений. Гардрейлы: аддитивно, прод не ломать, тесты; push/мерж через ГД+президент. Статус: **СТАРТ 24.07** — bundle ТЗ#2 эскалирован президенту (вне моих рук), ГД дал «дальше ось-2». `brownfield-task-workflow` = single-owner (не mirror), СВОИ start-task.md/run-implementation.md + агент `tacticum-workflow` в 3 CLI-вариантах (claude/codex/copilot — B/C синхронно во все три). Отдельная ветка от main. Разведка [[explore-axis2-brownfield]] — ⛔ **БЛОКЕР ЦЕЛИ:** пакет `brownfield-task-workflow` **DEPRECATED** (frozen, superseded by iva-brownfield-mail, без зависимостей, §3.0-гейта в нём НЕТ). Реальный Сц.4 Angular→KMP ложится на mirror-пару `iva-web-brownfield`(source)+`iva-kmp-brownfield`(target), где §3.0 УЖЕ есть (iva-web-brownfield tacticum-workflow.md:231-233) — это shared/mirror-лейны + территория lead-ds (kmp). → ГД сменил цель на ЖИВУЮ пару iva-web-brownfield(source)+iva-kmp-brownfield(target). Gap-карта [[explore-axis2-gapmap]]: freeze OK (не frozen); **место правок НЕ зеркалируется** (start-task+tacticum-workflow вне _mirrors → правим прямо в brownfield, mirror-риск ноль); A/B=чистый GAP, C=web PARTIAL/kmp GAP. Отправлен ГД пакет + **R1 scope** (рек. **kmp-only** — минимум-по-ТЗ, не лезем в горячий web). **R1=kmp-only. ✅ РЕАЛИЗОВАНО + ПРИЁМКА (24.07).** Ветка `feat/kmp-multirepo-axis2`, коммит `52d47e7` + merge origin/main (0 конфликтов, 0 behind). A/B/C только в `iva-kmp-brownfield` (6 файлов, аддитивно/условно «при source», 3 CLI-тела; §4a bump 0.4.5→**0.5.0**). Web/зеркала/сервер не тронуты. Приёмка: controller GO [[gate-axis2-kmp]] + critic/fidelity [[critic-fidelity-axis2]] (A/B/C верны, BLOCKER/MAJOR нет; copilot-MINOR + minor-bump внесены). Тесты 211/0, дисциплина 0 violations. Президент апрувнул push → свежий origin/main (#143+#1) сведён (0 конфликтов), валидаторы 211/0 + дисциплина 0 violations → **push выполнен** (ветка feat/kmp-multirepo-axis2). **✅ СМЕРЖЕНО в main (PR #145).** iva-kmp-brownfield 0.5.0 в main. Стык lead-ds: скилл `web-to-kmp-source-reference` в main (#1/#144) → ссылка валидна. Ось-2 закрыта по коду.

## Статус направления (24.07, конец дня)
- ✅ ТЗ#2 срез #1 (режимы: lite/research лейны + 2 гейта + companion + врезки) — в main (#142).
- ✅ Ось-2 (мульти-репо A/B/C в iva-kmp-brownfield) — в main (#145).
- ⏸ eval на 31 кейсе — блокирован внешне (валидация датасета у Солонко, ГД флагнул президенту).
- ⏸ GPT-5.6 подтрек — паркнут.
- eval — окончательно **без формального** (готового датасета/прогона нет; невалидированный эталон = недостоверно).
- **Проверка полноты** [[fidelity-tz2-completeness]]: ядро реализовано; **3 реальных пробела** на добор-PR — (1) ADR-first вход как параметр `/start-task` (приёмника нет — разрыв research→build), (2) 2-й слой не покрывает Фазу-1/design в tacticum-workflow, (3) lite→research(мини)+внутрипроц. триггеры лайта. + мелочь низкоприоритетная. GO ГД. Добор реализован+приёмка: ветка `feat/workflow-modes-addon`, коммит `7674d53` (amend), 3 пробела закрыты + 2 мелочи, аддитивно, §4a bumps, тесты 211/0, дисциплина 0 violations. Приёмка: controller GO [[gate-tz2-addon]] + critic/fidelity [[critic-fidelity-tz2-addon]] (Пробел 3 был частичен → MAJOR+2 MINOR исправлены). Держу локально. ⚠️ ОЧЕРЕДЬ МЕРЖА (ГД): lead-fr US#3+US#5 первым → мой добор вторым → ребейз на новый main: iva-analysis-base 0.1.6(fr)→**0.1.7**(я), CHANGELOG ниже fr, ingredients после fr (22). ✅ Ребейз выполнен (merge origin/main после fr #3, коммит fc9f043, 0 behind): конфликт CHANGELOG разрешён — iva-analysis-base 0.1.7, ingredients 22(fr), моя [0.1.7] над fr [0.1.6]. Валидаторы: дисциплина 0 violations + mirror-sync 64/6 + тесты 211/0. PR-diff 12 файлов в 3 пакетах. Дифф апрувнут ГД, отмашка президента → **push выполнен** (feat/workflow-modes-addon, без force, 0 секретов/подписей/мусора). PR-ссылка отдана ГД для сборки → президент на мерж. Дальше паттерн: по-ТЗ✓ → живой пилот тест-стенд → память → прод-сид(gated).

## Доставка ТЗ#2 (24.07) — ✅ СМЕРЖЕНО В MAIN
PR **#142** смержен президентом (origin/main top = fb87459). В main: research-base + lite-base лейны, 2 гейта (/start-task классификация + run-implementation пересмотр+handoff), companion bugfix-base, врезки в 6 dev + analyst роли, §4a version-discipline чисто. Приёмка: controller GO + critic-verify (B1/B2/M2 закрыты). **Направление ТЗ#2 закрыто по коду.** Открыто (опц., отд. шаг): сид в прод-каталог / провижн ролей.
- **Э5 — 2-й слой + handoff:** промпт пересмотра во все режимы (контрольные точки: план готов / фаза / блокер / повторный провал verify); `Tasks/<N>/handoff.md`; фиксация смены в `report.md`. Правило «агент сдался» → предложение сменить режим.
- **Э6 — eval (гейт):** см. §5. Метрика — совпадение класса с эталоном.
- **Э7 — замер токенов (thread B):** см. §5.
- **Э8 — зеркала:** прокинуть гейты на старые brownfield-профили через `_mirrors.yaml`.
- **Э9 — US в Taiga:** эпик E-LANES #705 (или отдельный — решение owner): research-лейн · ADR-first · 1-й гейт+bugfix-расширение · 2-й слой+handoff.

## 4b. Посадка на лейны — по разведке [[explore-modes-lanes-map]]
- **1-й гейт** → `templates/iva-analysis-base/ingredients/commands/start-task.md` (тонкая, 22 стр., безусловный хэндофф) — вставка блока классификации перед хэндоффом; и триаж в `templates/tacticum-bugfix-base/ingredients/commands/fix-bug.md`. Классификация по тексту ТЗ (совпадает с калибровкой «гейт видит только формулировку»). ⚠️ `iva-analysis-base` — guardrail Diaret → через эскалацию ГД.
- **2-й слой** → уже наполовину есть: scope-tripwire `fix-bug.md:51-52` + `bug-fix/SKILL.md:142-152` (bugfix→start-task). Добавить: двусторонность + промпт пересмотра в `templates/tacticum-development-core/ingredients/commands/run-implementation.md` (раздел «Failure handling», после ack-маркеров). Межкомандный вызов — текстовая инструкция оркестратору (автоматики нет).
- **Лайт** → правка `command_spec`/`skill_spec` внутри `tacticum-bugfix-base` (base, без depends_on, зеркал НЕТ — правка свободна). ⚠️ feature-S = change behaviour → ломает текущий инвариант скилла «restore→fix-bug / change→start-task» (развилка A/B).
- **research-base** → новый base-манифест (без depends_on), структурный аналог `tacticum-bugfix-base` (skill+command, single-tier, stack-agnostic); добавить в `depends_on` dev- и analyst-ролей. Образца-предшественника нет.
- **GPT-5.6** → модель в `agents-codex/*.toml` (`model="gpt-5.5"` сейчас) + `metadata.model` (claude=`opus`). Апгрейд 5.5→5.6 — сквозной по ВСЕМ лейнам, шире modes → отдельная оценка объёма/скоупа.
- **Зеркала**: bugfix — нет; start-task не в зеркалируемом наборе iva-fr-analyst (свериться CI `check_mirror_sync.py` / `test_role_replacement_parity.py` при любом сдвиге shared-ингредиента).

## 🔬 Живой пилот ТЗ#2 на teststand (GO президента 24.07, приоритет над §1.2-добором)
Границы: только teststand (не прод/adp), контент из main #147, честно докладывать не-прогоны, не деплой.
**Feasibility-оценка (сделана):** codex-cli 0.142.3 + codex exec есть (юзер tacticum); сеть до ИВА рабочая БЕЗ VPN (helm 200, catalog-VPS:443, tacticum-mcp/serena/codegraph настроены). НО modes-профиль на стенде НЕ установлен (skills пусто; catalog=прод=старый). → ставить из main-файлов. ГД: подход = лёгкий честный смоук. ✅ **ПИЛОТ ЗЕЛЁНЫЙ** [[pilot-tz2-teststand-results]]: 5 смоуков на реальном codex-cli 0.142.3 (контент из main #147, teststand-scratch) — V1 классиф.→research, V-ADR (Пробел1) ADR-first исключает research+работа ОТ ADR, V2 (Пробел2) mode-review Фаза-1 полный→lite+handoff, V3 (Пробел3) lite→research+триггеры+handoff, V4 роутинг bugfix→/fix-bug (фикс B1). Все дошли до решения, propose-не-молча, handoff, стоп до подтверждения. Честно вне скоупа: полный downstream (ТЗ#2 не менял), реальный ADR-файл (заглушка). **ТЗ#2 подтверждён рантаймом.**

## ✅ ФИНАЛ ТЗ#2 (24.07) — ЯДРО+ДОБОР В MAIN
**Реализовано по коду (в main):** режимы #1 (#142: lite/research лейны + 2 гейта + companion + врезки) · добор полноты (#147: ADR-first вход + 2-й слой Фаза-1 + lite→research). Ось-2 мульти-репо (#145, отдельная фича Сц.4). Все прошли controller+critic/fidelity, тесты 211/0, дисциплина 0 violations.
**Решения президента:** 2.1=b (без формального eval сейчас — нет валидир. датасета) · 2.2 GPT-5.6 → паркнут в тех-долг отдельного направления.
**Активной код-работы НЕТ.** Остаток (gated/внешнее, не инициирую сам):
- прод-сид → чеклист [[prep-tz2-prod-seed-checklist]] (gated президент)
- eval → ask Солонко [[prep-tz2-eval-readiness]] (координирует ГД/президент)
- живой пилот → сценарии [[prep-tz2-acceptance-package]] (при интеграции/тест-стенд)
- self-audit полноты main → [[self-audit-tz2-final]] ✅ **ПОЛНОТА ПОЛНАЯ** (3 пробела + 2 мелочи в main, не откачены). ⚠️ 1 остаточный minor (property): §1.2 ИНФРА-свойство (предупреждение о радиусе + снятие стоп-правил на toolchain, не эскалация) не реализовано; lite эскалирует на инфру. Решение ГД = B (микро-добор). ✅ РЕАЛИЗОВАН: ветка `feat/workflow-modes-infra`, коммит `1e66102`, только tacticum-lite-base (инфра из эскалации убрана + свойство: blast-radius warn + релакс стоп-правил toolchain; §4a 0.1.2→0.1.3, дисциплина 0). Отчёт [[impl-tz2-infra-property]]. ⚠️ ФЛАГ: предсущ. красный тест в main @ #149 (iva-role-web не покрывает iva-web-brownfield — потеряны angular-ds навыки lead-ds; не мой, территория lead-ds) — отдан ГД на роутинг. Жду решение ГД: батарея+push сейчас (мой скоуп зелёный) vs ждать фикс #149.

## 4c. Приёмка среза (US#0-батарея, 24.07)
- **controller** [[gate-modes-research-lite]] — **GO**: git-чистота, scope (iva-analysis-base/_mirrors не тронуты), тесты 211/0, 0 секретов, 0 AI-подписей, манифесты целостны.
- **fidelity** [[fidelity-modes-research-lite]] — соответствие ТЗ по срезу OK; отклонения D1 (lite=Б) и D2 (companion bugfix) санкционированы ГД; D3 (безфайловый lite vs handoff §3) — латентно, на этап 2-го слоя.
- **critic** [[critic-modes-lanes]] — ⛔ **НЕ готов к пушу**: **B1** двойной bugfix (/fix-bug ↔ /lite-task, багфикс 2-9 файлов в оба, тай-брейка нет; конфликт с proposal §1/§1.2 где lite=refactoring-S/feature-S, bugfix в bugfix-лейне) + **B2** коллизия триггеров. Рек. critic (=proposal §1.2): lite = refactoring+feature-S ТОЛЬКО, bugfix целиком в /fix-bug. + **M2** локация research-артефактов при no-clone. → правки до бандла, решение по B1 за ГД.

## 5. Eval и замер (доказательство на реальных данных)
**Eval гейта:** датасет — 31 кейс, **вход = реальные тексты задач из Jira** (`gate-calibration-inputs.md`), не commit-заголовки. Прогон промпта на моделях кохорт (gpt-5.5/5.6 Codex, sonnet Claude Code) на **E2E-стенде ADR-0036 (real Codex CLI)**. Метрика — точное совпадение класса с эталоном; **дорогие ошибки: эпик→лайт и рисёрч→полный** — считать отдельно. Для 2-го слоя — серия «неверный стартовый режим» (12470 как lite, 12150 как полный, 12114 как lite); эталон — факт эскалации/понижения в нужной точке + корректный handoff.
**Замер токенов (thread B):** N реальных задач × {с tacticum-mcp / без} под GPT-5.6 → таблица расход/результат. Цель — «понятный путь эффективности» для команды-потребителя (числа, не эффектность). Может идти независимо/раньше гейтов.

## 6. Развилки — статус решений ГД (24.07)
- **A/B. Посадка лайта** → ГД **провизорно Б** (новый `tacticum-lite-base`, отдельный скилл `lite-task-workflow`, НЕ расширение `/fix-bug`): чище + актуальная реальность kmp = отдельный скилл; вариант A ломает инвариант restore→fix-bug. ⚠️ **Живую посадку НЕ финализировать** до сверки со СКИЛЛОМ r.yarullin (ГД проверяет доступ к их GitLab). Как достанем → сверка 9 предположений + структуры → фиксируем A/B.
- **research-base** → ГД **GO сейчас** (независим от A/B) — строим с нуля по образцу `tacticum-bugfix-base`.
- **GPT-5.6 / замер токенов** → ГД: **отдельный подтрек** (шире режимов), в текущий срез НЕ тащим, вернёмся.
- Таксономия лайта (один `/lite-task` с подтипами vs fast-path+лёгкий) — свести до eval (при финализации лайта).

## 7. Внешние зависимости / риски
- Валидация эталона — **owner Солонко** (Э1 гейтит осмысленный eval).
- Генерализация kmp `lite-task-workflow` — согласовать с **r.yarullin** (автор скилла; repo-специфику kmp в лейн не тащим). Скилла в нашем репо НЕТ — только у потребителя.
- Встреча с командой-потребителем (их «light flow», кастомизация) — вход в дизайн лайта.
- Jira-дисциплина плохая (`jira-discipline-report.md`) → гейт не должен доверять типу тикета/заголовку/размеру текста; обязателен выход «ТЗ недостаточно» (до 3 вопросов).
- Пересечение с ADR-0060 (модель профилей, target-CLI) и с QA-направлением (носитель Codex/GPT-5.6) — кросс-скоуп через ГД, не молча.

## 8. Статус и на сверку с ГД
> Канон плана — карточка ГД [[План ТЗ-2 Режимы работы разработчика (лёгкие флоу + гейты) — lead-modes]]. Эта заметка — рабочая проработка деталей (eval, посадка на лейны, развилки). Режим: решения по плану — через **ГД**; президенту — только настоящие развилки. autonomy off.

**Первый шаг (решение ГД):** генерализовать kmp-скилл `lite-task-workflow` (r.yarullin) в профильный ингредиент.

**Развилки на сверку с ГД (не блокеры старта):**
- A. Лайт: расширение `tacticum-bugfix-base` vs отдельный `tacticum-lite-base` — рекомендую расширение bugfix (по ТЗ).
- B. Таксономия лайта: один `/lite-task` с подтипами vs два тира (fast-path + лёгкий цикл) — рекомендую один `/lite-task`, свести к главному документу ТЗ; зафиксировать **до eval** (иначе метрика «совпадение класса» неоднозначна).
</content>
</invoke>
