---
title: git-история и PR по ДС (трекер/вики — отдельно у лида)
type: note
status: draft
created: 2026-07-27
role: explorer
for: team-lead
repo: ~/tacticum/tacticum-dev (read-only)
tags:
- board
- design-system
- explore
- taiga
- wiki
- git-history
permalink: tacticum/00-board/explore-ds-tracker-wiki-2026-07-27
archived-at: 2026-08-04 10:01
---

# git-история и PR по ДС (трекер/вики — отдельно у лида)

> **Taiga и вики этим воркером НЕ проверялись — проверяет лид отдельно.**
> У воркера нет MCP-инструментов `taiga` и `wiki-mcp` (в тулсете только Read/Bash/Task*/SendMessage).
> Всё ниже — **только локальные источники**: git-история `~/tacticum/tacticum-dev` и рабочий vault.
> Read-only; ничего не создавал и не менял ни в репозиториях, ни в трекере, ни в вики.

**Терминология после поправки постановки:** #132 и #134 — **PR GitHub** в `TacticumApps/tacticum-dev`.
`iva-web 0.3.0` и `iva-mobile 0.2.0` — **версии в `design-system.yaml`**, приземлённые этими PR.
Совпадают ли эти номера с чем-то в Taiga — не проверено, проверяет лид.

---

## 1. Хронология: кто, когда, что менял в `design-systems/`

Полный список — **9 коммитов за всё время** (`git log --all -- design-systems/`). Больше в этой
папке не менял никто.

