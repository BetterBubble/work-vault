---
title: impl-keystore-2026-07-29
type: note
status: draft
role: implementer
tags:
- implementer
- iva-write
- helm
created: 2026-07-29
updated: 2026-07-29
repo: helm
project: iva-write
permalink: tacticum/00-board/impl-keystore-2026-07-29-1
archived-at: 2026-08-05 15:19
---

Инвариантное ядро хранилища персональных доступов (план [[plan-iva-write-v2-2026-07-29]]
§3.1–3.2, разведка [[explore-helm-keystore-2026-07-28]]). Роутеров, HTTP-ручек, OAuth и
бота нет — сделано ровно перечисленное в задании.

**Ветка:** `feat/iva-write-keystore` · worktree `/Users/bubblemac/tacticum-worktrees/helm-iva-write-keystore`
(создан от `main` @ `5c52811`). Не пушил, не мержил, на серверы не ходил.

## Коммиты (3)

| SHA | Что |
|---|---|
| `10686f3` | Крипто-слой AES-GCM + настройка `keystore_master_key` + зависимость `cryptography` |
| `e5b6d9e` | Схема: `external_credential` + `credential_use_log` + миграция |
| `94b2486` | Репозиторий доступов + тесты |

Диффстат ветки против `main`: **11 файлов, 989 строк, удалений 0.**

## Что создано

| Файл | Строки | Содержимое |
|---|---|---|
| `src/helm/infrastructure/crypto/keystore_crypto.py` | 1–154 (новый) | `KeystoreCrypto`, `encrypt`/`decrypt`, `KeystoreKeyError`, `KeystoreDecryptError`, `keystore_crypto()`/`reset_keystore_crypto()` |
| `src/helm/infrastructure/crypto/__init__.py` | 1 (новый) | докстринг пакета |
| `src/helm/infrastructure/db/credential_repo.py` | 1–217 (новый) | `upsert_credential`, `get_credential`, `revoke_credential`, `log_use`, `normalize_subject`, `reason_for_denial`, `list_use_log` |
| `src/helm/infrastructure/db/models.py` | 3034–3107 · 3110–3138 | `ExternalCredential` · `CredentialUseLog` (+ импорт `LargeBinary`, стр. 26) |
| `alembic/versions/key300_external_credential.py` | 1–93 (новый) | две таблицы + три индекса |
| `src/helm/config.py` | 510–516 | `keystore_master_key: str \| None = None` рядом с блоком секретов бота |
| `tests/infrastructure/test_keystore_crypto.py` | 1–138 (новый) | 17 тестов крипто |
| `tests/infrastructure/test_credential_repo.py` | 1–260 (новый) | 12 тестов репозитория |
| `tests/infrastructure/test_models_metadata.py` | +4 | две новые таблицы в `EXPECTED_TABLES` (иначе существующий тест падал) |
| `pyproject.toml` · `uv.lock` | +1 · +2 | `cryptography>=43` (в lock уже был транзитивно — дифф в две строки) |

### Решения, которые стоит знать

- **Репозиторий принимает уже зашифрованный секрет** (`ciphertext`/`nonce`/`key_version`),
  а не открытый текст. Параметра, через который открытый секрет мог бы попасть в слой
  БД, физически нет — это структурная гарантия, а не соглашение. Шифрует вызывающий
  (будущая ручка приёма) через `keystore_crypto`.
- **`get_credential` fail-closed:** отозванная и просроченная записи не отдаются; причина
  отказа (`missing`/`revoked`/`expired`) вынесена в отдельную `reason_for_denial` — журналу
  причина нужна, вызывающему наружу она не течёт.
- **Ключ отсутствует → `KeystoreKeyError` на инициализации.** Дефолта нет, fallback'а
  «пишем открытым текстом» нет. Текст ошибки не содержит материала ключа, `__repr__`
  инстанса — тоже (проверено тестами).
- **`_utc()` в репозитории:** Postgres отдаёт `TIMESTAMPTZ` aware, SQLite в тестах теряет
  смещение. Без нормализации сравнение сроков падало `TypeError` ровно там, где решается
  вопрос доступа (поймано тестом, не рассуждением).
