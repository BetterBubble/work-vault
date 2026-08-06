---
status: draft
role: implementer
task: ТЗ#3 / US#4 E-remainder — kmp brd (diverged merge) + kmp start-task
for: lead-fr → ГД
repo: tacticum-dev
branch: feat/us4-passE-remainder (от feat/us4-passE-kmp-pintests)
worktree: /Users/bubblemac/tacticum-worktrees/us4-passE-remainder
base: 8b0e3c9 (pin/tests, 0.5.1) поверх origin/main 5884bcd
head: c39ed5e
version: 0.5.1 → 0.5.2
permalink: tacticum/00-board/impl-us4-eremainder-1
generated: 2026-07-24
archived-at: 2026-08-03 11:16
---

# E-remainder: kmp brd (diverged merge) + kmp start-task гейт

Стек на E: база `feat/us4-passE-kmp-pintests` (pin/tests, 1 коммит 0.5.1 поверх main).
Создан worktree `feat/us4-passE-remainder`, добавлен 1 коммит `c39ed5e` (0.5.2).
Правки — ТОЛЬКО 2 файла контента (brd + start-task) + manifest + CHANGELOG. НЕ push.

## Часть 1: kmp brd-authoring — diverged merge (аккуратно, по карте+ACK ГД)

Файл: `templates/iva-kmp-brownfield/ingredients/skills/brd-authoring/SKILL.md`.
Внесены 5 частей дельты К-1/К-5/К-2 в НЕконфликтующие места, +33 строки нетто, только
добавления (одна bullet-строка расширена дописыванием v2-локации).

1. **Детекция маркера `fr_skeleton` (преамбула).** Новый абзац сразу под заголовком
   блока «FR на входе», над существующим текстом. Говорит: сначала прочитай маркер
   версии → `fr_skeleton: 2` = формальные требования в Части 1 (§1.4 FT-n / §1.5 UC-n
   + проектные §1.6–1.8); маркер отсутствует = плоское чтение из `fr.md` как раньше;
   не угадывать, битый маркер → v1.

2. **К-1 локация FT/UC — сшивка в существующий bullet (стр.18-19).** К bullet о
   дословном наследовании `UC-n`/`FT-n` ДОПИСАНА локация для v2: `FT-n` из §1.4,
   `UC-n` из §1.5 Части 1 (не из Приложения); маркер отсутствует → из `fr.md` как
   раньше. Оригинальный текст bullet (verbatim, не renumber, GIVEN/WHEN/THEN)
   сохранён без изменений — только добавлена v2-ветка.

3. **К-2 серии CT/DM/EV (v2-only).** Новый подблок ПОСЛЕ grep self-check (в пустом,
   неконфликтующем месте перед `## Document structure`): на v2 регистрировать и
   передавать вниз `CT-n`(§1.6)/`DM-n`(§1.7)/`EV-n`(§1.8) по стабильному ID; BRD
   только ссылается (реализует pin-authoring, покрывают TESTS); в v1 серий нет — шаг
   пропускается.

4. **Rules-дельта.** После правила «Write BRD in Russian» добавлены 2 правила:
   читать FT/UC по маркеру версии; на v2 ссылаться на серии CT/DM/EV для pin-authoring.

5. **Anti-patterns-дельта.** В конец добавлены 2 анти-паттерна: не читать FT/UC из
   Приложения на v2; не перенумеровывать FT/UC/CT/DM/EV.

### axis-2-блоки KMP — СОХРАНЕНЫ (не тронуты, по ACK ГД)
Подтверждаю, что MUST-KEEP axis-2-специфика KMP цела (git diff показывает только
добавления, единственное «-» = bullet 18-19, к которому дописана v2-локация):
- вход через `fr.md` из `/start-task <Jira-key>` (стр.15-16);
- перенос «Зафиксированных решений» `D-n` из FR как принятых;
- собственные добавления BRD продолжением серии с пометкой `(BRD)`;
- несколько FR → префикс серии `FR2-UC-3`, наследовать как есть;
- таблица расхождений FR↔KB + критичные → Jira issue label `needs-info`, владелец
  правки — аналитик;
