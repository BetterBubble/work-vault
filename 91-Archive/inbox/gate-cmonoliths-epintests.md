---
title: gate-cmonoliths-epintests
type: note
permalink: tacticum/00-board/gate-cmonoliths-epintests-1
tags:
- gate
- controller
- us4
- tz3
- verdict
archived-at: 2026-08-03 11:16
---

# Гейт-контроль: ветки C-монолиты + E-kmp-pintests (ТЗ#3 US#4)

Контролёр (read-only). Ничего не правил. База: origin/main = 5884bcd (зелёный).

## Вердикт

| Ветка | GO/NO-GO | не-DB тесты | падения (классиф.) | скоуп | git |
|---|---|---|---|---|---|
| feat/us4-passC-monoliths @433ec95 | **GO** | 100% зелёные (549 pass; parity/presets/schemas/mail/dev_base = 247 pass; mirror 64/6 OK; discipline 48 clean) | 2 failed + 120 errors = ВСЁ DB-инфра (docker/PG :5432 недоступны). 0 регрессий | чист: ровно 6 файлов mail+rn | чист |
| feat/us4-passE-kmp-pintests @8b0e3c9 | **GO** | 100% зелёные (549 pass; non-DB targeted 213 pass; mirror 64/6 OK; discipline 48 clean) | 2 failed + 120 errors = ВСЁ DB-инфра. 0 регрессий | чист: ровно 4 файла kmp pin/tests; brd+start-task НЕ тронуты | чист |

## Классификация падений (общая для обеих веток, урок PR-A применён)
- **120 errors** — фикстура `tests/conftest.py:35 postgres_url` → `docker run postgres:16-alpine` exit 125 (docker/PG отсутствуют). DB-инфра, НЕ регрессия.
- **2 failed** (`test_admin_catalog_authoring::test_patch_profile_404`, `::test_create_draft_404_unknown_profile`) — прямой async-connect к :5432 (Errno 61). Проверено на pristine origin/main (5884bcd) — падают ИДЕНТИЧНО. Pre-existing DB-инфра, НЕ введены ветками.
- Не-DB наборы (parity/coverage/schemas/role_presets/profile/mirror/discipline) — 100% зелёные на обеих.

## Ветка 1 (C-монолиты) — детали
- Дифф origin/main..HEAD = ровно 6 файлов: mail+rn {start-task, manifest, CHANGELOG}. Canon/brd/pin/tests/композиты/web/kmp НЕ тронуты.
- **Fidelity by identity ПОДТВЕРЖДЁН:** mail start-task == rn start-task (байт-в-байт). Оба vs канон tacticum-dev-base (origin/main) отличаются РОВНО на одной строке 73 — легитимный стек-скилл `pin-ui-pipeline-check` добавлен в список design-скиллов. Весь гейт-блок К-3 (plate-based D-n) + К-5 (fr_skeleton) — байт-в-байт == канон.
- Версии: mail 0.7.3→0.7.4, rn 0.5.3→0.5.4 (манифест + топ CHANGELOG консистентны, discipline OK).
- НИТ (не-блокер): тело коммита описывает бамп «0.7.2→0.7.3 / 0.5.2→0.5.3 относительно базы C-канон» — расходится с фактическим net-vs-main (0.7.3→0.7.4 / 0.5.3→0.5.4). Артефакты и discipline-скрипт корректны; косметика нарратива коммита.

## Ветка 2 (E-kmp-pintests) — детали
- Дифф = ровно 4 файла iva-kmp-brownfield: pin-authoring/SKILL, tests-authoring/SKILL, manifest, CHANGELOG.
- **Критично ПОДТВЕРЖДЕНО:** brd И start-task НЕ в диффе (отдельный diverged-проход не затронут).
- Контент аддитивный: К-2/К-4 pin (проектные серии CT-n/DM-n/EV-n под KMP, статус-словарь реализован/расхождение/blocked, уважает К-3), К-2 tests (Covers: CT-n, расхождение=xfail, blocked=@Ignore).
- tests-токен консистентен эталону B2 (iva-brownfield-mail): `Covers: CT-n` совпадает; словарь covered/расхождение/blocked присутствует (kmp опускает `partial` — выбор профиля, не рассинхрон токена).
- Версия kmp 0.5.0→0.5.1 (манифест + CHANGELOG консистентны, discipline OK). Coverage/parity не сломаны.

## Git-чистота (обе ветки)
- 0 AI-подписей в телах коммитов, 0 секретов/ключей/.env в добавленных строках, 0 мусора (__pycache__/.DS_Store/.serena/worktree). Автор — Александр Шульга (человек). Ветки feature, не main.
- Прим.: в shared-main worktree висит untracked `docs/adr/0060-...md` — вне ревьюируемых веток (грязь основного дерева), в коммиты веток не входит.

## Итог
Обе ветки — **GO**: оба гардрейла PASS, не-DB набор 100% зелёный, все падения = DB-инфра, скоуп чист, git чист.