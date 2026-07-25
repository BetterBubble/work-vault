---
title: audit-qa-profile-completeness
type: note
permalink: tacticum/00-board/audit-qa-profile-completeness
status: draft
lead: qa-audit
date: 2026-07-23
worktree: ~/tacticum/tacticum-dev-qa-kit
branch: feat/qa-kit-subagents
method: read-only (код на диске + git + прогон preset/schema-тестов в backend-venv
  + сверка с памятью + 5 QA-звонков)
tags:
- audit
- qa
- completeness
- go-no-go
- lead-qa
- director
---

# Аудит комплектности QA-профиля (iva-role-qa) — 2026-07-23

**Вопрос:** (1) соответствует ли профиль должному, (2) полна ли комплектация, (3) какие пробелы, (4) можно ли пушить/мерджить и реально отдавать QA на тесты.

**Ключевой факт:** код **опережает** прежние отчёты. Вариант B (тонкие MCP-лейны, helm read-срез, pure-composition leaf) и снятие opus→gpt-5.4 **уже смёржены в ветку** (коммиты 54812e6 / ed8b74b / fa7d62e / b866c0f). Прежние заметки verify/impl фиксировали промежуточное состояние (15 падавших тестов, own-MCP в роли) — оно **закрыто**. Сверено по факту на диске.

---

## 1. Матрица «должно быть ↔ есть»

| # | Должно быть (звонки+память) | Статус | Где / доказательство |
|---|---|---|---|
| 1 | **9 скиллов, все рабочие** (3 write/batch/fix были заблокированы) | **ЕСТЬ** | `ingredients/skills/` = write-autotest, batch-autotest, fix-failed-test, playwright-cli, run-tests, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro. 3 ранее-заблокированных **разблокированы**: их SKILL.md делегируют субагентам через `Task` (write:134/151/163, batch:36/55/58/59, fix:57-60/209-219). + `craft-stack` (10-й skill_spec — общий дом). |
| 2 | **3 субагента разблокированы и полны** (стек / оркестраторы / `[craft]`) | **ЕСТЬ (комплектом)** | `ingredients/agents/{codebase-analyst,dom-explorer,code-writer}.md` — полные defs, режимы write/fix/adhoc, tools проставлены. Стек `pytest-playwright-canvas` = 15 файлов `craft-stack/` (recon, rules/, test-templates, failure-taxonomy, fix-playbooks, allure-raw-parser, pytest-runner, batch-conventions, shared/). Оркестраторы write/batch/fix спавнят их bare-именами. `[craft]`-параметры — `craft-answers.example.toml` (stack/shared_users/research_user_cmd/vendored_dep/product_clone/default_env). |
| 3 | **TC review-augment + helm read-срез проставлен** | **ЕСТЬ (helm) / ЧАСТИЧНО (ревью)** | helm-analyst read-срез **проставлен** в `iva-qa-mcp` (7 тулов: requirement_tests, gap_questions, contradiction_check, nearest_spec, affected_systems, constraints, related_tasks); helm **снят с non_goals** (реверс Track-B-исключения). Само TC-ревью — byproduct `write-autotest/references/tc-review.md`; standalone «ревью фич-реквеста ДО разработки» живёт **у аналитика** (shared), координируется ADR lead-arch — по решению 23.07 (аналитик пишет TC, QA дополняет). |
| 4 | **Write-канал iva-atlassian-write сужен** | **ЕСТЬ** | `iva-qa-mcp`: `iva-atlassian-write`, `allowed_tools = jira_create/update_issue, add_comment, get/transition_issue`. `CONFLUENCE_*` НЕ выставляется → page-authoring не поднимается. Личный Atlassian PAT (JIRA_URL + JIRA_PERSONAL_TOKEN). |
| 5 | **env-санитизация секретов** | **ЕСТЬ** | 0 хардкодов `allure.iva.ru`/токенов в операционных телах (3 хита — только README/CHANGELOG-мета «что вычищено»). env-модель `TESTOPS_*` → gitignored `secrets.yaml` + `secrets.yaml.example` + `.gitignore` append. Каналы на env (JIRA_PERSONAL_TOKEN, TACTICUM_TOKEN). |
| 6 | **Укладка на единый механизм сборки** | **ЧАСТИЧНО (осознанно)** | Финализировано на текущей схеме manifest v2 (решение пользователя: не блокируемся на гардрейле Солонко «три→один», адаптируем когда готов). Валидно против manifest.v2 + ingredient.v1 (7/7 VALID). Целевой механизм Солонко ещё не готов → укладка «требует сверки», не «готово». |
| 7 | **База на скиллах Брейкина** | **ЕСТЬ** | 9 скиллов + 3 субагента + стек = перенос из upstream `kit` (git.hi-tech.org/ivaqa/kit, модуль craft) команды Брейкина/Жени. |
| + | **pure-composition leaf (ADR-0057)** | **ЕСТЬ (вар. B)** | `iva-role-qa`: `ingredients: []`, `depends_on = [tacticum-core-base, iva-qa-autotest-base, iva-qa-mcp]`. `test_iva_role_presets.py` + `test_manifest_schemas.py` = **73/73 PASS** (прогнал сам в backend-venv). |
| + | **Модель субагентов** | **gpt-5.4, НЕ opus** | 3 agent_spec `model: gpt-5.4` (Codex профиль 0). Хардкод opus снят (b866c0f) по поправке пользователя (базовый носитель — Codex). Задача просит «opus» → это **развилка** (см. пробел 5). |

