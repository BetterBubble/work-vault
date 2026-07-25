---
title: impl-axis2-kmp
type: note
permalink: tacticum/00-board/impl-axis2-kmp
tags:
- draft
- board
- lead-modes
- axis2
- impl
- kmp
- tz1
---

# impl-axis2-kmp — реализация оси-2 (A/B/C) в iva-kmp-brownfield

**От:** implementer. **Кому:** lead-modes. **Ветка:** `feat/kmp-multirepo-axis2` @ worktree `/Users/bubblemac/tacticum-worktrees/kmp-axis2`. **Коммит:** `ac30efc` (локальный, НЕ запушен — autonomy off).
**Основание:** карта `explore-axis2-gapmap` + спека `spec-axis2-workflow-for-lead-modes`. R1=kmp-only (ГД). Принцип президента: строго по ТЗ.

## Скоуп — что тронуто (6 файлов, ВСЕ в TARGET-пакете)

Все правки — в `templates/iva-kmp-brownfield/`, brownfield-специфичные (вне `_mirrors.yaml`). `iva-web-brownfield`, зеркалируемые coder/tester/test-runner, owner-base — НЕ тронуты. Mirror-sync подтвердил: 62 ингредиента/6 пар синхронны (мои файлы вне охвата).

## Требование A — опц. `$3` source-репо в start-task
Файл: `ingredients/commands/start-task.md`
- `argument-hint` → `<...> [target-dir] [source-repo]` (добавлен `[source-repo]`).
- Новый блок `$3` в Arguments: «Source/reference repository (optional)… READ-ONLY reference (e.g. Angular iva-one)… **Omit for the normal single-tree brownfield task — behaviour is unchanged.**»
- Hand-off: добавлена строка `> **Source/reference repo (read-only):** ${3:-none — single-tree task}` + прозный абзац «If a source/reference repo is given above (not "none"), treat it as a READ-ONLY reference… keep ALL writes and the design-system-of-record in the target repo (cwd). Omitted → normal single-tree flow.»
- **Аддитивность/условность:** использована ТОЛЬКО проверенная `:-`-форма (как `${1:-…}`/`${2:-…}` в этом же файле) — БЕЗ `:+` и прочей неопробованной раскрутки. Без `$3` строка рендерит «none — single-tree task», позиционная обработка `$1/$2` не тронута → splat-регресса нет (у kmp его и не было). copilot команду не поддерживает (`supports:[claude-code,codex]`) — для него A покрыт через тело агента (B).

## Требование B — read-only source + write target (во всех 3 CLI-телах)
Файлы: `ingredients/agents/tacticum-workflow.md` (claude), `ingredients/agents-copilot/tacticum-workflow.md`, `ingredients/agents-codex/tacticum-workflow.toml`
- **claude/copilot** — в Inputs добавлен пункт 3 «Source/reference repository (optional)… Present ONLY for cross-repo port tasks; absent for the normal single-tree brownfield task» + блок **«Cross-repo mode (source repo bound) — access model (additive; skip entirely when no source repo is given — the normal single-tree flow is unchanged)»**: source = READ-ONLY reference; ALL writes + «ДС письма» = current repo (cwd); source привязан к СВОЕМУ `.tacticum/context.yaml`/`installation_id` + отдельный `kb_discover`/`kb_run_id`; **контракт поведения живёт в скилле `web-to-kmp-source-reference` — тело даёт только механику доступа** (контракт НЕ дублирован).
- В Run Discovery добавлен абзац **«Source-tree discovery (only when a source/reference repo is bound — otherwise skip entirely; the single-tree flow in steps 1-4 above is unchanged)»**: отдельно читать SOURCE `.tacticum/context.yaml`→`installation_id`, отдельный `kb_discover`→ВТОРОЙ `kb_run_id` только на чтение; два run различны; при отсутствии source-KB → read-only file reads, не блокирует target-run.
- **codex .toml** — конденсированно, тот же смысл одним абзацем «Cross-repo source mode (only when `/start-task … [source-repo]` bound…)» в Run Discovery (не копипаста claude — сжато в стиле .toml).

