---
title: explore — история удалённого лейна iva-write-base
type: note
status: draft
created: 2026-08-05
updated: 2026-08-05
permalink: tacticum/00-board/explore-lane-istoriya-iva-write-base-2026-08-05
tags:
- board
- explore
- iva-write
---

# История лейна `iva-write-base` (факты из git)

Репозиторий: `tacticum-dev`, worktree `/Users/bubblemac/tacticum-worktrees/iva-write-lane`,
`origin/main` = `d607513`. Всё ниже — из `git log --all`, цитаты дословные.

## ГЛАВНЫЙ ВЫВОД

**Имя `iva-write-base` НЕ отравлено. Переиспользовать можно.**

Ключевой факт: **каталог `templates/iva-write-base/` никогда не был на `main`**.
`git merge-base --is-ancestor f69524d origin/main` → **NO**. Лейн жил ровно 2 дня
внутри одной фича-ветки `feat/qa-kit-subagents` (21.07 создан → 23.07 удалён), а PR #133
уехал в main сквошем `20ff9b8` (24.07) **уже без него** — `git ls-tree 20ff9b8 templates/`
лейна не содержит.

Следствия:
- **в прод-БД профиля быть не может.** Сид директорийный:
  `apps/backend/scripts/seed_community.py` — «reads templates/*/manifest.yaml → upserts to
  DB», `templates_dir.glob("*/manifest.yaml")`. Не было директории на main → нечего было
  сидить. Оба прод-сид-раннбука (`docs/runbooks/prod-seed-2026-07-25-rollback.md`,
  `…-07-27-…`) лейн не упоминают; в списке «намеренно НЕ выкачено» он тоже не значится.
- **в тестах, голденах, ROLE_LANES, `_mirrors.yaml`, миграциях следов нет.** Единственная
  запись в ROLE_LANES была снята `0ef6bd7` (23.07) в той же ветке.
- в текущем дереве осталось **5 упоминаний, и все — предупреждения «не воспроизводить
  адрес»**, а не рабочие ссылки (список ниже).

