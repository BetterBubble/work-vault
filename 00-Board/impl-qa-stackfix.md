---
status: draft
role: implementer
task: Правка QA-автотест-лейна под корректный стек (KMP/playwright-canvas/one-web-kmp)
date: 2026-07-24
branch: fix/qa-stack-naming-kmp
worktree: ~/tacticum/tacticum-dev-qa-stackfix
permalink: tacticum/00-board/impl-qa-stackfix-1
---

# impl — QA-стекфикс (iva-qa-autotest-base под one-web-kmp / pytest-playwright-canvas)

## ⚠️ Отклонение от инструкции по базе ветки (важно, требует внимания тимлида)

Задача велела ветвиться от **main** (`git worktree add … main`) и бампить **0.2.0 → 0.3.0**.
Но на `main` профиль стоит на **0.1.0** и НЕ имеет codex-тел (`skills-codex/`, `agents-codex/`).
Версия 0.2.0 и «claude-/codex-тело fix-failed-test» из ТЗ существуют только на ветке
**`feat/qa-codex-rework`** (0.2.0, не влита в main). Всё ТЗ (bump 0.2.0→0.3.0, dual claude/codex
тела §4, KMP-обрамление) сходится именно на ней.

**Решение:** пересоздал worktree с базой `feat/qa-codex-rework` (иначе 0.2.0→0.3.0 и «codex-тело»
недостижимы). Ветка `fix/qa-stack-naming-kmp` стекается поверх `feat/qa-codex-rework`, а не main.
Скрипт дисциплины версий (`--diff-against origin/main`) при этом зелёный (0.3.0 > 0.1.0).
Если задумывался всё же main как база — вернуть на доработку.

## Ключевой факт, снявший неоднозначность

Проверил природу «one-web» в `craft-stack`: env по умолчанию `kmp-stage`, Grep по `*.kt`
(Kotlin), «canvas-класс», «пилот one-web», dev-server — то есть «one-web» в текстах профиля = это
и есть **канвас/KMP-пилот**, ошибочно названный «one-web». Его корректное имя — `one-web-kmp`.
Значит все `one-web` в профиле → `one-web-kmp` консистентны (не путать с Angular-репо one-web,
который в текстах профиля отдельно не фигурирует как самостоятельная сущность).
`«референс Selenium: …»` — это обобщённый контраст-стек канона (dual-stack teaching), НЕ репозиторий.

## Что и где сделано

### 1. Мёртвые `_selenium-ref` (§4) — убраны 4 ссылки
Целевые файлы `craft-stack/_selenium-ref/manual-walkthrough.md` и `.../cross-browser-verification.md`
в `craft-stack` отсутствуют (проверено `find`). Вырезаны фрагменты, ведущие в никуда, канвас-ветки сохранены:
- `ingredients/skills/fix-failed-test/SKILL.md` — Phase 3 (было ~L195) и Phase 5 (было ~L246).
- `ingredients/skills-codex/fix-failed-test/SKILL.md` — Phase 3 (было ~L221) и Phase 5 (было ~L314).

### 2. Репо-нейминг one-web → one-web-kmp (§1)
Механическая замена с negative-lookahead `one-web(?!-kmp)` по всему профилю, КРОМЕ README/CHANGELOG
(их правил семантически). Инфра-имена (`autocore`/venv/`tools.testops`/`glab`/CI) и env-переменная
`ONE_WEB_DIR` — не тронуты. Затронуты: manifest, 9 SKILL claude + 4 codex-тела, все `references/`,
все `craft-stack/*`, 2 субагента `agents/*.md` + 2 `agents-codex/*.toml`, `craft-answers.example.toml`.

### 3. Стек-нейминг pytest/Selenium → pytest/Playwright (§2)
- `manifest.yaml` — name/description/scope/target_tasks/description_trigger (все `pytest/Selenium`).
- `write-autotest/SKILL.md` (claude+codex) — frontmatter `description`.
- README L3, CHANGELOG (историч. записи).
Контрастные `«референс Selenium: …»` в телах (dom-explorer/fix-failed-test/write-autotest/code-writer/
references) — ОСТАВЛЕНЫ: это dual-stack teaching канона (в Selenium так, в canvas иначе), не описание
стека профиля. Канвас-правило `page-objects.md` L153 «не использовать Selenium/XPath» — оставлено (верно).

### 4. `stack.required` (§3)
`manifest.yaml`: `[one-web]` → `[one-web-kmp]` (поле repo-id).

### 5. «web-репо» → «KMP-репо»
manifest (5 мест), README, CHANGELOG. Основание: one-web-kmp — KMP-клиент, не Angular-web; и ТЗ
требует «профиль явно на pytest-playwright-canvas (KMP)». **Судейское решение** — вынес отдельно,
т.к. это третья ось помимо repo/stack; если тимлид считает избыточным — легко откатить.

### 6. README «Мульти-стэк» + CHANGELOG (§6)
- README: секция переписана — этот лейн = KMP (`one-web-kmp`, pytest/Playwright canvas). Развёл с
  `iva-qa-kmp-autotest-base` (Espresso): пометил его как ОТДЕЛЬНЫЙ нативный мобильный лейн, НЕ этот.
- CHANGELOG: строка «web (one-web) = этот лейн» → «KMP (one-web-kmp) = этот лейн».

### 7. Версия
`manifest.yaml` 0.2.0 → 0.3.0 + новая секция CHANGELOG `[0.3.0] — 2026-07-24` с полным описанием.

## Что ОСТАВЛЕНО с пометкой (не выдумывал)

- **`«референс Selenium: …»`/`selenium.common.exceptions`** в телах и references — контраст-подсказки
  и примеры трейсов Selenium-стека; не мёртвые пути, не описание стека профиля → сохранены.
- **Коллизия наименования**: `iva-qa-kmp-autotest-base` (Espresso) в README назван «KMP», при том что
  ЭТОТ лейн теперь тоже KMP (canvas-клиент one-web-kmp). Развёл текстом (браузерный canvas vs нативный
  мобильный), но само имя стороннего лейна не трогал — это вне scope. Возможно, заслуживает ревью.
- **dom-explorer L38** «Angular-снимок к canvas-стеку неприменим» — переименован репо-токен
  (one-web→one-web-kmp), но смысл «Angular-снимок неприменим к canvas» сохранён как есть.

## Проверки
- `uv run --extra dev -m pytest tests/catalog/test_iva_role_presets.py tests/catalog/test_manifest_schemas.py -q` (из apps/backend) — **129 passed**.
- `check_profile_version_discipline.py --diff-against origin/main` — **OK, 46 profile(s) clean**.

## Ветка / коммиты
Ветка `fix/qa-stack-naming-kmp` (база `feat/qa-codex-rework`), 3 коммита, без AI-подписей:
- `e80970e` manifest — стек/репо под факт + version 0.3.0
- `a9129ca` скиллы/субагенты/craft-stack — one-web-kmp + чистка мёртвых _selenium-ref
- `b086776` README/CHANGELOG — one-web-kmp, Мульти-стэк на KMP, секция [0.3.0]

НЕ пушено, НЕ PR (по регламенту — решает пользователь после ревью).