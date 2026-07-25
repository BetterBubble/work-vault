---
status: draft
role: lead-fr
topic: ТЗ#3 US#1+US#2 — материалы PR (пара, вариант А владелец+зеркало), прошла critic+гейт
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum/tacticum-dev-us1-owner
branch: feat/us1-fr-authoring
commits: c028882 (US#1) + 43aaf68 (US#2)
date: 2026-07-24
permalink: tacticum/00-board/pr-us1-us2-ready
---

# US#1+US#2 — материалы PR (ждёт fidelity-сверки ГД → президента)

Вариант А: правки в ВЛАДЕЛЬЦЕ `iva-analysis-base`, зеркало `iva-fr-analyst` байт-в-байт → обе роли (обучение + design-capable) получают способность. autonomy off + READ-ONLY: **push/мерж — президент лично**.

## Заголовок PR
`feat(analyst): FR v2 (двухзонная §2 + валидатор границы) + контрактный формат 3.1/CT-n — ТЗ#3 US#1+US#2`

## Описание PR
**Что делает.** Даёт аналитику проектировать новые требования (to-be), а не только описывать as-is — по утверждённому ТЗ Солонко (§2, §3). Правки в общих скиллах владельца `iva-analysis-base`, зеркалятся в `iva-fr-analyst` (mirror-протокол) → способность получают ОБЕ роли: `iva-role-analyst` (обучение аналитиков) и `iva-fr-analyst` (design-capable).

**US#1 — fr-authoring (структура FR v2):**
- Двухзонная §2: Часть 2 (Приложение, as-is) — правило честности без ослаблений; проектные разделы Части 1 (§1.6/§1.7/§1.8, to-be) — проектные wire-имена разрешены под ТРЕМЯ предохранителями одновременно (основание в Ч.2 + плашка «требует утверждения dev+CTO»/Q-n + механический валидатор границы). Выдумка (имя без основания) запрещена везде.
- Механический валидатор границы (Шаг 7 + стоп-гейт Шага 9): основание обязательно ВСЕГДА (Q-n не заменяет); нарушение = дефект, публикация стоп.
- Переезд FT-n/UC-n в Часть 1 (§1.4/§1.5), UI в §1.9; переномерация 1.1–1.11 по ТЗ §2.3; маркер `fr_skeleton: 2` (переходная совместимость конвейера).

**US#2 — api-contracts-discovery (контрактный формат 3.1):**
- Формат 3.1: реестр изменений (+Потребители/Контур) + по-операционные секции (параметры запроса/ответа, примеры JSON, коды с семантикой) + права/403 + лимиты/429 + async/202 + трассировка CT-n/FT-n.
- Серия CT-n; чеклист полноты 3.2; порядок проектирования 3.3 (контракт после CT-n+проб; конвенции с цитатой; альтернативы в П.F); JUMP-контур (probe-first сохранён).
- Правила честности согласованы с §2 fr-authoring (П.F строго as-is; проектный контракт в §1.6 под теми же 3 предохранителями) — релаксация не расходится между скиллами.

**Границы/прод-safe.** Строго аддитивно: as-is флоу аналитика (шаги разведки, Часть 2, форматы FT/UC) не сломан, старые FR v1 работают (маркер скелета). Владелец правится штатно (заморозки нет).

**Файлы (8, коммиты c028882 + 43aaf68):** fr-authoring/SKILL.md (оба профиля) + api-contracts-discovery/SKILL.md (оба профиля) + manifest.yaml (оба: iva-analysis-base 0.1.3→0.1.4, iva-fr-analyst 0.1.10→0.1.11) + CHANGELOG.md (оба).

**Проверки (перепроверены controller-гейтом + повторно на финальном коммите):**
- `check_mirror_sync.py` → OK — 62 (оба скилла байт-идентичны owner↔mirror).
- `diff -q` fr-authoring и api-contracts owner↔mirror → identical.
- `check_profile_version_discipline.py --diff-against origin/main` → OK — 46 clean.
- pytest целевые (schemas + parity + role_presets + install_smoke) → 288 passed, 0 failed.

**Проверки лида:** critic §2 US#1 (дыра валидатора закрыта в ii + стоп-гейте) + critic US#2 (честность не разошлась, формат 3.1/3.2/3.3 дословно по ТЗ §3) + controller-гейт пары PASS 7/7. Все AI-подписи отсутствуют.

## Что нужно от президента (лично с лидом)
1. Разрешение на `git push origin feat/us1-fr-authoring`.
2. Ревью + мерж PR (одним PR — US#1+US#2 связаны куплингом §1.6/честность).

## После мержа
US#3 (навыки DM/EV) — но ПЕРЕД правкой теста подниму ГД parity-развилку (REPLACEMENTS allowlist = test-matrix). Затем US#5, US#4 (через ГД). Деплой всей capability — в конце (teststand→прод по runbook).

## Связано
[[impl-us1a-owner-mirror]] · [[impl-us2-api-contracts]] · [[critic-us1a-recheck]] · [[critic-us2-format31]] · [[gate-us1-us2-pair]]
