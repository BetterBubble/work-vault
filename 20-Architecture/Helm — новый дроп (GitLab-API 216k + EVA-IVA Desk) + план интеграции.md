---
title: Helm — новый дроп (GitLab-API 216k + EVA/IVA Desk) + план интеграции
type: plan
permalink: tacticum/20-architecture/helm-novyi-drop-git-lab-api-216k-eva-iva-desk-plan-integratsii
tags:
- helm
- control-tower
- data
- eva
- git
- wave-1b
- integration
- plan
---

Крупный дроп 2026-07-04 (оператор), разложен в `data/real/`. Закрывает 3 из 5 нехваток аудита. Продолжает [[Helm — аудит данных data/real: что есть на входе + что деривится без дозапросов]].

## Что приехало

- [reference] **`data/real/git/` (GitLab API, авторитет):** `all_commits.csv` (**216 161 коммит**, 233 репо, полная история, 35% keyed, 30 316 distinct issue-ключей) · `merge_requests_all.csv` (**1115 MR**: repo·!NNN·state·author·title·target_branch; 746 с jira-ключом в title; merged860/opened169/closed86) · `repo_activity_all.csv` (233 репо: commit_count·contributors·branches·mr_open/merged/total·top_group; продукт/generation заполнены только у кураторских, 223 пусто). Старые `commits.csv`/`merges.csv` (24k из локальных клонов) — суперсед. #data
- [reference] **`data/real/eva/` (EVA/IVA Desk, CMF, был «доступ не получен»):** `eva_tasks.csv` (**6 634 задачи**: code·project·type·status·**status_type** OPEN/IN_PROGRESS/IN_REVIEW/CLOSED·responsible(+login)·author(+login)·priority·SP·**epic**(29%)·parent(45%)·deadline(6%)·plan_start/end(12%)·tags·name) · `eva_projects.csv` (25: Messenger2.0 2507·ms 616·**Largo 1840**·iva-one 152·Aves 64·**operations-directorate=Операционный комитет**·new-desktop-qt 455…) · `eva_task_links.csv` (**2 332** depends_on/affects, §5.4). Аккаунт видит ВСЮ организацию (не урезан, в отличие от Jira-PAT). #data
- [outcome] **Overlap (Т1 оживёт):** commit-keys ∩ jira-задачи=788 + ∩ eva-задачи=769 ≈ **1557** (было 148). EVA logins (`*@msk.iva-tech.ru`/`*@hi-tech.org`) сшиваются с identity. #data

## План интеграции (батч)

- [plan] **WP-A (EVA source) — @implementer-1:** новый `ingest/eva_source.py`: eva_tasks→сигналы+инициативы (epic/operations-directorate genesis, §3), eva_task_links→зависимости (§5.4), responsible_login/author_login→identity. status_type→светофор, deadline+plan_*→Гантт. Покрывает Messenger/Largo/Aves. Владеет правками `real_source` (вшивает EVA + git-хуки от WP-B). #wave-1b
- [plan] **WP-B (git-API) — @implementer-2:** RealGitAdapter→`all_commits.csv` (216k вместо 24k) → Т1-ключи взрываются; новый `ingest/mr_source.py` из `merge_requests_all` (реальный PR/MR: state/author/title, jira-ключи из title)→PR/MR-сигналы+Т1; M3 `re_enrich` repo→product на 233 репо из `repo_activity_all` (кураторские заполнены, 223 — эвристика top_group/name). Отдаёт хуки в real_source (не трогает real_source сам — владелец impl-1). #wave-1b
- [plan] **WP-C (re-seed + сверка) — тимлид:** seed на всех источниках → Т1 привязано (должно вырасти с 0), EVA-покрытие, MR-счёт; check_competency; аудит точности повторно. #wave-1b
- [followup] После: registry на 233 репо (223 без продукта — ручная/эвристич. классификация); reviewers/approvals MR — API не отдал (по запросу). Остаётся из нехваток: зарплата-в-рублях, тела Confluence. #wave-1b

