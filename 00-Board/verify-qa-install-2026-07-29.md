---
title: Живой прогон установки QA-ролей (не автором)
type: report
status: draft
date: 2026-07-29
tags:
- qa
- install
- verify
- iva-role-qa
- iva-role-qa-web
permalink: tacticum/00-board/verify-qa-install-2026-07-29
---

# Живой прогон установки QA-ролей — вердикт

**Вердикт: доставка целая.** Потерь канона нет ни у одной из двух ролей, ни на одном
из двух CLI. Найдено два расхождения — оба вне авторства ветки (см. «Расхождения»).

Ветка `feat/qa-role-web`, HEAD `0eb79ef`, worktree
`~/tacticum/tacticum-dev-qa-web`, дерево оставлено чистым.

## Как гонялось

Изолированная одноразовая БД: `postgres:16-alpine` в docker на порту 55437,
`alembic upgrade head` (0039), после прогона контейнер остановлен. Ничего живого
не задето. Путь — клиентский: `seed_profile` → `provision_installation` →
`pull_installation_content` → `apply_actions` (оракул клиента из quickstart) в
пустой workspace.

## 1. Сид: порядок и заморозка

Семь лейнов, затем две роли — все `created`, отказов нет.

| лейн | версия | ингредиентов |
|---|---|---|
| tacticum-core-base | 0.1.1 | 6 |
| tacticum-autotest-core | 0.2.2 | 37 |
| iva-pytest-playwright-canvas-autotest-base | 0.1.3 | 17 |
| iva-pytest-selenium-autotest-base | 0.1.0 | 11 |
| iva-qa-tms-base | 0.2.0 | 15 |
| iva-qa-delivery-base | 0.1.0 | 4 |
| iva-qa-mcp | 0.1.0 | 2 |

Замороженные рёбра `depends_on` — ровно объявленные версии, в объявленном порядке:
`iva-role-qa 0.6.0` → core 0.1.1 / autotest-core 0.2.2 / canvas 0.1.3 / tms 0.2.0 /
delivery 0.1.0 / mcp 0.1.0; `iva-role-qa-web 0.1.0` — то же, но selenium 0.1.0
вместо canvas.

**Требование порядка проверено фактом, а не на слово.** Бампнул
`tacticum-autotest-core` до 0.2.3 с маркером в теле `write-autotest` уже ПОСЛЕ сида
роли: ребро роли 0.6.0 осталось на 0.2.2, маркер новой версии в установку **не
приехал** (`False`). Версия роли 0.6.1, засеянная после бампа, взяла 0.2.3. То есть
при обратном порядке сида роль молча встаёт на старый лейн, и ни одна статическая
проверка этого не увидит — они читают `templates/`, а не БД.

## 2. Установка: числа

| роль × CLI | ингредиентов в составе | actions | применено | пропущено | файлов на диске |
|---|---|---|---|---|---|
| iva-role-qa × claude-code | 85 | 83 | 82 | 1 | 80 |
| iva-role-qa × codex | 85 | 80 | 80 | 0 | 80 |
| iva-role-qa-web × claude-code | 79 | 77 | 76 | 1 | 74 |
| iva-role-qa-web × codex | 79 | 74 | 74 | 0 | 74 |

85 = 81 из лейнов + 4 своих; 79 = 75 + 4. Коллизий `ingredient_id` нет — состав
равен сумме. Единственный пропуск на claude-code — `merge_json` для `tacticum-mcp`
(шаг 0 quickstart, штатно).

По kind (iva-role-qa): agent_spec 3, instruction_pack 4, mcp_server_spec 4,
repo_config 59, skill_spec 15. Скиллы и субагенты доехали поштучно: 15/15 и 3/3 на
обоих CLI (у web-роли 14/14 и 3/3 — playwright-cli живёт в canvas-лейне и в
selenium-роль не входит по построению).

## 3. Целостность содержимого

- **пустых файлов: 0** во всех четырёх деревьях;
- **`{ingredient_id}` неподставленного: 0** — ни в путях, ни в телах;
- **склейки нет.** Проверено по объёму: каждый доставленный файл (кроме штатно
  сливаемых CLAUDE.md / AGENTS.md / .codex/config.toml / .mcp.json / .gitignore /
  settings.json) содержит тело своего владельца ДОСЛОВНО и не длиннее его больше
  чем на 400 символов;
- **все 85 и 79 тел найдены в дереве** — сверка по отпечатку тела из БД против
  установленного дерева, потерь 0;
- **канон стека одинаков на обоих CLI**: `.agent-kit/` — 57 файлов у qa и 52 у
  qa-web, списки claude-code и codex совпадают побайтно по составу. Ровно тот класс
  дефекта, на котором обожглись 28.07 (45 файлов канона не доехали на codex), —
  сегодня чист;
