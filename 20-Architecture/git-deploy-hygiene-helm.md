---
title: git-deploy-hygiene-helm
type: guide
permalink: tacticum/20-architecture/git-deploy-hygiene-helm-1
tags:
- helm
- git
- deploy
- hygiene
- ops
- worktree
- playbook
---

## Гигиена git и деплоя (helm)

Выводы из разбора 2026-07-17 (сессия доработки RAG#2). Как держать git-хозяйство и деплой helm чистыми, чтобы не планировать от расходящихся баз и не терять работу. Дополняет [[team-lead-playbook]] и раздел деплоя в [[session-state]].

## Три точки должны совпадать — проверять ДО работы
- Перед любой доработкой сверять **локальный `main` = `origin/main` = прод `/opt/helm` HEAD**. Все три на одном коммите — иначе планируешь/делаешь от устаревшей базы.
- В сессии 2026-07-17: локальный checkout стоял на устаревшей `feat/analyst-mcp-server` (отстал от `main` на 64 коммита, 8 тулов vs 14 на проде), локальный `main` отставал от `origin/main` на 22 коммита. Живой прод = `origin/main`. Планировать надо было от `main`, не от локального HEAD.
- Прод git-состояние читается через **ssh-manager** (`server helm`): `cd /opt/helm && git rev-parse HEAD && git status --short`. Read-only, безопасно, прод-гейт не режет.

## Observations
- [git] Всегда `git fetch origin` + сверка `git rev-list --left-right --count main...origin/main` перед стартом; локальный checkout переводить на `main` (не работать со случайной старой ветки).
- [git] Прод `/opt/helm` держать на `main` и **сверять его HEAD с origin/main** — они должны быть равны (в норме b5d1739 на момент записи).
- [drift] **Untracked-код в `src/` на проде = дрейф и риск.** Деплой = REBUILD с `COPY src` → любой файл в `src/` попадёт в образ, даже если его нет в git (пример: `mr_source.py`, лежал на проде вне git, никем не импортировался). Регулярно `git status --short` на `/opt/helm`; код возвращать в git, лишнее убирать осознанно.
- [worktree] Worktree-хозяйство разрастается незаметно (в сессии было **58 worktree**, стало 1). Чистить периодически: `git worktree remove <path>` **без `--force`** — git сам откажет по грязным/залоченным, ничего незакоммиченного не потеряешь; ветки при этом сохраняются. После — `git worktree prune`.
- [worktree] **Незакоммиченная работа воркеров оседает в worktree и теряется при сносе.** Пример: фикс cross-rerank (source-aware + rank-floor, 6 code + 4 test + ab-kit) лежал UNCOMMITTED в worktree `helm-cross-rerank-tune` — снёс бы worktree, потерял бы фикс. **Перед сносом worktree — закоммитить в ветку** (`salvage/<label>`), не `--force` вслепую.
- [branches] Ветки чистить через `git branch -d` (НЕ `-D`): удаляет только слитые в текущую, неслитым откажет — уникальная работа сохранится сама. Защищать `main` и служебную `deploy-main`. В сессии удалено 74 слитых, осталось ~30 неслитых (архив + salvage).
- [salvage] Незавершённую/неслитую работу не уничтожать — коммитить в `salvage/<тема>` ветки и оставлять решение владельцу. Так спасены: cross-rerank-фикс, 5 агентских фич (task-hygiene с alembic-миграцией, operator-review, conformance-ui×2, redact), docs-концепты (130KB, были вне git), mr_source.py с прода.
- [deploy] ~~Прод helm БЕЗ git-доступа к GitHub → доставка кода = git-bundle с локали.~~ **ОПРОВЕРГНУТО 30.07: доступ ЕСТЬ, но не у `root`.** Тянуть надо от пользователя `tacticum-deploy` — у него свой ключ `~/.ssh/helm_deploy_ed25519`, GitHub его принимает, и `/opt/helm/.git` принадлежит ему же. Под `root` тот же `git fetch` отвечает `Permission denied (publickey)`, и это легко принять за отозванный deploy-ключ — я на этом потерял время. Правильно: `sudo -u tacticum-deploy git fetch origin`. Bundle не нужен.
- [deploy] Деплой = **REBUILD** (`find src -name __pycache__ -delete; SEED=0 bash scripts/deploy.sh`). **volume-mount НЕнадёжен** (код не подхватывался, давал регрессии) — только rebuild.
- [deploy] **ВСЕГДА после деплоя verify getsource** нового кода из контейнера (`docker exec helm-helm-1 /app/.venv/bin/python -c 'import inspect; ...'`) — убедиться, что загрузился свежий код, не старый pyc.
- [deploy] Прод обслуживает живых аналитиков (MCP роздан) — rebuild = краткий рестарт, деплой только с подтверждения пользователя.
- [gate] Прод-классификатор режет: загрузку секретов, включение фича-флагов, exec/чтение прод-контейнера и **живые прогоны (A/B) на проде** без явной авторизации человека. Не обходить (base64 и т.п.) — STOP и спросить. Бинарные upload режет — использовать текстовые.
- [publish] Push только текущую feature-ветку явно (`git push origin <branch>`); PR не создавать (готовить материалы); без AI-подписей; не в `main`. Перед push показать `git log/diff origin/main..HEAD`, проверить что нет `.env`/секретов/мусора.

---

## Как деплоить helm — порядок, проверенный на живом 2026-07-30

Выполнено целиком при выкатке `iva-write` (`b7341ab`): данные целы, простоя не было. Команды
даны как есть; всё идёт через ssh-manager (`server helm`), сервер — `159.194.233.33`.

### ⚠️ Главное: `bash scripts/deploy.sh` с настройками по умолчанию СНОСИТ БОЕВОЙ ГРАФ

`SEED` по умолчанию **`1`**, источник по умолчанию **`synthetic`** (`scripts/deploy.sh:23-24`),
а `seed_db.py` вызывает `clear_graph(tenant)` с `clear=True` — тенант-скопнутый `DELETE` по
`person`, `person_email`, `signal`, `initiative`, `block`, `goal`, `sales_initiative`,
`sales_link`, `assignment`, `dependency`, `block_team`, — и заливает синтетику.
**Условия по источнику нет: чистит при любом.** README предлагает запускать именно так.

Либо `SEED=0 bash scripts/deploy.sh`, либо шаги руками (ниже). Бэкап всё равно первым.

### Шаг 0. Бэкап БД — в репозитории его НЕТ ВООБЩЕ

Ни скрипта, ни шага в деплое. Поэтому руками, и это не «на всякий случай»:

```
docker compose -f docker-compose.prod.yml exec -T postgres \
  pg_dump -U helm helm | gzip > /opt/helm/_run/pre-<тема>-$(date +%Y%m%d-%H%M).sql.gz
```

Ориентир размера: ~19 МБ на 30.07. Заодно запомнить точку отката: `git rev-parse HEAD`.

### Шаг 1. Снять состояние ДО

```
cd /opt/helm && git log --oneline -1 && git status -sb
docker compose -f docker-compose.prod.yml ps
docker compose -f docker-compose.prod.yml exec -T helm uv run --no-sync alembic current
```

`git status` смотреть обязательно: на проде бывают незакоммиченные правки (30.07 нашлась
правка прав у `scripts/helm_daily_refresh.sh`). Бэкапить их до обновления.

### Шаг 2. Код — от `tacticum-deploy`, НЕ от root

```
cd /opt/helm && sudo -u tacticum-deploy git fetch origin
sudo -u tacticum-deploy git merge --ff-only origin/main
```

`--ff-only` намеренно: untracked не трогает, слияний на проде не создаёт. `reset --hard` не
применять.

### Шаг 3. Пересборка. Только rebuild, не volume-mount

```
find src -name __pycache__ -delete
docker compose -f docker-compose.prod.yml up -d --build
```

**Если в compose появился сервис с внешним образом** — сначала забрать его отдельно и посмотреть,
что забрали, а не узнавать это из уже поднятого контейнера:

```
docker compose -f docker-compose.prod.yml pull <сервис>
docker image inspect <образ> --format 'создан: {{.Created}} | размер: {{.Size}}'
```

Почему отдельным шагом: `up` скачает образ сам и молча, и различить «взяли ожидаемое» от «взяли
что-то другое» уже нельзя. При пине по версии (`ghcr.io/…:0.23.0`) риск мал, при `latest` —
нет. На выкатке 30.07 этот шаг пропущен: образ `mcp-atlassian:0.23.0` скачал `compose`, проверка
шла уже после подъёма. Обошлось из-за пина, но порядок должен быть обратным.

Что при этом происходит: `helm` пересоздаётся, новые сервисы создаются, а **`postgres` и
`traefik` не пересоздаются**, если их секции в compose не менялись, — проверено.

### Шаг 4. Миграции — РУКАМИ, без сида

```
docker compose -f docker-compose.prod.yml exec -T helm uv run --no-sync alembic upgrade head
docker compose -f docker-compose.prod.yml exec -T helm uv run --no-sync alembic current
```

⚠️ **Если прод отстал от `main`, `upgrade head` протянет и чужие линии тоже** — 30.07 БД стояла
на `pmb216` (PM-бот), и накат прошёл по обеим линиям сразу. Поэтому `alembic current` на шаге 1
не пропускать: иначе неизвестно, что именно поедет.

⚠️ **В репозитории исторически бывает несколько голов.** Слияние двух фича-веток даёт две головы,
git об этом НЕ предупреждает (файлы миграций разные), а `upgrade head` падает уже на проде.
Лечится merge-ревизией (образец — `alembic/versions/mrg350_iva_write_pmb.py`).

### Шаг 5. Verify — иначе деплой не считается сделанным

1. **Существующее важнее нового.** `/api/auth/config` → 200; `/mcp/analyst`, `/mcp/process` →
   406 (норма без нужного `Accept`); `/mcp/hrd` → 403 (свой гейт). Прод обслуживает живых
   аналитиков.
2. **Ошибки в логе:** `docker compose ... logs helm --since 3m | grep -iE "error|traceback|exception"`
   плюс наличие `Application startup complete`.
3. **Данные целы** — счётчики после наката сравнить с ожидаемыми (30.07: person 1008,
   person_email 1009, signal 46285, initiative 589, sales_initiative 318). Это и есть проверка,
   что сид не отработал.
4. **Свежий ли код в контейнере** — `getsource` нового символа изнутри, иначе можно смотреть на
   старый `pyc`.

### Откат

`checkout` прежнего sha → `up -d --build helm` → снять добавленные сервисы. Миграции откатывать
НЕ надо: старый код о новых таблицах не знает. `downgrade` через merge-point не применять —
ручной выбор ревизии в аварии, плюс `drop` таблиц сотрёт уже выданные данные.

## Relations
- relates_to [[team-lead-playbook]]
- relates_to [[session-state]]
- part_of [[20-Architecture]]


## Рабочий контракт публикации (роли, установлено пользователем 2026-07-17)
Разделение обязанностей на линии доработки RAG#2 (и далее):
- **Пользователь мерджит PR в main** через веб — единственный, кто мерджит. Merge = его осознанное действие.
- **Лид (я) держит git чистым и отвечает за синхронизацию** трёх точек: локальный `main` = `origin/main` = прод `/opt/helm`. Сверяю до и после каждого мержа.
- **Каждый трек — в своём worktree-ветке от `main`**, дерево чистое (без мусора/секретов), salvage-ветки не трогаю.
- **Сигнал «готово к мержу» даю сам**, только когда трек прошёл ревью + тесты (для Р-2 — ещё измеренный A/B). Не раньше.
- **Перед пушем показываю пользователю, что уйдёт:** `git status`, `git log origin/main..HEAD --oneline`, `git diff --stat origin/main..HEAD`. Пушу feature-ветку явно (`git push origin <branch>`) с ОК пользователя. Никогда голый push / push в main / force.
- **PR готовлю материалами** (заголовок + описание: что сделано, что тестировалось, метрики + ссылка `github.com/TacticumApps/helm/pull/new/<branch>`), сам PR не создаю (gh нет). Без AI-подписей.
- **После мержа пользователя — деплой на прод** (bundle→ff-merge→rebuild `SEED=0 deploy.sh`→verify getsource) с его ОК на прод-деплой.

Текущие несмерженные ветки в работе (все от main, в worktree, чисто): `feat/rag2-confidence-threshold` (Р-2a), `feat/rag2-api-registry` (Р-1), `fix/rag2-cross-rerank-helm-tune` (готов, ждёт A/B), `docs/iva-analyst-concepts`, `salvage/*` (архив вне RAG#2).


## Контроль-гейт: прод-валидация на реальных данных (обязательна перед деплоем)
Установлено пользователем 2026-07-17 («веди контроль, может фичи/тесты уходят в сторону, не хватает теста в проде»). Подтверждено дважды на этой сессии.

- **Юнит-тесты на фикстурах ЗЕЛЁНЫЕ ≠ фича работает в проде.** Каждая фича доказывается прогоном на РЕАЛЬНЫХ данных (реальный Qdrant/Meili/спеки), не только фикстура, ПЕРЕД включением в деплой-пачку. Это гейт лида.
- **Пример 1 (baseline):** 785 rag2-тестов зелёные, а retrieval@10 ≈ 0.11 на реальных данных; корень (retrieval-miss) виден только прогоном за гейтом, не юнитами.
- **Пример 2 (Р-1 api_registry):** 1488 тестов зелёные, оба приёмочных кейса ТЗ зелёные НА ФИКСТУРЕ (6 операций). Прогон на реальных 419 операциях: `check("отозвать письмо")` → FOUND (checkEmail/resendEmail, score 4.0) вместо not_found — приёмка ТЗ провалена, т.к. на фикстуре не было email-операций. Плюс порог 2.0 низок (ложные на messageSync/mailboxFind), дубли операций. → Р-1 смержен, но НЕ деплоить до фикса.
- **Метод прод-валидации:** детерминированные/чистые модули (матчер Р-1) — прогон локально на скачанных реальных данных (спеки `helm:/tmp/api_specs`). Retrieval/rerank — sidecar/`docker exec` в helm-helm-1 read-only (стор+ключи внутри контейнера, наружу не тащить). Golden-приёмка — на реальном корпусе, не фикстуре.
- **Следствие для команды:** воркерам явно требовать приёмку на реальных данных, не только юнит. Лид сам прогоняет ключевую валидацию перед сигналом на мерж/деплой.


## ⚠️ ИСПРАВЛЕНИЕ 2026-07-17: прод-git к GitHub РАБОТАЕТ (bundle устарел)
Прежний факт «сервер helm без git-доступа к GitHub → только git-bundle» — **УСТАРЕЛ**. Проверено по reflog `/opt/helm` и процессам:
- Прод `/opt/helm` remote = `git@github.com:TacticumApps/helm.git` (SSH), **deploy-key настроен, `git pull --ff-only origin main` работает**.
- **Есть авто-CD:** после мержа PR в main внешний актор заходит по SSH как `tacticum-deploy` (@notty non-interactive) и делает `git pull --ff-only origin main` на `/opt/helm` (reflog: pull'ы совпадают с мержами PR по времени). Механизм НЕ в cron/systemd/deploy.sh — вероятно GitHub Actions CD или внешний deploy-скрипт (автор коммитов Diaret).
- **CD делает ТОЛЬКО git pull, НЕ rebuild** → git-дерево `/opt/helm` подтягивается само после мержа, но контейнер `helm-helm-1` работает на СТАРОМ образе до ручного `SEED=0 bash scripts/deploy.sh`. Git-дерево и образ расходятся, пока не сделан rebuild.

**Новый процесс деплоя (проще bundle):** после мержа PR прод подтянет код сам (или `ssh helm 'cd /opt/helm && git pull --ff-only origin main'`) → **ssh rebuild `SEED=0 bash scripts/deploy.sh`** → **verify getsource** (всегда — pull обновляет git, но образ только rebuild) → curl `/mcp/analyst` (406). git-bundle больше не нужен, но и не вредит (thin-bundle тоже довозит). Всегда сверять прод-HEAD ПЕРЕД деплоем — он мог уехать вперёд авто-pull'ом.


## GUARDRAIL чистки данных (2026-07-17, указание пользователя)
Чистка ложных путей/дампов на helm (прод+локально) — **особо аккуратно, отдельным осознанным шагом, ПОСЛЕ показа пользователю**, НЕ в потоке работы.
- **Нельзя задеть работающее:** RAG#1, RAG#2 (Qdrant `iva_*`/`knowledge`/`helm_*` коллекции + Meili), дэш, MCP-сервер (`helm-analyst`, `iva-mcp`). Рабочие корпуса Qdrant/Meili — НЕ трогать.
- Порядок: сначала карта данных (explorer, read-only) → показать пользователю что есть → он решает → только потом удаление, с явным списком.
- Р-5 пересборка (наполнение EpicTask/`helm_mgmt` из iva_jira) должна писать ТОЛЬКО в effort_hint-корпус, НЕ перезаписывать рабочие `iva_jira`/рабочие коллекции.
- **Приоритет сейчас:** доделать ТЗ (Р-5) + перепроверка (широкий перемер дотяжки/floor). Чистка — отложена до решения пользователя.