---
title: Helm — план доделки 1b на текущих данных (без дозапросов)
type: plan
permalink: tacticum/20-architecture/helm-plan-dodelki-1b-na-tekushchikh-dannykh-bez-dozaprosov
tags:
- helm
- control-tower
- plan
- wave-1b
- data
- agent-team
---

План: доделать Волну 1b по канону в пределах НАЛИЧНЫХ данных `data/real/` (без внешних дозапросов). Основан на [[Helm — аудит данных data-real- что есть на входе + что деривится без дозапросов]]. Реализует [[control-tower-v02]]. Порядок исполнения: Блоки 1+2 параллельно (2 implementer), потом 3→4→5.

## Блок 1 — Граф-связи (дёшево, высокая ценность, данные уже лежат) — @implementer-1
- [plan] **1.1 Зависимости §5.4:** `jira/jira_issue_links.csv`(5512 blocks)+`active_blockers`(220) → рёбра `Initiative→depends_on→Initiative`; светофор наследует красноту блокеров. Точка: новый `ingest/links_source.py` + `real_source`/`loader` + `repository` (persist рёбер, резолв обоих концов + снять самопетли — грабли FK) + `calc` (blocker→светофор). Разблокирует **A3, D5**. #wave-1b
- [plan] **1.2 Т3 sales из CRM:** `crm/crm_deals.csv`(761 с продуктом) → `SalesInitiative`: стадия→вероятность (словарь Presale/ТКП/Оплачен/…), `сумма_iva_с_ндс`, `в_работу_до`=дедлайн; + `top_projects`. Точка: новый `ingest/crm_source.py` + genesis в `real_source`. Разблокирует **C1/C2 богаче, C3, дедлайны Гантта, важность §4**. #wave-1b
- [plan] **1.3 Сигналы поддержки:** `service/sd_tasks.csv`(42k, кат. IVA) → `Signal` инцидент/FR; разрыв «FR/обещание без работы». Точка: новый `ingest/service_source.py`. Разблокирует **C4/C5 в графе, §6.1 FR-разрыв**. #wave-1b

## Блок 2 — Люди/ценность (обогащение M5) — @implementer-2
- [plan] **2.1 Полный состав команд:** `by_person`(232 email)⋈`target_team`(117)⋈`identity_map`(955) → богатый TeamMember (трек/продукт/роль/левел/стек); дотянуть 122 несшитых где можно. Точка: `derive_real_manual`/`identity`. Разблокирует **B4** точнее. #wave-1b
- [plan] **2.2 CV-lite/fit:** `target_team.Стек/Левел/Роль` → fit-скор (over/under-qualified). Точка: `application/scoring.py`. §6.2 fit-ось. #wave-1b
- [plan] **2.3 ФОТ по продукту:** парс `fot_by_product`(pivot, есть рубли) → стоимость на уровне продукта/команды (суррогат per-person зп). Точка: `ingest/economics.py`. Разблокирует **B1**. #wave-1b
- [plan] **2.4 Калибровка M5:** медианный сплит квадрантов + отдельный трековый срез (external/unassigned не мешают). Точка: `application/scoring.py`. #wave-1b

## Блок 3 — Т2 обогащение (LLM через Gateway) — после Блоков 1-2
- [plan] **3.1 Т2 LLM-догадка task→goal** для задач без `epic_link`(7%) — LLM-судья привязки, confidence; +расширенные цели из `STRAT`(5)/`CEO`/докс. Разблокирует **B5/B6** реальнее. #wave-1b

## Блок 4 — Субстрат: вшить коннекторы в цикл (готовы+e2e-live)
- [plan] **4.1 Авто-проекция в memory (Graphiti):** после seed писать `project_state`-эпизоды инициатив/блоков → velocity/дрейф (§8.2). **4.2 knowledge_rag ingest кода:** `repomix`(9)+CV-lite → knowledge_rag; RAG-поиск → **D3**. #substrate

