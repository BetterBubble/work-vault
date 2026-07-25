---
status: draft
role: lead-fr
topic: ТЗ#3 US#4 Проход A — структурная разбивка батареи (для доклада президенту)
repo: /Users/bubblemac/tacticum/tacticum-dev
worktree: /Users/bubblemac/tacticum-wt/us4-conveyor-brd
branch: feat/us4-conveyor-brd
commit: 81de699
date: 2026-07-24
permalink: tacticum/00-board/pr-us4-passA-ready
---

# US#4 Проход A — структурная разбивка батареи (2 гардрейла + git)

Ветка `feat/us4-conveyor-brd` @ `81de699` от свежего origin/main `384997f`. НЕ пушу. 3 файла (+62/−2), все в каноническом `tacticum-dev-base`.

## 1. СТРОГО ПО ТЗ (fidelity) — critic вердикт (а) верно ТЗ §4 [[critic-us4-passA]]
| К-item | ТЗ §4 требует | Факт кода (brd-authoring/SKILL.md) | Verdict |
|---|---|---|---|
| **К-1** | «BRD берёт FT-n/UC-n из Части 1 (сейчас из П.A/П.B); переходный период — по маркеру» | v2: FT-n из §1.4, UC-n из §1.5 Части 1; v1: П.A/П.B по-старому; ID наследуются verbatim (не переномеровано) | ✅ точно |
| **К-5** | «маркер `fr_skeleton: 2` в шапке; отсутствие = v1» | детект маркера ДО чтения: `2`→v2, нет→v1 | ✅ точно |
| **К-2-brd** | «CT/DM/EV наследуются как FT/UC: BRD ссылается» (PIN/TESTS — downstream) | brd регистрирует CT-n(§1.6)/DM-n(§1.7)/EV-n(§1.8) по стабильному ID для наследования вниз; v1 — серий нет, шаг пропускается | ✅ точно (pin/tests — проходы B+) |

**Ничего вне ТЗ, ничего не потеряно** — КРОМЕ одного edge-кейса → см. флаг ниже.
⚠️ **ФЛАГ (догадка сверх ТЗ — на решение ГД/президента):** implementer добавил «битый/неоднозначный маркер → трактовать как v1 + пометка ассумпшна». В ТЗ этого НЕТ (ТЗ только «отсутствие=v1»). critic: безопасный backward-совместимый дефолт, keep допустимо. **KEEP (защитный дефолт) или REMOVE (строго ТЗ)?**

## 2. ПРОД-SAFE — controller подтвердил [[gate-us4-passA]]
- **Аддитивно:** ДА — v2 добавлено секцией+правилами; v1-путь дословно сохранён.
- **Backward-compat:** v1 (нет маркера) читает П.A/П.B «exactly as before» → прод не ломается (v2-FR в проде пока нет).
- **Тесты:** catalog (schemas + role_presets + install_smoke + tacticum_dev_base, --noconftest) → **218 passed / 0 failed**.
- **Дисциплина версий:** version-discipline --diff-against origin/main → **OK, 48 clean**; bump tacticum-dev-base **0.2.5→0.2.6** + CHANGELOG [0.2.6].
- **mirror-sync:** OK — 64 (НЕ изменился). N/A по замыслу: конвейер-скиллы (brd/pin/tests/start-task) НЕ в `_mirrors.yaml` — модель НЕ mirror, правился канонический владелец (подтверждено: tacticum-dev-base отсутствует в _mirrors.yaml).
- **Scope:** 3 файла только `tacticum-dev-base`. НЕ тронуты: монолиты (iva-web/kmp/mail/rn-brownfield), start-task, pin-authoring, tests-authoring, iva-analysis-base, прочие профили.

## 3. GIT-ЧИСТО — controller PASS
- **0 секретов** (.env/ключи/токены/PAT — none), **0 AI-подписей** (claude/generated/co-authored/anthropic — none), **0 мусора** (.venv в .gitignore).
- **Ветка от свежего main** `384997f` ✅; 1 коммит `81de699`; `git status` чист.
- **bump** 0.2.5→0.2.6.

## Итог
Оба гардрейла (строго-по-ТЗ + прод-safe) + git-чисто — PASS. Единственный открытый вопрос — edge-кейс битого маркера (keep/remove, решение ГД). Проход A первый на `tacticum-dev-base` (broadcast) — жду очередь + GO push.

## Связано
[[critic-us4-passA]] · [[gate-us4-passA]] · [[impl-us4-passA-brd]] · [[plan-us4-conveyor-execution]]
