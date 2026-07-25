---
title: Решения и итоги — QA-профиль / Codex-переделка (2026-07-24)
type: decisions
status: current
role: lead-qa
date: 2026-07-24
permalink: tacticum/00-board/decisions-qa-codex-2026-07-24
tags:
- decisions
- qa
- codex
- iva-role-qa
- lead-qa
---

# Решения и итоги — QA-профиль, 2026-07-24

Сводка решений президента + ключевых находок за сессию. Досье направления: [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]].

## Итоги (закрыто)
- **PR #133 смержен в main** (`20ff9b8`) — профиль `iva-role-qa` + `iva-qa-autotest-base` + 3 per-role MCP-лейна. Пост-мерж гигиена: qa в `ROLE_LANES`/`ROLE_PERSONA`, статика **73/73**.
- **CI-сид каталога** отработал — роль устанавливаема.
- **Комплект передачи QA-команде** ([[qa-profil-onbording-komandy-iva-peredacha-i-zapusk]] + `12-Features/QA-профиль iva-role-qa — комплект передачи команде`) — готов, gpt-5.4, внешний регистр.
- **Quickstart** `docs/user_manuals/iva-role-qa-profile-quickstart.md` — готов (worktree, не закоммичен).
- **Прод-сид подготовлен** ([[prod-seed-iva-role-qa-prep]]) — до «нажать выполнить», передаём Солонко.
- **Чанк 1 Codex-переделки** ([[impl-qa-codex-mechanical]]) — механика (secrets/gitignore/retro→кросс, craft-stack пути, run-tests), версия 0.1.0→0.2.0, тесты 73/73.

## Решения президента (2026-07-24)
1. **Модель субагентов = `gpt-5.4`** (Codex, тир coder), не opus. «GPT-5.6» руководителя ≈ линия Codex GPT-5, точечный бамп не требуется (тривиально при желании).
2. **Профиль по варианту B ADR-0057** — тонкие per-role MCP-лейны, не own-MCP-в-роли.
3. **Codex-переделка ядра автотест-лейна — ДЕЛАЕМ** (вариант B): write/fix/batch + 3 субагента с Claude-native на Codex.
4. **Механизм субагентов на Codex = NATIVE Codex `[agents]`-треды** (НЕ kit dispatch, НЕ инлайн).
5. **Прод-выкатку делает СОЛОНКО**, не lead-qa. Прод-VPS `159.194.224.59` SELECT/сид + риск сида — с нашей тарелки; отдаём готовую последовательность.
6. **Тест-стенд `38.180.236.39`** — доступ у президента; Уровень 3 (живой Codex) — канон приёмки до передачи QA-команде.
7. **Порядок:** финализировать профиль полностью → полная проверка на стенде.
8. **Единый профиль с режимами** (QA как режим/пак аналитика) — стратегически на будущее, СЕЙЧАС не переделывать.

## Ключевая находка — механизм Codex-субагентов (из /openai/codex docs)
Снимает узел R1 и конкретизирует R7 для lead-arch:
- **Субагент Codex = агент-роль** (`[agents.<role>]` в `.codex/config.toml` + `config_file="./agents/<role>.toml"` с `instructions`=system-prompt+`model`; либо авто-дискавери `.codex/agents/*.toml`/`.agents/*.toml`). Это ДРУГОЙ канал доставки, чем скиллы (`.agents/skills`).
- **Спавн** = нативный тул `spawn_agent(agent_type,task_name,message,model)` + `wait_agent` + `close_agent`; глобально `[agents] max_threads/max_depth`.
- → Финализация возможна БЕЗ стенда и БЕЗ ожидания R7-схемы (провизорно кладём файлы, декларацию манифеста выравниваем по схеме lead-arch позже).

