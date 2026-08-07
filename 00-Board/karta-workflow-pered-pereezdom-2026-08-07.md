---
title: Карта CI-workflow перед переездом на GitLab
type: note
status: current
date: 2026-08-07
tags:
- ci
- gitlab
- migration
permalink: tacticum/00-board/karta-workflow-pered-pereezdom-2026-08-07
---

# Карта CI-workflow перед переездом на GitLab

Снято с `origin/main` = `757c5dc2` (07.08.2026), репозиторий `tacticum-dev`, каталог
`.github/workflows/`. Все прогоны — read-only, во временном worktree на detached HEAD.

## Риск, с которого надо начинать

**После переезда `.github/workflows/**` перестанут запускаться МОЛЧА.** GitLab читает
`.gitlab-ci.yml` и про каталог GitHub не знает: не будет ни ошибки, ни красного шага, ни
пропущенного джоба в интерфейсе — просто ничего. Пайплайн, которого нет, выглядит как пайплайн,
который прошёл.

Для двух джобов это особенно скверно, потому что они написаны **против того, чтобы расхождение
осталось незамеченным**, и умрут ровно тем способом, против которого написаны:

- **`taiga-commit-link`** — комментирует задачи в Taiga по ссылкам из коммитов. Перестанет
  комментировать → трекер начнёт расходиться с историей `main`, и никто не узнает.
- **`taiga-audit`** — еженедельная сверка «задачи в New, у которых уже есть коммит». Это
  страховка на случай, что первый молчал. Обе страховки снимаются одним переездом.

Тот же класс, что и в шапке `backend-ci.yml`: там записано, что до Taiga #620 файл
`backend-ci.yml` жил только на диске автора (каталог `.github/workflows/*` был в `.gitignore`) —
гейт существовал, но не запускался, и это не заметили. Переезд воспроизводит ту же ситуацию
для всех шести файлов сразу.

## Что есть: 6 файлов, 7 джобов

| # | Файл / джоб | Что делает | Триггеры | Фильтры путей | Секреты |
|---|---|---|---|---|---|
| 1 | `backend-ci.yml` → **`tests`** | `pytest tests -q -p no:randomly --ignore=tests/e2e_install` | `pull_request`; `push` в `main` | `apps/backend/**`, `scripts/**`, `.github/workflows/**`, `.gitignore` | — |
| 2 | `backend-ci.yml` → **`lint`** | `uv lock --check`; `ruff check .`; `ruff format --check .`; те же две команды **поимённо** по `../../scripts/taiga_commit_link.py` | те же | те же | — |
| 3 | `backend-ci.yml` → **`mypy`** | `mypy src` (strict); `mypy ../../scripts/taiga_commit_link.py` | те же | те же | — |
| 4 | `install-e2e.yml` → **`install-e2e`** | `pytest tests/catalog`, затем `pytest tests/e2e_install` | `pull_request`; `push` в `main` | `templates/**`, `apps/backend/**`, `docs/user_manuals/**`, сам файл | — |
| 5 | `nightly-install-e2e.yml` → **`install-e2e`** | реальный Codex ставит `tacticum-dev-base` на локальный стенд (positive + negative), артефакт — транскрипты | **только `workflow_dispatch`** | нет | `CODEX_AUTH_JSON` (без него шаг 1 валит джоб намеренно) |
| 6 | `profile-version-discipline.yml` → **`check`** | `check_profile_version_discipline.py` (с диффом против базы), `check_mirror_sync.py`, `check_install_links.py` в режимах `paths --all-profiles`, `xrefs --all-profiles`, `links` по `iva-role-qa` и `iva-role-qa-web` | `pull_request`; `push` в `main` | `templates/**`, три поимённых скрипта (`check_profile_version_discipline.py`, `check_mirror_sync.py`, `check_install_links.py`), сам файл | — |
| 7 | `taiga-audit.yml` → **`audit`** | `taiga_commit_link.py --report --report-ref main` | `cron: 0 6 * * 1` (пн 06:00 UTC) + `workflow_dispatch` | нет | `TAIGA_TOKEN`, `TAIGA_PROJECT_ID: "12"` |
| 8 | `taiga-commit-link.yml` → **`comment`** | `taiga_commit_link.py --range <диапазон>` — комментарии в Taiga | `push` в `main` | **нет фильтра путей** | `TAIGA_TOKEN`, `TAIGA_PROJECT_ID: "12"` |

