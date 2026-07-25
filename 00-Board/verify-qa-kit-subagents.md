---
title: verify-qa-kit-subagents
type: report
permalink: tacticum/00-board/verify-qa-kit-subagents
status: draft
role: verifier
lane: lead-qa
branch: feat/qa-kit-subagents
worktree: ~/tacticum/tacticum-dev-qa-kit
verifies: impl-qa-kit-subagents
baseline: main (fork feat/iva-write-base @ 8a7e854)
tags:
- verify
- qa
- kit
- subagents
- sanitization
- lead-qa
- acceptance
---

> ⚠️ **КОРРЕКЦИЯ (2026-07-24, lead-qa):** сценарий писался на ветке `feat/qa-kit-subagents`, где 3 субагента были `model: opus`. **Модель с тех пор исправлена → `gpt-5.4` (Codex, профиль 0)** — в смерженном профиле на `main` @ `20ff9b8` (`templates/iva-qa-autotest-base/manifest.yaml` стр. 206/218/230; CHANGELOG: «хардкод opus был ошибкой, снят»). **При живом прогоне ожидать `gpt-5.4`, НЕ `opus`** — везде ниже, где «opus», читать как «gpt-5.4». Комплект передачи QA-команде уже под gpt-5.4.

# Верификация кол ГД #1+#2 — субагенты kit + санитизация (ветка feat/qa-kit-subagents)

**Когда:** 2026-07-23 · **Кто:** verifier → lead-qa
**Метод:** read-only, по факту на диске + прогон валидатора схем на реальном манифесте. НЕ по самоотчёту.
**Основание:** [[explore-qa-kit-transfer-manifest]] · [[recon-kit-full-qa-dorabotka]] · отчёт [[impl-qa-kit-subagents]]
**Kit-эталон:** `/private/tmp/claude-501/-Users-bubblemac-tacticum/5bcc859c-bf80-431d-bfc0-adde0845932e/scratchpad/kit/kit-main`
**Лейн:** `~/tacticum/tacticum-dev-qa-kit/templates/iva-qa-autotest-base`

---

## Вердикт по пунктам (7/7 PASS, статика; живой прогон НЕ проведён — нет доступа)

| # | Пункт | Вердикт | Числа |
|---|---|---|---|
| 1 | Субагенты (3 agent_spec, bare, opus, ≈kit) | **PASS** (1 наблюдение) | 3 дефа, model:opus ×3 |
| 2 | Стек комплектом + $CRAFT/$TMS разрешены | **PASS** | 15 файлов; 0 операц. остатков |
| 3 | Оркестраторы (kit-тела, bare Task, .tasks/work) | **PASS** | 50 `.tasks/work/`; 0 craft:-субагентов |
| 4 | Санитизация (allure=0, токен→env, secrets) | **PASS** | allure.iva.ru=0 операц. |
| 5 | Валидность (manifest + ingredients) | **PASS** | manifest VALID, 15/15 VALID |
| 6 | Провенанс (6 коммитов, чистота) | **PASS/CLEAN** | 6 коммитов в ветке, 55 файлов |
| 7 | Остаточное (абс.пути/чужие юзеры) | **PASS** (флаг точен) | 1 флаг (retro:25), больше нет |

---

## #1 Субагенты — PASS (+1 системное наблюдение)

