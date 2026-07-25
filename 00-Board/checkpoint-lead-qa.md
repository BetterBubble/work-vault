---
title: Чекпойнт — lead-qa (QA-профиль iva-role-qa)
type: report
status: current
role: lead-qa
created: 2026-07-24 13:10
updated: 2026-07-25 10:51
permalink: tacticum/00-board/checkpoint-lead-qa
tags:
- checkpoint
- lead-qa
- iva-role-qa
- session-state
---

# Чекпойнт lead-qa — регидратация свежей сессии

**Регистрация:** на старте `bash ~/.claude/hooks/role-register.sh lead-qa` → проверь сигналы `bash ~/.claude/hooks/signals-check.sh`.
**Репо:** `~/tacticum/tacticum-dev` (прод-каталог; деплой РУЧНОЙ по SSH — Serena-заметка `deployment_prod_catalog_mcp`).
**Полная картина:** досье [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]] + решения/итоги [[decisions-qa-codex-2026-07-24]]. Этот файл — быстрый вход.

## Где стоим (ГОТОВО и в проде)
- **QA-профиль в проде и отдан команде.** Codex-переделка автотест-лейна (субагенты + write/fix/batch/jira → `spawn_agent`, ADR-0025/`codex_body_path`) сделана, смержена (PR #133 → #136 → #138 fix → #139 role-bump).
- **Прод-каталог `tacticum_prod` (159.194.224.59):** цепочка сомкнута — installation `b258bb6b` pin **0.5.1** → лейн `iva-qa-autotest-base` **0.3.0** (+ `iva-qa-mcp` 0.1.0, `tacticum-core-base` 0.1.1). readyz 200 / MCP 401.
- **Доступ выдан:** Брейкин (`n.breykin@iva.ru`) + Байрамбеков (`e.bayrambekov@iva.ru`) — у обоих личные токены, installation в workspace `base`. Токен per-person (не per-профиль). Ждём их первый прогон.
- **Инструкция команде** отдана: [[iva-role-qa-ustanovka-i-rabota-dlia-komandy]] (репо `one-web-kmp`, стек pytest+playwright/canvas).
- **Проверено:** Level 0+1 статика (288 тестов), контролёр-гейт GO, Level-3 smoke на стенде `teststand` (38.180.236.39, под юзером `tacticum`): скиллы+агенты видны на codex 0.142.3, `spawn_agent` работает (`SPAWN_OK`).

## Следующий шаг
- Ждём **фидбек команды** по первому pull/прогону (R1 закроется на первом pull: `sync_count` 0→1).
- Развитие — по **тех-долгу** (ниже), приоритет за президентом.

## Тех-долг (развитие направления)
1. **Мульти-репо/стек:** сейчас 1 стек = `pytest-playwright-canvas` (KMP/`one-web-kmp`). Добавить: **one-web (Angular/selenium)** — ⚠️ ЖДЁМ ОТВЕТ ЖЕНИ (нужен сейчас или гоняем на текущем); **`squish`** (desktop, `git.hi-tech.org/autotest3/squish`); мобильные (Женя докинет).
2. **Полноценный Claude-провайдер:** Claude-путь структурно есть, но НЕ рабочий/НЕ проверен — блокер: субагенты `model: gpt-5.4` бейкается в `.claude/agents/*` → развести модель per-CLI (claude→opus/sonnet) + прогнать e2e на стенде.
3. **Kit-тиринг Codex+Claude** (профили ёмкости 0/1/2 по подписке): спек готов [[explore-qa-kit-capacity-model]]. profile 1 первым (Claude-reviewer через модуль `dispatch`, тело `work-reviewer` из kit готовым). native `spawn_agent` и `dispatch` ортогональны.
4. **Upstream ivaqa/kit-синк:** сейчас разовый byte-copy снапшота, живого git-доступа нет → нужен доступ от Жени + модель обновления.
5. **golden `codex.json` реген** (CI Level-2): нужен Docker (нет ни локально, ни на стенде).

## Что отсеяно / НЕ делать
- **lead-arch — не тянуть** (решение президента): R7 (доставка agent_spec→Codex) закрыт сам — механизм ADR-0025 уже в репо; QA-направление ведём сами.
- **Прод-VPS сид/выкатку** изначально отдавали Солонко, но по факту сид делали сами по добро президента (точечно, с бэкапом). Полный платформенный #119 в прод под QA-задачей НЕ катить единолично.
- Модель субагентов **gpt-5.4** (Codex, профиль 0) — верно, opus НЕ форсить.

## Ключевые факты (не потерять)
- **Механика доставки:** рёбра `depends_on` замораживаются на latest-active базы В МОМЕНТ сида роли (`seed_profile.py:76`). Обновил лейн → обнови роль (bump) + перепин installation, иначе потребитель на старом.
- **Сид прода:** ручной one-off контейнер (`docker compose ... run --rm --no-deps -v scripts -v templates catalog-mcp python scripts/seed_community.py templates`), образ НЕ пересобирать для контента; бэкап `pg_dump -Fc` перед каждой записью. psql — в контейнере `tacticum-postgres-1` (user `catalog`, db `tacticum_catalog`), НЕ в app-контейнере. curl нет в app-контейнере → host-curl по `dev.tacticum.dev/admin/*`, admin-токен = env `CATALOG_ADMIN_TOKEN`.
- **Серверы ssh-manager:** `tacticum_prod` (прод-каталог), `teststand` (тест-стенд, тулчейн под юзером `tacticum`, codex в `~/.npm-global/bin`, Docker НЕТ). Стенд: клона one-web-kmp нет — для полного write-autotest e2e нужен целевой репо + VPN до ИВА + TestOps-креды.
- **Контакты:** Женя (автор kit/toolkit) · Брейкин Никита (целевой контакт передачи) · Солонко (платформа/выкатка) · Глеб (тест-матрица #119).
- **Бэкапы отката на проде:** `/tmp/catalog_backup_*` (по 11M).
