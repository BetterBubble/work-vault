---
title: verify-merge-iva-write-2026-07-30
type: note
permalink: tacticum/00-board/verify-merge-iva-write-2026-07-30-1
status: draft
created: 2026-07-30 14:30
updated: 2026-07-30 14:30
tags:
- board
- iva-write
- verify
- merge
archived-at: 2026-08-07 11:22
---

# Проверка мержа feat/iva-write-keystore ← origin/main (30.07.2026)

Проверял verifier, только чтение — код не правил, не коммитил, не пушил. Дерево
`/Users/bubblemac/tacticum-worktrees/helm-iva-write-keystore`, HEAD `49aeb3d`, состояние на
входе чистое. Baseline `origin/main` = `85a0676`, мерялся в ОТДЕЛЬНОМ worktree
(`git worktree add … 85a0676`), не через стэш общего дерева; после замеров worktree снят.
Postgres для миграций — два одноразовых контейнера `postgres:16` (порты 55437/55438), оба удалены.

**Вердикт: мерж прошёл чисто. Регрессий нет ни по одному из четырёх пунктов.**

## 1. Тесты — ПРОШЛО

| | passed | skipped | failed |
|---|---|---|---|
| `origin/main` (85a0676) | 2229 | 32 | 0 |
| после мержа (49aeb3d) | **2421** | **32** | **0** |

`uv run pytest -q`, exit 0, 77.6 s. Арифметика сходится точно: 2229 (main) + 192 (тесты десяти
новых файлов ветки, `pytest --collect-only`) = 2421. Ни один тест не потерян и не посчитан дважды.
Против названного лидом до-мержевого числа ветки (2371) дельта +50 — это тесты, пришедшие из
`origin/main`. Skipped как было, 32.

Падений нет. В выводе есть шум `PytestUnhandledThreadExceptionWarning: Event loop is closed`
(aiosqlite, `tests/interface/test_intake.py`) — это warning при закрытии соединения, не падение,
и он предсказуемо есть и на baseline.

## 2. Миграции — ПРОШЛО, накат от нуля выполнен на живом Postgres

`uv run alembic heads` → ровно одна голова: `mrg350_iva_write_pmb (head)`.
На baseline голова была `pmb216_pregate_decision` — то есть merge-ревизия действительно свела
линии, а не добавила третью.

**Накат от нуля сделан на настоящем Postgres 16, не на sqlite.** Sqlite тут не подошёл бы:
`alembic/env.py` строит async-engine из `Settings().database_url` под asyncpg, а часть из 110
ревизий (11 файлов) использует postgres-специфику. Поднял чистый контейнер, прогнал
`HELM_DATABASE_URL=postgresql+asyncpg://…:55437/verify uv run alembic upgrade head` — вся цепочка
от первой ревизии до головы применилась без ошибок, обе линии по очереди:
`hrd224_allure_activity → pmb208…pmb216` и `hrd224_allure_activity → key300 → oa310 → bot330 → idn340`,
затем `(idn340, pmb216) → mrg350_iva_write_pmb`.

После наката: `alembic current` → `mrg350_iva_write_pmb (head) (mergepoint)`, 115 таблиц в public.
Таблицы обеих линий на месте — `bot_task`, `intake_candidate`, `sales_file` (линия PM-бота) и
`external_credential`, `credential_use_log`, `oauth_state`, `bot_dialog_pending` (линия iva-write).
`pregate_decision` отдельной таблицей не является — ревизия `pmb216` кладёт решение колонками
на кандидата.

**Дрейф модели против БД (`alembic check`) — доэксистующий, мержем не изменён.** Команда FAILED
и на ветке, и на baseline: 34 расхождения там и 34 здесь, множества совпадают ПОБУКВЕННО
(`comm` в обе стороны пуст). Всё это косметика на старых таблицах: переименования индексов
(`ix_meeting_date` → `ix_meeting_meeting_date` и подобные) и `TEXT` vs `String`. Ни одна из новых
таблиц ветки в дрейф не попала — их миграции описывают модели точно.

## 3. Чужой код не потерян — ПРОШЛО

`git diff origin/main...HEAD` — 9806 вставок и ровно **4 удаления** на всё дерево. Разобрал каждое:

- `src/helm/config.py`, 1 строка — переформат однострочного докстринга
  `vectors_allowed_email_set` в многострочный (заодно чинит E501, см. п.4). Код не тронут.