## #119 СМЕРЖЕН (2026-07-24, `1061d10` «переезд на лейны+роли, ADR-0059»)
- ✅ **qa выжила** — Глеб сохранил `iva-role-qa` в `ROLE_LANES`+`ROLE_PERSONA`. Условие соблюдено.
- 🎁 **qa УЖЕ в новой тест-матрице** — `_GENERIC_ROLES` (e2e, стр.811+) + smoke `ROLES`. Отложенный шаг 1(b) (e2e-wiring qa) #119 сделал за нас.
- ✅ **Ребейз чистый** — #119 НЕ трогал наши файлы лейна (`templates/iva-qa-autotest-base`/`iva-qa-mcp`/`iva-role-qa` — diff пуст).
- **Разблокирован полный канон Глеба локально:** после ребейза `feat/qa-codex-rework` на `1061d10` — Level 1 (`test_iva_role_presets`+`test_role_install_smoke`+`test_role_replacement_parity`) + Level 2 (e2e-голдены `test_install_flow_roles_generic[codex|claude-code-iva-role-qa]`, Docker). ⚠️ codex-добавки меняют состав → **регенерить golden `iva-role-qa`** + возможно поправить `present/absent` в `_GENERIC_ROLES`.

## R7 ЗАКРЫТ (2026-07-24) — ложная тревога, схема НЕ нужна
Механизм `agent_spec`→Codex в репо УЖЕ ратифицирован: **ADR-0025** + поле **`codex_body_path`** (рендерер `codex.py:100` эмитит `.codex/agents/<id>.toml`), используется в **11 профилях** (прецедент `brownfield-task-workflow`). Моя карта была неполной (смотрел go-лейн = только skill_spec). 3 субагента упакованы по этому паттерну. lead-arch — снят (президент: ведём сами).

## 3 замечания Глеба (в план приёмки/правок)
- маркер `instruction_pack` — коллизия при composе (покажет smoke, Уровень 1);
- связка команда→агент — пары субагентов в оракул (тест-матрица);
- **сузить `iva-qa-mcp` allowed_tools** до канарейки (иначе вся поверхность helm-analyst).

## Контакты
- Передача — целевой контакт **Брейкин (Никита)**; автор toolkit/kit — **Женя**; прод-выкатка — **Солонко**; схема/ADR — **lead-arch**; тест-матрица #119 — **Глеб**.

## Порядок доставки (решение президента 2026-07-24) — ТЕСТ ДО МЕРЖА
1. Доделать Codex-переделку (keystone → fix/batch/jira + rebuild-триггер + сузить `iva-qa-mcp`).
2. **Тщательно проверить работу** (статика + контролёр-гейт/verifier) — ДО пуша.
3. **Пуш** ветки `feat/qa-codex-rework`.
4. **Тест на стенде** `teststand` (Level 3, аккуратно) — прогон профиля 0.
5. Зелёный → **мердж через PR** (git-шаг президента). НЕ мержить непроверенное.

**lead-arch — снят с зависимости (решение президента):** R7 решаем сами, провизорную Codex-упаковку субагентов делаем финальной (механизм известен).

## Тест-стенд `teststand` (38.180.236.39) — пре-флайт (read-only, 2026-07-24)
- В ssh-manager как `teststand` (root/password). Тулчейн **под юзером `tacticum`**, НЕ root: `codex` (`/home/tacticum/.codex`, used), `serena` (uv tools). Прогоны Level-3 — под `tacticum` (`su - tacticum -c 'codex exec …'`).
- Пилоты уже были: `disk/int/kmp-pilot`, `kttest` (java/kmp/internal из гайда) — стенд рабочий.
- ⚠️ **Клона `one-web` нет** — для полного Level-3 QA нужен целевой репо (VPN до ИВА + git) либо пилот на подходящем репо; setup к моменту теста.

## Текущая работа
Ветка `feat/qa-codex-rework` (worktree, не запушено). **Чанк 1 ✅** (механика). **Keystone ✅** ([[impl-qa-codex-keystone]], коммиты до `0721ed2`): 3 субагента упакованы под Codex (agent_spec+`codex_body_path`, ADR-0025), эталон `spawn_agent` во write-autotest, тесты 73/73. Отревьюено — паттерн годный. **Финальный проход ✅** ([[impl-qa-codex-final]], коммиты до `c8b6d7f`): fix/batch/jira→spawn_agent (batch веер до max_threads=4), rebuild-триггер задокументирован (ручной/git-hook, PostToolUse на Codex нет), path-desync вычищен, `iva-qa-mcp` уже сужен (правок не потребовалось). Тесты 73/73.

