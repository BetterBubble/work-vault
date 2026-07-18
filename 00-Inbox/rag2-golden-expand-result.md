---
title: rag2-golden-expand-result
type: report
permalink: tacticum/00-inbox/rag2-golden-expand-result
tags:
- rag2
- golden
- eval
- verifier
- contracts
- negative
- baseline
---

# RAG#2 golden — расширение (Task #4, результат)

Verifier, ветка `feat/rag2-golden-expand` (worktree helm-golden-expand, от main 1399787). Разметка semi — прогоном по **реальному Qdrant** (изолированный sidecar, read-only, артефакты убраны). Диф чистый (инлайн-массивы под оригинальный стиль). Retrieval-код НЕ трогал (Task #6).

## Итог: было 9 размеченных → стало 45 кейсов (3 файла)
| файл | кейсов | positive (keys) | negative/corpus-gap | aggregate (facts) | unlabeled |
|---|---|---|---|---|---|
| `rag2_golden_profile.json` | 25 | **14** | 4 | 6 | 2 |
| `rag2_golden_contracts.json` | 12 | 0 (until corpus) | 12 | — | — |
| `rag2_golden_negative.json` | 8 | 0 | 8 | — | — |
| **ИТОГО** | **45** | 14 | 24 | 6 | 2 |

## Что сделано
### 1. Схема + метрики (код)
- `metrics.py`: **`precision_at_k`** (|top-k ∩ rel|/|top-k|) + **`noise_kept_rate`** (доля negative-кейсов, где шум удержан выше порога; база для floor Task #5). Чистые функции, в `__all__`.
- `rag2_golden.py`: флаг **`expected_not_found`** (сводит алиасы `expected_no_answer`/`expected_below_floor`/`expected_not_found_until_corpus`) + `expected_source_type`. Docstring обновлён.
- Eval-suite: **53 passed** (28 + новые), регрессий нет.

### 2. Детерминированная разметка (из recon rag2-golden-ready)
- **a-status-03** — recon дал 15 epic-key (детерминированно из epics.json), НО прогон: **recall@10=0** (retrieval не поднимает отдельные эпики для статус-фильтра — это агрегат). Правило лида «не размечать ключами, которые не находятся» → **переклассифицировал в aggregate** (keys:[] + 15 эпиков в facts). Находка: агрегатные фильтр-запросы retrieval'ом не решаются (кандидат в structural/filter-ветку).
- **a-cross-01** — corpus-gap negative (IVAONEHALF отсутствует в срезе, 0 связей). `expected_not_found=true`.
- Агрегатные (keys:[]+facts): a-graph-03 (5512 Blocks), s-coverage-01 (1.0: met 477/partial 303/absent 76/planned 0 — сузил до 1.0 по фидбеку recon), s-coverage-02 (1.5 planned 416), s-feature-01 (адресная книга).

### 3. Semi-разметка (прогон по реальному Qdrant, semi-6 + бонус)
Правило лида: positive ТОЛЬКО если релевантный док реально извлечён; иначе corpus-gap negative.
**Positive (релевантный док в top-10):**
| кейс | expected_keys | ранг в прогоне | тип |
|---|---|---|---|
| a-reqtext-01 | 205489325 «Требования к IVA One 1.0 (Клиенты)» | @4 | confluence, точное совпадение |
| a-rationale-01 | 155463421 «Протокол встречи Адресная книга» | @1 | confluence, топикальный |
| s-similar-01 | IVAONE-7206 «[iOS] голосовые сообщения» | @3 | jira, точное |
| s-similar-03 | IVACS-166 «Генерация лицензий» | @3 | jira, топикальный |
| s-coverage-03 | IVAONE-3952 «[Desktop] Адресная книга: Сервис» | @3 | jira |

**Corpus-gap negative (нет релевантного дока — top нерелевантен):**
- a-rationale-02 (архитектура «Файловое хранилище» — top=«Архитектура IVA MCU», мимо)
- a-reqtext-02 (решение отключения сервисов Web/Desktop — top=«СВОД ЗАМЕЧАНИЙ», мимо)
- s-similar-02 (offline-инсталляция ЗС — нет чёткой задачи)

**Unlabeled (следующей волной, per лид «не форсируй»):** s-feature-02 (Calls/MCU), s-feature-03 (Почта) — агрегатные «что уже есть», единый ключ не выделяется.

## Baseline (расширенный набор, main 1399787, floor OFF, dotyazhka ON)
Прогон по реальному Qdrant, метки сверены.
- **Positive (14):** recall@1/3/5/10 = **0.643 / 0.857 / 0.929 / 0.929**, precision@5=0.186, MRR=0.732.
  - orig-9: recall@10=**0.889** (дотяжка ON), MRR=0.889.
  - semi-5: recall@5=**1.0** (по построению извлекаемы), recall@1=0.2, MRR=0.45.
- **Negative/corpus-gap (12 = 4 профиль + 8 negative-файл):** **`noise_kept_rate` = 1.0** (floor OFF → все 12 вернули хиты, 0 no_answer). **Это baseline «до floor» — Task #5 должен снизить с 1.0, не роняя recall на позитивах.**
- Contracts (12): все `expected_not_found_until_corpus` (корпуса api нет) — negative сейчас, flip после Task #7.

## Замечания / follow-up
- **a-status-03**: 15 epic-key детерминированы, но не retrievable (агрегат-фильтр) → в facts. Если нужен positive-кейс на epic-lookup — брать одиночный эпик по ключу (дотяжка), не список.
- Semi-positive'ы a-rationale-01/s-similar-03 — топикальные (best-effort), не «точные страницы». Помечены по релевантности title; при желании ужесточить — заменить/убрать.
- Диф: `metrics.py`(+31), `rag2_golden.py`(+36), профиль(+33/−24, только контент), 2 новых файла. Готово к ревью/мержу.

## Файлы
- `tests/data/rag2_golden_{profile,contracts,negative}.json`, `src/helm/eval/{metrics,rag2_golden}.py` (worktree helm-golden-expand).
- Связанные: `00-Inbox/rag2-golden-ready` (recon), `rag2-eval-baseline-v2`, `rag2-retrieval-miss-diag`.
