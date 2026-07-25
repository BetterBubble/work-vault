---
title: Автономный прогон RAG#2 — полный проход + глубокая проверка (мандат)
type: note
permalink: tacticum/01-sessions/avtonomnyi-progon-rag-2-polnyi-prokhod-glubokaia-proverka-mandat-1
tags:
- helm
- rag2
- mcp
- autonomous
- verification
- mandate
---

# Автономный прогон RAG#2 — мандат и чеклист (пользователь ушёл надолго 2026-07-15)

## Мандат пользователя
1. Довести полноглубинную выгрузку до КОНЦА (6-way оконный проход).
2. Потом ГЛУБОКО и ДЕТАЛЬНО проверить весь путь данных: достоверность · точность · полнота → как данные заходят в MCP → как работает сам MCP → нормальны ли данные.
3. Такие проверки — ~каждый час.
4. Деплой на helm по мере нужды; потом обязательно push + PR (sync сервер-локаль-github).
5. На развилках — выбор в сторону КАЧЕСТВА: тест → смотрю → если ок, применяю.
6. Финал: RAG#2 + MCP полностью рабочие, готовые, аналитику сильно облегчают жизнь; все нужные данные из helm и прочих отдаются ему.

## Текущее состояние (на момент старта прогона)
- 6 процессов выгрузки: `rag2-full-{c1,c2,c3}` (confluence, слайсы `/opt/helm/_run/slices/conf_{1,2,3}.txt`) + `rag2-full-{j1,j2,j3}` (jira, `jira_{1,2,3}.txt`). Флаги: `--window-size 500 --sleep-s 0.25`, out `/opt/helm/_run/out_full/<name>/`, общие staging-коллекции `iva_confluence__bge_m3_1024` / `iva_jira__bge_m3_1024` (Qdrant 10.16.0.19:6333). Оконный harness (PR#51, main 3a23974) задеплоен на сервер.
- jira слайсы (крупные разнесены): j1=VCSMOB,CEO,STRAT,IVATR · j2=IVAONE,IMP,IVAUC2,SCORE · j3=VCSWEB2,VCSWEB,VCSMASH,IVACS,IVASBC,LRGWEB.
- Счётчики на старте: conf ~17376, jira ~17480 (растут по мере ухода окон глубже капнутых 300).
- Rerank починен и активен (RAG#1 DocReranker + RAG#2 JiraReranker×3), e2e MCP с реальным токеном пройден (auth fail-closed ок). RAG#1 docs корпус `iva_docs__bge_m3_1024`=8272.

## Чеклист ГЛУБОКОЙ проверки (запустить, когда выгрузка ЗАВЕРШИТСЯ)
1. **Полнота (counts):** для каждого проекта/спейса сверить source-count (Jira `/search` total, Confluence space total) vs проиндексировано в Qdrant (`scroll_distinct_payload_values` по project/space). Расхождения — расследовать (QualityConfig-фильтр vs реальные потери). Крупные: IVAONE, VCSMOB, VCSWEB2, VCSWEB, IVCS.
2. **Полнота полей:** аудит по items.jsonl — Jira: desc/com/ch/links/att % (эталон капнутого IVAONE: desc 89%, changelog 99%, 8693 изменений). Confluence: body непустой, вложения, комменты.
3. **Достоверность (credibility):** взять 3-5 случайных ingested-задач/страниц и СВЕРИТЬ с живым источником через iva-mcp (`jira_get_issue`/`confluence_get_page`) — контент совпадает? Не побит? Не устаревший? (память verify-data-credibility — зелёные счётчики ≠ верные данные).
4. **Точность/качество ретрива:** набор аналитических запросов → analyst_search/context релевантны? rerank улучшает порядок? Нет мусора/стейла в топе?
5. **Заход в MCP:** прямой вызов внутреннего пути (_answer) + e2e через реальный токен (файл токена пользователь удалил — при необходимости попросить снова) — все 8 инструментов на ПОЛНЫХ данных.
6. **Данные нормальные:** чанки не побиты, кириллица целая, нет HTML-мусора/дублей, ссылки/ключи валидны.

## Треки в работе
- Воркер `cross-rerank` (пункт 4): кросс-корпусный реранк за флагом `rag2_cross_rerank_enabled` (default off, fail-soft) + тесты. Хендофф → `00-Board/cross-rerank-SUMMARY.md`. После: A/B на живых данных → если качество лучше, включить.

## Git-состояние
- origin/main = `3a23974` (PR#49 rerank-флаги, #50 rerank-фикс, #51 harness-windowing — все смержены). Сервер /opt/helm: rerank в .env + reranker.py/harness прямыми правками = совпадают с main (sha сверены).
- Связано: teamlead-delegate-not-do, dont-duplicate-agent-work, verify-data-credibility, agents-communicate-via-notes.


---

## Пункт 4 (кросс-корпусный реранк) — реализован + A/B сделан

- Воркер `cross-rerank`: ветка `feat/rag2-cross-rerank` (`2209f98`, запушена, PR github.com/TacticumApps/helm/pull/new/feat/rag2-cross-rerank). Флаг `rag2_cross_rerank_enabled` (default OFF), fail-soft, Protocol в application (слои соблюдены). Тесты: 14 passed + 195 (-k rag2), ruff/mypy чисто.
- **A/B на живых данных (baseline per-corpus+RRF vs cross-only):** cross-rerank ЗАМЕТНО лучше по релевантности. На «запись/транскрибация» baseline выдал мусор №1 («Шаблон запроса на скоринг фичи»), cross-only поднял 3 профильных дока (score 0.90/0.86/0.75). Скоры честные. Трейд-офф: cross иногда перекашивает в один корпус (теряет диверсити RRF — напр. топ-4 из Jira, док на #5).
- **Рекомендация:** включить cross-rerank (качество топа выше, убирает RRF-мусор). Возможное улучшение на будущее — diversity-preserving гибрид (гарантировать ≥1 из каждого корпуса в top-k). Enable = после мержа PR → deploy → `HELM_RAG2_CROSS_RERANK_ENABLED=1`. Пока OFF (решение о UX-трейд-офф за пользователем; не блокирует).

## Git-состояние (обновл.)
- Запушены и ждут мержа: `feat/rag2-cross-rerank` (2209f98). Остальное (rerank-флаги #49, rerank-фикс #50, harness-windowing #51) уже в main.


---

## Инцидент+фиксы (часовой чек #1): 6-way падал, устранено

**Симптом:** все 6 процессов упали Exited(1) через ~15-20 мин. Две разные причины, обе починены (ветка `fix/rag2-fetch-retry`, 2 коммита `83937c6`+`168c268`, запушена, задеплоена на сервер прямыми патчами по мандату «сразу деплой, потом push+PR»).

1. **`httpx.ConnectTimeout` (TLS handshake timed out)** — 6 параллельных fetch-потоков насыщали SSH-туннель к источникам; fetch делал `client.get` без ретраев → один таймаут ронял весь прогон. **Фикс:** `_get_with_retry` (экспоненциальный backoff на транзиентные сетевые ошибки + статусы 429/502/503/504), обёрнуты все 3 request-сайта (jira search, confluence content, attachments). Неретраибельные 4xx пробрасываются. +4 теста.
2. **`JSONDecodeError: Unterminated string`** на resume — `_seed_expected_from_windows` читал `.read_text().splitlines()`, а `splitlines()` дробит по всем юникод-границам (U+2028, \x85/NEL, \v, \f), которые тела Confluence содержат сырыми (json.dumps ensure_ascii=False не экранирует ≥0x20). Одна JSON-запись рвалась. **Фикс:** файловая итерация (по \n, как делает сам ингест `confluence.load_pages`) + пропуск усечённой строки. +2 теста. **Данные в Qdrant НЕ затронуты** (баг только в resume-seed; ингест читал корректно построчно).

**Итог:** перезапущены 6 процессов на исправленном образе (retry+seed), `sleep_s 0.4`. Resume по `progress.partial` работает. Все RUNNING, 0 крашей. Прогресс на момент фикса: conf ~21800, jira ~25200 точек; c1 done=19 спейсов.

**Важно для след. чеков:** если снова упадут — смотреть трейсбек (docker logs ... | grep -v HTTP | tail); ретраи логируются как «ретрай N/M». Ветку `fix/rag2-fetch-retry` пользователю надо СМЕРЖИТЬ (PR github.com/TacticumApps/helm/pull/new/fix/rag2-fetch-retry) — на сервере уже применено патчами, но для sync git нужен merge.


---

## Чек #3: КРИТИЧНЫЙ баг полноты найден и починен, идёт перевыгрузка jira

**JIRA завершилась (exit 0), НО данные катастрофически неполны:** проверка источник-vs-проиндексировано выявила IVAONE 463/11697 (96% потеря!), VCSWEB 448/6693, VCSWEB2 1444/9545, VCSMOB 3265/14391. Итого jira 9866/48853 (79% пропущено).

**Корень:** под параллельной нагрузкой (6 процессов) Jira/Confluence интермиттентно отдавали пустой 200 в СЕРЕДИНЕ проекта; `fetch` трактовал `if not issues/results: return` как конец → крупные контейнеры обрезались и помечались `done` (на resume пропускались). Проверено: при одиночной нагрузке fetch отдаёт полные 500 на всех offset — значит именно нагрузка.

**Фикс (3-й коммит в `fix/rag2-fetch-retry`, `4b3723e`, запушен+задеплоен):** Jira — пустая страница=конец ТОЛЬКО если cur>=total (из ответа), иначе повтор startAt (backoff). Confluence — повтор пустой только если уже были данные (обрыв середины); конец по `_links.next`. +2 теста. Тесты 57 passed 0.17с, ruff/mypy чисто. Заметка: фикс также исправил замедление тестов (реальные sleep на конце спейсов).

**CONFLUENCE в порядке:** gap done-спейсов 10-40% = легитимный фильтр QualityConfig (stale>2года/draft/personal/short — совпадает с «не грузить устаревшее»), НЕ обрезка. Обрезан только IVCS 19810→2067 (был не докачан, c1 offset 2500).

**Перевыгрузка (чек #3):** jira — сброшены progress+out (done=[]), 3 процесса с нуля на фикс-образе; confluence — progress сохранён (done-спейсы фильтр-полные, c2/c3 сразу exit), c1 дорезюмит IVCS. sleep_s 0.6 (мягче туннелю). Qdrant jira дополнится (idempotent upsert).

**След. чеки:** дождаться завершения jira-перевыгрузки → ПОВТОРИТЬ проверку полноты источник-vs-indexed (должно стать ~полно, остаток=фильтр). Затем проверить, что остаточные gap объясняются rejected-причинами (stale/draft/short), а не потерями. Потом достоверность vs iva-mcp + точность ретрива + MCP. Ветку `fix/rag2-fetch-retry` (3 коммита) пользователю СМЕРЖИТЬ.


---

## Чек #4: НАСТОЯЩИЙ корень обрезки найден (не empty-page!) и починен

Перевыгрузка jira (чек #3) СНОВА дала IVAONE 463/11697 — empty-page фикс не помог. Копнул глубже: `extract_jira_container(IVAONE, limit=500, start_at=0)` вернул **kept=463 + rejected=31 = 494** (31 rejected `empty_or_stub`). 494 < 500 (window_size).

**Настоящий корень (баг ОКОННОГО ЦИКЛА, не fetch):** `extract_jira_container` возвращает `dedup_jira_records(kept)` — dedup убирает дубли нестабильной пагинации `ORDER BY updated DESC` (активные проекты меняются между страницами), плюс filter-drop'ы. 500 сырых зафетчено → 494 после dedup. Оконный цикл считал контейнер завершённым при `kept+rejected < window_size` → полное окно (500) ложно принято за неполное → ОБРЕЗКА после win_0. Confluence НЕ затронут (start-пагинация без ORDER BY стабильна, дублей нет — потому его gap'ы были только фильтром).

**Фикс (4-й коммит `a3735d3` в fix/rag2-fetch-retry, задеплоен sha 30dd2a9):** `_extract` через checkpoint_cb считает СЫРОЙ fetched-курсор (до dedup/фильтра), возвращает 3-м элементом; окно завершает контейнер по `raw < window_size` (raw==0 — пустой хвост). kept+rejected больше не решает о конце. +1 регресс-тест (полное окно с kept<window_size не обрывается). 36 passed, ruff/mypy чисто.

**Перевыгрузка (чек #4):** 3 jira с нуля + c1 дорезюмивает IVCS, на фикс-образе (raw+empty-page+retry+seed), sleep 0.5. На момент записи j2 ещё ингестит win_0 IVAONE (partial пусто). СЛЕД. ЧЕК: подтвердить, что IVAONE идёт дальше win_0 (появятся win_500, win_1000…, kept растёт к ~11k) — это докажет фикс на бою. Потом повторить полную проверку полноты 14 проектов.

**Итого фикс-ветка fix/rag2-fetch-retry = 4 коммита:** retry(83937c6) + seed(168c268) + empty-page(4b3723e) + raw-останов(a3735d3). Все задеплоены патчами. Пользователю — СМЕРЖИТЬ.


---

## Чек #5+: raw-фикс ДОБИТ и РАБОТАЕТ (пользователь вернулся, попросил добить самому)

Предыдущий raw-останов (a3735d3) НЕ работал: `_extract` брал raw из `checkpoint_cb`, а `fetch_jira_issues` выходит по лимиту ВНУТРИ страницы (fetched>=limit→return) ДО вызова cb → raw недосчитывал последнюю страницу (500-окно → raw=450<500 → всё равно обрезка, IVAONE снова 463 на перевыгрузке чека #4).

**Правильный фикс (5-й коммит `9827c5a` в fix/rag2-fetch-retry, задеплоен: extract sha 6f8a807, harness 28fdbb6):** `extract_jira_container`/`extract_confluence_container` считают `raw_count` прямо в цикле по генератору, возвращают 3-м элементом. `_ExtractResult`→3-tuple, обновлены ВСЕ вызовы (harness windowed/non-windowed, CLI-обёртки `_extract_jira`/`_extract_confluence` + их caller в main). **Полный набор 1378 passed, ruff/mypy чисто.**

**ПОДТВЕРЖДЕНО НА БОЮ:** IVAONE теперь win_0+win_500, 932 записи, partial продвигается — идёт к полным 11697, НЕ 463. Фикс работает.

**Синхронизация:** origin/main = 3a23974 (никто больше не пушил, конфликтов нет). Ветка `fix/rag2-fetch-retry` = 5 коммитов (retry + seed + empty-page + raw-checkpoint[нерабочий] + raw-extract[рабочий]) — ГОТОВА к мержу после подтверждения полноты.

**СЛЕД. ЧЕК:** дождаться завершения jira-перевыгрузки (все done, крупные проекты ~полны) → ПОЛНАЯ проверка полноты 14 проектов источник-vs-indexed (крупные должны стать ~полны, остаток=фильтр) + IVCS confluence. Потом достоверность vs iva-mcp + точность + MCP на полных данных. Затем финал: push/PR/merge fix-ветки + cross-rerank.