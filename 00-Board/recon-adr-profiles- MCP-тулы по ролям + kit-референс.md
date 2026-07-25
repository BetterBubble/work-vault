---
title: 'recon-adr-profiles: MCP-тулы по ролям + kit-референс'
type: note
permalink: tacticum/00-board/recon-adr-profiles-mcp-tuly-po-roliam-kit-referens
tags:
- board
- draft
- adr
---

# recon-adr-profiles: MCP-тулы по ролям + kit-референс

status: draft · role: explorer · для ADR «модель взаимодействия профилей» (lead-arch)
Ключевой вопрос: делить ли mcp-analyst (helm-analyst) на разные MCP под роли.

Источники: server-instructions helm-analyst (загружены в сессию) · `/Users/bubblemac/tacticum/analyst-MCP-описание.md` · `/Users/bubblemac/tacticum/ТЗ-RAG2-доработка-контракты.md` · vault `20-Architecture/RAG#2 analyst-MCP — канон дизайна` · `00-Board/explore-my-todo-helm-analyst` · `00-Board/explore-analyst-web-vs-mcp` · ADR-0057 (`tacticum-dev-qa-kit/docs/adr/0057-...`) · распакованный `kit-main.zip`.

## ЧАСТЬ A — инвентарь тулов helm-analyst (= mcp-analyst)

18 тулов зарегистрировано сейчас (`@mcp.tool()` в `src/helm/interface/mcp/analyst_server.py`; `test_all_tools_registered` фиксирует ровно 18); `my_todo` — 19-й, уже вшит (см. `impl-my-todo`, `gate-my-todo`). Все, кроме `docs_ask`, отдают структурные данные без прозы (прозу собирает агент). Auth: Bearer→project-hub `/resolve`→tenant-gate `iva`, fail-closed.

| Тул | Назначение (одна строка) | Источник данных |
|---|---|---|
| analyst_search | Поиск знаний Confluence/Jira с цитатами (обёртка `/api/rag2/search`) | RAG#2 |
| analyst_context | RAG-контекст-ответ по Confluence/Jira/helm (обёртка `/api/rag2/context`; ровно это отдаёт веб `/analyst`) | RAG#2 |
| docs_ask | Публичная документация ИВА, RAG#1 (единственный генеративный) | RAG#1 |
| arch_map | C4-топология, drill L1→L3 (контейнер, tech, repos, owner, риски) | ArchNode/ArchEdge |
| arch_container | Детали одного C4-контейнера: за что отвечает, связи, владелец | ArchNode |
| arch_drift | Детерминированный детект дрейфа заявленной C4-топологии vs сигналы из репо (НЕ вердикт — сигналы-кандидаты) | ArchNode + repo-сигналы |
| affected_systems | Затронутые требованием системы (req→компонент→ArchNode, вывод) | C4 + Requirement |
| requirement_coverage | Что уже реализовано и где (карта покрытия реестра, против дублей) | Requirement/realization |
| related_tasks | Похожие задачи/история Jira | Jira/EpicTask |
| nearest_spec | k ближайших готовых постановок/FR (целые документы-аналоги) | Confluence (FR) |
| gap_questions | Черновой чеклист полноты постановки (детерм. эвристика по скелету FR, не гарантия) | Скелет FR (канон) |
| constraints | Арх-ограничения и НФТ для требования — слой «почему нельзя» (ADR + НФТ) | Confluence ADR/НФТ |
| contradiction_check | КАНДИДАТЫ на противоречие требования принятым ADR/НФТ (никогда не вердикт; пусто ≠ «чисто») | Confluence ADR/НФТ |
| api_registry_check | Детерм. матч по реестрам REST/OpenAPI ИВА (есть ли операция под задачу; без LLM; negative-результат структурный) | OpenAPI-реестры |
| contract_check | Детерм. проверка серверного контракта (JUMP/distrohost+Eva) по `contract_registry_dir` — есть ли операция/сообщение (без LLM) | Реестр контрактов |
| requirement_tests | Покрытие требования автотестами + статус из Allure TestOps (сколько, зелёные ли, что сломано; без LLM) | Allure TestOps |
| who_to_involve | С кем согласовать: владельцы затронутых систем + недавние контрибьюторы похожих задач | C4-граф + Jira |
| effort_hint | СПРАВКА как «едут» похожие задачи (lead_time/active_days из changelog) — НЕ оценка трудоёмкости | EpicTask changelog |
| my_todo (19-й) | Персональный список задач актора: blocked / high-priority / ждут согласования (фильтр по principal.email) | EpicTask + RequirementApproval |

