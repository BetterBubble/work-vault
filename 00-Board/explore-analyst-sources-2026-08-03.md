---
title: Разведка — какие источники реально есть у профиля системного аналитика
type: note
status: draft
created: 2026-08-03
tags:
- board
- analyst
- разведка
permalink: tacticum/00-board/explore-analyst-sources-2026-08-03
---

# Разведка: источники профиля системного аналитика

Репозиторий: `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`, вершина `b612fd9`
(`Merge pull request #199 from TacticumApps/feat/connect-ios-profile`). Все пути ниже —
относительно корня этого репозитория.

**Важная развилка терминологии.** «Профиль системного аналитика» в репозитории — это
ДВЕ разные сущности, и состав источников у них радикально разный:

- **`templates/iva-system-analyst`** — самостоятельный профиль (v0.2.2). **Скиллов у него
  НЕТ ни одного**, MCP два, помечен `superseded_by: iva-role-analyst`.
- **агент `system-analyst`** внутри `templates/iva-analysis-fr` → приезжает в роль
  `templates/iva-role-analyst` вместе с 19 скиллами и 6 MCP.

Какой из них применяла команда аналитики — по коду не устанавливается (см. раздел 5).

---

## 1. Состав пакета `templates/iva-system-analyst`

Манифест: `templates/iva-system-analyst/manifest.yaml`

