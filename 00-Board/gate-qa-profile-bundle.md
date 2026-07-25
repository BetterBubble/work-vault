---
title: gate-qa-profile-bundle
type: report
permalink: tacticum/00-board/gate-qa-profile-bundle
tags:
- controller
- gate
- qa
- lead-qa
- bundle
- acceptance
---

# Гейт controller — бандл QA-профиля (ветка feat/qa-kit-subagents)

**Когда:** 2026-07-23 · **Кто:** controller → lead-qa
**Метод:** read-only, по факту на диске + реальный прогон тестов в поднятом backend-venv. НЕ по самоотчётам.
**Worktree:** `~/tacticum/tacticum-dev-qa-kit` · ветка `feat/qa-kit-subagents`
**Пакет:** (#1+#2) субагенты kit + санитизация · (write) ретайр `iva-write-base` → 3 тонких per-role MCP-лейна (вар. B) · (helm) read-срез helm-analyst в `iva-qa-mcp` · (модели) 3 субагента opus→gpt-5.4.
**Проверяет:** [[impl-qa-kit-subagents]] · [[impl-qa-role-atlassian-write]] · [[impl-qa-mcp-thin-lanes]] · [[verify-qa-kit-subagents]]

---

## ВЕРДИКТ: GO с условием (CI-гейт по DB-тестам)

Всё, что проверяемо в этом окружении — **чисто и зелёно по всем 6 пунктам**. Единственный незакрытый риск — DB-backed catalog-тесты (нужен Docker/Postgres, недоступен здесь). Не найдено ни одного дефекта, только среда не позволяет добить DB-набор. **Мерж — только после зелёного прогона DB-catalog-набора в docker-enabled CI** + ребейз на main.

---

## Пункты

### 1. Гит-чистота — PASS (+1 не-блокер)
- `git status` пуст, дерево чистое.
- Все 18 коммитов авторства «Александр Шульга»; **AI-подписей НЕТ** (grep по телам: Claude/Co-Authored/Generated/anthropic = 0; совпадения грепа — легит-контент про `$CRAFT/$TMS`).
- Секреты: literal `phk_*`/`Bearer <token>`/`ATATT`/`password=…` — **0** по всем изменённым файлам. Каналы на env (`JIRA_PERSONAL_TOKEN`, `TACTICUM_TOKEN`) — верно.
- `allure.iva.ru` = **3 хита, все в README/CHANGELOG-мете** («что вычищено»); в операционных телах (SKILL/agents/references/craft-stack) = **0**.
- Мусор: `.DS_Store`/`__pycache__`/`.pyc` = 0. `.serena/` **предсуществует в main (21 файл), наша ветка его НЕ трогает** — не наша регрессия (общерепозиторный вопрос).
- **Не-блокер:** ветка отстала от main на **32 коммита** (форк от `feat/iva-write-base` @ 802ddc7; main ушёл вперёд PR #116–#130 — другие направления). Оттого `git diff main..HEAD` показывает 217 файлов — это РЕВЕРС чужих правок, НЕ наша работа. **Нужен ребейз на main перед PR** (возможны конфликты по catalog-тесту/ролям).

### 2. Скоуп — PASS
Истинная дельта ветки (802ddc7..HEAD, 18 коммитов) = **80 файлов templates/ + 1 apps**, БЕЗ выхода за templates/apps. Каталоги ровно авторизованные:
`iva-qa-autotest-base` (62) · `iva-role-qa`/`iva-role-architect`/`tacticum-role-techwriter` (3 каждый) · `iva-qa-mcp`/`iva-architect-mcp`/`iva-techwriter-mcp` (3 каждый) · снос `iva-write-base` · `apps/backend/tests/catalog/test_iva_role_presets.py`. **Другие направления не задеты.** Разрастания нет.

### 3. Достоверность/консистентность — PASS
- 3 роли `ingredients: []` (pure-composition, подтв. валидатором) ✓
- `grep -rn iva-write-base templates/` = **0**; каталог снесён физически ✓
- **iva-qa-mcp реально несёт оба сервера:** `iva-atlassian-write` (allowed_tools = `jira_create_issue/update_issue/add_comment/get_transitions/transition_issue`) + `helm-analyst` (http helm.tacticum.ru/mcp/analyst, bearer; allowed_tools = точный READ-срез `requirement_tests/gap_questions/contradiction_check/nearest_spec/affected_systems/constraints/related_tasks`) ✓
- helm **НЕ в non_goals** QA — наоборот, зафиксировано «QA ЧИТАЕТ покрытие requirement_tests через helm-analyst» (реверс Track-B-исключения) ✓
- 3 agent_spec `model: gpt-5.4` (строки 206/218/230); `opus` остался ЛИШЬ в комментарии «ранее был хардкод opus — снят» ✓

### 4. Тесты — прогнал сам (backend-venv поднят: `uv sync --extra dev`, python 3.13)
- **`test_iva_role_presets.py` + `test_manifest_schemas.py` = 73/73 PASS** (100%). Impl заявлял 35/35 — по факту ещё больше, все зелёные. Покрыто: pure-composition, depth1, single-owner, **golden-parity union==sum на уровне манифестов** — зелено.
- Весь `tests/catalog/` = **330 passed · 120 error · 2 failed**.
- **120 error + 2 failed — это DB-backed тесты, поднять НЕ удалось: нет Docker-демона.** `tests/conftest.py:35 postgres_url` спавнит контейнер `postgres:16-alpine` (`docker run` → exit 125 «Cannot connect to the Docker daemon»); async-тесты admin-authoring лезут к живому Postgres :5432 → OSError. Среда сандбокса без docker/БД.
  - Сюда попал целевой **`test_seed_depends_on.py`** (13 кейсов, все в error по docker) — **добить не смог по той же причине, что и implementer.** Это ключевой незакрытый риск: именно он валидирует сидинг depends_on-рёбер в БД (а мы меняли depends_on ролей и снесли лейн). Статически рёбра сверены (`test_iva_role_presets` зелёный), но DB-сидинг непроверен.
  - 2 «failed» (`test_admin_catalog_authoring::test_patch_profile_404`, `test_create_draft_404_unknown_profile`) — **средовые** (async к живому backend/DB), НЕ наши: наши коммиты admin-authoring не трогают.

### 5. Валидатор (manifest.v2 + ingredient.v1) — PASS, 7/7
```
iva-role-qa               manifest=VALID  ingredients=0  all VALID
iva-role-architect        manifest=VALID  ingredients=0  all VALID
tacticum-role-techwriter  manifest=VALID  ingredients=0  all VALID
iva-qa-mcp                manifest=VALID  ingredients=2  all VALID
iva-architect-mcp         manifest=VALID  ingredients=1  all VALID
iva-techwriter-mcp        manifest=VALID  ingredients=1  all VALID
iva-qa-autotest-base      manifest=VALID  ingredients=15 all VALID
```

### 6. Память/регламент — PASS
4 отчёта на доске на месте (3 impl + 1 verify); досье направления `napravlenie-profili-qa-profil-iva-role-qa-aqa-toolkit-iva` присутствует; сигнал lead-arch 13:00 (helm read-пресет, вар. B) зафиксирован в `00-board/signals`.

---

## Разведение

**Чисто-подтверждено (6/6 проверяемого):** гит-чистота (секреты/подписи/мусор), скоуп (только QA + 3 роли + 3 лейна), консистентность (ingredients==[], оба сервера в iva-qa-mcp, gpt-5.4, helm не в non_goals, iva-write-base=0), схемы 7/7 VALID, статические тесты 73/73 + весь non-DB catalog 330 passed, память.

**Не-блокеры (на чистку/тикет, вне гейта):**
- Ребейз ветки на main (отстала на 32 коммита; форк на feat/iva-write-base).
- `retro/SKILL.md:25` — абс. путь `-Users-oc4kxb-…/one-web/memory` (подтв.; больше абс.путей/чужих юзеров в скоупе НЕТ). Отдельным коммитом на чистку.
- Двойной фронтматер body+metadata при рендере `.claude/agents/*` — общерепозиторная конвенция, тикет.
- Живой прогон 3 скиллов (subagents на gpt-5.4) в one-web — по сценарию verifier'а, за держателем one-web+TestOps.

**Условие мержа (не дефект — непроверенное):**
- **DB-backed catalog-набор** (`test_seed_depends_on`, seed_community_*, migration-data, golden-render, admin-authoring) — **обязан пройти зелёным в docker-enabled CI до мержа.** Здесь не поднять (нет docker). Дефектов не найдено, но и PASS выдать не могу — это условие, а не блокер по факту.

**Настоящих блокеров (дефектов) — НЕ найдено.**

## Связано
[[impl-qa-kit-subagents]] · [[impl-qa-role-atlassian-write]] · [[impl-qa-mcp-thin-lanes]] · [[verify-qa-kit-subagents]] · [[napravlenie-profili-qa-profil-iva-role-qa-aqa-toolkit-iva]]
