---
title: explore-kmp-webcalls-2026-07-28 — проверка факта «WebCallsRepository удалён»
  по коду
type: note
status: current
created: 2026-07-28 09:47
repo: tacticum-dev
tags:
- board
- kmp
- calls
permalink: tacticum/00-board/explore-kmp-webcalls-2026-07-28-1
updated: 2026-07-28 09:47
archived-at: 2026-08-05 15:19
---

# Разведка: существует ли `WebCallsRepository` в KMP-репо ИВА

Проверка прод-фидбэка по `IVAONEHALF-693` (см. [[ds-prod-check-2026-07-28]], раздел
«Найдено рядом»). Утверждение человека: класс `WebCallsRepository` **удалён**, Web использует
общий `CallsRepositoryImpl` из `core/data`, в исходниках файла нет; источник —
`AI common/skills/custom/calls/RISKS.md` §4 в его локальном KMP-репо.

## Главное в двух строчках

**Опровергнуть утверждение нечем, и подтвердить нечем тоже.** Единственная доступная нам копия
KMP-кода — статический repomix-снимок от **2026-07-03**, и в нём `WebCallsRepository`
**есть, живой и содержательный**. Снимок на 25 дней старше фидбэка (27.07), поэтому он
описывает прошлое состояние и удаление после 03.07 не опровергает.

## Что за копия кода и насколько она свежая

`/Users/bubblemac/tacticum/helm/data/real/git/repomix/kmp.xml` — 40 МБ, repomix-снимок
`--style xml --compress`, 11 171 файл. Дата снятия — **2026-07-03**, прямо заявлена в
`/Users/bubblemac/tacticum/helm/data/real/git/repomix/README.md`, там же оговорка: «Снимок
статичен (2026-07-03), не следит за изменениями».

Репозиторий в реестре Helm называется `iva-m/android/kmp`. Самый свежий коммит по нему в наших
данных (`data/real/git/all_commits.csv`, выгрузка от 04.07) — **2026-07-04**. Дальше 04.07 у нас
нет ничего: ни кода, ни истории.

## (а) Существует ли `WebCallsRepository`

**На 2026-07-03 — да, существует.** Путь в снимке:

```
composeApp/src/webMain/kotlin/su/ivcs/messenger/composeapp/di/web/WebCallsRepository.kt
```

Объявление — `internal class WebCallsRepository(...) : CallsRepository`, 5 конструкторных
зависимостей (`callsNetworkSource`, `systemCacheSource`, `centrifugeClient`, `mcuSocketClient`,
`scope`). Всего 18 вхождений имени по снимку, в том числе:

- `composeApp/src/webMain/.../di/ComposeAppRepositoryBindings.web.kt` — импорт и реальная
  проводка: `val callsRepository = WebCallsRepository(`;
- `core/network-state/.../NetworkConnectionStateImpl.kt` — комментарии «Exposed so platform
  repository bindings (e.g. web `WebCallsRepository`) can subscribe to the same …»;
- `AI common/skills/custom/CONFERENCE_BACKLOG.md` — раздел M7 (аудит от 2026-06-27) с таблицей
  платформ, где web отдельной строкой.

Класс на 03.07 не был мёртвым: в нём правился `_participantMediaStates` с комментарием
«Зеркало core/data `CallsRepositoryImpl`» — то есть шло сближение с общей реализацией, а не
её замена.

**На 2026-07-28 — не знаем.** Кода свежее 04.07 у нас нет.

## (б) `AI common/skills/custom/calls/RISKS.md`

**Файла в нашем снимке нет, и каталога `calls/` тоже нет.** На 03.07 `AI common/skills/custom/`
был **плоским**: `CALLS.md`, `CALLS2.md`, `CALLS_MULTIPLATFORM_PLAN.md`, `CONFERENCE_BACKLOG.md`,
`WEBRTC_STATUS.md` и т.д. — без подкаталогов. Поиск по снимку: `RISKS` — 0 вхождений,
`calls/overview` — 0 вхождений.

**Это косвенно подтверждает, что человек прав про свежесть.** Наш собственный навык
`calls-voip-fragile-zone` (коммит `55c7675`, Dmitry Solonko, **2026-07-07**) уже ссылается на
`AI common/skills/custom/calls/overview/CALLS.md` — путь с подкаталогом, которого 03.07 ещё не
было. Значит `custom/` реструктурировали между 03.07 и 07.07, зона активно перестраивается, и
`calls/RISKS.md` — артефакт новее нашей копии. Процитировать §4 нечем.

