---
title: 'Helm — 1b git-ingestion: скелет готов + спека real-adapter/identity-map'
type: reference
permalink: tacticum/20-architecture/helm-1b-git-ingestion-skelet-gotov-speka-real-adapter-identity-map
tags:
- helm
- control-tower
- wave-1b
- git-ingestion
- identity-map
---

Статус и спека 1b git-ingestion. Продолжение [[2026-07-03 — Helm- внедрение курированного датасета (Agent Team)]]; запросы к IVA — [[Helm — что запросить у IVA (данные-доступы-ресурсы для 1a-1b)]].

## Сделано — скелет git-ingestion (M1/M2/M3) на синтетике

- [done] `#19`: `contract.RawGitCommit` (hash/repo/author_email/date/jira_keys/message/branch) + `GitSource` Protocol; `ingest/git_source.py` (SyntheticGitSource, `load_git_commits`, `extract_jira_keys` regex `[A-Z][A-Z0-9]+-\d+`, заглушка `GitAdapter`, `load_repo_product_map` M3); `loader.build_graph +raw_commits` → индекс `issue_key→initiative_id` (Т1) → `_build_git_signals` (Signal source="git") + Assignment(author→initiative). На курированном: 8 коммитов → 6 привязано / 2 висят (BE-501-коммит + chore/cleanup → work_without_goal). 175 тестов, ruff/mypy чисто. Реальный `data/repos` — заглушка. #wave-1b
- [note] git-коммит без разрешённого ключа → Signal.initiative_id=None → попадает в `work_without_goal` (§6.1 «коммитят, не привязано к цели») — /api/gaps обогащается git-evidence. #calc

## Спека follow-on (разведка #20)

### RealGitAdapter (реальный data/repos → RawGitCommit) — небольшой адаптер
- [spec] Две реализации `GitSource`: synthetic (jira-ключ ВНУТРИ `message` → regex) vs real (`repos/commits.csv`: готовая колонка `jira_keys` split; fallback-regex по subject/branch — в branch есть `feature/IVAONE-12066-…`). Поля 1:1 (hash/repo/author_email/date/branch); +author_name в реале. #git
- [spec] `merges.csv` → отдельный `RawGitMerge` (PR-прокси): `mr_ref`(!NNN, 46/145), `source_branch→target_branch` (ребро интеграции), jira_keys, статус «merged». Полный PR (title/reviewers/approvals) — НЕ здесь (блокер PAT). #git

### Identity-map слой — модель готова, нужна логика
- [spec] Опора: `teams.csv` несёт `repos`+`jira_projects` на человека; модель `Person` (JSON repos/jira_projects) + 1:N `PersonEmail` (email PK) — заложено без миграции. #identity
- [spec] `resolve_person(author_email)` через PersonEmail-lookup; seed из teams.csv. Реальные кейсы (из commits.csv ~40 authors): мульти-email одного человека (`a.rodionov@iva.ru`↔`@iva-tech.ru`), machine-local мусор (`*.local`), боты (`jenkins@ivcs.su`), внешние (`@gmail/@nwire.ru`). #identity
- [spec] Алиасы РАЗНЫХ доменов НЕ авто-мержить — эвристика по local-part + подтверждение оператора → доп. PersonEmail. Бакет external/unknown + отчёт «unresolved git authors» оператору (аналог разрыва). Фильтр не-людей (боты/*.local) до Person-загрузки. #identity

## Блокеры 1b
- [blocker] Т1-сшивка: git-проекты IVAONE/VCSWEB/P8 ≠ курированные ONE/MAIL → без **реального Jira-экспорта IVA** git-ключи повиснут (массовая «работа без цели»). #blocker
- [blocker] Алиасинг требует оператора (не полный автомат). #blocker
- [blocker] Полнота PR/MR — только GitLab PAT (на VPS нет). #blocker

## Следующий инкремент (рекомендация)
- [next] Identity-map резолвер + seed + unresolved-отчёт — НЕ блокирован (работает на teams + git-authors, модель готова). Лучший следующий шаг.
- [next] RealGitAdapter + RawGitMerge — код собирается, но осмысленная Т1-резолюция ждёт real Jira. Можно построить адаптер+PR-прокси-рёбра сейчас, резолв — потом.

## Отношения
- relates_to [[2026-07-03 — Helm- внедрение курированного датасета (Agent Team)]]
- relates_to [[Helm — что запросить у IVA (данные-доступы-ресурсы для 1a-1b)]]
- implements [[control-tower-v02]]

## Обновление 2026-07-03 — A и B реализованы (параллельно)

- [done] `#22` (A) Identity-map резолвер — чистый `ingest/identity.py`: `resolve_authors(author_emails, team_members) -> IdentityReport` (resolved / bot / machine_local / external + alias_candidates по local-part БЕЗ авто-мержа + unresolved-агрегат, §7). Wiring в loader/persist — СЛЕД (пока pure-модуль). #done
- [done] `#23` (B) `RealGitAdapter` + `RawGitMerge` — заглушка заменена рабочим адаптером реальной схемы `data/repos` (jira_keys из колонки + fallback-regex по ветке; merges.csv → RawGitMerge как PR-прокси). `loader.build_graph +raw_merges` → `_build_merge_signals` (Signal type="merge", резолв тем же индексом). Валидировано на РЕАЛЬНЫХ `data/repos`: 435 коммитов (85 с ключами), 145 merge (46 с mr_ref — как в README). #done
- [status] Всё интегрировано в `wave-1a-backend` (локально, не пушено): 191 тест зелёный, ruff+mypy чисто; курированный пайплайн без регресса (13 инициатив, 18 сигналов, git 8/6). RealGitAdapter в seed НЕ подключён (Т1-блокер: IVAONE/VCSWEB ≠ ONE/MAIL — нужен реальный Jira-экспорт). #status
- [next] Оставшиеся шаги 1b (все ждут внешнего): wiring identity-резолвера в loader/persist (можно сейчас) → рёбра merge source→target как Dependency → реальный Jira-экспорт разблокирует Т1 на data/repos → repomix для RE. #next
