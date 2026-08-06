---
status: draft
type: impl-report
tema: qa-smoke-oracle
date: 2026-07-24
branch: feat/qa-smoke-oracle
permalink: tacticum/00-board/impl-qa-smoke-oracle-1
archived-at: 2026-07-31 17:27
---

# impl · qa-smoke-oracle

Обобщил smoke-оракул `test_run_commands_have_their_agents`, чтобы он покрывал
и qa-автотест-семейство (замечание Глеба). Раньше оракул был жёстко завязан на
run-implementation-семейство → для `iva-role-qa` пересечение пустое → early-return,
связка «команда → её суб-агенты» НЕ ассертилась.

## Что изменил

Файл: `apps/backend/tests/catalog/test_role_install_smoke.py`

- **`test_role_install_smoke.py:141-152`** (новый module-const) — вынес
  `ORCHESTRATION_FAMILIES`: список пар `(множество команд-оркестраторов, множество
  воркеров-agent_spec)`. Data-driven: новое семейство = одна строка.
  - run-implementation-семейство (без изменений по сути): `run-implementation/
    run-coder/run-tester/run-test-runner` → `coder/tester/test-runner`.
  - qa-семейство (новое): `write-autotest/batch-autotest/fix-failed-test`
    (лейн `iva-qa-autotest-base`) → `codebase-analyst/dom-explorer/code-writer`.
- **`test_role_install_smoke.py:155-172`** — тело `test_run_commands_have_their_agents`
  теперь итерирует по `ORCHESTRATION_FAMILIES`: `return` заменён на `continue`, чтобы
  роль без одного семейства всё равно проверялась на другое. Логика проверки внутри
  цикла идентична прежней (present = пересечение; воркеры обязаны быть `agent_spec`).
  Docstring дополнен qa-связкой, отсылка на ревью dev#119 п.3 сохранена.

Диф: 1 файл, +30 −11. Стиль/инварианты соседних тестов не тронуты.

## Как обобщил (принцип)

Оракул больше не хардкодит одно семейство и не делает ранний выход на первом
пустом пересечении. Он проходит по всем зарегистрированным семействам; для каждого,
чьи команды присутствуют в составе роли, требует полный набор воркеров как `agent_spec`.
Покрытие run-implementation сохранено дословно (то же множество, та же ассерция).

## Проверка связки (не early-return)

Прямой прогон логики оракула для `iva-role-qa`:
- present cmds: `['batch-autotest', 'fix-failed-test', 'write-autotest']` (непусто → qa-ветка реально исполняется)
- workers required: `['code-writer', 'codebase-analyst', 'dom-explorer']`
- agent_spec found: те же три · MATCH: True

 Id-ы сверены с манифестами `templates/iva-role-qa` (depends_on → `iva-qa-autotest-base`)
и `templates/iva-qa-autotest-base` (три команды kind=skill_spec + три agent_spec).

## Числа тестов (uv run --extra dev -m pytest tests/catalog/test_role_install_smoke.py, из apps/backend)

- **До** (pristine origin/main): `77 passed`
- **После**: `77 passed`

Число item'ов не изменилось (правка усиливает существующий параметр, не добавляет
новых тест-функций/параметризаций). Разница качественная: qa-параметр
`test_run_commands_have_their_agents[iva-role-qa]` теперь реально ассертит связку,
а не проходит вакуумно.

## Ветка / коммит

- Ветка: `feat/qa-smoke-oracle` (от `origin/main` @ `1061d10` — merge PR #119; отдельная от qa-codex-rework).
- Коммит: `06c4d7b` test(catalog): install-smoke оракул run-команд покрывает qa-автотест-лейн
- НЕ пушено, PR не создан.

## Заметка по базе ветки

Локальный `main` в основном дереве был на 24 коммита позади `origin/main` (clean
fast-forward), и файл оракула живёт в `origin/main`, а не в устаревшем локальном `main`.
Пересоздал worktree от `origin/main`, иначе файла не было бы. Локальный `main` не трогал.