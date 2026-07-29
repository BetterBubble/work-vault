---
title: impl ДС кросс-ссылки и триггеры 2026-07-28
type: note
status: current
created: 2026-07-28 10:00
repo: tacticum-dev
tags: [board, design-system]
permalink: tacticum/00-board/impl-ds-crosslinks-2026-07-28
updated: 2026-07-28 10:00
---

# Реализация: обратные ссылки ДС-навыков + русская триггерная поверхность

Ветка: `fix/ds-skill-crosslinks-triggers`, база `26f5301` (`origin/main`).
Worktree: `/Users/bubblemac/tacticum/tacticum-dev-ds-crosslinks`.
Один коммит: `fc444e2` (правки, зеркала, версии и CHANGELOG — вместе, как просили).
Не пушил, не мержил, ветка локальная.

## Файлы (13, +244 / −24)

| Файл | +/− |
|---|---|
| `templates/iva-kmp-development-base/ingredients/skills/compose-multiplatform-ui/SKILL.md` | +34 / −3 |
| `templates/iva-kmp-development-base/ingredients/skills/ui-mockup-match/SKILL.md` | +24 / −4 |
| `templates/iva-kmp-development-base/ingredients/skills/web-to-kmp-screen-port/SKILL.md` | +7 / −2 |
| `templates/iva-kmp-development-base/ingredients/skills/design-token-usage/SKILL.md` | +8 / −0 |
| `templates/iva-kmp-development-base/ingredients/skills/design-system-discovery/SKILL.md` | +7 / −0 |
| `templates/iva-kmp-development-base/manifest.yaml` | +15 / −5 |
| `templates/iva-kmp-development-base/CHANGELOG.md` | +50 / −0 |
| те же 4 навыка в `templates/iva-kmp-brownfield/ingredients/skills/…` | +73 / −7 (зеркало) |
| `templates/iva-kmp-brownfield/manifest.yaml` | +3 / −3 |
| `templates/iva-kmp-brownfield/CHANGELOG.md` | +23 / −0 |

## Правка 1 — пять обратных рёбер в `ds-component-adoption`

Везде дано **условие перехода**, а не имя навыка.

1. **`web-to-kmp-screen-port`** — три точки, а не одна:
   - строка в таблице `## 8` — «Component level — where the §1.M M5 / §5 stop goes», с явным
     разделением уровней (один элемент против экрана) и оговоркой, что порт продолжается только
     когда композабл существует;
   - сам стоп **§1.M M5** («элемент фрейма без `Iva*`-эквивалента — остановись и сообщи») — теперь
     отчёт об этом элементе назван точкой входа в `ds-component-adoption`. Раньше стоп не
     маршрутизировал никуда, это и была дыра;
   - анти-паттерн **§5** «Do not hallucinate components/APIs … If it is not there, stop» — та же
     оговорка одной строкой.
2. **`design-token-usage`** — новая секция `## Related — when the job is not this one`, три строки:
   `ds-component-adoption` (литералы внутри самодельного блока, который ДС уже несёт, токенизировать
   бессмысленно — работа выбрасывается; плюс строка «No token» из §6 превращается в «нужен ли
   компонент»), `web-to-kmp-screen-port` (сдвинулся фрейм, а не значения), `design-system-discovery`
   (это ещё фаза дизайна).
3. **`ui-mockup-match`** — во врезку `Route out before you start` добавлен четвёртый пункт;
   поставлен **после** пункта о неоднозначном «экран не соответствует макету», чтобы не разрывать
   пару «фрейм сдвинулся / реализация уехала». Формулировка: сверка **сообщает** такую дельту и
   останавливается, adopt/promote/author не делает.
4. **`design-system-discovery`** — новая секция `## Related`: `design-token-usage` (фаза дизайна
   кончилась) и `ds-component-adoption` (перечень токенов не отвечает на вопрос «каким композаблом
   это рисуется»).
5. **`compose-multiplatform-ui`** — новая врезка `## Route out before you start` сразу после интро,
   таблица на всех четырёх соседей; плюс пункт в §Design system: перед тем как рисовать
   `Column`/`Box`/`Text` руками — проверь, не несёт ли `:core:design-system` готовый `Iva*`.

