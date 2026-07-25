---
title: fidelity-tz2-completeness
type: note
permalink: tacticum/00-board/fidelity-tz2-completeness
tags:
- draft
---

# fidelity-tz2-completeness

status: draft
Роль: explorer (read-only). Репо: /Users/bubblemac/tacticum/tacticum-dev, main @ 928fe37.
Задача: проверка ПОЛНОТЫ реализации ТЗ#2 (режимы workflow) против всего proposal Солонко (§0–§6 + research-task-flow-scenario). Только ТЗ#2 режимы, ось-2 (#145) не смотрел.

Пути реализации:
- lite: templates/tacticum-lite-base/ingredients/{commands/lite-task.md, skills/lite-task-workflow/SKILL.md, .../references/work-order-template.md}
- research: templates/tacticum-research-base/ingredients/{commands/start-research.md, skills/research/SKILL.md}
- bugfix: templates/tacticum-bugfix-base/ingredients/{commands/fix-bug.md, skills/bug-fix/SKILL.md}
- 1-й гейт: templates/iva-analysis-base/ingredients/commands/start-task.md (Гейт классификации)
- 2-й слой: templates/tacticum-development-core/ingredients/commands/run-implementation.md (Mode-review gate)
- role-манифесты: iva-role-{go,web,kmp,ios} depends_on = core+development-core+<стек>+bugfix+lite+research; iva-role-analyst = core+analysis+research.

## Таблица: элемент proposal → вердикт → доказательство → что доделать

