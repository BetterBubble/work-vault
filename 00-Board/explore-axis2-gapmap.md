---
title: explore-axis2-gapmap
type: note
permalink: tacticum/00-board/explore-axis2-gapmap
status: draft
tags:
- draft
- board
- lead-modes
- explore
- axis2
- tz1
- mirror
---

# explore-axis2-gapmap — ось-2 A/B/C на ЖИВОЙ mirror-паре (brownfield)

**От:** explorer (read-only). **Кому:** lead-modes. **Репо:** `tacticum-dev` @ main (не правил).
**Спека:** `tacticum/00-board/spec-axis2-workflow-for-lead-modes`. **Пред. разведка:** `explore-axis2-brownfield` (там цель — депрекированный `brownfield-task-workflow`; ГД сменил цель на живую пару).
**Пакеты-цель:** `templates/iva-web-brownfield/` (source-образец, читаем) + `templates/iva-kmp-brownfield/` (target, пишем). Оба `deprecated: false` — ЖИВЫЕ, не frozen.

---

## 0. ГЛАВНЫЙ ВЫВОД про место правок (mirror-владение)

`templates/_mirrors.yaml` содержит ДВЕ отдельные пары:
- **owner `iva-web-development-base` ↔ mirror `iva-web-brownfield`** (`_mirrors.yaml:38-55`), ингредиенты: `surgical-change-discipline, angular-build-verification, angular-module-integration, angular-run-launch, angular-ui-testing, coder, ivcs-libs-contract, local-skill-authoring, ngrx-signals-state, nx-workspace-discipline, openapi-codegen-pipeline, test-runner, tester, web-local-knowledge-routing, webrtc-conference-fragile-zone`.
- **owner `iva-kmp-development-base` ↔ mirror `iva-kmp-brownfield`** (`_mirrors.yaml:22-37`), ингредиенты: `calls-voip-fragile-zone, codegraph-first-navigation, coder, compose-multiplatform-ui, kmp-architecture-guards, kmp-build-verification, kmp-local-knowledge-routing, kmp-module-integration, kmp-run-launch, kmp-ui-testing, proguard-r8-discipline, test-runner, tester`.

**Ключ:** файлы, которые правят A/B/C — `start-task` (command) и `tacticum-workflow` (agent, 3 CLI-тела) — **НЕ входят ни в один mirror-список**. Подтверждено физически: у owner'ов `*-development-base` этих файлов НЕТ (`ingredients/commands/` пуст; `ingredients/agents/` = только `coder/test-runner/tester`; манифесты owner'ов не декларируют `start-task`/`tacticum-workflow`). Декларации есть ТОЛЬКО в `*-brownfield` манифестах (web `manifest.yaml:381,446`; kmp `manifest.yaml:354,419`).

➡️ **Все правки A/B/C — brownfield-СПЕЦИФИЧНЫЕ: правим ПРЯМО в `*-brownfield`.** Нет owner-копии, нет байт-в-байт зеркала, нет триггера mirror-sync CI. `scripts/check_mirror_sync.py:83` итерирует только `pair["ingredients"]` — start-task/tacticum-workflow вне охвата. `coder/tester/test-runner` зеркалируются, НО A/B/C их не трогают → mirror-риск нулевой при условии, что правки не заедут в эти три агента.

**Пара «web+kmp» из ТЗ — это RUNTIME source→target пара (Сц.4: Angular iva-one → KMP), НЕ `_mirrors.yaml`-пара.** Друг с другом эти пакеты байт-в-байт не связаны.

---

## 1. Требование A — `start-task`: аргумент source-репо

**Существует (as-is):** ни в одном пакете source/reference-аргумента НЕТ.
- **web** `iva-web-brownfield/ingredients/commands/start-task.md`:
  - `:3` `argument-hint: <task-description> [target-dir]`; `:11` `$1`=ТЗ, `:12` `$2`=Target directory.
  - Тело hand-off: `:45` `**ТЗ:** $ARGUMENTS` (splat всех арг!), `:46` `**Target directory:** ${2:-Tasks/}`.
- **kmp** `iva-kmp-brownfield/ingredients/commands/start-task.md`:
  - `:3` `argument-hint: <task-description | Jira-key | Confluence-URL> [target-dir]`; `:11-15` `$1`=task source (Jira/Confluence/ТЗ), `$2`=Target dir.
  - Тело: `:56` `**ТЗ:** ${1:-<full ТЗ text>}…` (явный позиционный, НЕ splat), `:57` `${2:-Tasks/}`.
- Оба: `supports: [claude-code, codex]` (web `manifest.yaml:449`, kmp `:422`) — **copilot НЕ поддержан**; одно `body_path` без codex-варианта (один текст на 2 CLI).

