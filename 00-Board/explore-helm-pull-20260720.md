---
title: explore-helm-pull-20260720
type: report
permalink: tacticum/00-board/explore-helm-pull-20260720
tags:
- explore
- helm
- git
- draft
---

# explore-helm-pull-20260720

status: draft
role: explorer (read-only + git pull ff-only)

## Репо
- Путь: `/Users/bubblemac/tacticum/helm`
- Remote: `git@github.com:TacticumApps/helm.git`
- Ветка: `main`, дерево **чистое**
- Pull: **fast-forward** выполнен (`git pull --ff-only origin main`).

## Было -> стало
- ДО: `c036842` (Merge PR #80, Шульга, 2026-07-18)
- ПОСЛЕ: `eaf10f8` (Diaret, 2026-07-19)
- Диапазон `c036842..eaf10f8` = **20 коммитов**, 77 files, +8519 / -1582.

## УТОЧНЕНИЕ: пользователь = Шульга (не betterbubble)
Пользователь коммитит под ДВУМЯ git-идентичностями:
- `Александр Шульга <aleksandr-shulga-0507@yandex.ru>` — 333 коммита (основной, feat/fix)
- `Шульга Александр Алексеевич <sasha_shulga0507@icloud.com>` — 67 коммитов (в осн. merge PR через GitHub UI)
- Итого ~400 коммитов Шульги в истории репо. Последний до pull — `c036842` (Merge PR #80).
- `betterbubble` в истории репо НЕ встречается вовсе — ложная зацепка.

## Авторы прилетевшего (20 коммитов)
- **Diaret <Diaret@users.noreply.github.com>: все 20.** Diaret = руководитель пользователя.
- **Шульги среди прилетевших НЕТ** (0 коммитов, в т.ч. нет merge-коммитов Шульги в диапазоне).

## Что менял Diaret (обобщённо)
- `web/src` (30) + `src/helm` (31) — редизайн req-card в маршрутизируемое «досье», редизайн backlog-дашборда.
- Allure TestOps: снапшот покрытия, requirement test coverage tool, секция тестов.
- Ingest: product area + sales matrix, cross-product seeding, fuzzy sales matching, инкрементальная экстракция Confluence, RAG2 refresh.
- `alembic/versions` — миграция схемы B1; `scripts/`, `docker-compose.prod.yml`, systemd — helm daily refresh.
- `docs/` — design-заметки; тесты ingest/interface/infrastructure/domain/application.

## Актуальность СЕЙЧАС (после повторного fetch)
- `HEAD` == `origin/main` == `eaf10f8` — **MATCH**. behind=0, ahead=0. Дерево чистое.
- Локальный репо полностью актуален, pull реально применён.