Строк восемь, потому что `backend-ci.yml` несёт три джоба; файлов шесть, джобов семь.

Две детали, которые легко потерять при портировании:

- **Все три джоба `backend-ci.yml` целиком работают в `apps/backend`** (`defaults.run.working-directory`).
  Шага с другим рабочим каталогом там нет — разделение идёт **по цели**: корневой скрипт линтуется
  по относительному пути `../../scripts/taiga_commit_link.py`, поимённо.
- **Ограничение списком — не глушение гейта, а сознательное решение.** В `backend-ci.yml`
  (строки 96–101) записано: по всему `scripts/` конфиг бэкенда даёт **27 ошибок ruff и 23 mypy в
  семи чужих файлах**, поэтому каталог целиком не включают — иначе первый же PR упрётся в долг,
  которого его автор не создавал, и кончится это снятием гейта. Правильный путь — вычищать скрипт
  и дописывать его в список ещё одной строкой. **При переезде список надо перенести как есть**;
  соблазн «упростить до каталога» приведёт к красному пайплайну на ровном месте.

## Что не переедет само

**Первое по важности — `github.*`-переменные, потому что ошибка в них не красит пайплайн, а
меняет поведение.** Это два места:

- **`taiga-commit-link.yml` — логика диапазона.** Скрипт получает `--range`, и диапазон
  вычисляется из `github.event.before` и `github.sha`, с явной проверкой: если `before` пустой,
  нулевой, или **не предок** текущего `sha` (`git merge-base --is-ancestor`), разбирается один
  коммит вместо диапазона. Перенести «в лоб» на `CI_COMMIT_BEFORE_SHA` мало: проверка на предка
  и на нулевое значение — не украшение, а защита от пропуска коммитов (при неверном диапазоне
  задачи молча останутся без комментария). Плюс джобу нужна полная история — в GitHub это
  `fetch-depth: 0`, в GitLab это `GIT_DEPTH: 0`, и при дефолтном shallow-клоне `merge-base`
  просто не найдёт предка.
- **`profile-version-discipline.yml` — выбор базы для диффа.** Развилка по `github.event_name`:
  на PR дифф идёт против `origin/${{ github.base_ref }}`, на push — против `github.event.before`,
  а если тот пуст/нулевой/недоступен (`git cat-file -e`), скрипт запускается **без `--diff-against`,
  то есть только со статическими проверками**. Это тихая деградация по замыслу; при неверном
  портировании она станет постоянной, и дисциплина версий будет «зелёной», ничего не сравнивая.
  Тоже требует полной истории.

Остальное — механика, ошибки в ней видны сразу:

- экшены: `actions/checkout@v4`, `astral-sh/setup-uv@v5` (с `enable-cache` и
  `cache-dependency-glob: apps/backend/uv.lock`), `actions/setup-python@v5`,
  `actions/upload-artifact@v4` — в GitLab это образы + `cache:` + `artifacts:`;
- `secrets.*` → переменные CI/CD проекта (`TAIGA_TOKEN`, `CODEX_AUTH_JSON`), маскирование задавать
  вручную;
- блоки `concurrency` (группы `taiga-audit`, `taiga-commit-link-main`, обе с
  `cancel-in-progress: false`) → `resource_group:`, а не `interruptible`;
- `permissions: contents: read` → в GitLab отдельного аналога нет, права даёт токен джоба;
- `shell: pwsh` в nightly (скрипты `run_install_e2e.ps1`) → нужен раннер с PowerShell либо образ с ним;
- `paths:`-фильтры → `rules:changes`, **и семантика другая**: в GitLab `changes` на merge-запросах
  и на push считается по-разному, а «файл не менялся → джоба нет» может дать пустой пайплайн там,
  где GitHub просто не запускал workflow;
- `cron` из `taiga-audit.yml` → расписание в GitLab задаётся **в интерфейсе проекта** (Pipeline
  schedules), а не только в файле: перенос yml сам по себе еженедельную сверку не воскресит.

## Оценка по джобам

Суждение моё, с обоснованием; порядок величины, не смета.

