---
title: Helm — инжест Need/потребности (механизм готов, черновик на отсмотр)
type: note
permalink: tacticum/20-architecture/helm-inzhest-need-potrebnosti-mekhanizm-gotov-chernovik-na-otsmotr
tags:
- helm
- control-tower
- need
- extraction
- cpo
- backlog
- operator-gate
- ready-not-deployed
---

# Helm — инжест Need/потребности (механизм готов, черновик на отсмотр)

Фоновая Задача 2 (2026-07-08). **Ветка `feature/need-ingest`** (worktree `~/tacticum-worktrees/helm-need-ingest`), 1 коммит `a679943`, **668 тестов зелёные**, НЕ задеплоено. Вычленяет кандидатов `Need` из клиентского спроса — разблокирует срез CPO «Бэклог/Потребности». Связано: [[Helm — аудит готовности срезов (что на проде + что следующее)]], [[Helm — потребности- первый прогон вычленения (31 из 779)]].

## Как работает (пайплайн `ingest/needs.py`)
- [done] Спрос = `DemandItem` из **FR-заявок (`sd_request`) + проигранных сделок (`crm_deals`, стадии отменено/проигран)**.
- [done] Кластеризация **по проблеме, не по решению**: эмбеддинги bge-m3 → агломератив → LLM-метка. **Переиспользован `sd_themes.build_themes` целиком** (+ `EmbeddingBackend` Gateway/лексический офлайн, `theme_label` нейминг, fallback-паттерн).
- [done] Кластер → `NeedCandidate`: title=метка · description=репрезент.фразы+сводка спроса · product=доминанта · client=если моно-клиент · demand_weight (**FR=1.0, проигранная сделка=2.0**) · confidence=чистота. `refresh_needs` — scoped-replace + ledger (source=`needs`).
- [decision] **Операторский гейт: всё `status="candidate"`**, не автопубликация. Оператор промотит `candidate→active`.

## Умные решения
- [decision] **id = контент-хеш кластера** (`NEED-CAND-<sha10>`): промотированный кандидат сохраняет id, повторный прогон не плодит дубль; scoped-replace сносит только `candidate` (не `active`/`merged`).
- [decision] **FK-safe**: `Need.product`/`client` в FK только если резолвится в справочник, иначе `None` + сырьё в описании (защита от IntegrityError на Postgres до засева рефдаты).
- [decision] Дефолты: `target_k=24`, `min_size=3` (флаги `--target-k/--min-size`). Тесты — синтетика с замоканными эмбеддингами/LLM.

## Чтобы активировать (реальный прогон)
- [todo] На сервере: `alembic upgrade head` → `refresh_servicedesk.py` (наполнить `sd_request`) → засеять рефдату `product`/`client` → `seed_needs.py --crm-deals data/real/crm/crm_deals.csv --as-of <дата>`; подбор `--target-k/--min-size` по факту.
- [todo] **Оператор отсматривает** таблицу `need` (status=candidate): метка, фразы, сводка спроса (N FR + M сделок, вес, чистота), продукт, клиент → валидные промотит в `active`.
- [open] Открытые: вес проигранной сделки (фикс 2.0 vs взвешивать по сумме `сумма_iva_с_ндс`); `source_url` пока None (нет URL в источниках).