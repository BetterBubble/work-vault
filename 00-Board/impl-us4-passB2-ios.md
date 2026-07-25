---
status: draft
role: implementer
topic: US#4 Проход B2 — pin/tests К-2/К-4 для iva-ios-brownfield
lead: lead-fr
spec: tacticum/00-board/spec-us4-passB2-pintests
branch: feat/us4-passB2-ios
worktree: /Users/bubblemac/tacticum-wt/us4-passB2-ios
date: 2026-07-24
permalink: tacticum/00-board/impl-us4-passB2-ios
---

# US#4 Проход B2 — iva-ios-brownfield (implementer report)

Готово. Ветка `feat/us4-passB2-ios` от свежего `origin/main` (`c6be10a`), 1 коммит `2e15d40`.
НЕ push, НЕ merge. Правил ТОЛЬКО own pin/tests профиля iva-ios-brownfield. Наследуемые из
tacticum-dev-base (brd / start-task) и другие профили НЕ тронуты.

## Предпроверка профиля
Подтверждено: iva-ios-brownfield несёт **own** `pin-authoring` и **own** `tests-authoring`
(оба — override базы, задокументированы в README/CHANGELOG). Базовый pin/tests серий CT/DM/EV
ещё не несёт (проверено grep) → B2 — первый проход К-2/К-4 в этих own-скиллах, дизайн под
iOS-стек с нуля (не mirror). Развилка по К-3 ниже.

## Что сделано (под iOS-стек, backward-safe)

### pin-authoring (own) — К-2 + К-4
- **К-2** — новая секция «Project series — CT-n / DM-n / EV-n (v2 FR only)» + пункт 12 в
  Document structure. Реализует каждый член серии по стабильному ID на iOS-стек:
  - `CT-n` (контракт §1.6) → integration point: SDK-bridge response shape
    (`Messenger/ModulesBridges/*` → ivaone-sdk/calls/imailframework/calendarsdk) или
    Centrifugo/IvaLink event payload; реализация = wiring + **чистый маппинг** (SDK/`CentrifugeEvent`
    → app-model), verify через `kb_verify_api_exists`. Fragile-зоны не мокаются (контракт = version
    tag, не код; `sdk-bridges-fragile-zone`, `ivalink-realtime`) — live path = ручной smoke.
  - `DM-n` (модель/состояния §1.7) → `Core/Models` struct + `@Observable @MainActor` + Actionable
    (`swift-observation-state`), DI через Injector, без singleton.
  - `EV-n` (событие §1.8) → Centrifugo/IvaLink event adapter/decoder + идемпотентность/порядок
    (`ivalink-realtime`), чистая логика, live — smoke.
  - **Статус каждого** CT/DM/EV: реализован / расхождение / blocked. Ссылка по стабильному ID.
  - **К-3** уважён: раздел планируется «реализован» только при утверждённом `D-n` (гейт
    start-task), иначе честный **blocked/pending D-n** — не имитирует.
- **К-4** — расширил существующую reconciliation (API Verification Gate) на проектные разделы:
  таблица «Project-series divergence» (`Series ID | Design point | Repo/SDK reality | Verdict`);
  сверка контракта/модели с реальностью репо/SDK через `kb_verify_api_exists` и iOS-аналоги
  (topology / `kb_get_code_context` для SDK-bridge/IvaLink типов); конфликт = **критичное
  расхождение**. +1 правило, +1 анти-паттерн.

### tests-authoring (own) — К-2
- Новая секция «Project-series tests — CT-n / DM-n / EV-n (v2 FR only)» + пункт 7 в Document
  structure. Контрактные тесты на **извлечённой чистой логике** (маппер, реализующий `CT-n`)
  несут `Covers: CT-n` наряду с `Covers: UC-…`; `DM-n` → state/model тесты, `EV-n` →
  event-consistency тесты. Fragile-границы (SDK/socket/media) не мокаются — device/live smoke.
  Таблица «Project-series test status» (`Series ID | Covered by | Status`:
  covered / smoke-only / blocked). +1 правило (трассировка `Covers: CT-n`), +1 анти-паттерн.

### manifest + CHANGELOG
- `manifest.yaml`: version `0.1.2` → `0.1.3`.
- `CHANGELOG.md`: секция `[0.1.3] — 2026-07-24` с описанием К-2/К-4.

## Backward-safe (прод-safe)
Аддитивно. На v1-FR (нет §1.6/§1.7/§1.8, нет маркера `fr_skeleton: 2`) секции серий
пропускаются — pin/tests работают как раньше. Стек-специфику (Core/Features/app/extensions,
Observation-MVVM+Actionable, Injector DI, fragile-зоны, Swift Testing, never-mock) НЕ ломал —
надстроил К-2/К-4 сверху в терминах существующего iOS-словаря.

## Проверки
- **version-discipline**: `scripts/check_profile_version_discipline.py --diff-against origin/main`
  → `OK — 48 profile(s) clean.`
- **pytest целевой** (`PYTHONPATH=apps/backend … --noconftest`):
  - `test_iva_ios_brownfield_profile.py` → **13 passed**.
  - 5 профильных тестов (ios + mail + firebird + dev-base + ui-base) → **65 passed**.
  - Единственный red — `test_role_replacement_parity.py::…[iva-role-web<-iva-web-brownfield]`
    (`angular-ds-component-authoring/-usage` из #149 ds) — **предсуществующий, не наш**, ровно
    как предупреждал лид.
  - 34 collection ERROR в полном `tests/catalog/` — `ModuleNotFoundError: No module named
    'backend'` (нет установленного backend-пакета/БД в изолированном venv) — **окружение, не
    правки**; целевые профильные тесты (не требующие БД) прогнаны отдельно и зелёные.

## git diff --stat (origin/main..HEAD)
```
 templates/iva-ios-brownfield/CHANGELOG.md          | 22 +++++++++
 .../ingredients/skills/pin-authoring/SKILL.md      | 57 ++++++++++++++++++++++
 .../ingredients/skills/tests-authoring/SKILL.md    | 34 +++++++++++++
 templates/iva-ios-brownfield/manifest.yaml         |  2 +-
 4 files changed, 114 insertions(+), 1 deletion(-)
```
Коммит: `2e15d40`. `git status` clean.

## Развилки (durable)
- **D-n gate (К-3) в main ещё нет.** grep по `templates/tacticum-dev-base/` не нашёл `D-n` —
  start-task D-n gate заявлен как «Проход C-канон» (будущий). Я сослался на `D-n` форвардно:
  pin честно репортит `blocked/pending D-n`, не имитирует. Когда Проход C приземлит D-n в
  start-task, словарь совпадёт (ссылка по стабильному ID). Если лид хочет иную формулировку до
  прихода C — верните на правку.
- **Own vs base**: оба скилла — own override, правки локальны профилю; на mail/rn/firebird не
  влияют (disjoint-профили, сборка в один feat/us4-passB2 без конфликта — как в плане лида).

## Связано
[[spec-us4-passB2-pintests]]
</content>
</invoke>