| Поле | Значение | Строка |
|---|---|---|
| `profile_id` | `iva-system-analyst` | `manifest.yaml:4` |
| `version` | `0.2.2` | `manifest.yaml:6` |
| `deprecated` | `false` | `manifest.yaml:9` |
| `superseded_by` | `iva-role-analyst` (комментарий: «ADR-0059 §8 / ТЗ #752: профиль замещён ролью») | `manifest.yaml:10` |
| `depends_on` | **отсутствует** — композиции лейнов нет, профиль плоский | — |

Полный список файлов пакета (8 файлов):

```
templates/iva-system-analyst/CHANGELOG.md
templates/iva-system-analyst/manifest.yaml
templates/iva-system-analyst/ingredients/agents/system-analyst.md
templates/iva-system-analyst/ingredients/agents-codex/system-analyst.toml
templates/iva-system-analyst/ingredients/commands/prepare-tz.md
templates/iva-system-analyst/ingredients/repo-configs/claude-code/CLAUDE.md.fragment
templates/iva-system-analyst/ingredients/repo-configs/codex/AGENTS.md.fragment
templates/iva-system-analyst/ingredients/repo-configs/codex/config.toml.template
```

Ингредиенты (7 штук, `manifest.yaml:50-142`):

| kind | ingredient_id | Строка |
|---|---|---|
| `agent_spec` | `system-analyst` | `manifest.yaml:52-64` |
| `command_spec` | `prepare-tz` | `manifest.yaml:67-75` |
| `mcp_server_spec` | `tacticum-mcp` (`https://mcp.tacticum.dev/mcp`) | `manifest.yaml:81-91` |
| `mcp_server_spec` | `iva-mcp` (`https://mcp.tacticum.ru/iva-read/mcp`) | `manifest.yaml:98-108` |
| `instruction_pack` | `claude-md-fragment` | `manifest.yaml:111-120` |
| `instruction_pack` | `codex-agents-md` | `manifest.yaml:122-131` |
| `instruction_pack` | `codex-config-toml` | `manifest.yaml:133-142` |

**`skill_spec` в пакете — НОЛЬ.** Директории `ingredients/skills/` в
`templates/iva-system-analyst` не существует.

**Зеркал у пакета нет.** `templates/_mirrors.yaml` (весь файл, 112 строк) не содержит
`iva-system-analyst` ни как `owner`, ни как `mirror`.

### 1.1 Сравнение по скиллам с тремя другими пакетами

| Пакет | Скиллов | Поимённо |
|---|---|---|
| `iva-system-analyst` | **0** | — |
| `iva-fr-analyst` | 6 | `fr-authoring`, `api-contracts-discovery`, `data-model-analyzer`, `events-analyzer`, `design-system-discovery`, `mockup-authoring` (`iva-fr-analyst/manifest.yaml:53-127`) |
| `iva-analysis-fr` | 8 | те же 6 + `process-analysis-stage`, `process-arch-signoff` (`iva-analysis-fr/manifest.yaml:116-210`) |
| `iva-role-analyst` | 0 своих, **19 через `depends_on`** | см. таблицу 1.2 |

**Скиллы, которых у `iva-system-analyst` нет, а у остальных есть — это ВСЕ их скиллы**
(у него нет ни одного). Полный перечень отсутствующего у него, с владельцами:

- `fr-authoring`, `api-contracts-discovery`, `data-model-analyzer`, `events-analyzer`,
  `mockup-authoring`, `design-system-discovery` — есть в `iva-fr-analyst` и `iva-analysis-fr`;
- `process-analysis-stage`, `process-arch-signoff` — есть только в `iva-analysis-fr`
  (`tier: full`, `iva-analysis-fr/manifest.yaml:190-210`);
- `brd-authoring`, `adr-authoring`, `multi-container-pin-authoring`, `pin-api-verification`,
  `pin-upstream-dependency-check`, `tests-authoring` — `tacticum-analysis-core`
  (`tacticum-analysis-core/manifest.yaml:109-176`);
- `getting-started`, `kb-navigation`, `tacticum-context`, `conventional-git` —
  `tacticum-core-base` (`tacticum-core-base/manifest.yaml:62-106`);
- `research` — `tacticum-research-base` (`tacticum-research-base/manifest.yaml:70-80`).

Обратно: `iva-fr-analyst` **не содержит** ни агента `system-analyst`, ни команды
`prepare-tz` (в списке файлов пакета их нет). `iva-analysis-fr` содержит и агента, и
команду, и они **байт-в-байт совпадают** с версиями из `iva-system-analyst`
(проверено `diff` по трём файлам: `ingredients/agents/system-analyst.md`,
`ingredients/agents-codex/system-analyst.toml`, `ingredients/commands/prepare-tz.md` —
все IDENTICAL).

### 1.2 Что реально получает роль `iva-role-analyst`

`templates/iva-role-analyst/manifest.yaml:20-24` — композиция depth-1:

```yaml
depends_on:
  - tacticum-core-base
  - tacticum-analysis-core
  - iva-analysis-fr
  - tacticum-research-base
```

Своих ингредиентов у роли только 4 (3 instruction_pack + 1 repo_config,
`iva-role-analyst/manifest.yaml:76-119`), контент — из лейнов.

Итого по композиции: **19 скиллов, 1 агент (`system-analyst`), 1 агент
(`tacticum-workflow`), 6 команд, 6 MCP-серверов.**

MCP-серверы роли:

| MCP | URL / транспорт | Откуда приезжает |
|---|---|---|
| `context7` | `https://mcp.context7.com/mcp` | `tacticum-core-base/manifest.yaml:124-132` |
| `tacticum-mcp` | `https://mcp.tacticum.dev/mcp` | `tacticum-core-base/manifest.yaml:135-143` |
| `helm-analyst` | `https://helm.tacticum.ru/mcp/analyst` | `tacticum-analysis-core/manifest.yaml:180-190` |
| `iva-read` | `https://mcp.tacticum.ru/iva-read/mcp` | `tacticum-analysis-core/manifest.yaml:192-202` |
| `helm-process` | `https://helm.tacticum.ru/mcp/process/` | `iva-analysis-fr/manifest.yaml:217-235` |
| `iva-atlassian-write` | stdio `uvx mcp-atlassian` | `iva-analysis-fr/manifest.yaml:240-250` |

`helm-analyst` в `tacticum-analysis-core` объявлен **без `allowed_tools`**
(`tacticum-analysis-core/manifest.yaml:186-190` — только transport/url/env/auth), то есть
отдаётся полным набором тулов. Для сравнения: в `iva-analysis-fr` у `helm-process`
`allowed_tools` задан явно и содержит 7 имён (`iva-analysis-fr/manifest.yaml:228-235`),
а в QA-контуре helm-analyst урезан READ-срезом (`iva-role-qa/README.md:78`) — значит
отсутствие поля у аналитика это осознанная разница, а не забытая настройка.

---

## 2. Какие источники знания предусмотрены — по механизмам

Легенда: **SA** = пакет `iva-system-analyst`; **RA** = роль `iva-role-analyst`
(= core-base + analysis-core + analysis-fr + research-base).

### 2.1 MCP-инструменты

| Механизм | SA | RA | Ссылка |
|---|---|---|---|
| `tacticum-mcp` (`kb_discover`, `kb_get_task_context`, `kb_get_block_compact`, `kb_get_coverage`, `kb_get_nfr`, `kb_get_code_context`, `kb_verify_api_exists`, `whoami`) | **да** | **да** | SA: `iva-system-analyst/manifest.yaml:78-91`; RA: `tacticum-core-base/manifest.yaml:135-143` |
| `iva-mcp` / `iva-read` — Jira 10.3 + Confluence 9.2, только чтение | **да** (`iva-mcp`) | **да** (`iva-read`) | SA: `iva-system-analyst/manifest.yaml:93-108`; RA: `tacticum-analysis-core/manifest.yaml:192-202` |
| `helm-analyst` — вся факт-база аналитика | **НЕТ** | **да** | RA: `tacticum-analysis-core/manifest.yaml:180-190` |
| `helm-process` — BPMS-гейты требования | **НЕТ** | **да** | `iva-analysis-fr/manifest.yaml:217-235` |
| `iva-atlassian-write` — запись в Jira/Confluence | **НЕТ** | **да** | `iva-analysis-fr/manifest.yaml:240-250` |
| `context7` — доки библиотек | **НЕТ** | **да** | `tacticum-core-base/manifest.yaml:124-132` |

**Отдельно и важно:** даже в роли `iva-role-analyst`, где `helm-analyst` подключён,
**под-агент `system-analyst` его не видит**. Его frontmatter ограничивает инструменты:

```
templates/iva-analysis-fr/ingredients/agents/system-analyst.md:9
tools: Read, Write, Glob, Grep, Bash, mcp__tacticum-mcp__*, mcp__iva-mcp__*
```

`mcp__helm-analyst__*` в списке отсутствует. Это согласуется с предупреждением в двух
местах: `iva-analysis-fr/ingredients/skills/fr-authoring/SKILL.md:79-80` —

> ⚠️ **MCP-тулы зови только в собственном контексте.** НЕ запускай под-агентов
> (Agent tool) для MCP-вызовов — они не наследуют MCP-контекст и вернут пустоту.

и `iva-role-analyst/ingredients/repo-configs/claude-code/CLAUDE.md.fragment:19` —
«MCP-тулы — только в собственном контексте; под-агентам они не наследуются».

### 2.2 Поимённо по тулам helm-analyst: где они прописаны

Проверял поиском по `ingredients/` четырёх пакетов (SA, core-base, analysis-core,
analysis-fr, research-base). Столбец «SA» пуст везде — у SA нет ни `helm-analyst`,
ни скиллов, которые его зовут.

| Тул helm-analyst | Где прописан в инструкциях |
|---|---|
| `analyst_search` | `iva-analysis-fr`, `tacticum-research-base` |
| `analyst_context` | `iva-analysis-fr` (`fr-authoring/SKILL.md:64`, `:90`), `tacticum-research-base` |
| `docs_ask` | `iva-analysis-fr` (`fr-authoring/SKILL.md:74`), `tacticum-research-base` |
| `arch_map` | `tacticum-analysis-core`, `tacticum-research-base` |
| `arch_container` | `iva-analysis-fr` (`fr-authoring/SKILL.md:68`, `:114`), `tacticum-analysis-core`, `tacticum-research-base` |
| `arch_drift` | `iva-analysis-fr` (`fr-authoring/SKILL.md:69`, `:118`), `tacticum-analysis-core` |
| `affected_systems` | `iva-analysis-fr` (`fr-authoring/SKILL.md:67`, `:111`), `tacticum-analysis-core`, `tacticum-research-base` |
| `requirement_coverage` | `iva-analysis-fr` (`fr-authoring/SKILL.md:65`, `:92`), `tacticum-analysis-core` |
| `related_tasks` | `iva-analysis-fr` (`fr-authoring/SKILL.md:73`), `tacticum-analysis-core`, `tacticum-research-base` |
| `nearest_spec` | `iva-analysis-fr` (`fr-authoring/SKILL.md:70`, `:151`), `tacticum-analysis-core`, `tacticum-research-base` |
| `who_to_involve` | `iva-analysis-fr` (`fr-authoring/SKILL.md:71`, `:155`), `tacticum-analysis-core` |
| `effort_hint` | `iva-analysis-fr` (`fr-authoring/SKILL.md:72`, `:156`) |
| `gap_questions` | `iva-analysis-fr` (`fr-authoring/SKILL.md:66`, `:98`) |
| `constraints` | `tacticum-analysis-core` (`multi-container-pin-authoring/SKILL.md:46`, `tacticum-workflow.md:94,155,166,224,245,310`), `iva-analysis-fr` (`process-arch-signoff/SKILL.md:29,52`) |
| `contradiction_check` | `iva-analysis-fr` (`process-arch-signoff/SKILL.md:29`) |
| `requirement_tests` | `tacticum-analysis-core` |
| `api_registry_check` | **НИГДЕ.** Поиск `grep -rn "api_registry_check" templates/` по всему каталогу шаблонов даёт **0 совпадений** — ни в одном манифесте, скилле, агенте или README |

### 2.3 Навыки навигации по кодовой базе

| Навык | SA | RA | Кто владелец |
|---|---|---|---|
| `kb-navigation` | **НЕТ** | **да** | `tacticum-core-base/ingredients/skills/kb-navigation` (+ 7 brownfield/dev-пакетов) |
| `tacticum-context` (привязка installation) | **НЕТ** | **да** | `tacticum-core-base/manifest.yaml:86-93` |
| `codegraph-first-navigation` | **НЕТ** | **НЕТ** | только `iva-kmp-brownfield` и `iva-kmp-development-base` |
| `*-local-knowledge-routing` (10 штук: firebird, ucim, ios ×2, ivcs, kmp ×2, web ×2) | **НЕТ** | **НЕТ** | только стековые dev/brownfield-лейны |
| `research` (кросс-репо разведка без клона) | **НЕТ** | **да** | `tacticum-research-base/ingredients/skills/research/SKILL.md` |

То есть ни у пакета SA, ни у роли RA нет ни одного стекового навигационного навыка —
они живут только в лейнах разработки.

### 2.4 Доступ к вики/Jira и artifact-store

- **Чтение Jira/Confluence:** есть у обоих (`iva-mcp` / `iva-read`).
- **Запись в Jira/Confluence:** только у RA (`iva-atlassian-write`, личный PAT,
  `iva-analysis-fr/manifest.yaml:240-250`). У SA явный `non_goal`:
  `iva-system-analyst/manifest.yaml:161` — «Write access to Jira/Confluence (iva-mcp is read-only by design)».
- **Artifact-store:** упоминаний не найдено ни в одном из четырёх пакетов
  (см. раздел 5 «Чего я НЕ смог установить»).
- **Дизайн-система как источник:** `design_list_systems` / `design_get_theme_tokens`
  через tacticum-mcp — только RA, через скилл `design-system-discovery`
  (`fr-authoring/SKILL.md:76`).

### 2.5 Требование к клону репозитория — противоречие внутри роли

Два утверждения из одной композиции:

- Агент `system-analyst` **требует локальный клон** и работает из cwd:
  `iva-analysis-fr/ingredients/agents/system-analyst.md:16-18` — «(2) живой код
  репозитория (текущая рабочая директория — пользователь заранее склонировал репо
  и запустил сессию из его корня)»; `:43` — `git remote get-url origin`.
- Инструкция роли говорит **обратное**:
  `iva-role-analyst/ingredients/repo-configs/claude-code/CLAUDE.md.fragment:10-12` —
  «**Клона репозиториев нет и не нужно** — факты о коде берутся ТОЛЬКО из Tacticum KB
  (`kb_verify_api_exists`, `kb_get_code_context`). Не проси склонировать репо и не
  «вспоминай» код по памяти.»

То же в `fr-authoring/SKILL.md:28` («Клона репозиториев НЕТ и не нужно») и в правиле
честности №10, `fr-authoring/SKILL.md:765-766`: «**Код-факты — только из kb_*.** Не
«вспоминай» код по памяти модели и не проси никого «посмотреть в репозитории» — у
профиля нет клона намеренно.»

---

## 3. Что скиллы профиля говорят про работу «с нуля» / без референсов

### 3.1 Фраза из строк 29-30 `fr-authoring/SKILL.md` — на месте

Проверено: фраза присутствует, дословно, в двух файлах и **только** в них. Поиск по
всему репозиторию (`grep -rn "Тяжёлой генерации с нуля" . --exclude-dir=.git
--exclude-dir=.serena`) даёт ровно 2 совпадения:

```
templates/iva-analysis-fr/ingredients/skills/fr-authoring/SKILL.md:30
templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md:30
```

Дословно (строки 29-30, в обоих файлах идентично — файлы совпадают байт-в-байт,
`diff` = IDENTICAL; это зарегистрированная пара зеркал, `templates/_mirrors.yaml:19-26`,
owner `iva-analysis-fr` → mirror `iva-fr-analyst`, ингредиент `fr-authoring`):

> ```
> 29  - Скилл — «водитель»: оркеструет тулы и собирает из их данных черновик.
> 30    Тяжёлой генерации с нуля НЕ делает — только композиция фактов из тулов.
> ```

Происхождение: строка введена одним коммитом `f07548f` от 2026-07-16
(`feat(templates): FR pipeline MVP Ф1+Ф2 — iva-fr-analyst profile + FR input for
iva-kmp-brownfield (epic #682)`) и с тех пор **не менялась** — `git log -S` по этой
строке даёт ровно один коммит, он же первый и последний.

Соседняя формулировка того же ряда — в описании скилла, `fr-authoring/SKILL.md:10-11`:

> ```
> 10  отвеченных вопросов в «Зафиксированные решения» (/update-feature). Не пишет
> 11  финальную постановку за аналитика.
> ```

и правило честности №7, `fr-authoring/SKILL.md:759-760`:

> ```
> 759  7. **Скилл не пишет финал за аналитика.** Результат — черновик + вопросы;
> 760     решение и утверждение — за аналитиком.
> ```

### 3.2 Проектные разделы в том же файле — противоположная установка

В том же файле проектирование to-be разрешено явно и подробно.
`fr-authoring/SKILL.md:448-452`:

> ```
> 448  **Правила проектных разделов Части 1 (§1.6 Контракты, §1.7 Модель данных,
> 449  §1.8 События):** здесь проектные wire-имена to-be РАЗРЕШЕНЫ, но только под тремя
> 450  предохранителями §2 (основание в Части 2 → П.F/П.E · плашка «Предложение,
> 451  требует утверждения: разработчик + CTO» + `Q-n` · валидатор границы). Всё, что
> 452  не проектируется, а есть уже сейчас, остаётся фактом в Приложении.
> ```

`fr-authoring/SKILL.md:732-734` (правило честности №2, различение выдумки и проектирования):

> ```
> 732  - **Проектное предложение с основанием** — имя, которое аналитик
> 733    ПРОЕКТИРУЕТ для to-be, опираясь на факт из Части 2 (конвенция ближайшего
> 734    реестра / аналог / negative evidence «операции нет → проектируем»).
> ```

`fr-authoring/SKILL.md:778-779`:

> ```
> 778  утверждения + `Q-n` (§2). Это НЕ дефект: проектное имя с основанием — норма
> 779  проектного раздела, а не расхождение с фактами.
> ```

Итого: строка 30 говорит «тяжёлой генерации с нуля НЕ делает», строки 448-452 и 732-734
описывают именно проектирование новых имён/контрактов под предохранителями. Оба текста
живут в одном файле, оба действующие.

### 3.3 «Не с нуля» в скиллах-анализаторах — тот же мотив, другими словами

`iva-analysis-fr/ingredients/skills/data-model-analyzer/SKILL.md:125`:

> ```
> 125  2. **Дельта к существующему, не с нуля.** Новую модель проектируй как изменение
> ```

`data-model-analyzer/SKILL.md:49`:

> ```
> 49     проектируем как ДЕЛЬТУ к ней, не с нуля.
> ```

`iva-analysis-fr/ingredients/skills/events-analyzer/SKILL.md:112`:

> ```
> 112  2. **Дельта к существующему, не с нуля.** Конвенции имён событий/топиков наследуй
> ```

### 3.4 Механизм «нет опоры» у api-contracts-discovery — negative evidence

`iva-analysis-fr/ingredients/skills/api-contracts-discovery/SKILL.md:16` и `:79`:

> ```
> 16  **зафиксировать проверяемое отсутствие** (negative evidence) и конвенции для
> 79  ### Шаг 5. Не нашёл — оформи negative evidence
> ```

Это единственный скилл FR-контура, где исход «не нашлось» описан как отдельный
предписанный шаг с оформлением результата.

---

## 4. Есть ли явный сценарий «задача новая, референсов нет»

### 4.1 У пакета `iva-system-analyst` / агента `system-analyst`: сценарий есть, и он — СТОП

`iva-analysis-fr/ingredients/agents/system-analyst.md:85-112` (файл идентичен копии в
`iva-system-analyst`), фаза 1b «гейт наличия реализации»:

> ```
> 96   Возможные исходы:
> 97   - **Хотя бы один US имеет реализацию** (полную или частичную) → продолжай
> 98     с фазы 2; нереализованные US зафиксирует матрица фазы 3.
> 99   - **НИ ОДИН US не имеет реализации** → **СТОП. Файл ТЗ НЕ создавать,
> 100    фазы 2–5 НЕ выполнять.** Вместо документа верни пользователю короткий
> 101    отчёт в чате:
> 102    1. вердикт: «функционал из спеки не реализован в этом репозитории»;
> 103    2. доказательство поиска: какие запросы задавались KB, какие паттерны
> 104       и синонимы Grep'ались по коду (чтобы пользователь видел, что искали
> 105       добросовестно, а не сдались рано);
> 106    3. ближайшие соседние механизмы, если есть (похожая фича, из которой
> 107       видно, где новый функционал мог бы жить), — 1-2 строки, без разведки;
> 108    4. рекомендация: это greenfield-задача — идти в обычный tacticum workflow
> 109       от спеки, либо функционал реализован в другом репозитории (если git
> 110       remote / KB подсказывают, в каком — назови).
> 111    Ложный СТОП хуже лишней работы: прежде чем остановиться, убедись, что
> 112    прошёл все волны синонимов и проверил соседние блоки топологии.
> ```

Тот же гейт продублирован в команде `prepare-tz`
(`iva-system-analyst/ingredients/commands/prepare-tz.md:22-23`):

> ```
> 22  > Если гейт 1b показал, что функционал не реализован
> 23  > в репозитории, — остановись без создания ТЗ и верни отчёт о поиске.
> ```

и в инструкции профиля
(`iva-system-analyst/ingredients/repo-configs/claude-code/CLAUDE.md.fragment:11-13`):

> ```
> 11  - Если гейт фазы 1b показывает, что функционал из спеки в репозитории
> 12    не реализован, агент останавливается **без создания ТЗ** и возвращает
> 13    отчёт о поиске (запросы KB + Grep-паттерны) с рекомендацией greenfield-пути.
> ```

Итого: для профиля `iva-system-analyst` сценарий «функционала нет нигде» **предписан
явно** — и он предписывает не делать документ. Фолбэк — «идти в обычный tacticum
workflow», то есть в другой контур.

Другие жёсткие СТОПы того же агента: фаза A, репо не в KB — `system-analyst.md:51-53`
(«Локального fallback у KB нет»); фаза 0, iva-mcp недоступен — `system-analyst.md:63-64`.

### 4.2 У `fr-authoring`: отдельного сценария «новая задача, референсов нет» НЕТ

Что есть по факту:

- **Признак новизны читается, но веток от него не заводится.** `requirement_coverage`
  возвращает `is_new` (`fr-authoring/SKILL.md:65`), шаг 1 п.2 предписывает
  «Зафиксируй вердикт и `requirement_id` — дальше работай по ID»
  (`fr-authoring/SKILL.md:92-93`). Что делать иначе при `is_new=true` — в файле не
  сказано ни в одном месте (проверено поиском `is_new` по файлу: 2 совпадения, оба выше).
- **Шаг 5 «Прецедент» фолбэка не имеет.** Полный текст шага, `fr-authoring/SKILL.md:150-152`:

  > ```
  > 150  ### Шаг 5. Прецедент
  > 151  `nearest_spec(требование)` — возьми структуру и формулировки за основу и
  > 152  адаптируй; помечай «адаптировано из <ключ>». Ускоряет ФТТ/НФТ/дизайн.
  > ```

  Ветки «`nearest_spec` ничего не вернул» в шаге нет.
- **Деградации прописаны только для инфраструктурных отказов, не для «нет опоры».**
  Единственный явный блок деградации в потоке — `fr-authoring/SKILL.md:146-148`:

  > ```
  > 146  **Деградация:** KB-рана для целевого репо нет / kb_* недоступны → шаг пропусти
  > 147  с явным дисклеймером в FR: «код-verify не выполнялся: нет KB для <repo>»;
  > 148  вопрос остаётся открытым `Q-n`. Молчаливый пропуск запрещён.
  > ```
- **Общий фолбэк — «[уточнить]» и Q-реестр**, а не альтернативный источник.
  `fr-authoring/SKILL.md:722-723` (правило честности №1):

  > ```
  > 722  1. **Не выдумывать.** Каждый факт — с цитатой `[n]`. Чего тулы не дали — пиши
  > 723     «[уточнить у …]», не заполняй по догадке.
  > ```

  и `fr-authoring/SKILL.md:97` — шаг 2 назван «Gap-чек (лечит «пустое окно») → Q-реестр»;
  механизм: `gap_questions` → вопросы `Q-n` с адресатом, критичные — аналитику до
  черновика (`:98-104`).
- **Выбор жанра** (FR vs ТЗ, `fr-authoring/SKILL.md:404-418`) новизну как критерий не
  использует — критерии там объёмные: число подсистем-владельцев, этапы релиза,
  объём ФТ >15–20 позиций (`:414-415`).

### 4.3 Единственный контур, где «не знаем» — штатный вход: `research`

`tacticum-research-base/ingredients/skills/research/SKILL.md` (189 строк), приезжает в
`iva-role-analyst` через `depends_on` (`iva-role-analyst/manifest.yaml:24`), но **не**
в `iva-system-analyst`.

Из frontmatter (`research/SKILL.md`, строки 2-12): «Use for INVESTIGATING A QUESTION
without cloning a repo and without writing code … Possible outcome — "no code needed"».

Правило выбора лейна, `research/SKILL.md` (раздел «When this lane — and when NOT»):

> «Rule of thumb: if you can state the *answer* already, it is not research — pick a
> build lane. If the honest state is "we do not yet know / need to find out", this is
> the lane. A valid research outcome is **"no code needed"** — ending at the report is
> success, not a stall.»

Про отсутствие фактовых тулов там же есть явный фолбэк:

> «If they are **not** in the role — research runs from the **KB catalog** alone.
> Note the missing tool once and continue; never block on an absent tool.»

Но это лейн **исследования вопроса**, а не авторинга FR/ТЗ: `research` не входит ни в
поток `fr-authoring` (9 шагов, `:84-334`), ни в фазы агента `system-analyst`
(`system-analyst.md:41-274`) — перекрёстных ссылок между ними нет (проверено поиском
`research` по обоим файлам: упоминаний нет).

---

## 5. Чего я НЕ смог установить

1. **Какой именно профиль применяла команда аналитики.** В репозитории два разных
   носителя имени «системный аналитик» (пакет `iva-system-analyst` и агент
   `system-analyst` внутри `iva-analysis-fr`/`iva-role-analyst`), состав источников у
   них отличается радикально — 2 MCP и 0 скиллов против 6 MCP и 19 скиллов. По коду
   различить, что было применено, невозможно; нужен лог сессии или запись установки
   (`installation_id`, `.tacticum/context.yaml` рабочей папки аналитика).
2. **Что именно реально отдают MCP-серверы.** Я читал только объявления в манифестах и
   упоминания в скиллах. Фактический набор тулов `helm-analyst` и `tacticum-mcp` на
   проде, их работоспособность и наполнение индексов не проверял — это уже не разведка
   по репозиторию.
3. **Есть ли artifact-store у профиля.** Поиска по слову `artifact` я не проводил
   исчерпывающе; в манифестах четырёх релевантных пакетов ингредиента с таким
   назначением нет, но что понимается под «artifact-store» в постановке задачи, мне
   неизвестно, поэтому фиксирую как неустановленное, а не как «нет».
4. **Была ли строка 30 `fr-authoring/SKILL.md` осознанным решением или остатком.**
   Установлено только: введена коммитом `f07548f` 2026-07-16 вместе с первой версией
   профиля и ни разу не редактировалась. Формулировка «фраза от старой версии» из
   записей направления по git-истории не подтверждается и не опровергается — файл
   с самого начала содержал и её, и проектные разделы.
5. **Как ведут себя `tier: full` скиллы при реальной установке.** `process-analysis-stage`
   и `process-arch-signoff` объявлены `tier: full` (`iva-analysis-fr/manifest.yaml:192`,
   `:204`); ставится ли аналитику `trial` или `full`, по репозиторию не видно.
6. **CHANGELOG'и я не разбирал** — сверял только манифесты, скиллы, агентов и команды.
---

## Уточнение 03.08 — ответы на три вопроса лида

Вводная от лида (факт с прод-базы, не моя разведка): автор жалобы `d.lavrov@iva.ru`,
ключ `phk_94c046a0`, работал пакетом **`iva-system-analyst` 0.2.2** (30.07 — 79 вызовов
против 3 у роли; 31.07 — 21 против 3), установка роли создана только 31.07 в 12:05.
Ниже — ответы по коду на вершине `b612fd9`.

### В1. Гейт 1b — в ПАКЕТЕ или только в роли? Ответ: в пакете, и пакет — ПЕРВОИСТОЧНИК

Агент лежит в пакете: `templates/iva-system-analyst/ingredients/agents/system-analyst.md`
(файл в составе 8 файлов пакета, см. раздел 1). Гейт 1b в нём — строки 85-112, то есть
**правило применимо к сессии Лаврова напрямую**.

Направление копирования — обратное тому, что можно было предположить по путям:

| Файл | Первое появление | Последнее изменение |
|---|---|---|
| `iva-system-analyst/ingredients/agents/system-analyst.md` | `bd244f0`, **2026-07-10** («feat(profile): iva-system-analyst 0.1.0 — as-is research & TZ authoring + e2e pair») | `bd244f0`, **2026-07-10** — единственный коммит, файл не менялся ни разу |
| `iva-analysis-fr/ingredients/agents/system-analyst.md` | `511041a`, **2026-07-29** («feat(templates): роли — честное надмножество профилей (энейблер миграции, ТЗ #738)»), `--diff-filter=A`, +304 строки, файл создан | `511041a`, 2026-07-29 |

Версии **идентичны**: `md5` обоих файлов = `cb8c6aec45b008b69dcbbebc7bcfdf5e`
(и `diff` = IDENTICAL). То есть 29.07 копия была заведена в лейн ровно как копия
пакетной, разойтись они не успели. Так что моя прежняя ссылка на путь
`iva-analysis-fr/…` указывала на зеркало; оригинал — в пакете.

Гейт 1b существует в пакете **с самой первой версии 0.1.0**: `git log -S "НИ ОДИН US
не имеет реализации"` по пакетному файлу даёт ровно один коммит — `bd244f0` от
2026-07-10. Он же и вводит файл. То есть на 30.07 гейт был в пакете в том же виде,
в каком я его процитировал (раздел 4.1).

Гейт продублирован ещё в трёх местах пакета:

- `iva-system-analyst/ingredients/agents-codex/system-analyst.toml:79` («## Фаза 1b —
  гейт наличия реализации (обязательный, до тяжёлой разведки)»), `:93` («НИ ОДИН US не
  имеет реализации → СТОП. Файл ТЗ НЕ создавать,»), `:102` («рекомендация: это
  greenfield-задача — идти в обычный tacticum workflow»), `:265-266`; и в самом
  `description` агента, `:2` — «Stops without a TZ when the spec'd feature is absent
  from the repository (phase-1b gate)»;
- `iva-system-analyst/ingredients/commands/prepare-tz.md:22-23`;
- `iva-system-analyst/ingredients/repo-configs/claude-code/CLAUDE.md.fragment:11-13`
  и `.../codex/AGENTS.md.fragment` (тот же абзац, «отчёт о поиске (запросы KB +
  поисковые паттерны) с рекомендацией greenfield-пути»).

**Даты изменений всех 8 файлов пакета** (подтверждают «пакет не менялся с 10.07»
по содержанию, кроме двух точечных правок и манифеста):

```
CHANGELOG.md                                             2026-07-19  c5d313a
ingredients/agents-codex/system-analyst.toml             2026-07-10  1a9c707
ingredients/agents/system-analyst.md                     2026-07-10  bd244f0
ingredients/commands/prepare-tz.md                       2026-07-10  bd244f0
ingredients/repo-configs/claude-code/CLAUDE.md.fragment  2026-07-10  bd244f0
ingredients/repo-configs/codex/AGENTS.md.fragment        2026-07-10  1a9c707
ingredients/repo-configs/codex/config.toml.template      2026-07-19  c5d313a
manifest.yaml                                            2026-07-29  75ac945
```

Вся история пакета — 5 коммитов: `bd244f0` (10.07, 0.1.0), `1a9c707` (10.07, 0.2.0 —
Codex CLI support), `16d2e45` (15.07, codex config), `c5d313a` (19.07, iva-mcp →
iva-read alias), `75ac945` (29.07, `superseded_by` в манифесте). Ни один из них не
трогал ни агента, ни команду после 10.07.

### В2. Семь ингредиентов пакета 0.2.2 — поимённо

Ровно то, с чем работал человек (`iva-system-analyst/manifest.yaml:50-142`):

| # | ingredient_id | kind | Что это физически | Куда ставится | Строки манифеста |
|---|---|---|---|---|---|
| 1 | `system-analyst` | `agent_spec` | **Сабагент**. Тело — `ingredients/agents/system-analyst.md` (305 строк): фазы A → 0 → 1 → 1b → 2 (14 осей) → 3 → 4 → 5, скелет ТЗ, финальный линт. Codex-тело — `ingredients/agents-codex/system-analyst.toml` (`gpt-5.5`, reasoning high) | `.claude/agents/system-analyst.md` (scope **user**) / `.codex/agents/system-analyst.toml` | `52-64` |
| 2 | `prepare-tz` | `command_spec` | **Слэш-команда** `/prepare-tz <URL-или-Jira-ключ> [название US]`, 25 строк. Всё, что делает, — передаёт задачу сабагенту №1 | scope **repo** | `67-75` |
| 3 | `tacticum-mcp` | `mcp_server_spec` | **MCP-сервер**, http, `https://mcp.tacticum.dev/mcp`, Bearer `TACTICUM_TOKEN`. `body: ""` — своего контента нет | scope repo | `81-91` |
| 4 | `iva-mcp` | `mcp_server_spec` | **MCP-сервер**, http, `https://mcp.tacticum.ru/iva-read/mcp`, Bearer `TACTICUM_TOKEN`. `body: ""` | scope repo | `98-108` |
| 5 | `claude-md-fragment` | `instruction_pack` | **Фрагмент `CLAUDE.md`**, 18 строк, `merge_strategy: append_section`, маркер `tacticum:iva-system-analyst` | `CLAUDE.md` репозитория | `111-120` |
| 6 | `codex-agents-md` | `instruction_pack` | **Фрагмент `AGENTS.md`** (Codex-аналог №5), 21 строка, тот же маркер | `AGENTS.md` | `122-131` |
| 7 | `codex-config-toml` | `instruction_pack` | **Шаблон `.codex/config.toml`**, `merge_strategy: create_if_missing` | `.codex/config.toml` | `133-142` |

Итого содержательного текста, который что-то предписывает агенту: **один файл агента
(305 строк) + команда-обёртка (25 строк) + фрагмент инструкции (18 строк)**. Скиллов
(`skill_spec`) — ноль, директории `ingredients/skills/` в пакете не существует.
Ни `depends_on`, ни зеркал у пакета нет (`templates/_mirrors.yaml` его не содержит).

### В3. Инструменты kb_* — откуда они и предписан ли порядок

**Откуда.** `kb_get_code_context` и `kb_get_task_context` — тулы сервера `tacticum-mcp`,
который в пакете объявлен (ингредиент №3). Заявленный пакетом набор — в комментарии
манифеста, `iva-system-analyst/manifest.yaml:78-80`:

> ```
> 78  # tacticum-mcp: kb_* knowledge-base tools (kb_discover, kb_get_task_context,
> 79  # kb_get_block_compact, kb_get_coverage, kb_get_nfr, kb_get_code_context,
> 80  # kb_verify_api_exists) + whoami for installation resolution.
> ```

То есть **оба вызывавшихся тула заявлены пакетом явно**. По `iva-mcp` заявлено,
`manifest.yaml:93-97`:

> ```
> 93  # iva-mcp: read-only Jira 10.3 / Confluence 9.2 access inside the IVA perimeter
> 94  # via the platform MCP gateway (mcp-atlassian: jira_search, jira_get_issue,
> 95  # confluence_search, confluence_get_page, …). Authenticates with the SAME
> 96  # phk_* key as tacticum-mcp (forward-auth via project-hub), so applying the
> 97  # profile requires no additional user-side configuration.
> ```

Ни у одного из двух MCP-ингредиентов пакета **нет поля `allowed_tools`** (в обоих
только `transport`/`url`/`env_required`/`auth_type`) — то есть со стороны конфигурации
профиля набор тулов сервера ничем не урезан, агент видит всё, что отдаёт сервер.
Отдельно: `design_list_systems` / `design_get_theme_tokens` в пакете **не упоминаются
нигде** (`grep -rn "design_" templates/iva-system-analyst/` → 0 совпадений), при этом
и не запрещены — ограничения на тулы в пакете нет.

Доступ агента к тулам ограничен его собственным frontmatter,
`iva-system-analyst/ingredients/agents/system-analyst.md:9`:

> ```
> 9  tools: Read, Write, Glob, Grep, Bash, mcp__tacticum-mcp__*, mcp__iva-mcp__*
> ```

**Предписан ли порядок. Да — единственным носителем инструкций в пакете является сам
файл агента, и порядок kb_*-вызовов в нём расписан пофазно** (все ссылки —
`iva-system-analyst/ingredients/agents/system-analyst.md`):

| Строка | Предписание |
|---|---|
| `16` | «Источники: (1) Tacticum KB исследуемого репозитория (kb_* tools), (2) живой код репозитория …, (3) спека аналитика в Confluence и/или Jira (iva-mcp tools)» |
| `45` | Фаза A: `mcp__tacticum-mcp__whoami` → список installations |
| `46-50` | Фаза A: `kb_discover(installation_id)` → сопоставить `repo_url` с `git remote`, взять свежайший `finalized_at`, «все дальнейшие kb_* вызовы выполняй с ними явно» |
| `51-53` | Фаза A, СТОП: «Если совпадения нет — СТОП … Локального fallback у KB нет» |
| `57-64` | Фаза 0: спека через iva-mcp (`confluence_get_page` / `jira_get_issue`), СТОП при недоступности; «Не выдумывай содержимое спеки» |
| `75-77` | Фаза 1 п.1: `kb_get_task_context(kb_run_id, query_text=<функционал>, top_k=5)`, «Запросов несколько: по каждому US из спеки отдельно, синонимы, англ. термины» |
| `78-79` | Фаза 1 п.2: `kb_get_block_compact` по каждому релевантному блоку |
| `80-81` | Фаза 1 п.3: `kb_get_coverage(kb_run_id)` → code_only / doc_only |
| `82-83` | Фаза 1 п.4: `kb_get_nfr(kb_run_id)`; пустой ответ → NFR добываются из кода в фазе 2 |
| `85-112` | Фаза 1b: гейт наличия реализации (KB-хиты + целевой Grep, «2-3 волны переформулировок») → продолжать или СТОП |
| `117` | Фаза 2: `kb_get_code_context` / `kb_verify_api_exists` для проверки якорей, «битый якорь → найди актуальное место в коде и исправь ссылку» |
| `123-124` | Фаза 2: «Для US из спеки, на которые KB ничего не дал, — целевой Grep по репозиторию … «не нашлось в KB» ≠ «не реализовано»» |

Так что вызовы kb_* были не «по собственному разумению»: порядок, аргументы и условия
СТОПа прописаны в файле агента, который входит в пакет. Чего в пакете нет — это
**скилла**, который бы описывал работу с источниками отдельно от этого одного файла:
ни `kb-navigation`, ни `tacticum-context` (оба живут в `tacticum-core-base`,
в пакет не входят, см. раздел 2.3).

### Что это уточнение НЕ устанавливает

- Что именно происходило в сессии Лаврова — сработал ли гейт 1b, до какой фазы дошёл
  агент, что вернулось из kb_*. Лога по-прежнему нет.
- Почему в телеметрии есть `design_*`: в пакете этих тулов не предписывает ни одна
  строка, но и запрета нет. Откуда пришёл вызов — по репозиторию не устанавливается.
- Совпадает ли установленная у Лаврова версия 0.2.2 с состоянием `main` на `b612fd9`.
  Я сверял репозиторий; что физически лежало в его `.claude/agents/`, не проверял.
