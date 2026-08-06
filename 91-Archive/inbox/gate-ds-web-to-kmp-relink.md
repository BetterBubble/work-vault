---
title: gate-ds-web-to-kmp-relink
type: note
permalink: tacticum/00-board/gate-ds-web-to-kmp-relink-1
tags:
- board
- design-system
- lead-ds
- tz1
- controller
- gate
archived-at: 2026-07-31 17:27
---

# Gate: пере-связка web-to-kmp-screen-port на реальные in-repo навыки (ТЗ#1 Сц.4)

**Роль:** controller-гейт для lead-ds. **Read-only, вердикт.** **Worktree:** `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp`, ветка `feat/ds-web-to-kmp`, коммит `0e52b6a` поверх `7882902`. Autonomy off — мерж не сейчас.

## ВЕРДИКТ: PASS

Пере-связка достоверна, скоуп чист, конформность подтверждена собственным прогоном валидаторов. Замечания только процедурно-косметические (ниже), блокеров нет.

---

## 1. Гит / скоуп — PASS
- `git status` — clean. Ветка `feat/ds-web-to-kmp` (не main).
- Коммит `0e52b6a` (name-status) трогает ровно 3 файла, все в `iva-kmp-development-base`: `CHANGELOG.md`, `manifest.yaml`, `ingredients/skills/web-to-kmp-screen-port/SKILL.md`. Вся ветка vs `origin/main` — те же 3 файла (+282/−1).
- Запретные зоны НЕ затронуты: `iva-role-kmp`, `iva-kmp-brownfield`, `ROLE_LANES`/тест-матрица, `iva-web-brownfield`, `iva-analysis-base`, любые `_mirrors.yaml`. Навык owner-only, в зеркалах не значится — подтверждено.
- Автор коммита — реальный `Александр Шульга <aleksandr-shulga-0507@yandex.ru>`, НЕ AI. AI-подписей (Claude/Co-Authored/Generated with/claude.ai) в теле коммита нет. Секретов/мусора нет.

## 2. Достоверность ссылок — PASS (главный пункт)
- Раздел §8 SKILL.md — таблица из ровно 11 навыков, все сверены с каталогом `recon-ds-inrepo-skills-catalog` по имени: `android-to-kmp-porting-su-ivcs-messenger-kmp`, `decompose-…`, `mvi-state-machine-…`, `compose-ui-patterns-…`, `compose-multiplatform-ui`, `clean-architecture-…`, `design-system-discovery`, `design-token-usage`, `ui-mockup-match`, `kmp-ui-testing`, `iva-web-ecosystem`. Совпадение 11/11.
- Все 5 токенов `*-su-ivcs-messenger-kmp` в файле — реальные (android-to-kmp-porting / clean-architecture / compose-ui-patterns / decompose / mvi-state-machine). Выдуманных имён нет.
- Запрещённого `angular-ds-component-usage` — НЕТ нигде (grep пусто). Мёртвых ссылок нет.
- frontmatter `description` (стр. 9–12) перечисляет тот же набор 11 + контраст android-to-kmp-porting — консистентно с телом.

## 3. Достоверность доктрины — PASS
- (а) **move vs rewrite** — §0 (стр. 34–41): android-to-kmp-porting подан как MOVE (релокация Kotlin androidMain→commonMain, expect/actual, каталог Android→KMP свапов); наш кейс — REWRITE (смена фреймворка Angular→Compose, «nothing carries over verbatim»). Контраст верный, expect/actual/@Parcelize/Dagger явно исключены (стр. 53–54).
- (б) **ui-mockup-match** — §7.2 (стр. 156–160): «playwright + DOM-diff… web runtime… NOT a Compose comparator». Compose-паритет — §7.4 через `kmp-ui-testing`/Roborazzi + VLM. Разведено корректно, как в каталоге.
- (в) **состояние** — §3 (стр. 110–119): Level-1 `MutableStateFlow + onXxx()` по умолчанию / Level-3 MVIKotlin `Store` (State/Intent/Msg/Label, Harel sealed-mode, one-shot через Label) по `mvi-state-machine`. Уровень берётся из целевого кода, не из веб-источника (стр. 121–122). Верно.
- (г) **Iva*-истина** — §9 (стр. 194–202): shared `iva-m/android/kmp` commonMain, ~49 composables; старый пилот ~41 явно помечен stale. Верно.