---

## 2. Пробелы

1. **[БЛОКЕР МЕРЖА, не пуша] DB-backed catalog-тесты не прогнаны** — `test_seed_depends_on` (валидирует сидинг depends_on-рёбер, а мы меняли depends_on и снесли лейн) и др. требуют docker/Postgres, здесь нет демона. 73/73 статических + 330 non-DB зелёные, дефектов НЕ найдено, но PASS по DB выдать нельзя. **Обязаны пройти зелёными в docker-CI до мержа.**
2. **[МЕРЖ] Ребейз на main** — ветка отстала на **32 коммита** (форк от `feat/iva-write-base`). `diff main..HEAD` = 217 файлов (реверс чужих правок, не наша работа). Нужен ребейз перед PR; возможны конфликты по catalog-тесту/ролям.
3. **[ОТДАЧА QA] Живой прогон 3 скиллов не проведён** — нет one-web/TestOps/стенда/кредов на нашей стороне. Verifier дал точный сценарий приёмки; прогон write/batch/fix (субагенты gpt-5.4) — за держателем one-web (QA-команда Брейкина).
4. **[ОТДАЧА QA] Provision + доступы** — лейн НЕ провижнит окружение (venv/autocore/playwright-cli/allurectl/glab/CI) — предполагает настроенное one-web. Доступы: **3 ключа** — `TESTOPS_*` (свой Allure PAT) + Atlassian PAT (Jira) + `TACTICUM_TOKEN` (helm phk); выпускает инженер, без них серверы/скиллы деградируют явно. На нашей стороне не закрыто (звонки: analysts/QA дают свои ключи Jira/Confluence/Allure).
5. **[РАЗВИЛКА] Модель субагентов gpt-5.4 vs opus** — текущий код gpt-5.4 (Codex-primary, поправка пользователя, kit-схема профиль 0/1/2). Задача упоминает opus. Нужно подтверждение носителя.
6. **[НЕ БЛОКЕРЫ] Мелочи:** `retro/SKILL.md:25` — абс.путь чужого юзера (`-Users-oc4kxb-…/one-web/memory`), на чистку; двойной фронтматер body+metadata при рендере `.claude/agents/*` — общерепозиторный тикет; интеграционный слой аналитик↔QA (канон TC-handoff, write-back TC в TestOps, ownership ревью-ingredient) **заблокирован ADR lead-arch** — координируется, не проектируется в QA.
7. **[ОБЛАСТЬ] Стек-привязка** — лейн web-only (one-web: autocore/venv/glab/CI). Мульти-стэк (iOS/KMP) = отдельные лейны, сегодня нет. Покрытие ~0 — вопрос capacity (один тестировщик), профиль его не расшивает (ADR §7 «вне скоупа»).

---

## 3. Go / No-Go (РАЗДЕЛЬНО)

### (а) Пушить / мерджить КОД
- **ПУШ ветки — GO.** Дерево чистое; скоуп только QA (80 файлов templates + 1 apps); 0 секретов / AI-подписей / мусора; статик 73/73 + non-DB catalog 330 passed + валидатор 7/7. Дефектов НЕ найдено.
- **МЕРЖ — NO-GO пока (условно).** Условие (не дефект): зелёный DB-catalog в docker-CI + ребейз на main. После них — GO на бандл-PR. PR/merge — ручной шаг пользователя.

### (б) Реально отдавать QA на тесты (живой прогон)
- **NO-GO на нашей стороне; GO — руками QA-команды в one-web.** У нас нет one-web/TestOps/стенда/3 ключей. Отдаём **ветку + точный сценарий приёмки** (verifier). Живой прогон 3 скиллов + provision (venv/autocore/CLI) + доступы (3 ключа) — за держателем one-web. До живого прогона «готовность» = статическая (built+verified), не боевая.

**Вердикт:** код готов к PR-бандлу (пуш — GO; мерж — после docker-CI DB-тестов + ребейз); реальная отдача QA на тесты — за держателем one-web (provision + доступы + живой прогон), у нас статик-приёмка PASS. Профиль **соответствует должному**, комплектация полна на статике; открыты только гигиена (ребейз/DB-CI) и внешняя приёмка.

## Сигналы
- [ ] to:director from:qa-audit Аудит комплектности QA-профиля: соответствует должному, комплектация полна на статике (9 скиллов+3 субагента+стек+helm read-срез+сужен write+env-санитизация, 73/73). GO на ПУШ; МЕРЖ — после docker-CI DB-тестов + ребейз на main; РЕАЛЬНАЯ отдача QA на тесты — за держателем one-web (provision+3 ключа+живой прогон). Развилка: субагенты gpt-5.4 vs opus. Детали [[audit-qa-profile-completeness]]

## Связано
[[gate-qa-profile-bundle]] · [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]] · [[recon-kit-full-qa-dorabotka]] · [[Решение- тест-кейсы пишут аналитики, QA дополняет на ревью (2026-07-23)]] · [[ADR (draft) — Модель взаимодействия профилей- аналитик ↔ QA ↔ разработчик + скоупинг MCP по ролям]] · [[verify-факты- ADR модель профилей]]