---
title: recon-aqa-kit-zhenya
type: note
status: draft
created: 2026-07-22 17:10
updated: 2026-07-25 20:30
permalink: tacticum/00-board/recon-aqa-kit-zhenya
project: tacticum-dev / профили
lead: aqa-scout
blocked: нет read-доступа к приватному GitLab git.hi-tech.org (группа ivaqa)
tags:
- recon
- aqa
- qa
- kit
- profiles
- lead-qa
---

# Разведка AQA-kit (Женя) — карта

**Когда:** 2026-07-22 ~17:10 · **Кто:** aqa-scout → director + lead-qa
**Состояние:** заблокировано — нет доступа к репо (было `status: blocked-no-access`, приведено к словарю §3 как `draft` + поле `blocked`).

## Доступ: НЕТ (репо не прочитаны)
Цель — `git.hi-tech.org/ivaqa/kit` (новый) и `.../aqa-agent-toolkit` (старый). Что пробовал:
- **helm** — `git ls-remote` → DNS **NXDOMAIN** (резолвер helm 198.18.18.18 не знает git.hi-tech.org). Хоста не видит.
- **adp_emb** — DNS резолвится во внутренний **10.0.207.3**, хост достижим. Платформа = **GitLab** (приватный, `nginx`, редирект на `/users/sign_in`). Но:
  - HTTPS `git ls-remote` → `could not read Username` (нужен логин/PAT — не сохранён).
  - SSH `git@git.hi-tech.org` → `Permission denied (publickey,password)` (нет ключа).
  - GitLab API `/api/v4/projects/ivaqa%2Fkit` → `404 Project Not Found` (анонимно недоступно).
- Клонов `kit`/`aqa-agent-toolkit` на серверах **нет**; PAT/`.git-credentials`/`.netrc` — **нет**. Единственный ivaqa-клон на adp_emb/helm — `ivaqa/iva-ai-qa-assist` (Flask-дашборд, к toolkit не относится).

**Вывод:** без выданного PAT или SSH-ключа к GitLab ivaqa репо не прочитать. **Нужен доступ от Жени** (PAT read_repository на группу `ivaqa`, либо деплой-ключ на adp_emb/helm, либо разовый архив/зеркало).

## kit: структура / маркетплейс / развёртка / фидбеки
**Не читал — не выдумываю.** Из досье направления (со слов, не из кода): `kit` = маркетплейс плагинов под **codex и claude**, deploy-ready на проекты автоматизации потребителей, механизм фидбеков/обновлений. Детали (что есть плагин, манифест, деплой, каналы обновлений) — **открыты до получения доступа**.

## 3 кросс-провайдерных профиля
**Не читал.** Из досье, заявлено 3: (а) **codex-only**; (б) **codex + claude как ревьюер**; (в) **codex + claude, забирающий существенную часть флоу**. Что именно делает каждый, критерии выбора — **открыты**.

## diff новый vs старый
**Не читал** ни `kit`, ни `aqa-agent-toolkit`. Заявлено: `kit` = следующая эволюция старого toolkit + улучшения (deploy на проекты, фидбеки, кросс-провайдерные профили). Конкретный diff — **открыт**.

## Связь с нашим iva-role-qa (наша сторона — прочитана из templates/)
Роль `iva-role-qa = [tacticum-core-base, iva-qa-autotest-base, iva-write-base]` (роль-пресет ADR-0057, своих ингредиентов 0). Лейн `iva-qa-autotest-base` = **9 скиллов** (write/batch-autotest, playwright-cli, run-tests, fix-failed-test, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro).

- **upstream-статус ПОДТВЕРЖДЁН из провенанса:** все 9 SKILL.md + 35 references — **byte-copy из «источника QA-команды (skills one-web)»**; правка только frontmatter (снят `allowed-tools`). Т.е. **toolkit Жени — фактический upstream наших скиллов** (сейчас — форк-копия, не связь).
- **3 заблокированных скилла** (`write/batch/fix`) делегируют через Task трём субагентам `codebase-analyst`/`dom-explorer`/`code-writer` — **их defs в источнике нет**, запрошены у QA-команды. **Первый кандидат «забрать из kit».**
- **Расхождение по провайдерам:** наш лейн `codex: best-effort` (половина скиллов на Claude-специфике: Task-субагенты, `Skill()`, PostToolUse-хуки), роль заявляет `codex: full`. У Жени `kit` изначально **кросс-провайдерный** (codex/claude) — их модель профилей может закрыть наш codex-гэп.
- **Репо-специфичность:** наш лейн жёстко на `one-web` (autocore/venv/tools.testops/glab/CI), не агностичен. У `kit` заявлен deploy на разные проекты потребителей — вероятно там обобщение окружения/CI.

### Перенять / интегрировать / согласовать
- **Перенять:** 3 def субагентов (`codebase-analyst`/`dom-explorer`/`code-writer`) — разблокируют 3 из 9 наших скиллов.
- **Интегрировать:** рассмотреть `kit` как upstream (адаптер/submodule/канал обновлений) вместо разовой byte-copy; их deploy-механизм — под нашу репо-специфичность one-web.
- **Согласовать:** (1) маппинг 3 кросс-провайдерных профилей Жени ↔ наша модель роль-пресетов и codex-гэп; (2) развилка «кто генерит TC» — аналитик (`tests-authoring`) vs QA (методолог-потребитель за ГД); (3) санитизация кредов/путей (в references зашиты `allure.iva.ru`, токен в `tools/testops/client.py`).

## Открытые вопросы к Жене (до чтения репо)
1. Дай read-доступ к `ivaqa/kit` и `aqa-agent-toolkit` (PAT read_repository / деплой-ключ на helm|adp_emb / архив).
2. Есть ли в `kit` defs субагентов `codebase-analyst`/`dom-explorer`/`code-writer`?
3. Как `kit` деплоится на проект-потребитель и как обобщает окружение/CI (наш кейс — one-web)?
4. Как 3 кросс-провайдерных профиля выбираются и что делает claude в вариантах (б)/(в)?

## Сигнал
- [ ] to:director from:aqa-scout Репо ivaqa/kit+toolkit НЕ прочитаны (нет PAT/ключа к приватному GitLab git.hi-tech.org); наша сторона: 9 скиллов = byte-copy из upstream Жени, нужен доступ + 3 def субагентов.

## Связано
[[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]] · [[Решения по QA-профилю (Трек B) — 2026-07-21]] · [[qa-profile-model — опись + мульти-стэк модель QA-лейнов]]