| # | Коммит | Дата | Автор | Что | Диф |
|---|---|---|---|---|---|
| 1 | `82dfe0b` | 2026-05-23 17:13 | **Diaret** | Add design system tokens, import script, ADRs — **рождение `iva-web` 0.1.0** | `iva-web/design-system.yaml` +18, `iva-web/tokens.json` +11862 → **2 файла, +11880** |
| 2 | `ed2272f` | 2026-05-23 18:53 | **Diaret** | Add design seed service, runner, and IVA mobile — **рождение `iva-mobile` 0.1.0** + пересборка iva-web | `iva-mobile/{yaml,tokens.json}` +18/+10920, `iva-web/{yaml,tokens.json}` правка → **4 файла, +13412/−2474** |
| 3 | `2e5f1f2` | 2026-06-18 17:33 | **Diaret** | Add IVA RN design system and tokens — **рождение `iva-rn` 0.1.0** | **2 файла, +729** |
| 4 | `4aad5e7` | 2026-07-19 22:16 | **Diaret** | Add tacticum-web design system and skill — **рождение `tacticum-web` 0.1.0** | `tacticum-web/{DESIGN.md +248, build_preview.py +167, design-system.yaml +22, preview.html +415, tokens.json +233}` → **5 файлов, +1085** |
| 5 | `8487a29` | 2026-07-22 15:09 | **Dmitry Solonko** | iva-web DS **0.2.0** — code-bindings ($extensions) — **ветка, в main НЕ попал** | **2 файла, +652/−1** |
| 6 | `6ccd176` | 2026-07-22 14:42 | **Dmitry Solonko** | то же, **squash-merge PR #122** — это версия, которая на main | **2 файла, +652/−1** |
| 7 | `2f42190` | 2026-07-23 14:42 | **Dmitry Solonko** | iva-web DS **0.3.0** — stable `figma_key` (**PR #132**) | `iva-web/design-system.yaml`, `iva-web/tokens.json` → **2 файла, +45/−37** |
| 8 | `bfbd33c` | 2026-07-23 17:11 | **Dmitry Solonko** | iva-mobile DS **0.2.0** — KMP code-bindings (**PR #134**) | `iva-mobile/{yaml,tokens.json}` → **2 файла, +446/−1** (в PR ещё +108 quickstart, см. п.2) |
| 9 | `5e28a5f` | 2026-07-24 14:49 | **Dmitry Solonko** | add **`iva-core`** design system (VCS/conference UI-kit) — **в main НЕ ВМЕРЖЕН** | `iva-core/{design-system.yaml +21, tokens.json +1040}` → **2 файла, +1061** |

Проверка `git merge-base --is-ancestor <c> origin/main`:

- **На main:** `82dfe0b`, `ed2272f`, `2e5f1f2`, `4aad5e7`, `6ccd176`, `2f42190`, `bfbd33c`.
- **НЕ на main:** `8487a29` (живёт только в `origin/feat/iva-web-ds-code-bindings`; его squash-двойник
  `6ccd176` — на main, дублем считать не надо) и **`5e28a5f` — `iva-core` DS**
  (только `origin/feat/iva-core-design-system`, `git diff --stat origin/main...` = 2 файла, +1061).

### ⚠ Факт, который стоит подсветить отдельно

**`design-systems/iva-core/` на main НЕТ.** На main лежат ровно четыре ДС: `iva-mobile`, `iva-rn`,
`iva-web`, `tacticum-web`. При этом на main **уже задеплоен скилл**, который к iva-core обращается:
`templates/iva-web-brownfield/ingredients/skills/iva-core-design-system/SKILL.md` (+83) и его копия
в `iva-web-development-base` (+83) приехали PR **#159** от 2026-07-25. То есть **скилл
конференц-поверхности на main есть, а самой ДС iva-core — нет**; она ждёт в невмерженной ветке
Solonko с 2026-07-24. Расхождение не проверял на предмет «так и задумано» — фиксирую как факт.

### Хронология версий (по `+version:` в `design-system.yaml`)

| ДС | 0.1.0 | 0.2.0 | 0.3.0 | Текущая на main |
|---|---|---|---|---|
| `iva-web` | `82dfe0b` 2026-05-23 (Diaret) | `6ccd176` 2026-07-22 (Solonko) | `2f42190` 2026-07-23 (Solonko) | **0.3.0**, `status: published`, `platform: web`, `framework_hint: react` |
| `iva-mobile` | `ed2272f` 2026-05-23 (Diaret) | `bfbd33c` 2026-07-23 (Solonko) | — | **0.2.0**, published, `platform: mobile`, `framework_hint: react-native` |
| `iva-rn` | `2e5f1f2` 2026-06-18 (Diaret) | — | — | **0.1.0**, published, mobile / react-native |
| `tacticum-web` | `4aad5e7` 2026-07-19 (Diaret) | — | — | **0.1.0**, published, web / react |
| `iva-core` | `5e28a5f` 2026-07-24 (Solonko) | — | — | **на main отсутствует** |

**Читаемый водораздел авторства:** до 2026-07-19 включительно все ДС заводит **Diaret**
(4 коммита, 4 «рождения»). С 2026-07-22 всё — **Dmitry Solonko** (5 коммитов, слой code-bindings
поверх токенов). Третьих авторов в `design-systems/` нет.

### Два расхождения метаданных, видные прямо в yaml

1. **`iva-mobile`: `framework_hint: react-native`, а биндинги — Compose Multiplatform.**
   `design-systems/iva-mobile/design-system.yaml` одновременно содержит `framework_hint: react-native`
   и описание «0.2.0: adds `$extensions."dev.tacticum.code-bindings"` (**stack compose-mp**) — …
   34 mobile components to Compose Multiplatform implementations in the KMP messenger
   `:core:design-system`». То есть `framework_hint` противоречит содержимому словаря.
2. **`iva-web`: `framework_hint: react`, а целевой репозиторий iva-one — Angular.** В том же файле
   описание: «hand-curated mapping of IVA-DS Figma master components to **Angular** implementations
   in iva-one `libs/design-system`». Расхождение известно и уже зафиксировано в
   `00-Board/map-sc3-and-remainder.md:80`, повторяю здесь как всё ещё живое на 2026-07-27.

### Проверенные мной числа словаря (не со слов коммита, а по файлам на main)

Разбор `tokens.json` → `$extensions."dev.tacticum.code-bindings"`:

- **`iva-web`: 49 компонентов, из них 32 с непустым `figma_key`** (совпадает с заявленным в
  коммите «32/49»). Ключи расширения: `schema, description, figma_file, codebase, storybook_base, usage`.
- **`iva-mobile`: 34 компонента, `figma_key` — 0 (ноль).** Ключи: `schema, stack, description,
  figma_file_key, codebase, gallery`. Это ровно то, что обещало тело `bfbd33c`: «figma_key pending
  until designers confirm the authoritative mobile DS file». **Открытая внешняя зависимость от
  дизайнеров, не снятая на 2026-07-27.**

---

## 2. Все PR и ветки, касающиеся ДС

Репозиторий использует **две схемы мержа одновременно** — merge-коммиты («Merge pull request #N
from …») и squash («… (#N)» в конце заголовка). Из-за squash часть PR в списке merge-коммитов
не видна; ниже учтены обе.

| PR | Ветка | Мерж-коммит | Дата | Автор мержа | Диф (файлов / строк) |
|---|---|---|---|---|---|
| **#106** | `feature/tacticum-web-design-system` | `a94b088` | 2026-07-19 | **Diaret** | 12 файлов, **+2362/−2**: `design-systems/tacticum-web/*` (DESIGN.md 248, preview.html 415, tokens.json 233, build_preview.py 167, yaml 22), тесты `test_tacticum_web_{tokens,contrast}.py` (+90/+70), план+спека в `docs/superpowers` (+902/+149), скилл `tacticum-design-tokens/SKILL.md` +45, `tacticum-internal-dev` manifest+CHANGELOG |
| **#122** | `feat/iva-web-ds-code-bindings` | `6ccd176` (**squash**) | 2026-07-22 | Dmitry Solonko | 2 файла, **+652/−1** — iva-web 0.2.0, словарь code-bindings |
| **#132** | `feat/iva-web-ds-figma-keys` | `df6b7d4` | 2026-07-23 | Dmitry Solonko | 2 файла, **+45/−37** — iva-web 0.3.0, стабильные figma_key |
| **#134** | `feat/iva-mobile-ds-code-bindings` | `01c1df4` | 2026-07-23 | Dmitry Solonko | 3 файла, **+554/−1** — iva-mobile 0.2.0 (+440 tokens) + `docs/user_manuals/iva-kmp-figma-mapping-quickstart.md` (+108) |
| **#144** | `feat/ds-web-to-kmp` | `034ef1b` | 2026-07-24 | Шульга Александр Алексеевич | 4 файла, **+555/−1**: `iva-kmp-development-base` — скиллы `web-to-kmp-screen-port` (+288), `web-to-kmp-source-reference` (+98), CHANGELOG +133, manifest |
| **#149** | `feat/ds-web-sc12` | `5552118` | 2026-07-24 | Шульга | 4 файла, **+404/−2**: `iva-web-brownfield` — `angular-ds-component-authoring` (+169), `angular-ds-component-usage` (+177), CHANGELOG +32, manifest +28 |
| **#154** | `feat/ds-web-mockup-figma` | `b9f5610` | 2026-07-24 | Шульга | 3 файла, **+197/−8**: `ui-mockup-match/SKILL.md` +153, CHANGELOG +48, manifest |
| **#159** | `feat/ds-web-axis1` | `7ffe389` | 2026-07-25 | Шульга | 12 файлов, **+489/−43**: `docs/user_manuals/iva-web-figma-mapping-quickstart.md` **+142**, новый скилл `iva-core-design-system` в двух пакетах (+83 ×2), правки `design-system-discovery` (+27), `angular-ds-component-{authoring,usage}` в обоих пакетах, два CHANGELOG, два manifest |
| **—** | `feat/iva-core-design-system` | **не вмержена** | 2026-07-24 | — | 2 файла, **+1061** — `design-systems/iva-core/{design-system.yaml,tokens.json}` |

**Смежное, но не про ДС в узком смысле** (по имени ветки попадает в фильтр, содержательно — про
композицию профилей и лимиты токенов): PR **#90** `feat/671-ui-base-multideps`, PR **#80**
`feat/648-depends-on-schema-seeder`, PR **#60** `fix/642-design-token-max-chars`, PR **#40**
`feat/arch-ui-pickers-autocompute-legends`, PR **#121** `feat/720-analyst-mockups`.

**Ветки по имени, релевантные ДС** (`git branch -r`): `feat/iva-core-design-system` (не вмержена),
`feat/iva-web-ds-code-bindings`, `feat/iva-web-ds-figma-keys`, `feat/ds-web-axis1`,
`feat/ds-web-mockup-figma`, `feat/ds-web-sc12`, `feat/ds-web-to-kmp`.

**Водораздел зон ответственности виден по PR:** **#106/#122/#132/#134 + iva-core** = серверная ДС
(словарь и токены), мержит **Solonko/Diaret**. **#144/#149/#154/#159** = скиллы и quickstart'ы,
мержит **Шульга**. Пересечение ровно одно: PR #134 (Solonko) занёс KMP-quickstart в
`docs/user_manuals/`, а PR #159 (Шульга) — веб-quickstart туда же.

---

## 3. Следы Taiga в репозитории — ответ фактом

**Taiga в репозитории есть, и обильно: 322 вхождения** слова `taiga` в отслеживаемых файлах
(`git grep -ci taiga | ...`), плюс **46 коммитов** (все ветки), чьё сообщение упоминает Taiga.
Дисциплина закреплена нормативно:

- `CLAUDE.md:36` — «**Taiga:** tenant `tacticum`, project `tacticum-dev` (project_id=12, slug
  `tacticum-tacticum_dev`). Все US, эпики, issues живут здесь. Каждая US **обязана** висеть на эпике».
- `CLAUDE.md:189` — «Issues, user stories и эпики живут в Taiga (project_id=12, slug
  `tacticum-tacticum_dev`)», ссылка на `docs/agents/issue-tracker.md`.
- `.serena/memories/feedback_issue_tracker_only.md:12` — «**Taiga is the single source of truth for
  outstanding work in this project.**»

Ссылки бывают трёх форм: `Taiga #N` в теле/заголовке коммита; `US #N` / `Issue #N` в заголовке;
и **полные URL** вида `https://project.cifragen.ru/project/tacticum-tacticum_dev/{epic|us}/N` в
ADR и `CONTEXT.md`.

### 3.1. Связаны ли ДС-задачи с трекером — раздельно по двум слоям

**Слой A — платформенная ДС (Design BC, 2026-05/06): СВЯЗАН, ссылок много.**

- **Эпик E27 #322 «Design as 6th bounded context»** — 9 вхождений URL
  `https://project.cifragen.ru/project/tacticum-tacticum_dev/epic/322`. Источники: `CONTEXT.md:825`
  («Implementation tracked в [Taiga E27 #322](…) (**8 US, refs #323-#330**)»), `docs/adr/README.md:75`,
  ADR-0026 (строки 6, 135), ADR-0027 (6), ADR-0028 (7, 11, 21, 134).
- Поимённые US того же эпика в ADR-0027/0029: **#324 (S2** — Pydantic-валидация DTCG),
  **#325 (S3** — идемпотентность сида / 409), **#326 (S4** — IVA bootstrap Tokens Studio),
  **#327 (S5** — CI seed flow), **#329 (S7** — MCP tools impl), #330.
- **Эпик #540 «Design Studio»** — `docs/superpowers/specs/2026-06-18-design-studio-prd.md:3`:
  «**Эпик:** Taiga tacticum-dev #540 (E «Design Studio»)», статус документа `ready-for-agent`,
  основа — ADR-0046.
- **US #696 «дизайн-система», эпик #540** — `.serena/memories/internal_profile_tacticum_internal_dev.md:41`:
  «Taiga: US #694 (эпик #554), issue #695 (пофикшен), **US #696 (дизайн-система, эпик #540)**».
  Строка 13 того же файла привязывает её к скиллу `tacticum-design-tokens` (добавлен в
  `tacticum-internal-dev` 0.2.0) — то есть **к PR #106**.
- План того самого PR #106 прямо предписывал завести US:
  `docs/superpowers/plans/2026-07-19-tacticum-web-design-system.md:30` — «**Step 1:**
  `mcp__taiga__list_epics(project_id=12)` — найти подходящий эпик … проверить, нет ли уже US про
  дизайн-систему tacticum-web (дубли запрещены)»; строки 385 и 797 — «Комментарий в Taiga US».
- Открытая дыра, признанная самим ADR: `docs/adr/0028-tacticum-design-two-phase-rollout.md:99` —
  «**Phase 2 epic не специфицирован — нет US в Taiga для design_web.**»

**Слой B — конвейер Figma→код (июль 2026, code-bindings и скиллы): НЕ СВЯЗАН, ноль ссылок.**

Ни один из **9 коммитов** в `design-systems/` не содержит ссылки на задачу Taiga — ни в
заголовке, ни в теле (проверил тела все девять). Единственное вхождение слова «Taiga» в этих
коммитах — у `82dfe0b`, и это **не ссылка на задачу**, а перечисление добавленных файлов:
«Add a small memory note (`.serena/memories/feedback_close_taiga_immediately_on_done.md`) about
closing Taiga items on completion». Легко принять за ссылку — не является ею.

Контраст надёжнее всего виден **в одном файле, у одного автора, с разницей в дни** —
`templates/iva-web-brownfield/CHANGELOG.md`:

| Версия | Дата | Чем помечена работа |
|---|---|---|
| 0.1.3 | 2026-07-15 | «installation_id discipline — context.yaml-first (**Taiga #657**)», «serena MCP launcher metadata (**Taiga #656**)» |
| 0.1.4 | 2026-07-15 | «codex config template ships tacticum-mcp active (**Taiga #670**)» |
| 0.1.6 | 2026-07-22 | «(**Taiga #674** wording)» |
| 0.1.9 | 2026-07-22 | «(incidents 2026-07-16 / 2026-07-20, **Taiga #697**)» |
| 0.2.0 | 2026-07-23 | «Bug-fix lane (**US #725**)», «the **US #724** top-level fix» |
| **0.3.0** | **2026-07-24** | «Design-system component skills (**ТЗ#1 figma-ds, gaps G1 / G4 / G6**)» — Taiga нет |
| **0.4.0** | **2026-07-24** | «`ui-mockup-match` Figma numeric-compare mode (**ТЗ#1 figma-ds, Scenario 2 step 7, gap G5 / PR-B**)» — Taiga нет |
| **0.5.0** | **2026-07-24** | «`iva-core-design-system` skill (**ТЗ#1 figma-ds, PR-C / axis-1**)», «`iva-web-figma-mapping-quickstart` user manual (**axis-1 / G8**)», «(**G7**)» — Taiga нет |

Та же картина в заголовках коммитов: не-ДС-работа Solonko несёт `US #725`, `US #720`, `US #672`,
`US #671`, `Taiga #697`, `Taiga #698`, `Taiga #726`, `Taiga #628/#629`; ДС-работа июля —
`ТЗ#1 Сц.4`, `Сц.2 шаг 7, gap G5`, `unblocks Сц.2 step 7 acceptance`.

**Вывод фактом:** ДС-работа **платформенного слоя** заведена в Taiga (эпики #322 и #540, US
#323-#330, #696). ДС-работа **конвейерного слоя июля 2026** — PR #122/#132/#134 и наши #144/#149/#154/#159 —
в трекере по коммитам и файлам репозитория **не отслеживается вовсе**; её система координат —
ТЗ Солонко (Сц.1-4, gaps G1-G8) и доска `00-Board/`. При том что `CLAUDE.md:36` требует, чтобы
всё жило в Taiga. Это расхождение нормы и практики, а не пробел разведки.

**Ограничение:** утверждение «в Taiga этих задач нет» я **не делал и сделать не могу** — я говорю
только, что **в репозитории на них нет ни одной ссылки**. US могли существовать и не быть
процитированы. Различить это может только лид со стороны трекера.

### 3.2. Прочая механика ссылок (чтобы не спутать номера)

В заголовках коммитов трейлинг-`(#N)` означает **разные вещи в разные периоды** — это ловушка
при разборе: у свежих коммитов (`(#122)`, `(#127)`, `(#133)`) это номер **GitHub PR**, у более
ранних (`(#655)`, `(#618)`, `(#475)`, `(#443)`, `(#439)`) — номер **Taiga**, поскольку номера PR
на те даты были двузначными. Встречается и оба сразу: `6a11925` — «… (**Taiga #697**) (**#127**)»
и `2d82590` — «… (#619) (#37)». Прежде чем сопоставлять номера, надо смотреть дату.

---

## 4. Про вики — то немногое, что видно из репозитория

Прямой проверки вики не было (см. шапку). Из репозитория механически следует вот что.

Скрипт `scripts/wiki_sync_manuals.py` — единственная автоматика публикации доков из репо в вики:

- **строка 164:** `for manual in sorted(MANUALS_DIR.glob("*-profile-quickstart.md")):`
- константы (строки ~40-45): `BASE = "https://wiki.iva.ru/rest/api"`, `PARENT_ID = "208703447"`,
  `SPACE = "IVAPROJECT"`; docstring — «Sync profile quickstart manuals to the **IVA wiki (Confluence DC)**.
  Parent page 208703447 «Набор профилей» (space IVAPROJECT)».

Отсюда два следствия:

1. **Оба figma-документа под маску не подпадают.** `iva-web-figma-mapping-quickstart.md` и
   `iva-kmp-figma-mapping-quickstart.md` названы `*-figma-mapping-quickstart.md`, а маска —
   `*-profile-quickstart.md`. Из 25 файлов в `docs/user_manuals/` маску не проходят ровно четыре:
   эти два плюс `iva-role-go-lead-guide.md` и `role-migration-runbook.md`.
2. **Цель синка — Confluence заказчика `wiki.iva.ru`, а не Wiki.js `wiki.cifragen.ru`**, который
   обслуживает `wiki-mcp` (тенанты `tacticum` / `cifragen`). Это разные системы. Значит в тенант
   `tacticum` нашей вики автоматикой не попадают вообще никакие quickstart'ы, не только figma-.
   Отдельно: `.serena/memories/internal_profile_tacticum_internal_dev.md:31` — «Wiki-sync
   (`wiki_sync_manuals.py`) для internal-профилей **НЕ выполнять**».

**Что это НЕ доказывает:** страницу про ДС могли завести в вики руками. Проверка — за лидом.

---

## 5. Что искал и не нашёл

- **Ссылок на Taiga в ДС-коммитах** — `git log --all -i --grep`, плюс ручной разбор тел всех 9
  коммитов `design-systems/`: **ноль**. Единственное вхождение слова — имя файла serena-памяти
  в `82dfe0b` (не ссылка).
- **ДС-коммитов от третьих авторов** — нет: только Solonko (5) и Diaret (4).
- **`design-systems/iva-core/` на main** — отсутствует; живёт в невмерженной
  `origin/feat/iva-core-design-system`.
- **`figma_key` у `iva-mobile`** — 0 из 34 (проверено разбором `tokens.json`, не по словам коммита).
- **ТЗ-исходники** `scratchpad/ds-scan/figma-ds-{process-tz,scenario-1..4}.md` и
  `figma-ds-review-report.md` — на них ссылаются `00-Board/map-existing-vs-gap-sc12.md:9`,
  `map-sc3-and-remainder.md:9,84` и тело коммита `2f42190`. `find` по `~/tacticum-vault` и
  `~/tacticum` — **пусто**. Первоисточник постановки ТЗ#1 локально не воспроизводится; это
  отдельный риск, а не мелочь: вся таксономия Сц.1-4 / G1-G8, которой размечены CHANGELOG и
  коммиты, держится на утраченном документе.

## Связано
- [[map-existing-vs-gap-sc12]] · [[map-sc3-and-remainder]] · [[explore-ds-repo-2026-07-27]]
- [[wiki-mcp-usage]]