- Три `ingredients/agents/{codebase-analyst,dom-explorer,code-writer}.md` — на месте.
- `name:` во фронтматере body — **bare** (`codebase-analyst`/`dom-explorer`/`code-writer`). ✓
- **model:opus** — объявлен в `manifest.yaml` metadata КАЖДОГО agent_spec: строки **203, 215, 227** (`grep model` = 3 хита). tools+description там же (schema требует model+description — выполнено).
- Содержимое body vs kit `craft/agents/*.md`: `diff` = только ожидаемые правки переноса — (а) `$CRAFT/stacks/{stack}/…` → `.claude/skills/craft-stack/…`; (б) `craft:`-префиксы в ПРОЗЕ → bare; (в) `$CRAFT/references/product-api/research_user.py` → ссылка на `craft-answers.example.toml`. Тело канона — байт-в-байт. ✓
- **Рантайм-опус подтверждён по пайплайну:** `metadata.model`/`tools` НЕ во фронтматере body (там только name+description), НО `ClaudeCodeRenderer._render_agent` (claude_code.py:58-71) генерирует фронтматер `name/description/model/tools` из metadata и инъектит в `.claude/agents/<id>.md` при рендере. Golden-тест `test_claude_code_renderer.py` подтверждает (`solution-architect.expected.md` = единый фронтматер с `model: opus`). → при штатной установке opus реально попадёт в файл субагента.

**Наблюдение (системное, НЕ регрессия ветки):** body-файлы агентов сохраняют СВОЙ фронтматер `---name/description---`, а рендер ещё и препендит свой → в устанавливаемом файле было бы два фронтматер-блока. Body грузится сырым (`seed_community.py:66 read_text`, без стрипа). НО это **идентично** существующему принятому шаблону `iva-2-client-shell-dev` (его agent/skill body тоже несут свой фронтматер + manifest metadata). Golden-фикстура использует body БЕЗ фронтматера. → расхождение «фикстура без фронтматера vs реальные body с фронтматером» — общерепозиторный вопрос конвенции сборки, а не дефект этой ветки. Отдельным тикетом на lead-qa/backend, не блокер приёмки переноса.

## #2 Стек комплектом — PASS

- `ingredients/craft-stack/` — **15 файлов**: recon, rules/{page-objects,tests,invariants}, test-templates, failure-taxonomy, fix-playbooks, allure-raw-parser, pytest-runner, batch-conventions, shared/{ledger-and-deviations,jira-candidates,coverage-ledger.template}, SKILL.md, craft-answers.example.toml. Совпадает с §1b чертежа. ✓
- `$CRAFT`/`$TMS`/`${CLAUDE_PLUGIN_ROOT}` — **0 в операционном контенте**. Остатки (5 `$CRAFT`, 4 `CLAUDE_PLUGIN_ROOT`, 1 `$TMS`) — ВСЕ в пояснительной прозе: README/CHANGELOG, комментарии manifest, craft-stack/SKILL.md («раньше пути были $CRAFT…»). ✓

## #3 Оркестраторы — PASS

- Тела write/batch/fix заменены телами kit: `.tasks/work/tc-{id}/` — **50 хитов** в 3 скиллах; контракт §2-расх.1 закрыт (write-autotest шаг 3: «Верни контент analysis.md итоговым сообщением — файл запишет оркестратор» + оркестратор персистит в `.tasks/work/tc-{allure_id}/analysis.md`). ✓
- Task-вызовы субагентов — **bare** (codebase-analyst/dom-explorer/code-writer). `craft:` — 10 хитов, ВСЕ ссылки на внешний скилл `craft:update` (не портирован), **0 craft:-префиксов у субагентов**. ✓
- Минор (не блокер): tc-review артефакты — плоско `.tasks/tc-review/TC-{id}.md` (отдельная под-фича tc-review, не дом задачи); `retro/SKILL.md:43` — старый `.tasks/batch-*` (retro вне scope 3 скиллов).

## #4 Санитизация — PASS

