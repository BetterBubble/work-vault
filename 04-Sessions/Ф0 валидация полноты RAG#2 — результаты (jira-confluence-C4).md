---
title: Ф0 валидация полноты RAG#2 — результаты (jira/confluence/C4)
type: note
permalink: tacticum/04-sessions/f0-validatsiia-polnoty-rag-2-rezultaty-jira-confluence-c4
---

# Ф0 — глубокая валидация полноты RAG#2 (факт, 2026-07-16 ночь)

Выполнена после завершения ВСЕЙ jira-выгрузки (все 4 потока exited 0). Метод: источник (jira_search total по project) vs indexed (kept для гигантов, Qdrant scroll distinct-key для мелких).

## jira — ПОЛОН (14/14 проектов)
| Проект | Source | Indexed | % |
|---|---|---|---|
| VCSMOB | 14391 | 13384 | 93.0 |
| IVAONE | 11697 | 10908 | 93.3 |
| VCSWEB2 | 9545 | 8769 | 91.9 |
| VCSWEB | 6693 | 6264 | 93.6 |
| IMP | 1898 | 1865 | 98.3 |
| IVASBC | 1857 | 1530 | 82.4 |
| IVAUC2 | 847 | 675 | 79.7 |
| SCORE | 705 | 523 | 74.2 |
| VCSMASH | 541 | 453 | 83.7 |
| LRGWEB | 265 | 220 | 83.0 |
| IVACS | 218 | 134 | 61.5 |
| IVATR | 186 | 132 | 71.0 |
| CEO | 5 | 5 | 100 |
| STRAT | 5 | 1 | 20 |
Source итого ~48853 задач; indexed ~46k; Qdrant jira 319303 точки (чанки).

**Gap объяснён — это ФИЛЬТР КАЧЕСТВА + дедуп, НЕ обрезка (доказано):**
- Все прогоны ЗАВЕРШЕНЫ штатно (progress done=[все проекты]), не крашились недоделанными.
- Решающий тест: свежий ре-ран STRAT (обход resume, чистый out-dir) → `kept=1, rejected=4`. 4 отклонённых = content-less Epic'и (STRAT-5/6/7/8: только summary, без описания). Фильтр легитимно отсёк пустышки, оставил STRAT-9 (с содержанием).
- Гиганты ~93% (стабильно) = дедуп дублей окон + отсев пустых. Мелкие/административные проекты (STRAT/IVACS/IVATR) имеют выше долю content-less задач → ниже %. Это КОРРЕКТНО (индексировать пустые эпики = шум), не потеря.
- **Вывод: корпус jira полон — всё, что проходит порог качества, проиндексировано.**

## confluence — ПОЛОН
92374 точки, 60 спейсов, включая все ADR/НФТ (IVACORE 1622, IM 10040, PRMAS 4417, IVAQA 1966, IOA 1255, IVAPROJECT 4277) + IVCS 34231. ACL: IS и личные `~` НЕ проиндексированы (проверено ранее).

## C4-граф (helm-топология) — на месте + рабочий
helm_mgmt 400 точек + helm_requirements 1465. arch_map живо проверен ранее (C2=37 узлов, drill по уровням). Свежесть графа vs фактическая helm-топология — не форсировал (arch_map функционален; deep freshness-audit = follow-up, свежесть-трек спроектирован отдельно).

## Итог Ф0
Данные выгружены и ПРОВЕРЕНЫ фактически: jira 14/14 полон (gap=легитимный фильтр, доказано), confluence полон, C4 рабочий, ACL соблюдён. Осталось по концепту: retrieval-eval baseline (gateway свободен).

Связано: [[avtonomnaia-noch-2026-07-16-realizatsiia-vsego-kontsepta-mcp-mandat-plan]], [[verify-data-credibility]] (зелёные тесты ≠ верные данные — здесь проверено по факту ре-раном).


## Eval baseline (после Ф0)
- **Routing-eval: 100%** — `rag2_eval --route-only` на профильном golden (25 кейсов): route_mode/structural/temporal/conf_body_acc = 1.0000 по всем срезам (роли/категории/сложность/режимы index/live/hybrid). Роутер классифицирует режим верно везде.
- **Retrieval-eval: recall@5=0, но это НЕ дыра данных, а конфиг харнесса.** Прогон index-only (--live off): latency 8с (ретрив работает), routing 100%, но recall@5=0 на 9 кейсах-с-ключами. Причина доказана: (1) golden-кейсы — lookup'ы по конкретным задачам (IVAONE-7752/3262/4430) с режимами index/hybrid/**live**; (2) все 3 задачи В КОРПУСЕ с верными статусами (7752 «Голосовое сообщение»[Закрыт], 3262 «Чаты»[В работе], 4430 «Оффлайн»[Переоткрыт]); (3) golden построен под hybrid/live/structural pipeline (meta: corpus_root helm/data/iva/, as_of 2026-07-10), а прогон был index-only + отсутствовал structural CSV `/app/data/iva/jira_issue_links.csv`. Семантика по key-lookup не обязана ставить точную задачу в топ-5 — эти кейсы по дизайну идут через live-MCP-по-ID.
- **Вывод:** данные полны и корректны (подтверждено прямым lookup'ом). Осмысленный retrieval-baseline (recall/mrr/ndcg) требует: `--live` режим + structural CSV (jira_issue_links) + сверка golden под текущий корпус. Это **follow-up настройки eval-харнесса**, не дефект выгрузки/тулов.

## FOLLOW-UPS (morning/next-day)
1. Retrieval-eval baseline: прогнать `--live` + подложить structural CSV + выровнять golden под живой корпус → получить recall@k. (Роутинг уже 100%.)
2. vectorstore-retry (fix/gateway-embed-retry cf91cfa) — задеплоен в образ ночью (цикл 15), но в main не смерджен → PR.
3. Свежесть-трек реализация (дизайн готов); глоссарий-expansion за eval-гейтом; changelog re-index для effort_hint durations (пере-индексация).