| Джоб | Оценка | Почему |
|---|---|---|
| `backend-ci / tests` | **механически**, ~1 ч | Одна команда `uv run … pytest`, внешних сервисов не нужно. Вся работа — образ с uv, кеш по `uv.lock`, `rules:changes` |
| `backend-ci / lint` | **механически**, ~0.5 ч | Четыре команды подряд, состояния нет. Не потерять поимённый список корневых скриптов |
| `backend-ci / mypy` | **механически**, ~0.5 ч | То же, две команды |
| `install-e2e` | **механически, но с инфраструктурой**, ~0.5 дня | Команды тривиальны, но `tests/e2e_install` поднимает свои контейнеры (postgres) — нужен docker-in-docker или `services:`, и это место, где обычно уходит время |
| `nightly-install-e2e` | **выключенный скелет — не переносить**, 0 | По собственному заголовку это disabled-by-default каркас (US #653): без `CODEX_AUTH_JSON` он намеренно валится первым шагом, триггер только ручной. Переносить нечего, пока его не решат включать; тогда это отдельная задача с pwsh, npm, docker и секретом |
| `profile-version-discipline` | **переписывать логику**, ~1 день | Развилка PR/push и три варианта базы диффа — ровно тот случай, где неверный перенос даёт молчаливую деградацию до статических проверок. Нужна проверка на первом пуше в ветку, на force-push и на shallow-клоне |
| `taiga-audit` | **механически + ручная настройка**, ~1 ч | Сама джоба проста; расписание заводится руками в интерфейсе, и об этом легко забыть — тогда сверка просто не будет идти |
| `taiga-commit-link` | **переписывать логику**, ~0.5 дня | Вычисление диапазона с проверкой предка; плюс `GIT_DEPTH: 0`; плюс нет фильтра путей — джоба обязана идти на каждый push в `main` |

**Порядок величины на весь порт: 2–3 человекодня** без nightly. Из них примерно половина — два
джоба с логикой диапазона (`profile-version-discipline`, `taiga-commit-link`), где цена ошибки
выше и нужна отдельная проверка поведения, а не «зелёный пайплайн».

## Что подтверждено прогоном

На `757c5dc2` прогнаны локально все 7 шагов `backend-ci.yml` командами из файла
(`uv run --python 3.12 --extra dev`), все — код возврата 0:

| шаг | результат |
|---|---|
| `uv lock --check` | Resolved 112 packages, дрейфа pyproject↔lock нет |
| `ruff check .` | All checks passed! |
| `ruff format --check .` | 541 files already formatted |
| `ruff check … ../../scripts/taiga_commit_link.py` | All checks passed! |
| `ruff format --check … taiga_commit_link.py` | 1 file already formatted |
| `mypy src` | Success: no issues found in 235 source files |
| `mypy ../../scripts/taiga_commit_link.py` | Success: no issues found in 1 source file |

Плюс pytest тем же днём: без `e2e_install` — 2408 passed / 43 skipped / 15 xfailed;
`tests/e2e_install` — 231 passed. Красного на `main` нет. Версии из lock: ruff 0.15.19, mypy 2.1.0.

Попутно (на зелёность не влияет): ruff печатает `warning: Invalid # noqa directive on
dev/e2e/install_scenario.py:173` — строка-пояснение `# noqa-S603/S607: …` принимается за
директиву; настоящие подавления стоят ниже и работают.

## НЕ проверено

- **Сами workflow не запускались.** Прогонялись только команды из их шагов, локально на
  macOS/arm64. Поведение раннера `ubuntu-latest` может отличаться — в первую очередь у
  `install-e2e` (докер) и nightly (секреты, pwsh, npm).
- **Четыре workflow не гонялись вовсе:** `nightly-install-e2e.yml`,
  `profile-version-discipline.yml`, `taiga-audit.yml`, `taiga-commit-link.yml` — по ним читались
  файлы, а не результаты.
- **Цифры долга по `scripts/` (27 ruff + 23 mypy в семи файлах) взяты из комментария в
  `backend-ci.yml`**, своим прогоном не подтверждены; давность долга не устанавливалась.
- **Что реально показывал GitHub на последних PR, не смотрел** — `gh` не установлен, вывод о
  зелёности сделан из локального прогона тех же команд.
- **Оценки трудоёмкости и раскладка «механически / переписывать» — суждение, а не замер.**
  Ни один джоб на GitLab не переносился даже пробно.
- **Синтаксис GitLab под каждый пункт не выверялся по документации** (`rules:changes`,
  `resource_group`, `GIT_DEPTH`, `CI_COMMIT_BEFORE_SHA` названы по памяти); перед портом
  сверить с актуальными доками.
