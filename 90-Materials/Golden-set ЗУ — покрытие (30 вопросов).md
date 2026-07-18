---
title: Golden-set ЗУ — покрытие (30 вопросов)
type: report
permalink: tacticum/90-materials/golden-set-zu-pokrytie-30-voprosov
tags:
- materials
- golden-set
- eval
- coverage
- zu
---

# Golden-set ЗУ — покрытие (30 вопросов клиента ↔ индекс, tenant cifragen)

**Источник на диске:** `~/tacticum/_analysis/golden_coverage.md`
**Тип:** анализ покрытия (read-only). ⚠️ Тексты эталонов/документов не выносятся — только метки тем, slug-идентификаторы, статистика.

## Метод
Эталонные ответы из `Тест Базы знаний.xlsx` сверены с содержимым корпуса (71 документ через document_processing → slug↔текст; TF-IDF-overlap + proximity). Реальный прогон поиска НЕ запускался.

## Главный вывод
Из 15 «отложенных» вопросов **13 реально покрыты** — упущенный источник `algoritm-deistvii-ks` (call-центровый свод) + `11-yuridicheskii-obozrevatel`. Контентных пробелов почти нет; узкие места — формат/извлечение, не отсутствие знания.

## Сводка
| Класс | Сколько |
|---|---|
| A — покрыто (документ содержит ответ) | 26/30 (87%) |
| B — частично | 4/30 (13%) |
| C — не покрыто | 0/30 (0%) |

## Relations
- part_of [[90-Materials]]
- relates_to [[Golden-set ЗУ — выверка по истине]]
- relates_to [[Руководство: составление golden-set]]
- relates_to [[Корпус ЗУ (КЦ папка) — индекс]]
