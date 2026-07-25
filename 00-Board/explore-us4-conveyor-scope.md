---
title: explore-us4-conveyor-scope
type: note
permalink: tacticum/00-board/explore-us4-conveyor-scope
status: draft
role: explorer
for: lead-fr
scope: ТЗ#3 US#4 (dev-конвейер К-1…К-5)
repo: tacticum-dev @ origin/main 928fe37
date: 2026-07-24
tags:
- explore
- us4
- conveyor
- dev-profiles
- lead-fr
---

# explore-us4-conveyor-scope

Карта скоупа US#4 (доработка dev-конвейера К-1…К-5) + SHARED-статус с другими лидами. Read-only разведка. Канон не пишу.

Репо: `/Users/bubblemac/tacticum/tacticum-dev`, origin/main = `928fe37` (после #145 kmp-axis2 и #144 ds-web-to-kmp).

## КОРРЕКЦИЯ вводной (важно для лида)

ТЗ §4 и постановка предполагают, что brd/pin/tests/start-task ходят по **mirror-протоколу** (`_mirrors.yaml`, owner→mirror). **Это не так.** `_mirrors.yaml` покрывает только **стек-скиллы** (coder, tester, test-runner, angular-*, kmp-*, ios-*, qt-*, firebird-* и т.п.) в парах `*-development-base (owner) → *-brownfield (mirror)`. Скиллы **конвейера** (brd-authoring / pin-authoring / tests-authoring / start-task / adr-authoring / kb-navigation) в `_mirrors.yaml` **отсутствуют** и НЕ имеют байтовой CI-сверки (`scripts/check_mirror_sync.py` + `tests/catalog/test_role_replacement_parity.py` их не трогают — grep по brd/pin/tests в `tests/` пуст).

Реальная модель владения конвейер-скиллами — **композиция ADR-0056/E-COMP (`depends_on`)** + ручная раскатка «roll out to dependents + standalone profiles»:
- `tacticum-dev-base` v0.2.5 = **БАЗА / канонический владелец** workflow-ядра (KB navigation, BRD/ADR/PIN/TESTS design flow, phase commands, start-task). Формулировка из манифеста: «stack-agnostic workflow core». Преемник депрекированного `brownfield-task-workflow` v0.3.10 (deprecated=true, frozen).
- **Новые композитные профили** (`iva-ios-brownfield`, `firebird-web-brownfield`) объявляют `depends_on: [tacticum-dev-base, tacticum-ui-base]` и **наследуют brd-authoring + start-task из базы** (своих копий НЕТ), несут только стек-дельту (own pin-authoring + own tests-authoring).
- **Старые монолиты** (`iva-web-brownfield`, `iva-kmp-brownfield`, `iva-brownfield-mail`, `iva-rn-brownfield`) **несут собственные копии** brd/pin/tests/start-task. Их brd синхронизируется с базой ВРУЧНУЮ (нет CI-гейта) — исторически через коммиты «roll out /start-task fix to dependents + 5 standalone profiles» (Taiga #697, #124, #127).

## 1. Профили-носители конвейера (версии + что несут)

Проверено `find … ingredients/skills/{brd,pin,tests}` + `commands/start-task.md`. own = собственная копия тела; inh = наследуется из `tacticum-dev-base` через depends_on.

| Профиль | Версия | deprecated | brd | pin | tests | start-task | Модель |
|---|---|---|---|---|---|---|---|
| tacticum-dev-base | 0.2.5 | нет | **own (владелец)** | own(generic) | own(generic) | **own (владелец)** | база |
| iva-web-brownfield | 0.2.1 | нет | own | own | own | own | монолит |
| iva-kmp-brownfield | 0.5.0 | нет | own(diverged) | own | own | own | монолит / mirror-of kmp-dev-base |
| iva-brownfield-mail | 0.7.1 | нет | own | own | own | own | монолит |
| iva-rn-brownfield | 0.5.1 | нет | own | own | own | own | монолит |
| iva-ios-brownfield | 0.1.2 | нет | **inh** | own | own | **inh** | композит (depends_on base+ui) |
| firebird-web-brownfield | 0.1.2 | нет | **inh** | own | own | **inh** | композит (depends_on base+ui) |
| iva-go-backend-brownfield | 0.2.6 | **ДА** | own | own | own | own | frozen — НЕ трогать |
| brownfield-task-workflow | 0.3.10 | **ДА** | own(frozen) | own | own | own | frozen предшественник — НЕ трогать |
| iva-analysis-base | 0.1.5 | нет | own | own + multi-container-pin | own | own | аналитик-лейн (территория lead-fr) |

Тела: `templates/<profile>/ingredients/skills/{brd-authoring,pin-authoring,tests-authoring}/SKILL.md` и `.../commands/start-task.md`.

### Байтовая идентичность (md5)

- **brd-authoring**: 6 профилей идентичны `d0df3c84…` (web, mail, rn, go-backend, analysis-base, tacticum-dev-base). **Расходятся:** `iva-kmp-brownfield` = `18d93be7…` (диверг под axis-2), `brownfield-task-workflow` = `5934d0db…` (frozen). ios/firebird — своей копии нет (inh).
- **pin-authoring**: почти все РАЗНЫЕ (стек-специфичные). Совпадают только `iva-brownfield-mail` = `tacticum-dev-base` = `34ff8a3c…`. web/kmp/ios/go/firebird/rn — каждый свой.
- **tests-authoring**: ВСЕ разные (стек-специфичные).
- **start-task**: почти все разные. Совпадают `iva-brownfield-mail` = `iva-rn-brownfield` = `87a2074c…`.

Вывод: brd-authoring — фактически один канонический body (владелец = tacticum-dev-base), размноженный вручную по 6; kmp уже отклонился. pin/tests/start-task — genuinely per-stack, синка нет by design.

## 2. Владение (по факту, не по _mirrors.yaml)

`_mirrors.yaml` пары (owner → mirror), ингредиенты = стек-скиллы (НЕ конвейер):
- iva-analysis-base → iva-fr-analyst : api-contracts-discovery, design-system-discovery, fr-authoring, mockup-authoring, start-feature, update-feature
- iva-kmp-development-base → iva-kmp-brownfield : coder, tester, test-runner, kmp-* (architecture-guards, build-verification, module-integration, run-launch, ui-testing, local-knowledge-routing), compose-multiplatform-ui, calls-voip-fragile-zone, codegraph-first-navigation, proguard-r8-discipline
- iva-web-development-base → iva-web-brownfield : coder, tester, test-runner, angular-* , ngrx-signals-state, nx-workspace-discipline, openapi-codegen-pipeline, ivcs-libs-contract, surgical-change-discipline, local-skill-authoring, web-local-knowledge-routing, webrtc-conference-fragile-zone
- iva-mail-development-base → iva-brownfield-mail : coder, tester, test-runner, cpp/qt-* , integration-checklist
- iva-ios-development-base → iva-ios-brownfield : coder, tester, test-runner, ios-* , swift-observation-state, ivalink-realtime, fastlane-build-verification, calls-av/sdk-bridges fragile
- firebird-web-development-base → firebird-web-brownfield : coder, tester, test-runner, firebird-*, rtk-effect-state, vite-* , jump-protocol-fragile-zone

**Ни один из brd/pin/tests/start-task/adr-authoring в этих списках НЕ значится.** → правки конвейера под US#4 идут НЕ через mirror-владельца, а по модели §1 (база + ручная раскатка по монолитам; композиты наследуют).

## 3. SHARED-статус (для окна ГД)

Открытых конкурентных веток по конвейер-профилям сейчас НЕТ (все feat/*-ветки лидов уже влиты в main: #142 modes, #143 us1-fr, #144 ds, #145 kmp-axis2; ahead-of-main = 0). Впереди main только `feat/718-tz-genre` и `feat/717-update-feature-reconciliation` — обе трогают ТОЛЬКО `iva-fr-analyst` (своя территория lead-fr). Контенция — на уровне **владения профилями и рабочих потоков лидов**, не открытых веток.

Привязка потоков (по git-истории per-profile):

- **KMP-лейн (iva-kmp-brownfield + iva-kmp-development-base) — ГОРЯЧАЯ ЗОНА, двойная контенция:**
  - lead-modes: `#145 feat/kmp-multirepo-axis2` → `52d47e7 iva-kmp-brownfield: cross-repo source/reference (axis-2)` — только что влит, правил ТЕЛО iva-kmp-brownfield (из-за чего его brd diverged).
  - lead-ds (ТЗ#1 web-to-kmp): `d715669/65e4fbf/39ae642` → бампы `iva-kmp-development-base` 0.5.0→0.6.0 + скилл `web-to-kmp-screen-port` / `web-to-kmp-source-reference`. Активный поток.
  - → US#4 правка kmp brd/pin/tests = редактирование `iva-kmp-brownfield`, который в активном flux у ДВУХ лидов (modes + ds). **Нужно окно.**

- **iva-web-brownfield — заявлено shared с lead-ds; git НЕ подтверждает прямого редактирования:** последние коммиты web-brownfield = `#727 serena popups`, `#725 fix-bug lane`, `#697 start-task` — это общие раскатки, НЕ ds. Работа ds (web-to-kmp) идёт на СТОРОНЕ KMP (web — только как read-source). Прямого пересечения тела iva-web-brownfield с ds в истории нет. **Статус: НЕ подтверждено как shared по коду** — контенция условная (web-brownfield как «источник истины» для ds-портирования, но не место правок). Уточнить у lead-ds планирует ли он трогать web-brownfield в остатке ТЗ#1.

- **iva-web-development-base (owner web-стека)** — `35e0cb0 пост-мерж синк после #122–#131 (зеркала отработали)`, `508854f матрица лейнов ADR-0059`. Общий инфраструктурный поток, не конкретный лид. US#4 его НЕ трогает (конвейер живёт в brownfield/base, не в web-dev-base).

- **tacticum-dev-base (владелец brd/start-task)** — последние: `#727 serena`, `#724 fix-bug top-level`, `#124 kb-navigation`. Это общие кросс-профильные раскатки (не закреплённый лид). US#4 К-1/К-3/К-5 будет править ЕГО brd/start-task → задевает всех, кто наследует (ios, firebird) и требует ручной раскатки в 4 монолита. **Это самый «широковещательный» файл — координировать раскатку.**

- **iva-analysis-base** — территория lead-fr (mirror-owner fr-authoring; US#0/#1/#2 ТЗ#3 уже прошли: `c028882` FR v2, `43aaf68` CT-n, `381f906` revert разъединения зеркала). Несёт own brd/tests/start-task/multi-container-pin. Внутри лейна lead-fr — контенция внутренняя, окно не нужно.

- **iva-ios-brownfield / firebird-web-brownfield** — свежесозданные композиты (#679/#672), никем активно не правятся; brd/start-task наследуют из базы автоматически. Правки US#4 их достигнут через базу без прямого редактирования (кроме own pin/tests, если К-2 требует стек-специфики).

## 4. Что конкретно правится под К-1…К-5 + объём

- **К-1 (brd читает Часть 1 §1.4/§1.5 вместо П.A/П.B, распознавание v1/v2 по fr_skeleton):** правка `brd-authoring/SKILL.md`. Владелец — `tacticum-dev-base`. Раскатка вручную в 4 монолита (web, kmp[diverged!], mail, rn) + analysis-base(own). ios/firebird наследуют. **~6 копий brd + kmp отдельно (уже разошёлся — merge-риск).**
  - Контракт уже задан продюсером: в `iva-analysis-base/ingredients/skills/fr-authoring/SKILL.md:431` шапка FR несёт `fr_skeleton: 2 <!-- … Читает dev-конвейер (/start-task): v2 = FT/UC в Части 1 (§1.4/§1.5) -->`. Продюсер-сторона (US#1) УЖЕ в main и явно документировала handoff под US#4.
- **К-2 (наследование CT/DM/EV: brd ссылается, pin реализует, tests «Covers: CT-n», report статус):** правка brd (ссылки) + **каждого per-stack pin-authoring и tests-authoring** (web/kmp/mail/ios/rn/firebird = 6×2 разных тел, синка нет) + report-раздел (в pin/start-task/run-implementation). **Самый объёмный пункт: pin/tests стек-специфичны — по каждому лейну своё.**
- **К-3 (гейт: проектный раздел без D-n → BLOCKED):** гейт в start-task / run-implementation. start-task: own у 4 монолитов + база (ios/firebird наследуют). ~5 копий start-task.
- **К-4 (таблица расхождений FR↔KB на проектные разделы, kb_verify_api_exists для контракт/DM):** правка pin-authoring (verification-шаг) — per-stack; связано с `pin-api-verification` (есть в analysis-base). Проверить наличие аналога в brownfield pin.
- **К-5 (маркер fr_skeleton:2, отсутствие=v1):** чтение маркера в brd + start-task. Тот же набор файлов, что К-1.

**Оценка объёма:** «широкие» файлы (brd, start-task) — ~5-6 копий каждый, но тело +/- одинаковое (кроме kmp-brd diverged) → механическая раскатка. «Глубокие» файлы (pin, tests) — 6 разных стек-тел × 2, каждое требует содержательной адаптации под К-2/К-4 → основной труд здесь. go-backend + brownfield-task-workflow — deprecated, НЕ трогать.

**Блокер US#3:** новых навыков-анализаторов DM-n/EV-n в `iva-analysis-base/ingredients/skills/` ПОКА НЕТ (есть только api-contracts-discovery для CT-n). Серии DM/EV, которые К-2 должен наследовать, появятся только когда US#3 сядет в main. До того К-2 реализуемо частично (CT-n есть, DM/EV — заглушка/после US#3).

## 5. Синхронность бандла (порядок релиза)

Что из бандла структуры УЖЕ в main:
- US#1 (fr-authoring FR v2, двухзонная модель, `fr_skeleton:2`, валидатор границы) — `c028882` ✅ в main (owner+mirror).
- US#2 (api-contracts CT-n, формат 3.1) — `43aaf68` ✅ в main (owner+mirror).
- US#0 (откат разъединения зеркала analysis↔fr-analyst) — `381f906` ✅ в main.
- US#3 (DM/EV навыки) — ❌ НЕ в main (навыков нет в analysis-base).
- US#5 (update-feature реконсиляция) — ❌ `feat/717-update-feature-reconciliation` 1 ahead, не влит.

Порядок для US#4: К-5/К-1 (чтение fr_skeleton + FT/UC из §1.4/§1.5) **опираются на US#1**, который уже в main и уже прописал handoff-контракт → К-1/К-5 можно делать сразу. К-2 в части CT-n опирается на US#2 (в main) → можно. К-2 в части **DM/EV — ждёт US#3**. Переходная совместимость (К-5: нет маркера=v1) делает US#4 backward-safe: старые FR v1 продолжают читаться, поэтому US#4 можно мержить до полной раскатки v2 в проде. ТЗ §4 «релизится синхронно» = US#4 не должен уехать в прод РАНЬШЕ US#1 (иначе конвейер не поймёт v2), но US#1 уже в main — риск снят; критичный порядок остаётся для US#3→К-2(DM/EV).

## Карта окон для ГД

| Профиль/лейн-владелец | Кто ещё трогает | Вердикт |
|---|---|---|
| **iva-kmp-brownfield** (+ kmp-development-base) | lead-modes (#145 axis-2, правил тело) + lead-ds (web-to-kmp, бампит kmp-dev-base) | **НУЖНО ОКНО** — двойная контенция, brd уже diverged, merge-риск |
| **tacticum-dev-base** (владелец brd/start-task) | общие кросс-раскатки (#724/#727), нет закреплённого лида | **Координация раскатки** — правка задевает ios/firebird (наследники) + 4 монолита; согласовать окно раскатки, конкурентного лида нет |
| iva-web-brownfield | lead-ds — по git НЕ подтверждено (web = read-source, не место правок) | **Уточнить у lead-ds**; вероятно можно независимо |
| iva-web-development-base | инфра-синк, US#4 его не трогает | Независимо (вне скоупа US#4) |
| iva-brownfield-mail / iva-rn-brownfield | общие раскатки #725; закреплённого лида нет | Можно независимо |
| iva-ios-brownfield / firebird-web-brownfield | свежие композиты, никто активно не правит; brd/start-task наследуют | Можно независимо (own pin/tests под К-2) |
| iva-analysis-base | lead-fr (своя территория ТЗ#3) | Внутри lead-fr, окно не нужно |
| iva-go-backend-brownfield, brownfield-task-workflow | deprecated=frozen | **НЕ трогать** |

## Открытые вопросы лиду
1. Подтвердить у lead-ds: планирует ли он править ТЕЛО `iva-web-brownfield` в остатке ТЗ#1, или web — только источник для порта в KMP.
2. KMP-окно: согласовать с lead-modes/lead-ds порядок (kmp brd уже разошёлся из-за axis-2 — К-1 по kmp делать ПОСЛЕ стабилизации axis-2, отдельным проходом).
3. К-2(DM/EV) поставить в зависимость от US#3 (навыков DM/EV ещё нет в main).