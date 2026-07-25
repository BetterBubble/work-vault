---
status: draft
role: lead-fr
topic: ТЗ#3 v2-FR пилот на teststand — план лёгкого честного смоука (по образцу ТЗ#2)
repo: /Users/bubblemac/tacticum/tacticum-dev
teststand: 38.180.236.39 (codex-cli read-only, scratch /home/tacticum/tz3-pilot/)
date: 2026-07-25
permalink: tacticum/00-board/plan-tz3-pilot-teststand
---

# ТЗ#3 v2-FR пилот на teststand — план

**Подход (= ТЗ#2, [[pilot-tz2-teststand-results]]):** лёгкий честный смоук. Проверяем, что ИЗМЕНЁННЫЕ ТЗ#3 поведения ДОХОДЯТ до ожидаемого решения. Границы (ГД): только teststand-scratch, контент из **main #-файлов** (ssh_upload, НЕ прод-каталог), не деплой, честный вердикт (не имитировать), teardown после.

## Метод (как ТЗ#2)
1. `ssh_upload` тела скиллов из main в `/home/tacticum/tz3-pilot/` (scratch, не каталог).
2. `codex exec --skip-git-repo-check -s read-only < prompt` с acceptance-входом на каждый смоук.
3. Снять РЕАЛЬНЫЙ вывод codex, сверить с «зелёный =» критерием. Падение — честно диагностировать (не фейк-pass).
4. Teardown: `rm -rf /home/tacticum/tz3-pilot/` после, подтвердить `ls`.

## Файлы из main (для upload)
Аналитик (US#1-3): `templates/iva-analysis-base/ingredients/skills/fr-authoring/SKILL.md` + `.../api-contracts-discovery/SKILL.md` (+ DM/EV если отдельные).
Конвейер (US#4, канон-owner): `templates/tacticum-dev-base/ingredients/skills/brd-authoring/SKILL.md` + `.../pin-authoring/SKILL.md` + `.../commands/start-task.md`.

## Смоуки (5, зеркалят структуру ТЗ#2)

| # | Что проверяем (ТЗ#3) | Вход codex | 🟢 Зелёный = |
|---|---|---|---|
| **P1** | Аналитик генерит v2-FR (US#1-3): §1.4 FT-n/§1.5 UC-n + серии CT/DM/EV + плашки | fr-authoring + фича («экспорт отчёта в PDF» или реальная ИВА-фича) | маркер `fr_skeleton: 2`; §1.4 FT-n, §1.5 UC-n; ≥1 CT-n с плашкой «требует утверждения: разработчик + CTO» + Q-n; DM/EV где уместно |
| **P2** | Двухзонная честность (§2): Ч.2 as-is строгая, проектные wire-имена ТОЛЬКО под плашкой | тот же v2-FR из P1, анализ на честность | Ч.2 без невыдуманных wire-имён как факт; проектные имена в Ч.1 несут плашку+Q-n + основание в Ч.2 (не голое имя) |
| **P3** | brd читает v2-FR (К-1/К-5) | brd-authoring + v2-FR (маркер есть) | читает FT из §1.4, UC из §1.5, ССЫЛАЕТСЯ на серии CT/DM/EV; детектит маркер (не падает в v1) |
| **P4** | start-task гейт D-n (К-3) — честный BLOCKED | start-task + v2-FR (плашка+Q-n, БЕЗ D-n) | честный BLOCKED: имя раздела+Q-n, возврат владельцу, НЕ выдумывает контракт/не имитирует; вариант С D-n → proceed |
| **P5** | Backward-safe v1 (К-5) | brd/start-task + v1-FR (без маркера) | legacy-поведение; гейт НЕ срабатывает ложно на v1 |

*(pin К-2/4 — при бюджете: pin + v2-FR с D-n approved → реализация по стабильному ID + статус, FR↔KB расхождение→статус. Опционально, ядро — P1-P5.)*

## Что осознанно НЕ гоним (честно, вне скоупа)
- Полный downstream до реального кода/сборки (как ТЗ#2 — не гоним, конвейер-логика проверяется смоуком гейта/чтения, не билд KMP/Angular; билд-среды нет — см. [[pilot-runtime-env-finding]]).
- Провижн через прод-каталог (tacticum-mcp) — намеренно НЕ используем (там может быть старый контент; берём main-файлы).

## Вердикт (ожидаемый формат)
Таблица 5 смоуков (вход → реальный вывод codex → 🟢/🔴), честный итог: доходят ли изменённые ТЗ#3 поведения до ожидаемого решения. Падения — диагностировать, не прятать.

## Связано
[[pilot-tz2-teststand-results]] · [[summary-us4-done-vs-tz4]] · [[map-us4-remaining-to-prod]]
