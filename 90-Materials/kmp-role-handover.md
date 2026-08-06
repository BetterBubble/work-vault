---
title: kmp-role-handover
type: note
permalink: tacticum/90-materials/kmp-role-handover
---

# KMP-роль Tacticum — передача сопровождения тиража

Для нового разработчика tacticum-dev. Дата: 2026-07-31. Владелец в отпуске;
контакт команды KMP — Пётр Корнюшин (обращение на «ты»).

## 1. Что это и из чего состоит

**iva-role-kmp** — тонкая роль-пресет (ADR-0059): весь контент приходит из
лейнов по `depends_on`. Стековая специфика — лейн **iva-kmp-development-base**
(правки навыков делаются здесь). Старый профиль **iva-kmp-brownfield** ещё жив
у боевой когорты и связан с лейном зеркалами переходного периода:
`templates/_mirrors.yaml`, байт-в-байт, правка — только у владельца-лейна и
зеркало обновляется **тем же PR** (CI: `scripts/check_mirror_sync.py`).

Текущие версии в проде: лейн **0.21.0**, роль **0.4.4**, brownfield **0.10.0**.

Установки (прод БД, таблица `installations`):
- боевая `687db0e8-…` — brownfield, репо D:/iva/kmp, org-shared (primary
  n.gribtsov@iva.ru), pin 0.10.0;
- пилот `cbb20b79-…` — роль, workspace iva-codex-smoke, pin 0.4.4;
- `0a4d393e-…` — smoke-фикстура со спец-пином 0.5.4 — **не перепинывать**.

## 2. Где зафиксированы знания

| Что | Где |
|---|---|
| Композиция ролей/лейнов | ADR-0055…0060 (docs/adr/) |
| ДС KMP: iva-mobile = оверлей веб-основы, эталон iva-one `@iva/design-system`, гард источников макета, DQA-реестр, ручная геометрия | **ADR-0061** |
| Петля фидбэка: maintainer-feedback, 5 меток, читалка, helm | **ADR-0062** |
| Кросс-репо чтение KB (workspace-семантика) | ADR-0034 D3 + навык kb-navigation (core) |
| Прод-runbook (pull/seed/rebuild/перепин/гочи) | Serena memory `deployment_prod_catalog_mcp` (.serena/memories/) |
| ТЗ дизайн-процесса Figma | wiki project.cifragen.ru → tacticum-tacticum_dev → figma-ds-process-tz |
| Словарь Figma↔код iva-core, review-queue | figma-ds-process/iva-core-mapping/ |
| Задачи/стори | Taiga, проект tacticum-dev (id 12), эпик #540 |
| Quickstart'ы профилей | docs/user_manuals/ (wiki.iva.ru синкается сам по расписанию — руками не запускать) |

## 3. Проблемы, которые мы встречали, и где лежит решение

1. **Figma-доступ рушился посреди работы** (403 нет scope, 429 квота
   `/v1/images` ~4 батча → бан на ~4,5 суток per-account; Variables API — 403
   навсегда, scope Enterprise-only). Решение: навык **figma-access-setup**
   (визард канала: Dev Mode probe → выпуск PAT со scope → режим экономии REST →
   fallback на скрины) + протокол квоты в **ui-mockup-match** (батч-/nodes,
   429/Retry-After >10 мин = стоп) + token-join цветов через
   `design_get_tokens("iva-mobile")`, не через Figma Variables.
2. **Агент считал задокументированные решения дефектами** (82 raw dp, прямые
   Material3). Решение: known-facts преамбула в ui-mockup-match: raw dp —
   ручная геометрия by design (в ДС нет spacing-токенов), дрейф тема-холдеров —
   известный, letterSpacing исключён из сверки навсегда; Iva*-обёртки
   (IvaText/IvaButton) команда планирует сама — прямые M3 = инвентарь +
   component-gap, не FAIL. Всё в ADR-0061.
3. **Пустой kb_discover принимался за «KB недоступна»** (repo-bound
   context.yaml указывал на smoke-workspace). Решение: правило в kb-navigation
   (core 0.4.0): пустой discover → `whoami` → перебор установок.
