---
title: critic-fidelity-axis2
type: note
permalink: tacticum/00-board/critic-fidelity-axis2-1
status: draft
tags:
- draft
- critic
- fidelity
- lead-modes
- axis2
archived-at: 2026-08-03 11:16
---

# critic/fidelity — ось-2 (A/B/C) в iva-kmp-brownfield @ ac30efc

> Отчёт critic/fidelity (записан тимлидом — у агента нет write-инструмента). Ветка feat/kmp-multirepo-axis2.

## Вердикт
- **A/B/C верны по ТЗ: ДА** (по спеке spec-axis2 + карте gapmap, в правильных местах).
- **Аддитивно/условно, без регресса: ДА** (вставки «skip entirely when no source repo», `${N:-…}`-форма, одно-древесный путь не тронут).
- **3-CLI синхронны: ДА** (claude↔copilot почти дословно, codex.toml конденсация, смысл консистентен).
- **Строго по ТЗ: ДА** (web не тронут, copilot-команда не добавлена, сервер/ось-1 не тронуты, лишних инвариантов нет).

## Пофакторно
- **A** ($3 source): argument-hint + блок Arguments с «Omit → behaviour unchanged» + условная hand-off; позиционная обработка (не splat). ✓
- **B** (read-only src + write tgt): во всех 3 телах блок «Cross-repo mode (additive; skip when no source)» + source-tree discovery со своим context.yaml/kb_discover/kb_run_id (read-only); write+ДС=target. Контракт «в источник не писать» НЕ скопирован — ссылка на скилл web-to-kmp-source-reference. ✓
- **C** (§3.0 два дерева): новый пункт 7 «Cross-repo trees», условный, перед Gate decision, во всех 3 телах; items 1-6 не переписаны. ✓

## Находки
- **BLOCKER/MAJOR:** нет.
- **MINOR** (copilot): `agents-copilot/tacticum-workflow.md:36` Inputs п.3 «passed via /start-task … [source-repo]» — copilot команду start-task НЕ поддерживает → неточно для copilot-тела. Смягчить: «provided by the invoking session». → ФИКСИМ.
- **NIT** (версия): bump 0.4.5→0.4.6 patch; новая (условная) возможность → semver ближе к minor 0.5.0. → лид: бампаем до 0.5.0.
- **NIT** (косметика): «never write»/«read-only» повторяется 3-4× на тело — плотность можно сократить. Пропускаем (ясность важнее).
- **Caveat:** verify-*.ps1 не прогнаны (нет pwsh); структурные, инвентарь команд не менялся → регресса не ждём, но фактической приёмки через них нет.

## Не-цели соблюдены
Сервер ✓ · ось-1 ✓ · web-симметрия не делалась ✓ · copilot-команда не добавлена ✓ · без сверх-ТЗ ✓. Маршрутизация обычного brownfield не изменена.

## Итог
**Готов к бандлу** после 2 штрихов (copilot MINOR + minor-bump). BLOCKER/MAJOR нет.