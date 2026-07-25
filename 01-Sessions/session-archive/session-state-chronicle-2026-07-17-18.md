---
title: session-state-chronicle (2026-07-17→18)
type: note
permalink: tacticum/01-sessions/session-archive/session-state-chronicle-2026-07-17-18-1
status: archived
updated: 2026-07-18
tags:
- session
- checkpoint
- rag2
- analyst-mcp
- bot
- support-concept
- helm
---

## session-state (чекпойнт)

Обновлён **2026-07-17**. Направление — **analyst-MCP (RAG#2) для аналитиков ИВА** + смежное (бот поддержки, концепт support-MCP). Прод: `helm.tacticum.ru`, healthy.

## Следующий шаг (активное)
**Доработка analyst-MCP** — новая сессия. Промпт для неё готов: `~/tacticum/mcp-доработка-промпт-новой-сессии.md` (роль тимлида, команда/агенты, git-правила, деплой helm). Контекст самой доработки пользователь даст в новой сессии.

## Сделано в этой сессии (2026-07-16→17)
1. **Тест analyst-MCP (14 тулов) — пройден.** Оценка `~/tacticum/analyst-MCP-оценка.md`: реальные данные, честность (кандидаты≠вердикт, null≠баг, not_found, ACL без IS/~), латентность ок. 4.7/5.
2. **Cross-rerank A/B.** Per-corpus реранк включён (флаги `HELM_*_RERANK_ENABLED=1`). **Cross-corpus** (`HELM_RAG2_CROSS_RERANK_ENABLED`) — A/B +17% nDCG@10 на семантике, НО регресс на helm-корпусе; тюнинг (обогащение+rank-floor) **провалился** (0/34). **Флаг OFF, не мержено** (ветка `fix/rag2-cross-rerank-helm-tune`). Данные: `00-Board/cross-rerank-*`.
3. **Бот «Поддержка» — задеплоен, работает end2end.** Webhook `helm.tacticum.ru/api/bot/support/webhook`: платформа IVA (`iva-uc.ru`) → webhook → `docs_ask` (RAG#1) → send-message в чат. Двухконтур (только публичная дока наружу, allowlist `https://iva.ru/` fail-closed). Секрет hmac `Iva-Bot-Api-Secret-Token`, токен `X-Iva-Bot-Api-Token` в env. Серия хотфиксов формата (PR #61–66): текст в blocks, чистка markdown (`![]`/ссылки/`###`), множественные цитаты `[3, 6]`, дедуп источников по title, plain-режим docs_ask, **корень пустых citations** (`select_cited_chunks` в `domain/docs.py` не парсил `[2,5]`). Токены 4 ботов в `~/tacticum/.secrets/bots.env`. Детали: `20-Architecture/bot-podderzhka-webhook-rag-1-v-chat-iva-zadeploen`.
4. **Deploy-урок:** сервер `helm` БЕЗ git-доступа → bundle с локали + `SEED=0 bash scripts/deploy.sh` (REBUILD). **volume-mount НЕнадёжен** (код не грузился) — только rebuild. **ВСЕГДА verify `getsource`.** Классификатор блокирует прод-секреты/exec без авторизации человека.
5. **Triva tool-calling** — наш прокси прозрачен, tool-calling не отдаёт апстрим ИВА `10.0.196.14:9034` (chat-only façade). Ждёт `host:port` tool-capable инстанса от Dmitrii.
6. **Support-MCP концепт — финализирован** (отложен, дорабатываем позже): `helm/docs/iva-support-mcp-concept.md`. Процесс оператора (8 шагов), выходной контракт (3 выхода + 8 полей, needs_info), триаж-гейт, репомикс/code_context в диагностику. Разборы: `00-Board/support-*-DISCOVERY`.

## Отложено / открытое
- **Support-MCP концепт** — доработка позже: UX-режим (сжатие vs уточнение продукта), порядок Волны 2, индексация `d10task_sd`.
- **Cross-rerank** — если возвращаться: source-aware пул (helm мимо кросс-энкодера) вместо провалившегося обогащения; иначе off.
- **Triva** — адрес tool-capable эндпоинта от Dmitrii.
- **Бот** — остальные 3 бота по мере готовности; ускорение деплоя (deploy-key) — опция.

## Ключевые операционные факты
- Сервер helm: ssh-manager `helm` (159.194.233.33). `/opt/helm` git-репо без GitHub-доступа. Деплой = bundle→ff-merge→`SEED=0 deploy.sh`→verify getsource. Env `/opt/helm/.env`.
- Git: push только feature-ветку явно; PR не создаём (материалы→юзер мержит); без AI-подписей; не в main.
- Команда: делегировать, plan-approval, общение SendMessage+заметки, implementer=тип claude для интерактива.

## Ссылки
- memory://tacticum/20-architecture/bot-podderzhka-webhook-rag-1-v-chat-iva-zadeploen
- memory://tacticum/20-architecture/team-lead-playbook
- memory://tacticum/20-architecture/agent-team-design
- `~/tacticum/mcp-доработка-промпт-новой-сессии.md` (промпт новой сессии)
- `helm/docs/iva-support-mcp-concept.md` (концепт support-MCP)


## Апдейт 2026-07-17 (сессия доработки RAG#2, до старта работ)
- **База выверена: работаем от `main`.** Локальный `main` = `origin/main` = прод `/opt/helm` = `b5d1739` (14 тулов). Локальный checkout стоял на устаревшей `feat/analyst-mcp-server` (−64 коммита) — переведён на `main`.
- **Cross-rerank — переоценка (прежняя формулировка «провалили тюнинг 0/34» НЕточна):** причина регресса на helm найдена (helm матчит по обогащённому embed-тексту, кросс-энкодер судит по обеднённому `payload.text` → недооценка). Фикс (source-aware `helm_rerank_text` + rank-floor, юнит-тесты зелёные) **был незакоммичен в worktree — теперь закоммичен** в `fix/rag2-cross-rerank-helm-tune` (+ `ab-kit/`: patch/driver/analyze/RUNBOOK). Остаток = **живой A/B на проде** (за прод-гейтом, нужен человек). Не «провал», а «фикс готов, валидация заблокирована доступом».
- **Наведён порядок в git-хозяйстве helm:** worktree 58→1, удалено 74 слитых ветки. Спасено в git (было бы потеряно): cross-rerank-фикс, docs-концепты (ветка `docs/iva-analyst-concepts`, 130KB), `mr_source.py` с прода (`salvage/mr-source-prod`), 5 агентских фич вне RAG#2 (`salvage/*`: task-hygiene+alembic, operator-review, conformance-ui×2, redact). Правила зафиксированы в [[git-deploy-hygiene-helm]].
- **Eva разложена на 3 источника:** EVA-трекер `eva.iva.ru` (ingest-код есть — `eva_source.py`/`migration/wave1b-eva`, но в управленческий контур, не RAG#2), Eva-wiki `orionpro` (greenfield), distrohost JUMP-контракты (greenfield = Р-4). Пользователь: по Еве уже что-то выгружено на helm — нужен прод-аудит полноты (трек D0).
- **Открытый прод-дрейф:** `mr_source.py` всё ещё лежит на `/opt/helm` вне git (в git спасён) — удаление с прода отложено до отдельного ОК.

## Ссылки (доп.)
- memory://tacticum/20-architecture/git-deploy-hygiene-helm


## Апдейт 2026-07-17 (ход доработки RAG#2 — вехи)
**Доставка/данные:**
- Туннель `iva-contour-sources-tunnel` создан на helm (через adp-jump/adp_emb, ничего на adp_emb не менять) → beta.hi-tech.org/distrohost/eva доступны, ингест на helm. См. [[iva-contour-access-helm]].
- Реальный корпус — Qdrant на `10.16.0.19` (6 коллекций), не в `data/real` CSV. IVAONEHALF в векторном корпусе ОТСУТСТВУЕТ (0), в scope/графе есть — не было в выгрузке. Манифест обновлён. См. [[rag2-corpus-map]].

**Треки (все ветки от b5d1739, ребейзить на 58008da перед мержем; конфликты config/service/analyst_server разрешать сохраняя оба):**
- **Р-1 `api_registry_check`** — детерминированный тул по OpenAPI-реестрам. **СМЕРЖЕН в main (PR #67, main=58008da).** Деплой — в пачке.
- **Р-2a confidence+порог** — готов (ветка `feat/rag2-confidence-threshold`, 4 коммита + сериализация hits). Floor по confidence; активация после калибровки τ (нужен negative-set).
- **exact-key boost** — пин готов (`feat/rag2-exact-key-boost`), НО A/B показал: ключ вне пула → расширяем на **retrieval-дотяжку** (lookup по key).
- **Р-3 split+join** query-side за флагом (`feat/rag2-query-expand-code`) — eva-prod-audit кодит. Корень (Meili separatorTokens=[]) подтверждён эмпирикой.
- **cross-rerank+A2 source-aware** (`fix/rag2-cross-rerank-helm-tune`) — в пачку как floor-база (реальные скоры 0.72-0.97 vs вырожденные 0.016). helm-регресс лечит A2.

**Ключевая находка (baseline за гейтом, read-only):** retrieval почти нулевой (recall@10=0.11 на 9 кейсах). **Корень — retrieval-MISS (даже точный issue-key не в пуле k=30), НЕ «шум скоров» из ТЗ и НЕ burial реранка.** cross-rerank recall НЕ чинит (переупорядочивает пул). Приоритет развёрнут на retrieval-side.

**Стратегия деплоя:** пачкой (Р-1+cross-rerank+exact-key+Р-3+Р-2a одним rebuild), прод осознанно отстаёт (b5d1739) до готовности пачки. Контракт: воркеры в worktree → PR по готовности+доказательству → пользователь мержит → лид деплоит с ОК. Правила git/деплой — [[git-deploy-hygiene-helm]].


## Консолидация 2026-07-17 (пользователь: меньше параллельности, больше качества)
**Scope текущего захода сужен до: `key-routed дотяжка` + `Р-1-фикс скоринга`.** Довести качественно (прод-валидация на живых данных) → мерж → деплой пачкой, затем стоп.

**Активны (2 трека + verifier):** `impl-p2a` (дотяжка по key, реализует), `eva-prod-audit` (Р-1-фикс скоринга на реальных 419 операциях), `eval-baseline` (валидация дотяжки на реальном Qdrant + помощь Р-1). standby-агенты `recon-rag2-tz`/`audit-meili-tokenize` — освобождены (работа сдана).

**Отложено (код/наработки сохранены в ветках/заметках, вернуться следующим заходом):**
- Р-3 split+join (`feat/rag2-query-expand-code`, код готов, Meili-валидация свёрнута)
- cross-rerank+A2 (`fix/rag2-cross-rerank-helm-tune`, прод-валидирован A/B)
- Р-2a confidence+floor (`feat/rag2-confidence-threshold`, floor-калибровка τ нужна negative-set)
- hybrid ре-баланс ft-веса (семантический retrieval-корень #2), golden-расширение, Р-4/Eva-контракты (distrohost+eva-wiki), Р-5 effort_hint changelog

**Контроль-режим:** каждая фича доказывается на реальных данных перед мержем (см. [[git-deploy-hygiene-helm]] контроль-гейт). Р-1 уже поймал дефект прод-валидацией (баг подстроки mail⊂email, приёмка «отозвать письмо» проваливалась).


## Режим работы (уточнение пользователя 2026-07-17)
После текущих двух (дотяжка + Р-1-фикс) — **строго последовательно, ОДИН трек за раз**, не параллельно. Цикл на трек: код → прод-валидация на живых данных → ревью лида → ребейз на актуальный main → мерж (с ОК пользователя) → следующий. Меньше токенов, больше качества/времени.

**Предполагаемый порядок следующих (уточнить по ходу):** Р-3 (готов, прод-валидирован — быстрый мерж) → cross-rerank+A2 (прод-валидирован) → Р-2a floor (калибровка τ) → hybrid ре-баланс ft-веса (семантика, нужен golden-расширение) → Р-4/Eva-контракты → Р-5 effort_hint.


## Выводы доработки RAG#2 (2026-07-17, A/B-замер на реальном Qdrant)
- **Главный recall-фикс — exact-key дотяжка** (issue-key в запросе → прямой Qdrant key-lookup + пин): recall@10 0.11→0.93. В проде, дефолт ON. Корень боли был retrieval-miss (точный тикет не попадал в пул из-за размывания NL-запросом), НЕ «шум скоров» из ТЗ.
- **hybrid ре-баланс ft-веса и cross-rerank — recall НЕ дают** (доказано A/B на 45-golden): снижение ft роняет recall (ft-канал net-полезен); cross recall не меняет + helm-регресс. Код в main, дефолт **off, не включать**.
- **floor τ=0.7 (по confidence 0..1, не RRF-score) — модест −17% шума, recall цел.** Единственный из «доработок скоринга», что даёт чистый плюс. Включать после re-validate на дотяжка-базе.
- **Р-1 (api_registry) и Р-4 (JUMP contract_registry) — детерминированные реестры** (без вектора/LLM), приёмка на реальных данных: «отозвать письмо»→not_found (chat-API) / messageRevoke (JUMP). Скоринг: концепт-дедуп + границы токена + конъюнкция действие+объект. Прод-валидация на реальных спеках обязательна — фикстуры скрыли дефекты (подстрока mail⊂email).
- **Р-5 корень — полнота changelog 4.6%** (extract гонялся узко под IVAONEHALF/velocity), не query-time. Фикс — broad-extract (оба проекта, все статусы) + переингест.
- **Урок замера:** мержить код с флагом off безопасно, но **включать флаг только с A/B-замером на реальном корпусе** (гипотезы отсеиваются данными — «ft вредит» оказалась неверна на полном наборе).


## Отложено (2026-07-17)
- **Eva-wiki (orionpro, вторая часть Р-4)** — ОТЛОЖЕНА. distrohost/JUMP (первая часть Р-4) закрыл боль пилота («отзыв письма»→messageRevoke). Eva-wiki требует session-login (не basic-auth) — нужны креды/cookie от пользователя. Вернуться отдельной под-задачей позже.


## Деплой RAG#2-пачки (2026-07-17) — СДЕЛАН
Прод helm `1399787`→`9774cab` (7 PR: Р-3/volume/hybrid/cross/Р-5/Р-4/floor). rebuild+verify getsource пройден, MCP жив. **Флаги в проде (по A/B-замеру):** floor `NOISE_FLOOR=0.5 drop` ON, дотяжка `exact_key_boost` ON, **hybrid ft_weight=1.0 (off), cross_rerank=off** (замер: recall не дают). api-корпус (Р-1, 410 операций) на volume — persist, тул вживую работает («отозвать письмо»→not_found). Прод-git подтягивается pull'ом ТОЛЬКО под `tacticum-deploy` (deploy-key), не root. Новый туннель-порт → ufw allow bridge (8444 api, 8081 contract).
**Хвосты:** contract-volume (Р-4 persist, compose-PR) + Р-5 changelog bulk-extract (нужен Jira PAT+авторизация).


## Р-4 contract — В ПРОДЕ НАПОЛНЕН (2026-07-17)
contract-volume merged (`5d4c6c2`), rebuild, contract-ингест на volume: **101 команда JUMP** (`data/real/contracts/JUMP.commands.json` persist). Тул `contract_check` вживую: «отзыв письма в JUMP»→messageRevoke, «удалить сообщение»→messageDelete, негативы вне JUMP→not_found. Р-4 закрыт.
**Осталось только Р-5** changelog bulk-extract: образец разведан — `scripts/extract_jira_changelog.py` stdlib(urllib), PAT из `/opt/iva-mcp/env` (mcp-atlassian), запуск на хосте `sudo python3 ... --env /opt/iva-mcp/env`. **Туннель 8443 (jira) сейчас FAIL — надо поднять iva-sources-tunnel.** Нужна авторизация юзера.


## Р-5 root-cause + план пересборки (разведка r5-and-datamap, 2026-07-17)
Полный отчёт: [[rag2-r5-rebuild-and-datamap]] (+ scratchpad r5_datamap.md, сырьё 00-Board/explore-r5-changelog-recon-raw).
- **Реконструкция из Qdrant iva_jira работает** (доказано): IVAONE-1175→active_days=20.9, -3803→7.3, -1→0.0(открыта). Overlap чанков=150 симв → склей `\n`+дедуп (ts,from,to). PAT/extract НЕ нужны.
- **Root-cause в SQL:** таблица `epic_task`=8073 строк, changelog только у **377** (старый velocity-дамп), и он **date-only** (без времени → внутридневные длительности потеряны). Два дефекта: покрытие 377/8073 + date-only.
- **effort_hint читает `EpicTask.changelog` из SQL, НЕ из Qdrant helm_mgmt.** Путь наполнения: скрипт-реконструкция пишет `data/iva/tasks_changelog_v2/` → `ingest_epic_tasks.py --dir tasks_rich,tasks_changelog_v2` → пишет ТОЛЬКО таблицу epic_task (iva_jira/RAG-корпуса не трогаются). Старый tasks_changelog в --dir НЕ передавать.
- **Карта данных:** все 6 Qdrant-коллекций рабочие, дублей нет. На чистку помечено 9 групп (data/_superseded, *.tgz×2, пустой tasks/, _wiki_extract, .DS_Store, прод vitrines/*.bak ~100М, прод /tmp 963М, .env-бэкапы[секреты!]). Чистка — ОТЛОЖЕНА, после показа.


## Р-5 записан в прод (2026-07-17) + 2-й код-дефект найден
Хирургический UPDATE `epic_task.changelog` выполнен (тимлид, ssh-manager, транзакция BEGIN…COMMIT): охват **377→6676**, все **8073 строки целы** (0 потерь). Бэкап `/opt/helm/data/backup_epic_task_2026-07-17.sql` (8.3MB). Данные верны: эталоны совпали (push 24.8/138.7, экспорт 23.8/151.7, IVAONE-1175→20.9).
**НО живой effort_hint нужен код-фикс:** `_search_epic_by_terms` делает `LIMIT 20` БЕЗ `ORDER BY` → после MVCC-UPDATE покрытые строки ушли в конец heap, seq-scan берёт непокрытые → null несмотря на 87% покрытие. Доказано: `ORDER BY key LIMIT 20`→16/20. Фикс #5 (1 строка, ORDER BY + предпочесть непустой changelog) дослан в code-fixes. Р-5 зелёный «из коробки» — ПОСЛЕ этого фикса в проде.
IVAONEHALF (~450) в Р-5 не покрыт (нет в iva_jira) — отдельный трек ivaonehalf-ingest.


## Решение: EVA-трекер НЕ ингестить в analyst-корпус (2026-07-17)
Анализ [[eva-tracker-relevance]]: EVA-трекер (eva.iva.ru, `data/real/eva/*.csv`, 6635 задач, source="eva") для analyst-MCP/ТЗ — **НЕ нужен**. Причины: `related_tasks` фильтрует `source=="jira"` (EVA by design исключён); `affected_systems`/`contradiction_check` EVA не потребляют; ТЗ не упоминает; source=eva=0 в Qdrant ничего не ломает. EVA ценна управленческому контуру (coverage_rate, дашборды, §5.4-зависимости) — он УЖЕ кормится offline CSV, вектор не нужен. Стратегически EVA сворачивается (переход на Jira). **Вывод: не ингестить — это extra scope без потребителя.**
**Разграничение:** «ева» из ТЗ = **eva-wiki** (orionpro HTML-доки, Р-4 вторая часть, ждёт orionpro-creds), НЕ EVA-трекер. RAG2-роутер `_EVA`-интент = live-метрики дашборда, тоже не task-корпус.


## Тайминг деплоя код-фиксов (решение пользователя, 2026-07-17)
Ветка `fix/rag2-audit-fixes` (5 фиксов: floor×pin, weak+weak, tz, ORDER BY, парсер-тест) запушена, ждёт создания PR + мержа пользователем. **Деплой (rebuild прода) — ТОЛЬКО ПОСЛЕ завершения перемера `wide-remeasure`.** Причина: фикс floor×pin меняет floor-поведение (pinned exact-key иммунны к drop) → преждевременный деплой исказит текущий замер floor. Порядок: перемер завершается (~ETA) → мерж PR → скоординированный деплой фиксов → опц. перемер floor заново на фикшеном коде. Мерж PR можно раньше (доставка ≠ релиз), деплой — нет.


## НОЧНОЙ ПЛАН (автономно, 2026-07-17, пользователь ушёл, широкая автономия)
Роль: тимлид. Максимум 1-2 доп-воркера (не плодить). Каждые ~25 мин проверять что работа идёт. **Полный тест на РЕАЛЬНОМ проде на КАЖДОМ этапе (не фикстуры).** Всё писать в память для утренней сверки. Откат прода если что-то не так.

**ГРАНИЦЫ (пользователь подтвердил устно):**
- Деплой в прод (фиксы + RAG#1) — МОЖНО, с бэкапом + verify + запоминанием + откатом при сбое.
- IVAONEHALF запись в корпус + epic_task — МОЖНО (бэкап/verify/аддитивно, IVAONE==87097 не трогать).
- RAG#1 фиксы/фичи + деплой — МОЖНО (как выше: бэкап/запомнить/откат).
- НЕЛЬЗЯ: Eva-wiki (нет orionpro-кредов) — оставить пользователю. Р-3 включение — измерить и доложить, но не включать без пользы.

**ФАЗЫ по порядку (каждую — прод-тест на реальном + фиксация в память):**
1. Деплой 5 фиксов (rebuild PID 3163671, c2796a5): verify key_lookup 0.46→~1.0, effort_hint null→численные, тулы живы.
2. Пост-verify: Run P на golden /tmp/golden_wide.json (key вернулся?); floor τ ре-калибровка (pinned иммунны → подобрать τ для шума без вреда key).
3. IVAONEHALF все 574: фетч (iva-mcp, 1 сабагент) → аддитивный upsert iva_jira + changelog в epic_task → verify IVAONE цел + IVAONEHALF findable + effort_hint ИВА-1.5.
4. jira_issue_links: данные ЕСТЬ `/opt/helm/data/real/jira/jira_issue_links.csv`, контейнер ищет `/app/data/iva/...` — путь/mount-фикс → blockers() не падает, structural_coverage>0.
5. RAG#1 + бот: разведать (греп rag1/bot/telegram/slack, compose, память), тесты, фиксы, деплой можно.
6. Финал: память + заметка `night-work-summary` (сделано/числа/осталось пользователю).


## НОЧЬ — прогресс (firing 1, ~19:40)
- **Фаза 1 ДЕПЛОЙ применён:** прод `c2796a5`, rebuild done, сервис жив (uvicorn startup complete, StreamableHTTP session manager). floor×pin код на месте (`pinned`×5 в rag2.py).
- **verifier `deploy-verify` запущен** (1 воркер): полный прод-verify фаза 1+2 на реальном golden 640 — Run P на фикшеном коде (key_lookup 0.463→~1.0?), effort_hint числа (ORDER BY), тулы, floor τ ре-калибровка. Прогоны тяжёлые (~2ч), собирает в фоне.
- Сам двигаю фазу 4 (jira_issue_links путь-фикс, read-only разведка) пока verifier гонит.


## НОЧЬ — фаза 4 links разведана (firing 1)
- Код (`graph.py`/`service.py` `_DEFAULT_CSV`) ждёт `/app/data/iva/jira_issue_links.csv`. Файл на хосте: `/opt/helm/data/real/jira/jira_issue_links.csv` (5512 связей Blocks). Контейнер mounts: только `data/real/api` + `data/real/contracts` — `data/iva` НЕ смонтирован → blockers падает.
- Стоп-гэп docker cp НЕ прошёл (контейнер под непривилег. юзером, /app/data только через volume).
- **Фикс = compose-volume `./data/iva:/app/data/iva` + rebuild.** Файл скопирован на хост в `data/iva/` (подготовка). **Rebuild нельзя пока verifier гонит прогоны** → применить volume+rebuild ПОСЛЕ deploy-verify. Compose-правку оформить git-clean (ветка/PR) — прод-дрейф не плодить, утром синхрон с пользователем.
- ПОРЯДОК ночи скорректирован: ждём verifier (фаза 1+2) → потом фаза 4 links volume+rebuild (заодно можно с фазой 3 IVAONEHALF, если тоже нужен rebuild — объединить в один rebuild).


## НОЧЬ — фаза 1 ФУНКЦИОНАЛЬНО ЗЕЛЁНАЯ (deploy-verify, ~20:10)
Деплой 5 фиксов (c2796a5) функционально здоров на живом проде:
- **effort_hint РАБОТАЕТ (Р-5 закрыт вживую):** численные медианы, НЕ null. «push-уведомления» active_days med=1.9/lead 15.4, «экспорт» active 1.1/lead 79.2, coverage 20/20. ORDER BY фикс + UPDATE 377→6676 подтверждены end-to-end.
- api_registry_check «отозвать письмо»→not_found ✅; contract_check «отзыв письма JUMP»→messageRevoke ✅.
- Sweep 4 прогона (τ=0/0.5/0.6/0.7, floor drop, live) идёт — вердикт floor×pin по key_lookup (0.463→~1.0?) + рекомендация τ придёт ~2.4ч.
- Live-сервис аналитиков цел, деградации нет.
ИТОГ: Р-1, Р-4-JUMP, Р-5 — зелёные вживую после деплоя. Р-2 floor×pin — ждём sweep-число.


## НОЧЬ — фаза 3 IVAONEHALF ОТЛОЖЕНА до кредов (firing ~20:20)
Пайплайн готов (ivaonehalf-fetch: `prepare_items.py` на прод-коде, паритет 1-в-1 с 87k IVAONE доказан на синтетике, /root/ivaonehalf_prep/). НО фетч требует **Jira-креды** (Basic HELM_RAG2_JIRA_USER/PASSWORD или PAT) — на helm ОТСУТСТВУЮТ (разово подавались при исходном фетче 87k). Вариант B (iva-mcp нормализованный + ре-эмит 577 задач через контекст) ОТКЛОНЁН — риск порчи паритета рабочего корпуса без надзора. Живой total IVAONEHALF вырос 574→**577**.
→ **Фаза 3 отложена до утра (нужны Jira-креды, как Eva-wiki).** Утром: креды → фетч+запись за минуты (пайплайн готов).
Ночь продолжается: verify-sweep идёт, фаза 4 links (volume+rebuild после sweep), фаза 5 RAG#1 (разведка сейчас).


## НОЧЬ — фаза 5 RAG#1 аудит промежуточно (rag1-audit, ~20:35)
- **Тесты RAG#1 ЗЕЛЁНЫЕ:** 128 passed, 0 skip/xfail, обманок нет (моки только на I/O-границах, доменная логика реальная).
- **Корпус RAG#1:** `iva_docs__bge_m3_1024` = 8272 чанка / 796 страниц. Golden `/root/iva-docs/golden/golden_iva_rag1.json` = 1306 кейсов, slug 796/796 совпадение (eval мерит реальный ретрив). Прод-флаги: hybrid+rerank ON, query_rewrite/near_dup OFF.
- **Находки:**
  - **F1 (главное): нет score-floor на ретриве RAG#1** → риск галлюцинаций на вопросах вне корпуса (known_gaps ловит только 0 чанков / явный отказ LLM). RAG#2 floor имеет, RAG#1 нет. Воркер реализует score-floor default-OFF + тесты (симметрия с RAG#2). Включение/τ — после eval.
  - F2: near_dup_dedup OFF, 39/796 (4.9%) дубли занимают до 8/10 слотов контекста. Флаг+код есть → A/B после eval.
  - F3: query_rewrite (синонимы МСУ↔MCU) OFF → ft-recall аббревиатур, не измерено.
- **docs_eval прогон ОТЛОЖЕН до завершения sweep** (rerank×1306 грузит тот же Gateway, что sweep+аналитики — не параллелить). Прогоню сам после sweep с `--compare` (semantic/hybrid/hybrid_rerank). F2/F3 измерю тогда же.
- **Деплой RAG#1-фиксов (F1) + links-volume — одним rebuild после sweep** (не в пик замера).


## НОЧЬ — RAG#1 F1 запушен (firing ~20:45)
RAG#1 аудит завершён (rag1-audit, отчёт `rag1-audit-result`). Полный suite 1606 passed.
- **F1 score-floor RAG#1 запушен: ветка `fix/rag1-rerank-floor-v2`** (для PR пользователю утром). ⚠️ Исходная ветка воркера была на УСТАРЕВШЕМ base (9774cab, до PR #78) → смерж откатил бы 5 фиксов; **пересоздал v2 + ребейз на origin/main (c2796a5)** → чистый diff: только F1 (docs_assistant score-floor, config, service, bot_support, docs, +73 теста), 98 insertions, фиксы не тронуты. Тесты 91 passed после ребейза.
- **F1 НЕ деплою ночью:** default-off (нулевой эффект без флага, не срочно) + мерж в main делает пользователь. PR-материалы готовы.
- **PR F1 — заголовок:** `feat(docs-assistant): score-floor реранка RAG#1 против галлюцинаций (default-off)`. Суть: RAG#1 (бот наружу) не имел score-floor → риск галлюцинаций вне корпуса; добавлен default-off, симметрия с RAG#2. Включение/τ — после docs_eval.
- **F2 (near_dup 4.9%) / F3 (query_rewrite синонимы) — измерю docs_eval после sweep, решу A/B.**
- rag1-audit получил shutdown (работа сдана).


## НОЧЬ — F1 поправка: актуальная ветка v3 (firing ~20:50)
Воркер сделал amend F1: c6fea90(raw score)→**7f57d23** (симметрия с RAG#2: floor по calibrated confidence 0..1, score=None иммунен, floor<=0 no-op default-off). Пересоздал: **`fix/rag1-rerank-floor-v3`** (ребейз 7f57d23 на origin/main, только F1, 113 insertions, 5 юнит-тестов, 92 passed). **Устаревшая v2 (raw score) удалена с origin.** PR пользователю утром — из **v3**.
**PR F1 актуальный:** ветка `fix/rag1-rerank-floor-v3`, заголовок `feat(docs-assistant): score-floor реранка RAG#1 против галлюцинаций (default-off)`. Суть: RAG#1 (бот наружу) score-floor по confidence 0..1 (симметрия RAG#2 apply_noise_floor), None-иммунитет, default-off. Тесты: drop weak / all-below→refuse-без-LLM / None-immune / no-op default / без-реранка. Включение/τ — после docs_eval.
rag1-audit shutdown. Активен только deploy-verify (sweep).


## НОЧЬ — links-ветка + локальный main подтянут (firing ~00:00)
- **Локальный main был устаревший (9774cab)** — не подтянут после contract-volume+fix мержей. origin/main и прод целостны (c2796a5, contracts volume есть). Подтянул локальный → c2796a5. (Фиксы/F1 ребейзил на origin/main, push'и верны — устарел только checkout.)
- **Фаза 4 links: ветка `fix/rag2-issue-links-volume` запушена** (compose +4: volume `./data/iva:/app/data/iva`). Данные (5512 рёбер) уже на хосте в data/iva. PR пользователю утром (деплой не срочен, дрейф прода не плодил). После мержа+rebuild → blockers работают, structural_coverage>0.
- Урок: проектный CLAUDE.md ЗАПРЕЩАЕТ AI-подписи в коммитах (Co-Authored-By: Claude) — случайно добавил, исправил amend+пересоздал ветку. НЕ добавлять впредь.
- Sweep 3 параллельно (τ=0.5/0.6/0.7) идёт, verdict ~01:00-01:30.
**Ветки к PR утром:** `fix/rag1-rerank-floor-v3` (F1 RAG#1), `fix/rag2-issue-links-volume` (links). Обе default-safe, пользователь мержит + деплой.


## НОЧЬ ЗАВЕРШЕНА (~23:00, 2026-07-18) — все измерения закрыты
Полная сводка: [[night-work-summary]]. docs_eval RAG#1: прод hybrid+rerank оптимален (0.930); F2 near_dup ВРЕДИТ (не включать); F3 query_rewrite маргинально +0.01 (опция). **Все фазы ночи, что не требуют пользователя, сделаны.** Таймер (cron 25a0a324) остановлен. deploy-verify shutdown.
**ЖДЁТ ПОЛЬЗОВАТЕЛЯ (утро):** мерж 2 PR (F1 rag1-rerank-floor-v3 + links) + деплой; Jira-креды→IVAONEHALF; orionpro→Eva-wiki; решения τ=0.7/F3-ON/Р-3 (все маргинальные, прод работает). F2 — НЕ включать.
**ИТОГ ТЗ:** Р-1/Р-2/Р-4-JUMP/Р-5 + дотяжка — 🟢 зелёные вживую на реальном проде. Р-3 ⚪, Eva-wiki/IVAONEHALF ⏸️ креды.


---
# ✅ ЧЕКПОИНТ 2026-07-18 (после ночной работы + F3-догон) — КОНСОЛИДАЦИЯ

## Прод-состояние
- helm `main = c036842`, задеплоено, сервис жив (uvicorn startup complete). Все фиксы+F1+links в проде.
- **Unit-тесты: 1619 passed / 30 skipped** (RAG#1 133 · RAG#2 268 · MCP 67). Не обманки (аудит подтвердил).
- Бэкап epic_task: `/opt/helm/data/backup_epic_task_2026-07-17.sql`.

## Итог по ТЗ (RAG#2-доработка-контракты)
- **Р-1** api_registry 🟢 (419 операций, тул вживую, precision-фикс)
- **Р-2** floor/rerank 🟢 (floor×pin: key_lookup 0.46→1.00, шум-фильтр 38/40)
- **Р-3** code-term ⚪ (код есть, флаг off, НЕ измерен — открытый вопрос)
- **Р-4** JUMP contract 🟢 (101 команда, messageRevoke) · Eva-wiki ⏸️ (orionpro-креды)
- **Р-5** effort_hint 🟢 (changelog 377→6676, численные медианы вживую)
- **дотяжка** (recall-ядро) 🟢 (key_lookup 0.303→1.00)
- **blockers/structural** 🟢 (links volume, 5512 рёбер, граф 8102 узла)

## Eval-метрики (реальные данные)
- RAG#2 (golden 640): recall@10 0.973, key_lookup **1.00**, title_dense 0.947, MRR 0.697.
- RAG#1 (docs_eval 1306): hybrid+rerank recall@5 **0.949** (прод, оптимален).
- MCP: effort_hint численные медианы, api_registry 419, contract 101 — приёмка на реальных.

## Отсеяно замером — НЕ включать (доказано на реальных данных)
- **F2 near_dup_dedup** — вредит RAG#1 (0.930→0.877).
- **F3 query_rewrite** — вредит на полном 1306 (0.949→0.930, −0.019; ночной +0.01 на 300 был артефакт узкой выборки).
- **cross-rerank, hybrid ре-баланс ft** — recall не дают (RAG#2, прежние замеры).
- **τ=0.7** — маргинально (+1 OOD из 40), прод-τ оставлен 0.5.

## Ждёт пользователя (открытое)
1. **IVAONEHALF** (corpus-gap ИВА-1.5, 577 задач): нужны Jira-креды → `HELM_RAG2_JIRA_USER/PASSWORD` или PAT → `bash /root/ivaonehalf_prep/run.sh` (пайплайн готов, паритет 1-в-1 доказан).
2. **Eva-wiki** (Р-4 2-я часть): orionpro session-креды + HTML-парсер вики.
3. **Р-3 code-term**: не измерен — можно замерить (единственный неопределённый пункт ТЗ).

## Ключевые заметки
[[rag2-ab-measurements]] (все числа), [[night-work-summary]] (ночь), [[deploy-verify-result]], [[docs-eval-result]], [[f3-fulleval-result]], [[rag2-r5-rebuild-and-datamap]], [[rag2-ivaonehalf-gap]]. Все агенты завершены, таймер снят.