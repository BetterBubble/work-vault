---
status: draft
tema: US#4 Проход D — iva-web-brownfield (последний проход конвейера)
ot: implementer
komu: lead-fr
data: 2026-07-25
worktree: /Users/bubblemac/tacticum-worktrees/us4-passD-web
vetka: feat/us4-passD-web
commit: d0572c3
permalink: tacticum/00-board/impl-us4-pass-d-web
---

# Проход D (web) — приём FR v2 конвейером, готово

Полный профильный проход `iva-web-brownfield`: brd/pin/tests/start-task учатся читать
FR v2 (маркер `fr_skeleton: 2`) и наследовать проектные серии CT-n/DM-n/EV-n. Всё
аддитивно и backward-safe (на v1-FR — прежнее web-поведение). 1 коммит, 6 файлов.

## (а) Вердикт расхождения web-brd / web-start-task

- **web-brd — КАНОН-ВЫРОВНЕН, чистый синк (как mail/rn).** До правки web-brd был
  побайтово равен каноническому `tacticum-dev-base` brd БЕЗ v2-надстройки и БЕЗ
  какой-либо web-специфики в самом brd (0 web-строк; web-специфика профиля живёт в
  pin/start-task, не в brd). → Взял канонический brd **verbatim** (теперь web-brd
  идентичен канону, проверено `git diff --no-index`). Никакого «чужого layout» не
  навязано — для чистого-синк профиля канон И ЕСТЬ v1-layout (v1 = П.A/П.B), ровно
  как решено для mail/rn. Дивергентный merge (как kmp с его русским разделом «FR на
  входе») здесь НЕ требовался.
- **web-start-task — ДИВЕРГЕНТЕН, merge-вставка.** Нёс богатую web-специфику
  (step-0 local-knowledge gate, детект репо iva-one/iva-connect по package.json,
  список design-скиллов с `pin-ui-pipeline-check`, верификация `@ivcs/*` против
  installed-версии, design-токены) — НО был БЕЗ К-5 и К-3. → Вставил канонические
  блоки К-5 (детекция `fr_skeleton`) и К-3 (plate-based гейт D-n) verbatim в
  неконфликтующее место (между «Context first» и «Target-dir resolution»), вся
  web-специфика сохранена. Остаточный `git diff --no-index` vs канон = ровно
  web-специфика (проверено). v1-локация FT/UC согласована: и web-brd, и
  web-start-task теперь говорят v1 = Приложение П.A/П.B (совпадают, нет kmp-рассинхрона).

## (б) Тронутые web authoring-файлы (поверхность коллизии с ds PR-C для ребейза)

| Файл | Тип правки | Суть |
|------|-----------|------|
| `ingredients/skills/brd-authoring/SKILL.md` | replace verbatim | К-1/К-5, = канон |
| `ingredients/skills/pin-authoring/SKILL.md` | merge (вставки) | К-2/К-3/К-4, item 12 + Rules + секция Project-design realization + anti-patterns, адаптировано под web-стек |
| `ingredients/skills/tests-authoring/SKILL.md` | merge (вставки) | К-2, item 5 + секция Contract tests + Rules + anti-patterns, TestBed+Jest/Vitest |
| `ingredients/commands/start-task.md` | merge (вставки) | К-5/К-3, блоки канона в неконфликтующее место |
| `manifest.yaml` | 1 строка | version 0.4.0 → 0.4.1 |
| `CHANGELOG.md` | +блок | [0.4.1] по-русски |

**Для твоего ребейза (ds PR-C → web 0.5.0 → ребейз в 0.5.1):** поверхность коллизии —
`manifest.yaml` (version-строка) и `CHANGELOG.md` (заголовок [0.4.1]). ds PR-C
(`ui-mockup-match` Figma-режим по CHANGELOG 0.4.0) касается ДРУГИХ файлов
(`ui-mockup-match`), authoring-скиллы brd/pin/tests/start-task он НЕ трогает → по
телу skill-файлов коллизии с PR-C НЕ ожидаю, только version/CHANGELOG.

## (в) Проверки

- **git show --stat** (commit d0572c3):
  - CHANGELOG.md +50, start-task.md +36, brd-authoring +47/-…, pin-authoring +78,
    tests-authoring +56, manifest.yaml 1 строка. Итого 6 файлов, 265 insertions, 4 deletions.
- **Версии:** manifest `version: "0.4.1"` (было 0.4.0). ⚠️ временно — после мержа
  ds PR-C (web → 0.5.0) ты ребейзишь в 0.5.1.
- **version-discipline** (`--diff-against origin/main`): `OK — 48 profile(s) clean`.
- **pytest каталог** (schemas + role_presets + install_smoke):
  `206 passed in 3.94s`. Прогон через `uv run` с доп-зависимостями
  (`pytest-asyncio alembic sqlalchemy` — общий conftest импортирует их на уровне
  модуля; venv backend в worktree/main отсутствует, Docker-фикстуры не стартовали,
  т.к. целевые тесты их не запрашивают) + `PYTHONPATH=apps/backend`.

## Границы соблюдены
Только 6 файлов профиля `iva-web-brownfield`. `tacticum-dev-base` и другие профили
НЕ тронуты. Backward-safe везде. Секретов/AI-подписей нет. Не мержил, не пушил.
</content>