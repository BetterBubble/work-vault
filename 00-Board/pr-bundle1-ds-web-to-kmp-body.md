---
title: 'PR-тело: бандл #1 web-to-kmp capability (feat/ds-web-to-kmp) — ДРАФТ'
type: note
status: draft
permalink: tacticum/00-board/pr-bundle1-ds-web-to-kmp-body
tags:
- board
- design-system
- lead-ds
- tz1
- pr
---

# PR — бандл #1: capability переноса экранов Angular → KMP Compose (Сц.4)

**Ветка:** `feat/ds-web-to-kmp` → `main` (репо `TacticumApps/tacticum-dev`). **Драфт для президента через ГД — НЕ пушено, ждёт апрув.** (Финализируется после батареи: git/controller + fidelity + critic.)

## Заголовок PR
`feat(iva-kmp): capability web-to-kmp-screen-port + source-reference (Сц.4 перенос экранов Angular→Compose)`

## Описание

### Что сделано
Добавлена capability для переноса экранов iva-one (Angular) → su.ivcs.messenger (KMP Compose) по ТЗ figma-ds Сценарий 4. Два новых навыка в пакете `iva-kmp-development-base`, доставляются репо-нативно в `AI common/skills/` целевого KMP-репо (`install_scope: repo` + `target_path_template`):

1. **`web-to-kmp-screen-port`** — навык-оркестратор переноса. Содержит:
   - процедуру чтения источника (Angular-образец: структура/store/view-state/DS-имена/Transloco/REST — полный набор файлов .ts+.html+data-access+route);
   - таблицу соответствия Angular→Compose (14 строк);
   - маппинг состояния signalStore → Decompose-компонент + StateFlow (Level-1) / MVIKotlin (Level-3, по in-repo навыку);
   - гардрейлы приземления (`feature/<name>/impl/commonMain` + `Iva*`; не `ucim`/`presentation.*`);
   - «что не переносить» (web-only/WebRTC/Electron);
   - трёхсторонний паритет (веб-образец + токены + Compose-скриншот/Roborazzi);
   - принцип «rewrite-port, не move-port» (контраст с in-repo `android-to-kmp-porting`);
   - оркестрирует 11 реальных in-repo навыков (decompose/mvi-state-machine/compose-ui-patterns/design-system-discovery/…), не дублирует.

2. **`web-to-kmp-source-reference`** — навык доступа к двум деревьям (ось-2 п.3): исходное (iva-one) READ-ONLY + целевое (su.ivcs.messenger) write; жёсткое «в источник не писать»; ДС письма = целевой репо.

Плюс словарь `Iva*`(KMP)↔веб-мастер-компонент с реальными figma_key (данные, отдельно от PR — на доске `phase2-provisional-iva-web-dictionary`, resolved по 32 ключам из репо-каталога code-bindings).

### Что тестировалось
- **Пре-PR батарея (push-флоу):** git/controller PASS (3-точечный diff, скоуп только пакет, 0 секретов/AI-подписей, валидаторы зелёные) · fidelity к ТЗ Сц.4 PASS (0 сверх-ТЗ) · critic (сильный, 3 обязательные правки внесены + nice-to-have) · финальный гейт critic-правок.
- Controller-гейты по каждому юниту по ходу — все PASS.
- Валидаторы репо зелёные: `ingredient.v1` + `manifest.v2` схемы (0 ошибок), `check_mirror_sync.py` (62/6 синхронны), version-discipline (46 clean), pytest `test_manifest_schemas` (38 passed).
- **Сухой пилот** навыка на реальном экране `ContactDetailScreen` (su.ivcs.messenger): навык валидирован статически (state hoisting ✅, незамапленное→СТОП ✅, Iva* не выдуман — пробелы честны). Вскрыл реальный дефект целевого кода (raw Spacer вместо IvaSpacer) — зафиксирован как находка для реального порта.
- **Рантайм-верификация** (компиляция + runComposeUiTest + Roborazzi + VLM) — окружение teststand не поддерживает (нет Android SDK, не билд-стенд); отложена до KMP/Android-среды команды. Статическая приёмка (сухой пилот) на данном этапе достаточна для валидации навыка.
- figma_key словаря сверены построчно с `design-systems/iva-web/tokens.json` (jq) — не выдуманы, баланс 32+17=49.

### Что осознанно отложено (не в этом PR)
- Рантайм-верификация пилота (компиляция + `runComposeUiTest` + Roborazzi + VLM) — требует write+build-окружения (решение президента по модели прогона).
- Полный инвентарь `Iva*` с сигнатурами (закрыть пробелы словаря обеих сторон) + промоция словаря в серверные code-bindings (server-write).
- Ось-2 п.1/2/4 (`start task` source-arg + workflow read-only-source + гейт §3.0) — лейн lead-modes (пара `iva-web-brownfield`+`iva-kmp-brownfield`).
- Ось-1 (несколько ДС: iva-core/VCSWEB) — отдельный заход.
- Подтверждение Figma-фрейма экрана + `IvaBottomSheet→Dialog` семантический матч — у владельца ДС.

### Файлы (заполнится из git/controller-гейта — 3-точечный diff от merge-base)
- `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md` (new)
- `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-source-reference/SKILL.md` (new)
- `templates/iva-kmp-development-base/manifest.yaml` (2 skill_spec, version 0.7.0)
- `templates/iva-kmp-development-base/CHANGELOG.md` (`[0.4.0]`…`[0.7.0]`)

*Финальный коммит ветки: `39ae642` (после critic-правок). Итог: 4 файла от merge-base.*

### Ссылки
ТЗ Сц.4 (`figma-ds-scenario-4-kmp-port.md`), карточка [[План ТЗ-1 Дизайн-процесс Figma↔код — Сц.4 перенос форм one→kmp (lead-ds)]].

---
*Примечание: коммиты без AI-подписей, дом навыка = Вариант 1 (решение президента). Мерж — осознанное действие президента.*
