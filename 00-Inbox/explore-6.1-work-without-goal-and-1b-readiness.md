---
title: explore-6.1-work-without-goal-and-1b-readiness
type: report
permalink: tacticum/00-inbox/explore-6.1-work-without-goal-and-1b-readiness
tags:
- helm
- explore
- gaps
- 1b
---

# Разведка #13 — §6.1 «работа без цели/денег» + готовность 1b (data/repos)

Черновик разведчика (explorer). Каноническую запись делает лид.

## A. §6.1 — недостающие разрывы

Реализованы (calc.py + GapReport + GapReportOut): `goals_without_work`, `promises_without_work`, `urgent_important_without_owner`.
Канон §6.1 перечисляет ещё: **работа без цели**, **работа без денег/клиента**, **дубли** (semantic_dedup).

### A1. «Работа без цели» — ГОТОВО К РЕАЛИЗАЦИИ (задача #14)

Механизм уже есть в графе: `loader._issue_target_initiative` возвращает `None`, когда Jira-задача не резолвится ни по epic_key, ни по parent, ни по label (loader.py:206-215, коммент «разрыв работа без цели §6.1 — валидный, не ошибка»). Такой Signal кладётся в граф с `initiative_id=None` (loader.py:228-240). Детектор просто вычитывает эти сигналы.

Факт в датасете (jira_issues.csv): задачи без epic и без совпадающего label →
- **BE-501** (Bug, labels=backend;auth) — auth hardcoded domains
- **CS-401** (Story, labels=sorm) — SORM cert prep

Оба резолвятся в `None` (нет goal/epic с ключом backend/auth/sorm). Подтверждено по факту.

Мини-спека:
- calc.py (после promises_without_work):
  `def work_without_goal(signals: Sequence[Signal]) -> list[str]:` → `[s.external_id for s in signals if s.initiative_id is None]` (external_id = Jira-ключ, см. loader.py:230).
- portfolio.GapReport: добавить поле `work_without_goal: tuple[str, ...]`; в build_portfolio (portfolio.py:207) — `work_without_goal=tuple(calc.work_without_goal(signals))`.
- schemas.GapReportOut + gap_report_out (schemas.py:106-116): поле `work_without_goal: list[str]`.
- routers/gaps.py — правок НЕ нужно (отдаёт весь report).

Нюанс/ограничение (кандидат в 1b): эпик-genesis инициатива без родительской цели (напр. MCU-300 не привязан к Goal) — это разрыв «работа без цели» второго порядка, простой детектор его НЕ ловит (дети эпика резолвятся в эпик-инициативу, initiative_id != None). При желании — отдельный детектор «initiative genesis=jira_epic и не merged в goal».

### A2. «Работа без денег/клиента» — 1a-выполнимо, но со скоупингом

Данные УЖЕ есть — новых не нужно: `compute_deadline(...) is None` ⇔ нет привязанной продажи ⇔ «работа без денег» (calc.py:56-70, докстринг прямо называет это «работа без денег/клиента §6.1»). Эквивалент: `importance_full == 0`.

Черновой детектор: инициативы, где `not _bound_sales(id, sales, refs)`. НО без скоупинга шумно (все эпик-инициативы без сделок попадут). Канон: «цель есть, но за ней нет пайплайна» → скоупить на goal-genesis инициативы (или «есть работа/сигналы, но нет денег»).

`product_economics.csv` (revenue/margin/priority по продукту; CS/Largo — отрицательная маржа) — это M3-обогащение (work→product→экономика), не привязка пайплайна. → богатая версия «низкой ценности» = **1b**. Минимальный детектор «нет пайплайна» = 1a.

### A3. «Дубли» — semantic_dedup, требует эмбеддингов/LLM → вне 1a, ближе к 1b/2.

## B. Готовность 1b по data/repos/ (реальный git ИВА)

Источник: 8 репо ИВА, `git log --all` метаданные (кода нет), снято 2026-07-03, ≤60 коммитов/≤40 merge на репо. Конфиденциально (реальные @iva.ru).

Файлы и что дают:
- `commits.csv` (435 строк): `repo, hash, author_email, author_name, date, branch, jira_keys, subject` → **M1** (git-снимок) + **M2** (Т1 commit→task через jira_keys). Реальные ключи: IVAONE, VCSWEB, P8.
- `merges.csv` (145): + `mr_ref(!NNN), source_branch, target_branch` → **M1** (PR/MR прокси; только 46/145 несут !NNN) + зависимости.
- `repos_manifest.csv` (8): repo→product/generation → **M3** (work→product).

Хуки в нашем коде (зеркалят Jira-адаптер, слой ingest/):
- contract.py: новый `RawGitCommit` (mirror RawJiraIssue/RawEpic).
- ingest/git_source.py: `load_git_dataset(base)` читает commits/merges/manifest (csv_io уже глотает BOM utf-8-sig) → RawGitCommit + repo→product map.
- loader.build_graph: новый параметр `raw_commits` → `_build_git_signals` → `Signal(source="git", external_id=hash, entity_refs=(author_email,*jira_keys), initiative_id=resolve)` + `Assignment(author_email→initiative)`. Зеркало `_build_signals_and_assignments`.
- M3: manifest repo→product — для роллапа git-инициатив в блок / «активность по продукту».

Связки/поля: author_email = ключ идентичности (сшивка с teams/staff; видны внешние подрядчики @nwire.ru/@gmail.com — нужна карта идентичности). jira_keys = мост commit→task→initiative.

Блокеры:
1. **Т1 резолюция**: git jira_keys (IVAONE-12113) — это ключи ЗАДАЧ, не эпиков; резолвятся в инициативу только через индекс issue_key→initiative_id, которого сейчас нет (ref_to_id держит только genesis-refs: goal_id/epic_key/merged_from). Нужен индекс issue→initiative из Jira-сигналов. Без реального Jira-экспорта IVA (проекты IVAONE/VCSWEB ≠ курированные ONE/MAIL) git-ключи повиснут → массово «работа без цели».
2. **Полный MR API** (title/reviewers/status/approvals) — только через **GitLab PAT** (на VPS не найден). Сейчас MR = merge-коммиты (прокси).
3. Синтетический git 1a (`git_commits.csv`/`git_prs.csv` в Обезличенные данные) СЕЙЧАС НЕ грузится (csv_source читает только goals/blocks/teams/sales/jira) — подтверждает, что git-ingestion — реальный 1b-гэп.

Объём: M1+M2-скелет (commits→signals, идентичность, jira_keys→initiative через issue-индекс) ≈ 1 сфокусированный PR масштаба S1-адаптера. M3-роллап — небольшая добавка. Полный PR/MR-коннектор — отдельно, ждёт GitLab PAT.
