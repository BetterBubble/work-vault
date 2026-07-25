---
title: explore-axis2-brownfield
type: note
permalink: tacticum/00-board/explore-axis2-brownfield
tags:
- draft
- board
- lead-modes
- explore
- axis2
- tz1
---

# explore-axis2-brownfield — разведка под ось-2 (A/B/C)

**От:** explorer (read-only). **Кому:** lead-modes. **Репо:** `tacticum-dev` @ main. **Спека:** `tacticum/00-board/spec-axis2-workflow-for-lead-modes`.
**Пакет-цель:** `templates/brownfield-task-workflow/`.

> ⚠️ ГЛАВНЫЙ РИСК ВПЕРЁД (см. п.6-R1/R2): пакет-цель `brownfield-task-workflow` **`deprecated: true`**, superseded by `iva-brownfield-mail`, ни от кого не зависит и **не в `_mirrors.yaml`**. При этом реальный Сц.4 (Angular→KMP) механически ложится на пары `iva-web-brownfield`(источник) + `iva-kmp-brownfield`(цель), у которых §3.0-гейт УЖЕ есть. Требование C в текущем виде в `brownfield-task-workflow` **не имеет "as-is" (гейта §3.0 тут нет вообще)** — оно живёт в `iva-web-brownfield`. Перед планом стоит подтвердить, что цель — именно этот депрекированный пакет.

---

## 1. `start-task` — аргументы команды (Требование A)

Файл: `templates/brownfield-task-workflow/ingredients/commands/start-task.md` (весь, 22 стр.)
- Стр.3: `argument-hint: <task-description> [target-dir]`
- Стр.8-10: `$1` — ТЗ (required, multi-line); `$2` — **Target directory** (optional, default `Tasks/<next-number>-<slug>/`). Т.е. «2-й арг = папка доков» = именно `target-dir` (куда пишутся brd/adr/pin/…), НЕ source-репо.
- Стр.12-21: hand-off-блок в саб-агента. Стр.16 `**ТЗ:** $ARGUMENTS`; стр.17 `**Target directory:** ${2:-Tasks/}`.
- «Рабочее дерево» команда понимает **имплицитно = cwd** (репо, где запущен `/start-task`); `target-dir` — подпапка внутри cwd. Понятия второго дерева нет.

**Куда встроить A:** новый опциональный `$3` (source/reference-репо) + правка `argument-hint: <task-description> [target-dir] [source-repo]` + строка в hand-off-блоке (условно: «если задан source-репо — привязать исходное дерево read-only»).
**Подводный камень (аддитивность):** стр.16 `$ARGUMENTS` splat-ит ВСЕ аргументы в текст ТЗ — наивное добавление `$3` протечёт путь репо в тело ТЗ. Нужна явная позиционная обработка, чтобы одно-древесные задачи (без `$3`) не регрессировали.
**Охват CLI:** в manifest `start-task` `supports: [claude-code, codex]` (стр.191) — **copilot НЕ поддержан**. Тело команды одно (`body_path`, без codex-варианта). Значит арг A доедет до 2 из 3 CLI; copilot-юзеры стартуют через агента напрямую.

## 2. Модель «одно дерево» (Требование B)

**Явной фразы «рабочая директория = одно дерево» нет.** Привязка к одному дереву — имплицитная, через два якоря:
- **`.tacticum/context.yaml`** → источник `installation_id`:
  - `ingredients/commands/approve-docs.md:24` — "…and `installation_id` from `.tacticum/context.yaml`."
  - `ingredients/commands/run-implementation.md:24` — то же.
- **`kb_discover()`** ищет прогон в `.tacticum/` (cwd):
  - `agents/tacticum-workflow.md:25` / `agents-copilot/…:26` / `agents-codex/….toml:14` — "Call `kb_discover()` → receive `kb_run_id`".
  - description-строки всех трёх: "finds latest Tacticum run in .tacticum/" (agents:5, copilot:5, codex:2).
- `installation_id` сейчас **единственный**, берётся из cwd-`.tacticum/context.yaml` и прокидывается всем воркерам (agents:93-94, copilot:91-92, codex:63-64: "Subagents must NOT call `kb_discover`/`whoami`").

**Что ослабить до read-only-source + write-target:** ввести понятие ИСХОДНОГО дерева со СВОИМ `.tacticum/context.yaml`/installation/`kb_discover`, при этом write и «ДС письма» = ЦЕЛЕВОЙ (cwd) репо. Сейчас всё завязано на один cwd.

**Расхождение 3 CLI-вариантов (для консистентности B/C):**
- `agents/tacticum-workflow.md` (142 стр.) и `agents-copilot/tacticum-workflow.md` (140 стр.) — **почти дословный синк** (Inputs стр.15-20 идентичны, Phase 3 HARD GATE — тот же текст).
- `agents-codex/tacticum-workflow.toml` (96 стр.) — **сознательная КОНДЕНСАЦИЯ**, не байт-в-байт: нет секции `## Inputs`, урезаны MOCKUPS/KB-детали, та же суть короче. НЕ строгий mirror.
- Вывод: B/C править во ВСЕ ТРИ, но codex — в его сжатом стиле, не копипастой.

## 3. Гейт Phase 3.0 (Требование C)

**В `brownfield-task-workflow` §3.0 (Environment Readiness) ОТСУТСТВУЕТ.** Phase 3 (`agents/tacticum-workflow.md:82-101`) сразу идёт в runtime HARD GATE (проверка глубины саб-агента, стр.84-90) → пасс-тру контекста (92-95) → запуск coder/tester (97-101). Никакой проверки «клон другого репо доступен / иначе отдельная сессия» тут нет.