- **codex-тела субагентов — рукописные**, не регенерированные: `.codex/agents/*.toml`
  несут `model = "gpt-5.4"`, `model_reasoning_effort`, `sandbox_mode` (13–15 КБ каждый).

## 4. Ссылки глазами агента

Проход по УСТАНОВЛЕННОМУ дереву (не по `templates/`), резолв по семантике из самого
доставленного контента (`craft-stack/SKILL.md`: `$CRAFT/` = `.agent-kit/craft/`,
`<скилл>/references/<файл>` от `.agent-kit/craft/skills/`).

**186 ссылок проверено в каждом однорольном дереве, битых 0.** `{stack}` резолвится
в реальный каталог: `pytest-playwright-canvas` у `iva-role-qa`,
`pytest-selenium` у `iva-role-qa-web` — каталог в дереве ровно один, все
`stacks/{stack}/*.md` после подстановки указывают на доставленные файлы.

Пропущено как шаблон 2 кандидата на дерево — оба артефакты brace-expansion в
shell-команде уборки в `.agent-kit/README.md` (`craft/{recon,pytest-runner,...}.md`),
не ссылки.

Штатный `scripts/check_install_links.py` на обеих ролях × обоих CLI — чисто.

## 5. Две роли в одном workspace

Файлы **не затирают** друг друга. После второй установки в общем дереве 91 файл
(80 + 11 файлов selenium-лейна). Изменился ровно один путь на CLI — точка входа
(`CLAUDE.md` / `AGENTS.md`), и это штатный `append_section`: обе маркерные секции
`tacticum:iva-role-qa` и `tacticum:iva-role-qa-web` присутствуют по одному разу,
содержимое первой роли сохранилось.

**Но:** в совмещённом дереве оказываются ДВА каталога стека
(`pytest-playwright-canvas` и `pytest-selenium`), и `{stack}` перестаёт резолвиться
однозначно — 10 ссылок канона становятся неопределёнными. Это ровно тот дефект,
который `check_install_links` описывает как «две несовместимые правды о том, что
читать». Практический риск невысок (роли заводятся на разные поверхности и разные
репозитории), но совмещённая установка контрактом `{stack}` не поддержана — стоит
это либо запретить явно, либо признать в доке.

## Расхождения (оба ВНЕ авторства ветки)

**1. `~/.codex/config.toml` доставляется битым TOML.** Файл не парсится
`tomllib`: `Expected '=' after a key in a key/value pair (at line 15, column 18)`.
Причина — `renderers/codex.py:162`: `f"{k} = {json.dumps(v)}"`; для словаря
`env` json.dumps даёт JSON-синтаксис с двоеточиями (`{"JIRA_URL": "..."}`) вместо
TOML inline table (`{ JIRA_URL = "..." }`). Срабатывает на любом stdio-MCP с
`env_required` — здесь это `iva-atlassian-write` из `iva-qa-mcp`.
Строка идентична `main`; тот же путь бьёт ещё 5 профилей (`iva-analysis-fr`,
`iva-architect-mcp`, `iva-fr-analyst`, `iva-techwriter-mcp`,
`iva-2-client-shell-dev`), то есть и роль аналитика на проде. Дефект платформенный,
предсуществующий, не этой ветки.

**2. `~/.codex/config.toml` идёт без `merge_strategy`** — то есть перезаписывает
глобальный конфиг пользователя целиком. Репозиторный `.codex/config.toml` защищён
`create_if_missing`, пользовательский — нет. Тоже поведение платформы, не ветки.

## Пробел покрытия

`iva-role-qa-web` не покрыта живыми проверками установки: её нет ни в
`_GENERIC_ROLES` (`tests/e2e_install/test_install_flow.py:914`), ни в `ROLES`
(`tests/catalog/test_role_install_smoke.py:36`), golden-каталога
`tests/e2e_install/golden/iva-role-qa-web/` не существует. Роль проверяется только
манифестными тестами (`test_iva_role_presets`, `test_stack_placeholder_links`).
Сегодняшний живой прогон её доставку подтвердил, но в CI это не закреплено — на
следующей правке лейнов регрессия по web-роли не будет поймана.

## Прогнанные проверки репозитория

- `scripts/check_install_links.py --profile <роль> --cli <cli>` × 4 — чисто;
- `pytest tests/catalog/{test_role_install_smoke,test_iva_role_presets,test_stack_placeholder_links}.py` — 281 passed;
- `pytest tests/e2e_install/test_install_flow.py -k "qa or generic"` — 18 passed
  (матрица ролей × 2 CLI, golden совпал).

Автотесты продуктов (`one-web-kmp`, `one-web`) не гонялись — вне границы проверки,
стендов и учётных данных нет.