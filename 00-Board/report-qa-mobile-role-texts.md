---
title: report-qa-mobile-role-texts
type: note
permalink: tacticum/00-board/report-qa-mobile-role-texts
status: draft
date: '2026-08-04'
project: tacticum-dev / QA / роль iva-role-qa-mobile
tags:
- board
- qa
- mobile
- implementer
- report
---

# Тексты роли `iva-role-qa-mobile` приведены к объединённому репозиторию

**Ветка:** `fix/qa-mobile-role-texts` (от `feat/qa-mobile-kit-adoption`), worktree
`/Users/bubblemac/tacticum-worktrees/qa-role-texts`. Два коммита, НЕ пушил.

- `a487d09` — тексты роли + версия + CHANGELOG (5 файлов роли);
- `45d33dc` — перегенерация golden дерева установки (2 json).

Зона соблюдена: правился только `templates/iva-role-qa-mobile/` (плюс golden как следствие).
Канон стека (`iva-pytest-appium-autotest-base`) не трогал — он ветка соседнего воркера.

## Что было ложью и чем заменено

Долг, признанный в CHANGELOG роли ещё в 0.2.0: тексты САМОЙ роли описывали две репы, пару
идентификаторов `su.ivcs.ucim` / `su.ivcs.kmp` и «два поколения клиента».

Приведено к фактам (сверено с кодом `ivaqa/mobile` и с уже переписанным каноном стека 0.2.0,
чтобы точка входа и канон говорили одно):

- репозиторий один, приложение одно — `su.ivcs.aura` на обеих платформах;
- платформа выбирается только `--device-os` (`Android` | `iOS`), и парный маркер
  (`android` | `ios`) обязан совпадать — иначе пустой отбор или тест на не том устройстве;
- пакетов локаторов два, по платформам, они не взаимозаменяемы: Android (Compose) —
  `id` → `content-desc` → точный текст через `tools.i18n` → `bounds`; iOS (XCUITest) —
  `-ios predicate` по типу плюс видимый текст через `i18n`, `bounds` нет ни одного;
- опора у обеих платформ одна — видимый текст через `i18n`, поэтому `--lang` ломает адресацию
  на обеих, а не только на Android.

## Места правки

**По списку задачи (5):** шапка `manifest.yaml` (блок «одна роль на две поверхности»
переписан, прежнее обоснование убрано в историю), `description` (первый абзац + пункт 3),
`ingredients/repo-configs/claude-code/CLAUDE.md.fragment`,
`ingredients/repo-configs/codex/AGENTS.md.fragment` (в каждом по четыре места: вводный абзац,
«Незыблемое», раздел адресации целиком, оговорка про `WebView`), запись 0.1.0 в `CHANGELOG.md`
не переписана — добавлены три сноски-уточнения (0.1.0 конструкция, пункт 3 «две стратегии»,
и «долг закрыт» к разделу долга в 0.2.0).

**Сверх списка нашлось четыре места:**

1. `manifest.yaml:name` — «(pytest + Appium, нативный и KMP-клиент)» → «Android и iOS одного
   клиента»;
2. `manifest.yaml:stack.optional` — `[pytest-appium, one-mobile-ios, one-mobile-android]`
   называл два репозитория, которых больше нет → `[pytest-appium, mobile]` с комментарием;
3. шапка ОБОИХ фрагментов — `the autotest-repo conventions of the mobile repos`, репы во
   множественном числе → `this mobile autotest repository`;
4. `ingredients/repo-configs/codex/config.toml.template` — Serena описана как навигация по
   автотест-коду «pytest + **Selenium**», хвост копирования с web-роли: чужой стек в тексте
   мобильной роли. → `pytest + Appium`.

Плюс два содержательных добавления, которых в точке входа не было, а в каноне стека есть
(чтобы не расходились): маркер обязан совпадать с `--device-os`, и обратная ошибка WebView
(`WEBVIEW_su.ivcs.aura` в списке контекстов ≠ экран в WebView; для защиты входа и PIN
правильный контекст `NATIVE_APP`).

## Версии и прогоны

- Версия роли `0.3.1` → `0.4.0`, запись `## [0.4.0] — 2026-08-04` в CHANGELOG.
- `scripts/check_profile_version_discipline.py --diff-against origin/main` → **OK, 63 профиля
  чисто**.
- Перегенерация golden: `E2E_INSTALL_REGEN_GOLDEN=1 pytest tests/e2e_install/test_install_flow.py
  -k qa` → **8 passed, 43 deselected**; изменились `golden/iva-role-qa-mobile/claude-code.json`
  и `codex.json`.
- Полный прогон `pytest tests/catalog tests/e2e_install -q` → **1247 passed, 42 skipped**
  (100.9 s). Красных нет.

Прогон делал интерпретатором основного дерева
(`/Users/bubblemac/tacticum/tacticum-dev-qa-combined/apps/backend/.venv/bin/python -m pytest`) —
в worktree своего `.venv` нет; rootdir и templates при этом брались из worktree.

## Хвост вне моей зоны (не трогал)

`docs/user_manuals/iva-role-qa-mobile-profile-quickstart.md` несёт ту же протухшую таблицу:
строки 13–14 — `one-mobile-ios` / `su.ivcs.ucim` и `one-mobile-android` / `su.ivcs.kmp`,
строка 115 — «клон `one-mobile-ios` или `one-mobile-android`». Это `docs/`, а не
`templates/iva-role-qa-mobile/`, поэтому оставил лиду на решение: отдельной задачей или
довеском к этой ветке.

## Связано

[[recon-mobile-repo-delta-2026-08-04]] · [[critic-mobile-fit-0804]]
