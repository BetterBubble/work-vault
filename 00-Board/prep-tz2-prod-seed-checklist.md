---
title: 'prep: ТЗ#2 прод-сид чеклист (режимы workflow) — ФИНАЛ'
type: note
permalink: tacticum/00-board/prep-tz2-prod-seed-checklist
status: draft
tags:
- draft
- lead-modes
- tz2
- prod-seed
- prep
---

# Прод-сид ТЗ#2 (режимы работы разработчика) — финальный чеклист

> **DRY-подготовка, версия от 2026-07-24 22:5x (после публикации §1.2).** Прод-сид — **gated-шаг президента**, сам НЕ выполняю и не деплою. Всё доведено до состояния «нажать go».
> Паттерн: как QA-профиль [[prod-seed-iva-role-qa-prep]] + рунбук [[deployment_prod_catalog_mcp]].
> Код ТЗ#2 в main (#142 режимы + добор), §1.2 — в origin `feat/workflow-modes-infra`, **ждёт мержа президента**. Прод-каталог автоматически НЕ обновляется — нужен ручной сид по SSH.
> Этот чеклист рассчитан на включение в **общий прод-сид всех 3 ТЗ** — он покрывает только пакеты ТЗ#2, границы с ТЗ#1/#3 явно отмечены в §5.

## 0. Предусловие (иначе сид неполный)
**PR §1.2 `feat/workflow-modes-infra` должен быть смержен ДО сида.** Без него в прод уедет `tacticum-lite-base` **0.1.2**, в котором лёгкий лейн всё ещё эскалирует инфра-задачи — то самое противоречие ТЗ, ради которого делался добор. Если мержа ещё нет — либо ждём, либо сидим осознанно без §1.2 и повторяем сид позже (лишний цикл, не рекомендую).

## 1. Что сидить — 12 пакетов, версии на момент подготовки

**Лейны-режимы (5):**

| Пакет | Версия | Что несёт |
|---|---|---|
| `tacticum-research-base` | **0.1.0** (новый) | режим research, `/start-research` — отчёт + черновик ADR, без кода |
| `tacticum-lite-base` | **0.1.3** ⚠️ *после мержа §1.2; в main сейчас 0.1.2* | режим lite, `/lite-task` (refactoring-S / feature-S) + инфра как свойство лейна |
| `tacticum-bugfix-base` | **0.1.3** | companion-роутинг + выход в research |
| `tacticum-development-core` | **0.1.1** | 2-й слой пересмотра режима в `run-implementation` |
| `iva-analysis-base` | **0.1.7** | 1-й гейт классификации в `/start-task` + ADR-first вход + 2-й слой Фазы-1 |

**Роли-композиты (7) — обновлённый `depends_on`, подхватывают новые лейны:**

| Роль | Версия | Несёт |
|---|---|---|
| `iva-role-go` | **0.2.1** | lite + research |
| `iva-role-ios` | **0.1.1** | lite + research |
| `iva-role-java` | **0.1.1** | lite + research |
| `iva-role-kmp` | **0.1.1** | lite + research |
| `iva-role-mail` | **0.1.1** | lite + research |
| `iva-role-web` | **0.1.1** | lite + research |
| `iva-role-analyst` | **0.1.2** | research (lite аналитику не положен) |

**⚠️ Версии обязательно пере-сверить непосредственно перед сидом** — main двигается, соседние ТЗ поднимают версии тех же ролей-композитов. Команда сверки (read-only, из чистого `tacticum-dev` на свежем main):

```
git fetch origin && git checkout main && git pull
for p in tacticum-research-base tacticum-lite-base tacticum-bugfix-base \
         tacticum-development-core iva-analysis-base \
         iva-role-go iva-role-ios iva-role-java iva-role-kmp \
         iva-role-mail iva-role-web iva-role-analyst; do
  printf "%-28s %s\n" "$p" "$(grep -m1 '^version:' templates/$p/manifest.yaml)"
done
```

Сидим **фактические версии из main на момент сида**, не эти — таблица выше только для сверки «ничего не потерялось».

