---
title: gate-modes-research-lite
type: note
permalink: tacticum/00-board/gate-modes-research-lite
tags:
- draft
---

# Гейт controller — lead-modes ТЗ#2 (research-base + lite-base)

Worktree: `/Users/bubblemac/tacticum-worktrees/modes-workflow` · ветка `feat/workflow-modes` (от main) · read-only прогон.

## Итоговый вердикт: **GO**

Все 6 пунктов PASS. Гит чист, скоуп ровно по ожидаемым путям, guardrail Diaret (iva-analysis-base) соблюдён, 211 тестов зелёные, 0 секретов, 0 AI-подписей, манифесты консистентны.

---

## Пункт 1 — ГИТ-ЧИСТОТА: **PASS**
- `git status` → `nothing to commit, working tree clean`.
- `git rev-list --count main..HEAD` → **8** коммитов (ожидалось 8).
- Сообщения по сути, conventional-стиль (`feat(lanes)`, `feat(roles)`, `feat(bugfix-base)`, `draft:`, `modes:`), с привязкой к ТЗ#2 / ADR-0057. Мусора/незакоммиченного нет.

## Пункт 2 — SCOPE: **PASS**
`git diff --name-only main..HEAD` = 24 файла, все в ожидаемых зонах:
- docs/proposals/workflow-modes/* (4: ingredient-mapping, lite-task-workflow.SKILL.draft, work-order-template.draft + skill draft)
- templates/tacticum-research-base/* (5) и templates/tacticum-lite-base/* (6)
- templates/tacticum-bugfix-base/ingredients/{commands/fix-bug.md, skills/bug-fix/SKILL.md} — companion-правки маршрутизации (manifest bugfix-base НЕ тронут)
- templates/iva-role-{go,ios,java,kmp,mail,web,analyst}/manifest.yaml (7)
- apps/backend/tests/catalog/test_iva_role_presets.py
- **Guardrail Diaret: iva-analysis-base НЕ тронут** → подтверждено (grep по diff пуст).
- **_mirrors.yaml НЕ тронут** → подтверждено (grep по diff пуст).
- Лишних путей нет.

## Пункт 3 — ТЕСТЫ (без docker): **PASS**
`uv run pytest tests/catalog/{test_iva_role_presets,test_manifest_schemas,test_role_replacement_parity}.py` →
**`211 passed in 3.79s`**, 0 failed / 0 error.

## Пункт 4 — 0 СЕКРЕТОВ: **PASS**
grep по добавленным строкам (.env/password/secret/api_key/token/ghp_/github_pat/PEM/AKIA/bearer) → только 2 совпадения, оба false-positive проза о «token economy» / «token and time cost». Реальных секретов/ключей/токенов нет.

## Пункт 5 — 0 AI-ПОДПИСЕЙ: **PASS**
- `git log --format=%B main..HEAD` grep (Generated with Claude / Co-Authored-By: Claude / claude.ai/code / claude.com/claude-code / Claude-Session) → `NONE_IN_COMMITS`.
- `git diff main..HEAD` тот же grep → `NONE_IN_DIFF`.

## Пункт 6 — Целостность манифестов: **PASS**
- research-base и lite-base: `kind` base-лейн, `version: 0.1.0`, **NO depends_on** (корректно для base), по 2 ингредиента (skill_spec + command_spec) с distinct ingredient_id (research/start-research; lite-task-workflow/lite-task). Валидность по схеме покрыта test_manifest_schemas (зелёный).
- Врезки в роли консистентны манифест ↔ ROLE_LANES:
  - 6 dev-ролей (go/ios/java/kmp/mail/web): depends_on = bugfix + lite + research — совпадает с ROLE_LANES.
  - analyst: depends_on = research-base only (ROLE_LANES = core-base + iva-analysis-base + research-base) — намеренная асимметрия (lite/bugfix = код-лейны, аналитику не нужны), совпадает с manifest.
  - parity/single-owner подтверждён test_role_replacement_parity + test_iva_role_presets (зелёные).

---

### Что чинить: ничего. Гейт пройден, можно двигать к тимлиду → OK Президента.
