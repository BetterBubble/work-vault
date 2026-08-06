---
status: draft
role: explorer
topic: lead-fr / ТЗ#3 — карта текущего состояния iva-fr-analyst
repo: /Users/bubblemac/tacticum/tacticum-dev (main)
date: 2026-07-24
permalink: tacticum/00-board/explore-fr-analyst-baseline-1
archived-at: 2026-07-31 17:27
---

# Разведка: baseline профиля iva-fr-analyst (для планирования US#1–3)

Read-only карта. Пути абсолютные. Числа/имена сверены с файлами.

## 0. ГЛАВНЫЙ ГАРДРЕЙЛ (читать первым) — mirror-протокол

`templates/_mirrors.yaml` (US #714, ADR-0059 Решение 7): **iva-fr-analyst — это ЗЕРКАЛО (mirror), владелец (owner) = iva-analysis-base.**
Пара:
```
- owner: iva-analysis-base
  mirror: iva-fr-analyst
  ingredients: [api-contracts-discovery, design-system-discovery, fr-authoring, mockup-authoring, start-feature, update-feature]
```
Правило дословно: «задекларированные здесь ингредиенты обязаны быть БАЙТ-В-БАЙТ одинаковыми у владельца (лейна) и в зеркале. **Правки — только у владельца; зеркало обновляется тем же PR.**»
Проверено фактически: `fr-authoring/SKILL.md` и `api-contracts-discovery/SKILL.md` в iva-fr-analyst и iva-analysis-base — **байт-в-байт идентичны** (diff -q → IDENTICAL).
Энфорсеры:
- `/Users/bubblemac/tacticum/tacticum-dev/scripts/check_mirror_sync.py` (CI: profile-version-discipline.yml)
- `/Users/bubblemac/tacticum/tacticum-dev/apps/backend/tests/catalog/test_role_replacement_parity.py`

**Следствие для лида:** любая правка fr-authoring / api-contracts-discovery (US#1–3, «формат контрактов», DM/EV, скелет) физически должна лечь В ОБА профиля одним PR, ИЛИ надо осознанно закрыть пару в `_mirrors.yaml` (депрекация зеркала). Это прямо конфликтует с формулировкой задачи «iva-analysis-base НЕ трогаем». Требует решения лида — см. открытые вопросы.

## 1. Профиль iva-fr-analyst

- Manifest: `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-fr-analyst/manifest.yaml`
- **version: 0.1.10** (совпадает с планом), schema_version «2», deprecated: false, maintainer mr.diaret@ya.ru.
- `depends_on` — ОТСУТСТВУЕТ (standalone-профиль, ingredients лежат в нём самом; не лейн).
- persona.role: `requirements-analyst`. profiles: trial == full (single-tier v0.1).
- Состав ingredients:
  - skill_spec (4): `fr-authoring`, `api-contracts-discovery`, `design-system-discovery`, `mockup-authoring`
  - command_spec (2): `start-feature`, `update-feature`
  - mcp_server_spec (4): `helm-analyst` (https://helm.tacticum.ru/mcp/analyst), `iva-mcp` (https://mcp.tacticum.ru/iva-read/mcp, ТОЛЬКО чтение), `iva-atlassian-write` (stdio uvx mcp-atlassian, личные PAT), `tacticum-mcp` (https://mcp.tacticum.dev/mcp, kb_*)
  - instruction_pack (3): claude-md-fragment, codex-agents-md, codex-config-toml
- CHANGELOG: `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-fr-analyst/CHANGELOG.md`. Последняя запись **[0.1.10] — 2026-07-22** (мокапы аналитика, US #720). Хронология значимого: 0.1.9 — жанр ТЗ (US #718); 0.1.5 — двухчастная структура FR (US #715); 0.1.4 — write-канал + диаграммы (US #699/#713); 0.1.2 — probe-first JUMP (Taiga #698).

## 2. Скилл fr-authoring

Файл: `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md` (622 строки).

**Структура FR — уже ДВУХЧАСТНАЯ (не плоская П.A…П.J):**
- Часть 1 «Постановка» (для человека): §1.1 Суть, §1.2 Как это будет работать (+Диаграммы), §1.3 Ключевые решения, §1.4 Что входит/не входит, §1.5 Что уже есть в системе, §1.6 Требования к контрактам, §1.7 Что осталось выяснить.
- Часть 2 «Приложение» (факты): **П.A ФТТ, П.B UC, П.C Q-реестр, П.D Решения, П.E Что выяснено по коду, П.F Проверка контрактов (пробы/результаты), П.G НФТ, П.H Интерфейсные/UI, П.I Метаданные, П.J Источники.** anchor-макросы p-a…p-j + src-n.
- Порядок сборки: сначала Приложение, потом Часть 1 как выводы.

**Второй жанр — ТЗ** (крупная фича с этапностью, образец «ТЗ IVA Disk 1.5» page 211846538), линейная структура из 18 разделов. Серии в ТЗ: `FR-nn / EV-nn / UC-nn / D-nn` (с ведущим нулём). Выбор жанра — на гейте вопросов (шаг 2 п.4).

**Поле fr_skeleton / версия скелета — НЕ найдено** (grep по fr_skeleton/skeleton_version/«версия скелета» пусто). Скелеты вкладены прямо в SKILL.md как markdown-блоки, без версионирования.

**Оркестрация — 9 шагов** (раздел «Поток (9 шагов)»): 1 Понять требование → 2 Gap-чек/Q-реестр (+гейт жанра п.4, +гейт мокапов п.5) → 3 Затрагиваемые системы (контейнер+трек) → 4 Проверка по коду (kb_*) → 5 Прецедент → 6 Координация/срок → 7 Собрать черновик (+7б Диаграммы drawio, +7в Мокапы) → 8 Ревью/фиксация D-n → 9 Публикация.

**Идентификаторы:** FT-n, UC-n, Q-n, D-n (в жанре ТЗ — FR-nn/EV-nn/UC-nn/D-nn). Правило: номер выдан один раз, не переиспользуется.

**Правила честности §1–§12** (раздел «Правила честности», строки 593–622). §2 (ключевой инвариант — запрет wire-имён), дословно:
> «**Не отмывать выдумку через маркировку.** Конкретные имена параметров, enum-значения, команды и пути (`recipientIds`, `action: "recall"`, `messageRecall`, `POST /messages/{id}/recall`), которых нет в источнике, ЗАПРЕЩЕНЫ даже с пометками «выведено из требования» / «предполагаемая структура» / «по паттерну» — разработчик скопирует их как спеку. Пиши требованиями без wire-имён: «операция должна принимать идентификатор письма и признак замены».»

Прочие §: §1 не выдумывать (каждый факт [n]); §10 код-факты только из kb_*; §11 диаграммы = визуализация, не источник; §12 Часть 1 = выводы, Приложение = факты (расхождение — дефект Части 1).

## 3. Скилл api-contracts-discovery

Файл: `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-fr-analyst/ingredients/skills/api-contracts-discovery/SKILL.md` (121 строка).

- **Версии формата «3.1» / серии «CT-n» — НЕ найдено** (grep по CT-/формат 3/версия формата пусто). Скилл структурирован как «Карта источников (проверено 2026-07-16)» (таблица из 6 источников) + 5-шаговый алгоритм + «Правила честности» (8 пунктов). Никакой нумерованной серии контрактов и версии формата в тексте нет.
- **REST vs JUMP — оба контура поддержаны.** REST/OpenAPI: реестры MCU beta/alpha.hi-tech.org (IVA Clients API v2.30.0 ~315 операций; integration ~54) и «IVA One API Reference» Redoc http://10.0.204.7:6336/ (~67 операций). JUMP (почта/календарь/контакты): канон distrohost.
- **Конвенции distrohost цитируются** через JUMP-источник (строка 25): канон `http://distrohost.msk/Docs/Sessions.html` + дубль Eva DOC-000245 + wiki-указатель page 208701103 (имена команд messageSync/messageUpdated/mailboxCreate/calObj*, auth Bearer stateId+JWT). Шаг 3 — **probe-first**: статус «Недоступен» только по фактически упавшему запросу (фикс Taiga #698).
- Негативный результат оформляется таблицей negative evidence (шаг 5), не серией CT-n.

## 4. Навыки для модели данных (DM-n) и событий (EV-n)

**Отдельных скиллов НЕТ.** В `ingredients/skills/` всего 4 директории: api-contracts-discovery, design-system-discovery, fr-authoring, mockup-authoring. Скиллов dm-*/data-model/event-* нет.

Однако концепты частично УЖЕ живут внутри жанра ТЗ в fr-authoring/SKILL.md:
- **EV-nn** — раздел «## 9. События и консистентность» скелета ТЗ (серия EV-nn: публикация, идемпотентность, сверка, порядок).
- Модель данных — раздел «## 7. Доменные модели» скелета ТЗ (ролевая матрица операции×роли, «предлагаемая, требует подтверждения (Q-n)»), «## 8. Состояния сущности».
- В жанре FR (двухчастном) отдельных DM/EV разделов НЕТ.

Т.е. «DM-n/EV-n как навыки» — новые; но при проектировании учесть, что EV-nn/доменные модели уже присутствуют как разделы ТЗ-скелета (риск дублирования нумерации/семантики).

## 5. ROLE_LANES / single-owner тест

Файл: `/Users/bubblemac/tacticum/tacticum-dev/apps/backend/tests/catalog/test_iva_role_presets.py` (ADR-0057, US #710, E-LANES S5).
- `ROLE_LANES` (dict, строки 44–112) — 13 ролей. **iva-fr-analyst в ROLE_LANES ОТСУТСТВУЕТ** (это не роль и не лейн — это standalone/зеркало).
- Ближайшая роль: `iva-role-analyst: ["tacticum-core-base", "iva-analysis-base"]` — именно она композирует лейн iva-analysis-base (владельца зеркала fr-analyst). Подтверждено: depends_on iva-role-analyst = [tacticum-core-base, iva-analysis-base].
- Тест `test_single_owner_lanes_are_pairwise_disjoint` (строки 210–219): для каждой роли лейны, которые она композирует, попарно disjoint по ingredient_id (кроме `KNOWN_OVERRIDES`, где только iva-role-architect: {iva-atlassian-write}). Плюс `test_golden_parity_union_equals_sum_of_lanes` (строгая переформулировка disjointness).
- Лейны — единица авторинга, роль — единица установки. iva-fr-analyst НЕ участвует в этой disjointness-проверке (он вне ROLE_LANES), поэтому добавление скиллов в него этот тест не ломает НАПРЯМУЮ — но ломает mirror-parity (см. §0), т.к. владелец iva-analysis-base — лейн внутри ROLE_LANES.

## 6. Заморожённые соседи

- `iva-analysis-base` — ЕСТЬ. `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-analysis-base/manifest.yaml`, version 0.1.3, deprecated: false. Это ПРОЦЕСС-ЛЕЙН аналитики. Его skills: adr-authoring, api-contracts-discovery, brd-authoring, design-system-discovery, **fr-authoring**, mockup-authoring, multi-container-pin-authoring, pin-api-verification, pin-upstream-dependency-check, tests-authoring (+ commands start-feature/update-feature/prepare-tz/start-task/approve-docs, + system-analyst, tacticum-workflow).
- `iva-role-analyst` — ЕСТЬ. version 0.1.1, deprecated: false, depends_on: [tacticum-core-base, iva-analysis-base]. Своих skills-директорий нет (тонкая роль-обёртка, несёт только паки).
- Связь с iva-fr-analyst: НЕ через наследование (depends_on у fr-analyst нет), а через **mirror-протокол** (§0): iva-analysis-base — ВЛАДЕЛЕЦ 6 общих ингредиентов, iva-fr-analyst — зеркало. Т.е. «замороженность» соседей и «редактирование только у владельца» — в противоречии, требует решения лида.

## 7. Точка пересечения с lead-ds (iva-web-brownfield)

Файл: `/Users/bubblemac/tacticum/tacticum-dev/templates/iva-web-brownfield/manifest.yaml`, version 0.2.1, deprecated: false (Angular 21 / Nx two-repo). Само оно — зеркало владельца iva-web-development-base (см. _mirrors.yaml).
Skills (26 директорий), пересекающиеся по теме с аналитикой/постановкой:
- **brd-authoring, pin-authoring, tests-authoring, adr-authoring** — постановочные навыки (та же семья, что в iva-analysis-base: brd-authoring/adr-authoring/tests-authoring; PIN там в виде multi-container-pin-authoring).
- **design-system-discovery** — присутствует и там, и в iva-fr-analyst (общий по имени навык; в fr-analyst он в зеркале analysis-base).
- Смежные: pin-api-verification, pin-upstream-dependency-check, design-token-usage, ui-mockup-match, openapi-codegen-pipeline.
Зона пересечения lead-fr ↔ lead-ds: navыки brd-authoring / pin-authoring / tests-authoring / design-system-discovery / api-contracts (контрактная фактура). Важно: iva-web-brownfield — мусор dev-конвейера (Angular), а не аналитики; общий знаменатель — постановочные authoring-навыки и design-system-discovery.

---

## Открытые вопросы для лида

1. **Mirror-конфликт (критично).** Задача велит «iva-analysis-base НЕ трогать», но `_mirrors.yaml` делает его ВЛАДЕЛЬЦЕМ fr-authoring/api-contracts/… и требует правок ТОЛЬКО у владельца + байт-в-байт зеркало. Любая правка US#1–3 в iva-fr-analyst без синхронной правки iva-analysis-base упадёт в CI (check_mirror_sync.py, test_role_replacement_parity.py). Варианты для решения: (а) правим у владельца iva-analysis-base + зеркалим fr-analyst одним PR; (б) осознанно закрываем пару в _mirrors.yaml (депрекация/расхождение). Нужно решение до планирования US.
2. **Формат «3.1» и серия «CT-n» из плана — в коде НЕ существуют.** api-contracts-discovery не версионирован и не использует CT-n. Лиду уточнить: US вводит НОВЫЙ формат/серию CT-n «с нуля» или план опирался на устаревшее представление?
3. **DM-n/EV-n — частично уже есть в ТЗ-скелете** (раздел 7 «Доменные модели», раздел 9 «События EV-nn»). Новые навыки DM/EV должны либо расширять эти разделы, либо переопределять — риск дублирования нумерации EV-nn. Лиду решить границу «новый навык vs существующий раздел ТЗ».
4. **Двухчастная структура FR уже внедрена (0.1.5), «Часть 1/Часть 2» существует** — план описывал это как целевое? Если US#1 про «деление на части» — оно уже сделано; уточнить фактический дифф US относительно 0.1.10.
5. Куда попадают DM/EV в жанре FR (двухчастном)? Сейчас в FR-скелете нет отдельных DM/EV разделов (только в ТЗ). Нужен ли новый раздел в П.* Приложения FR.