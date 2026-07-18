---
title: 'Helm — прод: git/stash/владельцы (отложенный блокер деплоя)'
type: note
permalink: tacticum/03-decisions/helm-prod-git-stash-vladeltsy-otlozhennyi-bloker-deploia
tags:
- helm
- prod
- git
- deploy
- blocker
- deferred
- stash
- permissions
---

# Helm — прод: git/stash/владельцы (отложенный блокер деплоя)

Обнаружено 2026-07-08 при попытке деплоя через main. **Статус: ОТЛОЖЕНО — не в фокусе.** Чиним, когда будем правильно вливать Этап 1 (Conformance v2) и деплоить. Сейчас работаем над Этапом 1 (см. [[Helm — Conformance v2- 3-этапный план (многомерная матрица требований 1.0+1.5)]]).

## Симптом
- [context] Прод-хост: git в неконсистентном состоянии. `stash@{0}` держит **50 файлов / 5403 строки** незакоммиченного (Settings/ServiceDesk/EVA/MR). Закоммиченные миграции (напр. `c2d3e4f5a6b7`, она в main) показываются как **untracked, owner=UNKNOWN** (писались из контейнера под чужим UID). Есть root-owned `_front/`, `_incidents/`, `settings.py`. Из-за смешения root/deploy-юзера стэш прошёл частично, `pull` прервался.

## Главное: НИЧЕГО НЕ ПОТЕРЯНО (проверено по git)
- [decision] Прод-стэш **полностью избыточен** — всё его содержимое есть в git:
  - В **main** (PR #3/#4/#5): `settings.py`, `interface/api/jobs.py`, `sd_themes.py`, `llm/theme_label.py`, `Settings.tsx`, `ServiceDesk.tsx`.
  - На **wave-1b ветках**: `eva_source.py` (`e2ea9b1`, `migration/wave1b-eva`), `mr_source.py` (`ff1b909`, `migration/wave1b-gitapi`) + 9 сохранённых worktree'ов.
- [decision] **Проблема — права доступа (владельцы файлов), а не потерянный код.** Форс (reset/checkout/stash) НЕ нужен для спасения работы — нужен для выравнивания после починки прав.

## Безопасная последовательность (когда вернёмся)
- [howto] 1) Страховка: `git stash show -p 'stash@{0}' > ~/helm-prod-stash-2026-07-08.patch`, скопировать off-host.
- [howto] 2) Права (sudo, делает человек): `sudo chown -R <deploy-user>:<deploy-user> /opt/helm` + `git config --global --add safe.directory /opt/helm` → снимает «untracked owner=UNKNOWN».
- [howto] 3) Убрать junk `_front/` `_incidents/` (не в git).
- [howto] 4) После бэкапа+chown: `git fetch origin && git reset --hard origin/main` → `git stash drop 'stash@{0}'` (деструктивно, но всё в git — с явного go пользователя).
- [howto] 5) `alembic upgrade head` (подхватит `c2d3e4f5a6b7` + `cf1a2b3c4d5e` conformance-модель) → сид требований → рестарт сервисов.
- [reference] Деплой — только через main (ветка→коммит→PR→merge→заливка), см. runbook [[Helm — runbook работы с dev-сервером (SFTP-доставка, не rsync)]].

## Relations
- relates_to [[session-state]]
- relates_to [[Helm — wave-1b data-ingest отложен (не влит в main)]]
- relates_to [[Helm — runbook работы с dev-сервером (SFTP-доставка, не rsync)]]


---

## РЕШЕНО 2026-07-08 — Stage 1 задеплоен, механизм деплоя выяснен
- [done] Прод выровнен на GitHub main `38567c2` (Этап 1 conformance-v2 живой). `reset --hard` + `SEED=0 bash scripts/deploy.sh`. Проверено: DB-счётчики НЕ изменились (commit 231358, sd_request 43059, signal 21911, initiative 589, conformance_row 170, **requirement_assessment 1038** — руководитель засеял требования). Эндпоинт `/api/cio/conformance-v2` → 401 (маршрут есть, auth-гейт).
- [reference] **Механизм деплоя (выяснено):** прод-remote = `git@github.com` (SSH). Обновляют **интерактивно `git pull --ff-only` с проброшенным ssh-agent человека** (у руководителя ключ есть). Автоматическая ssh-manager сессия (root, alias `helm`) в GitHub НЕ аутентифицируется — у root нет github-ключа/known_hosts.
- [decision] **Обход для авто-деплоя:** `git bundle create` локально → `ssh_upload` на прод → `git -C /opt/helm fetch <bundle> main:refs/remotes/origin/main` → `reset --hard`. Не требует доступа прода к GitHub. Так и задеплоили Stage 1.
- [decision] Сгенерён **deploy-key на проде** (`/root/.ssh/id_ed25519`, pub `ssh-ed25519 AAAAC3...TI5O helm-prod-deploy-2026-07-08`). Чтобы прод тянул GitHub напрямую — **админ репо** добавляет pub как Deploy Key (read-only). У пользователя admin-прав на `TacticumApps/helm` нет — просить руководителя/владельца орг.
- [gotcha] **`deploy.sh` дефолт `SOURCE=synthetic`, `SEED=1` → пере-сид ЗАТИРАЕТ реальную БД. Деплоить строго `SEED=0 bash scripts/deploy.sh`** (только пересборка + `alembic upgrade head`).
- [reference] Страховки на хосте (можно чистить позже): ветка `rescue/prod-pre-conformance-2026-07-08`, дамп `/root/helm-db-2026-07-08.dump`, `stash@{0}`, патчи `/root/helm-*.patch`.
- [gotcha] После reset остались **untracked** на проде (`_front/ _incidents/ src/helm/ingest/eva_source.py mr_source.py`) — не в git, сборке не мешают; eva/mr — wave-1b, есть на своих ветках.