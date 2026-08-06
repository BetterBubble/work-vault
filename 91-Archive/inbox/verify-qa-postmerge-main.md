---
status: draft
type: report
role: verifier
topic: verify-qa-postmerge-main
branch: main
commit: 20ff9b8
date: 2026-07-24
permalink: tacticum/00-board/verify-qa-postmerge-main-1-1
archived-at: 2026-07-31 17:27
---

# Верификация QA post-merge (main @ 20ff9b8, PR #133)

Независимый реальный прогон двух НЕ-DB наборов тестов каталога после мержа PR #133
(QA-профиль iva-role-qa + автотест-лейн + per-role MCP-лейны). Не по логам — реальный pytest.

## Окружение
- Репо: `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main` @ `20ff9b8`.
- Штатного venv в репо нет (`apps/backend/.venv` / `.venv` отсутствуют). Создан временный venv на Python 3.12.4
  (requires-python >=3.12; системный homebrew python 3.14 — слишком новый).
- Установлены зависимости, достаточные для сбора и прогона: pytest, pytest-asyncio, pyyaml, jsonschema,
  alembic, sqlalchemy. Alembic/sqlalchemy/pytest_asyncio импортируются в `tests/conftest.py` на верхнем уровне,
  поэтому нужны даже для collection. Docker/Postgres НЕ поднимался — целевые тесты его не требуют.

## Почему БД/докер не нужны
`tests/conftest.py` содержит docker/Postgres-фикстуры (`postgres_url`, `db_session`), но они lazy — вызываются
только тестами, которые их запрашивают. Оба целевых файла БД-фикстуры не используют: импортируют лишь
json/yaml/pathlib/jsonschema и валидируют статические манифесты/JSON-схемы. Пропущенных из-за БД/докера тестов НЕТ.

## Команды запуска
Пофайлово:
- `python -m pytest tests/catalog/test_iva_role_presets.py -p no:cacheprovider -rsx`
- `python -m pytest tests/catalog/test_manifest_schemas.py -p no:cacheprovider -rsx`

Совместно (как в задании):
- `python -m pytest tests/catalog/test_iva_role_presets.py tests/catalog/test_manifest_schemas.py -p no:cacheprovider -rsx`

(из `apps/backend`, интерпретатор — временный venv на Python 3.12)

## Результаты

| Файл | passed | failed | skipped | RC |
|------|--------|--------|---------|----|
| test_iva_role_presets.py | 35 | 0 | 0 | 0 |
| test_manifest_schemas.py | 38 | 0 | 0 | 0 |
| ИТОГО (совместно) | 73 | 0 | 0 | 0 |

- `35 passed in 0.51s` / `38 passed in 0.07s` / совместно `73 passed in 0.54s`.
- Флаг `-rsx` не показал ни одного skip/xfail/xpass — skipped = 0 честные.

## Падения
Нет.

## Вердикт
ЗЕЛЁНЫЕ оба набора на `main` @ 20ff9b8. 73/73 passed, 0 failed, 0 skipped, docker не требовался.
Acceptance для этих двух НЕ-DB наборов: PASS.

## Примечания
- В рабочем дереве один untracked-файл `docs/adr/0060-profile-interaction-model-mcp-scoping-pipeline-gates.md`
  (предсуществующий, не создан верификатором) — на результат тестов не влияет.
- Верификатор производит доказательство; подлинность аудитирует контролёр.