**Gap:** чистый GAP в обоих. Добавить опц. `$3` (source-репо) + `argument-hint … [source-repo]` + условная строка в hand-off-блок.
**Нюанс аддитивности:** web `:45` `$ARGUMENTS` splat-ит ВСЕ арги в тело ТЗ → наивный `$3` протечёт путь в текст ТЗ (нужна явная позиционная обработка, как в kmp `:56`). kmp этой проблемы лишён.
**copilot:** арг доедет только до claude+codex; для copilot A реализуется в Inputs агента `tacticum-workflow` (см. B).

## 2. Требование B — read-only source + write target (модель «одно дерево»)

Привязка к ОДНОМУ дереву — через `.tacticum/context.yaml` (root cwd) → `installation_id` → `kb_discover` once. Присутствует во всех 6 телах:
- **web claude** `iva-web-brownfield/ingredients/agents/tacticum-workflow.md`: `:28-32` явная фраза **«The working directory is ONE of the two repos»** (но «two repos» = iva-one/iva-connect, ОДНА платформа Angular, НЕ cross-framework source/target); Inputs `:34-38` (ТЗ + Target dir); Run Discovery `:44` `.tacticum/context.yaml`→installation_id, `:47` kb_discover once.
  - **copilot** `…/agents-copilot/tacticum-workflow.md`: `:29-33` та же фраза, Inputs `:35`, `.tacticum` `:45` — почти дословно.
  - **codex** `…/agents-codex/tacticum-workflow.toml`: `:14-15` detect repo, `:22-27` run discovery (единый context.yaml) — конденсация.
- **kmp claude** `iva-kmp-brownfield/ingredients/agents/tacticum-workflow.md`: **фразы «одно дерево» НЕТ** (пакет одно-целевой); Inputs `:29-33` (ТЗ/FR + Target dir); Run Discovery `:52` `.tacticum/context.yaml` root single.
  - **copilot** `…:62`; **codex** `…:22-25`.

**Gap:** чистый GAP — понятия ИСХОДНОГО дерева со своим `.tacticum/context.yaml`/installation/`kb_discover` нет нигде; всё завязано на один cwd. Ослабить до: source читаем read-only (свой context.yaml/kb_discover), write + «ДС письма» = target(cwd). Существующая web-фраза «ONE of the two repos» — про same-framework, её ослабление НЕ покрывает cross-framework (нужен отдельный абзац, а не переписывание существующего).

## 3. Требование C — гейт §3.0 (реальная проверка двух деревьев)

- **web ИМЕЕТ §3.0 + cross-repo пункт (образец, но слабый):**
  - claude `iva-web-brownfield/ingredients/agents/tacticum-workflow.md`: `:216` `### 3.0 Environment Readiness Check (GATE)`, **item 5 `:231-233`**: «confirm the other repo's clone/publishing path is available; otherwise record … caveats requiring a separate session».
  - copilot `…/agents-copilot/…`: item 5 `:232-234` (зеркало claude, offset +1).
  - codex `…/agents-codex/…toml:71`: «cross-repo prerequisites available».
  - ⚠️ это same-framework (iva-one↔iva-connect, `@ivcs/*`) и КОНВЕНЦИЯ «caveat/отдельная сессия», НЕ реальная валидация двух деревьев.
- **kmp §3.0 ЕСТЬ, но cross-repo пункта НЕТ ни в одном из 3 тел:**
  - claude `iva-kmp-brownfield/ingredients/agents/tacticum-workflow.md:268-320` (items 1-6: submodules/JDK/compile/files/Android-iOS/codegraph).
  - copilot `…:331-361` (те же 6).
  - codex `…toml:176-197` (те же 6).

**Gap:** 
- **kmp (target Сц.4) — чистый GAP:** добавить в §3.0 (перед Gate decision, kmp claude `:297`) новый пункт: при привязанном source-репо проверить доступность/валидность ОБОИХ деревьев (source read-only + target write) до запуска воркеров. Во все 3 тела (codex/copilot — в их стиле).
- **web — PARTIAL:** item 5 существует как образец; по ТЗ его надо усилить «клон доступен/иначе сессия» → реальная проверка двух деревьев при привязанном source. Правка in-place существующего item 5 (не новый пункт).

## 4. ТАБЛИЦА: требование → место правки → existing → что добавить

| Треб. | Место правки (brownfield-специфично, НЕ mirror) | CLI-тела | Existing (as-is) | Что добавить (gap, per ТЗ) |
|---|---|---|---|---|
| **A** source-арг | `iva-kmp-brownfield/…/commands/start-task.md` (target, где стартует задача); при необходимости симметрии — `iva-web-brownfield/…/commands/start-task.md` | 1 body (claude+codex); copilot нет | НЕТ source-арг. kmp `:3,11-15,56`; web `:3,11-12,45` | опц. `$3` source-repo + `argument-hint … [source-repo]` + hand-off-строка. copilot — через Inputs агента |
| **B** read-only src + write tgt | `iva-kmp-brownfield/…/agents/{tacticum-workflow.md, agents-copilot/…md, agents-codex/…toml}` | 3 тела | одно-дерево через `.tacticum/context.yaml`(cwd). kmp claude Inputs `:29-33`, RunDisc `:52`. web `:28-32,44` | понятие source-дерева (свой context.yaml/kb_discover, read-only); write+ДС=target |
| **C** гейт §3.0 два дерева | kmp: 3 тела §3.0 (**GAP**, claude `:268-320`, copilot `:331-361`, codex `:176-197`). web: усилить item 5 in-place (**PARTIAL**, claude `:231-233`, copilot `:232-234`, codex `:71`) | 3 тела ×2 пакета | web item5 = «caveat/отд.сессия» (образец). kmp — нет cross-repo пункта | kmp: новый §3.0-пункт «2 дерева при source». web: усилить caveat→реальная проверка |