### iva-read / iva-write
В ТЕКУЩЕЙ сессии как MCP-тулы НЕ видны. Доступные MCP-серверы: Figma, context7, helm-analyst, serena, taiga, wiki-mcp. `iva-read`/`iva-write` фигурируют в ADR-0057 (лейн `analysis` владеет `helm-analyst` + `iva-read`; целевой `iva-write` с личным PAT — follow-up Taiga #712) и в канон-дизайне («готового ИВА Read MCP нет — строим»), но их тулы отсюда не инвентаризируемы. Явно: не могу перечислить их тулы — не подключены.

## Группировка тулов по роли-потребителю

Основной потребитель / вторичный / не нужен:

- **Универсальный слой (нужен всем ролям):** `analyst_search`, `analyst_context`, `docs_ask`, `requirement_coverage`, `who_to_involve`, `my_todo`. Это общая read-ткань; `my_todo` вообще role-agnostic (per-actor).
- **Аналитик (ядро, вторичный — редко кто ещё):** `nearest_spec` (только аналитик), `gap_questions` (только аналитик; QA — слабо), `related_tasks`, `affected_systems`, `effort_hint` (аналитик/лид — планирование). Это самый плотный, аналитик-специфичный кластер.
- **Архитектор (основной) + аналитик (вторичный):** `arch_map`, `arch_container`, `arch_drift` (арх-drift скорее архитектор/dev), `constraints`, `contradiction_check` (слой «почему нельзя» — аналитик и архитектор пополам).
- **Разработчик (основной/паритет):** `contract_check`, `api_registry_check` (импл: есть ли операция/контракт), `arch_*` (вторично), `requirement_coverage`, `docs_ask`. Импл-facing подмножество.
- **QA (основной):** `requirement_tests` — фактически ЕДИНСТВЕННЫЙ QA-специфичный тул (Allure TestOps). Плюс из универсального: `requirement_coverage`, `analyst_search`, `my_todo`. Аналитик-кластер (nearest_spec/gap_questions/constraints/effort_hint) QA НЕ нужен.

### Наблюдение под вопрос «делить/не делить» (фактура, не рекомендация)
- Кластеры пересекаются через общий универсальный слой (6 тулов): раздельные MCP-серверы под роли ДУБЛИРОВали бы этот слой (search/context/docs/coverage/who_to_involve/my_todo) в каждом сервере.
- QA-поверхность узкая (≈1 специфичный тул), dev/architect-поверхности — подмножества аналитической + арх-тулы; «чисто своих» тулов у не-аналитика мало.
- Т.е. водораздел проходит не по «серверам под роль», а по РОЛЕВОМУ СРЕЗУ тул-поверхности над одним сервером (аналог role-presets / lanes). Решение — за ответственной ролью.

## ЧАСТЬ B — kit-референс (kit-main.zip, ivaqa/kit), модель композиции

Zip распакован успешно. kit — кросс-провайдерный (Claude Code + Codex) marketplace-каталог модулей AQA. «0/1/2» в постановке ложится на три яруса kit (kit называет их иначе — Стандарт / Каталог / base), эквивалент лейн-модели ADR-0057 (core / lane / role):

- **Ярус 0 — Стандарт (base-модуль):** инвариантный «пол» — конвенции, таксономия задач/артефактов, контракт наблюдаемости (`base/CONTRACT.md`: `events.jsonl`/`meta.json`). Рендерится в репо потребителя тощей базой `.kit/` (генерат схемы + answers проекта + локальный файл конвенций), `--check` ловит отставание. Ядро живёт ВНЕ репо потребителя — в кэше платформы, обновления привозит платформа.
- **Ярус 1 — Каталог модулей (композируемые capability по рабочим петлям):** `craft` (автоматизация), `ship` (доставка), `track` (прогоны/разбор), `tms` (чтение TMS), `keep` (хозяйство), `audit` (встречное ревью), `atlas`/`vendor` (осевые), `dispatch` (кросс-вызов ролей). Ставится ТОЛЬКО выбранное («своя среда под проект»); осевые — лишь при соответствующей оси проекта.
- **Ярус 2 — Роли (тонкие leaf-пресеты):** роли живут в answers-файле проекта `[roles.*]` (carrier/tier/effort/permissions), человек берёт ОДНУ роль. Роль ссылается на модули + ИМЯ пресета радиуса. Радиусы — канон в `dispatch/presets.toml`, НЕ в answers.

### Что перенять (референс чистой композиции)
1. **Платформо-нейтральный канон + генерат-проекции.** Один источник (Agent Skills / `presets.toml`) → тонкие обёртки под провайдера (`.claude-plugin/` канон, `.codex-plugin/`+`.agents/` — генерат `render_codex.py`/`render_presets.py`, руками не правятся). Общая часть пишется один раз, специфика провайдера — механически проецируется.
2. **Разделение общее/специфичное по ярусам:** инварианты — в base (ярус 0, byte-identical), capability — в модулях (ярус 1, ставится по выбору), сборка под человека — тонкая роль (ярус 2). Единица авторинга (модуль/лейн) ≠ единица установки (роль).
3. **Радиус разрешений как отдельный конфиг-первичный слой (`presets.toml`): 3 пресета** — `writer-full` (repo, acceptEdits), `scout-recon` (read + запись только в дом задачи `.tasks/**`, без Edit), `reviewer-readonly` (read-only, без Write/Edit). Answers хранит ТОЛЬКО имя пресета; радиусы — канон модуля; проекции `claude.*`/`codex.*` в одном файле. Прямой шаблон для «инструменты/MCP на разных слоях»: тул-доступ роли = пресет над общей поверхностью, а не отдельный сервер.
4. **MCP/инструменты подключаются на ярусе модуля/лейна, не роли.** В ADR-0057 (наш аналог) `helm-analyst` + `iva-read` принадлежат лейну `analysis`; роль их наследует через `depends_on`. То же в kit: инструментарий — в модуле, роль лишь композирует.
5. **Радиус асимметричен по провайдерам честно** (пример `reviewer-readonly.codex`: sandbox=read-only режет сеть → фаза сетевого ревью деградирует, зафиксировано как skipped, не blocked). Референс: модель профилей допускает провайдер-специфичную деградацию возможностей без слома контракта.

## Файлы (абсолютные)
- `/Users/bubblemac/tacticum/helm/src/helm/interface/mcp/analyst_server.py` — MCP-сервер helm-analyst, все тулы.
- `/Users/bubblemac/tacticum/analyst-MCP-описание.md` — канон описаний тулов.
- `/Users/bubblemac/tacticum/ТЗ-RAG2-доработка-контракты.md` — Р-1 api_registry_check, Р-4 contract_check.
- `/Users/bubblemac/tacticum/tacticum-dev-qa-kit/docs/adr/0057-capability-layers-and-role-presets.md` — наша лейн/роль-модель.
- kit-main распакован: `.../scratchpad/kit/kit-main/` — `README.md`, `CLAUDE.md`, `dispatch/presets.toml`, `dispatch/references/role-rule.md`, `base/`.