- **`ExternalCredential.__repr__` явный** — печатает только метаданные. Дефолтный repr
  SQLAlchemy значений и так не печатает, но строка представления модели с секретом это
  типовой канал утечки, и он закрыт схемой класса.

## Голова alembic

**Была одна:** `hrd224_allure_activity`. Подтверждено командой в worktree ДО написания
`down_revision`:

```
$ uv run alembic heads
hrd224_allure_activity (head)
```

Новая ревизия `key300_external_credential`, `down_revision = "hrd224_allure_activity"`.
После добавления голова снова одна:

```
$ uv run alembic heads
key300_external_credential (head)
```

DDL проверен офлайн-рендером `alembic upgrade hrd224_allure_activity:key300_external_credential --sql`:
обе таблицы, `PRIMARY KEY (tenant, subject, system)`, `BYTEA` под шифротекст/nonce,
`TIMESTAMP WITH TIME ZONE` под сроки, три индекса
(`ix_external_credential_expires_at`, `ix_credential_use_log_subject`, `ix_credential_use_log_ts`).

**Риск из плана §3.1 остаётся:** параллельная миграция того же дня от того же родителя
даст вторую голову и уронит деплой (`scripts/deploy.sh:53` делает `alembic upgrade head`).
Перед мержем сверить с соседними лейнами.

## Прогон

| Проверка | Результат |
|---|---|
| Новые тесты | `33 passed` (крипто 17 + репозиторий 12 + метаданные 4), 1.3 с |
| Полный `uv run pytest` | **2208 passed, 32 skipped, 0 failed**, 71.5 с |
| `ruff check` на изменённых файлах | 2 × E501, **обе на строках, которых я не касался** |
| `mypy` (strict, весь проект) | в моих файлах **0 ошибок** |

Про ruff и mypy честно: **базовая линия репозитория грязная** — на всём дереве
`ruff check src tests alembic` даёт 55 ошибок, `mypy` — 237 в 58 файлах, и это было до
меня. Что мои файлы ничего не добавили, проверено не на глаз:

- `git show main:src/helm/config.py | ruff check --stdin-filename … -` → `Found 1 error`,
  на ветке в том же файле по-прежнему 1 (строка 543, докстринг `vectors_allowed_email_set`);
- то же для `models.py` → 1 на `main`, 1 на ветке (строка 1947, комментарий про `status`);
- в выводе `mypy` grep по `credential|keystore|crypto|config.py|models.py` пуст.

Новых файлов (`keystore_crypto.py`, `credential_repo.py`, миграция, оба теста) ни один
линтер не поминает вовсе.

## Что осталось не сделано

Сознательно, вне объёма задания:

- **Приём токена** (MCP-тул / страница по одноразовой ссылке), статус, ручка отзыва,
  внутренняя ручка для шлюза — план §3.5 и §3.4. Ядро к ним готово.
- **`.env.example` и README** про `HELM_KEYSTORE_MASTER_KEY` не трогал — задание
  ограничивало правку настроек файлом `config.py`. Оператору строку в `.env.example`
  всё же стоит добавить: без неё ключ придётся выяснять из кода.
- **ADR в helm под «первое хранение чужих секретов»** не писал. Разведка отмечает, что
  действующая норма репозитория — «читаем чужие системы, пишем только в свою БД», и это
  первое её нарушение; документа нет вообще.
- **Процедура при компрометации мастер-ключа** (план §3.2) — нужна как текст-инструкция,
  кодом она не выражается: пометить записи недействительными, погасить канал, попросить
  людей перевыпустить токены.
- **Ротация ключа** заложена структурно (`key_version` рядом с шифротекстом, чужая версия
  падает честно), но процедуры перешифровки нет — при смене ключа старые записи станут
  нерасшифровываемыми, и это должно быть осознанным шагом с уведомлением людей.
- **Канонический выбор почты при 1:N `PersonEmail`** (план §3.3) — хранилище ключуется
  нормализованной почтой, но какая из нескольких почт человека канонична, решает не этот
  слой. Ручной путь привязки не делал.