**Что проверить перед сидом (2 SELECT'а, 1 прогон):**
1. `SELECT profile_id, version FROM profiles WHERE profile_id LIKE 'iva-write%';` на проде
   — ожидание: 0 строк. Если строка есть (кто-то сидил руками с фича-ветки 21–23.07) —
   версия `0.1.0` занята, и новый лейн обязан стартовать **не с 0.1.0**, иначе сид
   отвергнет: `version_already_exists_with_different_content` (ADR-0009,
   `test_seeder_immutability.py::test_different_content_same_version_is_rejected`).
2. **Реальный риск не в имени профиля, а в `ingredient_id`.** Если новый `iva-write-base`
   понесёт `helm-iva-write`, а `iva-analysis-fr` продолжит его нести — у аналитика,
   который композит оба лейна, падает
   `test_iva_role_presets.py::test_single_owner_lanes_are_pairwise_disjoint` (ADR-0057).
   Выбор: либо снять ингредиент из `iva-analysis-fr`, либо вписать в `KNOWN_OVERRIDES`
   (сегодня там только `iva-role-architect: {iva-atlassian-write}`).
3. Новый лейн придётся внести в `ROLE_LANES` каждой роли, которая его композит — оракул
   полноты каталога иначе сообщит о забытой композиции (`_NOT_A_ROLE` рядом).
4. Адрес: **`https://helm.tacticum.ru/mcp/iva-write`**. Прежний
   `https://mcp.tacticum.ru/iva-write/mcp` не существует — манифест с ним провалидируется,
   а сервер не ответит (об этом три предупреждения в дереве, см. §4).

---

## 1. Кто завёл и кто удалил

| Что | Хеш | Дата | Автор | Ветка |
|---|---|---|---|---|
| Заведён | `f69524d` | authored 2026-07-21, committed 2026-07-23 | Александр Шульга | `feat/qa-kit-subagents` |
| Удалён | `81fa9fd` | 2026-07-23 | Александр Шульга | `feat/qa-kit-subagents` |

Заведение — `feat(iva-write-base): скелет leaf-лейна write-канала (2A, Вариант A)`,
3 файла / 173 строки (`manifest.yaml` 91, `README.md` 52, `CHANGELOG.md` 30).

Удаление — `architect+techwriter: ретайр iva-write-base → own iva-atlassian-write (скоуп по
роли)`; в том же коммите −173 строки лейна и переписаны манифесты двух ролей.

Обе ветки-носителя: только `feat/qa-kit-subagents` (`git branch -a --contains` — одна
локальная + одноимённая remote). В `main` не попадал.

## 2. Манифест на момент жизни (`git show f69524d:templates/iva-write-base/manifest.yaml`)

- `schema_version: "2"`, `profile_id: iva-write-base`, `version: "0.1.0"`,
  `maintainer: mr.diaret@ya.ru`, `deprecated: false`
- `name: "IVA Write Base — продакшн write-канал публикации FR (Confluence/Jira) под подписью актора"`
- **`depends_on` НЕТ вообще** — leaf-лейн («Лейн leaf: НЕ несёт depends_on (он сам база);
  role-пресеты (techwriter/architect/qa) его composе'ят (2B)»)
- `stack.required: []`, `stack.optional: []`
- `persona.role: requirements-author`
- `ide_targets`: claude-code full, codex full, остальные unsupported
- **ровно 1 ингредиент:**

```yaml
  - ingredient_id: iva-write
    kind: mcp_server_spec
    tier: trial
    supports: [claude-code, codex]
    install_scope: repo
    body: ""
    metadata:
      transport: http
      url: "https://mcp.tacticum.ru/iva-write/mcp"
      env_required: [TACTICUM_TOKEN]
      auth_type: bearer
      required_scopes: [iva-req-write]
      allowed_tools:
        - confluence_create_page
        - confluence_update_page
        - jira_create_issue
        - jira_add_comment
        - jira_transition_issue
```

- `non_goals`: авторинг постановки (это analysis-лейн), read-тулы знаний, клиентские PAT в
  манифесте, роль-композиция (`depends_on` — задача 2B).
- Из CHANGELOG 0.1.0: «`required_scopes` — **первое применение поля** (в схеме есть,
  dev-сидер его не читает, gateway-registry энфорсит)».
- Из README: «Целевой endpoint `iva-write` **физически ждёт Ф1 ADR-0058**… До подъёма —
  действующий write физически идёт через `iva-read/mcp`».

**Гардрейл, ради которого выбирали имя ингредиента** (комментарий в манифесте, дословно):
«ingredient_id `iva-write` (НЕ `iva-read`/`iva-mcp`) — во избежание single-owner-коллизии с
analysis-base». Эта же логика сегодня работает против нас в другую сторону — см. пункт 2
блока «что проверить».

## 3. Почему удалили

**Не из-за адреса шлюза. Архитектурное решение + смена auth-модели.**

Тело `81fa9fd`, дословно:

> Унификация write-канала (**решение ГД / ADR-0058**): iva-role-architect и
> tacticum-role-techwriter переведены с лейна iva-write-base на собственный
> mcp_server_spec **iva-atlassian-write** (локальный mcp-atlassian под личным Atlassian PAT).

Тело `86e6038` (тот же день, QA-роль — первой):

> Ретайр write-лейна iva-write-base из QA-роли (решение ГД / ADR-0058): depends_on: убран
> iva-write-base → […]; добавлен собственный mcp_server_spec iva-atlassian-write (прод-блок
> из iva-fr-analyst): локальный mcp-atlassian (uvx, Server/DC против jira.iva.ru), личный
> Atlassian PAT через env (JIRA_URL + JIRA_PERSONAL_TOKEN)

Тело `4532173`:

> После снятия границы (все 3 write-роли унифицированы, лейн iva-write-base сносится): флаг
> «лейн физически НЕ удалён / нужен architect+techwriter» заменён на актуальный
> (унификация: все три роли на own iva-atlassian-write; прежний write-лейн снесён);
> литеральные упоминания снесённого лейна убраны из нарратива (**grep iva-write-base = 0**).

Причина по существу: **лейн описывал канал, которого физически не было.** Он был построен
под «Вариант A» (шлюз `mcp.tacticum.ru/iva-write/mcp` + phk_*-ключ + принудительная подпись
актора), а этот endpoint «физически ждёт Ф1 ADR-0058» — его не подняли. Рядом уже работал
живой канал `iva-atlassian-write` (локальный mcp-atlassian под личным PAT, заведён 20.07
в `iva-fr-analyst` 0.1.3, `b6728a9`). Выбрали живой и снесли ненаступивший.

Фоном — конфликт, зафиксированный на доске 21.07
(`[[⚠️ Конфликт 2A iva-write ↔ ADR-0058 (личный PAT) — развести до дизайна]]`): план 2A
строился на личном PAT + мультиключевой схеме, а ADR-0058 Решение 5 личный PAT **отверг**
в пользу техучётки `iva`. 2A запарковали, а через два дня лейн сняли.

**Разворот сразу после удаления (важно):** ретайр в own-ингредиенты сломал инвариант
ADR-0057 «роль = pure-composition leaf» — оба коммита ретайра несли явный ФЛАГ:
«роль несёт 1 own-ингредиент → перестаёт быть pure-composition leaf (ADR-0057);
test_iva_role_presets.py инвариант … не держится». В тот же день это откатили обратно в
лейны — но **не в общий, а в три per-role** (`6a3aa44`, `4bfbbdc`, «вариант B»):

> Один MCP-лейн на роль (per-role скоуп; во избежание двух mcp_server_spec на один сервер
> mcp-atlassian в одной композиции).

То есть от «один общий write-лейн на все роли» отказались осознанно — и ровно к этой схеме
мы сейчас возвращаемся. Оговорку про «два mcp_server_spec на один сервер в одной
композиции» стоит держать в голове при дизайне.

## 4. Что осталось в дереве (main `d607513`)

Директории нет. 5 упоминаний, все — предупреждения об адресе, ни одного рабочего:

- `docs/runbooks/iva-write-rollout-to-roles.md:45` — «⚠️ Адрес — `helm.tacticum.ru/mcp/iva-write`.
  Прежний шлюз `mcp.tacticum.ru/iva-write/mcp` (удалённый лейн `iva-write-base`) не
  воспроизводить: **манифест с ним провалидируется, а сервер не ответит**»
- `templates/iva-analysis-fr/manifest.yaml:256` — то же в комментарии к ингредиенту
- `templates/iva-analysis-fr/README.md:29` — то же
- `templates/iva-analysis-fr/CHANGELOG.md:38` — «Прежний шлюз … больше не существует — в
  манифесте стоит предупреждение, чтобы адрес не воспроизвели по памяти»
- `apps/backend/tests/catalog/test_role_install_smoke.py:283` — комментарий к пину адреса:
  «уехал адрес. Прежний шлюз … больше не существует, и вписать его по памяти легко —
  манифест провалидируется, а MCP просто не ответит»

Проверено грепом по всей истории (`git log --all -S'iva-write-base'`): 15 коммитов, из них
9 — жизненный цикл 21–23.07 в фича-ветке, 6 — эти самые предупреждения (30.07–31.07).
Голденов, миграций, `_mirrors.yaml`, ROLE_LANES с этим именем сегодня **нет**.

## 5. Хронология канала записи

| Дата | Хеш | Событие |
|---|---|---|
| 2026-07-20 | `b6728a9` | Первый живой write-канал: `iva-atlassian-write` в `iva-fr-analyst` 0.1.3 (US #699-701) — локальный mcp-atlassian, личный Atlassian PAT |
| 2026-07-21 | `f69524d` | **Рождение `iva-write-base` 0.1.0** — leaf-лейн, шлюз `mcp.tacticum.ru/iva-write/mcp`, phk_*-ключ, `required_scopes: [iva-req-write]` |
| 2026-07-21 | `8a1a92a` | Три роли-пресета 2B (architect/qa/techwriter) на `[core, analysis, write]` |
| 2026-07-21 | `11ac8e9` | `iva-qa-autotest-base`; `iva-role-qa` пересобран на `[core, qa-autotest, iva-write-base]` |
| 2026-07-21 | доска | Флаг конфликта 2A ↔ ADR-0058 (личный PAT vs техучётка); 2A **запаркован** |
| 2026-07-23 | `86e6038` | Ретайр из QA-роли → own `iva-atlassian-write` (Jira-only) |
| 2026-07-23 | `1e0a8b3` | Автотест-лейн переключил ссылки на `iva-atlassian-write` |
| 2026-07-23 | `81fa9fd` | **Удаление каталога лейна** + перевод architect/techwriter |
| 2026-07-23 | `4532173` | Вычистка нарратива, `grep iva-write-base = 0` |
| 2026-07-23 | `6a3aa44` | **Разворот:** три тонких per-role MCP-лейна (`iva-qa-mcp`, `iva-architect-mcp`, `iva-techwriter-mcp`), «вариант B» |
| 2026-07-23 | `4bfbbdc` | Роли вернулись в чистую композицию (`ingredients: []`) |
| 2026-07-23 | `0ef6bd7` | `ROLE_LANES` под вариант B — `iva-write-base` заменён на per-role лейны |
| 2026-07-24 | `20ff9b8` | **PR #133 сквошем в main — уже без лейна** |
| 2026-07-25 / 07-27 | runbooks | Прод-сиды; `iva-write-base` нет ни в выкаченном, ни в «намеренно не выкаченном» |
| 2026-07-29 | — | `iva-analysis-fr` 0.1.0 (расщепление `iva-analysis-base`), владелец FR-ингредиентов |
| 2026-07-30 | `2ef9f08` | **`helm-iva-write` объявлен** у `iva-analysis-fr` (0.2.0) — `https://helm.tacticum.ru/mcp/iva-write`, тот же `TACTICUM_TOKEN` |
| 2026-07-30 | `ede30c6`, `160a163`, `cf1f3c3` | Пин адреса в тестах, квикстарт аналитика, раннбук раскатки |
| 2026-07-31 | `0b5558e`…`74e8a6c` | Ветка `feat/iva-write-rollout-roles`: QA + архитектор, 4 уровня оракулов, страница согласия |
| 2026-08-04 | `7250d6c` | Сверка `iva-analysis-fr` с ролью (канал записи, README) |

**Развороты, видимые в истории:** (1) общий write-лейн → own-ингредиенты в ролях → три
per-role лейна (всё 23.07); (2) шлюз `mcp.tacticum.ru/iva-write/mcp` → локальный
mcp-atlassian под личным PAT → шлюз `helm.tacticum.ru/mcp/iva-write`; (3) auth: техучётка
`iva` с принудительной подписью (ADR-0058 Решение 5) → **однократное согласие сотрудника и
его личный доступ** — реализация с ADR **расходится осознанно**, комментарий в
`iva-analysis-fr/manifest.yaml`: «Реализация РАСХОДИТСЯ с текстом ADR-0058 Решение 5 …
ADR ждёт правки — расхождение осознанное, а не опечатка».

## 6. Ветка `feat/iva-write-rollout-roles` (worktree `td-rollout`)

13 коммитов поверх main, 44 файла, +1670/−194. Дерево чистое.

**Что сделано:**
- `iva-qa-mcp` 0.1.0 → **0.2.0**: объявлен `helm-iva-write` (обе QA-роли композят один лейн)
- `iva-architect-mcp` 0.1.1 → **0.2.0**: то же, со **сужением `allowed_tools` по роли**
  (`e63ed71` «сузить канал записи по роли, а не выдать его заново»)
- `iva-role-architect` 0.4.2 → **0.4.3**
- Паки трёх ролей (`CLAUDE.md.fragment`, `AGENTS.md.fragment`, `config.toml.template`)
  переписаны: `helm-iva-write` — основной канал, `iva-atlassian-write` — временный запасной
- `docs/user_manuals/iva-write-consent.md` (89 строк) — **одна общая страница согласия**
  вместо текста в квикстарте
- `docs/runbooks/iva-write-rollout-to-roles.md` +213 строк
- `apps/backend/tests/catalog/test_role_install_smoke.py` **+395 строк** — четыре уровня
  оракулов: композиция из БД (не только манифесты), README роли, карточка роли в каталоге,
  распознавание канала по перифразу; пин стал data-driven и проверяет скоуп и паки
- `test_install_flow.py` `_IVA_WRITE_ROLES` + голдены четырёх ролей

**Коллизия версий (дословно, из diff):**

`iva-role-qa` — версия **0.7.0 и в ветке, и в main**, содержимое разное. В main
CHANGELOG 0.7.0 = «переиздание под `tacticum-autotest-core` 0.5.0 и canvas-лейн 0.2.0.
Содержание пака роли не менялось». В ветке 0.7.0 = «Паки роли переключены на
`helm-iva-write` как ОСНОВНОЙ канал записи в Jira». Один номер — два разных содержания.

`iva-role-qa-web` — **0.3.0 в обоих**, ровно та же картина (в main «переиздание под
autotest-core 0.5.0 и selenium-лейн 0.3.0», в ветке — переключение паков + правка
`post_install_notes`).

Механика поломки названа прямо в CI: `.github/workflows/profile-version-discipline.yml` —
«Content under templates/<profile_id>/** changed without manifest.version bump (would cause
`[rejected] version_already_exists_with_different_content` on next seed)». Ветку нельзя
смержить как есть: обеим QA-ролям нужен бамп (0.7.1 / 0.3.1) + перенос записи CHANGELOG
под новый номер. Остальные профили ветки конфликта не дают (`iva-qa-mcp`,
`iva-architect-mcp`, `iva-role-architect` подняты выше main).

**Что из ветки переиспользуемо для нашего лейна:**
- `docs/user_manuals/iva-write-consent.md` — общая страница согласия. Ровно то, что нам
  нужно: «тексты живут в лейне, иначе девять копий разъедутся».
- Оракулы в `test_role_install_smoke.py` (+395) — проверка канала на четырёх уровнях, в том
  числе **композиции из БД**. Переносится почти как есть, если поменять ожидаемого
  владельца ингредиента.
- Раннбук `iva-write-rollout-to-roles.md` — точные файлы и строки; наша задача его во многом
  обесценивает (лейн вместо девяти правок), но «решения головой» из него (скоуп
  `allowed_tools` по роли) остаются в силе.
- **Что помешает:** ветка кладёт `helm-iva-write` в `iva-qa-mcp` и `iva-architect-mcp`.
  Если мы кладём его в общий `iva-write-base`, эти правки становятся конфликтующими — те же
  роли получат ингредиент дважды из двух лейнов и упрутся в single-owner. Порядок мержа и
  владелец ингредиента должны быть решены до, а не после.

## Что НЕ найдено

- Записи об `iva-write-base` в прод-сид-раннбуках, миграциях, голденах — нет.
- Коммита, где лейн был бы депрекирован штатно (`deprecated: true` / `superseded_by`) —
  нет: его удалили файлами, не проводя через lifecycle каталога. Для профиля, не
  доехавшего до БД, это корректно.
- Подтверждения из живой прод-БД (SELECT не выполнялся) — отсюда пункт 1 проверок.

## Связано

`[[report-iva-write]]` · `[[postanovka-dve-raboty-iva-write-2026-08-05]]` ·
`[[grabli-iva-write-na-chto-ne-nastupat-2026-08-05]]` ·
`[[⚠️ Конфликт 2A iva-write ↔ ADR-0058 (личный PAT) — развести до дизайна]]` ·
`[[Решения по 2A iva-write-base (2026-07-21)]]`