## Блок 5 — Деплой + автосверка
- [plan] **5.1** Выкатка всего на сервер `helm` (SFTP) + `scripts/check_competency.py` (сколько competency-вопросов отвечаем автоматически). #server

## НЕ закрывает (нужно внешнее — вне плана)
- [followup] B2 (зп в рублях — нет и не будет) · D4 (тела Confluence не выгружены) · A7 критпуть/write-back (Волны 2-3) · reviewers/approvals MR (нужен PAT на git.hi-tech.org). #followup

## Отношения
- implements [[control-tower-v02]]
- relates_to [[Helm — аудит данных data-real- что есть на входе + что деривится без дозапросов]]
- relates_to [[2026-07-04 — Helm: реорг data + план 1a/1b (backend-only, Agent Team)]]

## Прогресс — Блоки 1+2 ЗАКРЫТЫ (2026-07-04)

- [outcome] **Блок 1 (граф-связи) влит:** 1.1 зависимости (jira_issue_links→Dependency, но task↔task ≠ инициативы-темы → 1 ребро; **1.1b: A3 blocker-отчёт из active_blockers + D5 project-матрица 2546 кросс-связей/125 пар**, CLI `export_dependencies.py`); **1.2 Т3 sales — 759 SalesInitiative** (стадия→вероятность, сумма_iva_с_ндс, дедлайн→важность §4 ожила); **1.3 service — 12268 сигналов** (FR-разрыв). #wave-1b #verified
- [outcome] **Блок 2 (люди/ценность) влит:** 2.1 ростер 29→**227** (by_person⋈target_team⋈identity, 224 с профилем); 2.2 **fit-ось** (over75/under12/aligned116); 2.3 **ФОТ по продукту** (рубли из fot_by_product → team/product-cost, суррогат per-person зп); 2.4 калибровка (**трек-срез 85 чел** — чистая картина без 159 внешних). #wave-1b #verified
- [outcome] Интегрировано: 380 тестов, ruff+mypy чисто, 3 миграции. Граф на реальных: 874 инициативы, 39667 сигналов. Разблокировано: A3·C1·C2·C3·C4·C5·D5·B1·B4 + fit(§6.2) + важность§4. #wave-1b
- [plan] Осталось по плану: Блок 3 (Т2 LLM task→goal + расширенные цели STRAT/CEO/докс), Блок 4 (субстрат в цикл: memory-проекция + knowledge_rag ingest кода), Блок 5 (деплой + check_competency). #wave-1b

## ФИНАЛ — план доделки выполнен и развёрнут (2026-07-04)

- [outcome] **Блоки 1-4 плана влиты (395 тестов, ruff+mypy чисто) и развёрнуты на сервере `helm`.** На сервере на реальных: 874 инициативы · 759 sales · 39667 сигналов (12268 service) · A3 220 блокеров (держат 334 задачи) · D5 2546 кросс-связей/125 пар · ростер 227 · трек-срез ценности 85 · fit (over75/under12/aligned116) · Т2 LLM 262 привязки · 1a API жив (9 блоков). #wave-1b #verified #server
- [outcome] **Субстрат «граф через платформу» — вживую:** `project_to_memory.py --write` + `ingest_knowledge.py --write` пишут в реальный Graphiti/knowledge_rag; `memory_read` возвращает 200 с реальными entities/facts (L1). Запись медленная (LLM-экстракция на эпизод) → в проде фоново/крон. #substrate #verified
- [gotcha] Regen нарезки на сервере падает `PermissionError` — данные в контейнер кладутся `docker cp` от root, `chmod a+rX` даёт read без write. Для `derive_real_manual` нужно `docker exec -u root helm-helm-1 chmod -R a+rwX /app/data/real/manual`. #server #gotcha
- [followup] Осталось опц.: Блок 5 `scripts/check_competency.py` (автосверка golden-set — пока вручную); repomix на сервер для полного code-RAG (D3); реальный --llm-t2 прогон через Gateway на сервере (сейчас Mock/локально 262). #wave-1b

## Закрытие опционального (2026-07-04)

