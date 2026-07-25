---
title: fidelity-modes-research-lite
type: note
permalink: tacticum/00-board/fidelity-modes-research-lite
tags:
- draft
- explore
- lead-modes
- fidelity
- tz2
---

# FIDELITY-сверка: реализация lead-modes (ТЗ#2 Солонко) vs proposal

status: draft · роль: explorer (read-only) · ветка `feat/workflow-modes` · worktree `/Users/bubblemac/tacticum-worktrees/modes-workflow`

Эталон: `scratchpad/modes-pkg/workflow-modes-proposal.md` + kmp-оригинал `scratchpad/kmp-original/lite-task-workflow.SKILL.md` / `work-order-template.md`.

## Что в срезе (файлы реализации)
- `templates/tacticum-research-base/` — manifest, `/start-research`, skill `research`
- `templates/tacticum-lite-base/` — manifest, `/lite-task`, skill `lite-task-workflow` (+ ref `work-order-template.md`)
- `templates/tacticum-bugfix-base/ingredients/skills/bug-fix/SKILL.md` — companion-правка routing
- `apps/backend/tests/catalog/test_iva_role_presets.py` — ROLE_LANES + врезки в 7 role-manifests (`iva-role-{go,kmp,web,mail,ios,java}` + `iva-role-analyst`)

Факт границ: `git diff 20412ff..HEAD` НЕ трогает `templates/iva-analysis-base/` и `templates/tacticum-development-core/` — то есть `/start-task` и `/run-implementation` в этом срезе не менялись.

## Таблица поэлементной сверки

