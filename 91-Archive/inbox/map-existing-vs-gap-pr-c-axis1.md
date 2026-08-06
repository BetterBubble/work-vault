---
title: map-existing-vs-gap-pr-c-axis1
type: note
permalink: tacticum/00-board/map-existing-vs-gap-pr-c-axis1-1
status: draft
role: explorer
for: lead-ds
tz: figma-ds ТЗ#1, PR-C = ось-1
repo: ~/tacticum/tacticum-dev (канон) + worktree ~/tacticum/tacticum-dev-web-mockup
  (ветка feat/ds-web-mockup-figma, содержит PR-A+PR-B)
tags:
- ds-web
- figma-ds
- pr-c
- axis1
- explore
- gap-map
archived-at: 2026-08-03 11:16
---

# Карта existing-vs-gap — PR-C (ось-1: несколько ДС по репо/поверхности) + G7/G8/iva-core

Вход для плана лида. Строить ТОЛЬКО реальный gap, не дублировать, знать mirror-scope. Read-only разведка.

## TL;DR (5 пунктов)

1. **G7 не единый.** `design-system-discovery` существует в **7 физических копиях** с **6 разными хэшами** — они содержательно РАЗНЫЕ. 3 всё ещё хардкодят `iva-web` (web-brownfield, rn-brownfield, mail), 3 уже нет (analysis, fr-analyst, ui-base), 1 (kmp) даже читает framework_hint. **Ни одна не знает про поверхность/iva-core.** «Пофиксить G7» = не одна правка, а осознанный выбор канона + приведение web-копий.
2. **Mirror-scope узкий.** `_mirrors.yaml` CI-запирает `design-system-discovery` **только в ОДНОЙ паре**: owner `iva-analysis-base` ↔ mirror `iva-fr-analyst` (байт-идентичны, 00914d6d). Остальные 5 копий **НЕ под CI-локом** — каждая standalone. Для web-поверхности НЕТ owner/mirror-лока → лид сам назначает канон.
3. **Overlap с PR-B реальный, но узкий.** PR-C и PR-B на одной ветке трогают одни файлы `iva-web-brownfield/CHANGELOG.md` + `manifest.yaml` (разные skill-тела: PR-B=ui-mockup-match, PR-C=design-system-discovery — конфликта тел нет). Нужна сериализация только по CHANGELOG/manifest.
4. **G8 gap подтверждён.** Есть только `docs/user_manuals/iva-kmp-figma-mapping-quickstart.md` (108 строк, 10 секций). Web-версии НЕТ. Новый файл — чистый, ни с чем не пересекается.
5. **iva-core template-side почти пуст.** НЕТ `design-systems/iva-core/`, НЕТ скилла iva-core. Наш PR-C = тонкий скилл + router-note. Серверная ДС `iva-core` + словарь code-bindings = ОТЛОЖИТЬ (DS-команда/сервер).

---

## 1. Реальный GAP PR-C (по пунктам)

