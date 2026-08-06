---
title: recon-kit-full-qa-dorabotka
type: note
permalink: tacticum/00-board/recon-kit-full-qa-dorabotka-1
status: draft
lead: kit-scout
tags:
- recon
- kit
- qa
- profiles
- subagents
- lead-qa
- aqa
archived-at: 2026-08-03 11:16
---

# Разведка kit (полная) + доработка QA-профиля

**Когда:** 2026-07-23 ~11:40 · **Кто:** kit-scout → director + lead-qa
**Источник:** распакованный архив `kit-main` (149 файлов, marketplace v1.5.1, репо создан 2026-07-19, `git.hi-tech.org/ivaqa/kit`). Читано напрямую — не со слов. Это подтверждённый **upstream** наших скиллов.

---

## 1. Структура / маркетплейс / деплой / обновления

**kit = marketplace-репо плагинов сразу под две платформы** (Claude Code + Codex). Три слоя: **стандарт** (базовый модуль + доки) · **каталог из 10 модулей** по рабочим петлям · **base** (рендер тощей базы в репо потребителя).

**Канон = Claude Code-формат.** `.claude-plugin/marketplace.json` (список модулей) + у каждого модуля `.claude-plugin/plugin.json` + `skills/<имя>/SKILL.md` (платформо-нейтральный Agent Skills). Codex-обёртки (`.agents/plugins/marketplace.json`, `<модуль>/.codex-plugin/plugin.json`) — **генерат** `scripts/render_codex.py` (руками не правятся, `--check` ловит дрейф). При соседстве Codex берёт нативный манифест, CC видит только свой.

**10 модулей:** `base` (стандарт+рендер базы) · `tms` (read-only чтение Allure TestOps) · `craft` (изготовление автотестов — наш главный интерес) · `ship` (MR/land) · `track` (прогоны/разбор) · `keep` (хозяйство/ретро/refactor) · `audit` (встречное ревью) · `atlas` (осевой: living-AUT-map) · `vendor` (осевой: пересборка vendored-библиотеки, первый hook-модуль) · `dispatch` (кросс-вызов ролей Codex↔Claude). Осевые (`atlas`/`vendor`) ставятся только при соответствующей оси проекта, остальные — по выбору.

**Деплой на потребителя** = base рендерит каталог `.kit/` из 3 файлов: `schema.md` (генерат конвенций, детерминирован без дат — `--check` честно сравнивает байты) + `answers.toml` (единственный носитель параметров проекта) + `local.md` (местные конвенции). **Ядро модулей живёт вне репо потребителя — в кэше платформы**; в репо проекта только тощая база `.kit/`.

**Каналы:** `main` (latest, полигон — первый потребитель `one-web-kmp`, отделён от one-web) / `stable` (парк потребителей, промоушен main→stable — ручное действие владельца) / пин тегом (заморозка). Канал зашивается в маркетплейс при `add` навсегда. **Автообновления нет** — только явные команды: `base:update` (скрипт `update_kit.py` гоняет CLI обеих платформ) + `base:check` (детект отставания через `git ls-remote`).

**Фидбек-петля потребитель→ядро** = скилл `base:contribute` (`deliver_feedback.py` — доставка по GitLab-issue + `inventory_local.py` — инвентарь зрелых локальных наработок).

**Наблюдаемость** = контракт base v1 (`base/CONTRACT.md`): `events.jsonl` + `meta.json`, дома задач `.tasks/work/<ключ>/` (CLI `task_meta.py`); потребитель — `ivaqa/pulse`.

---

## 2. Три профиля ёмкости `[capacity].profile` (0/1/2)

**ПОДТВЕРЖДЕНО.** Claude — аддитивный слой поверх Codex-only базы. Определение — `dispatch/references/role-rule.md` + `base/skills/install/SKILL.md` + `answers.template.toml`. Профиль выбирается **человеком в анкете `base:install`** (одно поле `[capacity].profile`); смена подписки = правка этого поля + шаг 2а (карта ролей) + перерендер.

Провайдер-нейтральная таксономия ролей: `pipeline_writer` (конвейер — по готовым паттернам), `fog_writer` (туман — без паттерна), `scout` (разведчик), `reviewer` (встречный ревьюер, «counter» = провайдер, противоположный писателю).

| Профиль | Смысл | Карта носителей (`[roles.*].carrier`) |
|---|---|---|
| **0 — Codex-only** | подписки Claude нет, база полноценна без него | все роли `codex` |
| **1 — CC-ограниченный** | Claude есть, но дорогой/лимитный → только в самой ценной на токен роли | `reviewer=claude`, остальные `codex` |
| **2 — CC-полный** | Claude без жёстких лимитов | `fog_writer=claude`, `scout=claude`, `pipeline_writer=codex`, `reviewer=counter` |

**Правило деградации:** базовый носитель КАЖДОЙ роли — Codex; при недоступности Claude (квота/сбой) роль тихо откатывается на Codex с событием `degraded` — процесс не встаёт. Ревьюер на профиле 1 — единственная роль, где кросс-провайдерность незаменима (нужен независимый взгляд другого провайдера).