## Правка 2 — русские триггеры и анти-триггеры

- **`compose-multiplatform-ui`**: одиночные «экран» и «компонент» **убраны** — именно они и
  перехватывали запросы соседей. Вместо них 11 фраз, каждая однозначна для авторского прохода без
  внешнего эталона: «собери новый экран», «написать экран с нуля», «новый экран без макета», «как
  правильно собрать экран», «структура экрана на Compose», «новый composable», «состояние экрана в
  Compose», «подключить экран к Decompose», «стабильность состояния», «лишние рекомпозиции»,
  «вёрстка на Compose Multiplatform». Добавлен блок «Four things this is NOT» по образцу
  `web-to-kmp-screen-port` — стрелки в `web-to-kmp-screen-port`, `design-token-usage`,
  `ui-mockup-match`, `ds-component-adoption`.
- **`ui-mockup-match`**: русских триггеров **4 → 11** (добавлены «проверь экран по макету»,
  «насколько экран совпадает с макетом», «отклонения от макета», «экран съехал от макета», «приёмка
  экрана по макету», «сверить реализацию с фреймом», «ΔE по токенам»). Добавлен тот же блок
  анти-триггеров, и в поверхность вынесен разделитель для неоднозначной фразы «экран не соответствует
  макету»: решает **что сдвинулось** — фрейм отправляет в авторский проход, дрейф реализации остаётся
  здесь.
- Английские одиночные триггеры (`Screen`, `UI`, `Compose`) **не тронуты**: замер был по русским
  формулировкам, а трогать неизмеренное в той же правке значило бы менять поведение вслепую. Болезнь
  у них та же — это в «спорном» ниже.

## Правка 3 — зеркала и манифесты

- Четыре зеркалируемых навыка (`ui-mockup-match`, `compose-multiplatform-ui`, `design-token-usage`,
  `design-system-discovery`) скопированы в `iva-kmp-brownfield` **байт-в-байт**: `cmp` даёт 0 отличий
  по всем четырём, `scripts/check_mirror_sync.py` → **OK, 68 зеркальных ингредиентов в 6 парах
  синхронны**.
- `description_trigger` синхронизирован в обоих манифестах для `ui-mockup-match` и
  `compose-multiplatform-ui`. У `compose-multiplatform-ui` в `iva-kmp-development-base` пришлось
  сменить скаляр на `>-` (в плоском plain-скаляре текст с двоеточиями сломал бы YAML). Оба манифеста
  парсятся `yaml.safe_load`.
- Версии: `iva-kmp-development-base` 0.12.0 → **0.12.1**, `iva-kmp-brownfield` 0.5.4 → **0.5.5**.
  `scripts/check_profile_version_discipline.py --diff-against 26f5301` → **OK, 48 profile(s) clean**.

## Проверки — числа

| Проверка | Результат |
|---|---|
| `scripts/check_mirror_sync.py` | **OK** — 68 ингредиентов, 6 пар |
| `scripts/check_profile_version_discipline.py --diff-against 26f5301` | **OK** — 48 профилей |
| `tests/catalog/test_role_replacement_parity.py` | **91 passed**, 0 failed |
| `tests/catalog` целиком | **588 passed, 2 failed, 120 errors** — все 122 непрохода это `OSError: Connect call failed … 5432`, Postgres не поднят. Проверено на образце (`test_patch_profile_404`); ни один непроход не касается `templates/**` |
| `tests/e2e_install/` | **прогнать не удалось**: docker-демон не запущен (`Cannot connect to the Docker daemon at unix:///Users/bubblemac/.docker/run/docker.sock`), фикстура поднимает `postgres:16-alpine` контейнером. Ни один тест не стартовал |
| `git status` в worktree | чисто, мусора (`.DS_Store`, `__pycache__`, `.serena/`) нет |

### Golden-деревья — что изменится (посчитано хешами, не наблюдением)

E2E прогнать нельзя, поэтому дрейф посчитан прямо: sha256 новых тел против записей golden.
**Задеты 2 файла, 10 записей** — и только они (полный скан всех golden на старые хеши):

