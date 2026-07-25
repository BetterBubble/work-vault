---
title: 'Направление: Профили → QA-профиль (iva-role-qa) + AQA-toolkit ИВА'
type: note
permalink: tacticum/11-directions/napravlenie-profili-qa-profil-iva-role-qa-aqa-toolkit-iva
status: current
created: 2026-07-22 17:00
updated: 2026-07-22 17:00
project: tacticum-dev / профили
lead: lead-qa
tags:
- direction
- qa
- role-presets
- aqa
- profiles
- tacticum-dev
- lead-qa
---

# Направление: Профили → QA-профиль + AQA-toolkit ИВА

**Ведёт:** `lead-qa` (взял 2026-07-22 16:54). **Суть:** довести QA-профиль (`iva-role-qa`) до боевого, сомкнув его с **AQA-toolkit команды ИВА (Женя)** — кросс-провайдерным инструментом автоматизации, из которого наши скиллы и растут.

## Действующие лица (со стороны ИВА)
- **Женя** — разработчик флоу AQA (автор toolkit).
- **Методолог/конечный потребитель** — ведёт методологию и приёмку.
- Ещё один тестировщик.
Спросили нас: **«какой дальше план?»** → нужно изучить их репо, дать наш план + вопросы/уточнения.

## Что у нас уже есть (наша сторона)
Роль `iva-role-qa = [tacticum-core-base, iva-qa-autotest-base, iva-write-base]`. QA = **исполнение/автоматизация** автотестов; авторинг TC — на аналитике (развилка ниже). Собрано и принято ([[resheniia-po-qa-profiliu-trek-b-2026-07-21]], [[qa-profile-model-opis-multi-stek-model-qa-leinov]]):
- Лейн `iva-qa-autotest-base` = **9 скиллов** автотест-команды one-web: **6 рабочих** (playwright-cli, run-tests, jira-issue-autotest, prepare-mr-branch, rebuild-autocore, retro); **3 заблокированы** (write-autotest, batch-autotest, fix-failed-test) — ждут **3 субагентов** (`codebase-analyst`, `dom-explorer`, `code-writer`), **запрошены у QA-команды**.
- Флаги: скиллы жёстко на репо `one-web` (autocore/venv/glab/CI) — репо-специфично, не агностично; своего QA-MCP нет; Allure/TestOps локально (не MCP).
- Мульти-стэк: каждый стэк (web/iOS/KMP) = отдельный лейн; сегодня реально **web-only**.

## Новый вход (сторона ИВА, Женя)
Два репо на `git.hi-tech.org/ivaqa` (доступ через helm/adp):
- **`kit`** (НОВЫЙ, главный) — маркетплейс плагинов для **codex и claude**; deploy-ready под проекты автоматизации потребителей, фидбеки/обновления, **3 кросс-провайдерных профиля** (codex-only / codex+claude-ревьюер / codex+claude-флоу).
- **`aqa-agent-toolkit`** (СТАРЫЙ) — предыдущая версия.
⚠️ **Доступа к репо НЕТ** (разведка [[recon-aqa-kit-zhenya]]): `git.hi-tech.org` — helm не резолвит; adp видит внутренний GitLab (10.0.207.3), но аноним → 404/логин, ключей/PAT нет. **Нужен доступ от Жени:** PAT `read_repository` на группу `ivaqa` / деплой-ключ на adp/helm / архив-зеркало. Содержимое kit не изучено. **Подтверждено:** наши 9 скиллов = byte-copy из источника QA-команды → **toolkit = upstream наших скиллов** (разовый форк). ⚠️ Безопасность: в references зашиты креды (`allure.iva.ru` + токен) — вынести в env.

## Ключевые вопросы направления (уточнить у Жени/методолога)
1. **3 недостающих субагента** (`codebase-analyst`/`dom-explorer`/`code-writer`) — есть ли в `kit`? Можем забрать их defs?
2. **`kit` как upstream** — брать его как источник (адаптер/сабмодуль), а не копировать 9 скиллов? Как он деплоится на проекты потребителей?
3. **3 кросс-провайдерных профиля** — как ложатся на нашу модель роль-пресетов (codex/claude)?
4. **Репо-специфичность one-web** — `kit` обобщает это под разные проекты автоматизации? Как берёт окружение/CI?
5. **Развилка «кто генерит тест-кейсы»** — аналитик (`tests-authoring`) vs QA. Методолог как конечный потребитель — за ним.

## Дальше
1. Разведка `kit`/`toolkit` (идёт) → карта.
2. ГД сводит: **наш план + вопросы/уточнения для команды ИВА** (ответ на «какой план»).
3. `lead-qa` ведёт исполнение по approved-решениям + новому входу.

## Ход и решения — 2026-07-23

**Кол ГД #1+#2 в работе.** Снапшот kit (`90-Materials/kit-main.zip`) распакован и разобран ([[recon-kit-full-qa-dorabotka]], [[explore-qa-kit-transfer-manifest]]). Работа изолирована в worktree `tacticum-dev-qa-kit` (ветка `feat/qa-kit-subagents`, от `feat/iva-write-base` — лейна ещё нет в main). Implementer перенёс 3 субагента (agent_spec, все на opus) + стек pytest-playwright-canvas комплектом + тела оркестраторов kit в write/batch/fix-autotest + санитизацию секретов (0 хардкодов `allure.iva.ru`, env-модель `TESTOPS_*`→gitignored `secrets.yaml`) — [[impl-qa-kit-subagents]]. Идёт независимая верификация ([[verify-qa-kit-subagents]]).

