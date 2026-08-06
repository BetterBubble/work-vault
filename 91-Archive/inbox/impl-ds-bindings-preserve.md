---
title: 'Реализация: пересборка токенов ИВА не затирает code-bindings и version'
type: note
status: draft
created: 2026-07-27
tags:
- board
- design-system
- impl
permalink: tacticum/00-board/impl-ds-bindings-preserve
archived-at: 2026-08-04 10:01
---

# Реализация: пересборка токенов ИВА не затирает code-bindings и version

Ветка `fix/ds-bindings-preserve` в worktree `/Users/bubblemac/tacticum-worktrees/ds-bindings`
(репозиторий `tacticum-dev`, база — `main` @ `119ccb0`). Коммиты:

- `f20ba25` fix(design): keep hand-curated data when rebuilding IVA tokens
- `db73fb0` test(design): cover rebuild preservation of code-bindings and version

Итого `git diff main --stat`: 2 файла, +281 / −9.

## Что изменено

Файл `apps/backend/scripts/merge_iva_tokens.py` (было 341 строка, стало 455).

| Что | Было | Стало |
|---|---|---|
| Константы для сохранения | — | `YAML_KEY_RE` (103), `PRESERVED_YAML_KEYS` (109) |
| Чтение прошлого `$extensions` | — | `read_existing_extensions()` (232–248) |
| Разбор YAML на блоки верхнего уровня | — | `yaml_top_level_blocks()` (250–272) |
| Генерация yaml | `build_yaml_stub(profile)`, `version: 0.1.0` хардкодом (221–241) | `build_yaml_stub(profile, existing=None)` (274–319): stub тот же, но блоки из `PRESERVED_YAML_KEYS` подставляются из существующего файла посимвольно |
| Перенос `$extensions` в дерево | — | 402–408: читается прошлый `tokens.json`, блок дописывается последним ключом (там же, где он лежит сейчас) |
| Запись yaml | `write_text(build_yaml_stub(profile))` (332) | 431–437: читается существующий yaml и передаётся в генератор |
| Источник и приёмник | жёстко `SRC` / `DST_BASE` | флаги `--source-dir` (333) и `--out-dir` (339), дефолты прежние |

Сохраняются: `$extensions` верхнего уровня целиком (сейчас там только
`dev.tacticum.code-bindings`) и блоки yaml `description`, `organization_id`, `version`,
`status`. Токены при этом пересобираются из Figma-входа как раньше. Новая ДС (файлов ещё
нет) получает stub-значения — поведение не изменилось.

Почему сохраняем не только `version`, а четыре ключа: `description` в обоих
`design-system.yaml` руками дополнен абзацами про 0.2.0/0.3.0, `organization_id`
проставляется при сиде. Ни то, ни другое не выводится из Figma-экспорта — регенерация
откатила бы их той же механикой, что и версию.

Флаги `--source-dir` / `--out-dir` добавлены не ради теста «на коленке», а потому что без
них скрипт непроверяем: реальный вход `docs/concept/design/tokens/` заигнорен
(`.gitignore:33`) и на машинах разработки отсутствует.

## Приёмка

Прогон на реальных данных невозможен (входных Figma-экспортов нет). Сделано так: копии
реальных `design-systems/iva-web` и `design-systems/iva-mobile` положены в скретчпад,
скрипт прогнан на синтетическом входе с `--out-dir` в эти копии. Сами файлы в worktree не
трогались (`git status` чист).

Хеши блока `$extensions` и файла `design-system.yaml` до и после прогона:

```
8089a55…  BEFORE.iva-mobile.design-system.yaml     8089a55…  AFTER.iva-mobile.design-system.yaml
fa07565…  BEFORE.iva-mobile.extensions.json        fa07565…  AFTER.iva-mobile.extensions.json
8f470a7…  BEFORE.iva-web.design-system.yaml        8f470a7…  AFTER.iva-web.design-system.yaml
ecd20fe…  BEFORE.iva-web.extensions.json           ecd20fe…  AFTER.iva-web.extensions.json

diff: iva-web extensions: IDENTICAL / iva-mobile extensions: IDENTICAL
      iva-web yaml: IDENTICAL / iva-mobile yaml: IDENTICAL
```

Совпал не только `version`, а весь `design-system.yaml` — байт в байт.

Контрольная проверка на версии скрипта из `main` (тот же синтетический вход, копия
`iva-web`): `'$extensions' in tokens.json: False`, `version: 0.1.0`. То есть проблема
воспроизводится и починка не пустая.