| # | Элемент ТЗ | Статус | Доказательство (путь / цитата) | Причина |
|---|---|---|---|---|
| 1 | §1 Таксономия — багфикс-режим | СООТВЕТСТВУЕТ | `tacticum-bugfix-base` существовал ранее; на ветке уточнён routing | — |
| 1 | §1 — лайт-доработка (refactoring-S/feature-S) | СООТВЕТСТВУЕТ (по варианту Б) | реализовано **отдельным** лейном `tacticum-lite-base`, а не расширением bugfix | см. отклонение D1 |
| 1 | §1 — полный конвейер | СООТВЕТСТВУЕТ (не трогали) | `iva-analysis-base`+`tacticum-development-core` не менялись | вне среза |
| 1 | §1 — рисерч | СООТВЕТСТВУЕТ | новый лейн `tacticum-research-base` / `/start-research`, артефакты research-report + adr-draft, без кода | — |
| 1 | §1 — рефакторинг как отдельный режим | СООТВЕТСТВУЕТ (сознательно отложено) | нет `/start-refactor`; refactoring-S — тип внутри lite. ТЗ: «отложено: refactoring-S покрывает лайт; кампании — отдельный лейн по спросу» | по плану ТЗ |
| 2 | §1.1 три типа bugfix/refactoring/feature-S | СООТВЕТСТВУЕТ | lite `SKILL.md` шаг 0, таблица типов (стр. 83-87) — 1:1 с kmp-оригиналом | — |
| 2 | §1.1 диагностика-до-правок (для всех 3 типов) | СООТВЕТСТВУЕТ, усилено | «Step 1 — Diagnose (read-only) — for **all three types**… mandatory for each»; в kmp диагностика тоже на 3 типа | генерализация точна |
| 2 | §1.1 рабочий ордер + ок-гейт (класс-я первой строкой, оспорима, правки только после «ок») | СООТВЕТСТВУЕТ | lite `SKILL.md` шаг 2, гейт-слова, «Generated the order and immediately started is the forbidden pattern»; `work-order-template.md` шаблон+пример | 1:1 |
| 2 | §1.1 список принудительной эскалации | СООТВЕТСТВУЕТ (обобщён) | kmp «новый Gradle-модуль / вне libs.versions.toml» → generic «new module / dependency outside the version catalog»; остальное (экран, сценарий, серверный контракт, >10 файлов/>3 модулей) дословно | стек-специфика вынесена в [REPO-СЛОЙ] |
| 2 | §1.1 отказ от KB внутри лайта | СООТВЕТСТВУЕТ | Preconditions: «NOT required: an external KB… do not call them… Needing KB depth is a sign of escalation» | — |
| 2 | §1.1 хрупкие зоны = обязательный smoke (не эскалация) | СООТВЕТСТВУЕТ | шаг 0 «fragile zone… NOT an escalation but a mandatory triage»; шаг 4 «device/manual smoke mandatory always» | граница зоны вынесена в [REPO-СЛОЙ] |
| 2 | §1.2/§4 «режь артефакты, не проверку» | СООТВЕТСТВУЕТ | test-first + FROZEN-тесты + платформенное само-ревью сохранены; выкинут только процесс (5 арт./суб-агенты/KB) | — |
| 2 | точность vs kmp-оригинал | СООТВЕТСТВУЕТ | README lite: «Механика цикла перенесена ТОЧНО… сверка коммит 86bb469». Проверил лично: ок-гейт, FROZEN-снапшот, гейт по тулсету (plan mode/Codex), эскалация посреди работы, отчёт в чат без файлов — все на месте | — |
| 3 | §1.2 режим = процесс-лейн | СООТВЕТСТВУЕТ | manifest lite/research: «процесс-лейн (ADR-0057)», `depends_on` отсутствует (base-лейн) | — |
| 3 | §1.2 research — отдельный лейн, стек-агностик, нужен dev-ролям | СООТВЕТСТВУЕТ | research `SKILL.md` «Stack-agnostic… reused by every role, including dev-roles that have no analysis lane»; факт-тулы helm-analyst/iva-read опциональны | — |
| 3 | §1.2 lite как ОТДЕЛЬНЫЙ лейн (развилка A/B) | СООТВЕТСТВУЕТ решению **Б** | README lite: «Развилка A/B — финализирована в пользу Б (отдельный лейн, НЕ расширение fix-bug)» | см. D1 |
| 3 | врезки в роли (depends_on) + ROLE_LANES | СООТВЕТСТВУЕТ | `test_iva_role_presets.py` ROLE_LANES: lite+research в 6 dev-ролях, research в analyst; manifests обновлены (коммиты 3f6aa5a, 65d4984) | — |
| 4 | §2/§4 первый гейт (классиф→предложить→подтвердить, «ТЗ недостаточно», приоритет сигналов) в `/start-task` | НЕ В СРЕЗЕ | `iva-analysis-base` не менялся; grep по start-task.md — 0 упоминаний режима/рисерча/lite/достаточности | 1-й гейт в start-task в этом срезе не делали (ожидаемо) |
| 5 | §3 второй слой — полный гейт пересмотра во ВСЕХ режимах + промпт «ГЕЙТ ПЕРЕСМОТРА» | ЧАСТИЧНО / НЕ В СРЕЗЕ | единого промпта нет; `tacticum-development-core` не менялся. Есть фрагменты: bugfix scope-tripwire (`bug-fix/SKILL.md` «Scope tripwire»), research Phase 4 outcome-gate, lite «Escalation mid-flight» | полный 2-й слой в run-implementation не делали (ожидаемо) |
| 5 | §3 handoff (`Tasks/<N>/handoff.md`, маппинг наработок, фиксация в report.md) | ЧАСТИЧНО / НЕ В СРЕЗЕ | `handoff.md` как файл нигде не создаётся; research отдаёт handoff через wiki/Jira (cross-role) + артефакты в Tasks/; lite «mini-plan and code carry over as handoff into /start-task» без файла; bugfix пишет `fix.md` | единый handoff-протокол — часть невыполненного 2-го слоя |
| 6 | §6 п.1 research-лейн `tacticum-research-base` | ЗАКРЫТО | лейн создан + composed в роли | — |
| 6 | §6 п.2 ADR-first вход `/start-task` | НЕ В СРЕЗЕ | правка `/start-task` (analysis) не входит | — |
| 6 | §6 п.3 первый гейт (start-task + fix-bug триаж) + расширение bugfix типами | ЧАСТИЧНО | расширение лайта сделано отдельным лейном (не в bugfix); первого гейта нет; fix-bug триаж уточнён под routing lite | — |
| 6 | §6 п.4 второй слой + handoff | НЕ В СРЕЗЕ | см. #5 | «после валидации пунктов 1-2» по плану ТЗ |
| 7 | Routing `/fix-bug` vs `/lite-task` (companion bugfix) | СООТВЕТСТВУЕТ | `bug-fix/SKILL.md`: «restore intended behaviour → /fix-bug; small change (no new screen/flow/module) → /lite-task; new screen/flow/module, ADR, dep вне каталога, серверный контракт, >10ф/>3мод → /start-task». Идентичное правило в lite `SKILL.md` «When this lane — and when NOT» и в README lite | консистентно с proposal §1/§4 |