**Развилка «наш лейн = шаблон, не плагин kit» разрешена (решение пользователя):** шаблон делаем **стек-нейтральным** — кладём канон, one-web-специфику (пути/юзеры) НЕ бейкаем, только плейсхолдеры + `secrets.yaml.example`.

**Три решения пользователя 2026-07-23:**
1. **Живая приёмка** — 3 скилла реально прогоняет **QA-команда** в one-web (у нас нет one-web/TestOps/стенда). Мы отдаём ветку + точный сценарий приёмки (готовит verifier).
2. **Механизм сборки** — финализируем на **текущей схеме** лейна (manifest v2); под единый канон Димы Солонко (три→один) адаптируем, когда он готов. Не блокируемся на гардрейле, но держим его в виду.
3. **Скоуп «готовый профиль» = автоматизация (#1+#2) + интеграция с аналитиком** — TC-handoff «аналитик пишет TC / QA ревьюит-дополняет» + «дырка под общие части профилей» (из ДОПОЛНЕНИЯ). Этот слой **пересекается с ADR lead-arch** (модель аналитик↔QA↔разработчик) — координируем, не проектируем поверх.

**Верификация #1+#2 (verifier, [[verify-qa-kit-subagents]]):** независимо, по факту на диске — **acceptance PASS на статике, 7/7**. 3 agent_spec на opus (manifest 203/215/227), стек 15 файлов, тела оркестраторов kit, санитизация (0 операционных `allure.iva.ru`), manifest + 15/15 ingredients VALID. Провенанс чист: 6 транспортных коммитов сделаны в ветке сегодня, только лейн (55 файлов). Живой прогон НЕ проведён (нет one-web/кредов) → приложен точный сценарий приёмки для держателя one-web. **Один не-блокер на тикет:** двойной фронтматер body+metadata при рендере `.claude/agents/*` — общерепозиторный вопрос сборки (идентично принятому `iva-2-client-shell-dev`), не дефект ветки; рантайм-opus подтверждён golden-тестом.

**Шов аналитик↔QA (explorer, [[explore-qa-analyst-integration-seam-karta-shva-analitik-qa]]):** интеграционный слой в основном **упирается в ADR lead-arch**. (1) **TC-handoff — код-пути нет:** аналитик даёт TC как markdown-стратегию (`tests.md`, скилл `tests-authoring`) + Jira-линк, QA читает структурированные кейсы из TestOps (`tools.testops`) — форматы не совпадают, кто заносит TC аналитика в TestOps в коде не выражено. (2) **«QA ревьюит-дополняет»** — шов есть частично (`write-autotest/references/tc-review.md`), но это byproduct автоматизации, а не standalone-ревью фичи ДО разработки; пробел. (3) **Общие части** — механизм `depends_on` есть; ⚠️ флаг: шаренные лейны (`iva-analysis-base`/`tacticum-core-base`/`iva-write-base`) физически дублированы в worktree и **дрейфуют** — риск при мердже. **QA без ADR не решает:** канонический формат TC, размещение ревью-ingredient, write-back в TestOps.

**Что осталось до «готово»:** (а) controller-гейт по #1+#2 → передача ветки QA-команде на живой прогон (verifier дал сценарий); (б) чистка `retro/SKILL.md:25` + тикет на двойной фронтматер; (в) **интеграционный слой аналитик↔QA заблокирован ADR lead-arch** — нужен канон TC-handoff + ownership ревью-ingredient; сигналить lead-arch/ГД; (г) worktree-дрейф шаренных лейнов — свести при мердже; (д) порядок доставки: QA-ветка на `feat/iva-write-base` (не в main) — PR-порядок за пользователем.

**Write-канал пересобран (implementer, [[impl-qa-role-atlassian-write]]) — но вскрыл блокер ADR-0057.** По сигналу lead-arch (реш. ГД) ретайрнут лейн `iva-write-base`; все 3 write-роли (`iva-role-qa`/`iva-role-architect`/`tacticum-role-techwriter`) пересобраны на own `mcp_server_spec` `iva-atlassian-write` (mcp-atlassian, личный PAT), скоуп тулов по роли (QA — Jira дефекты/статусы). 4 манифеста VALID, `grep iva-write-base` = 0, лейн физически снесён. ⚠️ **БЛОКЕР:** это ломает инвариант ADR-0057 «пресет = pure-composition leaf (own ingredients = 0)» — **15 упавших кейсов** `test_iva_role_presets.py` (own-MCP в пресете + `ROLE_LANES` ссылается на снесённый лейн). Прецедент own-MCP в роли уже есть (`iva-fr-analyst`). **Решение — за lead-arch:** (A) обновить инвариант + `ROLE_LANES`, либо (B) вернуть write тонким лейном. Сигналю lead-arch.

**Модель субагентов — под пересмотр (поправка пользователя).** Хардкод `model: opus` был ошибочно протащен из «моей» инструкции; для QA-профиля модель — по kit-схеме носителей: базовый носитель Codex (`[capacity].profile` 0/1/2, деградация на Codex), QA-команда на Codex → **Codex-primary, не opus**. Фикс отдельной правкой (сверить, как `tacticum-dev` выражает codex-модели, gpt-5x).

**Новый вход (lead-arch 13:00, реш. ГД вар.B):** QA получает **узкий read-срез `helm-analyst`** для ревью — `requirement_tests` + `gap_questions`/`contradiction_check` + support (`nearest_spec`/`affected_systems`/`constraints`/`related_tasks`); реверс Track-B-исключения (снять helm-analyst из `non_goals`). ⚠️ НЕ авторинг — TC и владение покрытием у аналитика, QA только ЧТЕНИЕ для ревью. Ставится как `mcp_server_spec` в роль/лейн — **тот же own-MCP-в-роли, что упёрся в ADR-0057**. → Реализую вместе с write-каналом **ПОСЛЕ** решения lead-arch A/B + placement (роль-пресет vs лейн), чтобы не плодить падения. Контекст: ADR §8 п.3.

**РЕШЕНО (пользователь) — вариант B, реализовано ([[impl-qa-mcp-thin-lanes]]).** ADR-0057 сохранён (роли = чистая композиция); MCP вынесен в **3 тонких per-role лейна**: `iva-qa-mcp` (atlassian-write Jira-дефекты + helm-analyst read-срез — закрывает сигнал 13:00), `iva-architect-mcp` (Confluence+Jira), `iva-techwriter-mcp` (Confluence+jira-comment). 3 роли → `ingredients: []`, `depends_on` на свой MCP-лейн; `ROLE_LANES` обновлён → `test_iva_role_presets.py` 35/35 + schema-тесты зелёные; `grep iva-write-base`=0; helm снят с `non_goals` QA. **Модели субагентов исправлены:** opus → `gpt-5.4` (Codex профиль 0, поправка пользователя). ⚠️ DB-backed catalog-тесты не прогнаны (нужен backend-venv) → на controller-гейт. Дальше: гейт → чистка `retro:25` + тикет двойной фронтматер → ребейз на main → **бандл-PR**.

**Controller-гейт пройден ([[gate-qa-profile-bundle]]) — GO с условием, дефектов нет.** По факту + реальный прогон в backend-venv: гит-чистота/скоуп/консистентность/валидатор 7/7/память — PASS; **73/73 профильных + 330 non-DB catalog зелёные**; секретов/AI-подписей/мусора нет. **Единственное непроверенное:** DB-backed набор (`test_seed_depends_on` и др.) — локально нет docker-демона → **обязан пройти в docker-CI до мержа** (дефектов не найдено, но PASS не выдать). Настоящих блокеров нет. Не-блокеры: ребейз на main (ветка на 32 коммита позади форка `feat/iva-write-base`), `retro:25`, тикет двойного фронтматера. **Статус: бандл готов к PR — ждёт OK пользователя** (PR/пуш — его шаг); финалка+диф с аналитиком — [[explore-qa-vs-analyst-final]]. **Онбординг команды ИВА (передача+запуск, 6 этапов):** [[qa-profil-onbording-komandy-iva-peredacha-i-zapusk]].

## Связано
[[resheniia-po-qa-profiliu-trek-b-2026-07-21]] · [[qa-profile-model-opis-multi-stek-model-qa-leinov]] · [[qa-validation-plan-validatsiia-iva-role-qa-1-v-1-chek-list]] · [[recon-aqa-kit-zhenya]] · [[recon-kit-full-qa-dorabotka]] · [[explore-qa-kit-transfer-manifest]]

## Уточнения по созвону 2026-07-23 ([[razbor-sozvona-2026-07-23-10-30-profili-qa-protsess-demo-rks]])
- **TC-авторинг → аналитик, QA ревьюит/дополняет** (решение [[reshenie-test-keisy-pishut-analitiki-qa-dopolniaet-na-reviu-2026-07-23]]). QA-профиль = исполнение/автоматизация + ревью-дополнение, НЕ авторинг с нуля.
- **Два слоя QA:** (1) тест-дизайн → к аналитику (первый слой почти есть); (2) наши 9 скиллов + 3 субагента (из kit) = генерация/проверка/автозапуск/валидация.
- **Интеграция с аналитиком без разрыва процесса** + «дырка» (лёгкая опция) под общие/переиспользуемые части профилей.
- Строить на скиллах команды Брейкина (Никита) — они уже привыкли.
- ⚠️ **ГАРДРЕЙЛ — единый механизм сборки:** сейчас ТРИ механизма → цель ОДИН (сводит Дима Солонко за Глебом/Лебедевым). Наш QA-профиль строить по целевому единому механизму, **чтобы за нами не пришлось доделывать**. Посмотреть, как Дима Солонко сводит, и следовать канону.
- **ADR-концепт взаимодействия профилей** (пересечения/отделяемые части анализ-QA-разработка, ownership) — отдельная задача (исполнителя определяем).

## ADR-0060 финализирован (2026-07-23) — КАНОН
Модель взаимодействия профилей → **ADR-0060** (репо `tacticum-dev/docs/adr/0060-profile-interaction-model-mcp-scoping-pipeline-gates.md`; **ранее номер 0059 — переименован в 0060**; 0058 — IVAREQ/iva-write). Ключевое: `allowed_tools` в **capability-лейне** (не на роли); разработчик **implement-only + read-способность-лейн (вариант b)**; **QA = review-augment TC, НЕ авторинг** (уточняет ADR-0058 §6); **гейты §6** (стоп-кран: `done`⇐`covered`+`verified:yes`; покрытие-за-изменением по PIN Impact — обязательная часть гейта шага 4); write через **`iva-write`** (ADR-0058: техучётка+подпись+scope `iva-req-write`). Задача lead-arch на ADR — закрыта; доделки лидам разосланы. Решение TC: [[reshenie-test-keisy-pishut-analitiki-qa-dopolniaet-na-reviu-2026-07-23]].

**Доделки ADR-0060 — ВСЕ закрыты (2026-07-23), ЗАПУШЕНО (`origin/feat/qa-kit-subagents` @ `dc7456f`, PR обновлён):** (1) helm-read через capability-лейн — уже так ✅; (2) write→`iva-write` — **интерим** (сервера нет, платформенный milestone) ⏸; (3) QA=review-augment — уже так ✅ (явный non_goal); (4) **soft-гейты §6** ([[impl-qa-gates-soft]], commit `12be10e`): regression-from-PIN обязателен + `verified:yes` + локальный coverage-ledger, SOFT ✅; (5) **вписан в матрицу** ([[impl-qa-role-matrix-fit]], `iva-role-qa` 0.4.0→0.5.0): пак уровня роли (4 ингредиента по образцу `iva-role-go`) + e2e golden + реконсиляция теста ✅. Ветка: 23 коммита, 87 файлов, чисто (0 мусора/секретов/AI-подписей). ⚠️ **Гарантированный merge-конфликт с #119 на `test_iva_role_presets.py`** (оба меняют `ROLE_LANES`; при мерже qa вернуть вручную — сигнал Глебу отбит); e2e-wiring qa в `_GENERIC_ROLES` — тривиальный шаг после мержа #119. **Push обновит тот же PR — держим за президентом.**

> **Правка номера (2026-07-23):** ADR выше = **ADR-0060** (переименован с 0059 из-за коллизии номера с single-axis-ADR на ветке; lead-arch подтвердил). Канон в репе `docs/adr/0060-profile-interaction-model-mcp-scoping-pipeline-gates.md`.

## Ход — 2026-07-24 (пост-мерж, старт отдачи QA)

**PR #133 СМЕРЖЕН в main** (решение президента; ГД кол 10:01) — `origin/main` @ `20ff9b8` «feat(profiles): QA-профиль iva-role-qa + автотест-лейн + per-role MCP-лейны (#133)». `lead-qa` подхватил (адрес зарегистрирован). Кол ГД: шаги 1-3 (пост-мерж гигиена → сид каталога → финал-пакет передачи); шаги 4-6 (provision/3 ключа/живой прогон) — держатель one-web, ГД готовит бриф.

**Шаг 1 — пост-мерж гигиена (частично закрыт):**
- (a) ✅ конфликт разрешён верно — на main `iva-role-qa` в `ROLE_LANES` (`[tacticum-core-base, iva-qa-autotest-base, iva-qa-mcp]`) и в `ROLE_PERSONA` (`qa`); per-role MCP-лейны на месте.
- (c) ✅ **73/73 зелёные на main** (реальный прогон verifier, [[verify-qa-postmerge-main]]): `test_iva_role_presets` 35/35, `test_manifest_schemas` 38/38, 0 skipped, docker не требовался.
- (b) ⚠️ **ЗАБЛОКИРОВАН — посылка кола не сходится с фактом.** **PR #119 в main НЕ смержен** (грепом: нет `#119`/`role-packs`/`single-axis`/`_GENERIC_ROLES` в истории main; нет `golden/iva-role-go`). Символа `_GENERIC_ROLES` в репе НЕТ — его вводит #119. → «дописать e2e-wiring qa в `_GENERIC_ROLES`» физически некуда, это **пост-#119** шаг. Наш `golden/iva-role-qa/` лежит orphan (подключится механизмом #119). **Конфликт с #119 НЕ закрыт мержем #133 — он живой и переехал на будущий мерж #119:** Глеб при вливании #119 обязан сохранить qa в `ROLE_LANES`, иначе снесёт роль. Условие передано ГД+Глебу сигналом.
- (d) ❓ **DB-catalog в CI на main** — статус не подтверждён (нет доступа к CI/`gh`); на президента/ГД.

**Побочно:** `docs/adr/0060-…md` в рабочем дереве main — untracked (канон ADR не закоммичен; doc-коммит за президентом, на отдачу QA не влияет).

**Шаг 2 — сид каталога: ✅ ПОДТВЕРЖДЁН (президент).** CI-сид на push в main отработал, `iva-role-qa`+`iva-qa-autotest-base`+`iva-qa-mcp` в прод-каталоге, роль устанавливаема. (Заодно закрывает вопрос (d) — DB-catalog в CI прошёл.)

**Шаг 3 — финал-пакет передачи QA-команде: ✅ ГОТОВО.** Один QA-facing `.md` — `12-Features/QA-профиль iva-role-qa — комплект передачи команде.md` (6 разделов: обзор · что нужно до старта · установка · первый прогон · что важно знать · открытые уточнения). Внешний регистр соблюдён (вычищены «президент»/роли/vault-ссылки; Diaret по провижну оставлен как реальный руководитель). Открытые уточнения в документе: env-форма helm-analyst Bearer; наличие workspace QA-команды.

**⚠️ Расхождение модели субагентов — разрешено.** Заметка приёмки [[verify-qa-kit-subagents]] писалась на kit-ветке с `model: opus`; **истина — `gpt-5.4`** (сверено на main `templates/iva-qa-autotest-base/manifest.yaml` стр.206/218/230 + CHANGELOG «opus был ошибкой, снят»). Комплект передачи собран под gpt-5.4; в заметку приёмки добавлен баннер-коррекция (opus→gpt-5.4, при живом прогоне ожидать gpt-5.4). Шаг 5 (quickstart) тоже под gpt-5.4.

**Шаг 4 — подготовка прод-сида: ✅ ГОТОВО** ([[prod-seed-iva-role-qa-prep]], explorer, read-only, к проду не подключался). Последовательность для VPS `159.194.224.59` `/opt/tacticum` БД `tacticum_catalog` готова до «нажать выполнить» (применяет ПРЕЗИДЕНТ, gated). Итоги:
- **(а) контент:** `iva-role-qa` 0.5.0 ✅ · `iva-qa-mcp` 0.1.0 ✅ (новый) · `iva-qa-autotest-base` 0.1.0 ⚠️ **R1** — контент менялся под ТОЙ ЖЕ 0.1.0; если прод уже сидил 0.1.0 со старым контентом → reject `version_already_exists_with_different_content`.
- **(в) dry-run: НЕТ** у `seed_community.py` (только позиционный аргумент templates). Безопасные обходы: локальный `check_profile_version_discipline.py --diff-against origin/main` (репо, не БД); **read-only SELECT прод-БД по `profile_versions`** (снимает R1 И вопрос CI-vs-VPS разом); либо seeder против одноразовой копии прод-БД.
- **(г) re-pin:** для сида НЕ нужен; только если есть существующие installations `iva-role-qa` → перевод на 0.5.0 отдельным действием.
- **Нюансы:** seeder сидит ВЕСЬ каталог `templates` (нельзя один профиль), commit по-профильно (не атомарно), two-pass (лейны до роли), docker-образ НЕ пересобирать.
- **Открытый вопрос президенту:** запустить read-only pre-flight SELECT прод-БД (снимет R1 + разведёт CI-каталог vs прод-VPS-каталог)?

**Шаг 5 — quickstart-мануал: ✅ ГОТОВО** (implementer, worktree `~/tacticum/tacticum-dev-qa-manual`, ветка `docs/iva-role-qa-manual` от main @ 20ff9b8). Файл `docs/user_manuals/iva-role-qa-profile-quickstart.md` по образцу analyst/go + приём негатив-теста секретов из web-brownfield. Факты сверены по манифестам (3 лейна, 4 MCP, 3 секрета, web-only), субагенты **gpt-5.4** везде. Не закоммичен/не запушен (git-шаг президента).

**⚠️⚠️ КЛЮЧЕВАЯ НАХОДКА (шаг 5) — Codex-primary vs Claude-native ядро.** Манифест `iva-qa-autotest-base` (сверено, стр.65-70): `claude-code: full` / **`codex: best-effort`** — «половина скиллов опирается на Claude-специфику (Task-субагенты, `Skill()`, PostToolUse-хуки), в codex не воспроизводится 1:1». Все 3 субагента + флагманские `write/batch/fix-autotest` — `supports: [claude-code]` (хотя `model: gpt-5.4`). При этом вся рамка направления и онбординг: «команда QA на **Codex**, ведущий путь Codex». → **Ядро профиля (генерация автотестов субагентами) first-class только на Claude Code; на Codex — best-effort.** Это противоречие рамки (Codex-primary) и реализации (Claude-native), НЕ решается на уровне lead-qa: задевает дизайн профиля (ADR lead-arch, target-CLI) + kit Жени (craft-субагенты) + фактический CLI команды QA. Эскалировано ГД. Варианты: (A) QA использует Claude Code для автотест-скиллов; (B) добавить Codex-поддержку субагентам/скиллам (dev-работа, kit-зависима); (C) принять best-effort на Codex, задокументировать. В quickstart и комплекте передачи оставлены пометки «уточнить у тимлида».

**РЕШЕНИЕ ПРЕЗИДЕНТА (2026-07-24): вариант B — всю Claude-специфику ядра переделать под Codex** (команда QA на Codex). Задача feature-размера, задевает автотест-лейн + 3 субагента + kit Жени. Запущена разведка-карта переделки [[explore-qa-codex-rework-map]] (что Claude-специфично → Codex-замена → что готово в kit; kit содержит codex-only профиль — вероятны готовые Codex-паттерны). После карты — план на апрув президента, потом implementer→verifier. Код до OK не трогаю.

**ТЕСТ-ГАЙД получен от президента** ([[profile-testing-guide]], 90-Materials) — канон-проход 5 уровней (0 матрица → 1 статика → 2 e2e-голдены → 3 живой агент на стенде 38.180.236.39 → 4 штатная выкатка), тот же, что для 10 ролей ADR. ⚠️ **Уровень 0/2 завязан на механизм #119** (`test_role_install_smoke.py`, `test_role_replacement_parity.py`, `_GENERIC_ROLES`, `test_install_flow_roles_generic[codex|claude-code-iva-role-qa]` — на main НЕТ, вводит #119). Полный проход (матрица + e2e-голдены обоих CLI) — после мержа #119. Уровень 1 (частично) и Уровень 3 (живой агент, one-web клон на стенде, codex headless `--dangerously-bypass-approvals-and-sandbox`, tac-токен из kbconsole.env) — доступны сейчас. Гайд подтверждает: механизм generic-roles = #119.

**Карта Codex-переделки готова** ([[explore-qa-codex-rework-map]]): 15 ingredient — 2 уже кросс (playwright/prepare-mr), 4 тривиально (secrets/gitignore/retro/craft-stack пути), 5 адаптация M (run-tests/jira/write/fix + субагенты), batch L; тела всего есть в kit Жени (нейтральные). **R1 РЕШЁН (президент 2026-07-24): native Codex `[agents]`-треды** (config уже `max_threads=4 max_depth=1`, паттерн репо go-лейн/iva-role-qa) — НЕ kit dispatch, НЕ инлайн. **Осталось развести до старта implementer:** (1) **R7 — схема `agent_spec`→Codex** (в манифесте `codex_target_path` только для skill_spec; субагентов на Codex доставлять нечем) → сигнал lead-arch (schema/ADR-0023). (2) **Доступ к тест-стенду `38.180.236.39`** для прототипа R1 (codebase-analyst как Codex-тред → write-autotest на 1 TC) и приёмки профиля 0 — у lead-qa нет, нужен от президента/Diaret. Механическая часть (тривиал + пути + run-tests) — можно начинать в worktree без рантайма/R7. Порядок: прототип R1 → тривиал → run-tests → субагенты → write/fix → jira → batch → приёмка профиля 0 на one-web (Уровень 3).

## Уточнения президента — 2026-07-24 (упрощают картину)
- **R7 (схема agent_spec→Codex) НЕ блокирует тестирование.** Для стенда дерево установки кладётся руками (Уровень 3 гайда) — 3 субагента в `.agents/` клона one-web, запуск Codex, БЕЗ каталожной доставки. Схема нужна только для финальной выкатки через каталог → делает lead-arch параллельно.
- **Codex-переделка = обычный цикл** ветка-от-main → переделать тела `templates/iva-qa-autotest-base/` + манифест (адаптация из kit) → новый PR.
- **Риск сида R1 (та же 0.1.0)** — не блокер сейчас, всплывает только на прод-сиде; учтём.
- **#119** — Глеб мержит скоро (сам закроется, ждём).
- **Тест-стенд `38.180.236.39`** — доступ У ПРЕЗИДЕНТА есть → Уровень 3 доступен.
- **Прод-выкатку делает СОЛОНКО**, не мы. П3 (прод-VPS доступ) + П6 (риск сида) уходят с тарелки lead-qa — мы проверяем на стенде и передаём готовую прод-сид-последовательность Солонко. → SELECT/сид на `159.194.224.59` больше не наша задача.
- **Итог — реально у нас:** (1) Codex-переделка (ветка→код→PR); (2) приёмка на стенде (Уровень 3); (3) ждём #119 (Уровень 2/матрица); (4) lead-arch — R7 параллельно.

## СТАРТ Codex-переделки — 2026-07-24 (президент дал GO)
Ветка `feat/qa-codex-rework` от main, worktree `~/tacticum/tacticum-dev-qa-codex`. **Чанк 1 (механика) — ✅ ГОТОВО** ([[impl-qa-codex-mechanical]], 3 коммита, не запушено, 12 файлов). Переделано: secrets/gitignore→кросс; run-tests→кросс+`codex_target_path` (заменены `Skill()`+`AskUserQuestion`, Bash 1:1); craft-stack→кросс+`codex_target_path` (рерайт `.claude/skills/craft-stack/`→`$CRAFT`, убран `${CLAUDE_PLUGIN_ROOT}`); retro→провайдер-нейтраль (нота памяти `.claude`↔`.codex`, интро нейтрально). **Версия 0.1.0→0.2.0 + CHANGELOG** (side-benefit: снимает риск сида R1 — новая версия сидится как `created`, без reject). Тесты: `test_manifest_schemas`+`test_iva_role_presets` = 73 passed (baseline 73), version-discipline OK. ⚠️ implementer флажнул трактовку retro: выровнял нейтральность, НЕ заменил тело целиком на kit `keep:retro` (полный kit тянет worktrees/plugin-cache/`$KEEP`/marketplace — чужеродно нашей модели лейна) — разумно, оставляем. Не трогал: субагенты/write/fix/batch/jira/rebuild-триггер (ждут прототип native [agents] + R7). Дальше: чанк 2 — прототип R1 (native [agents] на codebase-analyst→write-autotest 1 TC на стенде) → субагенты → write/fix → jira → batch → приёмка профиля 0.

## Codex-переделка ЗАВЕРШЕНА + PR СМЕРЖЕН + ДЕПЛОЙ-READY (2026-07-24)
Полная сводка решений/итогов — [[decisions-qa-codex-2026-07-24]]. Кратко: вся Codex-переделка (субагенты+write/fix/batch/jira→spawn_agent, ADR-0025/codex_body_path) сделана на `feat/qa-codex-rework` (12 коммитов, +оракул Глеба #2), Level 0+1 зелёно (288), контролёр GO, **Level-3 на живом codex-стенде PASS** (скиллы+агенты видны, spawn_agent работает), **PR СМЕРЖЕН президентом**.
**Деплой:** прод-каталог `tacticum_prod` — **R1 pre-flight ЧИСТО** (0 строк по 3 профилям, сид=created, re-pin не нужен). Деплой ручной по SSH (deployment_prod_catalog_mcp), rebuild не нужен (контент). Gated-сид ждёт команды президента/Солонко.
**Осталось (follow-up):** golden `codex.json` реген (docker); полный write-autotest e2e (one-web); wiki-sync мануала.

## Тех-долг (записан ГД 2026-07-24, НЕ блокер передачи) — живая синхронизация с upstream ivaqa/kit
Сейчас профиль = **разовый byte-copy снапшот** `kit-main.zip`; живого доступа к `git.hi-tech.org/ivaqa` НЕТ → со временем расходимся с upstream Жени. Нужно: доступ от Жени (PAT `read_repository` / деплой-ключ / зеркало) + модель обновления (зеркало-реимпорт / submodule-adapter / ре-снапшот по каналам kit main/stable/пин). Скоуп синка: craft-стек + 3 субагента + write/batch/fix + KMP-линия (iOS в kit нет). Вернуться, когда дойдёт приоритет. Детали: [[tekh-dolg-qa-zhivaia-sinkhronizatsiia-ivaqa-kit-git-lab-nash-qa-profil]].

## ⚠️ Уточнение стека/репо (Женя, 2026-07-24) + правка профиля
**Наш профиль по факту несёт стек `pytest-playwright-canvas` = KMP/canvas** (не selenium). Карта репо/стеков:
- **`one-web-kmp`** (`git.hi-tech.org/ivaqa/one-web-kmp`) — новый **KMP**-клиент iva-one, **pytest+playwright(canvas)** → **ЦЕЛЕВОЙ репо нашего профиля** ✅ (Женя подтвердил: описанный процесс применим на one-web-kmp+playwright).
- **`one-web`** — **Angular**-клиент iva-one, **pytest+selenium** — ДРУГОЙ стек, профиль его НЕ несёт.

**Правка (в работе, решение президента «исправить всё под текущий стек»):** метаданные профиля были мис-описаны как «pytest/Selenium для one-web», `stack.required:[one-web]` + висячие `_selenium-ref` — привести к **one-web-kmp / pytest-playwright-canvas / KMP**, почистить dead selenium-ссылки. Инструкция команды уже поправлена ([[iva-role-qa-ustanovka-i-rabota-dlia-komandy]]). Профиль → новая версия → ре-сид в прод.

## ТЕХ-ДОЛГ — мульти-репо/стек покрытие QA (записано 2026-07-24)
Профиль сейчас = **один стек (pytest-playwright-canvas / KMP / one-web-kmp)**. Расширения:
1. **one-web (Angular/selenium)** — добавить selenium-стек, если Жене нужен Angular-клиент сейчас (президент спросил Женю: «нужен one-web сейчас или гоняете на текущем стеке» — **ждём ответ**). Пока НЕ покрыт.
2. **`squish` (desktop, `git.hi-tech.org/autotest3/squish`)** — проиндексировать/поддержать (десктоп-стек).
3. **Мобильные репо** — Женя докинет позже.
Каждый стэк/репо = потенциально отдельный лейн/стек в craft. Вернуться по приоритету.

## ТЕХ-ДОЛГ #1 — полноценный Claude-провайдер (Claude-only реально работает) (записано 2026-07-24, решение президента)
Сейчас профиль **структурно** двух-провайдерный (Claude-тела лежат, `claude-code: full`), но **Claude-путь НЕ рабочий как есть и НЕ проверен**. Конкретный блокер (сверено по факту): у 3 субагентов `metadata.model: gpt-5.4`; рендерер (`_render_via_canonical`, `renderer.py`) бейкает эту модель во frontmatter установленного `.claude/agents/<id>.md` → на Claude Code субагенты пришпилены к `gpt-5.4` (GPT-модель, Claude Code её не гоняет) → ядро write/fix/batch (спавн 3 субагентов) на Claude ломается. Сделать:
- **Развести модель субагентов per-CLI:** claude-code → Claude-модель (opus/sonnet), codex → gpt-5.4. Сейчас одна `gpt-5.4` на оба.
- Сверить остальные Claude-примитивы в телах (Task/Skill()/пути `.claude/`) — что Claude-путь самодостаточен.
- **Прогнать end-to-end на тест-стенде** (Claude Code smoke: субагенты видны + Task-спавн + первый цикл write→run→fix), как гоняли Codex-smoke, ПРЕЖДЕ чем заявлять команде «Claude работает».
Пока честно: основной проверенный путь — **Codex**; Claude — структурно есть, но end-to-end не валидирован.

## ТЕХ-ДОЛГ #2 — kit-механика «Codex + Claude вместе» (провайдер-тиринг по подписке) (записано 2026-07-24, решение президента)
**Спек готов** ([[explore-qa-kit-capacity-model]]). Как устроено в kit (факт, file:line в заметке):
- **profile 0/1/2 = ярлык + карта `[roles.*].carrier`** (codex|claude|counter) в `answers.toml`. Роутинг определяет ТОЛЬКО carrier роли; профиль→carrier выставляет ЧЕЛОВЕК при установке, `render_base.py` не форсит (валидирует диапазоны). **Авто-проверки подписки НЕТ** — ручной выбор (5x/20x → profile 2); исчерпание квоты ловится реактивно (`degraded` → откат на Codex).
- **Claude-ревьюер = модуль `dispatch`** (`invoke_role.py`: `claude -p`/`codex exec` подпроцессом, встречная роль, чистая сессия, depth=1). profile 1 = reviewer на Claude (`audit:review`→`work-reviewer`, 6 фаз, анти-предвзятость); profile 2 = + fog_writer/scout на Claude.
- **Ортогональность:** наш native `spawn_agent` (тройка Codex в пайплайне, решение #4) и kit-`dispatch` (кросс-провайдерный reviewer) НЕ конфликтуют. Недостающее у нас: native Codex НЕ спавнит Claude → counter-review на Claude требует ровно dispatch. Сейчас мы = де-факто «зашитый profile 0» без механизма выбора (нет `answers.toml`/`[capacity]`/reviewer).
- **Связь с тех-долгом #1:** reviewer сильнее на Claude (у Codex-субагентов нет per-agent tool-allowlist; read-only sandbox режет сеть) → естественный якорь profile 1.
**План внедрения (порядок):** (1) **profile 1 первым** — минимальный ценный срез (Claude-reviewer через dispatch: забрать готовым тело `work-reviewer` + оркестратор `review` + пресеты; урезанный `claude -p` launcher; маппинг ролей→carrier); (2) переключатель профиля (config-поверхность: развилка b1 лёгкий / b2 полный kit-base); (3) profile 2. **Риск R-a:** profile 2 требует переключения носителя тройки — объём L; profile 1 этой проблемы не имеет.

Черновая суть тиров (как описал Женя):
- **profile 0 — codex-only (дефолт):** все роли на Codex, без Claude.
- **profile 1 — codex + Claude-ревьюер (опционально):** Claude только как встречная reviewer-роль, при минимальной личной подписке Claude.
- **profile 2 — codex + Claude (там где Claude сильнее):** при личной подписке 5x/20x.
Механизм в kit (забрать/адаптировать): `[capacity].profile` 0/1/2 в `base/templates/answers.template.toml`, резолв `[roles.*].carrier` (носитель роли), модуль `dispatch` (`invoke_role.py` → `codex exec`/`claude -p`), деградация carrier→Codex + событие `degraded`, `auto_counter_review`. Карта: [[explore-qa-codex-rework-map]] §«3 профиля ёмкости». Вернуться по приоритету.

## Из созвона руководителя 24.07 (кол ГД 11:35, на ознакомление) — что влияет
- **Руководитель ЯВНО: «QA-команда на Codex + GPT-5.6, всё под Codex»** — усиливает нашу находку target-CLI: вариант A (QA на Claude Code) конфликтует с руководителем → вес в B/C; A/B/C финально решает президент (lead-arch разбирает). Мы уже идём B (Codex-переделка). ⚠️ **Модель:** репо тирит gpt-5.x по сложности (gpt-5.4 частый для coder/test-runner — наш случай; 5.5 оркестраторы; 5.6 редкий). «5.6» руководителя ≈ «линия Codex GPT-5», не точный бамп. Держим gpt-5.4 для субагентов (консистентно с coder-тиром), если президент не скажет бампнуть — тривиально меняется.
- **profile-testing-guide Глеба = ОФИЦИАЛЬНЫЙ канон приёмки** (уровни 0-4), им тестировались 10 ролей. Прогон iva-role-qa 1→2→3. Уровень 3 (живой пилот на стенде 38.180.236.39) — можно ДО передачи QA-команде, не завися от one-web.
- **3 замечания Глеба (в план приёмки/правок):** (a) маркер `instruction_pack` — коллизия при composе, покажет smoke (Уровень 1); (b) связка команда→существующий агент — добавить пары субагентов в оракул (тест-матрица); (c) **СУЗИТЬ `iva-qa-mcp` allowed_tools до канарейки** (иначе роль видит всю поверхность helm-analyst) — сверить, что read-срез уже сужен, при необходимости ужать.
- **Целевой контакт передачи — БРЕЙКИН (Никита)** (плотно пообщаться), не только Женя.
- **Порядок подтверждён президентом:** сервер tacticum-dev (сид) → тест по гайду Глеба → QA-команда.
- **Стратегически (на будущее, НЕ сейчас):** руководитель хочет ЕДИНЫЙ профиль (аналитик) с режимами внутри — QA может стать режимом/паком, а не отдельным профилем. Сейчас не переделывать, держать в уме.

**⚠️ Доступ к прод-VPS — у lead-qa НЕТ.** Сервер каталога `159.194.224.59` (tacticum_dev) НЕ заведён в ssh-manager (есть только zu_demo/helm/gateway/adp_emb). Read-only pre-flight SELECT прод-БД (снятие R1 + CI-vs-VPS) сам выполнить не могу — команды готовы ([[prod-seed-iva-role-qa-prep]] §в п.2), отданы президенту/Diaret; либо завести сервер в ssh-manager (креды от Diaret).

## Канон приёмки роли — гайд Глеба (созвон 2026-07-24) [[profile-testing-guide-1]]
Глеб дал `profile-testing-guide.md` — **канонический процесс приёмки роли перед выкаткой (уровни 0-4)**, им тестировались все 10 ролей матрицы ADR-0059 (вкл. живые пилоты java/kmp/internal). Наш порядок отдачи QA ложится на него:
- **Ур.0 — матрица:** роль подключить в 4 места `apps/backend/tests/` (`test_iva_role_presets` ROLE_LANES+ROLE_PERSONA ✅ уже; `test_role_install_smoke` ROLES; `test_role_replacement_parity` — у qa предшественника нет, пары не будет; `e2e_install/test_install_flow` `_GENERIC_ROLES` — **пост-#119**, символ вводит #119).
- **Ур.1 — статика** (секунды): preset+smoke+parity + `check_profile_version_discipline.py` + `check_mirror_sync.py`.
- **Ур.2 — e2e install** (нужен Docker/Postgres): сид→provision→pull→раскладка→golden; каждый параметр отдельным процессом (asyncpg-флак).
- **Ур.3 — живой пилот ДО мержа/сида на тестовом стенде `38.180.236.39`** (codex/node/serena-LSP + VPN до контура ИВА): смок (whoami через tacticum-mcp) → read-only задача → полный цикл (write-autotest/run-tests/fix-failed-test + playwright + write-канал в ТЕСТОВУЮ issue). **Опция прогнать роль ДО передачи QA-команде, не завися от one-web.**
- **Ур.4 — выкатка:** сид на dev.tacticum.dev + **quickstart обязателен** (образец `iva-role-java-profile-quickstart.md`) + канарейка→когорта (`role-migration-runbook.md`).
- **3 замечания по iva-role-qa:** маркер `instruction_pack` (коллизия при composе лейнов — smoke покажет); связка «команда→существующий агент» (добавить пары субагентов в оракул); **сузить `iva-qa-mcp` allowed_tools до канарейки** (иначе роль видит всю поверхность helm-analyst).

**Статус на 2026-07-24:** #119 (матрица ролей Глеба) ещё НЕ в main (пик — #133 наш QA-профиль). Прод-сид готовится ([[prod-seed-iva-role-qa-prep]]). Полный e2e-wiring (`_GENERIC_ROLES`) + golden — после мержа #119. Разбор созвона: [[call-2026-07-24-10-30-dizain-sistema-tokeny-rezhimy-edinyi-profil-change-management-qa-vykatka]].

## Тех-долг (2026-07-24)
- **Живая синхронизация upstream:** подтягивать/обновлять инфу из `ivaqa/kit` GitLab в наш профиль (сейчас — разовый снапшот, живого доступа нет; расходимся с upstream). Нужен доступ от Жени (PAT/деплой-ключ/зеркало) + модель обновления. Детали: [[tekh-dolg-qa-zhivaia-sinkhronizatsiia-ivaqa-kit-git-lab-nash-qa-profil]]. KMP — тестируется как web (Playwright+canvas); iOS в kit нет.
