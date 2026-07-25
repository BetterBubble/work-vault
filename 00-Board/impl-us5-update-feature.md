---
status: draft
role: implementer
task: ТЗ#3 US#5 — /update-feature пер-раздельно + фиксация утверждений как D-n (вариант
  А owner+mirror)
branch: feat/us3-dm-ev
worktree: /Users/bubblemac/tacticum-worktrees/us3-dm-ev
commit: 98639dc (US#5, поверх US#3 366c798) + 8ffff73 (доправка api-contracts §2.4-шапка)
head: 8ffff73
date: 2026-07-24
permalink: tacticum/00-board/impl-us5-update-feature
---

> **Доправка (тот же PR, укрупнение — ГД разрешил фолдить):** коммит `8ffff73` —
> честная ссылка §2.4. Навыки DM/EV ссылались на «тот же контракт §2.4, что у
> `api-contracts-discovery`», но одноимённой шапки-декларации у api-contracts не
> было — ссылка висела. Добавил симметричную шапку «**Единый контракт
> навыка-анализатора (ТЗ §2.4)**» перед разделом «Контрактный раздел FR — формат
> 3.1»: вход = фактура Части 2 (пробы П.F + код-verify П.E + capability-требования
> FT/UC), выход = §1.6 со своей серией CT-n, деградации явные («не применимо» /
> «фактуры мало → Q-n»). Остальной контент api-contracts (формат 3.1 / CT-n /
> чеклист / JUMP из US#2) не тронут. Оба профиля identical. Version bump не
> делался. Проверки после доправки: mirror_sync OK (64/6), diff -q api-contracts
> identical, DM/EV/fr-authoring/update-feature целы (identical), version-discipline
> clean (48), pytest целевые **290 passed**. Лог `git log origin/main..HEAD`:
> `8ffff73` (§2.4-шапка) + `98639dc` (US#5) + `366c798` (US#3).

# US#5 — /update-feature пер-раздельно + D-n-фиксация утверждений

## Итог
Реализовано аддитивно поверх US#3 НОВЫМ коммитом `98639dc` (US#3 `366c798` не тронут / не amend). Оба профиля байт-в-байт. Версии НЕ поднимались (остаются iva-analysis-base 0.1.6, iva-fr-analyst 0.1.12); CHANGELOG-записи под текущими версиями расширены строками про US#5.

## Развилка архитектуры (durably)
`update-feature` — физически ОДИН тонкий файл-обёртка (`.md`, 34 строки), а не папка с под-структурой разделов. Он делегирует исполнение скиллу `fr-authoring` (Skill tool), раздел «Режим /update-feature», шаги У-1…У-3. Поэтому пер-раздельность нельзя «разложить по файлам разделов» — её описываю как ЛОГИКУ в двух местах:
1. **Контракт команды** — `update-feature.md` (Порядок): что делает /update-feature.
2. **Исполняющая механика** — `fr-authoring/SKILL.md`, У-3 Режима /update-feature (там реально работает агент).
Плюс обязательная согласовка валидатора границы. `fr-authoring` — mirror-пара с базой, так что все правки владелец↔зеркало байт-в-байт (не рвёт 64 mirror-пары).

## Что изменено (человеческим языком)

### 1. Пер-раздельная перегенерация (§2.4 п.5)
Раньше У-3 говорил обобщённо «решение меняет скоуп/UC — обнови UC/FT». Теперь /update-feature перегенерирует ТОЛЬКО затронутый раздел, определяя его по природе изменения, и перезапускает ЛИШЬ соответствующий навык:
- контракт/интеграция (§1.6) → `api-contracts-discovery` (CT-n);
- модель данных/состояния (§1.7) → `data-model-analyzer` (DM-n);
- события/консистентность (§1.8) → `events-analyzer` (EV-n);
- скоуп/сценарий (§1.4/§1.5) → шаги `fr-authoring` (вкл. код-verify, если вопрос про код).
Разделы вне изменения не трогаются, номера FT/UC/CT/DM/EV стабильны; несколько затронутых разделов — каждый навык по отдельности.

### 2. Фиксация утверждений проектных разделов как D-n (§2.4 п.5, §2.2 п.3)
Утверждение проектного раздела (§1.6/§1.7/§1.8) разработчиком + CTO приходит комментарием на странице ПОСЛЕ публикации и фиксируется через /update-feature. Механика: У-2 классифицирует комментарий-утверждение как «ответ на вопрос» по `Q-n` раздела → У-3 записывает `D-n` в П.D (дата, авторы, «в пользу чего»), СНИМАЕТ с раздела плашку «Предложение, требует утверждения: разработчик + CTO» и ЗАКРЫВАЕТ его открытый `Q-n` (след сохраняется). Раздел переходит «требует утверждения» → утверждён; имена остаются с прежним основанием в Приложении — меняется только состояние (плашка → D-n).

### 3. Согласовка с двухзонной §2 и валидатором границы US#1
Валидатор границы, п.(iii), раньше давал *fail* всегда, если у проектных имён нет плашки+`Q-n`. Это конфликтовало бы с утверждённым состоянием (после D-n плашка снята, Q-n закрыт). Переписал п.(iii) на ДВА правомерных состояния: (а) не утверждён — плашка + открытый `Q-n`; (б) утверждён — имена зафиксированы `D-n` в П.D (пришло через /update-feature). *fail* остаётся только на голые имена без того и без другого. Основание в Приложении по-прежнему обязательно в обоих состояниях (проверяет п.(ii)). §2 не нарушен: утверждённое имя больше не «требует Q-n».

## Файлы (абсолютные пути)
- `/Users/bubblemac/tacticum-worktrees/us3-dm-ev/templates/iva-analysis-base/ingredients/commands/update-feature.md`
- `/Users/bubblemac/tacticum-worktrees/us3-dm-ev/templates/iva-fr-analyst/ingredients/commands/update-feature.md`
- `/Users/bubblemac/tacticum-worktrees/us3-dm-ev/templates/iva-analysis-base/ingredients/skills/fr-authoring/SKILL.md`
- `/Users/bubblemac/tacticum-worktrees/us3-dm-ev/templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md`
- `/Users/bubblemac/tacticum-worktrees/us3-dm-ev/templates/iva-analysis-base/CHANGELOG.md`
- `/Users/bubblemac/tacticum-worktrees/us3-dm-ev/templates/iva-fr-analyst/CHANGELOG.md`

US#5-коммит трогает ровно эти 6 файлов (`git diff --stat 366c798..98639dc`).

## Проверки (свежий прогон, env uv run --with pyyaml --with pytest --with jsonschema)
- `check_mirror_sync.py` → **OK — 64 зеркальных ингредиента в 6 парах синхронны** (число как было).
- `diff -q` update-feature (оба профиля) → **IDENTICAL**; fr-authoring (оба) → **IDENTICAL**.
- Целостность US#3/US#1-2: `data-model-analyzer`, `events-analyzer`, `api-contracts-discovery` (оба профиля) → **IDENTICAL** (не сломаны).
- `check_profile_version_discipline.py --diff-against origin/main` → **OK — 48 profile(s) clean**.
- pytest целевые (schemas + parity + role_presets + install_smoke), `--noconftest` → **290 passed, 0 failed**. parity зелёный без allowlist.
  (Прочие тесты каталога падают на сборе из-за отсутствия sqlalchemy в env — это не целевые и не связано с правкой.)
- `git log origin/main..HEAD --oneline` → `98639dc` (US#5) + `366c798` (US#3).

## Прод-safe
Аддитивно: базовый флоу реконсиляции/публикации (У-1…У-3) сохранён, надстроены пер-раздельность и D-n-фиксация. US#3-контент (навыки DM/EV, §1.7/§1.8) не переписан — правки только в update-feature, У-3 Режима, п.(iii) валидатора и CHANGELOG.

## Не делал
Push / PR / merge — не выполнялись (по заданию, autonomy off).