### G7 — фикс design-system-discovery (ось-1 web)
**EXISTING (что уже есть, НЕ дублировать):**
- `templates/iva-web-brownfield/ingredients/skills/design-system-discovery/SKILL.md` (хэш 8358daba) — **баг из ТЗ**: строка 44 хардкодит ``iva-web` for web and Electron/desktop surfaces``, использует старый `design_get_tokens` (без groups-probe), НЕ читает platform/framework_hint, НЕ знает про поверхность. Это ось-1 web-цель.
- `templates/tacticum-ui-base/ingredients/skills/design-system-discovery/SKILL.md` (хэш 208b12a0) — **SoT новой композиции ADR-0056/0059** (web/mobile UI-кластер, подключается через `depends_on`). Хардкод `iva-web` УЖЕ убран, но введено допущение `«Iva DS unified across surfaces»` (2 вхождения) — **прямо противоречит ось-1** (iva-core = отдельная ДС на конференц-поверхности!). framework_hint по-прежнему НЕ читает, surface-routing НЕТ.
- `design-systems/iva-web/design-system.yaml` — **дрейф подтверждён**: `platform: web` (стр.17) + `framework_hint: react` (стр.18) при том что iva-one — Angular. Это seed-метаданные для сервера (в этом же репо).

**GAP (реальный, template-side):**
- в web-копии(ях) заменить хардкод/`unified`-допущение на выбор по `platform`/`framework_hint` (сервер уже отдаёт через `design_list_systems`) + идентичность репо (`package.json name`);
- добавить осведомлённость о **поверхности** (основной UI iva-one → `@iva/design-system`; конференц/MCU/VCS → `iva-core`) — **router-note** (см. п.4);
- опциональный `default_design_system_id` из `.tacticum/context.yaml` (сейчас live-файл разошёлся со схемой скилла; **в шаблоне репо `.tacticum/context.yaml` НЕТ** — это live-only файл, template-схему держит скилл `tacticum-context`);
- >1 подходящей ДС → спросить пользователя (ADR-0026).

**Состояние всех 7 копий (карта дрейфа):**
| профиль | хэш | хардкод iva-web | «unified» | читает fw/platform | surface/iva-core |
|---|---|---|---|---|---|
| iva-analysis-base (owner пары) | 00914d6d | нет | нет | нет | нет |
| iva-fr-analyst (mirror пары) | 00914d6d | нет | нет | нет | нет |
| **iva-web-brownfield** | 8358daba | **ДА** | нет | нет | нет |
| tacticum-ui-base (SoT) | 208b12a0 | нет | **ДА(2)** | нет | нет |
| iva-kmp-brownfield | 936d6ce2 | нет | ДА(2) | **ДА(1)** | нет |
| iva-rn-brownfield | 72273262 | ДА | нет | нет | нет |
| iva-brownfield-mail | 9c7d6ab6 | ДА | нет | нет | нет |

Примечание: `design-system-discovery` фигурирует в **11 manifest.yaml**, но физических папок **7** — профили без папки (`iva-web-development-base`, `tacticum-dev-base`, `firebird-web-brownfield`, `iva-ios-brownfield`, `iva-kmp-development-base`) получают скилл через `depends_on: tacticum-ui-base`. То есть **для новой композиции фактический owner web-скилла = tacticum-ui-base**, а `iva-web-brownfield` — старый монолитный профиль переходного периода (не deprecated, но заменяемый).

### G8 — web figma-quickstart
**EXISTING:** только `docs/user_manuals/iva-kmp-figma-mapping-quickstart.md` (108 строк). Web-версии НЕТ (подтверждает карту Сц.1/2).
**GAP:** новый `docs/user_manuals/iva-web-figma-mapping-quickstart.md` (или аналог) по образцу KMP. Форма KMP: 10 секций — Что понадобится / Шаг0 MCP в Figma (пользователь) / Шаг1 профиль+промпт (агент) / Шаг2 подключить Figma MCP (пользователь) / Шаг3 разовое правило репо / Макеты→компоненты ДС / Шаг4 проверка связки / Как пользоваться / Если пошло не так. Чистый новый файл, overlap = 0.

### iva-core — тонкий скилл + router (template-side)
**EXISTING:** НЕТ. `design-systems/iva-core/` отсутствует; скилла iva-core в templates НЕТ. Единственные упоминания `iva-core` — в KMP как **source-репо**: `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-{screen-port,source-reference}/SKILL.md` + его manifest/CHANGELOG (не наша зона).
**GAP template-side (наш PR-C):** тонкий отдельный скилл ДС `iva-core` + **router-note** в web/shared-скилле (конференц/MCU-поверхность → iva-core; основной UI → @iva/design-system). Скилл тонкий — у репо iva-core нет .mdx/AGENTS.md (витрина = demo-app, доки = CHANGES.md).

---

## 2. MIRROR-план design-system-discovery (owner + пары + правило ГД)

- **Файл декларации:** `templates/_mirrors.yaml`. Сверка CI: `scripts/check_mirror_sync.py` (workflow `profile-version-discipline.yml`) + `tests/catalog/test_role_replacement_parity.py`. Проверяет **только задекларированные пары**, байт-в-байт, папку скилла целиком.
- **Единственная пара с design-system-discovery:** owner `iva-analysis-base` ↔ mirror `iva-fr-analyst` (пара «iva-analysis-base ↔ iva-fr-analyst», ingredients включают design-system-discovery). Обе байт-идентичны (00914d6d) — лок работает.
- **web-копии (iva-web-brownfield, tacticum-ui-base) в _mirrors.yaml НЕ задекларированы** для этого скилла → CI их НЕ запирает друг с другом. Отсюда их дрейф (8358daba vs 208b12a0) — легален, не нарушение.
- **Правило ГД (как править):** правка — у **владельца**, зеркало обновляется тем же PR; чужой drift не трогаем. Практически для PR-C:
  1. Если правим web-поверхность через **новую композицию** → канон = `tacticum-ui-base` (владелец UI-кластера по depends_on). Правим его; `iva-web-brownfield` — отдельное решение (он не в паре, но фактически заменяем; лид решает: синхронизировать вручную или пометить осознанный дрейф).
  2. Если PR-C трогает `iva-analysis-base` копию → **обязан** тем же PR обновить `iva-fr-analyst` (CI-лок, иначе красный build). Но analyst — не web-цель; вероятно НЕ трогаем.
  3. **НЕ чинить чужой drift скопом** (rn-brownfield/mail всё ещё хардкодят iva-web — это не ось-1-web-цель PR-C; отдельные заходы или осознанно вне scope).
- **Вывод для лида:** назначить канон web-скилла (рекомендация разведки: `tacticum-ui-base` как SoT новой композиции) и явно решить судьбу standalone-копии `iva-web-brownfield`. Пары для новой пары в _mirrors.yaml сейчас НЕТ — если лид хочет CI-лок web-копий, это отдельное решение (расширить _mirrors), но это уже правка процесса, не gap PR-C.

---

## 3. Overlap с PR-B (сериализация)

Worktree `~/tacticum/tacticum-dev-web-mockup` (ветка `feat/ds-web-mockup-figma`) уже содержит PR-A+PR-B. `git diff --stat origin/main..HEAD` трогает ровно:
- `templates/iva-web-brownfield/CHANGELOG.md` (+48)
- `templates/iva-web-brownfield/ingredients/skills/ui-mockup-match/SKILL.md` (+153) ← PR-B
- `templates/iva-web-brownfield/manifest.yaml` (±4)

**Пересечение PR-C ↔ PR-B:**
- **Тела скиллов НЕ пересекаются:** PR-B = `ui-mockup-match/SKILL.md`; PR-C(G7) = `design-system-discovery/SKILL.md` — разные папки, конфликта нет.
- **Пересекаются 2 файла:** `iva-web-brownfield/CHANGELOG.md` и `iva-web-brownfield/manifest.yaml` — оба PR будут дописывать CHANGELOG и, возможно, бампить версию в manifest.
- **Если PR-C на ТОЙ ЖЕ ветке** (аддитивные коммиты поверх PR-B) → merge-конфликта нет, только следить за порядком записей в CHANGELOG/manifest. **Рекомендация: PR-C коммитить после закоммиченного PR-B.**
- **Если PR-C отдельной веткой от main** → гарантированный конфликт по CHANGELOG.md + manifest.yaml `iva-web-brownfield` → нужна сериализация (мержить PR-B первым, ребейз PR-C).

---

## 4. iva-core: template-side vs server-side/RE

**Template-side (наш PR-C):**
- тонкий скилл ДС `iva-core` (скилл-папка в нужном профиле/базе);
- **router-note** в web/shared design-system-discovery: конференц/MCU/VCS-поверхность (`iva-connect`, calls/mcu/conference libs, импорт `from 'iva-core'`) → скилл iva-core; основной UI iva-one → `@iva/design-system`.

**Server-side / RE / DS-команда (НЕ наш PR-C, ОТЛОЖИТЬ):**
- серверная ДС `iva-core` (attach к workspace, N:M — сервер уже умеет, ADR-0026);
- **seed-метаданные** `design-systems/iva-core/design-system.yaml` (аналог iva-web — формально файл в этом репо, но это seed для серверной БД + требует реальных токенов VCSWEB Figma / генерации словаря — не template-профиль);
- **словарь code-bindings** для iva-core (каталог iva-datepicker/iva-grid/iva-seeker/charts…) — extraction в RE-репо.

Серверная часть **не блокирует** текущий пилот iva-one.

---

## 5. Осознанно ОТЛОЖИТЬ

- **Серверная ДС iva-core + словарь code-bindings** — не template-профиль (server/RE/DS-команда). Наш PR-C = только тонкий скилл + router-note.
- **Массовый фикс дрейфа design-system-discovery** в rn-brownfield / iva-brownfield-mail (всё ещё хардкодят iva-web) — не ось-1-web-цель PR-C; отдельные заходы.
- **Расширение _mirrors.yaml** новой парой для web-копий design-system-discovery (CI-лок ui-base↔web-brownfield) — правка процесса, не gap PR-C; решение лида/ГД отдельно.
- **Дрейф `framework_hint: react` в `design-systems/iva-web/design-system.yaml`** при Angular — это seed серверной ДС (server-side); фиксить осознанно с DS-командой, не в template-скилле. Скилл лишь должен корректно ЧИТАТЬ то, что сервер отдаёт.
- **Ось-2** (два репо source+target) — вне PR-C.

## Пути (якоря для плана)
- G7 web-цель: `~/tacticum/tacticum-dev/templates/iva-web-brownfield/ingredients/skills/design-system-discovery/SKILL.md`
- G7 SoT-канон: `~/tacticum/tacticum-dev/templates/tacticum-ui-base/ingredients/skills/design-system-discovery/SKILL.md`
- mirror: `~/tacticum/tacticum-dev/templates/_mirrors.yaml` + `scripts/check_mirror_sync.py`
- DS seed дрейф: `~/tacticum/tacticum-dev/design-systems/iva-web/design-system.yaml`
- G8 образец: `~/tacticum/tacticum-dev/docs/user_manuals/iva-kmp-figma-mapping-quickstart.md`
- PR-B файлы (overlap): `templates/iva-web-brownfield/{CHANGELOG.md,manifest.yaml}`
- iva-core упоминания (не наши): `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-*`