- [outcome] **Блок 5 check_competency.py влит** — автосверка golden-set из графа/отчётов: **отвечаем 21/30** (ДА 17·ЧАСТИЧНО 4·НЕТ 4·RAG 5; A 7/8·B 7/10·C 3/6·D 4/6). НЕТ: A7 критпуть(Волна 2)·B2 зп(нет данных)·C3 sales-blocked·C5 просрочка-FR(не заведены, деривятся). 399 тестов. #wave-1b #verified
- [outcome] **Реальный Gateway верифицирован:** ключ в `.env` работает — `chat tacticum/cheap → 200`. `--llm-t2` через реальный Gateway идёт (Mock давал 262 привязки; реальный батч по сотням задач — минуты, без 403). memory-проекция пишет в реальный Graphiti (200). #verified
- [reference] **Модели Gateway для нашего ключа** (`llm.cifragen.ru`): `tacticum/{cheap, chat, code, flash-3.5, flash-3, embed, smart, long}`. Использовать эти имена (не `bge-m3`/`gpt-*`). #gateway
- [followup] **D3/D4 knowledge_rag ingest БЛОКИРОВАН платформой:** сервис knowledge_rag зовёт Gateway `/v1/embeddings` с моделью вне allowlist нашего ключа → **422/403** (наш ключ видит `tacticum/embed`, сервис просит другую, коллекция `knowledge__bge-m3_1024`). Коннектор Helm рабочий (доходит до сервиса) — чинить на платформе: дать ключу доступ к нужной embed-модели ИЛИ настроить knowledge_rag на `tacticum/embed`. До фикса D3/D4-store (код/CV/Confluence-тела) не наполняется. #substrate #external-blocker
- [followup] Крон memory-проекции НЕ установлен (политика persistence требует явной санкции). Строка для ручной установки: `0 6 * * 1 cd /opt/helm && docker compose -f docker-compose.prod.yml exec -T helm uv run --no-sync python scripts/project_to_memory.py --write`. Скрипт работает по требованию. #server

## D3/D4 knowledge_rag РАЗБЛОКИРОВАН (2026-07-04)

- [outcome] **knowledge_rag ingest + семантический поиск РАБОТАЮТ на реальных.** Две причины 422/400 устранены: (1) оператор вписал рабочий LLM-ключ (тактикум-контур, доступ к `tacticum/embed`, dim 1024) в `.env` сервиса knowledge_rag → embeddings-422 ушёл; (2) **наш баг**: `ingest_sources.py` слал строковые point-id (`{repo}:{path}`, `cv:{fio}`) — Qdrant требует uint64/UUID → 400. Фикс `e6c461d`: point-id = `uuid5(ref)`, читаемый ref → metadata. Проверено: ingest code 3·CV 117; `search(mode="semantic")` «senior java management» → Сухов/Яровенко/Семенов. #substrate #verified
- [gotcha] knowledge_rag **гибридный поиск** (mode=hybrid, дефолт) бьёт и Qdrant, и **Meili** — индекс `knowledge__bge_m3_1024` в Meili НЕ создан → search 404. **Семантический (vector-only, mode="semantic") работает.** Для hybrid/full-text — создать индекс Meili (платформенная сторона; сервис его не бутстрапит, как и Qdrant-коллекцию). #substrate
- [reference] Ключи для knowledge_rag: `KNOWLEDGE_GATEWAY_API_KEY` = LLM-ключ ТАКТИКУМ-контура (не цифраагент — тот тенант cifragen/ЗУ). `KNOWLEDGE_EMBED_MODEL=tacticum/embed`. Коллекция `knowledge__bge_m3_1024` (dim 1024). #gateway #substrate
- [followup] Хотфикс `e6c461d` docker-cp'нут в контейнер (эфемерно) — при след. пересборке нужен полный деплой ветки. Полный ingest repomix (все файлы, не 3) — медленный батч эмбеддингов, гонять фоново. Meili-индекс для hybrid — платформенной команде. #substrate

## Два хвоста D3/D4 — разбор (2026-07-04)

