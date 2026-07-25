---
title: profile-testing-guide
type: note
permalink: tacticum/90-materials/profile-testing-guide-1
---

# Как протестировать роль перед выкаткой (на примере iva-role-qa)

Проверенный проход: так тестировались все 10 ролей матрицы ADR-0059, включая живые
пилоты java/kmp/internal на стенде. Уровни идут от дешёвых к дорогим — не перескакивай:
каждый следующий ловит то, что не видит предыдущий.

## Уровень 0 — подключи роль в тестовую матрицу (без этого остальное не работает)

Роль, которой нет в матрице, не проверяется вообще. Четыре места в `apps/backend/tests/`:

1. `catalog/test_iva_role_presets.py` → `ROLE_LANES` + `ROLE_PERSONA`: добавь
   `"iva-role-qa": ["tacticum-core-base", "iva-qa-autotest-base", "iva-qa-mcp"]`.
   Это даст: схему, «роль несёт только пак-kinds + маркер», depth-1, дизъюнктность
   лейнов, golden-parity композиции.
2. `catalog/test_role_install_smoke.py` → `ROLES`: смок материализации — body-файлы
   существуют, целевые пути обоих CLI без коллизий, packs не дублируются, роль даёт
   входные точки (CLAUDE.md / AGENTS.md + .codex/config.toml), оракул «команды,
   оркестрирующие суб-агентов ⇒ агенты в составе».
3. `catalog/test_role_replacement_parity.py` → `REPLACEMENTS`: только если роль
   ЗАМЕЩАЕТ старый профиль из каталога. У qa предшественника нет — пары не будет
   (как у iva-role-java).
4. `e2e_install/test_install_flow.py` → `_GENERIC_ROLES`: добавь запись с
   `present` (5–8 ключевых ingredient_id, которые обязаны приехать: например
   `write-autotest`, `codebase-analyst`, `iva-atlassian-write`, `bug-fix`…) и
   `absent` (что НЕ должно попасть: `coder`, `run-implementation`, `fr-authoring`).
   Лейны тест возьмёт из manifest.depends_on сам, ожидаемый состав посчитает из
   файловых манифестов и сверит с тем, что вернула БД.

## Уровень 1 — статика (секунды)

```bash
cd apps/backend
uv run -m pytest tests/catalog/test_iva_role_presets.py \
  tests/catalog/test_role_install_smoke.py \
  tests/catalog/test_role_replacement_parity.py -q
cd .. && python3 scripts/check_profile_version_discipline.py --diff-against origin/main
python3 scripts/check_mirror_sync.py   # если у твоих лейнов есть зеркала в старых профилях
```

## Уровень 2 — e2e установки (сид → provision → pull → раскладка → голден)

Тот же код, что боевой сид на dev.tacticum.dev; нужен Docker (тест поднимает Postgres):

```bash
# первый раз — записать голдены (эталонные деревья, path→sha256):
E2E_INSTALL_REGEN_GOLDEN=1 uv run -m pytest \
  "tests/e2e_install/test_install_flow.py::test_install_flow_roles_generic[codex-iva-role-qa]"
E2E_INSTALL_REGEN_GOLDEN=1 uv run -m pytest \
  "tests/e2e_install/test_install_flow.py::test_install_flow_roles_generic[claude-code-iva-role-qa]"
# затем строго (без флага) — оба параметра; голдены закоммить в golden/iva-role-qa/
```

⚠️ Каждый параметр гоняй ОТДЕЛЬНЫМ процессом: в одной pytest-сессии второй ловит
известный локальный asyncpg-флак «Event loop is closed» (на CI не воспроизводится).

После зелёного прогона любое будущее изменение состава роли станет видимым
golden-диффом в ревью — это твоя страховка навсегда.

## Уровень 3 — живой агент на реальной репе ДО мержа/сида

Как это делалось для java (полный цикл на diskstorage) и kmp/internal (read-only).
Суть: e2e-тест уже материализовал точное дерево установки — берёшь его и подкладываешь
настоящему агенту.

1. **Достань дерево** из pytest tmp (последний прогон уровня 2):
   `/private/var/folders/.../pytest-of-<user>/pytest-N/test_install_flow_roles_gener*/bulk/{repo,user}`
   → `tar czf qa-role.tgz repo user`.