Вызов встречной роли — **только** через обёртку `invoke_role` (`dispatch:invoke`); руками `claude -p`/`codex exec` не собирает никто. Глубина кросс-вызова = 1. Радиус роли = пресет разрешений (см. §5).

---

## 3. Три субагента — ЕСТЬ, defs полные (`craft/agents/`)

Все три на месте — это ровно то, что разблокирует наши 3 заблокированных скилла.

- **`codebase-analyst.md`** — `tools: Read, Glob, Grep`. Read-only анализатор тест-кодобазы. Режимы: `write` (инвентарь page-слоя/flows/хелперов/i18n/enum + пробелы для craft:write) · `fix` (факты для классификатора падения) · `adhoc` (произвольный вопрос о кодобазе). **На диск не пишет вообще** — артефакт `analysis.md` возвращает текстом финального сообщения, файл в `.tasks/` пишет оркестратор (обход эвристики CC «report file», блокирующей Write субагента в имена вида `analysis.md`).
- **`dom-explorer.md`** — `tools: Bash, Read, Write, Glob, Grep`. **Единственный субагент с браузером.** Разведка живого UI, подбор адресации по стратегии стека. Режимы write/fix/adhoc. Пишет `locators.md` в дом задачи. Знает preflight носителя Codex (headless Chromium не стартует в sandbox macOS → штатная эскалация или явный выход, НЕ диагностировать как мёртвый стенд), изоляцию через эфемерных research-юзеров (`research_user_cmd`), инвариант «любой видимый текст — через i18n».
- **`code-writer.md`** — `tools: Read, Edit, Write, Glob, Grep, Bash`. Генератор кода тест-слоя строго по `analysis.md` + `locators.md`. Режимы write (добавляет методы/flows/i18n/enum) / fix (правит по failure-категории: DOM_CHANGED / UX_FLOW_CHANGED / API_CONTRACT_CHANGED / FLAKY_TIMING / IMPORT_OR_REGISTRY / PRODUCT_BUG / UNKNOWN). Самопроверка `py_compile` перед возвратом (Bash выдан ТОЛЬКО для этого). **Никогда не редактирует vendored-библиотеку.** Запрет слепых таймаутов, комментарии только предметные (не про агент-инфру — код уезжает в общий репо).

**Как забрать к нам:** это стек-нейтральный канон; проектная специфика вынесена в параметры `[craft]` (`stack`/`shared_users`/`research_user_cmd`/`vendored_dep`/`product_clone`) и в `stacks/<stack>/` (у нас — `pytest-playwright-canvas`). Пути к стеку в defs — через `$CRAFT` (`${CLAUDE_PLUGIN_ROOT}` в CC). Забирать не тремя голыми файлами, а **вместе с их контрактом**: (а) файлы стека `craft/stacks/pytest-playwright-canvas/` (recon.md, rules/, test-templates.md, failure-taxonomy.md — на них ссылаются все три дефа); (б) параметры `[craft]` в answers; (в) оркестраторы craft:write/fix/batch, которые их спавнят и пишут их артефакты в `.tasks/`. Голые defs без стека и оркестраторов работать не будут.

---

## 4. Diff: 9 скиллов нашего лейна vs kit

Наши 9 скиллов `iva-qa-autotest-base` = byte-copy из этого источника → kit — их upstream. Маппинг на текущий kit:

| Наш скилл (one-web) | kit-модуль:скилл | Статус в kit |
|---|---|---|
| write/batch-autotest | `craft:write` + **`craft:batch`** (оркестратор 5+) | kit **разделил** на write (1 TC) и batch (набор); добавлен `craft:issue` (задача трекера→автотесты→ланч) |
| fix-failed-test | `craft:fix` | новее: явная таксономия падений (7 категорий), референс-стеки |
| playwright-cli (разведка UI) | субагент `craft:dom-explorer` + `stacks/*/recon.md` | переосмыслено в субагента |
| run-tests | `track:run` | вынесено в отдельный модуль track |
| jira-issue-autotest | `craft:issue` | новее |
| prepare-mr-branch | `ship:mr` (+ `ship:land` — сведение ворктри) | вынесено в модуль ship, добавлен land |
| rebuild-autocore | `vendor:rebuild` | обобщено: vendored-dep как ось, `[craft].vendored_dep` вместо хардкода `autocore`; первый hook-модуль |
| retro | `keep:retro` | + `keep:dashboard`, `keep:refactor` |
| (update — ресинк TC) | `craft:update` | новый роутинг metadata/minor/major/removed |

**Что добавилось сверх наших 9:** модуль `tms:read` (read-only порт Allure TestOps — низкоуровневый, зовётся всеми), `audit:review` (встречное ревью субагентом work-reviewer), `atlas` (living-AUT-map), `dispatch` (кросс-вызов ролей — механизм профилей). Ключевое: **три субагента, которых в нашем источнике не было** — теперь есть.

