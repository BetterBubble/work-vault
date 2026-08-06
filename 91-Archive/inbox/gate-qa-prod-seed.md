---
title: gate-qa-prod-seed
type: report
permalink: tacticum/00-board/gate-qa-prod-seed-1
tags:
- gate
- qa
- prod-seed
- controller
- draft
archived-at: 2026-07-31 17:27
---

# Гейт: точечный сид QA-профилей в прод-каталог (draft)

**Роль:** контролёр (независимый гейт перед записью в ПРОД). Read-only, без доступа к серверу.
**Предмет:** сид 4 профилей (`tacticum-core-base` 0.1.1 + `iva-qa-autotest-base` 0.2.0 + `iva-qa-mcp` 0.1.0 + `iva-role-qa` 0.5.0) в `tacticum_catalog` образом #134, без rebuild/миграций.

## ВЕРДИКТ: GO (условный)

Сид сам по себе безопасен — механику подтвердил по коду. Условия перед GO: (A) подтвердить `visibility=public` осознан для IVA-профилей; (B) снять быстрый `pg_dump` каталог-таблиц до сида; (C) выполнить расширенный пост-verify (ниже), особенно рендер-pull роли на #134. Ни одного блокера-NO-GO не нашёл, но нашёл 3 недооценённых риска, которые нужно закрыть проверкой, а не верой.

---

## По пунктам ТЗ

### 1. Безопасен ли сид образом #134 при профилях под #119-архитектуру — есть ли дыра?
**Прошло (с уточнением дыры, которая закрыта анализом).**
- Seeder (`apps/backend/scripts/seed_community.py` + `apps/backend/src/backend/catalog/application/seed_profile.py`) — чистый upsert манифест→БД. **Не вызывает renderer и НЕ валидирует манифест против схемы** (docstring seed_profile: «validation happens at admin endpoint boundary, seeder trusts caller-side payload»). Манифест кладётся как JSON-блоб. → schema-drift #119↔#134 на сиде НЕ проявляется. Вывод «rebuild/миграции не нужны» верен: пишутся только в таблицы Profile/ProfileVersion/ProfileVersionDependency/Ingredient, существующие на схеме 0037.
- **Дыра, которую надо было проверить (закрыта):** профили авторились под #677-renderer merge-контракт (безопасный append роли в CLAUDE.md/AGENTS.md через marker+merge_strategy). Прод-образ #134 — ДО #677 (renderer.py как раз менялся в `_dedupe_actions_by_path`, строки 344-373). На #134 при коллизии путей, где markerless-фрагмент идёт перед append_section роли, merged-action деградирует в plain `write_file` → клиент ПЕРЕЗАПИШЕТ пользовательский CLAUDE.md/AGENTS.md. **НО:** в этой QA-композиции единственный писатель CLAUDE.md и AGENTS.md — фрагменты самой роли (marker `tacticum:iva-role-qa`). Проверил: `tacticum-core-base` эмитит только skills (→ `.claude/skills/`) + mcp (→ config/mcp.json), лейны `iva-qa-autotest-base`/`iva-qa-mcp` — skills/agents/mcp/secrets.yaml/.gitignore, НИКТО не таргетит CLAUDE.md/AGENTS.md. → коллизии путей нет → dedupe-merge не задействуется → баг #134 НЕ кусается для этого семейства. **Остаточный вектор:** `.codex/config.toml` имеет несколько писателей (role config.toml.template `create_if_missing` + mcp-специи core-base/qa-mcp через codex-рендер) — агрегация конкатенацией. На #134 (pre-#677) конкатенация есть (E26 S5 Finding-4), под вопросом только сохранение marker/merge_strategy. Риск низкий (config.toml — create_if_missing, не user-owned проза), но это надо ПОДТВЕРДИТЬ рендером (см. пост-verify).

