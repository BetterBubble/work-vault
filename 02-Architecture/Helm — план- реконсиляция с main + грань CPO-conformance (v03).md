---
title: 'Helm — план: реконсиляция с main + грань CPO/conformance (v03)'
type: note
permalink: tacticum/02-architecture/helm-plan-rekonsiliatsiia-s-main-gran-cpo-conformance-v03
tags:
- helm
- control-tower
- v03
- conformance
- cpo
- plan
- git
- wiki
- lightrag
---

# Helm — план: реконсиляция с main + грань CPO/conformance (v03)

Зафиксировано 2026-07-04 (тимлид-сессия). Работа поставлена на СТОП по просьбе оператора —
это чекпойнт плана для следующей сессии. Полный файл плана:
`/Users/bubblemac/.claude/plans/scalable-leaping-valley.md`.

## Что изменилось (2 события параллельно нашей работе)

- [reference] **Новый канон `data/control-tower-v03.md`** (596 строк). Helm поглотил **Compass**
  (ADR-0010) → единый App полного продуктового цикла. Ключевое: **третья грань CPO/Продукт (§5A)**
  — модель `Signal → Need ↔ Requirement → Initiative` (было `Signal → Initiative`), ось поколений
  first-class (1.0/1.5 RN/1.5 KMP/2.0/3.0), первый продукт — **conformance 1.5 RN vs KMP** (G1 LLM
  + eval-gate, тут «LLM не в критпути» сознательно снят). Трек C параллельно 1a/1b. Третий контур
  доступа **amber** (commercial/roadmap). §8.3 — авторитетная RAG-арх: 3 плоскости (Graphiti-граф ·
  вектор knowledge_rag · структурный warehouse), router, **числа только из структурной плоскости**,
  контур-фильтр ДО ретрива, identity first-class. **LightRAG = переиспользовать платформенный, НЕ
  строить самим** (закрыло развилку). Генезис Initiative +4-й источник = validated Requirement. #v03
- [decision] **На `origin/main` — параллельная линия разработки (автор Diaret), у нас её нет.**
  Коммиты: `Add conformance and product catalog domains` (**`domain/conformance.py` — G2 структурный
  прокси: Requirement/Generation/WorkItem/ConformanceReport, вердикты met/partial/no_work, метрика
  met/total; G1-LLM надстраивается сверху** + `product_catalog.py` канонизация продуктов, ДУБЛИРУЕТ
  наш M3), `company_kb.py` facade + gap-детекторы + identity directory + salary queries, **`web/` —
  React+Vite+TS фронтенд** (хотя мы вели backend-only). Наш PR #1 (wave-1a-backend, до дропа) уже
  смёржен в main. #remote

## Git-статус расхождения (проверено)

- [reference] merge-base `01c555b`. У нас (wave-1a-backend HEAD `38fb96e`) — **6 уникальных коммитов**
  (дроп EVA/git-API), на `origin/main` — **11 уникальных**. **Пересечение изменённых файлов = ПУСТО
  → merge ЧИСТЫЙ, без конфликтов** (мы трогали `ingest/*`, они — `domain/conformance|product_catalog`,
  `web/`, KB). Наш дроп в main ещё НЕ влит. Remote: `git@github.com:TacticumApps/helm.git`. #git

## Данные CPO (проверено)

- [reference] Корпус требований **`data/arch/generations/` НЕТ ни у нас, ни на main** → C1 блокирован
  до его появления в Data. Есть только `reference_generations.csv` (таксономия поколений). **Код-сторона
  готова: `repomix/scip` для `rn.xml` И `kmp.xml`** (+ iva-one) — обе стороны 1.5 индексированы. #data

## План (порядок)

- [plan] **Шаг 0 — git-реконсиляция (первым):** `git merge origin/main` в wave-1a-backend (чисто) →
  прогон pytest+ruff+mypy → **свести дубль product-канонизации** (их `product_catalog.py` vs наш
  `re_enrich.build_full_repo_product_map` M3) → PR `wave-1a-backend → main` с дропом. Фронт `web/`
  проверить, что бьёт в реальные `/api/*` (наш scope backend-only не меняется). #step0
- [plan] **WP-C1 — грань CPO/conformance 1.5:** РАСШИРЯЕМ удалённый G2, не greenfield. Нужно:
  (1) корпус `data/arch/generations/` [БЛОКЕР], (2) G1-слой LLM «код↔требование» через Gateway поверх
  `ConformanceReport`, (3) eval-gate faithfulness+contour-leakage=0 ДО показа, (4) фикс SCIP-асимметрии
  RN/KMP (grep-fallback Kotlin + confidence на сторону), (5) матрица «требование 1.5 × RN/KMP». #trackC
- [plan] **WP1 — wiki-каталог → knowledge_rag (только Data):** каталог из `data/real/wiki/pages_index.csv`
  (28 990: `source_type="wiki_index"`, title+space+url) + 3 готовых тела (`source_type="wiki"`). Билдеры
  `wiki_index_items`/`wiki_items` в `ingest_sources.py` + ветка в `ingest_knowledge.py` + тесты. Full-text
  ждёт тел в Data (внешний фетч ЗАПРЕЩЁН оператором). #wiki
- [plan] **WP2 — мост task→initiative (A7):** починить резолюцию task→epic→initiative (1/5512 → плотно)
  epic_link+LLM-judge+MR-топология, `origin="inferred"`. Разблокирует A7 критпуть (v03 Волна 3) + полный C3. #a7
- [plan] **WP3 — Graphiti:** прогнать `project_to_memory.py --write` (проекции в графе не видно) + замкнуть
  `read()` на velocity для A6. Мягко, Волна 2-3. #graphiti
- [plan] **WP4 — LightRAG:** переиспользовать платформенный (v03), `/lightrag/mcp` сейчас 404 → к платформе. #lightrag

## Открытые предусловия / вопросы
- [followup] Оператор ищет корпус `data/arch/generations/` — без него WP-C1 не собрать. #blocker
- [followup] Ограничение «только Data, без внешнего фетча» действует (снято на этой сессии).
- [followup] Развилка ADR-0010 D2: requirements-грань — выносимый bounded-context (когда отделять обратно).

## Что НЕ ломается
- [outcome] Наши 1a Монитор ГД (§5) + 1b Ценность (§6) в v03 валидны — Need↔Requirement добавляется
  ПЕРЕД `Signal→Initiative`, не заменяет. Т1/Т2/Т3, разрывы §6.1, светофор, снапшоты — на месте.

- relates_to [[Helm — новый дроп (GitLab-API 216k + EVA/IVA Desk) + план интеграции]]
- relates_to [[Helm — runbook работы с dev-сервером (SFTP-доставка, не rsync)]]
