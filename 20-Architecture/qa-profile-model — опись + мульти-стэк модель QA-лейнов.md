---
title: qa-profile-model — опись + мульти-стэк модель QA-лейнов
type: report
permalink: tacticum/20-architecture/qa-profile-model-opis-multi-stek-model-qa-leinov
status: current
role: lead-qa
date: 2026-07-21
updated: 2026-07-23
tags:
- qa
- role-presets
- architecture
- multi-stack
- tacticum-dev
- lead-qa
---

# Модель QA-профиля: финальная карта + мульти-стэк

Канонная опись профиля `iva-role-qa`. Собран из kit-снапшота Жени, пересобран на прод write-канал, прогнан через гейт (GO). Ветка `feat/qa-kit-subagents` (не в main). Направление: [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]].

## Финальная композиция (`iva-role-qa` v0.4.0)

Чистый пресет (`ingredients: []`) — всё через `depends_on`:

| Лейн | Что даёт |
|---|---|
| `tacticum-core-base` | репо-биндинг (tacticum-context), KB-навигация, онбординг (getting-started), git-конвенции + базовый MCP (context7 + tacticum-mcp) |
| `iva-qa-autotest-base` | автотест-конвейер: 9 скиллов + `craft-stack` + 3 субагента (привязан к репо one-web) |
| `iva-qa-mcp` | MCP-обвязка: write Jira-дефектов (`iva-atlassian-write`) + READ-срез факт-базы (`helm-analyst`) |

### 9 скиллов (автоматизация автотест-кода, НЕ тест-дизайн)
`write-autotest` (1 TC → pytest) · `batch-autotest` (пачка TC) · `playwright-cli` (локаторы в живом UI) · `run-tests` (прогон pytest+браузер+allure-raw) · `fix-failed-test` (разбор падений + починка) · `jira-issue-autotest` (Jira issue → автотесты → glab-пайплайн) · `prepare-mr-branch` (чистая MR-ветка) · `rebuild-autocore` (пересборка autocore-venv) · `retro` (housekeeping). (+ `craft-stack` — референс-стек `pytest-playwright-canvas`, не user-навык.)

### 3 субагента (`agent_spec`, model: **gpt-5.4 — Codex**, профиль ёмкости 0)
| Субагент | Роль | tools |
|---|---|---|
| `codebase-analyst` | read-only инвентарь page-слоя/flows/i18n/enum + пробелы | Read, Glob, Grep |
| `dom-explorer` | разведка живого UI (единственный с браузером), `locators.md` | Bash, Read, Write, Glob, Grep |
| `code-writer` | генерация кода тест-слоя по `analysis.md`+`locators.md`, `py_compile` | Read, Edit, Write, Glob, Grep, Bash |

### MCP и скоуп
| Сервер | Скоуп (allowed_tools) |
|---|---|
| `iva-atlassian-write` | **только Jira дефекты/статусы** — `jira_create_issue`, `jira_update_issue`, `jira_add_comment`, `jira_get_transitions`, `jira_transition_issue`; env без `CONFLUENCE_*` (uvx mcp-atlassian, Server/DC jira.iva.ru, личный PAT) |
| `helm-analyst` | **READ-срез, 7 тулов** — `requirement_tests`, `gap_questions`, `contradiction_check`, `nearest_spec`, `affected_systems`, `constraints`, `related_tasks`; без C4/coverage-владения/write (http helm.tacticum.ru/mcp/analyst, Bearer) |
| context7 + tacticum-mcp | базовый MCP из core-base (kb_*, context7) |

Своего write-лейна нет (ретайр `iva-write-base` → тонкий `iva-qa-mcp`). Allure TestOps — через `tools/testops` внутри репо one-web (не MCP).

### Покрытие workflow
Генерация ТС→код · прогон · починка · публикация (Allure+MR) · дефекты (Jira) = **есть, verified**. Валидация + дополнение ТС = **read-опора есть** (helm `requirement_tests` + `write-autotest/references/tc-review.md`), сам flow — **в парке** (интеграция аналитик↔QA, ждёт ADR lead-arch). Покрытие требований = **только чтение**, не владение.

### Чего профиль НЕ делает
Авторинг ТС / тест-дизайн (аналитик) · продуктовый код · постановка требований (BRD/FR/ADR) · авторинг FR-страниц в Confluence · владение покрытием требований.

### Провайдер/модель
Субагенты — Codex `gpt-5.4`. Роль: claude-code = full, codex = full (⚠ автотест-лейн на codex реально best-effort — Task-субагенты/`Skill()`/хуки Claude-специфичны).