- **allure.iva.ru = 0 хардкодов** в операционном контенте. `grep -rn` = 3 хита, ВСЕ в мета-прозе README/CHANGELOG («что вычищено»). ✓
- Нарратив токена переписан на env-модель: `testops-api.md` — «Секреты по env-модели, **не** зашиты в коде»; порядок env `TESTOPS_ENDPOINT/TOKEN/PROJECT_ID` → gitignored `secrets.yaml` (секция testops:) → явная ошибка; apitoken→JWT (`POST {endpoint}/api/uaa/oauth/token`, grant_type=apitoken). **0** старого «токен зашит/не секрет» в ingredients. ✓
- `secrets.yaml.example` — `ingredients/config/secrets.yaml.example`, секция `testops:` c `endpoint/token/project_id`, порядок резолва задокументирован (соответствует `resolve_settings()` kit). ✓
- `secrets.yaml` → `.gitignore`: repo_config `gitignore-secrets` (append_lines, marker_id `iva-qa-autotest-base:secrets`), body `gitignore-secrets.txt` содержит `secrets.yaml`. ✓ (в git-дереве шаблона .gitignore не тронут — верно, append в консьюмера при install.)

## #5 Валидность — PASS

Прогон Draft7Validator (как `apps/backend/tests/catalog/test_manifest_schemas.py`) на **venv 3.12.13 + jsonschema 4.26.0**, схемы `templates/_schema/`:

```
=== MANIFEST manifest.v2.schema.json ===       VALID
=== INGREDIENTS ingredient.v1.schema.json (n=15) ===  15/15 VALID
  10x skill_spec, 3x agent_spec, 1x instruction_pack, 1x repo_config
=== body_path existence ===                    all body_path present
SUMMARY: manifest VALID; ingredients 15/15 VALID
```

## #6 Провенанс — PASS/CLEAN

- Ветка `feat/qa-kit-subagents` ответвлена от `feat/iva-write-base` @ **8a7e854** (merge-base совпадает с HEAD базы).
- **6 транспортных коммитов** (`d51ed73 01d94bf 5c7fe66 5fba328 a2c2753 905112e`) — все авторства сегодня **2026-07-23 11:39–11:51**, `git branch --contains` = **только feat/qa-kit-subagents** → сделаны В ВЕТКЕ, в базе iva-write-base их НЕТ. Ответ на вопрос чертежа: «6 коммитов» — не из базы, а созданы в ветке. ✓
- 6 коммитов трогают **ТОЛЬКО лейн**: 0 файлов вне `iva-qa-autotest-base`; **55 файлов, +3869/−2288**.
- Дерево чистое: `git status` пуст; 0× `.DS_Store`/`__pycache__`/`.pyc`/`.serena`; **0 .py в лейне**.
- Сверка с самоотчётом: impl писал «62 файла, +8288» — это `main..HEAD` лейн-скоуп (включает создание исходных 9 скиллов базовыми коммитами). Честная дельта именно ПЕРЕНОСА (feat/iva-write-base..HEAD) = 55 файлов/+3869/−2288. Расхождение методологическое (что считать baseline), не подлог.
- Прим.: полный `main..HEAD` = 211 файлов (149 вне лейна) — это унаследованные 4 базовых scaffold-коммита (роли/techwriter/backend от 2026-07-21), не работа переноса.

## #7 Остаточное — PASS (флаг implementer'а точен и полон)

- `retro/SKILL.md:25` — `mem="$HOME/.claude/projects/-Users-oc4kxb-Projects-AT-one-web/memory"` — подтверждён (абс. путь + чужой юзер oc4kxb). Retro вне 3 разблокируемых скиллов и вне точек санитизации #2 — флаг корректен.
- По ВСЕМУ лейну больше **НЕТ** абсолютных путей (`/Users/`=0 кроме retro, `/home/`=0), чужих юзеров (oc4kxb только в retro:25). Флаг полон. ✓

---

## Живой прогон — НЕ ПРОВЕДЁН (нет доступа)

Схема/статика — проверено. **Живой прогон 3 скиллов с субагентами (opus) НЕ проводился**: нет чекаута one-web, нет TESTOPS-кредов, нет стенда. Лейн исполним только внутри репо one-web (manifest привязан к autocore/venv/tools.testops/glab/CI). Ниже — точный сценарий приёмки для того, у кого есть one-web + TestOps + стенд.

### Сценарий приёмки для one-web (доказать живую работу)

