---
title: 'Доступы: серверы и репозитории ИВА (adp / teststand / git.hi-tech.org)'
type: reference
status: current
created: 2026-07-24 16:42
updated: 2026-07-25 20:30
permalink: tacticum/20-architecture/dostupy-servery-i-repozitorii-iva-adp-teststand-git.hi-tech.org
project: tacticum-dev
tags:
- reference
- infra
- access
- servers
- repos
- gitlab
- adp
- teststand
- read-only
---

# Доступы: серверы и репозитории ИВА

Карта, где физически лежит продуктовый код ИВА и как мы к нему ходим. **ГЛАВНОЕ ПРАВИЛО (решение президента 2026-07-24): все серверы READ-ONLY — состояние НЕ менять.** Любой git push / мерж / деплой / изменение состояния сервера — только через директора + одобрение президента. Работа с кодом — в локальных worktree.

## Серверы (в ssh-manager)
| Имя | IP | Что это |
|---|---|---|
| `adp_emb` | 194.36.208.242 (root) | VPN-прокси в контур ИВА + **полный мирор репо** ИВА. `git.hi-tech.org` резолвится тут во внутренний `10.0.207.3`. |
| `teststand` | 38.180.236.39 (root) | **Пилот-окружение уровня-3** (гайд Глеба): тулчейн codex/node/serena-LSP + VPN до контура + пилот-клоны. |
| `helm` | 159.194.233.33 | RAG#1 docs-бот (helm.tacticum.ru). |
| `gateway` | 155.212.134.20 | LLM Gateway (litellm). |
| `tacticum_prod` | 159.194.224.59 | Прод catalog-mcp (каталог профилей, dev.tacticum.dev / mcp.tacticum.dev, БД tacticum_catalog). |

## Где продуктовый код (мирор на adp_emb: `/srv/iva/repos/`, ~220 репо, юзер `tacticum`)
- **iva-one** (Web One, Angular/TS) — `git.hi-tech.org/iva/one/web/iva-one` — отв. Савицкий Максим. Путь: `/srv/iva/repos/iva-one`.
- **su.ivcs.messenger** (One Android, Compose) — `git.hi-tech.org/iva-m/android/su.ivcs.messenger` — отв. Легин Денис. = цель переноса форм one→kmp.
- **kmp** (Kotlin Multiplatform SHARED-модуль) — `git.hi-tech.org/iva-m/android/kmp`. Путь: `/srv/iva/repos/kmp`. **Содержит `AI common/skills/` — 40 репо-нативных навыков команды** (android-to-kmp-porting, decompose, mvi-state-machine, compose-ui-patterns, design-system-discovery, **lite-task-workflow** r.yarullin и др.). 49 Iva* в commonMain — авторитетный набор для словаря.
- Прочее: backend (users/chats/call/api-gateway…), ios (messenger/apple-rapido), desktop, iva-connect (отдельная ДС `iva-core`), diskstorage, notifications и т.д. Полный список — Excel Глеба «План индексаций» (`90-Materials/План индексаций.xlsx`): стек/URL/команда/ответственный.
- **`one-web-kmp` как отдельного репо НЕТ** (подтверждено) — есть web (iva-one) + android (su.ivcs.messenger) + shared (kmp).

## Модель доступа к git.hi-tech.org (важно)
- **Своей GitLab-веб-учётки/PAT у нас НЕТ.** Доступ обеспечивает **машинный SSH-deploy-ключ** на adp_emb: `~tacticum/.ssh/id_ed25519_iva` (`tacticum-vps-iva-gitlab`); `~tacticum/.gitconfig` переписывает https→ssh. Даёт clone/fetch по мирору. Git-операции — из-под юзера `tacticum` (`su - tacticum -c '…'`). Для push от имени / API — нужен отдельный PAT (нет).
- Публичный GitHub (`TacticumApps/*`) = наш профильный репо `tacticum-dev`; продуктовые репо ИВА там НЕ лежат (они в git.hi-tech.org).

## teststand — пилот-окружение
`/home/tacticum/`: `kmp-full` (=клон su.ivcs.messenger, был старый release26Q1 — НЕ авторитет для инвентаря, брать shared kmp с adp), `kmp-pilot`, `int-pilot`, `disk-pilot`, `iva-role-{go,ios,mail}-post`. Тулчейн codex/node/serena тут — сюда сажать уровень-3 живой пилот роли.

## Связано
[[session-state]] · [[napravlenie-edinaia-dizain-sistema-tacticum-figma-tokeny-kod]] · [[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]] · [[tekh-dolg-qa-zhivaia-sinkhronizatsiia-ivaqa-kit-git-lab-nash-qa-profil]]

## Инфо-фикс (2026-07-24): GitHub-репо профилей переехал
Наш профильный репо на GitHub: **`TacticumApps/dev`** (переименован из `TacticumApps/tacticum-dev`). Push проходит через редирект со старого URL; локальный `origin` обновить на новый — низкий приоритет (не критично, редирект работает). Локальный путь без изменений: `~/tacticum/tacticum-dev`.