## 4. Конформность — PASS (прогнал сам, не по отчёту)
- frontmatter: `name: web-to-kmp-screen-port` + `description` присутствуют.
- Версия: `manifest.yaml` version `0.3.0` (стр. 8) == `CHANGELOG.md` секция `## [0.3.0] — 2026-07-24`. Прежняя `0.2.0` тоже в CHANGELOG.
- `check_profile_version_discipline.py` (static): `OK — 46 profile(s) clean`.
- `check_profile_version_discipline.py --diff-against HEAD~1`: `OK — 46 profile(s) clean`.
- `check_mirror_sync.py`: `OK — 62 зеркальных ингредиентов в 6 парах синхронны`.
- `ingredient.v1.schema.json` (Draft7, все 15 ингредиентов манифеста): 0 ошибок. `manifest.v2.schema.json`: 0 ошибок.
- `pytest tests/catalog/test_manifest_schemas.py`: 38 passed.
- Валидаторы гонялись через `apps/backend/.venv/bin/python` (в системном python3 нет pyyaml/jsonschema).

## 5. Память — PASS
- Отчёт implementer'а `00-Board/impl-ds-web-to-kmp-relink` — на доске, прочитан.
- Каталог `00-Board/recon-ds-inrepo-skills-catalog` — на доске, использован как истина.

---

## Замечания (не блокеры)
1. **Расхождение путей в отчёте implementerّа (косметика).** Отчёт указывает `pytest tests/catalog/test_manifest_schemas.py`, фактический путь — `apps/backend/tests/catalog/test_manifest_schemas.py`. Тесты проходят (38). diff-stat в отчёте усечён (`.../skills/…`), реальный путь SKILL.md — `.../ingredients/skills/web-to-kmp-screen-port/SKILL.md`. На содержание не влияет.
2. **Смешение «домов» одного имени (не дефект).** §8 объявляет все 11 «real skills in su.ivcs.messenger KMP repo (AI common/skills/)», а TODO (стр. 222) зовёт `design-system-discovery`/`design-token-usage` «tacticum-ui-base skills». Оба верны в своей рамке: имена существуют И в adp `AI common/skills` (по каталогу), И как tacticum-шаблоны в `tacticum-ui-base` (проверено — папки есть). Противоречия нет, но при финализации дома навыка стоит унифицировать формулировку.
3. **Граница проверки.** Наличие 6 «общих» навыков (design-system-discovery/design-token-usage/ui-mockup-match/compose-multiplatform-ui/kmp-ui-testing/iva-web-ecosystem) именно в adp-репозитории я подтвердить не могу (adp read-only remote вне worktree) — опираюсь на каталог `recon-…` как назначенную истину. compose-multiplatform-ui/kmp-ui-testing локально лежат в самом `iva-kmp-development-base`; DS-тройка — в `tacticum-ui-base`; `iva-web-ecosystem` в tacticum-шаблонах отсутствует (ожидаемо — он adp-side).

## Контекст (учтено, не дефекты)
- Скелет + TODO до пилот-репо — норма.
- Дом доставки навыка эскалирован президенту; навык пока в `iva-kmp-development-base` — ОК, это контентная правка, не переезд.
- Autonomy off, мерж/пуш не сейчас.

## Связано
`00-Board/impl-ds-web-to-kmp-relink` · `00-Board/recon-ds-inrepo-skills-catalog` · `00-Board/gate-ds-web-to-kmp-phase1`