## QA ↔ Аналитик (границы)
| Ось | QA (`iva-role-qa`) | Аналитик (`iva-role-analyst`/`iva-fr-analyst`) |
|---|---|---|
| Назначение | автоматизация автотестов + дефекты + ревью-сверка | тест-дизайн/авторинг ТС + постановка требований |
| Скиллы | 9 автотест | analysis (fr-authoring, tests-authoring, api-contracts, brd/adr/pin…) |
| Субагенты | 3 craft, **gpt-5.4 (Codex)** | 2 в analysis-base, **opus**; fr-analyst — 0 |
| `helm-analyst` | **узкий read, 7 тулов** | **полный сервер** (+arch_map, requirement_coverage, docs_ask, RAG#2) |
| `atlassian-write` | только **Jira дефекты** | Jira **+ Confluence** (авторинг+публикация FR) |
| `iva-read`/`iva-mcp` | нет | есть |
| Ownership | автотест-код + дефекты; **читает** ТС/покрытие | владеет ТС/тест-дизайном/**покрытием**/требованиями + публикацией FR |

**Стык:** аналитик авторит ТС и владеет покрытием → QA берёт готовые ТС, автоматизирует, покрытие лишь **читает** (тот же helm-сервер, узкий срез; разница только в `allowed_tools`). Дефекты — QA; Confluence/FR — аналитик. Полная карта: [[explore-qa-vs-analyst-final]].

## Мульти-стэк модель (принцип)
- **web-only — не ограничение дизайна, а следствие:** скиллы команды жёстко на web-тулинг (Playwright/Selenium/pytest/autocore/glab).
- **Один общий лейн исполнения невозможен** — браузерная автоматизация ≠ нативное мобильное тестирование.
- **Каждый стэк = отдельный лейн** со скиллами его команды:
  - `iva-qa-autotest-base` (web, есть) → `iva-role-qa` = `[core, iva-qa-autotest-base, iva-qa-mcp]`
  - `iva-qa-ios-autotest-base` (XCUITest, будущее) → `iva-role-qa-ios`
  - `iva-qa-kmp-autotest-base` (Espresso/KMP, будущее) → `iva-role-qa-kmp`
- **Стэк-агностичное — НЕ дублировать:** авторинг ТС (`tests-authoring`) + покрытие (`requirement_tests`) живут в analysis, шарятся всеми QA-ролями (QA — только чтение среза).
- **Расширение на iOS/KMP** = получить автотест-скиллы тех команд (как получили web). Пока их нет → **web-only = реальный скоуп сегодня.**

## Развилка «кто генерит TC» — РЕШЕНА (2026-07-23)
**Аналитики пишут ТС**, **QA дополняет на ревью**. Авторинг у аналитика (`tests-authoring` в analysis); QA-лейн = исполнение/автоматизация + **ревью-дополнение**. Решение: [[Решение- тест-кейсы пишут аналитики, QA дополняет на ревью (2026-07-23)]].

## Статус доставки (2026-07-23)
Бандл собран, **controller-гейт GO с условием, дефектов нет** ([[gate-qa-profile-bundle]]): гит-чистота/скоуп/консистентность/валидатор 7/7/память — PASS; 73/73 профильных + 330 non-DB catalog зелёные. **Условие мержа:** DB-backed catalog-тесты (`test_seed_depends_on`…) — в docker-CI (локально нет docker). Не-блокеры: ребейз на main, `retro:25`, тикет двойного фронтматера. **Готов к PR — ждёт OK пользователя.** Осталось до «QA тестят»: PR+merge → provision в one-web + доступы (`TESTOPS_*`/Atlassian PAT/helm Bearer) → живой прогон 3 скиллов QA-командой по сценарию [[verify-qa-kit-subagents]].

## Связано
- [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]] · [[gate-qa-profile-bundle]] · [[explore-qa-vs-analyst-final]] · [[verify-qa-kit-subagents]]
- [[Решения по QA-профилю (Трек B) — 2026-07-21]] · [[recon-kit-full-qa-dorabotka]] · [[Решение- тест-кейсы пишут аналитики, QA дополняет на ревью (2026-07-23)]]

## Уточнено по ADR-0060 (2026-07-23)
Финальная модель — **ADR-0060** (репо `tacticum-dev/docs/adr/`; ранее номер 0059 — переименован). Для QA: **QA = исполнение/автоматизация + review-augment TC** (авторинг TC у аналитика; уточняет ADR-0058 §6); helm read-срез доставляется **через capability-лейн** (`allowed_tools` в ЛЕЙНЕ, не на роли); write — `iva-write` ADR-0058 (техучётка+подпись, scope `iva-req-write`, не личный PAT); **гейты §6** (`/qa-coverage` + «regression from PIN Impact» = обязательная часть гейта шага 4; `tc: reviewed→covered` при зелёных).

> **Правка номера (2026-07-23):** «ADR-0059» выше = **ADR-0060** (переименован из-за коллизии номера; канон `docs/adr/0060-profile-interaction-model-mcp-scoping-pipeline-gates.md`).