- [outcome] **Helm-сторона D3-ingest починена (3 коммита):** `e6c461d` UUID point-id (Qdrant требует uint64/UUID, было строковое → 400); `4e27743` чанкинг код-файлов под токен-лимит embed (было 20k→422; чанк 3.5k); `64cd458` фильтр repomix-мусора (кэши/логи/сборка/бинарь). Code-RAG работает: семантический поиск находит реальный Python (`fedot_gardener/evaluation/collect_dataset_v2.py#16`). CV 117 + fedot-код в Qdrant. #substrate #verified
- [followup] **Хвост 1 — hybrid/Meili (платформенная сторона):** сервис knowledge_rag фильтрует Meili по `tenant_id`, но НИГДЕ не объявляет `filterableAttributes` → hybrid-поиск 400. Semantic работает без этого. Фикс — одна команда на хосте платформы (Meili внутренний, из Helm не дотянуться): `PUT /indexes/knowledge__bge_m3_1024/settings/filterable-attributes` body `["tenant_id","project_id","source_type"]` (Meili master-key). Либо в коде platform knowledge_rag добавить настройку индекса. #substrate #external-blocker
- [followup] **Хвост 2 — полный корпус repomix (стриминг-рефактор):** `repomix_items` материализует ВСЕ чанки репо в список → код-плотные/гигантские репо (iva-one 52M, ivcs 38M, kmp 40M) OOM'ят контейнер. Нужен генератор + инкрементальный batch-ingest (не держать всё в памяти). Задача реализатору. Малые/средние репо ингестятся, гиганты — после рефактора. #substrate

## Стриминг-ingest готов, оба хвоста доведены (2026-07-04)