## Отношения
- relates_to [[Helm — аудит данных data/real: что есть на входе + что деривится без дозапросов]]
- implements [[control-tower-v02]]


## Интеграция ЗАВЕРШЕНА (2026-07-04, тимлид-сессия)

- [outcome] **Дроп влит и развёрнут на сервере `helm` (docker-compose.prod, seed --with-git).** Ветка `wave-1a-backend`, 5 коммитов (WP-A EVA, WP-B×3 git-API, WP-C сшивка) — локально закоммичено, **НЕ запушено** (пуш ask-gated, оператор отошёл). 429 тестов зелёные, ruff+mypy strict чисто, golden-set держит **23/30** (трим не срезал возможности). #done
- [outcome] **EVA (2-й трекер):** eva_tasks 6634 → 6475 сигналов (source=eva; привязано к инициативам 1789, висит 4686), epic-инициатив jira+eva 258, eva_task_links 2332 → §5.4-зависимости. Логины через identity_map. #eva
- [outcome] **git-API all_commits 216161** (233 репо, полные пути `iva/core/id/server`) заменил клон-снимок commits.csv (24k). Реальные merge_requests_all 1115 через `mr_source` вместо merge-коммитов. #git
- [outcome] **M3 repo→product ожил: покрытие 0 → 74509 коммитов** (51 репо, curated манифест по basename + консервативная эвристика, `unknown` не выдумывается — `build_full_repo_product_map`). **M5-скоринг раскрылся на полный корпус: ≈2800 человек** (было ~196): квадранты star 1053 / underloaded 1732, heatmap 7 команд, internal-трек 141. #m3 #m5
- [decision] **Т1 commit→task привязка = 9** (спарс). ПРИЧИНА — лимит данных, НЕ код: коммиты ссылаются на issue-уровень (SWL 16k, VCSWEB2, VCSDESK, KEYCLOAK, IVAONE 7159…), а загружены только ~100 IVAONE-**эпиков** + EVA. Прямой джойн key∩task = 9. Ценность git — в M3 product-level, не в issue-джойне. #gotcha
- [decision] **Честность разрыва §6.1 «работа без цели»: 223210 → 7600.** Сузил метрику до ТРЕКЕРОВ ЗАДАЧ (jira/eva): несопоставленная задача = governance-разрыв; сырой коммит без тикета — рутина, не разрыв (иначе 216k коммитов подряд = шум для ГД). `calc._WORK_GAP_SOURCES = {jira, eva}`. #decision
- [decision] **Трим git-сигналов графа: 239178 → 21911.** git-сигнал персистится ТОЛЬКО при резолве в инициативу (Т1/M1-ребро). 216k несопоставленных коммитов остаются в `raw_commits` для M5-скоринга (scoring.compute_contributions читает commit.repo напрямую) + субстрат-проекции, но НЕ грузят read-only-граф orphan-узлами на каждый /api/portfolio. `loader._build_git_signals`/`_build_merge_signals`: `if target is None: continue`. #decision #perf
- [outcome] **Сервер проверен:** `/api/portfolio` → 200, 9 блоков, 589 инициатив (🟡278 🟢251 🔴60); `/api/gaps` живой. Сид 1:1 с локальным (сигналы 21911, EVA 6475, M3 74509, deps 37, sales 316). #server
- [reference] Деплой git-данных: залил ТОЛЬКО мелкие CSV нужные для with-git (all_commits 30M, merge_requests_all, repo_activity_all + eva/*), НЕ весь `real/git` 552M (repomix-снимки не нужны для базового seed — M3 берёт activity+manifest). Полный src-тарбол обязателен (образ не имел новых модулей eva_source/mr_source/re_enrich). #howto

- relates_to [[Helm — runbook работы с dev-сервером (SFTP-доставка, не rsync)]]