2. **Площадка** — отдельный клон целевой репы (НЕ рабочая копия пайплайна/коллег):
   для QA логично one-web. На тестовом стенде (38.180.236.39) есть тулчейн
   (codex, node, serena-LSP) и VPN до контура ИВА.
3. **Раскладка**: `tar xzf` → `cp -R repo/. .` в клон; user-scope скиллы —
   в `.agents/` клона (self-contained). ⚠️ Ручной `cp` ПЕРЕЗАПИСЫВАЕТ существующий
   AGENTS.md репы — если он там есть, допиши секцию между маркерами
   `tacticum:iva-role-qa` руками (штатная установка делает append сама).
4. **Codex headless** (две ловушки, съевшие у нас по часу):
   - папку нужно затрастить: в `~/.codex/config.toml` →
     `[projects."/path/to/pilot"]` `trust_level = "trusted"`;
   - MCP-вызовы в headless режутся аппрувом — запускай
     `codex exec --dangerously-bypass-approvals-and-sandbox -C /path "<промпт>" </dev/null`.
   - токен: `export TACTICUM_TOKEN=…` (для стендовых прогонов подойдёт tac-токен из
     kbconsole.env; учти — он пришпилен к installation IVA, kb_discover видит только её).
5. **Прогоны по нарастающей**:
   - смок: «перечисли свои скиллы и агентов; вызови whoami через tacticum-mcp» —
     проверяет доставку и MCP-коннективность;
   - read-only задача: «пользуясь скиллом X, проанализируй Y; ЗАПРЕЩЕНО менять/создавать/
     удалять файлы и запускать сборку» — проверяет, что знания скиллов работают на живом
     коде. После прогона сверь: `git status --porcelain | grep -v '^??'` пуст (агент
     ничего не тронул);
   - полный цикл: боевая задача через твои команды (write-autotest / run-tests /
     fix-failed-test) на канареечной задаче. Для QA-роли отдельно проверь playwright-цепочку
     и write-канал: iva-atlassian-write ходит с ЛИЧНЫМИ Atlassian PAT (env
     JIRA_URL/JIRA_PERSONAL_TOKEN/CONFLUENCE_URL/CONFLUENCE_PERSONAL_TOKEN) — на
     пилоте пиши только в тестовую issue, ничего не публикуй в боевые.
6. **Фиксация**: лог прогона + git-дифф (если полный цикл) — это и есть ОС пилота.

## Уровень 4 — после мержа: штатная выкатка

1. **Сид** на dev.tacticum.dev (мерж в git сам по себе пользователям ничего не меняет —
   контент раздаёт БД каталога): `seed_community.py` / CI-джоба; проверка —
   `provision_installation(profile_id="iva-role-qa")` работает, состав совпадает с голденом.
2. **Quickstart** роли в `docs/user_manuals/` (по образцу `iva-role-java-profile-quickstart.md`):
   2 хостовых MCP + provision + apply + проверка боем. Без quickstart'а роль не выдать.
3. **Канарейка → когорта** по `docs/user_manuals/role-migration-runbook.md`: 1–2 живых
   пользователя, неделя наблюдения, потом остальные. Реестр установок:
   `DATABASE_URL=… uv run python scripts/installation_registry.py`.

## Частные замечания по iva-role-qa (посмотрел смёрженный состав)

- В `iva-qa-autotest-base` лежит `instruction_pack` (secrets-example) — у нас правило
  «пак — по одному НА РОЛЬ, маркер tacticum:<role>» (иначе при composе двух лейнов с
  паками в один файл конфликтуют маркеры). Проверь, что маркер у лейн-пака не совпадает
  с ролевым и что оба не целятся в один target_file, — уровень 1 (smoke) это покажет.
- Твои суб-агенты (codebase-analyst / dom-explorer / code-writer) и команды — проверь
  связку «команда ссылается на существующего агента»: наш смок-оракул покрывает семейство
  run-implementation; если твои команды спаунят твоих агентов — добавь их пары в оракул
  по аналогии.
- `iva-qa-mcp`: если серверы без `allowed_tools` — роль видит всю поверхность
  helm-analyst; узкий read-пресет для QA обсуждался в твоём же ADR — сузить лучше до
  канарейки, расширять проще, чем отбирать.

Вопросы — пиши; наши прогоны (java/kmp/internal) лежат примерами: e2e-тесты в
`tests/e2e_install/test_install_flow.py`, живой пилот — история PR #119.