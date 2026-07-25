---
title: Сводка сессии тимлида 2026-07-21 — 2C на проде, 2A+2B built (held)
type: report
permalink: tacticum/00-board/svodka-sessii-timlida-2026-07-21-2-c-na-prode-2-a-2-b-built-held
status: in-progress
role: lead (тимлид)
date: 2026-07-21
autonomy: 'off'
tags:
- summary
- lead
- iva
- session
- checkpoint
created: 2026-07-21
updated: 2026-07-21 16:48
---

# ЧЕКПОЙНТ тимлида — план iva-tri-deliveravla + Трек B (QA)

Репо: helm (2C) + tacticum-dev (2A/2B/QA). autonomy: off — до гейта, мерж по OK. Guardrail: analysis (iva-analysis-base/iva-role-analyst) — не трогать (можно с эскалацией). Всё, кроме 2C, живёт на ветке `feat/iva-write-base` (worktree `/Users/bubblemac/tacticum/tacticum-dev-iva-write`), НЕ мержено.

## Где стоим
- **2C my_todo** — ✅ на проде (verify PASS на живой БД, данные целы). Закрыт. [[2-c-my-todo-zadeploen-na-prod-verify-pass-na-zhivoi-bd]]
- **2A iva-write-base** — 🟡 built+валидирован (Вариант A), коммит `cf8dd61`. Держим. [[dizain-speka-iva-write-base-variant-a-na-apruv]]
- **2B три роли** — 🟢 built, test_iva_role_presets 35 passed, коммит `69eb440`. [[impl-2b-roles]]
- **Трек B QA** — 🟢 лейн `iva-qa-autotest-base` (9 скиллов) + `iva-role-qa` пересобран `[core,iva-qa-autotest-base,iva-write-base]`, тесты 35 passed, README+опись+слот defs, коммиты `5df061c`+`8a7e854`. Документация: [[qa-two-layers — два слоя QA, что готово, зачем ждём 3 субагента]] · [[qa-profile-model — опись + мульти-стэк модель QA-лейнов]] · [[qa-validation-plan — валидация iva-role-qa 1-в-1 (чек-лист)]]

## Следующий шаг
- Ждём внешние входы (ниже). Активной работы «до результата» нет.
- Как придут решения → финализировать 2A (URL/auth) → controller-гейт по всей связке → мерж (OK пользователя) → провижн.
- Как придут 3 agent_spec QA → вшить → пересобрать+тест → валидация 1-в-1 по чек-листу → раздача QA.

## Заблокировано (внешние входы)
1. Грилл **ADR-0058** (auth-модель iva-write, серверная).
2. **A/C мульти-ключ** — согласование с руководителем (provisional=A). [[resh-enie-provisional-iva-write-multi-kliuch-variant-a]]
3. **Монахов** — IVAREQ + Confluence-space + write-endpoint (PoC 2A + прод).
4. **3 agent_spec** от QA-команды (codebase-analyst/dom-explorer/code-writer) — оживляют 3 из 9 QA-скиллов.

## Отсеяно (не делать)
- Мульти-ключ Вариант C (расширение схемы+6 рендереров) — пока не нужен.
- Переносить tests-authoring в QA — НЕТ (развилка «кто генерит TC» за Diaret).
- Трогать analysis без эскалации; мержить без OK; PoC до появления endpoint.

## Ключевые факты
- Прод helm: ssh `helm`, деплой `/opt/helm/scripts/deploy.sh` **строго SEED=0** (иначе синтетика затрёт данные). Бэкап `/tmp/helm-predeploy-mytodo-*.sql.gz`.
- QA-скиллы жёстко на one-web (autocore/testops/glab) — репо-специфичны; iOS/KMP = отдельные лейны.
- iva-write Вариант A: 1 клиентский phk_ + подпись актора gateway + серверные downstream-креды.

## Ссылки
- Планы: [[plan-tri-deliveravla-iva-iva-write-3-roli-my-todo-priviazka-k-sisteme]] · [[plan-qa-profile — обогатить iva-role-qa реальными QA-скиллами (Трек B, лиду)]]
- Отчёт QA для ГД: [[qa-profil-trek-b-gotovo-built-verified-flagi-dlia-gd]]