Мутационная проверка теста: при отключённом переносе (`preserved_ext = None`,
`build_yaml_stub(profile, None)`) падают 5 тестов из 6; шестой — `fresh_design_system` —
проходит, как и должен. Мутант откачен, файл восстановлен из коммита.

## Тест

`apps/backend/tests/design/test_merge_iva_tokens_preserve.py` (167 строк), 6 кейсов:
новая ДС получает stub-дефолты; пересборка сохраняет `$extensions`; правленый yaml не
откатывается; токены при этом обновляются из входа; `$extensions` из входного файла не
вытесняет курируемый блок; iva-mobile ведёт себя так же. Скрипт грузится по пути через
`importlib` — паттерн из `tests/test_governance_gate.py` (скрипт вне пакета `backend`).

## Тесты

Прогон `pytest tests/design` тем же интерпретатором (venv основного дерева), до и после:

| | passed | failed | errors | skipped |
|---|---|---|---|---|
| до правки (`main`) | 20 | 1 | 28 | 1 |
| после правки | 26 | 1 | 28 | 1 |

+6 — ровно новые тесты. 1 failed + 28 errors были и до правки: на машине **не поднят
Docker**, а conftest поднимает контейнер Postgres на сессию; всё падающее — это тесты с
БД (`test_admin_workspace_attach`, `test_mcp_design_tools`, `test_seed_design_system`,
`test_migration_0015…`). Чужое не чинил.

`ruff check` и `mypy` (strict) на обоих изменённых файлах — чисто.

## Осталось незакрытым

- **Реальный прогон не выполнен** — входных Figma-экспортов нет на машине. Доказательство
  строится на копиях реальных выходов + синтетическом входе; сами токены при настоящем
  прогоне, разумеется, пересоберутся.
- **Тесты с БД не проверены** (Docker выключен) — но их код не затрагивался.
- **Сохранение `$extensions` — «всё или ничего»**: переносится весь блок верхнего уровня.
  Если в Figma-экспорте когда-нибудь появится собственный корневой `$extensions`, он
  проиграет курируемому (это зафиксировано тестом). Точечного слияния по ключам нет
  намеренно — оно бы усложнило без нужды при одном курируемом ключе.
- **`$extensions` внутри отдельных токенов** (например `com.figma.hiddenFromPublishing`)
  как приходили из входа, так и приходят — их сохранять не требуется, они генерируемые.
- Заметил рядом, не трогал: у `iva-web` в yaml `framework_hint: react`, тогда как
  описание и code-bindings говорят про Angular (`iva-one/libs/design-system`). Похоже на
  расхождение в метаданных — вне объёма задачи.

---

# Доделка по ревизии (2026-07-27, та же ветка)

Номера строк в первой части относятся к состоянию на `db73fb0`; после доделки файл вырос
до 599 строк и они сместились — ниже номера актуальные (`161f347`).

Ревизия нашла блокер в самой правке и четыре замечания. Все закрыты, коммиты поверх:

- `3e44e3e` fix(design): refuse to rebuild over output that cannot be read
- `161f347` test(design): cover the refusal paths and a real-file regression

Итого по ветке: 4 коммита, `git diff main --stat` — 2 файла, +609 / −13.

## Блокер: молчаливая потеря при нечитаемом выходе

Было: `JSONDecodeError` → WARNING → `None` → запись всё равно шла, словарь терялся,
`rc=0`. Стало: класс исключений `ExistingOutputError` (стр. 249–256), `read_existing_extensions`
на битом JSON поднимает его (стр. 257–281), `main` ловит один раз и возвращает **3 до любой
записи** (стр. 548–553). Порядок в `main` переставлен: всё чтение и все проверки — до
`write_text`, обе записи в самом конце.

Проверено на реальных данных: в копию `design-systems/iva-web/tokens.json` вставлен
`<<<<<<< HEAD`, прогон дал

```
ERROR: …/tokens.json is not valid JSON (Expecting property name enclosed in double quotes:
line 5 column 7 (char 52)). Cannot tell whether it holds $extensions to preserve — fix the
file (unresolved merge conflict?) and rerun. Nothing was written.
rc=3
```

sha256 обоих файлов после прогона не изменились.

## Четыре замечания

1. **Валидация результата.** `preserved_yaml_text()` (стр. 419–465) прогоняет собранный
   текст через `yaml.safe_load` и сверяет, что `design_system_id` в результате всё ещё равен
   профилю. Плюс более ранняя проверка: строки верхнего уровня существующего yaml, которые
   парсер не может отнести ни к одному ключу, — отказ (их содержимое иначе исчезло бы).
   `- item` на нулевом отступе теперь считается продолжением ключа (это легальный YAML и
   ровно то, что печатает `yaml.dump`) — иначе отказ срабатывал бы на честном файле.
