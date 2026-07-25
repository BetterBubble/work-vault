---
status: draft
role: implementer
task: ТЗ#3 US#4 Проход B2 — pin/tests К-2/К-4 (профиль iva-brownfield-mail)
lead: lead-fr
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum-worktrees/us4-passB2-mail
branch: feat/us4-passB2-mail
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-passB2-mail
---

# US#4 Проход B2 — iva-brownfield-mail (pin/tests К-2/К-4)

Реализация по спеке [[spec-us4-passB2-pintests]] под mail-стек (C++/Qt/vmime).
Ветка `feat/us4-passB2-mail` от свежего origin/main (`c6be10a`). НЕ push.

## Что сделано (человеческим языком)

Продолжение синка от brd-authoring 0.7.2: BRD уже наследует проектные серии v2-FR
(`CT-n`/`DM-n`/`EV-n`) вниз по конвейеру — теперь их подхватывают **pin** и **tests**.

### pin-authoring (`ingredients/skills/pin-authoring/SKILL.md`)

- **К-2** — новая входная секция «v2 FR project series» + пункт 12 Document
  structure + правила. На v2-FR (`fr_skeleton: 2`) PIN реализует каждый контракт
  `CT-n` (§1.6) / модель `DM-n` (§1.7) / событие `EV-n` (§1.8) **по стабильному
  ID** конкретным шагом под mail-стек: интеграция vmime/IMAP/SMTP /
  `MailAttachmentController` (контракты), C++-структуры и Qt item-модели / машины
  состояний сообщений / матрицы операция×роль (модель), Qt-сигналы + консистентность
  / идемпотентность (события) — **либо** фиксирует расхождение. Новая таблица
  **Project series status** репортит статус каждого ID: реализован / расхождение /
  blocked.
- **К-3 уважается** — проектный раздел реализуется только при утверждённом `D-n`
  (гейт start-task, Проход C-канон). Нет `D-n` → PIN честно репортит `blocked`
  (design pending) и НЕ выдумывает контракт/модель/событие ради вида готовности.
  `blocked` — честный статус, не провал для сокрытия.
- **К-4** — новая секция «FR↔KB project-section discrepancies»: проектный
  `CT-n`/`DM-n` сверяется с реальным кодом через `kb_verify_api_exists` /
  `kb_get_code_context` (существование endpoint/сигнатуры/поля/состояния); конфликт
  проекта с реальностью = **критичное расхождение** в отдельной таблице FR↔KB,
  которое блокирует пометку «реализован». Надстроено над существующим
  API Verification Gate (тот же kb-инструментарий), не тихая правка.
- Добавлены 4 анти-паттерна (не перенумеровывать серии, не выдумывать проект при
  отсутствии `D-n`, не подгонять проект под код молча, не выводить таблицы на v1).

### tests-authoring (`ingredients/skills/tests-authoring/SKILL.md`)

- **К-2** — новая входная секция + пункт 6 Document structure + секция
  «Contract / model / event tests». На v2-FR TESTS порождает **контрактные тесты
  с тегом `Covers: CT-n`** (форма интеграции, коды статусов, ошибочные пути) +
  тесты модели/состояний для `DM-n` и сигналов/консистентности для `EV-n`. Под
  Catch2+QTest (стек mail) тег переносится в имя теста / комментарий `// Covers:
  CT-1`, чтобы трассировка дожила до раннера. Новая таблица **Project series
  coverage** репортит статус каждого ID: covered / partial / blocked.
- `blocked` из PIN остаётся `blocked` в tests — тесты на несуществующий проект
  (без утверждённого `D-n`) не пишутся. 4 новых анти-паттерна.

### manifest + CHANGELOG

- `manifest.yaml` version 0.7.2 → **0.7.3** (счётчик скиллов не менялся — контент,
  не новые ингредиенты).
- `CHANGELOG.md` — запись [0.7.3] с разбивкой по К-2/К-3/К-4 и пометкой
  backward-safe + `Content change → re-seed (ADR-0009)`.

## Backward-safe / прод-safe

Всё **аддитивно и v2-only**. v1-FR (без маркера `fr_skeleton`, нет серий CT/DM/EV)
— pin/tests работают ровно как раньше, новые секции/таблицы не выводятся.
Стек-специфика существующих pin/tests (mockup-fidelity, UI-pipeline-check,
API-verification gate, Qt/Catch2 E2E, Verified/NEW маркеры) НЕ тронута — надстройка
сверху. Не mirror: адаптировано под mail-стек (vmime/Qt/Catch2), не копия других
профилей.

## Результаты проверок

- **version-discipline** (`--diff-against origin/main`): `OK — 48 profile(s) clean`.
- **pytest целевые** (`PYTHONPATH=apps/backend`, `--noconftest`, venv py3.12):
  `test_manifest_schemas` + `test_iva_role_presets` + `test_role_install_smoke`
  → **206 passed**. Унаследованный red iva-role-web из #149 ds в этих файлах НЕ
  всплыл (он в другом наборе/требует DB — вне целевого прогона).
- `git status` — clean. Коммит один: `f38125f`.

## git diff --stat (origin/main..HEAD)

```
 templates/iva-brownfield-mail/CHANGELOG.md         |  32 +++++++
 .../ingredients/skills/pin-authoring/SKILL.md      | 106 ++++++++++++++++++
 .../ingredients/skills/tests-authoring/SKILL.md    |  80 ++++++++++++++
 templates/iva-brownfield-mail/manifest.yaml        |   2 +-
 4 files changed, 219 insertions(+), 1 deletion(-)
```

Ровно 4 файла (pin + tests + manifest + CHANGELOG). brd/start-task/другие
профили/tacticum-dev-base НЕ тронуты.

## Развилки / решения (durable)

- **К-4 без «существующей» таблицы расхождений.** Спека К-4 говорит «расширить
  существующую таблицу расхождений FR↔код». В mail-профиле pin-authoring
  отдельной таблицы расхождений FR↔код НЕ было — есть `## API Verification` gate
  (phantom/existing/new через `kb_verify_api_exists`). Решение: К-4 реализован как
  новая секция «FR↔KB project-section discrepancies», **надстроенная над тем же
  kb-инструментарием** (verify_api_exists / get_code_context), а не как правка
  несуществующей таблицы. Стек-специфика сохранена, дублирования механизма нет.
  Если лид ждал буквально другую якорную таблицу — сигнализируй, поправлю.

## Не сделано (по границам задачи)

- НЕ push / merge / PR (autonomy off, HOLD PUSH до зелёного main).
- НЕ трогал brd (готов Проходом B1), start-task (C-канон), другие профили,
  tacticum-dev-base.

## Связано
[[spec-us4-passB2-pintests]] · [[map-us4-passB-rollout]]
