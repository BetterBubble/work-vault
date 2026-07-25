---
title: QA-профиль (Трек B) — готово (built+verified), флаги для ГД
type: report
permalink: tacticum/00-board/qa-profil-trek-b-gotovo-built-verified-flagi-dlia-gd
status: built-verified-held
role: lead (тимлид)
date: 2026-07-21
autonomy: 'off'
tags:
- summary
- qa
- role-presets
- trek-b
- for-director
---

# QA-профиль (Трек B) — сделано, верифицировано, held

## Что сделано
- **Лейн `iva-qa-autotest-base`** (leaf, 9 skill_spec, 0 mcp_server_spec) — перенесены 9 реальных QA-скиллов автотест-команды + 35 references. Привязка к репо `one-web` зафиксирована честно (`stack.required: [one-web]`, non_goals — не агностичный).
- **`iva-role-qa` пересобран** → `[tacticum-core-base, iva-qa-autotest-base, iva-write-base]` (analysis убран), v0.2.0, суть = исполнение/автоматизация автотестов (не авторинг TC).
- Ветка `feat/iva-write-base`, коммит `5df061c` (51 файл). Не мержено (autonomy off).

## Проверено (verifier, реальные прогоны)
- `test_iva_role_presets.py` → 35 passed; `test_manifest_schemas.py` → 38 passed.
- Целостность: 9 скиллов ↔ 9 директорий ↔ 9 SKILL.md, references=35, дизъюнктность core∩qa-autotest∩write=∅. Guardrail чист.

## ФЛАГИ — требуют действий ГД / QA-команды
1. **3 недостающих субагента** — `codebase-analyst`, `dom-explorer`, `code-writer` (зовутся из write-autotest/batch-autotest/fix-failed-test через Task). В архиве их НЕТ → **нужен agent_spec от QA-команды**, иначе 3 из 9 скиллов функционально неполны. (Файлово/композиционно лейн собран, тесты зелёные — это про рантайм.)
2. **Привязка к one-web** — лейн репо-специфичный (autocore/venv/glab/CI), не стэк-агностичный. Осознанно зафиксировано; если нужен агностичный вариант — отдельная работа.
3. **Docs ≠ скиллы** — 2 docs описывают тест-ДИЗАЙН (Qwen→IVA QA Agent→TestOps, PoC −75%), а 9 скиллов = автотест-КОД. Лейн покрывает слой кода; Qwen/TestOps-пайплайн — отдельный вопрос.
4. **codex:full расхождение** (низкий риск) — роль заявляет `codex: full`, лейн best-effort; тест не ловит (не считает min-over-lanes). Follow-up: понизить codex роли ИЛИ починить тест.
5. **Развилка «кто генерит TC»** (аналитик vs QA) — НЕ решена, за ГД/созвоном. Пока QA без analysis (без requirement_tests/tests-authoring).

## Связано
- [[impl-qa-autotest-base]] · [[verify-qa-autotest-base]] · [[explore-qa-autotest-skills]] · [[resheniia-po-qa-profiliu-trek-b-2026-07-21]]