| # | Элемент proposal | Вердикт | Доказательство (путь:строка) | Если ПРОБЕЛ/ЧАСТИЧНО |
|---|---|---|---|---|
| §1 | Багфикс `/fix-bug` (лейн+команда) | РЕАЛИЗОВАНО | fix-bug.md + bug-fix/SKILL.md; в ролях (iva-role-go/manifest.yaml:22,38) | — |
| §1 | Лайт `/lite-task` = refactoring-S + feature-S | РЕАЛИЗОВАНО | lite-task.md:6-9; SKILL.md:16,86-88 (2 типа) | — |
| §1 | Полный `/start-task`+`/run-implementation` | РЕАЛИЗОВАНО | start-task.md; run-implementation.md | — |
| §1 | Research `/start-research` (новый лейн) | РЕАЛИЗОВАНО | start-research.md; research/SKILL.md; в dev+analyst ролях | — |
| §1 | Рефакторинг = refactoring-S в лайте; кампании отдельно | ОТЛОЖЕНО-по-ТЗ | proposal §1 явно «кампании — по спросу»; refactoring-S ∈ lite SKILL.md:87 | — |
| §1.1 | 2 типа (refactoring-S/feature-S) | РЕАЛИЗОВАНО | SKILL.md:86-88 таблица типов | — |
| §1.1 | Диагностика ДО правок (mandatory each type) | РЕАЛИЗОВАНО | SKILL.md:114-124 «Step 1 — Diagnose (read-only)» | — |
| §1.1 | Рабочий ордер + ок-гейт, классификация первой строкой | РЕАЛИЗОВАНО | SKILL.md:136-154; work-order-template.md | — |
| §1.1 | Список принудительной эскалации | РЕАЛИЗОВАНО | SKILL.md:91-99 (новый экран/диалог/модуль/dep вне catalog/серв.контракт/>10 файлов/>3 модулей) | — |
| §1.1 | Сознательный отказ от KB внутри лайта | РЕАЛИЗОВАНО | SKILL.md:70-76 «NOT required: external KB … do not call» | — |
| §1.1 | Хрупкие зоны = обязательный smoke (не эскалация) | РЕАЛИЗОВАНО | SKILL.md:107-112, 228 «device/manual smoke mandatory» | — |
| §1.1 | test-first + frozen tests | РЕАЛИЗОВАНО | SKILL.md:167-197 (frozen + snapshot-diff proof) | — |
| §1.2-1 | Режим = лейн (посадка на роли+лейны) | РЕАЛИЗОВАНО | все *-base как процесс-лейны; depends_on в ролях | — |
| §1.2-2 | 1-й гейт в ДВУХ местах: start-task И триаж fix-bug | РЕАЛИЗОВАНО | start-task.md:12-60 (классификация); bug-fix/SKILL.md:22-42 «When this lane — and when NOT» = маршрутизация по объекту (не только tripwire) | — |
| §1.2-3 | Гейт знает состав роли (нет лейна → re-provision/передать) | РЕАЛИЗОВАНО | start-task.md:48-51 «Состав роли (обязательно)… добавить лейн (re-provision) или передать роли Y» | — |
| §1.2 | **ADR-first вход = параметр /start-task** | **ПРОБЕЛ** | start-task.md принимает только ТЗ($1)+dir($2); хэндофф :70-71 всегда «generate … ADR …» с нуля; НЕТ ветки «подан ADR → работать от него». Приёмник ADR-first не реализован, хотя research на него ссылается (research/SKILL.md:173; start-research.md:56,60) | добавить (см. список ниже) |
| §1.2-5 | Смена режима 2 вида: внутри роли (handoff.md) / между ролями (wiki/Jira) | РЕАЛИЗОВАНО | run-implementation.md:63 (cross-lane не автоматом), :68-77 handoff.md; research/SKILL.md:173 (межролевой через wiki/Jira) | — |
| §2 | Шаг0 достаточность → до 3 вопросов | РЕАЛИЗОВАНО | start-task.md:19-22 | — |
| §2 | Порядок сигналов research→составное→рефактор→лайт→иначе-полный | РЕАЛИЗОВАНО | start-task.md:26-42 (порядок: РИСЕРЧ→СОСТАВНОЕ→РЕФАКТОРИНГ→ЛАЙТ→БАГФИКС→полный; багфикс добавлен как отд. лейн — консистентно) | — |
| §2 | СОСТАВНОЕ → split A→B реально предлагается | РЕАЛИЗОВАНО | start-task.md:30-32, 56-57 «предложи split на задачи A→B» | — |
| §2 | «Чему не доверять» (тип тикета/глагол/размер) | РЕАЛИЗОВАНО | start-task.md:45-46 | — |
| §2 | Проверка состава роли | РЕАЛИЗОВАНО | start-task.md:48-51 | — |
| §3 | Контрольные точки 2-го слоя | РЕАЛИЗОВАНО (dev-цикл) | run-implementation.md:46-51 (plan in hand; ack-маркеры; blocker; 2-й провал verify) | — |
| §3 | lite→полный (ордер готов: экран/сценарий/модуль/dep/контракт/>10ф/>3м/KB) | РЕАЛИЗОВАНО | SKILL.md:91-99 + 235-243 «Escalation mid-flight» | — |
| §3 | lite→research(мини)/полный (в ходе: файлы вне ордера / корень не подтвердился / verify падает 2й раз) | **ЧАСТИЧНО** | lite SKILL.md:235-243 эскалирует только → full cycle; ветки → research (мини) и триггеров «вне ордера / корень / 2й провал verify» в lite НЕТ | добавить строку в lite Escalation |
| §3 | полный→lite (PIN=1 стадия без UI/классов) | РЕАЛИЗОВАНО | run-implementation.md:56 | — |
| §3 | полный→research (Фаза1: KB молчит / механизм неясен / шаги неизвестны) | ЧАСТИЧНО | run-implementation.md:55 (mechanism unknown → /start-research) — но на входе реализации, НЕ в Фазе1 design (в tacticum-workflow агенте mode-review нет) | врезка в analysis (см.ниже) |
| §3 | полный→split «независимая вторая задача/заодно» (Фаза1) | **ПРОБЕЛ (в Фазе1)** | run-implementation.md:53-54 покрывает split ТОЛЬКО для prerequisite и root-cause (Фазы3-4); триггер «в ТЗ вскрылись независимые работы» на этапе Фазы1 не реализован (design-фаза без mode-review) | врезка в analysis |
| §3 | Фазы3-4→split (coder блокер / test-runner исчерпал 3 итерации) | РЕАЛИЗОВАНО | run-implementation.md:53-54 | — |
| §3 | research→lite (тривиальное решение) | РЕАЛИЗОВАНО | research/SKILL.md:169-172 «Known/trivial → /lite-task» | — |
| §3 | research→задача /start-task ADR-first | РЕАЛИЗОВАНО-как-предложение | research/SKILL.md:173 — но приёмник ADR-first = ПРОБЕЛ (см. выше) | зависит от ADR-first |
| §3 | refactor→полный (поведение меняется) | ЧАСТИЧНО | нет отдельной строки; покрыто общей эскалацией лайта (SKILL.md:91-105, объект=подсистема→escalation) | опц.: явная строка |
| §3 | Формула охвата (>10 файлов/>3 модулей) | РЕАЛИЗОВАНО | SKILL.md:99; fix-bug/SKILL.md:41; run-implementation косвенно | — |
| §3 | handoff.md формат (причина/сделано/выяснено/НЕ подтвердилось/вопросы) | РЕАЛИЗОВАНО | run-implementation.md:68-77; lite SKILL.md:242-243 (исключение из no-files) | — |
| §3 | Маппинг наработок (lite→PIN stage0; полный→research; Фазы3-4→split стеш) | РЕАЛИЗОВАНО | run-implementation.md:73 | — |
| §4 | Промпт гейта (Шаг0-2) | РЕАЛИЗОВАНО | start-task.md:19-57 | — |
| §4 | Промпт гейта Шаг3 «исполнение с поправками» + ИНФРА-признак (радиус, снятие стоп-правил на toolchain) | ЧАСТИЧНО | Шаг3-исполнение вынесено в SKILL-ы лейнов (архит. реорг — ОК); но ИНФРА-свойство (предупреждение о радиусе + toolchain-ошибка = предмет работы, не блокер) нигде не реализовано | опц., низкий приоритет |
| §4 | Промпт 2-го слоя (вставка во ВСЕ режимы) | РЕАЛИЗОВАНО | run-implementation.md:44-77; lite SKILL:235-243; bug-fix SKILL:149-164; research SKILL:164-180 | — |
| §5 | Диалоги-примеры | ОТЛОЖЕНО-по-ТЗ | иллюстрации proposal, не артефакт реализации | — |
| §6 | research-лейн | РЕАЛИЗОВАНО | tacticum-research-base + в ролях | — |
| §6 | ADR-first вход /start-task | **ПРОБЕЛ** | см. §1.2 выше | — |
| §6 | 1-й гейт (start-task + fix-bug триаж) | РЕАЛИЗОВАНО | start-task.md + bug-fix SKILL routing | — |
| §6 | 2-й слой + handoff | ЧАСТИЧНО | реализован в dev-цикле (run-implementation); Фаза-1 (design) слой отсутствует | врезка в analysis |
| bugfix триаж → research (плавающий дефект/неизвестный механизм) | ЧАСТИЧНО | fix-bug scope-tripwire (SKILL:149-164) предлагает только /lite-task и /start-task, не /start-research | опц.: добавить research-выход |