## Требование C — гейт §3.0 реальной проверки двух деревьев (во всех 3 телах)
Те же 3 файла, секция §3.0 Environment Readiness — новый пункт **7** ПЕРЕД «Gate decision»:
- claude/copilot: «**Cross-repo trees** — *only when a source/reference repo is bound; skip entirely for single-tree tasks (checks 1-6 above are the whole gate then)*… verify BOTH trees: (a) source exists/readable + holds referenced original files (spot-check PIN-cited path); (b) target (cwd) writable per checks 1-4. Missing/unreadable source = **blocker**, not a silent skip. Source stays **read-only**.»
- codex: тот же пункт 7 конденсированно.
- **Условность:** явно «skip entirely for single-tree tasks (checks 1-6 are the whole gate then)» → обычный одно-древесный гейт не меняется (R5 прод-safe соблюдён).

## §4a дисциплина
- `manifest.yaml`: `version` 0.4.5 → **0.4.6** (patch-bump).
- `CHANGELOG.md`: секция `## [0.4.6] — 2026-07-24` в стиле файла (Added), описывает A/B/C и аддитивность.

## Проверки
- **Дисциплина-чек** `check_profile_version_discipline.py --diff-against origin/main`: **OK — 46 profile(s) clean** (0 violations).
- **pytest** `tests/catalog/test_manifest_schemas.py`: зелёные (38 passed). Запуск: `uv run --extra dev python -m pytest` (без `--extra dev` pytest не в venv; `uv run pytest` подхватывал системный py3.12 → чинится флагом).
- **Доп. parity/preset** `test_role_replacement_parity.py` + `test_iva_role_presets.py`: зелёные (173 passed).
- **mirror-sync** `check_mirror_sync.py`: OK — 62 ингредиента/6 пар синхронны.
- **verify-*.ps1** (структурные тесты пакета): **НЕ гонял — powershell/pwsh недоступны в среде** (`command -v` пусто). Per gapmap они проверяют инвентарь команд (`start-task` в списке), не контент аргументов/§3.0; инвентарь команд НЕ менялся → регресса по ним не ожидается, но фактически не прогонял.

## diff --stat
```
CHANGELOG.md                          | 24 +
agents-codex/tacticum-workflow.toml   | 18 +
agents-copilot/tacticum-workflow.md   | 36 +
agents/tacticum-workflow.md           | 37 +
commands/start-task.md                | 14 +
manifest.yaml                         |  2 +-
6 files changed, 129 insertions(+), 2 deletions(-)
```

## Риски / открытые вопросы
1. **`${3:-…}` рендер в hand-off** — опираюсь на доказанную `:-`-форму (уже используется в файле), НЕ на `:+`. Если lead хочет, чтобы при отсутствии `$3` строка вовсе исчезала (а не «none — single-tree task») — это потребовало бы `:+`, чью поддержку в раскрутке команд Claude/Codex я не проверял. Оставил безопасный вариант.
2. **verify-*.ps1 не прогнаны** (нет powershell). Если нужна приёмка через них — прогнать на Windows/с pwsh; при желании можно добавить content-ассерты на `[source-repo]`/пункт §3.0 (сейчас их нет, A/B/C не ломают и не проверяют структурные тесты).
3. **Симметрия web** — НЕ делал (сверх-ТЗ, R1=kmp-only). iva-web-brownfield не тронут.
4. **Контракт read-only-источника** — не дублирован; ссылаюсь на скилл `web-to-kmp-source-reference` (в iva-kmp-development-base, вне репы). Стык только на уровне названия скилла.
5. **copilot A** — команда copilot не поддерживает `start-task`; source для copilot доступен через тело агента (B). Если lead хочет копилот-команду — это отдельный объём.

Готово к ревью. Не пушу — жду решения lead/пользователя.