## 5. Fidelity: existing vs gap (вердикты)

- **A:** GAP (оба пакета, оба CLI). Частично — kmp уже имеет чистую позиционную обработку `${1:-…}` (`:56`), web нет (splat, риск R-A).
- **B:** GAP. «Одно дерево» — имплицит через context.yaml во всех 6 телах; web имеет фразу «ONE of the two repos» (`:28`), но она про same-framework — cross-framework source не покрыт → добавлять, не переписывать.
- **C:** web = **PARTIAL** (образец item5 есть, усилить in-place); kmp = **GAP** (cross-repo пункта нет). Реальный target Сц.4 = kmp → основной объём C в kmp.

## 6. Fidelity 3-CLI (для консистентности B/C)

- web: claude↔copilot почти байт-в-байт (§3.0 offset +1, идентичный текст). codex.toml — сознательная конденсация (не mirror).
- kmp: claude↔copilot близки по смыслу, НО расходятся по объёму (claude 461 стр. vs copilot 552 стр. — у copilot доп. материал; codex 309). codex — конденсация.
- ➡️ B/C класть во ВСЕ 3 тела каждого пакета, codex/copilot — в их стиле, не копипастой.

## 7. Версии / тесты

- Версии (для bump при правке brownfield): `iva-web-brownfield` **0.2.1**, `iva-kmp-brownfield` **0.4.5** (оба `deprecated:false`). Owner'ы `iva-web/kmp-development-base` = 0.1.0 — **A/B/C их не трогают** (файлов там нет) → их версии/CHANGELOG не меняются.
- CHANGELOG есть во всех 4 (`iva-*-brownfield/CHANGELOG.md`: web 203, kmp 451 стр.). Правки → bump версии + запись в CHANGELOG затронутого brownfield-пакета.
- Тесты: **repo-wide `tests/` пары не упоминают** (grep пусто). Локальные `iva-*-brownfield/tests/verify-{claude-code,codex,copilot}.ps1` — структурные (`verify-claude-code.ps1:69 $expectedTrialCommands = @('start-task')`), **content-ассертов на аргументы/§3.0 нет** → A/B/C их не сломают, но и не проверят (нужны новые ассерты, если хотим приёмку в тестах).
- mirror-sync CI (`profile-version-discipline.yml` → `scripts/check_mirror_sync.py`) правками A/B/C НЕ триггерится (start-task/tacticum-workflow вне `_mirrors.yaml`).

## 8. Риски

- **R1 (место C — решение лида):** Сц.4 однонаправлен (web→kmp). Основной write = **iva-kmp-brownfield**; iva-web-brownfield = read-source-образец. Нужна ли правка A/B и усиление C ТАКЖЕ в iva-web-brownfield (чтобы web мог быть target с привязкой source) — по ТЗ Сц.4 НЕ требуется. Минимум-per-ТЗ = править kmp; web трогать только для усиления образца C (item5) если лид хочет симметрию. Уточнить у лида объём по web.
- **R2 (аддитивность A / web splat):** web `start-task.md:45 $ARGUMENTS` splat-ит все арги в ТЗ — `$3` протечёт; нужна явная позиционная обработка (kmp уже ок). Одно-древесные задачи без `$3` не должны регрессировать (приёмка A).
- **R3 (3-CLI синк):** claude+copilot почти дословны, codex.toml — конденсация; kmp copilot объёмнее claude. B/C во все три, codex/copilot в их стиле.
- **R4 (copilot без команды):** `start-task supports:[claude-code,codex]` — для copilot A только через Inputs агента, не через команду.
- **R5 (горячий iva-web-brownfield 0.2.1 / iva-kmp-brownfield 0.4.5):** оба живые, активно используются; правки B/C в центральный §3.0/Run Discovery workflow-агента затрагивают ВСЕ задачи пакета, не только ось-2 — держать изменения строго аддитивными (условными «при привязанном source»), чтобы одно-древесный обычный brownfield не регрессировал.
- **R6 (reference-скилл lead-ds):** `web-to-kmp-source-reference` в репе ОТСУТСТВУЕТ (зона lead-ds, `iva-kmp-development-base`). Жёсткий контракт read-only-источника живёт там; workflow (A/B/C) даёт только механику доступа. Файлового пересечения нет.