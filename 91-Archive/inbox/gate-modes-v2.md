---
title: gate-modes-v2
type: note
permalink: tacticum/00-board/gate-modes-v2-1
tags:
- draft
archived-at: 2026-07-31 17:27
---

# gate-modes-v2 — контроллер-гейт lead-modes (ТЗ#2), повторный прогон

Worktree: `/Users/bubblemac/tacticum-worktrees/modes-workflow` · ветка `feat/workflow-modes` · 12 коммитов main..HEAD.
Прогон после правок B1/B2/M2 + 1-й гейт в start-task + 2-й слой в run-implementation. Read-only.

## Вердикт: GO

Все 6 пунктов PASS. Блокеров нет.

## По пунктам

### 1. ГИТ-ЧИСТОТА — PASS
- `git status`: working tree clean.
- Ветка явная: `feat/workflow-modes` (не main).
- `git rev-list --count main..HEAD` = **12** коммитов (совпадает).
- Сообщения по сути: каждый коммит — content-scope (`feat(lite-base): align lite lane with proposal §1.2`, `feat(iva-analysis-base): встроить 1-й гейт классификации режима`, и т.д.). Автор — Александр Шульга.

### 2. SCOPE — PASS
`git diff --name-only main..HEAD` — все 26 файлов в ожидаемых путях, лишнего нет:
- `docs/proposals/workflow-modes/*` (3 файла)
- `templates/tacticum-{research,lite,bugfix,development-core}-base/*`
- `templates/iva-analysis-base/ingredients/commands/start-task.md`
- `templates/iva-role-{go,ios,java,kmp,mail,web,analyst}/manifest.yaml`
- `apps/backend/tests/catalog/test_iva_role_presets.py`

**Аддитивность прод-чувствительных правок — подтверждена (только вставки):**
- (a) `start-task.md` — коммит a2ec93d: numstat **50 insertions / 0 deletions**, затронут ТОЛЬКО start-task.md. Гейт добавлен, существующий хэндофф/blockquote не тронуты (0 содержательных удалений).
- (b) `run-implementation.md` — коммит a11c68e: numstat **36 insertions / 0 deletions**, затронут ТОЛЬКО run-implementation.md. 2-й слой добавлен, ack-gate последовательность не тронута.

**iva-analysis-base — тронут ТОЛЬКО `ingredients/commands/start-task.md`** (`git diff --name-only main..HEAD -- templates/iva-analysis-base/` вернул один файл). Агент tacticum-workflow не тронут, `_mirrors.yaml` не тронут.

Роль-манифесты (композиция лейнов): dev-роли go/ios/java/kmp/mail/web — +2/0 каждая (research + lite); analyst — +1/0 (research only). Чистые вставки.

### 3. ТЕСТЫ (без docker) — PASS
`uv run pytest test_manifest_schemas.py test_iva_role_presets.py test_role_replacement_parity.py`:
**211 passed / 0 failed in 3.66s.**

### 4. СЕКРЕТЫ — PASS
grep по диффу main..HEAD (api_key/secret/password/token/PRIVATE KEY/aws/bearer/sk-): 0 реальных секретов. Единственные совпадения на «token» — проза про экономию токенов («token economy», «tokens and time»). `.env`/ключей/бинарников/мусора нет.

### 5. AI-ПОДПИСИ — PASS
- В сообщениях коммитов (`git log --format=%B`): NONE.
- В диффе: NONE. Совпадения на «claude» — это имя платформы в manifest (`supports: [claude-code, codex]`, `claude-code: full`), легитимная конфигурация, не футер-атрибуция. Ни «Generated with», ни «Co-Authored-By», ни ссылок claude.ai/claude.com.

### 6. ЦЕЛОСТНОСТЬ МАНИФЕСТОВ — PASS
- lite-base `manifest.yaml`: **version 0.1.1** (bump подтверждён), CHANGELOG содержит `[0.1.1] — 2026-07-24` с описанием выравнивания по proposal §1.2 (lite = только refactoring-S / feature-S, bugfix убран в /fix-bug).
- research-base: version 0.1.0, консистентен.
- Все манифесты проходят test_manifest_schemas (в 211 passed).

## Резюме
Батарея пройдена полностью. Правки строго аддитивны на прод-чувствительных файлах (0 удалений), скоуп ровно по плану, тесты зелёные, ни секретов, ни AI-подписей. **GO.**