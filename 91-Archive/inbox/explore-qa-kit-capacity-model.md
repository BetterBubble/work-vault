---
title: explore-qa-kit-capacity-model
type: report
permalink: tacticum/00-board/explore-qa-kit-capacity-model-1
status: draft
role: explorer
topic: qa-kit-capacity-model
source_kit: kit-main snapshot (marketplace v1.5.1)
source_profile: tacticum-dev-qa-codex @feat/qa-codex-rework
date: 2026-07-24
tags:
- explore
- qa
- kit
- capacity
- dispatch
- counter-review
- lead-qa
archived-at: 2026-07-31 17:27
---

# Модель ёмкости kit (Codex+Claude тиринг) — точный разбор + план внедрения в наш QA-профиль

Read-only разведка. Источники: **K** = снапшот kit `…/scratchpad/kit/kit-main/` · **P** = наш профиль в worktree `feat/qa-codex-rework` (`templates/iva-qa-autotest-base` + `templates/iva-role-qa`). Углубляет [[explore-qa-codex-rework-map]] (§4, R1, R5-R7).

## TL;DR

- Модель ёмкости kit держится на **одном поле-переключателе** `[capacity].profile` (0/1/2) в `.kit/answers.toml` + **карте `[roles.*].carrier`** (codex|claude|counter). Профиль — **информативный ярлык + строка в схема-доке**; фактический роутинг читает ТОЛЬКО `carrier` каждой роли. Соответствие profile→carrier выставляет **человек на установке** (скилл `base:install`), рендер его НЕ форсит (валидирует лишь диапазоны).
- **Claude включается через модуль `dispatch`** — обёртка `invoke_role.py`, которая собирает `claude -p …` / `codex exec …` как подпроцесс. Это механизм **встречной роли** (другой провайдер / чистая сессия, depth=1) — прежде всего reviewer. НЕ внутрипайплайновый спавн.
- Наш профиль внутрипайплайновую тройку (analyst/dom-explorer/code-writer) уже гоняет **native Codex `spawn_agent`** (решение президента #4), а НЕ через dispatch. **Эти два механизма ортогональны** и не конфликтуют: spawn_agent — тройка одного провайдера в пайплайне; dispatch — кросс-провайдерный reviewer. Тиринг ёмкости ложится именно на reviewer-роль.
- **Подписка не детектится автоматически.** Профиль — ручной выбор человека (5x/20x → profile 2; минимальная → profile 1; нет → profile 0). Исчерпание квоты Claude ловится в рантайме реактивно: `claude -p` упал → откат на Codex + событие `degraded`.
- Ключевая точка боли (тех-долг #1): у Codex-субагентов **нет per-agent tool-allowlist** — reviewer-readonly радиус на Codex деградирует (sandbox read-only режет сеть → Ф1 сверки с TMS недоступна). На Claude тот же пресет ложится честно. Поэтому reviewer — ровно та роль, где Claude сильнее, и почему profile 1 сажает reviewer на Claude.

---

## 1. `[capacity].profile` 0/1/2 + рендер/валидация

**Носитель профиля** — `K:base/templates/answers.template.toml`:
- `[capacity].profile = 0` — L21-27. Комментарий-канон (L22-26): **0** Codex-only (все роли на Codex); **1** CC-ограниченный (Claude только встречный reviewer); **2** CC-полный (fog_writer+scout = Claude, pipeline_writer = Codex, reviewer = counter — противоположный писателю провайдер).
- `[strictness].auto_counter_review = "strict"` — L29-33. Ось строгости автозаказа встречного ревью: `strict` (заявка стоп, ждёт человека) | `auto` (синхронно сразу).
- 4 роли `[roles.*]` — каждая с `carrier` / `tier` / `effort` / `permissions`:
  - `pipeline_writer` L42-46 — carrier=codex, permissions=`writer-full`;
  - `fog_writer` L48-52 — carrier=codex, `writer-full`;
  - `scout` L54-58 — carrier=codex, `scout-recon`;
  - `reviewer` L60-64 — carrier=codex, `reviewer-readonly`.
  - `carrier`: `codex` | `claude` | `counter` (counter = провайдер, противоположный писателю данной задачи; **только** для reviewer). `tier`=id модели, `effort`=reasoning-уровень; пусто = дефолт платформы.

**Профиль → дефолты carrier выставляет человек на установке**, НЕ скрипт — `K:base/skills/install/SKILL.md` §2а L60-68:
- profile 0 → у всех `carrier="codex"` (заготовка уже такая);
- profile 1 → `reviewer.carrier="claude"`, остальные codex;
- profile 2 → `fog_writer.carrier=claude`, `scout.carrier=claude`, `pipeline_writer.carrier=codex`, `reviewer.carrier="counter"`.
- Пояснение человеку (L58): «смена подписки потом = правка одного поля `profile` + шаг 2а + перерендер».

**Валидация/рендер** — `K:base/scripts/render_base.py`, функция `validate()` L79-142:
- `profile in (0,1,2)` — L114-116 (иначе ошибка);
- `strictness.*` ∈ `MODES {strict,auto}` — L118-123; наличие `auto_counter_review` обязательно L119-120;
- `carrier ∈ CARRIERS {codex,claude,counter}` — L131-133; **counter допустим только у reviewer** — L134-135;
- каждая из 4 ролей `ROLES` (L42-47) обязана присутствовать с `tier`/`effort` (строки) и непустым `permissions` — L126-141.
- ⚠️ **Важное: валидатор НЕ проверяет соответствие profile↔carrier.** Можно поставить profile=0, а carrier reviewer=claude — рендер пропустит. Профиль — ярлык для человека и строка в доке; роутинг реально определяет только `carrier`.

**Рендер-выход** — `render_schema()` L166-184 пишет `.kit/schema.md` (генерат, руками не править) из `K:base/templates/schema.template.md`: подставляет `{{capacity.profile}}` (L39 шаблона), `{{roles_table}}` (`roles_table()` L145-154), `{{strictness_table}}` (L157-163). Правило деградации и fallback обязательного делегирования зафиксированы текстом в schema.template.md L44-63.

**Правило деградации** (канон, `K:dispatch/references/role-rule.md` L30-32 + schema.template.md L45-48): базовый носитель КАЖДОЙ роли — Codex; при недоступности Claude (квота/сбой) роль тихо откатывается на Codex, процесс не встаёт; откат ОБЯЗАН оставить событие `degraded` в следе.

## 2. Модуль `dispatch` — провайдер-нейтральная обёртка спавна

`K:dispatch/scripts/invoke_role.py` — единственный механизм кросс-вызова роли (решение 015 карты agent-env, вариант C). Руками `claude -p`/`codex exec` не собирает никто.

**Резолв роль→носитель** — `main()` L464-470: читает `[roles.<role>]` из answers → `resolve_carrier()` L115-120 (`counter` → противоположный `writer_provider`; иначе как есть). `writer_provider` из `--writer-provider` или best-effort автодетект по env `detect_writer_provider()` L104-112.

**Сборка команды** — `build_command()` L219-242:
- claude (L225-235): `claude -p <prompt> --output-format json --append-system-prompt <DEPTH_GUARD>` + `claude_flags()` + `--agent <bare>` (L230, неймспейс навешивает платформа) + `--model <tier>` (L232) + `--effort` (L234);
- codex (L236-242): `codex exec <guard+prompt> -C <root>` + `codex_flags()` + `-m <tier>` (L241).

**Пресеты → флаги** — `K:dispatch/presets.toml` (конфиг-первичный источник радиуса, решение 015 п.5):
- `writer-full` L19-30 (repo scope; claude acceptEdits + Read/Edit/Write/Bash; codex workspace-write/never);
- `scout-recon` L32-46 (task-home; claude default + Write(.tasks/**) + deny Edit; codex read-only/on-request);
- `reviewer-readonly` L48-72 (task-home; claude default + deny Edit,Write; codex read-only/never). **Асимметрия платформ L61-72:** Codex sandbox=read-only режет И сеть → Ф1 ревью (чтение ТК из сетевого TMS) структурно недоступна → ревью деградирует в статическое; Claude-проекция сеть не режет — там Ф1 живёт.
- `claude_flags()` L164-178 предпочитает сгенерённый `.kit/presets/claude/<preset>.settings.json` (флаг `--settings`), иначе инлайн `--permission-mode/--allowedTools/--disallowedTools`. `codex_flags()` L181-189: `-s <sandbox> -c approval_policy=…`.
- Генерат конфигов — `K:dispatch/scripts/render_presets.py` пишет `.kit/presets/{claude,codex}/*` из канона (закрывает «дыру схемы base»).

**depth=1** — env `KIT_DISPATCH_DEPTH` инкрементится в `run_carrier()` L315; на входе `main()` L432-443 если depth≥1 → отказ (встречная роль не заказывает свои кросс-вызовы) + событие failed. Промпт-страж `DEPTH_GUARD` L194-201 (запрет собственных кросс-вызовов/субагента-ревьюера; итоговый вывод роли = вердикт-файл; машинное резюме `DISPATCH_SUMMARY: {...}`).

**События** `.tasks/events.jsonl` (контракт base v1) — `append_event()` L141-151, атомарный O_APPEND: `requested` (L496-499), `started` (L344-350), `finished` (L402-407, несёт verdict result/findings/session), `degraded` (L368-371), `failed` (L357-360/376-379/385-387).

**Гейт строгости** — `execute` vs стоп: L478-479 читает `auto_counter_review`; L503-506 — strict → печатает путь заявки и СТОП (исполнение по отмашке `--execute <request.json>`); auto → `execute()` синхронно L508.

**Реактивная деградация CC→Codex** — `execute()` L363-380: если `provider=claude` и `returncode!=0` → событие `degraded` + повторный `do_run("codex")`, `prov_final=codex`. Таймаут → `failed`, заявка остаётся для идемпотентного ретрая.

Фасад-скилл `K:dispatch/skills/invoke/SKILL.md` — тонкая обёртка (не дублирует логику); трактует исход (strict/auto/degraded/failed). Правило ролей — `K:dispatch/references/role-rule.md`.

## 3. Claude-ревьюер (counter-review): profile 1 vs profile 2

**Оркестратор** `K:audit/skills/review/SKILL.md` — «конкретный механизм роли `reviewer` схемы base». Политику берёт из answers (L26-34): кто ревьюер (`[roles.reviewer].carrier`), когда автозаказ (`[strictness].auto_counter_review`). Спавнит субагента `audit:work-reviewer` с **чистым контекстом**; главный инвариант входа — **анти-предвзятость** (L53-58): промпт несёт ТОЛЬКО цель (unstaged | A..B), TC id и путь вердикта; никакого пересказа, что делал исполнитель. Ревьюер только находит (read-only вердикт), фиксы вносит основной агент по одобрению человека (L81-90).

**Субагент** `K:audit/agents/work-reviewer.md` — процедура роли `reviewer` (permissions=`reviewer-readonly`), 6 фаз (L52-161): Ф0 инвентарь дифа/следа; Ф1 сверка с ТК из TMS (при бессетевом Codex-sandbox → `skipped(TMS unreachable)`, НЕ blocked, L74-79); Ф2 план↔код; Ф3 конвенции/качество; Ф4 прогоны ×2 + canary соседей; Ф5 гигиена; Ф6 вердикт (классы находок корректность>конвенция>качество, файл:строка). Пишет ТОЛЬКО в `.tasks/` (вердикт).

**Что ревьюит / когда / чистая сессия / depth:**
- **что** — дифф исполнителя (write/fix/update) против первоисточников: ТК из TMS, план исполнителя, конвенции стека craft, + обязательные прогоны;
- **когда** — по автозаказу писателя; gate `auto_counter_review` (strict = после отмашки человека);
- **чистая сессия** — субагент даёт свежий контекст даже внутри одного провайдера (L30-34 SKILL); при кросс-провайдерности запуск идёт через `dispatch:invoke` (`claude -p --agent audit:work-reviewer`);
- **depth** — кросс-вызов depth=1 (ревьюер не заказывает своих кросс-вызовов, не спавнит своего суб-ревьюера).

**profile 1 vs profile 2:**
- **profile 1** (минимальная подписка): Claude добавляется ТОЛЬКО как reviewer (`reviewer.carrier=claude`). Единственная роль, где кросс-провайдерность незаменима и «самая ценная на токен». Пайплайн (writer/scout) остаётся на Codex.
- **profile 2** (личная 5x/20x): дополнительно `fog_writer.carrier=claude` (задачи без готового паттерна) и `scout.carrier=claude` (разведка живого UI/кодбазы/фактов); `pipeline_writer` (задачи по готовым паттернам) остаётся на Codex; `reviewer.carrier=counter` — всегда противоположный писателю провайдер (если писал Claude — ревьюит Codex, и наоборот). Т.е. на Claude уходят роли, где он сильнее: рассуждение без шаблона + разведка.

## 4. Как это ложится на НАШУ реализацию (native Codex `[agents]` + spawn_agent)

**Текущее состояние P** (`feat/qa-codex-rework`, тесты 288/0/0, стенд Level-3 PASS):
- 3 субагента как agent_spec с ДВОЙНОЙ доставкой: claude (`.claude/agents/<id>.md`, model gpt-5.4) + codex (`codex_body_path` → `.codex/agents/<id>.toml`, ADR-0025). Claude-тела лежат (`P:iva-qa-autotest-base/ingredients/agents/*.md`), Codex-тела рядом (`ingredients/agents-codex/*.toml`). Пример `codebase-analyst.toml`: keys `model=gpt-5.4`, `model_reasoning_effort=medium`, `sandbox_mode=read-only`, `developer_instructions` (verbatim из Claude-тела).
- 4 оркестратора write/fix/batch/jira: дивергентные codex-тела через `codex_body_path` → `.agents/skills/` (R7-FLAG), делегирование `spawn_agent(agent_type,task_name,message,model)`+`wait_agent`+`close_agent` вместо Claude `Task`; batch — веер до `max_threads=4`.
- `[agents] max_threads=4 max_depth=1` — доставляется как repo_config `P:iva-role-qa/ingredients/repo-configs/codex/config.toml.template` L4-6.
- **НЕТ:** `.kit/answers.toml`, `[capacity]`, `[roles.*]`, `[strictness]`, dispatch, counter-review, reviewer-роли, `presets.toml`. Тиринга ёмкости нет — профиль де-факто «зашитый profile 0», но без самого механизма выбора.

**dispatch vs native — совместимо ли:**
- **Ортогональны, не конкурируют.** native `spawn_agent` = внутрипайплайновая тройка одного провайдера (Codex→Codex, в сессии, потолок max_threads/max_depth). dispatch = кросс-провайдерная/чистосессионная встречная роль (подпроцесс `claude -p`/`codex exec`, depth=1). Президент (решение #4, [[decisions-qa-codex-2026-07-24]]) выбрал native `[agents]` для тройки — dispatch для неё НЕ берём.
- **Но для profile 1/2 нужен кросс-провайдерный запуск Claude-ревьюера, которого native Codex не умеет:** `spawn_agent` спавнит только Codex-агентов (`.codex/agents/*.toml`), запустить Claude из Codex-сессии он не может. Значит counter-review на Claude требует ровно dispatch-механики (`claude -p --agent … --settings <preset>`). Это и есть недостающий кусок.
- **Что менять:** ввести (а) минимальную конфиг-поверхность выбора профиля; (б) reviewer как роль на Claude-носителе; (в) launcher кросс-провайдерного ревью. Тройку write/fix/batch НЕ трогать — она уже native и профилю 0 соответствует.

**Связь с тех-долгом #1 (per-CLI модель субагентов):**
- Codex-субагенты **не имеют per-agent tool-allowlist** — зафиксировано FLAG'ом в `P:…/agents-codex/codebase-analyst.toml` (шапка: «Claude `tools:` restriction has no exact Codex-native equivalent … Mapped best-effort onto sandbox_mode»). ADR-0025 optional keys: model, model_reasoning_effort, sandbox_mode, mcp_servers, skills.config.
- Это ровно причина, по которой **reviewer сильнее на Claude:** kit `reviewer-readonly` на Codex (sandbox=read-only) режет сеть → Ф1 сверки с TMS недоступна, ревью деградирует в статическое (`K:dispatch/presets.toml` L61-72). На Claude тот же радиус (deny Edit/Write, сеть жива) даёт полное ревью. → тиринг ёмкости и тех-долг #1 сходятся в одной точке: reviewer-роль. Это и есть естественный якорь profile 1.

## 5. Подписки/ключи — детект или ручной выбор?

**Ручной выбор человеком, автопроверки НЕТ.**
- Профиль выбирается на установке живым ответом человека — `K:base/skills/install/SKILL.md` L51-58 («есть ли у проекта Claude и в каком объёме»: 0 нет / 1 дорогой-лимитный / 2 без жёстких лимитов). Kit НЕ пробит ключи/квоту/подписку.
- `detect_writer_provider()` (`K:invoke_role.py` L104-112) — best-effort по env, но это резолв провайдера ВЫЗЫВАЮЩЕЙ стороны для `carrier=counter`, НЕ проверка доступности Claude.
- Доступность Claude проверяется **реактивно в рантайме**: `claude -p` упал (квота/нет CLI/сбой) → `execute()` L365-380 пишет `degraded` и откатывается на Codex. Т.е. «личная подписка 5x/20x» = человек ставит profile 2; исчерпание лимита посреди работы = тихий откат на Codex без остановки. Смена подписки = правка поля `profile` + carrier'ов + перерендер.

---

## План внедрения в наш профиль

Цель: profile 0 (дефолт, codex-only — текущее) / profile 1 (codex + Claude-ревьюер, опц., минимальная подписка) / profile 2 (codex + Claude там, где сильнее — личная 5x/20x). Ставить задачу implementer'у по этому.

### A. Забрать из kit ГОТОВЫМ (перенос тел, санитизация путей — как уже делали с craft)
1. **Тело ревьюера** `K:audit/agents/work-reviewer.md` → новый agent_spec `work-reviewer` (Claude-носитель; `.claude/agents/work-reviewer.md`). Тело нейтрально, `$CRAFT`/`$BASE`/`$TMS`-ссылки разрешить в наши пути (как в craft-stack). Это профиль-специфика «Claude там, где сильнее».
2. **Оркестратор** `K:audit/skills/review/SKILL.md` → skill_spec `review` (анти-предвзятость, разбор вердикта, фиксы по одобрению). Claude-тело; при желании — codex-тело позже.
3. **Канон пресетов** `K:dispatch/presets.toml` (радиус `reviewer-readonly` + `scout-recon`/`writer-full`) — как справочник радиусов; для нас критичен `reviewer-readonly`.
4. **Правило деградации + fallback** (текст `K:base/…/schema.template.md` L44-63, `role-rule.md`) — как норматив в README/квикстарт профиля.

### B. АДАПТИРОВАТЬ (тело есть, механизм под нас)
1. **Конфиг-поверхность выбора профиля.** У нас нет `.kit/answers.toml`. Два варианта (развилка тимлиду):
   - (b1) лёгкий: ввести профиль-переключатель как параметр установки нашего лейна (profiles trial/full в manifest уже есть — добавить ось «capacity 0/1/2»), маппящий carrier reviewer'а. Без тащить весь kit `base`.
   - (b2) тяжёлый: перенести kit `base` (answers.toml + render_base.py) целиком — даёт полную kit-совместимость, но конфликтует с нашей manifest/ingredient-моделью и ADR-0023. **Рекомендация: b1** (минимализм; см. риск R-b).
2. **Launcher кросс-провайдерного ревью.** native Codex не спавнит Claude → нужен подпроцесс-запуск `claude -p --agent work-reviewer --settings <reviewer-readonly.settings.json>` с чистой сессией. Либо:
   - (c1) забрать урезанный `invoke_role.py` (только ветка claude + reviewer-readonly + события + деградация CC→Codex) — готовый, проверенный;
   - (c2) написать тонкий launcher под наш профиль. **Рекомендация: c1 урезанный** (деградация и depth=1 уже решены в kit).
3. **Пресет `reviewer-readonly` под Claude** — сгенерить `.claude/settings.json`-форму (allow Read/Glob/Grep/Bash, deny Edit/Write) как ingredient repo_config; аналог `render_presets.py`.

### C. ПИСАТЬ ЗАНОВО
1. **Маппинг наших ролей на carrier по профилям.** У нас тройка = analyst(scout)/dom-explorer(scout)/code-writer(pipeline_writer), оркестратор = writer. Расписать:
   - profile 0: всё codex (текущее, ноль изменений);
   - profile 1: + reviewer на Claude (новый agent_spec + launcher);
   - profile 2: fog-класс задач и scout-разведка → Claude. ⚠️ У нас тройка жёстко на Codex через spawn_agent/`.codex/agents/*.toml`. Перевод scout/fog на Claude в profile 2 = вторые Claude-тела уже лежат (`.claude/agents/*.md`) — нужен только переключатель носителя оркестратором. Механизм выбора носителя субагента по профилю — писать.
2. **Событийный след ревью** — если берём урезанный invoke_role, `.tasks/events.jsonl` уже пишется; иначе — мини-логгер.

### D. Порядок работ
1. **profile 1 первым** (минимальный ценный срез): agent_spec `work-reviewer` (Claude) + skill_spec `review` + launcher `claude -p` (урезанный invoke_role, c1) + пресет reviewer-readonly. Приёмка: на диффе одного TC заказать встречное ревью, получить вердикт-файл.
2. **Переключатель профиля** (b1) — ось capacity в manifest/установке, дефолт 0.
3. **profile 2** — маппинг scout/fog на Claude-носитель (переключение носителя тройки оркестратором); reviewer=counter.
4. **Деградация + strictness gate** — перенести strict-дефолт (заявка ждёт отмашки) + откат CC→Codex.
5. Прогон на стенде: profile 0 (регресс — ничего не сломалось), profile 1 (ревью на Claude), деградация (Claude недоступен → codex + degraded).

### E. Риски / развилки
- **R-a (носитель тройки в profile 2).** Наши codex-тела write/fix/batch хардкодят `spawn_agent(...model="gpt-5.4")` (Codex-агенты). Для profile 2 (scout/fog→Claude) оркестратор должен уметь выбирать носитель субагента по профилю — сейчас носитель зашит в дивергентное codex-тело. Возможно profile 2 требует третьей ветки тел оркестратора, либо параметризации носителя. **Оценить объём — может быть L.** profile 1 этой проблемы НЕ имеет (reviewer — отдельная роль, не тройка).
- **R-b (конфиг-поверхность).** Тащить kit `base`/answers.toml (b2) даёт совместимость с апстримом (см. тех-долг синка ivaqa/kit), но ломает нашу manifest-модель. Лёгкий переключатель (b1) проще, но расходится с kit-каноном ещё сильнее. Решение тимлида/lead-arch.
- **R-c (per-agent tools на Codex, тех-долг #1).** reviewer-readonly на Codex деградирует (нет сети в read-only sandbox → Ф1 skipped). Значит counter-review за Codex-писателя, если ревьюер тоже Codex (profile 0 «ревью внутри провайдера»), даёт лишь статическое ревью. Полное ревью с TMS-сверкой доступно только на Claude → profile 1/2. Зафиксировать в доке ограничение profile 0.
- **R-d (ADR).** Нужен ли ADR на «ось capacity» в схеме манифеста (по аналогии с ADR-0025 codex_body_path / R7). Сверить с lead-arch.
- **R-e (strictness).** Наш профиль не имеет оси строгости; вводя автозаказ ревью — принести дефолт `strict` (заявка ждёт человека), не `auto`.

---

## Ссылки на факты (файл:строка)

- Профиль/роли/строгость: `K:base/templates/answers.template.toml` L21-27 (capacity), L29-33 (strictness), L42-64 (4 роли).
- Валидация: `K:base/scripts/render_base.py` L114-116 (profile), L118-123 (strictness), L131-135 (carrier/counter), L126-141 (роли); рендер L166-184.
- Профиль→carrier на установке: `K:base/skills/install/SKILL.md` L51-68.
- Схема-док: `K:base/templates/schema.template.md` L39 (profile), L44-63 (деградация+fallback).
- dispatch: `K:dispatch/scripts/invoke_role.py` — резолв L464-470/L115-120, команды L219-242, пресеты→флаги L164-189, depth L315/L432-443, события L141-151, gate L478-479/L503-506, деградация L363-380. Пресеты `K:dispatch/presets.toml` L19-72 (reviewer асимметрия L61-72). Генерат `K:dispatch/scripts/render_presets.py`. Фасад `K:dispatch/skills/invoke/SKILL.md`. Правило `K:dispatch/references/role-rule.md` L23-64.
- Counter-review: `K:audit/skills/review/SKILL.md` (оркестратор, анти-предвзятость L53-58); `K:audit/agents/work-reviewer.md` (6 фаз L52-161, Ф1 skip L74-79).
- Наш профиль: `P:iva-qa-autotest-base/manifest.yaml` (двойная доставка агентов L239-276, codex_body_path оркестраторов L106-184); `P:…/ingredients/agents-codex/codebase-analyst.toml` (ADR-0025 keys, tools-gap FLAG); `P:iva-role-qa/ingredients/repo-configs/codex/config.toml.template` L4-6 (`[agents] max_threads=4 max_depth=1`).
- Решения президента: [[decisions-qa-codex-2026-07-24]] (#4 native [agents], R7 закрыт ADR-0025). Тех-долг синка апстрима: [[Тех-долг QA- живая синхронизация ivaqa-kit GitLab → наш QA-профиль]].
- Снапшот kit: `/private/tmp/claude-501/-Users-bubblemac-tacticum/47359696-4a55-470f-bf12-3b80ea797f36/scratchpad/kit/kit-main/`.
</content>
</invoke>