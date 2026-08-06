---
title: impl-ds-web-to-kmp-variant1
type: report
permalink: tacticum/00-board/impl-ds-web-to-kmp-variant1-1
status: draft
archived-at: 2026-07-31 17:27
---

# impl-ds-web-to-kmp-variant1

Отчёт implementer для lead-ds (ТЗ#1 Сц.4). Навык `web-to-kmp-screen-port` оформлен под ВАРИАНТ 1 доставки (репо-нативно в KMP-репо). Worktree `/Users/bubblemac/tacticum/tacticum-dev-ds-web-to-kmp`, ветка `feat/ds-web-to-kmp`, коммит `62bc27c` поверх `0e52b6a`. НЕ мержено/пушено.

## Итог
Все 4 пункта задачи выполнены. Комбо `install_scope: repo` + `target_path_template` для `kind: skill_spec` — **валидно по схеме** (эскалация НЕ потребовалась). Валидаторы зелёные.

## Изменённые файлы (git diff --stat)
```
 templates/iva-kmp-development-base/CHANGELOG.md                         | 19 +++++++++++++++++++
 templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md |  2 +-
 templates/iva-kmp-development-base/manifest.yaml                        |  8 ++++----
 3 files changed, 24 insertions(+), 5 deletions(-)
```

## 1. Проверка комбо `install_scope: repo` + `target_path_template` (kind=skill_spec)
Развилки/эскалации нет — комбинация валидна на всех уровнях:
- **`ingredient.v1.schema.json`** валидирует ТОЛЬКО `kind` + per-kind `metadata` (для skill_spec — обязателен `description_trigger`). Поля `install_scope`/`target_path_template`/`codex_target_path`/`body_path` схемой не ограничиваются (нет `additionalProperties:false` на уровне ingredient). Никакого запрета repo+template.
- **`manifest.v2.schema.json`** требует лишь `schema_version` и не описывает ingredients детально.
- **Доменная модель `IngredientBase`** (`apps/backend/.../ingredients/base.py`): `install_scope: InstallScope = "repo"` — repo это вообще ДЕФОЛТ и валидное значение; `target_path_template: str | None`. Оба поля сосуществуют без ограничений. `SkillSpec` наследует базу без сужения.
- **Seed-пайплайн** (`seed_profile.py` `IngredientPayload` / `canonical_payload_hash`) знает `install_scope` + `target_path_template` и включает их в контент-хэш. `codex_target_path`/`body_path` — авторские manifest-поля, в хэш не входят.

Прецедент repo-scope + skill_spec + target_path_template в каталоге отсутствует (все существующие skill_spec — `install_scope: user` с `.claude/skills/...`; repo-scope встречается у command_spec/mcp_server_spec/instruction_pack, но без target_path_template). Поскольку схема/модель комбо разрешают явно — изобретать ничего не пришлось.

## 2. Как решён codex_target_path — ВАРИАНТ (a): тот же репо-нативный путь
`codex_target_path: 'AI common/skills/{ingredient_id}/SKILL.md'` (= target_path_template).

Обоснование:
- Доктрина Варианта 1: KMP-репо потребляет навыки через opencode (`opencode.json`, `ai-skills` path=`AI common/skills`); и claude-code, и codex-агенты читают ОДИН репо-нативный файл. Единый путь для обоих CLI дословно выражает это «один файл, оба агента» — консистентно с `install_scope: repo` и `supports: [claude-code, codex]`.
- Технически поле для skill безопасно: канонический `CodexRenderer._render_skill` (`infrastructure/renderers/codex.py`) жёстко пишет в `.agents/skills/{id}/SKILL.md` и `codex_target_path` для скиллов НЕ читает (это платформенный profile-install рендер, а не репо-нативная доставка); в контент-хэш поле не входит. Т.е. значение документирующее и не ломает ни хэш, ни рендер.
- Альтернатива (b) «убрать codex_target_path» тоже валидна по схеме, но менее выразительна: оставляла бы codex в supports без явного пути. Выбрал (a) как самодокументирующее.

## 3. Финальная запись ingredient (целиком)
```yaml
- ingredient_id: web-to-kmp-screen-port
  kind: skill_spec
  tier: full
  supports:
  - claude-code
  - codex
  install_scope: repo
  target_path_template: 'AI common/skills/{ingredient_id}/SKILL.md'
  codex_target_path: 'AI common/skills/{ingredient_id}/SKILL.md'
  body_path: ingredients/skills/web-to-kmp-screen-port/SKILL.md
  metadata:
    description_trigger: Port an iva-one Angular screen to KMP — read source (signalStore,
      view-state, DS names, Transloco, REST), rewrite into Decompose + Compose on Iva*, verify
      three-way parity (web-sample + tokens + Compose-screenshot); rewrite-port not move-port;
      orchestrates in-repo decompose-/mvi-state-machine-/compose-ui-patterns-su-ivcs-messenger-kmp,
      clean-architecture-su-ivcs-messenger-kmp, compose-multiplatform-ui, design-system-discovery,
      design-token-usage, ui-mockup-match, kmp-ui-testing, iva-web-ecosystem, and contrasts
      android-to-kmp-porting-su-ivcs-messenger-kmp (move-port vs rewrite-port)
```
`body_path` сохранён (источник в каталоге). Роль `iva-role-kmp` тянет через `depends_on` — её манифест и зеркало `iva-kmp-brownfield` НЕ трогались (навык owner-only, в `_mirrors.yaml`/brownfield отсутствует — подтверждено grep).

## 4. SKILL.md — унификация формулировок (замечание гейта)
Единственная содержательная правка — строка TODO «Figma bridge in DS skills»: `«those tacticum-ui-base skills»` → `«those in-repo DS skills (AI common/skills/)»`. Убран рассинхрон дома: в контексте доставки в KMP-репо `design-system-discovery`/`design-token-usage` — in-repo навыки. §8 (таблица reused skills) и §9 (source-of-truth) уже фреймили все референс-навыки как in-repo (`iva-m/android/kmp`, `AI common/skills/`) — теперь TODO с ними согласован. Доктрина (rewrite-port, трёхсторонний паритет, уровни state-holder) не менялась.

## 5. Version + CHANGELOG
`iva-kmp-development-base` `version: 0.3.0 → 0.4.0`; добавлена секция `## [0.4.0] — 2026-07-24` (перевод навыка на repo-native доставку, Вариант 1, + унификация формулировок SKILL.md).

## Вывод валидаторов (venv репо `apps/backend/.venv/bin/python`)
- `manifest.v2.schema.json`: 0 ошибок по манифесту iva-kmp-development-base.
- `ingredient.v1.schema.json`: 0 ошибок по всем ingredients (включая целевой).
- `scripts/check_profile_version_discipline.py` (static): `OK — 46 profile(s) clean` (exit 0).
- `scripts/check_profile_version_discipline.py --diff-against 0e52b6a`: `OK — 46 profile(s) clean` (exit 0) — bump зафиксирован в диффе, CHANGELOG-heading на месте.
- `scripts/check_mirror_sync.py`: `OK — 62 зеркальных ингредиентов в 6 парах синхронны` (exit 0).

## Самопроверка
- Манифест консистентен: install_scope/target_path_template/codex_target_path согласованы под repo-native, body_path сохранён, description_trigger нетронут; version=0.4.0 совпадает с CHANGELOG.
- SKILL.md читается, формулировки унифицированы (in-repo рамка сквозная §8/§9/TODO).
- Схемы + version-discipline + mirror-sync зелёные.

## Что НЕ делалось (границы задачи)
- Не трогались `iva-role-kmp/manifest.yaml` и зеркало `iva-kmp-brownfield` (роль тянет через depends_on).
- Провижн в KMP-репо (`AGENTS.md` ручной роутер — владелец Легин, `opencode.json`) — этап доставки, вне этого пакета.
- Не мержено/не пушено (autonomy off) — ждёт ревью лида/решения президента.

## Вопрос лиду (не блокирующий)
codex_target_path решён вариантом (a) — единый репо-нативный путь. Если лид предпочитает вариант (b) «убрать codex_target_path для repo-scope skill» — правка тривиальна (1 строка), скажи. Оба валидны по схеме.