**Пред-условия (env):**
```bash
export TESTOPS_ENDPOINT="https://<стенд-testops>"
export TESTOPS_TOKEN="<apitoken>"            # или ./secrets.yaml секция testops:
export TESTOPS_PROJECT_ID="<numeric>"
# venv one-web активен; в PATH: pytest, playwright-cli, allurectl, glab, git
# субагенты установлены (роль iva-role-qa применена): .claude/agents/{codebase-analyst,dom-explorer,code-writer}.md
```

**Шаг 0 — установка субагентов на opus (обязательный гейт):**
- Команда: в Claude Code — `/agents`.
- **PASS:** три субагента видны, у КАЖДОГО `model: opus` (не дефолт сессии). Открыть `.claude/agents/codebase-analyst.md` — во фронтматере ровно один блок с `model: opus` и корректным `tools:`. FAIL, если фронтматера два/model отсутствует/пусто (см. системное наблюдение #1 — проверить именно на рендере one-web).

**Шаг 1 — write-autotest (1 TC):**
- Дать скиллу URL тест-кейса `${TESTOPS_ENDPOINT}/project/<pid>/test-cases/<id>`, попросить «напиши автотест».
- **PASS:** создан `.tasks/work/tc-{id}/input.md`; `.tasks/work/tc-{id}/analysis.md` записан ОРКЕСТРАТОРОМ (в трейсе Task видно: codebase-analyst вернул текст, файл писал скилл); при пробелах адресации — `.tasks/work/tc-{id}/locators.md` от dom-explorer; тест-функция в `tests/test_<feature>.py`; `pytest --collect-only <файл>` зелёный. В логе Task — спавн codebase-analyst/dom-explorer/code-writer **на opus**.
- **FAIL:** analysis.md пишет субагент (эвристика report-file заблокирует) / раскладка плоская `.tasks/tc-…` / субагенты на дефолтной модели.

**Шаг 2 — batch-autotest:**
- Дать URL фильтра TestOps (набор TC).
- **PASS:** создан `.tasks/work/wave-<кампания>/PROGRESS.md` + ≥1 дом `.tasks/work/tc-{id}/`; веер research-агентов реально стартовал (дефолт 3: 1 general-purpose TMS + 2 codebase-analyst); тесты пишет оркестратор, не code-writer.

**Шаг 3 — fix-failed-test:**
- Дать имя упавшего теста / ланч-URL.
- **PASS:** `.tasks/work/batch-{ts}/fix-{test}-analysis.md` от codebase-analyst(fix); при категории DOM_CHANGED — `fix-{test}-locators.md` от dom-explorer; правка тест-слоя от code-writer (с самопроверкой py_compile); верификационный прогон зелёный; отчёт `fix-batch-{ts}-report.md`.

**Шаг 4 — секреты (негативный):**
- Снять `TESTOPS_TOKEN` и убрать `secrets.yaml` → выполнить любой TestOps-запрос.
- **PASS:** `TestOpsError` с инструкцией (не молчаливый фейл, не хардкод-фолбэк). `grep -rn "allure.iva.ru\|Bearer <literal>"` по установленной копии = 0.

**Шаг 5 — opus-инвариант (сквозной):**
- **PASS:** в трейсе Task всех шагов 1–3 у ВСЕХ трёх субагентов модель = opus.

---

## Итог

Статика/схема/провенанс — **чисто по всем 7 пунктам**, самоотчёт implementer'а подтверждён по факту (одно методологическое расхождение в счёте файлов — не подлог). Один системный не-блокер (двойной фронтматер body — общерепозиторная конвенция, отдельным тикетом). Приёмка переноса kit (#1+#2) — **acceptance: PASS на статике**. Живой прогон — за держателем one-web по приложенному сценарию.

## Связано
[[impl-qa-kit-subagents]] · [[explore-qa-kit-transfer-manifest]] · [[recon-kit-full-qa-dorabotka]]