---
permalink: tacticum/00-board/step3-observability-2026-08-06
---

---
title: Шаг 3 — наблюдаемость: журнал неудачных опознаний и access-лог
type: report
status: draft
date: 2026-08-06
tags:
- iva-write
- observability
- helm
permalink: tacticum/00-board/step3-observability-2026-08-06
---

# Шаг 3 — наблюдаемость канала записи

Ветка `feat/iva-write-observability`, worktree `~/tacticum-worktrees/helm-observability`,
три коммита от `origin/main` (`0ef2cbe`). Не мержено, не пушено, на прод не катилось.

## Что сделано

**1. Каждая неудачная попытка опознания пишется в журнал.** `require_write_identity`
(`src/helm/interface/api/routers/iva_write.py`) кладёт строку перед КАЖДЫМ `raise`:
время, префикс предъявленного ключа (8 символов, `phk_ad91`), путь запроса без
query-строки, код причины. Значение ключа целиком не попадает никуда: в репозиторий
нет параметра, через который оно могло бы пройти.

Это единственная точка опознания на поверхности записи — её же зовёт напрямую MCP
(`iva_write_surface.py:957`), так что канал записи покрыт вместе с HTTP-ручками.

Причины (закрытый словарь):

| код | что случилось | кто чинит |
|---|---|---|
| `no_header` | заголовка `Authorization` нет вовсе | человек (не настроен токен) |
| `bad_scheme` | заголовок есть, но не `Bearer …` | человек (настроен неправильно) |
| `not_recognized` | оба резолвера токен не признали | человек (негодный ключ) |
| `hub_unreachable` | project-hub не ответил, запасной резолвер личность не дал | **мы** |
| `foreign_domain` | токен признан, но домен почты чужой | никто, объясняется словами |
| `no_membership` | личность есть, членства в тенанте нет (403) | мы, выдачей членства |
| `auth_not_configured` | не настроены `HELM_RESOLVE_URL`/`SERVICE_KEY` (503) | **мы** |

**Ответ клиенту не изменился ни на букву** — коды, тексты, `WWW-Authenticate`
сверяются в каждом тесте на отказ.

**2. Место записи — новая таблица `identity_attempt_log`, сосед не тронут.**
`credential_use_log` не подошёл, и это не вкусовщина: у него `subject` и `system`
NOT NULL и содержательны — он про подстановку доступа конкретному человеку в
конкретную систему. Здесь обе величины неизвестны по природе события: опознание не
состоялось, значит человека нет, а систему запрос ещё не называл. Писать туда
пришлось бы выдуманный `subject` и выдуманный `system` и испортить журнал, по
которому считают успехи согласий. Разные вопросы — разные таблицы.

Миграция `alembic/versions/obs460_identity_attempt_log.py`, голова одна
(`obs460_identity_attempt` поверх `mat250_requirement_material`, проверено).

**3. Access-лог traefik включён** в `docker-compose.prod.yml`: `--accesslog=true`,
`--accesslog.format=json`, `User-Agent` и цепочка прокси в `keep`, `Authorization`
не `keep` никогда. Пишем в stdout, а не в файл: `--accesslog.filepath` потребовал бы
тома и своей ротации, а без неё access-лог шлюза однажды доедает диск; через stdout
он идёт в `json-file`, которому здесь же поставлен потолок `50m × 5`.

## Три вещи, которые стоит знать при приёмке

**Ловушка отката, из-за которой запись живёт в своей сессии.** `deps.get_session`
откатывает транзакцию на любом исключении, а отказ опознания и есть `HTTPException`.
Строка, добавленная в сессию запроса, уехала бы в `rollback` ровно в том случае,
ради которого журнал заведён. Поэтому `record_failure` открывает СВОЮ сессию из
`app.state.session_factory` и коммитит сама. Тесты читают журнал после отказа
отдельным подключением и потому ловят эту ошибку, если она вернётся (мутация 1
ниже — про неё).

**Журнал не вправе ломать отказ.** `record_failure` не бросает наружу ничего:
упавшая база не должна превращать честный 401 ни в 500, ни тем более в проход.
Строка в Python-лог пишется до базы и безусловно — она и остаётся следом, если базы
нет. Проверено тестом.

**Расширение сверх буквы брифа, называю явно.** Добавил `ResolveUnavailable` —
подкласс `ResolveError` в `resolve_client.py`, который бросается на сетевую ошибку.
Все четыре нынешних вызывающих (`api/auth`, три MCP-сервера) ловят базовый класс и
продолжают ловить, решение о доступе не меняется. Нужно ради журнала: «наш резолвер
лёг и не пустил ВСЕХ» и «конкретный человек предъявил негодный ключ» — аварии
разной цены, по одному классу исключения неразличимые, и в разборе жалобы это и
стоило часов. Тест на то, что подкласс не обошёл `except ResolveError`, есть.

## Чего в журнале сознательно НЕТ

Причины «истёк» нет, хотя она была в брифе. На этом слое срок ключа не наблюдаем:
`resolve_token` отдаёт один и тот же `ResolveError` и на просроченный, и на
подделанный токен, а `resolve_dev_token` fail-closed отвечает `None` на любую
осечку. Код, который никогда не выставляется по-настоящему, был бы ложью в журнале.
Истечение видно только там, где оно решается, — у владельца ключа.