4. **ПК разработчика завис (32 ГБ)** — параллельные implementation-агенты, у
   каждого Gradle daemon + безлимитный Kotlin daemon + Node. Решение: секция
   «Build resource discipline» в **kmp-build-verification** (бюджет RAM
   спрашивается один раз → `.tacticum/context.yaml`; caps в
   `~/.gradle/gradle.properties`; одна сборка за раз; `--stop` в конце) +
   в плане тиража параллельные волны заменены серийным конвейером.
5. **Обратная связь не доходила до мейнтейнеров.** Решение (ADR-0062):
   `submit_feedback` с метками `skill-drift:/skill-gap:/skill-conflict:/
   alt-approach:/component-gap:`; перед релизом смотреть
   `GET /api-backend/stats/feedback` (префикс `/api-backend` обязателен, scope
   stats:read); сигналы для команды KMP — сводкой в helm-реестр, не в разработку.

## 4. Текущий тираж — что сопровождать

**Программа Equipment parity** (Пётр, репо D:\projects\kmp): переданы
`mockup-match-report-v2.md` (редакция 2 — рекалиброванные вердикты) и
`agent-task-board-v2.md` (серийный конвейер, 11 промптов, ТЗ скрипта в §8).

Ближайшие шаги программы:
1. Пётр обновляет установку (`update=true`) — приезжают все навыки выше.
2. Jira-ключ программы (без него — ни ветки, ни MR).
3. A2 ∥ A3 (read-only research) — можно сразу.
4. **A1 — только после сброса Figma-квоты ~03–04.08** (промпт в борде v2).
5. G0 (`/start-task`) → human review доков → конвейер гонит скрипт
   (`codex exec`, проверено на codex-cli 0.142.2). Стоп-правило: не-PASS/
   NOT_APPLICABLE останавливает цепочку.
6. Device smoke в реализацию НЕ входит — это тестировщик, позже (планируется
   отдельный навык). Контроль вёрстки — браузерный таргет в суженном окне (D1).

**Как выкатывать правки навыков** (повторялось трижды, отработано):
PR от `github/main` (GitHub TacticumApps/dev — источник правды; GitLab —
зеркало, туда не мержить) → бамп версий по ADR-0009 (лейн + роль: рёбра ролей
пинуются при сиде, лейновый бамп требует бампа роли; зеркала тем же PR) →
реген только своих goldens (`E2E_INSTALL_REGEN_GOLDEN=1`) → после мержа на VPS:
pull + seed (runbook в Serena memory; rebuild для контента не нужен) → перепин
установок SQL-ом (`installations.profile_version_pinned`).

## 5. Открытые хвосты

- Ответ Саши по критерию hoisting `component:`-виджетов (урок 6) → потом
  одна правка лейна.
- Figma-хвосты после квоты: #747 (каталог Elements), #748 (чтение DQA findings),
  #744 второй проход.
- US #759 — enabler `installations.kb_repo_id` для аудита kb_cross_read (New,
  ждёт приоритета owner'а).
- Догон kb-navigation в 3 старых профилях (iva-rn-brownfield,
  iva-web-brownfield, tacticum-dev-base) — при следующем касании.
- brownfield-only файлы (tacticum-workflow, repo-configs) местами упоминают
  AppColors — мелочь, при следующем касании.

## 6. Гочи, которые сэкономят день

- **Пустой kb_discover — это не «нет KB»** (см. §3.3).
- **CRLF-фантомы:** массовые «падения» тестов/mirror-check на Windows — чаще
  всего CRLF в working copy при LF-индексе (`templates/** eol=lf`).
  Нормализовать, не «чинить» тесты. На чистом main всё зелёное.
- **После rebuild образа проверять MCP-транспорт** (`POST /mcp` → должен быть
  401, не 421): pip игнорирует uv.lock — rebuild может подтянуть новый мажор
  зависимости. `/healthz` этого не ловит.
- **Не восстанавливать UUID установок по памяти** — только whoami/БД.
- **Номера PR не угадывать** — брать из вывода `gh pr create`.
- В tacticum-dev обязательна Serena MCP для навигации/правок кода (CLAUDE.md,
  hard rule); в worktree-субагентах Serena запрещена (LSP цепляется к
  основному дереву).