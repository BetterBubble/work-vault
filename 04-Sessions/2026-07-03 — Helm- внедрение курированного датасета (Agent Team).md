---
title: '2026-07-03 — Helm: внедрение курированного датасета (Agent Team)'
type: report
permalink: tacticum/04-sessions/2026-07-03-helm-vnedrenie-kurirovannogo-dataseta-agent-team
tags:
- helm
- control-tower
- data
- wave-1a
- agent-team
- session
---

Внедрён курированный обезличенный датасет в конвейер Helm (Волна 1a), силами Agent Team. Продолжение [[Helm — внедрение данных и drop-zone конвенция]] и [[2026-07-03 — Helm backend Волны 1a собран]].

## Итог — ПРИНЯТО (E2E ACCEPT)

- [outcome] S1 CSV-адаптер + S2 резолюция/сид + S3 genesis-мержи внедрены; E2E зелёный: `seed_db --source csv` → 13 инициатив без FK, `/api/gaps` 4 разрыва (`work_without_goal=[CS-401,BE-501]`), `/api/gantt` goal:G2 «ГОСТ» deadline 2026-10-01. Мержи ГОСТ (G2∪S1∪MAIL-200) и One 1.5 (G5∪S2∪ONE-100) схлопнули 17→13; дедлайн/важность живы (G2=8.75M, G5=4.8M). 154 теста, ruff+mypy чисто. Ветка `wave-1a-backend`, локально, не запушено. #verified
- [decision] Добавлен 4-й операционный разрыв §6.1 `work_without_goal` (Jira-сигналы с initiative_id=None — задачи вне целей); в датасете BE-501/CS-401 заложены под это. #calc

## Ключевой урок — FK-баги, невидимые in-memory

- [bug] Два FK-блокера персистентности, которые in-memory тесты (build_graph/build_portfolio) НЕ ловят — FK enforce'ится только на Postgres (в SQLite-тестах нужен `PRAGMA foreign_keys=ON`): (1) `sales_depends_on.initiative_id` — сырой эпик-ключ без namespace + поглощённый merge-таргет; (2) `dependency.from_initiative` — поглощённый merge `sales:S1`. #persistence
- [lesson] Любое ребро графа при персистентности → резолвить ОБА конца через `ref_to_id` (genesis_ref+merged_from→финальный id) И снимать самопетли (from==to после merge). Проверять на РЕАЛЬНОМ Postgres, не только in-memory — независимый verifier обязателен. #lesson

## Follow-ups (не сделано)

- [followup] `promises_without_work` пере-флагает S3/S4/S7: их delivery привязан через `depends_on`-эпик, но наша gap-логика кредитует работу только через merge/decomposes_to/genesis, не через depends_on. Кандидат на уточнение семантики (§6.1). #followup
- [followup] Разрыв «работа без денег» (§6.1): признак = `compute_deadline is None`, скоуп на goal-genesis (иначе шумно). Лёгкий 1a, не реализован. #followup
- [reference] 1b-готовность `data/repos` (реальный git ИВА): хуки RawGitCommit + `ingest/git_source.py` + build_graph `raw_commits` → M1/M2(Т1 commit→task)/M3(work→product). Блокеры: индекс `issue_key→initiative_id` отсутствует (ref_to_id держит только genesis-refs); реальные Jira-проекты IVAONE/VCSWEB ≠ курированные ONE/MAIL; полный MR API нужен GitLab PAT. Объём M1+M2-скелет ≈ 1 PR масштаба CSV-адаптера. #wave-1b

## Процесс (Agent Team)

- [process] Тимлид + @explorer + @implementer + @implementer-2 + @verifier, все Opus/High. Воркеры в git-worktree, лид cherry-pick'ает в основную ветку + пишет каноническую память. Независимый @verifier поймал ОБА FK-бага, которые прошли зелёные unit-тесты. Координация — task-board + SendMessage. #agent-team

## Отношения
- relates_to [[Helm — внедрение данных и drop-zone конвенция]]
- relates_to [[agent-team-design]]
- implements [[control-tower-v02]]