Префикса чужой схемы (`Basic …`) в журнале тоже нет: в этом заголовке лежит чужой
секрет (логин с паролем в base64), и колонки под него быть не должно.

Успешное опознание не пишется вовсе — журнал про отказы, «кому подставили доступ»
пишет `credential_use_log`.

## Тесты

Новый файл `tests/interface/test_iva_write_identity_log.py`, 13 тестов — по одному
на каждую ветку логирования плюс безопасность секрета, отказоустойчивость журнала и
закрытость словаря причин. `require_write_identity` берётся настоящая, без
`dependency_overrides`.

Собрано **3023 → 3036** (+13). Полный прогон: **3004 passed, 32 skipped** за 1:42.
`ruff` и `mypy` по затронутым файлам чисто (в репозитории есть предсуществующие
ошибки в `models.py:2258`, `domain/tenancy.py`, `db/repository.py` — не мои).

### Мутации (сломал → упало, вернул → прошло)

**1. Убрал `await session.commit()` в `record_failure`** — то есть вернул ту самую
ловушку отката:

```
FAILED test_no_header_logged_and_answer_unchanged
FAILED test_bad_scheme_is_its_own_reason
FAILED test_not_recognized_keeps_only_prefix
FAILED test_hub_unreachable_is_distinguished_from_bad_token
FAILED test_foreign_domain_records_the_person
FAILED test_no_membership_records_confirmed_email
FAILED test_not_configured_is_logged_too
7 failed, 6 passed in 1.05s
```

**2. Схлопнул различение `no_header` / `bad_scheme`** (обе ветки пишут `no_header`):

```
>       assert row.reason == attempts.REASON_BAD_SCHEME
E       AssertionError: assert 'no_header' == 'bad_scheme'
FAILED test_bad_scheme_is_its_own_reason
1 failed, 12 passed in 1.01s
```

**3. Сломал `prefix_of` — вернул ключ целиком** (утечка секрета в журнал):

```
E       AssertionError: assert 'phk_ad91f0e7...899aabbccddee' == 'phk_ad91'
FAILED test_not_recognized_keeps_only_prefix
FAILED test_hub_unreachable_is_distinguished_from_bad_token
FAILED test_foreign_domain_records_the_person
FAILED test_no_membership_records_confirmed_email
FAILED test_prefix_never_returns_more_than_allowed
5 failed, 8 passed in 1.03s
```

После возврата каждой правки: `13 passed`.

## Границы

«Что не трогаем» из плана соблюдено дословно: механика согласия и порядок
Jira → Confluence, ленивое продление, канал записи и его поверхность (поведение,
коды ответов, тексты отказов), схема ключей и их выдача, фоновый таймер продления —
в диффе не встречаются. Не мержил, не пушил, не деплоил. Прод трогал одним
read-only вызовом: `docker exec helm-traefik-1 traefik --help` — сверить имена
флагов access-лога с настоящим бинарём v2.11, чтобы шлюз не отказался стартовать на
деплое. Все шесть флагов подтверждены. Секретов в вывод не попадало.

## Что осталось

- Миграцию накатить на прод — отдельным шагом, не мой (`alembic upgrade head`).
- Access-лог traefik требует пересоздания контейнера шлюза (`up -d traefik`),
  короткий разрыв соединений — тоже прод, не мой.
- Читалка журнала (ручка/скрипт «что было по ключу за период») не заводилась: бриф
  просил запись, а таблица читается одним `SELECT … WHERE ts > … ORDER BY ts`.
  Если нужна ручка — это отдельная задача.

## Файлы

- `src/helm/infrastructure/db/identity_attempt_repo.py` (новый)
- `src/helm/infrastructure/db/models.py` — модель `IdentityAttemptLog`
- `alembic/versions/obs460_identity_attempt_log.py` (новый)
- `src/helm/interface/api/routers/iva_write.py` — `_note_identity_failure` + `require_write_identity`
- `src/helm/infrastructure/auth/resolve_client.py` — `ResolveUnavailable`
- `docker-compose.prod.yml` — access-лог traefik + ротация
- `tests/interface/test_iva_write_identity_log.py` (новый), `tests/infrastructure/test_models_metadata.py`

## Чем правил

Символьных операций Serena — 8 (`insert_after_symbol` ×2, `insert_before_symbol`,
`replace_symbol_body`, `replace_content` ×4, `find_referencing_symbols` перед
правкой `require_write_identity` — сигнатура не менялась, call-sites не тронуты).
`Write` — 4 (три новых файла + один целиком). `Edit` — 5: `test_models_metadata.py`
(запись в множестве, не символ), `docker-compose.prod.yml` (YAML, LSP нет) и три
правки в тестах/мутациях. Помеха: другие агенты в этой сессии переключали активный
проект Serena на свой worktree, приходилось переактивировать перед каждой символьной
операцией — один `find_referencing_symbols` из-за этого сперва ушёл в чужое дерево.