- секции BRD и их порядок не меняются;
- grep self-check `UC-` по fr.md/brd.md == одно множество номеров.

### Как решена v1-ветка (развилка карты, «ОТКРЫТЫЙ ВОПРОС»)
Карта фиксировала неоднозначность: KMP-brd (18-19) читал `UC-n`/`FT-n` из `fr.md`
БЕЗ локации (ни §1.4/§1.5, ни П.A/П.B). По ACK/указанию: **v1-ветку НЕ менял на
канон-П.A/П.B** — оставил текущее KMP-поведение «читать из `fr.md` как раньше
(плоское чтение)». Т.е. добавлен ТОЛЬКО v2-путь (маркер → §1.4/1.5 + серии); при
отсутствии маркера — существующее KMP-поведение сохраняется дословно. Преамбула,
К-1 и Rules сформулированы в KMP-редакции (v1 = «из `fr.md` как раньше», НЕ П.A/П.B),
в отличие от канона, где v1 = П.A/П.B. Backward-safe, текущий KMP не ломается.

## Часть 2: kmp start-task — синк канонического гейта

Файл: `templates/iva-kmp-brownfield/ingredients/commands/start-task.md`. Вставлен
канонический гейт между Context-first блоком (шаг 4 `kb_get_task_context`) и
«Target-dir resolution», +36 строк:
- **К-5** — версионная осведомлённость по маркеру `fr_skeleton` (v2 §1.4–1.8 / v1
  Приложение П.A/П.B, без гейта; не угадывать);
- **К-3 (CRITICAL)** — plate-based гейт `D-n`: раздел FR с плашкой «Предложение,
  требует утверждения: разработчик + CTO» + открытый `Q-n` → BLOCKED до фиксации
  `D-n` в П.D; не выдумывать контракты/модель/события/UI, не имитировать реализацию;
  триггер по плашке независим от маркера (no fail-open); v1 без плашек/D-n → гейт
  не срабатывает (backward-safe).

Стек-специфика KMP start-task НЕ затёрта: Jira/Confluence fetch → `fr.md`, step-0
local-knowledge gate, source-repo cross-repo (read-only), no-sub-agent для Phase 1,
расширенный skills-список (вкл. `pin-ui-pipeline-check`) — всё сохранено. Внесена
только дельта гейта.

## Версия / CHANGELOG
- `manifest.yaml`: `0.5.1` → `0.5.2`.
- `CHANGELOG.md`: новая секция `[0.5.2]` (brd читает v2-FR §1.4/1.5 + серии; start-task
  гейт `D-n`; axis-2-специфика сохранена; backward-safe v1).

## Проверки (свежий прогон)
- **version-discipline** `--diff-against origin/main`: `OK — 48 profile(s) clean`.
- **pytest** (schemas + role_presets + install_smoke): `206 passed in 3.30s`.
  Прогон через `apps/backend` `uv run --extra dev` (conftest тянет alembic/sqlalchemy/
  pytest-asyncio — минимального `--with pyyaml/pytest/jsonschema` не хватило; venv
  gitignored, не в коммите).
- **git diff --stat vs база pin/tests:** 4 файла (brd + start-task + manifest +
  CHANGELOG), +98/-2.
- **git diff --stat vs origin/main:** 6 файлов (те 4 + pin/tests brd базы:
  pin-authoring + tests-authoring), +218/-2.
- **git log** origin/main..HEAD: `c39ed5e` (E-remainder) + `8b0e3c9` (pin/tests база).
- **git status:** clean.

## Развилки / durable
- **v1-ветка kmp brd** = плоское чтение из `fr.md` (НЕ П.A/П.B). Отличается от канона
  (канон v1 = П.A/П.B). Сознательно, по указанию — сохранение текущего KMP-поведения,
  backward-safe. Если позже канон-владелец решит унифицировать KMP-v1 на П.A/П.B —
  это отдельная задача, здесь НЕ делал.
- pin/tests (база) не трогал; другие профили, tacticum-dev-base, композиты — не трогал.
- НЕ push / НЕ merge / НЕ PR (autonomy off).