### 2. Резолв depends_on: против БД или сид-папки? Достаточно ли core-base в папке?
**Прошло.** Резолв в `seed_profile.py:273-311`: для каждого declared base — `SELECT ProfileVersion WHERE profile_id=base AND status='active' ORDER BY seeded_at DESC LIMIT 1` → **против БД-каталога**, НЕ против сид-папки. Ребро замораживается на latest-active версию базы в БД. Сид-папка нужна только как источник манифестов для апсерта.
- core-base в папке — **страховка-noop, НЕ требование**. core-base 0.1.1 уже active в проде → резолв роли на него сработает и без папки. Включение идентичного 0.1.1 → `noop` (identical_payload), seeded_at НЕ меняется → на выбор ребра не влияет. Достаточно — да.
- Two-pass ордеринг (`_seed_all:148-161`) гарантирует порядок: без depends_on (core-base, autotest-base, mcp) — первый проход, commit по-профильно (`:194`); роль — второй проход, к моменту её сида autotest-base+mcp уже active в БД. → резолв всех трёх рёбер проходит.
- Depth-1 (`:295-310`) и self/dup-guards (`:160-178`) — базы без depends_on, ок.
- **Мелкий контр-риск включения core-base в папку:** если git-archived core-base 0.1.1 окажется НЕ байт-идентичен засиженному в проде (нарушение version-discipline в прошлом) → seeder вернёт `version_already_exists_with_different_content` (rejected) → **exit 1 на весь прогон**, хотя роль засидилась бы корректно (её ребро всё равно резолвится на существующий active core-base). Т.е. лишний core-base в папке может дать ложный fail прогона. Митигировать: либо убрать core-base из папки (резолв и так против БД), либо заранее сверить hash. Не блокер, но чистка риска.

### 3. Риск частичного сида (лейны прошли, роль упала) — проявление, обратимость.
**Прошло — риск управляем идемпотентностью.** Commit по-профильно (`:194`) + two-pass: core-base/autotest-base/mcp коммитятся ДО роли. Если роль упадёт после коммита лейнов — в каталоге окажутся 2 orphan-лейна (+core-base noop) БЕЗ роли-композита.
- **Обратимость:** DELETE-пути в сидере нет; откат = ручной SQL (удалить profile_version-строки, каскад). НО installations=0 → orphan-лейны ничего не ломают у пользователей (валидные leaf-профили, просто раньше времени видны в list_profiles). **Само-лечение:** повторный прогон идемпотентен (лейны→noop, роль→created), поэтому фикс частичного сида = просто **пере-запуск**, реального отката обычно не требуется.
- Что проверить при частичном: `profile_versions` содержит все 4; если роли нет — перезапустить сид. Причины падения роли после коммита лейнов маловероятны (deps уже active), но перезапуск лечит.

### 4. Не тащим ли #119 через эти 4 профиля? Изоляция.
**Прошло.** Сид пишет РОВНО 4 profile_version + их ingredients + 3 ребра роли (role→core-base, role→autotest-base, role→mcp). Ни один из ~40 профилей #119-бэклога (лейны/роли go/analyst/architect/java/ios/kmp/mail/web/platform/internal и пр., присутствующие в `templates/`, но НЕ в сид-папке `/tmp/qa-seed`) не сидится. Рёбра роли указывают только на профили из набора. core-base 0.1.1 идентичен проду → noop, новых версий не плодит.
- **Одна тонкость (в чеклист):** ребро role→core-base резолвится на latest **seeded_at** active core-base, а в проде active ДВЕ (0.1.0 и 0.1.1). Резолв — по `seeded_at DESC`, НЕ по semver. Если 0.1.1 засижен позже 0.1.0 (норма) → ребро→0.1.1 (как в плане). Но если порядок seeded_at инвертирован — ребро уедет на 0.1.0. План уже проверяет «ребро на core-base 0.1.1» в verify — хорошо, это ловит.

### 5. Чеклист пост-верификации — полный ли?
**Не прошло как есть — неполный.** Базовый план (4 версии + рёбра + readyz 200 + MCP 401) проверяет, что СТРОКИ легли, но НЕ проверяет главный недотестированный путь — РЕНДЕР роли образом #134. Дополнения ниже.

---

## Что упущено / риски (сверх плана)

