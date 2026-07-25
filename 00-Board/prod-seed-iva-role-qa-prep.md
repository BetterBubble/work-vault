---
title: prod-seed-iva-role-qa-prep
type: note
permalink: tacticum/00-board/prod-seed-iva-role-qa-prep
status: draft
tags:
- prod-seed
- iva-role-qa
- catalog-mcp
- deploy-prep
---

# Прод-сид iva-role-qa (+ лейны) — подготовка к «нажать выполнить»

READ-ONLY подготовка. Ничего не применено, ssh к проду НЕ выполнялся. Прод-сид применяет ПРЕЗИДЕНТ (gated).

**Контекст:** VPS `tacticum_dev` (159.194.224.59), `/opt/tacticum`, БД `tacticum_catalog`, контейнер `tacticum-catalog-mcp-1`. Репо `~/tacticum/tacticum-dev` @ `20ff9b8` (main, после мержа #133). Working tree по трём профилям чистый (`git status --short` пусто). Docker-образ НЕ пересобирать — только контент (templates + seed_community.py); rebuild для контентного PR не нужен (рунбук шаг 2 «Пропускается, если менялся только контент профилей»).

Источники: рунбук `apps/backend/.serena/memories/deployment_prod_catalog_mcp.md` (в репо — `tacticum-dev/.serena/memories/deployment_prod_catalog_mcp.md`), скилл `.claude/skills/profile-authoring/SKILL.md`, `apps/backend/scripts/seed_community.py`, `apps/backend/src/backend/catalog/application/seed_profile.py`.

---

## (а) Version bump + CHANGELOG — по факту в репо

| Артефакт | manifest.yaml version | CHANGELOG-запись | Вердикт |
|---|---|---|---|
| `iva-role-qa` | **0.5.0** (стр. 22) | `## [0.5.0] — 2026-07-23` есть (смена pure-composition→пак роли, 0.4.0→0.5.0) | ✅ version + CHANGELOG на месте |
| `iva-qa-autotest-base` | **0.1.0** (стр. 22) | `## [0.1.0] — 2026-07-21` есть; правки 2026-07-23 внесены **под той же 0.1.0** (пометка «WIP под той же 0.1.0 — лейн не зарелизен») | ⚠️ version + CHANGELOG есть, НО контент менялся без bump относительно первой 0.1.0 (см. риск R1) |
| `iva-qa-mcp` | **0.1.0** (стр. 19) | `## [0.1.0] — 2026-07-23` есть (новый профиль) | ✅ version + CHANGELOG на месте |

Файлы: `templates/<id>/manifest.yaml` + `templates/<id>/CHANGELOG.md`.

**Механика reject** (`seed_profile.py:253-271`): если в БД уже есть строка `(profile_id, version)` и `payload_hash` отличается → `status="rejected", reason="version_already_exists_with_different_content"`, seeder завершается кодом 1. Идентичный payload → `noop / identical_payload` (не пишет). Нет строки этой версии → `created` (пишет + commit).

Для `iva-role-qa` 0.5.0 — версия новая, если в проде максимум 0.4.0/раньше → будет `created`.
Для двух лейнов 0.1.0 — `created`, **только если в проде их ещё НЕТ**. Если 0.1.0 уже засиживалась раньше с более старым контентом → reject (R1).

---

## (б) Точные команды сида (цитата рунбука)

**Шаг 1 — pull** (как `tacticum-deploy`, НЕ root — иначе dubious ownership):
```
sudo -u tacticum-deploy git -C /opt/tacticum pull --ff-only origin main
```

**Шаг 5 — seed** (образ НЕ содержит scripts/ и templates/ → one-off контейнер с монтированием host-путей на compose-сети; наследует env сервиса DATABASE_URL=…@postgres:5432 и сеть):
```
docker compose -f docker-compose.prod.yml run --rm --no-deps \
  -v /opt/tacticum/apps/backend/scripts:/app/scripts:ro \
  -v /opt/tacticum/templates:/app/templates:ro \
  catalog-mcp python scripts/seed_community.py templates
```

**Важные факты про seeder (`seed_community.py`), которые надо знать до нажатия:**
- Скрипт сидит **ВЕСЬ каталог** `templates` (glob `*/manifest.yaml`), **нельзя** указать один профиль аргументом. Единственный позиционный аргумент = каталог. Прочие (неизменённые) профили дадут `noop` — безопасно.
- **Two-pass порядок** (стр. 148-161): сначала профили без `depends_on` (в т.ч. `tacticum-core-base`, `iva-qa-autotest-base`, `iva-qa-mcp`), потом зависимые (`iva-role-qa`). Т.е. лейны сидятся ДО роли — `depends_on_missing_ref` не будет, если лейны прошли.
- **Commit — по-профильно внутри цикла** (стр. 194): нет единой транзакции на весь прогон. `created` пишет и коммитит сразу; `rejected`/`noop` не мутируют. Значит частичное применение возможно: если роль отклонится, уже засиженные лейны останутся в БД.
- Роль composе'ит по `depends_on: [tacticum-core-base, iva-qa-autotest-base, iva-qa-mcp]` — при сиде резолвится **последняя ACTIVE версия** каждого лейна (`seed_profile.py:276-294`). Значит лейны обязаны быть засижены и активны в этом же прогоне (two-pass это обеспечивает).
- Ожидаемый вывод при чистом проде: `[created]` для `iva-role-qa`, `iva-qa-autotest-base`, `iva-qa-mcp`; `[noop]` для остальных; финал `Seeded N profile(s) successfully.` + exit 0. Любой `[rejected]` → exit 1 и список failure в stderr.

**Проверка после сида (рунбук шаг 6-7):**
```
curl -sk https://dev.tacticum.dev/readyz
```
(rebuild не делаем → MCP-транспорт-проверка шага 7 не обязательна, но безвредна.)

---

## (в) DRY-RUN / предпросмотр — ЕГО НЕТ

**Флага dry-run/preview у `seed_community.py` НЕТ.** Скрипт не использует argparse — единственный вход это `sys.argv[1]` (каталог templates) + env `DATABASE_URL`, `GITHUB_SHA`. Коды выхода: 0 (created/noop), 1 (reject/error). Rollback в скрипте есть только per-manifest на исключении (`seed_community.py:201`), не как режим предпросмотра.

Безопасные способы «посмотреть, что засидится, без применения к проду» (по возрастанию усилий):

1. **Локальный статический guardrail (контент↔CHANGELOG, БЕЗ БД)** — `scripts/check_profile_version_discipline.py`. Ловит: `bump-needed` (контент менялся без bump — та самая причина `version_already_exists_with_different_content`), `changelog-missing`, `manifest-lags-changelog`. Запуск локально в репо (НЕ на проде):
   ```
   cd ~/tacticum/tacticum-dev
   python scripts/check_profile_version_discipline.py --diff-against origin/main
   ```
   ⚠️ Это проверка репо, а НЕ состояния прод-БД. Она НЕ покажет, лежит ли в проде конфликтующая 0.1.0. Зелёный результат ≠ «сид пройдёт на проде».

2. **Read-only запрос прод-БД (снимает главную неопределённость R1)** — БЕЗ применения посмотреть, какие версии трёх профилей уже есть в проде:
   ```
   docker exec tacticum-catalog-mcp-1 psql "$DATABASE_URL" -c \
     "select profile_id, version, status from profile_versions \
      where profile_id in ('iva-role-qa','iva-qa-autotest-base','iva-qa-mcp') \
      order by profile_id, version;"
   ```
   (плейсхолдер: точные имена таблицы/колонок сверить — модель `ProfileVersion`, поля `profile_id`/`version`/`status`; это SELECT, чтение). Если строк 0.1.0 для лейнов нет и 0.5.0 для роли нет → сид создаст без reject.

3. **Полноценный предпросмотр created/noop/rejected без риска** — прогнать seeder против ОДНОРАЗОВОЙ копии прод-БД: `pg_dump tacticum_catalog` → restore в scratch-postgres → `DATABASE_URL=<scratch> python scripts/seed_community.py templates`. Покажет точные статусы по каждому профилю. Тяжелее, но единственный способ увидеть реальный вердикт seeder до прод-прогона.

Встроенного dry-run нет — честный вывод: минимальный безопасный предпросмотр = п.1 (репо) + п.2 (SELECT прод-БД); полный — п.3.

---

## (г) Re-pin существующих installations

По рунбуку (чек-лист релиза, п.7) и скиллу (Фаза 4): **re-pin НЕ является частью сида и НЕ обязателен**.
- «Существующие installations остаются на пинах — пользователи обновляются через update-флоу мануала (`update=true`) либо админ двигает пины (staged rollout)» (рунбук стр. 73-74).
- Сид только публикует новую версию в каталог; существующие пины на неё не переезжают автоматически.

Следствия для 0.5.0 `iva-role-qa`:
- Если инсталляций `iva-role-qa` в проде **ещё нет** → re-pin не требуется (нечего перепинивать).
- Если инсталляции есть (напр. на 0.4.0) и президент хочет их на 0.5.0 → это **отдельное осознанное действие** после сида (staged rollout: админ двигает пины / `provision_installation` с новой версией / update-флоу), НЕ часть шага сида.
- Лейны `iva-qa-autotest-base` / `iva-qa-mcp` самостоятельно не устанавливаются (composе'ятся в роль) → их installations как таковых нет, re-pin к ним неприменим.

Проверить наличие инсталляций роли (read-only, до решения о re-pin):
```
docker exec tacticum-catalog-mcp-1 psql "$DATABASE_URL" -c \
  "select id, profile_id, pinned_version from installations where profile_id='iva-role-qa';"
```
(плейсхолдер: имя таблицы/колонки пина сверить с моделью installations).

---

## ⚠️ Открытые / на решение президента

- **R1 (главный риск reject):** `iva-qa-autotest-base` менял контент под неизменной версией `0.1.0` (CHANGELOG: правки 2026-07-23 «WIP под той же 0.1.0»). CHANGELOG утверждает «лейн не зарелизен» → в прод, вероятно, ещё не сидился, тогда `created` без проблем. **НО** если 0.1.0 уже попадала в прод-БД раньше со старым контентом — сид отклонит его с `version_already_exists_with_different_content`, и роль `iva-role-qa` не досидится (или досидится с устаревшим лейном). **Снять неопределённость запросом (в) п.2 ДО прод-прогона.** Если конфликт есть — нужен version bump лейна до 0.2.0 (это правка репо + новый PR, вне текущей подготовки).
- **Частичное применение:** commit по-профильно. Если роль отклонится после успешных лейнов — лейны останутся засижены, роль — нет. Не атомарно.
- **Dry-run отсутствует** — президент применяет «вслепую» относительно прод-БД, если не выполнить предпросмотр (в) п.2/п.3 заранее.
- **Re-pin** — требуется ТОЛЬКО если есть существующие installations `iva-role-qa`, которые надо двинуть на 0.5.0; это отдельное решение/действие, не сид. Проверить наличие — (г).
- **Не проверено вживую (нет ssh к проду по заданию):** реальное состояние прод-БД (какие версии/инсталляции уже есть), точные имена таблиц/колонок в SELECT-запросах (сверить с моделями `ProfileVersion` / installations перед запуском).
- **Wiki-sync (рунбук шаг 8)** — обязательный шаг релиза профиля после сида, но это не блокер самого сида; отдельным действием.