## 2. Порядок (по рунбуку `deployment_prod_catalog_mcp`)

1. **Read-only pre-flight (обязательно):** `SELECT` по `profile_versions` в Postgres `tacticum_catalog` (через `docker exec tacticum-catalog-mcp-1`) по всем 12 пакетам — что уже засижено и совпадает ли контент под текущими версиями. Вердикт по каждому: `created` / `noop` / ⚠️ `collision` (`version_already_exists_with_different_content` → нужен bump версии, сид не форсировать). Прод-VPS `tacticum_prod` (159.194.224.59) в ssh-manager.
2. **Бэкап (обязательно, точка отката):** `pg_dump` каталога ДО сида — как на QA-профиле (там бэкап вышел ~11 МБ). Без снятого бэкапа сид не начинать.
3. **Pull контента:** `git pull` в `/opt/tacticum` — свежий main с пакетами.
4. **Доставка + сид:** `docker cp` `templates/` + `seed_community.py` в контейнер → `docker exec` сид по 12 пакетам. **Пересборка образа НЕ нужна** — меняется только контент профилей.
5. **Verify на живой БД** (см. §3) — не по логу сида, а запросом к каталогу и живым `readyz`.
6. **Re-pin:** проверить `installations` затронутых ролей — нужен ли re-pin. У новых лейнов `installations` скорее всего 0 → re-pin не потребуется (как было у QA-профиля).
7. **Wiki-sync** — если применимо, нужен токен.

## 3. Verify-критерии (зелёный = сид можно закрывать)

- Все 12 пакетов в каталоге `active`, версии совпадают с main на момент сида.
- Рёбра `depends_on` ролей резолвятся без orphan: 6 dev-ролей несут `lite` + `research` рядом с `bugfix`; `iva-role-analyst` несёт `research`; `core-base` подтягивается из БД.
- Команды доступны там, где положено: `/start-research` — у 7 ролей, `/lite-task` — у 6 dev-ролей.
- **Проверка §1.2 (новое):** в засиженном `tacticum-lite-base` 0.1.3 скилл `lite-task-workflow` НЕ содержит `infrastructure` в списке поводов эскалации Step 0 и содержит блок **«Infra trait»** в Step 3. Это единственное смысловое отличие 0.1.3 от 0.1.2 — если его нет, засижена старая версия.
- `readyz` отдаёт 200; MCP-транспорт здоров (401 на голый запрос = сервис жив, это норма).
- Бэкап-дамп снят и лежит доступным для отката.

## 4. Откат
Точка отката — `pg_dump` из шага 2. Сид аддитивный (новые версии профилей), но при `collision` или битом verify откатываемся восстановлением дампа каталога, не ручной правкой строк.

## 5. Границы (важно при объединении с прод-сидом ТЗ#1 и ТЗ#3)
- **Сам не сижу и не деплою** — gated-шаг президента (autonomy off); применяет президент / Diaret по SSH.
- Этот чеклист покрывает **только 12 пакетов ТЗ#2**. Смежные пакеты — `iva-web-development-base` (ТЗ#1, DS-скиллы), `iva-kmp-brownfield` / `iva-web-brownfield`, аналитические лейны ТЗ#3 — сидят соответствующие лиды, я их не трогаю.
- **Точка пересечения — роли-композиты.** `iva-role-web` и другие роли могут получить bump версии от ТЗ#1/#3 (например #151 добавлял DS-скиллы в `iva-web-development-base`, от которого зависит `iva-role-web`). При общем сиде роль сидится **один раз последней актуальной версией из main**, а не по разу на каждое ТЗ — иначе коллизия версий. Это надо развести на уровне общего сида, отдельно от моего чеклиста.
- Docker-образ не пересобирать, прод трогать точечно.

## Связано
[[plan-modes]] · [[prep-tz2-acceptance-package]] · [[prep-tz2-eval-readiness]] · [[deployment_prod_catalog_mcp]] · [[prod-seed-iva-role-qa-prep]] · [[pilot-tz2-teststand-results]]