**Архитектурный сдвиг:** наш лейн — плоские 9 скиллов, жёстко на one-web (autocore/venv/tools.testops/glab/CI зашиты). kit — **трёхслойная модель** (стек-нейтральный канон / референс-реализация стека `stacks/` / экземпляр проекта в `[craft]`+CLAUDE.md) + кросс-платформенность (pathlib, явный utf-8, Python не bash). Это закрывает наш codex-гэп (наш лейн заявлял codex:full, а был best-effort на Claude-специфике).

---

## 5. Безопасность — env-модель ПОДТВЕРЖДЕНА, чище нашей копии

Принцип kit: **«схема в git, значения вне git».** `tms/scripts/testops/client.py`: `resolve_settings()` берёт `TESTOPS_ENDPOINT`/`TESTOPS_TOKEN`/`TESTOPS_PROJECT_ID` из **env → gitignored `secrets.yaml`** (секция `testops:`, форма — `secrets.yaml.example`) → при отсутствии `TestOpsError` с инструкцией. Обмен api-token на короткоживущий JWT (`POST /api/uaa/oauth/token`, `grant_type=apitoken`), дальше `Authorization: Bearer`. **Никаких кредов/хостов в коде.** Наша копия — грязнее: в references зашиты `allure.iva.ru` и токен в `tools/testops/client.py`.

**Две линии санитизации, что перенять:**
1. **Секреты рантайма** (§выше): env `TESTOPS_*` → gitignored `secrets.yaml`, из корня репо. Убрать из нашей копии хардкод endpoint/token.
2. **Санитизация артефактов** (`craft/skills/write/references/sanitization.md`): в `.tasks/`-артефактах URL→`{stand_url}`, креды→`shared_users`, генерируемый контент→плейсхолдеры `<generate: ...>`; в **коде теста** — фабрика юзеров/фикстуры, генератор случайных строк, i18n. Инвариант: «в артефактах — санитизировано, в коде — фикстуры и генераторы».
3. **Радиусы ролей** (`dispatch/presets.toml`): три пресета — `writer-full` (repo, acceptEdits/workspace-write), `scout-recon` (task-home, read-only + `Write(.tasks/**)`, `deny Edit`), `reviewer-readonly` (истинно read-only, без Write/Edit). Провайдер-нейтральный канон с проекциями на флаги CC/Codex. **Прямая параллель нашей роли explorer** — read-only-разведка с записью только в дом задачи.

---

## ДОРАБОТКА QA-профиля — конкретные шаги для lead-qa

1. **Забрать 3 субагента с контрактом (разблокирует write/batch/fix).** Взять `craft/agents/{codebase-analyst,dom-explorer,code-writer}.md` НЕ голыми, а комплектом: + `craft/stacks/pytest-playwright-canvas/` (recon.md, rules/, test-templates.md, failure-taxonomy.md, allure-raw-parser.md, fix-playbooks.md) + оркестраторы `craft:write`/`craft:fix`/`craft:batch` + параметры `[craft]` в answers. Проверить, что наши 3 скилла спавнят субагентов через Task с теми же именами артефактов (`analysis.md`/`locators.md` в `.tasks/work/tc-{id}/`).

2. **Санитизировать нашу копию (секреты).** Вырезать хардкод `allure.iva.ru` и токена из `tools/testops/client.py` наших references → перейти на env-модель kit (`TESTOPS_*` → gitignored `secrets.yaml`, `secrets.yaml.example` как форма). Добавить `secrets.yaml` в `.gitignore`. Это прямой байт-перенос из `tms/scripts/testops/client.py`.

3. **Решить стратегию отношений с upstream.** Сейчас наш лейн — разовая byte-copy (форк, дрейфует). Варианты: (а) принять kit как upstream через канал `stable` + `base:install`/`base:update`, наш one-web становится потребителем (`[craft].stack=pytest-playwright-canvas`); (б) продолжить форк, но с явной точкой синхронизации. Рекомендация к обсуждению — (а): закрывает codex-гэп (кросс-платформенность + профили 0/1/2), даёт фидбек-петлю `base:contribute`, снимает ручную пересадку. Нужен доступ к `git.hi-tech.org/ivaqa/kit` (PAT read_repository / деплой-ключ).

4. **(доп.) Смапить наши роль-пресеты на профили `[capacity]` 0/1/2** — согласовать нашу модель роль-пресетов (ADR-0057) с кросс-провайдерной картой ролей kit; определить, на каком профиле живёт наш one-web (по наличию Claude-подписки).

---

## Сигналы

- [ ] to:director from:kit-scout Разбор kit полный (архив прочитан): marketplace CC+Codex, 10 модулей, профили 0/1/2 подтверждены, 3 субагента ЕСТЬ (defs в craft/agents/), env-модель секретов чище нашей. Топ-доработка: забрать 3 субагента с контрактом стека → разблокировать write/batch/fix + санитизация секретов. Детали [[recon-kit-full-qa-dorabotka]]
- [ ] to:lead-qa from:director взять разбор+доработку [[recon-kit-full-qa-dorabotka]]

## Связано
[[recon-aqa-kit-zhenya]] · [[Направление- Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА]] · [[qa-profile-model — опись + мульти-стэк модель QA-лейнов]]