**Ребейз на новый main ✅** (`1061d10` с #119) — чисто, 0 конфликтов, 11 коммитов. Полный канон Глеба теперь доступен.

**Верификация в работе** ([[verify-qa-codex-branch]]): Level 1 (presets+smoke+parity+schemas+version-discipline через `uv run`) + Level 2 (e2e-голдены, если Docker; иначе CI). Ожидаемо: дрейф golden `iva-role-qa` из-за codex-добавок → регенерить.

**Верификация ✅** ([[verify-qa-codex-branch]]): Level 0 (матрица) + **Level 1 зелёный 288/0/0** (presets 91/smoke 77/parity 82/schemas 38, version-discipline 46 clean, mirror-sync 62). Level 2 (e2e-голдены) — локально Docker нет → CI; прогноз: `[codex-iva-role-qa]` упадёт golden-дрейфом (наши codex-тела) → регенерить `golden/iva-role-qa/codex.json`.

**Контролёр-гейт** ([[gate-qa-codex-branch]]) — в работе (чистота/скоуп/секреты/AI-подписи/дисциплина).

## ХВОСТЫ до пуша/мержа (на решение президента)
1. **⚠️ Golden `codex.json` регенерить — НЕГДЕ (нужен Docker+Postgres).** Локально нет; **стенд `teststand` — тоже без Docker/psql** (проверено). e2e-harness (`E2E_INSTALL_REGEN_GOLDEN=1`) без docker не запустить. Нужен: docker-машина (президент/дев-бокс) для регена+коммита `golden/iva-role-qa/codex.json`, ИЛИ CI-джоба с регеном. Гейтит Level-2-green / CI / мердж.
2. **Оракул smoke под qa (Глеб #2) — ✅ ЗАКРЫТО** ([[impl-qa-smoke-oracle]]): `test_run_commands_have_their_agents` обобщён (data-driven `ORCHESTRATION_FAMILIES`), qa-связка `write/batch/fix→3 субагента` реально ассертится (MATCH True, не early-return). 77 passed. Ветка `feat/qa-smoke-oracle` (от origin/main, коммит `06c4d7b`, НЕ запушена) — правка тест-файла Глеба → отдельный PR + heads-up Глебу (shared infra).
3. **R7-FLAG** (`codex_body_path` на skill_spec, 4 оркестратора) — работает+валиден, принимаем как наше решение (ведём сами); зафиксировать в CHANGELOG/ADR-упоминании.

## Level-3 smoke на стенде — ✅ PASS (2026-07-24) — риск-часть доказана вживую
Материализовал 3 Codex-агента + config в `/home/tacticum/qa-pilot/.codex/` на `teststand`, прогнал codex-cli **0.142.3** под `tacticum` (`codex` = `/home/tacticum/.npm-global/bin/codex`, PATH `.bashrc:118`):
- ✅ **Дискавери:** `codex exec` перечислил 3 наших субагента (`code-writer, codebase-analyst, dom-explorer`) — упаковка `agent_spec`+`codex_body_path` (ADR-0025) РАБОТАЕТ на живом codex. Проектный `.codex/config.toml`+`.codex/agents/*.toml` подхватываются с `-C`.
- ✅ **`spawn_agent`→`wait_agent`→`close_agent`:** заспавнил codebase-analyst с задачей (прочитать probe.txt), вернул `SPAWN_OK 42` (трейс `collab: SpawnAgent/Wait/CloseAgent`). Механизм спавна из наших write/fix/batch-тел подтверждён.
- Модель оркестратора = gpt-5.5 (глоб.дефолт); субагенты — gpt-5.4 по нашим файлам.

- ✅ **Скиллы+агенты вместе:** `codex exec` перечислил SKILLS `write/fix/batch/jira-autotest` (наши 4) + AGENTS `codebase-analyst/dom-explorer/code-writer` (наши 3). Весь Codex-слой профиля виден на живом codex.

**Итог стенд-проверки (достижимое без one-web): ЗЕЛЁНО** — скиллы видны, агенты видны, spawn_agent работает. Материализовано вручную в `/home/tacticum/qa-pilot/` (3 агента + 4 codex-скилла + config).

**Осталось (Phase 3b):** полный цикл `write-autotest` end-to-end (реальный TC TestOps → сгенерированный тест) — нужен клон **one-web** + TestOps-креды + VPN до ИВА. Ядро (скиллы/субагенты/спавн) уже зелёное на стенде.

**Оракул влит в ветку переделки** (cherry-pick `359a620`) — 12 коммитов. Запушено, **PR СМЕРЖЕН президентом**.

## ДЕПЛОЙ (2026-07-24) — прод-каталог `tacticum_prod` (159.194.224.59)
Канон процедуры: Serena-заметка `deployment_prod_catalog_mcp` (деплой РУЧНОЙ по SSH, авто-workflow нет).
**R1 pre-flight (read-only, кол ГД 12:56) — ✅ ЧИСТО:** SELECT `profile_versions` в контейнере `tacticum-postgres-1` (БД `tacticum_catalog`) → **0 строк** по всем 3 профилям (`iva-role-qa`/`iva-qa-autotest-base`/`iva-qa-mcp`) — в проде их НЕТ. Сид пройдёт как `created`, коллизия версий невозможна. `installations iva-role-qa` = 0 → re-pin не нужен. Побочно: авто-CI-сида в этот каталог не было.
**Готовый gated-сид** (жмёт президент/Солонко; rebuild НЕ нужен — только контент `templates/`): pull `/opt/tacticum` от github main (как `tacticum-deploy`) → one-off seed `seed_community.py templates` → verify readyz+MCP-транспорт → wiki-sync. Версии: `iva-qa-autotest-base 0.2.0` · `iva-role-qa 0.5.0` · `iva-qa-mcp 0.1.0`.

### ДЕПЛОЙ ВЫПОЛНЕН ✅ (2026-07-24, добро президента) — ТОЧЕЧНЫЙ QA-сид
**Решение: только QA (3 профиля), НЕ весь #119.** Независимый контролёр-гейт ([[gate-qa-prod-seed]]) → GO условный; условия закрыты.
- **Пред-анализ (read-only):** прод `/opt/tacticum` на #134 (−40 от main); в диапазоне backend менял только `renderer.py` (merge-контракт #677, НЕ codex/сид), `seed_community.py`+схема НЕ менялись, миграций нет (БД 0037) → образ #134 сидит наши профили корректно, **rebuild и миграции НЕ нужны**. Сидер резолвит depends_on против БД (не папки).
- **Условия гейта:** R3 бэкап `pg_dump` снят (`/tmp/catalog_backup_20260724_102534.dump`, 11M); R4 сид-папка `/tmp/qa-seed` = ТОЛЬКО наши 3 (core-base убран — резолвится из БД); R2 public — консистентно (ВСЕ iva-* профили public).
- **Сид:** one-off контейнер (образ #134 + примонтированные сидер+сид-папка), `git archive origin/main` (не трогая `/opt/tacticum`). Результат: **все 3 `created`** (autotest 15 ing, mcp 2 ing, role 4 ing), two-pass, 0 ошибок.
- **Пост-verify ✅:** 3 профиля `active` в каталоге (верные версии+хэши); рёбра роли → core-base 0.1.1 + autotest 0.2.0 + mcp 0.1.0; `readyz` 200; MCP-транспорт 401 (здоров, регресса нет).
- **Остаточное:** R1 рендер-pull роли на #134 (закроется при первом pull); R5 wiki-sync (шаг 8, нужен `CONFL_TKN_SOL`); golden `codex.json` реген (docker).

### ПРОВИЖН ВЫПОЛНЕН ✅ (2026-07-24) — доступ QA-команде
Люди (решение президента): **Брейкин Никита** (`n.breykin@iva.ru`) + **Байрамбеков Евгений** (`e.bayrambekov@iva.ru`).
- **Модель токена:** per-person `membership_api_keys(org=IVA, user)` — НЕ per-профиль/команда. Токен резолвится в workspace орг → его installations.
- **Оба уже имеют активные токены** → новый НЕ нужен (Евгений подтверждённо работает в `base`: `iva-rn-brownfield … for Evgeny`).
- **Схема как у аналитиков:** installation роли в workspace `base` (`c9f28fcf…`), где живут все IVA-роли. Installation привязана к workspace (не юзеру) → одна запись на обоих.
- **Действие:** `POST /admin/installations` (admin-токен из env контейнера) → HTTP 201, installation `b258bb6b-f241-47b9-8b8d-3ec8ed04cdaa`, `iva-role-qa @ 0.5.0` в `base`, active. Валидация active-версии пройдена.
- **Итог:** оба видят роль своим существующим tacticum-mcp токеном → Codex-агенту «поставь профиль iva-role-qa» → whoami → pull. R1 закроется на первом pull (`sync_count` 0→1).

### 0.3.0 — сид ЛЕЙНА в прод ✅ + бамп РОЛИ (2026-07-24)
**Механика доставки (важно, `seed_profile.py:76`):** рёбра `depends_on` замораживаются на latest-active базы В МОМЕНТ сида роли. Роль `iva-role-qa 0.5.0` заморожена на лейн `0.2.0` → сид лейна 0.3.0 сам по себе команде НЕ доставляет.
- **Сид лейна 0.3.0 ✅** (бэкап `/tmp/catalog_backup_before_030_…`, 11M): `created`, аддитивно — 0.2.0 и 0.3.0 обе `active` (работающий 0.2.0 не тронут); readyz 200 / MCP 401. Ребро роли пока 0.2.0 (ожидаемо).
- **Бамп роли `0.5.0 → 0.5.1`** (ветка `chore/qa-role-bump-051`, запушена, **PR ждёт президента**): только version+CHANGELOG (ингредиенты роли не менялись) — чтобы её ребро переморозилось на лейн 0.3.0. Тесты 129 passed, version-discipline OK, чисто.
- **После мержа 0.5.1:** сид роли 0.5.1 (ребро→0.3.0) + **перепин инсталляции** `b258bb6b` на 0.5.1 → команда получает 0.3.0. Работающая инсталляция 0.5.0 при этом не ломается (0.5.0 остаётся active рядом).

### ДОСТАВКА 0.3.0 ЗАВЕРШЕНА ✅ (2026-07-24) — PR #139 смержен
- **Сид роли `iva-role-qa 0.5.1`** (бэкап `before_role051`): `created`; ребро переморожено → **лейн 0.3.0** (+ mcp 0.1.0, core 0.1.1). 0.5.0 и 0.5.1 обе active.
- **Перепин инсталляции** `b258bb6b` 0.5.0 → **0.5.1** (PATCH /admin/installations, HTTP 200).
- **Итоговая цепочка (сверено):** installation pin 0.5.1 → iva-qa-autotest-base **0.3.0**. readyz 200 / MCP 401.
- **Команда (Никита/Евгений)** на следующем pull/update получит профиль с лейном 0.3.0 (one-web-kmp/playwright-нейминг, без мёртвых _selenium-ref). Работающий 0.2.0/0.5.0 не тронут — откат мгновенный.
- **Бэкапы отката:** `/tmp/catalog_backup_before_030_*`, `before_role051_*` (по 11M).

**Тест-стенд:** тулчейн под `tacticum` (codex 0.142.3 в `.npm-global/bin`, serena в uv-tools); Docker/psql НЕТ (golden-реген негде).