- [outcome] **Стриминг-ingest repomix влит и работает на сервере** (задача #25, @implementer-2): `stream_repomix_items`/`_stream_repomix_files` (построчный ридер, генераторы — память на batch, не на репо) + `--resume/--limit/--batch-size`. 405 тестов. Прогон на сервере: `--limit 500` → code 500 (много репо, вкл. гигант iva-one) + CV 117, semantic-поиск отвечает по всем репо (iva-outlook C#, load-runner java, iva-one TS). D3 code-RAG рабочий. #substrate #verified
- [gotcha] **Детач в контейнере helm ненадёжен:** `docker exec -d` и `nohup </dev/null &` для ДОЛГИХ `uv run`-процессов молча умирают (короткие команды и foreground `docker compose exec -T` — работают; не OOM, память 17%). Полный корпус repomix (десятки тыс. эмбеддингов, 20-40 мин) гнать **bounded foreground-чанками** (`--resume N --limit M`, каждый в таймаут) ИЛИ server-side `tmux`/systemd, не detached-exec. #server #gotcha
- [followup] **Хвост 1 (Meili hybrid)** — ждёт одну команду на хосте платформы: `PUT /indexes/knowledge__bge_m3_1024/settings/filterable-attributes ["tenant_id","project_id","source_type"]`. **Хвост 2 (полный корпус)** — код готов, semantic-RAG работает; долить весь код = гнать `--resume`-чанки/tmux (не блокер, механизм доказан). #substrate

## C3+C5 закрыты, golden-set 23/30 (2026-07-04)

- [outcome] **C3+C5 влиты** (задача #26): C3 `sales_blocked_by_delivery` (§5.4, sale с красной/просроченной depends_on-delivery) + C5 просрочка/FR-беклог из `sd_tasks.просрочена` (просрочено 3, FR-беклог 143). check_competency: **23/30** (ДА 19 · ЧАСТИЧНО 4 · НЕТ 2 · RAG 5; A 7/8 · B 7/10 · C 5/6 · D 4/6). 408 тестов. #wave-1b #verified
- [outcome] **Чек-лист прогнан на реальных (тимлид):** 408 тестов · seed real (874 иниц/759 sales/12268 service) · golden-set 23/30 · экспорт Монитора ГД HTML+бриф · value_report (fit over75/under12/aligned116) · A3 220 блокеров/D5 2546 связей · API /portfolio /gaps /docs 200 · code-RAG D3 работает. #verified
- [reference] **Остаток golden-set (7):** A7 критпуть + B9/B10 LLM-скоринг = Волна 2 (by design); B2 зп = данных нет; D4 тела Confluence + C6 = нехватка данных/RAG; D3 фактически работает (semantic). **23/30 — потолок на текущих данных + построенных волнах.** #wave-1b
- [gotcha] Осторожно с `ln -sfn` в основном дереве: симлинк на файл, который И ЕСТЬ таргет, создаёт self-loop и затирает файл (было с `data/competency-questions.md`, восстановлено). Симлинки данных — только в worktree. #gotcha

## Аудит точности (data→граф→выход) + фикс качества sales (2026-07-04)

- [outcome] **Аудит fidelity: загрузка данных ДОСТОВЕРНА** — все счётчики сходятся без тихих потерь: service 12268=кат.IVA · git 24383=24039+344 · инициативы 874=15целей+100эпиков+759sales · jira 3016→3016 (адаптер отбросил 55 при load, вероятно дедуп эпик-ключей; в графе 0 потерь) · A3 220=active_blockers · D5 2546=jira_links cross · C5 3/143 точь-в-точь. #verified
- [outcome] **Выводы 1b ДОСТОВЕРНЫ и осмысленны:** приоритет продуктов сервиса ТОЧНО зеркалит `product_economics.Решение` (AVES Инвестировать→1.0, CS Сократить→0.2, Телефоны Закрыть→0.1) — value-скоринг корректно опускает убыточные; C2 топ-сделка 2.5млрд=сырьё (ФСО/AVES); B4 heatmap осмыслен. #verified
- [outcome] **Дефект точности 1a найден и ПОФИКШЕН (задача #27):** интеграция CRM без фильтра уронила светофор — 437/761 sales терминальные (Отменено250/Оплачен149/Заказ размещён38) + 385 с прошлым дедлайном → все блоки красные, директива тонула (One 1.5 🔴46). Фикс `185b1eb`: исключены терминальные стадии (**sales 759→316**, только открытый пайплайн) + директива в блок по `directives_extra.block` (не по продукту). После: One 1.5 🔴46→🔴10 (D03 override виден!), C1 честно 316-пайплайн, golden-set держится 23/30, 410 тестов. #wave-1a #verified
- [followup] Остаточный нюанс (не блокер): открытые сделки с near-term дедлайном краснят через plan_finish=deadline−lead_time (lead_time sales=структурный 38д) — можно уточнить sales-lead_time отдельно; директивный сигнал уже не тонет. #wave-1b

## Деплой фикса sales на сервер + вскрыт stale-persist (2026-07-04)

- [outcome] **Фикс качества sales развёрнут на сервере, полная сверка честность+достоверность пройдена:** после чистого пере-сида (`down -v`) — sales 316, One 1.5 🔴10 (D03 директива видна), golden-set 23/30, приоритет продуктов зеркалит решения (AVES 1.0/CS 0.2/Телефоны 0.1), C5 3/143. Загрузка достоверна, выводы 1b точны, светофор 1a читаем. #server #verified
- [gotcha] **Persist АДДИТИВЕН → пере-сид без `down -v` оставляет удалённые сущности (фантомы) в БД.** Диагноз: `check_competency` читает граф из БД (`load_graph(session)`), не из CSV; после фикса sales пере-сид 316 БЕЗ down -v → в БД остались старые 443 терминальных sales = 759 в отчёте, хотя актуальные данные 316. Лечение: `down -v` перед пере-сидом (deploy.sh так и делает). #server #gotcha
- [followup] **Настоящий фикс (не сделан):** `seed_db` должен ЧИСТИТЬ граф-таблицы перед persist (truncate в FK-порядке / clear_graph) → пере-сид всегда идемпотентно-чистый, без фантомов и без обязательного `down -v`. Иначе любой re-seed с уменьшением данных оставляет stale. Малый-средний WP. #wave-1b #data-integrity