- `src/helm/main.py`, 3 строки — расширения, а не удаления: импорт
  `from helm.interface.mcp import …` дополнен `iva_write_surface`; докстринг
  `_McpMountTrailingSlash` дописан третьим адресом; кортеж `_PREFIXES` дополнен
  `iva_write_surface.MOUNT_PATH`.

**`src/helm/infrastructure/db/models.py` — чистый append, 0 удалений.** Единственный хунк
`@@ -3185,3 +3185,236 @@`, то есть всё новое дописано ПОСЛЕ конца файла origin/main.
3187 строк на main → 3420 на ветке, ровно +233. **Класс `BotTask` присутствует целиком и
побайтово совпадает с версией origin/main** — он заканчивается на строке 3185, а хунк начинается
после него. Конфликт разрешён правильно.

Остальные мерженные файлы:

- `docker-compose.prod.yml` — только добавления: 7 строк env для `/mcp/iva-write` в сервис helm
  и новый сервис `mcp-atlassian`. Правки origin/main (4 строки, пришли мержем) на месте.
- `tests/infrastructure/test_models_metadata.py` — только +9 строк в `EXPECTED_TABLES`
  (четыре новые таблицы с комментариями). Ничего из ожидаемых таблиц main не вычеркнуто.

## 4. Линтер и типы — ПРОШЛО, ветка немного ЛУЧШЕ baseline

ruff 0.15.20, одна и та же версия на обеих сторонах:

| цель | baseline | после мержа | дельта |
|---|---|---|---|
| `ruff check .` | 80 | **79** | −1 |
| `ruff check src tests` | 45 | **44** | −1 |

Ветка не добавила ни одной ошибки и убрала одну (тот самый E501 в докстринге `config.py`).
Замечу: обе цифры ненулевые и были такими до ветки — 79 ошибок это долг репозитория
(42 × E501, 15 × I001, 8 × F401 и прочее), размазанный по `web/`, `scripts/`, `alembic/versions/`
и src/tests. К этому мержу отношения не имеет.

mypy (strict, `files = ["src", "tests"]`):

| | ошибок | файлов с ошибками | проверено файлов |
|---|---|---|---|
| baseline | 272 | 66 | 618 |
| после мержа | **272** | **66** | 647 |

Совпадение чисел не случайное — сверил множества ошибок построчно. Разница ровно в двух записях,
и это ОДНИ И ТЕ ЖЕ две ошибки со сдвигом номеров строк
(`test_models_metadata.py:165→174` и `:180→189`), потому что ветка дописала в этот файл 9 строк.
Больше отличий нет.

**В новых файлах ветки — 0 ошибок mypy.** ~9800 добавленных строк (`iva_write`, `keystore_crypto`,
`credential_repo`, `oauth_state_repo`, `bot_iva_write`, `iva_write_surface` и их тесты) проходят
strict начисто; grep по `iva_write|keystore|credential|oauth` в выводе mypy даёт пустой результат.

## Что осталось за рамками

Гонял только то, что просил лид. Не проверял: поведение `mcp-atlassian` вживую (контейнер не
поднимался, только конфиг прочитан), реальный OAuth-поток к Jira/Confluence, деплой. Долг репо по
ruff (79) и mypy (272) — доэксистующий, отдельной задачей.

## Наблюдения

- [regression] Ни одной регрессии по четырём проверенным пунктам не найдено #verify
- [factual] 2421 passed / 32 skipped / 0 failed после мержа против 2229 / 32 / 0 на origin/main; дельта +192 равна числу тестов ветки #tests
- [factual] Одна голова alembic `mrg350_iva_write_pmb`; вся цепочка из 110 ревизий накатывается от нуля на чистом Postgres 16 без ошибок #migrations
- [factual] models.py смержен чистым append, 0 удалений; `BotTask` совпадает с origin/main #merge
- [debt] `alembic check` FAILED и до, и после мержа — 34 идентичных расхождения на старых таблицах (имена индексов, TEXT vs String) #debt
- [debt] ruff 79 ошибок и mypy 272 — долг репозитория, к мержу отношения не имеет #debt
- [method] Baseline снимался в отдельном worktree, а не стэшем общего дерева — рядом работали другие агенты #method

## Связи

- relates_to [[verify-iva-write-branch-2026-07-30]]