- **R1 — рендер-pull роли на #134 не проверяется (главное).** Всё «безопасно» держится на анализе, что коллизий путей нет и passthrough kind-agnostic. Это надо ПОДТВЕРДИТЬ живым рендером роли (и claude-code, и codex), а не только наличием строк в БД. Особый фокус: `.codex/config.toml` агрегация (pre-#677) и codex-тела skill_spec (write-autotest/fix/batch/jira через `codex_body`).
- **R2 — visibility=public для IVA-профилей.** Манифесты НЕ объявляют `visibility` → сидятся `public` (seed_profile default). Значит `iva-role-qa`/лейны становятся видимы/pull'абельны ЛЮБОМУ подписчику каталога. Внутри — раскрытие внутренней топологии ИВА (jira.iva.ru, helm.tacticum.ru/mcp/analyst, репо one-web). **Секретов нет** (mcp bodies пустые, env_required — только ИМЕНА переменных JIRA_URL/JIRA_PERSONAL_TOKEN/TACTICUM_TOKEN, secrets.yaml — example). Но публичная выдача внутренних эндпоинтов — вопрос скоупа/раскрытия. **Подтвердить у тимлида/президента:** public осознан, либо нужен tenant-scope (`visibility: private` + owner_org) — что потребовало бы правки манифестов и ре-сида.
- **R3 — нет DB-бэкапа перед записью.** Рунбук шага бэкапа каталога перед сидом не содержит. Сид аддитивен и обратим ручным DELETE, но дешёвая страховка — `pg_dump` таблиц profiles/profile_versions/profile_version_dependencies/ingredients ДО прогона. Рекомендую.
- **R4 — лишний core-base в сид-папке = вектор ложного fail** (см. п.2). Опционально убрать из папки.
- **R5 — wiki-sync (рунбук шаг 8) — обязательный шаг релиза профиля**, в плане отсутствует. Для гейта БД-безопасности не блокер, но релиз без него неполон: обновить «Набор профилей» (wiki.iva.ru id 208703447) + quickstart, добавить blurb в TABLE_BLURBS для 3 новых профилей. Отдельным шагом после сида.
- **R6 — git-чистота источника:** сид-папка собирается `git archive` из ref (origin/main на `/opt/tacticum`) → некоммиченные/мусорные файлы в архив НЕ попадают by construction. Секретов в 4 манифестах нет. Чисто.

---

## Расширенный пост-verify чеклист (обязательный)

1. **Exit code сидера = 0** + stdout: 4 строки статусов; ожидаемо `noop` core-base, `created`×3 (лейны+роль), финал «Seeded 4 profile(s) successfully». Любой `rejected`/ERROR → разбор (особенно core-base different-content → см. R4).
2. **`profile_versions`:** ровно 4 целевых, все `status='active'` (не draft — иначе не выдаются), версии 0.1.1/0.2.0/0.1.0/0.5.0.
3. **`profiles`:** 3 QA-профиля `is_active=true`, `visibility='public'` (подтвердить осознанность — R2).
4. **`profile_version_dependencies`:** 3 ребра dependent=role-qa@0.5.0 → base ∈ {core-base@**0.1.1**, autotest-base@0.2.0, mcp@0.1.0}, `position` 0/1/2 в порядке манифеста. **Явно сверить, что core-base-ребро указывает на 0.1.1, а не 0.1.0** (R4/п.4 — резолв по seeded_at).
5. **core-base не расплодился:** в `profile_versions` по `tacticum-core-base` по-прежнему только 0.1.0/0.1.1, никаких 0.1.2/новых.
6. **РЕНДЕР-PULL роли (R1) — критично.** Симулировать pull `iva-role-qa` образом #134 для **claude-code И codex**; проверить:
   - CLAUDE.md/AGENTS.md приходят как marker-wrapped append (`tacticum:iva-role-qa`), НЕ как bare-overwrite;
   - `.codex/config.toml` собирается корректно (mcp-серверы core-base + iva-atlassian-write + helm-analyst + template роли), без потери контента;
   - codex-тела skill_spec доставлены verbatim (write-autotest/fix-failed-test/batch-autotest/jira-issue-autotest → `.agents/skills/*`), agent_spec → `.codex/agents/*.toml` (model gpt-5.4);
   - список ingredients композита = core+autotest+mcp+пак роли (нет утечки чужих профилей).
7. **list_profiles / tacticum_init:** роль дискаверится и отдаёт latest-active 0.5.0 (выбор по seeded_at DESC; recommended_version НЕ требуется — подтверждено кодом; пин не нужен, installations=0).
8. **`readyz` 200** + **MCP-транспорт POST → 401** (рунбук шаг 7; хотя rebuild не было — дешёвая проверка живости).
9. **Alembic на 0037** (не уехало) — контроль, что сид не дёрнул миграции.
10. **Wiki-sync (R5)** — отдельным шагом релиза после успешного сида.

---
**Итог:** механика сида проверена по коду и непротиворечива фактам о проде. GO при закрытии R2 (подтверждение public), R3 (pg_dump), и выполнении пункта 6 пост-verify (рендер-pull на #134) — это единственный путь, который анализ подтверждает, но живой прогон ещё нет. Ничего не правил и не запускал (read-only).