**Формулировка из спеки («клон доступен, иначе отдельная сессия») реально живёт в ДРУГОМ пакете:**
- `templates/iva-web-brownfield/ingredients/agents/tacticum-workflow.md:216` — `### 3.0 Environment Readiness Check (GATE)`; **item 5 (стр.231-233):** "**Cross-repo prerequisites** — if the PIN has iva-connect stages while working in iva-one (or vice versa), confirm the other repo's clone/publishing path is available; otherwise record the cross-repo stages as caveats requiring a separate session." (+ copilot-зеркало `…copilot/…:233`).
- `templates/iva-kmp-brownfield/ingredients/agents/tacticum-workflow.md:268-320` — тоже богатый §3.0, но item 5 (стр.285-288) про Android/iOS-prereqs, **cross-repo пункта нет**.

**Куда встроить реальную проверку двух деревьев:** добавить в `brownfield-task-workflow` секцию `### 3.0 …` ДО runtime HARD GATE (перед `agents/tacticum-workflow.md:84`, т.е. между «After user approves» стр.97 и запуском воркеров) — с пунктом: при привязанном source-репо проверить доступность/валидность ОБОИХ деревьев (source read-only + target write) прежде чем пускать в работу. Модель — item 5 из `iva-web-brownfield` §3.0, но усиленный до реальной проверки (а не «caveat»).

## 4. Состав/схема пакета

`templates/brownfield-task-workflow/manifest.yaml`:
- `schema_version: "2"`, `profile_id: brownfield-task-workflow`, **`version: "0.3.10"`**.
- **`deprecated: true` (стр.12)**, комментарий стр.9: "Superseded by iva-brownfield-mail" (US #369: следующий CI-seed ставит `is_active=False`, пропадает из публичного MCP-surface).
- `ide_targets`: claude-code/codex/copilot = `full` (стр.34-37).
- Ингредиенты: 6 skill_spec, 4 agent_spec (tacticum-workflow/coder/tester/test-runner), 3 command_spec (start-task/approve-docs/run-implementation — все `supports: [claude-code, codex]`, copilot нет), 3 mcp_server_spec, 3 instruction_pack, 1 repo_config, 1 copilot_config.
- agent_spec `tacticum-workflow` (стр.123-137): `body_path` (claude) + `copilot_body_path` + `codex_body_path` — три тела.

**Depends_on / роли:** grep по всем `templates/*/manifest.yaml` — **ни один пакет не зависит от `brownfield-task-workflow`** (совпадения только self-marker `marker_id`). Пакета **нет в `_mirrors.yaml`** (комментарий стр.9-10: депрекированные зеркал не имеют, frozen). Сирота-standalone.

**Тесты:** repo-wide `tests/` пакет **не упоминает** (grep пусто). Локальные `templates/brownfield-task-workflow/tests/verify-{claude-code,codex,copilot}.ps1` — **структурные** (наличие ингредиентов, marker в CLAUDE.md), `$expectedTrialCommands = @('start-task')` (verify-claude-code.ps1:68). **Content-ассертов на аргументы/§3.0 нет** → правки A/B/C тесты не сломают, но и не проверят (нужны новые ассерты, если хотим приёмку в тестах).

## 5. Стык с reference-скиллом `web-to-kmp-source-reference` (п.3 lead-ds)

- Навык **`web-to-kmp-source-reference` в репе ОТСУТСТВУЕТ** (grep по всему `templates/` — 0 совпадений). Ещё не создан (это зона lead-ds, в `iva-kmp-development-base`).
- Модель-прототип `angular-legacy-web-context` — **существует** в `templates/iva-rn-brownfield/ingredients/skills/angular-legacy-web-context/SKILL.md` (+ manifest:310). Файлово с `brownfield-task-workflow` **не пересекается** — подтверждаю развязку A/B/C (механика) vs скилл (контракт).

## 6. РИСКИ и открытые вопросы для плана оси-2

- **R1 (блокер-вопрос):** цель — **депрекированный** пакет (`deprecated: true`, superseded by `iva-brownfield-mail`, orphaned, скоро `is_active=False`). Стоит ли вкладывать A/B/C сюда? Комментарий `_mirrors.yaml:9` прямо: «Депрекированные профили — frozen». Правка frozen-пакета противоречит конвенции.
- **R2 (несовпадение премисы C):** «текущий гейт клон-доступен/отдельная-сессия» в `brownfield-task-workflow` **не существует** — он в `iva-web-brownfield §3.0 item5`. Реальный Сц.4 (Angular→KMP) — это пары `iva-web-brownfield`(src) + `iva-kmp-brownfield`(tgt), у которых §3.0 уже есть. Вопрос лиду: не должна ли ось-2 жить в этих mirror-парах (owner: `iva-web-development-base`/`iva-kmp-development-base`), а не в депрекированном standalone?
- **R3 (3-CLI синк):** claude+copilot почти дословны, codex.toml — сжатый парафраз (не байт-в-байт). B/C класть во все три консистентно, codex — в его стиле.
- **R4 (аддитивность A):** `start-task.md:16 $ARGUMENTS` splat-ит все арги в ТЗ — `$3` протечёт в текст ТЗ; нужна явная позиционная обработка, чтобы одно-древесные задачи не регрессировали (приёмка A).
- **R5 (охват copilot):** `start-task` не поддержан для copilot (`supports:[claude-code,codex]`) — арг source-репо доедет только до 2 CLI; для copilot A реализуется на уровне агента.
- **R6 (версия/CHANGELOG):** SemVer (CHANGELOG.md), текущая 0.3.10 → A/B/C = minor bump (0.4.0) + запись. Но если R1 подтверждает freeze — bump сам по себе спорен.