2. **Ключ в кавычках.** `YAML_KEY_RE` (стр. 118–121) распознаёт `version:`, `"version":` и
   `'version':`. Это был тот же самый баг в новой обёртке: сохранение молча не срабатывало.
3. **Неизвестные ключи.** `yaml_carryover_warnings()` (стр. 344–371) печатает два списка:
   что отброшено (`figma_file_key`, `owner` — stub о них не знает) и что перегенерировано
   поверх ручной правки. Комментарий человека над сохраняемым ключом теперь переезжает
   вместе с ключом (`_extend_over_leading_comments`, стр. 328–343); шапка файла не
   захватывается — её пишет stub.
4. **`--out-dir`.** Справка исправлена (пишет В указанный каталог, имя профиля не
   дописывается). Добавлена защита: если yaml на месте назначения описывает другую ДС —
   отказ. Проверено: `--profile iva-mobile --out-dir <каталог web>` даёт `rc=3` и
   `ERROR: … describes design system 'iva-web', but --profile is 'iva-mobile'.`

Дополнительно (не просили, но дыра того же класса): если `tokens.json` есть, а yaml нет,
скрипт теперь вслух говорит, что пишет свежий stub 0.1.0 и что каталог никак не подтверждает
профиль.

Коды возврата зафиксированы в docstring: `0` — успех, `2` — неразрешённые алиасы (файлы
записаны), `3` — отказ писать.

## Побочный эффект, который стоит знать

`import yaml` сделал **PyYAML обязательным** для скрипта. Системный `python3` его не имеет —
прогон падает `ModuleNotFoundError`. Запускать нужно интерпретатором с бэкендовыми
зависимостями (`apps/backend/.venv/bin/python`); строка запуска в docstring обновлена.
Прецедент рядом: соседний `apps/backend/scripts/seed_community.py:23` импортирует `yaml`
так же. CI скрипт не вызывает (проверил: ссылок на `merge_iva_tokens` вне тестов и
сгенерированных заголовков нет).

## Приёмка после доделки

Повторил на свежих копиях реальных `iva-web` и `iva-mobile`: `$extensions` и весь
`design-system.yaml` — снова байт-в-байт (`diff` пуст, sha256 `ecd20fe…` / `fa07565…`
совпали с BEFORE). Предупреждений на реальных файлах ноль — то есть новые проверки не
шумят на том, что уже лежит в репозитории.

Регрессионный тест на том же: `test_committed_iva_web_yaml_rebuilds_byte_for_byte` берёт
закоммиченный `design-systems/iva-web/design-system.yaml` и `$extensions` из
закоммиченного `tokens.json` и требует no-op.

## Тесты

| | passed | failed | errors | skipped |
|---|---|---|---|---|
| до правки (`main`) | 20 | 1 | 28 | 1 |
| после первой части | 26 | 1 | 28 | 1 |
| после доделки | **36** | 1 | 28 | 1 |

+16 к базе — все новые. 1 failed и 28 errors те же, что и до правки: Docker выключен,
падают тесты с БД. `ruff` и `mypy --strict` на обоих файлах чисто.

Новых кейсов 10: битый JSON, сломанный блок-скаляр, незакрытая кавычка в переносимом
блоке, чужая ДС в `--out-dir` (все четыре — `rc=3` и оба файла байт-в-байт целы),
кавычки в имени ключа, комментарий над ключом, отброс неизвестных ключей с логом,
перегенерация `framework_hint` с логом, `tokens.json` без yaml, регрессия на реальном
файле.

## Наблюдение ревизора — фиксирую

**`framework_hint` в `PRESERVED_YAML_KEYS` не входит намеренно.** Значит «поправлю руками в
yaml» не работает: следующая пересборка вернёт значение из `PROFILE_METADATA`. Чинить это
можно **только** в скрипте. Теперь скрипт об этом предупреждает при прогоне, но сама
несогласованность остаётся:

- `iva-web`: `framework_hint: react`, а code-bindings ведут на Angular
  (`iva-one/libs/design-system`);
- `iva-mobile`: `framework_hint: react-native`, а привязки — на Compose Multiplatform
  (`:core:design-system`).

Обе строки живут в `PROFILE_METADATA` (стр. 96 и 110). Не трогал — отдельная задача, как и
велено.

## Что не делал (по указанию)

Атомарная запись через tmp + `os.replace`, правка `framework_hint`, матч компонентов по ID
вместо имён.