## (в) Использует ли Web общий `CallsRepositoryImpl` из `core/data`

**На 03.07 — нет, категорически.** Разделение было архитектурно осознанным и задокументированным
в самом коде. Дословно из KDoc `WebCallsRepository.kt`:

> Network-backed calls repository for JS/Wasm targets (no Room). … `callInChat` joins the MCU
> exactly like the desktop `core.data...CallsRepositoryImpl` (no Room needed). **That impl lives in
> the `:core:data` module, which the web target intentionally does not depend on, so it is
> referenced by name only.**

Причина разделения названа прямо: `core/data` завязан на Room, а JS/Wasm-таргет Room не имеет.
`CallsRepositoryImpl` в снимке лежит в
`core/data/src/commonMain/kotlin/su/ivcs/messenger/core/data/repositories/implementation/CallsRepositoryImpl.kt`
(плюс отдельная androidMain-реализация в `feature/calls/`). Таблица M7 в `CONFERENCE_BACKLOG.md`
подтверждает: android / desktop / iOS → `core/data CallsRepositoryImpl`, **web → `WebCallsRepository`
(composeApp webMain)**.

То есть п.7 нашего навыка на 03.07 был **фактически верен**. Если он неверен сегодня — значит
зависимость web на `:core:data` за три недели появилась, что для KMP-таргета JS/Wasm означает
снятие Room-барьера. Это крупная перестройка, а не косметика; правдоподобно (см. §б —
реструктуризация в зоне идёт), но нашими данными не проверяется.

## Что именно проверялось и чем закончилось

| Что | Результат |
|---|---|
| Локальные каталоги: `~/tacticum/*`, `~/tacticum-worktrees/*`, `Downloads`, `Desktop`, `Documents` | клона KMP-репо нет |
| `find ~ -maxdepth 10 -name "*.kt"` (без Library/Trash) | 3 файла, все — тестовые фикстуры `KB-Brownfield-Bootstrap/tests/fixtures/kotlin/` |
| `find ~ -maxdepth 8 -type d -name composeApp / webMain / commonMain` | пусто |
| `find ~ -maxdepth 7 -name RISKS.md` | пусто |
| Проекты Serena (`~/.serena/serena_config.yml`) и Claude (`~/.claude.json`) | KMP-репо не зарегистрирован ни в одном |
| Прямой `ssh helm` из bash | `Could not resolve hostname helm` — хостов в `~/.ssh-manager/config.json` нет, доступ только через MCP |
| MCP `ssh-manager`, `helm-analyst`, `basic-memory` | **у роли explorer этих инструментов нет** — доступны только Read/Bash. Серверное зеркало и `analyst_search` не проверены |
| repomix-снимок `helm/data/real/git/repomix/kmp.xml` | **найден, использован** — единственный источник кода |

## Где утверждение живёт у нас (для того, кто будет править)

Навык `calls-voip-fragile-zone`, п.7 и анти-паттерн, **две байт-идентичные копии** (md5
`4312e02a44a97001d86f1c8c87d596ad`), связаны mirror-парой в `templates/_mirrors.yaml`
(owner `iva-kmp-development-base` → mirror `iva-kmp-brownfield`, строки 26–29):

- `templates/iva-kmp-development-base/ingredients/skills/calls-voip-fragile-zone/SKILL.md:46-47`
  и `:79` — **владелец, правится здесь**
- `templates/iva-kmp-brownfield/ingredients/skills/calls-voip-fragile-zone/SKILL.md:46-47`
  и `:79` — зеркало, тем же PR
- `templates/iva-kmp-brownfield/CHANGELOG.md:513` — та же формулировка в описании навыка

Происхождение: навык принесён коммитом `55c7675` (Dmitry Solonko, 2026-07-07) в brownfield;
в development-base приехал `508854f` (glebfel, 2026-07-22, ADR-0059).

**Правок не вносил** — роль read-only.

## Открытый вопрос для лида

Чтобы закрыть факт, нужен код свежее 04.07. Три пути, все вне моих инструментов:
зеркало на сервере через MCP `ssh-manager`; `mcp__helm-analyst__analyst_search` по
`WebCallsRepository`; либо попросить у автора фидбэка сам текст `RISKS.md` §4. Пере-снятие
repomix (Волна 1b в README) закрыло бы и это, и все будущие такие вопросы разом.

---

**Проверял:** explore-kmp-webcalls, 2026-07-28. Только чтение.
Связано: [[ds-prod-check-2026-07-28]]