## Реальные отклонения содержания (не «ещё не сделано»)

**D1 — lite реализован как ОТДЕЛЬНЫЙ лейн, а не расширение bugfix-лейна.**
Тип: расхождение с *дефолтной рекомендацией* ТЗ, санкционированное решением.
- Proposal §1.1/§1.2/§6 ставит основным вариантом «расширить `tacticum-bugfix-base` типами refactoring-S/feature-S», а отдельный `tacticum-lite-base` — как *альтернативу* «только если owner захочет держать bugfix-лейн идеологически чистым».
- Реализация выбрала альтернативу (вариант Б). Обоснование зафиксировано в `tacticum-lite-base/README.md`: feature-S намеренно МЕНЯЕТ поведение, а инвариант bug-fix «restore → /fix-bug, change → /start-task»; посадка change-семантики в restore-only лейн разрушила бы инвариант; guardrail «живые лейны не переписываем».
- Вывод: это осознанная смена решения относительно дефолта ТЗ (ТЗ такой вариант допускал явно), а не молчаливое расхождение. Стоит убедиться, что owner/ГД санкционировал именно Б.

**D2 — companion-правка bugfix сузила инвариант «change → /start-task» до «change → /lite-task».**
Тип: следствие D1, консистентно.
- kmp-оригинал и исходный bugfix-лейн эскалировали изменение поведения сразу в полный цикл. На ветке (коммиты 1102198, 0f6df10) правило в `bug-fix/SKILL.md` изменено: мелкое изменение поведения (feature-S) уходит в `/lite-task`, а не в `/start-task`.
- Это правка ШАРЕНОГО живого лейна `tacticum-bugfix-base`. README lite сам помечает её как «⚠️ делается ТОЛЬКО тимлидом после сигнала ГД». Проверить, что сигнал/санкция были, — правка чужого лейна.

**D3 — handoff-механика lite при эскалации расходится с §3.**
Тип: расхождение содержания (не только «не сделано»).
- Proposal §3 требует `Tasks/<N>/handoff.md` для перехода lite→полный (мини-план и код → вход PIN «stage 0 уже сделано»).
- Реализация lite (`SKILL.md` «Escalation mid-flight» + «No artifact files»): при эскалации ордер перепечатывается в ЧАТ с пометкой «ТРЕБУЕТСЯ эскалация», код «carry over as handoff», но файл `handoff.md` НЕ создаётся — лейн принципиально «no artifact files».
- Это прямое напряжение между «lite ничего не пишет в файлы» (перенос из kmp) и «handoff.md обязателен» (§3). Пока полный 2-й слой не построен — не проявляется, но при его реализации потребует решения: где живёт handoff при выходе из безфайлового lite. Заметка на будущее для ответственной роли.

## Итог
- Всё, что заявлено «в этом срезе» (research-лейн, lite-лейн, companion-routing bugfix, врезки в роли), — реализовано и по содержанию соответствует ТЗ; перенос kmp-механики в lite точный, стек-специфика корректно вынесена в [REPO-СЛОЙ].
- Первый гейт (§2/§4) и полный второй слой + единый handoff (§3) — НЕ в срезе (ожидаемо, `iva-analysis-base`/`tacticum-development-core` не тронуты).
- Реальные расхождения содержания: D1 (вариант Б вместо дефолтного расширения bugfix — санкционированный выбор), D2 (сужение инварианта в шаренном bugfix-лейне — проверить санкцию), D3 (безфайловый lite vs обязательный handoff.md §3 — заложенное напряжение к моменту постройки 2-го слоя).

canon НЕ пишу — повышает ответственная роль (тимлид).