## Реальные пробелы (НЕ «отложено-по-ТЗ») — для укрупнённого добора одним PR

1. **ADR-first вход как параметр `/start-task`** (ПРОБЕЛ, §1.2 стр.149 + §6 п.2). Приёмник не реализован: start-task.md берёт только ТЗ, tacticum-workflow всегда пишет ADR с нуля. Research-лейн уже отдаёт наработку «в /start-task ADR-first», но принять её некому — разрыв контракта хэндоффа research→build.
   Объём: СРЕДНИЙ. Нужен: (а) доп. параметр/распознавание поданного ADR-черновика в start-task.md; (б) врезка в tacticum-workflow агент Phase 1 — «если подан ADR: работать ОТ него, проверить наличие context/decision/alternatives/consequences, сверить упомянутые API/модули с кодом (KB), расхождения показать до планирования, НЕ писать решение с нуля» (по research-task-flow-scenario Step 2). Правки в 2 файлах (+ codex-зеркало tacticum-workflow.toml).

2. **Второй слой в фазе постановки (Фаза-1 design)** (ЧАСТИЧНО→ПРОБЕЛ для 3 триггеров, §3 строки «полный/Фаза1»). Mode-review есть только в run-implementation.md (вход реализации). В iva-analysis-base / агенте tacticum-workflow пересмотра режима после Фазы1 нет. Не отлавливаются на этапе design: полный→lite (PIN выродился), полный→research (механизм неясен/шаги неизвестны), полный→split (в ТЗ независимая вторая задача). Design-конвейер отработает целиком до отлова на входе run-implementation.
   Объём: НЕБОЛЬШОЙ. Врезка «Mode-review после Фазы1» в tacticum-workflow.md (+ .toml зеркало) с 3 триггерами и форматом предложения (тем же, что run-implementation.md:60-61).

3. **lite→research(мини) + внутрипроцессные триггеры лайта** (ЧАСТИЧНО, §3 строка 2). lite «Escalation mid-flight» эскалирует только вверх в полный; нет ветки → research и триггеров «правишь файлы вне ордера / корень не подтвердился / verify падает 2-й раз в одном месте».
   Объём: МАЛЫЙ. Дополнить секцию Escalation в lite-task-workflow/SKILL.md (+ отражение в lite-task.md).

Мелочь (низкий приоритет, можно тем же PR или отложить):
4. bugfix scope-tripwire без выхода в /start-research для плавающего дефекта/неизвестного механизма (fix-bug SKILL:149-164). Малый.
5. ИНФРА-свойство в гейте (предупреждение о радиусе + toolchain-ошибка не блокер) — §1.2/§4. Малый, отчасти «свойство, не режим».
6. refactor→полный «поведение меняется» — явной строки нет, покрыто общей эскалацией. Опционально.

Итог: ядро ТЗ#2 (4 режима + лейны + 1-й гейт + маршрутизация fix-bug + 2-й слой dev-цикла + handoff + role-wiring) реализовано. Один существенный контрактный разрыв — **ADR-first вход /start-task** (замыкает research→build). Плюс 2-й слой не покрывает design-фазу (Фаза-1 триггеры) и лайт не умеет →research. Всё это укладывается в один добор-PR на 4-5 файлов (+ codex-зеркала).