| golden | записей | навыки |
|---|---|---|
| `apps/backend/tests/e2e_install/golden/iva-role-kmp/claude-code.json` | 5 | `compose-multiplatform-ui` `1703cd67→95e26969`, `design-system-discovery` `2f154e53→83b2b107`, `design-token-usage` `05fe5126→bb7be698`, `ui-mockup-match` `cb50a5e6→868c663e`, `web-to-kmp-screen-port` `24dffa40→0e49bda1` |
| `apps/backend/tests/e2e_install/golden/iva-role-kmp/codex.json` | 5 | те же пять |

Golden **не переписаны** — ни молча, ни громко: перегенерация штатно делается прогоном
(`E2E_INSTALL_REGEN_GOLDEN=1 pytest tests/e2e_install`), а прогнать нечем. Ручная подстановка десяти
хешей технически возможна и однозначна, но она подтвердила бы только то, что я и так посчитал, и
скрыла бы любой дрейф, которого я не предвидел. Перегенерировать нужно **до мержа**, на машине с
docker.

Побочно проверено: `description_trigger` в golden-деревья **не попадает** — установленное тело
идентично исходному `SKILL.md` байт-в-байт (хеши совпали), то есть правка манифестов на golden не
влияет, только правка тел.

## Спорное и то, на что не хватило оснований

1. **Преамбула §8 `web-to-kmp-screen-port` врёт — не трогал.** Она говорит «All of these are **real
   skills in the su.ivcs.messenger KMP repo** (`AI common/skills/`)», но половина строк таблицы —
   ингредиенты профиля, а не навыки репозитория, и ставятся в `.claude/skills/` (`install_scope:
   user`): `compose-multiplatform-ui`, `design-token-usage`, `design-system-discovery`,
   `ui-mockup-match`. Добавленный мной `ds-component-adoption` как раз `install_scope: repo` и
   ложится в `AI common/skills/`, то есть **соответствует преамбуле лучше**, чем половина уже
   стоящих там строк. Правка преамбулы — отдельная задача, в объём не входила.
2. **Длина `description` во frontmatter.** После правки: `ui-mockup-match` 2323 символа,
   `compose-multiplatform-ui` 1689. Спецификация Agent Skills ограничивает `description` 1024
   символами; в репозитории эта граница нигде не проверяется, а образец, на который меня послали
   (`web-to-kmp-screen-port`), сам весит 2353 символа **в текущем main**. То есть я не ввёл новое
   нарушение, а встал в существующий ряд. Но если лимит где-то живой (на стороне клиента, а не
   каталога) — рискуют теперь три навыка вместо одного. Оснований проверить это у меня нет.
3. **Английские одиночные триггеры оставлены.** `Screen`, `UI`, `Compose`, `theme` у
   `compose-multiplatform-ui` перетягивают ровно так же, как перетягивали «экран»/«компонент».
   Гейт мерил русские формулировки; менять неизмеренное вместе с измеренным — значит потерять
   возможность сказать, что именно помогло.
4. **«натянуть макет» остался триггером `ui-mockup-match`.** По смыслу это скорее авторский проход
   (`web-to-kmp-screen-port`), чем пост-проверка. Фраза досталась в наследство, я её не трогал:
   убрать — значит потерять маршрут вообще (у соседа её нет), переносить — правка чужой поверхности
   вне задачи. Кандидат на следующий заход.
5. **Зеркало ссылается на навыки, которых на профиле нет.** В `iva-kmp-brownfield`
   `ds-component-adoption` и `web-to-kmp-screen-port` не установлены (owner-only), а тела навыков
   обязаны совпадать байт-в-байт — значит ссылки на них уезжают в зеркало и открыть навык там нельзя.
   Как маршрутизация «это не мой случай» они осмысленны, как ссылка — нет. Это цена зеркала, а не
   недосмотр; в CHANGELOG зеркала записано явно.
6. **Замер после правки не повторён.** Исходное «две формулировки из пяти уходят не туда» — число из
   разведки, снятое гейтом. Своего прогона гейта я не делал: чем он запускается, в постановке не
   сказано, а угадывать не стал. Проверить, что стало лучше, пока нечем.
