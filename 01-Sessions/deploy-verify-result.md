---
title: deploy-verify-result
type: note
permalink: tacticum/01-sessions/deploy-verify-result-1
---

# deploy-verify-result

Ночной прод-verify деплоя 5 фиксов RAG#2 на helm (`fix/rag2-audit-fixes`). Прод `helm` на **c2796a5** (floor×pin, ORDER BY, weak+weak, tz, парсер-тест), контейнер `helm-helm-1` Up, сервис жив весь прогон. Реальные данные (не фикстуры): live Rag2Orchestrator + Qdrant `10.16.0.19`, golden `/tmp/golden_wide.json` (640: 300 key_lookup + 300 title_dense + 40 negative_ood). Verifier, 2026-07-17 ночь.

## Итог одной строкой
key_lookup вернулся **ДА** (0.463→1.00) · effort_hint работает **ДА** (численные медианы) · рекомендация τ: **можно 0.5→0.7 (маргинально), 0.5 уже отлично**. Деплой функционально здоров, регрессий ноль.

## Фаза 1 — floor×pin (главный verdict)
Прежний широкий перемер на СТАРОМ коде: Run P (прод: дотяжка + floor τ=0.5 drop) ронял запиненные exact-key → key_lookup=**0.463**. Ожидание после фикса (pinned иммунны к drop): ~1.0.

**Подтверждено в коде на проде:** `apply_noise_floor` (domain/rag2.py) имеет `if d.pinned: append; continue` — образ реально несёт фикс (не только git).

| срез key_lookup (300) | СТАРЫЙ код (Run P) | НОВЫЙ код c2796a5 (τ=0.5) |
|---|---|---|
| recall@10 | **0.463** | **1.000** |

🟢 Floor×pin ФИКС РАБОТАЕТ. Аналитик по issue-key снова находит задачу в 100% (было ~46%). key_lookup no_answer_rate=0 (позитивы не режутся).

## Фаза 1b — effort_hint (ORDER BY фикс)
Прежде `_search_epic_by_terms` делал LIMIT 20 без ORDER BY → после MVCC-UPDATE (охват 377→6676) seq-scan брал непокрытые строки → null-медианы. Фикс: детерминированный ORDER BY + предпочесть непустой changelog.

Прод-проверка (MCP helm-analyst → прод):
- «push-уведомления в чате» → active_days med=**1.9** (p25 0.2/p75 8.7), lead_time med=15.4; changelog coverage **20/20** (ratio 1.0).
- «экспорт списка пользователей» → active_days med=**1.1**, lead_time med=79.2; coverage **20/20**.

🟢 Медианы ЧИСЛЕННЫЕ, покрытие полное. ORDER BY фикс + UPDATE подтверждены на живом проде.

## Фаза 1c — тулы живы
- `api_registry_check` «отозвать письмо» → **found=false (not_found)**, реестры bot/clients/integration проверены. 🟢
- `contract_check` «отзыв письма в JUMP» → **found=true → messageRevoke** (JUMP, score 4). 🟢

## Фаза 2 — floor τ ре-калибровка
Новый код, floor drop, дотяжка ON, k=10, golden 640. Метод: полные live-прогоны (оффлайн-sweep отклонён — `retrieved_hits` не сериализует флаг `pinned`, оффлайн-порог соврал бы по иммунным exact-key). 3 прогона запущены ПАРАЛЛЕЛЬНО (I/O-bound, CPU простаивал) ~92 мин wall-clock.

| τ | recall@10 | MRR | nDCG@10 | key_lookup | title_dense | negative_ood no_answer (шум-фильтр) |
|---|---|---|---|---|---|---|
| **0.5** (прод) | 0.9733 | 0.6969 | 0.7657 | **1.00** | 0.9467 | 0.950 (38/40 подавлено) |
| 0.6 | 0.9733 | 0.7021 | 0.7697 | **1.00** | 0.9467 | 0.950 (38/40) |
| 0.7 | 0.9750 | 0.7067 | 0.7736 | **1.00** | 0.9500 | **0.975 (39/40)** |

Наблюдения:
- **key_lookup = 1.0 на ВСЕХ τ** — пин-иммунитет держит при любом пороге (floor не трогает exact-key). Это и есть эффект фикса.
- title_dense no_answer=0 везде (легит-позитивы не режутся). recall почти плоский (0.9467→0.95).
- negative_ood no_answer (сигнал фильтрации шума): 0.95 при τ=0.5/0.6 → **0.975 при τ=0.7**. Выше τ = больше OOD-запросов корректно уходит в no_answer.
- MRR/nDCG чуть растут с τ.

### Рекомендация τ
**Можно поднять прод-τ 0.5→0.7.** τ=0.7 строго ≥ по всем осям: key_lookup 1.0 (без изменений), title_dense +0.0033, negative_ood шум-фильтр 0.975 (best), MRR/nDCG чуть выше. Регрессий ноль.
**ЧЕСТНО: выигрыш МАРГИНАЛЬНЫЙ** — 1 негатив из 40 (при τ=0.5 просачивались 2 OOD, при 0.7 — 1). Безопасно, но не драматично. τ=0.5 уже отлично (key_lookup восстановлен). Решение по прод-τ — за лидом/пользователем (verifier прод-τ НЕ менял).

## Операционка
- Сервис ЦЕЛ весь прогон: PID 1 = `uvicorn helm.main:app`, контейнер не перезапускался, health 8444 отвечал, CPU пики ~12%, MEM <450MiB/3.8GiB. Деградации live-аналитиков нет (несмотря на параллельные фетч IVAONEHALF + docs_eval RAG#1).
- Латентность прогонов ~8.6с/кейс (I/O-bound на Qdrant+rerank), не CPU. 3 прогона параллельно ≈ same wall-clock, что один (backend сериализует), но 3 результата — выигрыш.
- Инцидент (ложная тревога): при убийстве контрольного tau0 `docker exec kill $(ps -C python)` внутри контейнера не сработал (ps не установлен в образе); eval убран с хоста `pkill -f rag2_eval`. App-сервер под uvicorn (не comm=python) под удаление не попадал. Перепроверено — сервис не пострадал.
- Сырые прогоны: `/tmp/new_tau0{5,6,7}.json` на helm; черновик-разбор в `00-Board/deploy-verify-raw-2026-07-17`.

## Relations
- relates_to [[rag2-ab-measurements]]
- relates_to [[session-state]]
</content>
</invoke>