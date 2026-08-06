---
title: impl-qa-kit-subagents
type: report
permalink: tacticum/00-board/impl-qa-kit-subagents
status: draft
role: implementer
lane: lead-qa
branch: feat/qa-kit-subagents
worktree: ~/tacticum/tacticum-dev-qa-kit
tags:
- impl
- qa
- kit
- subagents
- sanitization
- lead-qa
archived-at: 2026-07-31 04:54
---

# Реализация кол ГД #1+#2 — субагенты kit + санитизация секретов

**Когда:** 2026-07-23 · **Кто:** implementer → lead-qa
**Основание:** [[explore-qa-kit-transfer-manifest]] + [[recon-kit-full-qa-dorabotka]]
**Лейн:** `templates/iva-qa-autotest-base/` · ветка `feat/qa-kit-subagents` (НЕ смёржено, НЕ запушено).

---

## Итог: реализовано, статически провалидировано

Основной объём (#1 субагенты комплектом + #2 санитизация) был выполнен в ветке 6 коммитами;
я провёл **сплошной аудит соответствия брифу**, закрыл пробел консистентности доков и прогнал
статические проверки. Всё сходится с чертежом-манифестом.

### #1 — субагенты комплектом (разблокировка write/batch/fix)

- **3 agent_spec** `ingredients/agents/{codebase-analyst,dom-explorer,code-writer}.md` — bare-имена,
  target `.claude/agents/<bare>.md`, **все `metadata.model: opus`** (подтверждено в manifest и валидатором).
  tools: analyst `[Read,Glob,Grep]`, dom-explorer `[Bash,Read,Write,Glob,Grep]`,
  code-writer `[Read,Edit,Write,Glob,Grep,Bash]`.
- **Дом стека** `craft-stack` (skill_spec → `.claude/skills/craft-stack/`): recon, rules/{page-objects,tests},
  test-templates, failure-taxonomy, fix-playbooks, allure-raw-parser, pytest-runner, batch-conventions +
  `rules/invariants.md` + `shared/{ledger-and-deviations,jira-candidates,coverage-ledger.template}`.
  Ссылки `$CRAFT`/`${CLAUDE_PLUGIN_ROOT}`/`$TMS` **разрешены** в конкретные пути лейна
  (`.claude/skills/craft-stack/...`, `tools/testops`) — неразрешённых в контенте нет (остались только
  в поясняющих комментариях manifest/craft-stack SKILL).
- **Тела оркестраторов** write/batch/fix заменены телами kit целиком (сохранён `ingredient_id`).
  Три несовместимости §2 закрыты: (а) `analysis.md` пишет ОРКЕСТРАТОР, субагент возвращает текстом;
  (б) раскладка `.tasks/work/tc-{id}/` (проверено grep'ом); (в) Task-вызовы — bare-имена.
- **[craft]-параметры** — стаб `ingredients/craft-stack/craft-answers.example.toml` со стек-нейтральными
  плейсхолдерами (`stack="pytest-playwright-canvas"` фикс.; остальное = `<placeholder: ...>`).

### #2 — санитизация секретов

- **`allure.iva.ru` вычищен** из всех точек write/batch/fix (grep по контенту лейна — 0 хардкодов;
  остались лишь мета-упоминания в README/CHANGELOG как описание «что вычищено»).
- **Нарратив «токен зашит, не секрет» переписан** на env-модель kit: `TESTOPS_ENDPOINT`/`TESTOPS_TOKEN`/
  `TESTOPS_PROJECT_ID` (env первичен → gitignored `secrets.yaml` секция `testops:` → явная ошибка),
  apitoken → JWT (`POST {endpoint}/api/uaa/oauth/token`, `grant_type=apitoken`) → `Bearer`.
  Эталон — `tms/scripts/testops/client.py` kit; сам `.py` в шаблон НЕ тащим (ответственность one-web).
- **`secrets.yaml.example`** создан (instruction_pack, `create_if_missing`) — секция `testops:` с
  `endpoint/token/project_id`. **`secrets.yaml` → `.gitignore`** (repo_config, `append_lines`, marker_id).

### Что доделал я (пробел консистентности)

Предыдущая сессия обновила шапку manifest, но **README.md и CHANGELOG.md остались в состоянии ДО
переноса** и прямо противоречили поставленному: «9 ingredients — 9 skill_spec», «3 субагента ОТСУТСТВУЮТ»,
«3 скилла заблокированы», «санитизация частичная — на follow-up». Привёл оба в соответствие
(коммит `905112e`): 15 ingredients, субагенты реализованы (opus), env-модель секретов, дом стека
craft-stack, флаг субагентов помечен снятым.

---

## Проверки (static)

- **manifest.yaml VALID** против `manifest.v2.schema.json`; **все 15 ingredients VALID** против
  `ingredient.v1.schema.json` (agent_spec: `model`+`description`; skill_spec: `description_trigger`;
  instruction_pack/repo_config: `target_file`+`merge_strategy` из enum). Валидатор Draft7 (как в
  `apps/backend/tests/catalog/test_manifest_schemas.py`), прогнан на venv 3.12.
- grep: `allure.iva.ru` = 0 хардкодов; `$CRAFT`/`$TMS` в контенте = 0; `craft:`-префиксы в Task = 0
  (оставшиеся `craft:` — ссылки на скилл `craft:update`, не субагенты); `.py` в лейне = 0; мусора нет.
- `.tasks/work/tc-{id}/` подтверждена в write/batch/fix; env-нарратив подтверждён в testops-api.md.
- **Живой прогон 3 скиллов с субагентами (opus) НЕ выполнялся** — осмыслен только в консьюмер-репо one-web.

---

## Приёмка для verifier (живая работа — в one-web)

Лейн исполним только в one-web (manifest привязан к autocore/venv/tools.testops/glab/CI). После установки:
1. `/agents` показывает три субагента с **model=opus**.
2. **write-autotest** (1 TC, `TESTOPS_TOKEN` задан): PASS = `.tasks/work/tc-{id}/{input,analysis}.md`
   (analysis пишет ОРКЕСТРАТОР), при пробелах `locators.md` от dom-explorer, тест-функция, `--collect-only`
   зелёный; в трейсе Task — спавн трёх на opus.
3. **batch-autotest**: URL фильтра TestOps → `.tasks/work/wave-<кампания>/PROGRESS.md` + ≥1 TC-дом.
4. **fix-failed-test**: имя упавшего теста/ланч → `.tasks/work/batch-{ts}/fix-{test}-analysis.md`,
   при DOM_CHANGED — `fix-{test}-locators.md`, правка code-writer, верификационный прогон зелёный.
5. **Секреты**: `TestOpsClient()` без аргументов резолвит из env/secrets.yaml; при снятом env и без
   secrets.yaml — `TestOpsError` с инструкцией (не молчаливый фейл).

---

## Флаги / развилки к lead-qa (scope молча НЕ расширял)

1. **Остаточный хардкод окружения one-web вне scope переноса kit** — НЕ трогал:
   - `retro/SKILL.md:25` — абсолютный путь `-Users-oc4kxb-Projects-AT-one-web/memory` (retro не входит
     в 3 разблокируемых скилла и не в список точек санитизации #2). Секретов нет, но путь стек-специфичен.
   - стенды/`autocore`-пути в per-skill references — в README помечены как follow-up.
   Нужно вычистить и это — верните задачу, сделаю отдельным коммитом.
2. **`craft-answers.example.toml` внутри `ingredients/craft-stack/`**, разворачивается в
   `.claude/skills/craft-stack/` как reference-форма (потребитель копирует `[craft]` в `.kit/answers.toml`).
   Отдельным ingredient не объявлял — следовал схеме «reference рядом с SKILL». Если канон сборки требует
   отдельного носителя `.kit/answers.toml` — это развилка по механизму сборки (бриф велит СТОП): подтвердите.
3. **client.py в шаблон не тащим** (ответственность one-web); `secrets.yaml.example` авторил по форме
   `resolve_settings()` kit (в снапшоте kit его не было).

---

## Коммиты (ветка `feat/qa-kit-subagents`, лейн)

- `d51ed73` субагенты craft (opus) + общий дом стека craft-stack
- `01d94bf` тела оркестраторов write/batch/fix целиком из kit
- `5c7fe66` санитизация секретов на env-модель + secrets.yaml.example
- `5fba328` manifest — 3 agent_spec (opus) + craft-stack + config-носители
- `a2c2753` batch — убрать локально-выглядящую ссылку на tms-референс
- `905112e` README/CHANGELOG — привести к состоянию после переноса kit ← мой

`git diff --stat main..HEAD` (лейн): 62 файла, +8288. Дерево чистое, мусора/py нет.

## Связано
[[explore-qa-kit-transfer-manifest]] · [[recon-kit-full-qa-dorabotka]]