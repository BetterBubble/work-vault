---
title: Ф3 — constraints / contradiction_check — SUMMARY воркера
type: note
tags:
- mcp
- rag2
- analyst
- constraints
- handoff
permalink: tacticum/00-inbox/constraints-tools-summary
---

# Ф3 — тулы constraints / contradiction_check — хендофф тимлиду

Воркер-implementer, 2026-07-16. Ветка `feat/mcp-constraints` от `feat/mcp-actions-triad`,
worktree `~/tacticum-worktrees/helm-mcp-constraints`. Коммит `3a08745`.

## Что сделано

Добавлены два тула в `src/helm/interface/mcp/analyst_server.py` по паттерну `nearest_spec`
(поиск rag2 + фильтр source=confluence), плюс обновлены `instructions` MCP и
`test_all_tools_registered`.

### `constraints(ctx, requirement_text, k=5)`
- Семантический поиск по confluence-корпусу → релевантные ADR/НФТ/реестры «нельзя».
- Возвращает `{query, constraints: [{title, space, url, snippet, kind}], note}`.
- `kind` ∈ {adr, нфт, ограничение, прочее} — эвристика `_constraint_kind` по заголовку+спейсу:
  adr = «adr»/«развилк»/«архитектурн…решени»/ID вида `S2-D-08`; нфт = «нфт»/«нефункци»;
  ограничение = «нельзя»/«ограничени»; иначе прочее. Приоритет adr→нфт→ограничение→прочее.
- `snippet` — фрагмент из payload text, центрированный на маркере сути ограничения
  (ограничени/fallback/нельзя/не входит/не должн/запрещ/нфт), окно ~400 симв. Текст НЕ выдумывается.
- `note`: «Ограничения найдены семантически; проверьте применимость к вашему требованию.»

### `contradiction_check(ctx, requirement_text, k=5)`
- Тот же поиск, результат подаётся как КАНДИДАТЫ: `{query, candidates: [{title, space, url,
  snippet, why_candidate}], note, disclaimer}`.
- `why_candidate` = «может конфликтовать с <архитектурным решением/НФТ/ограничением>: <тема>».
  Честная формулировка, НЕ утверждение факта.
- `disclaimer` (дословно): «Это КАНДИДАТЫ на противоречие, найденные по близости. НЕ вердикт.
  Требует проверки аналитиком/архитектором. Отсутствие кандидатов ≠ отсутствие противоречий.»
- Пустой список → `note` честно «кандидатов не нашлось, это НЕ значит, что противоречий нет».
  НИКОГДА не отдаёт вердикт «чисто/противоречит». В структуре нет полей verdict/contradicts.

### Технические детали
- **Тело хита берём напрямую из оркестратора** `rag2_router._answer(ctx, body)` через
  `run_in_threadpool` — тот же путь, что REST `search` использует внутри. Причина: REST-морда
  `Rag2HitOut` НЕ несёт `text`, а для честного snippet нужен payload text (`JiraDoc.text`).
  Логику ретрива не дублируем — зовём ровно `_answer`.
- **ACL**: `_acl_blocked_space` отсекает спейс `IS` и личные (`~...`) из выдачи обоих тулов.
- Запас пула `k*5` (федеративная выдача мешает источники + ACL-фильтр отсекает часть).
- RAG#2 не сконфигурирован → `ValueError` (как остальные тулы). Пустой текст → `ValueError`.

## Диффы (кратко)
```
 src/helm/interface/mcp/analyst_server.py | 193 +++++++++++++++++-  (хелперы + 2 тула + instructions)
 tests/interface/test_analyst_mcp.py      | 149 ++++++++++++++++    (+2 в registered, 14 новых тестов)
 2 files changed, 341 insertions(+), 1 deletion(-)
```

## Статус тестов
- **`pytest tests/interface/test_analyst_mcp.py` — 55 passed** (было 41 + 14 новых).
- ruff — clean; mypy по analyst_server.py — 0 ошибок (19 mypy-ошибок в cio.py — предсуществующие, не мои).
- Новые тесты покрывают: регистрацию тулов, эвристику `_constraint_kind` (вкл. ID `S2-D-03`),
  ACL-фильтр (IS + `~...`), центрирование сниппета на маркере, пустой сниппет;
  constraints (adr+нфт в выдаче, jira отфильтрован, snippet несёт суть, note),
  ACL-фильтрацию спейсов, requires_text, not_configured;
  contradiction_check (кандидаты не вердикт, why_candidate «может конфликтовать», disclaimer
  дословно, отсутствие полей verdict/contradicts; пустой ≠ «чисто»; requires_text).

## ⚠️ ЖИВАЯ ПРОВЕРКА — ЗАБЛОКИРОВАНА ПОЛИТИКОЙ (нужно действие пользователя)

Живую проверку на реальном корпусе в `helm-helm-1` я выполнить НЕ смог: авто-режим Claude
Code блокирует и `docker exec` в прод-контейнер, и создание скрипта с монкипатчем
`_require_principal` (auth-bypass) — по политике это требует явного подтверждения ЧЕЛОВЕКА,
а мандат тиммейта такого барьера не снимает. Я НЕ обходил блок.

**Что нужно от тебя/пользователя** — запустить живую проверку самому (подтвердив прод-доступ).
Готовый однократный скрипт (положить, напр., в `/tmp/live_check_constraints.py` контейнера
или запустить через `python -c`), обход auth монкипатчем, только чтение rag2.search:

```python
import asyncio, json
from types import SimpleNamespace
from helm.interface.mcp import analyst_server
from helm.interface.api.main import app

class _Ctx: request_context = SimpleNamespace(request=None)
analyst_server.configure(app)
async def _bypass(_c): return SimpleNamespace(email="live@check", tenant="iva")
analyst_server._require_principal = _bypass  # обход Bearer/resolve для внутр. проверки

REQS = [
    "MFE должен читать состояние Shell через React Context",
    "система должна работать только на Windows Server",
    "почта работает напрямую по JUMP из Shell core",
]
async def main():
    for r in REQS:
        print("="*80, "\nТРЕБОВАНИЕ:", r)
        print("[constraints]", json.dumps(
            await analyst_server.constraints(_Ctx(), requirement_text=r, k=5),
            ensure_ascii=False, indent=2))
        print("[contradiction_check]", json.dumps(
            await analyst_server.contradiction_check(_Ctx(), requirement_text=r, k=5),
            ensure_ascii=False, indent=2))
asyncio.run(main())
```

Запуск: `docker exec -i helm-helm-1 python < этот_файл`  (или скопировать в контейнер и запустить).

**Что проверить фактически по выводу (критерии приёмки лида):**
1. `constraints` для React-Context-требования → непустой список с реальными ADR/НФТ; среди
   kind есть `adr`/`нфт`; snippet несёт суть из payload (не только заголовок).
2. `contradiction_check` для того же → в кандидатах всплывает что-то про **S2-D-03 / MFE /
   React Context / Context Bus**; каждый `why_candidate` начинается «может конфликтовать»;
   `disclaimer` на месте; НЕТ вердикта.
3. Для Windows-Server-требования → ожидаемо кандидаты про НФТ платформы (Astra Linux SE
   «Ленинград»/Эльбрус, PRMAS 64985720) — конфликт с «только Windows».
4. Для JUMP-требования → ожидаемо S2-D-08 «JUMP не входит в целевой SDK/Shell core».
5. Убедиться, что в выдаче НЕТ спейсов `IS` и личных `~...` (ACL).

Как только запустишь — приложи реальный вывод; если что-то не всплывает (например, ADR-спейсы
IVACORE/IM не проиндексированы в корпусе — открытый вопрос из discovery-заметки), это сигнал,
что слой ограничений не на чем строить, и надо чинить покрытие (Ф0), а не тулы.

## Риски / открытые вопросы
- **Покрытие ADR-спейсов в корпусе не подтверждено вживую** (discovery-заметка отмечала это
  как открытый вопрос: IVACORE/IM/PRMAS/IVAQA — проиндексированы ли?). Юнит-тесты зелёные на
  моках, но зелёные тесты ≠ реальные данные — живая проверка обязательна перед приёмкой.
- Эвристика `kind`/`why_candidate` грубая и детерминированная (без LLM) — намеренно; при желании
  можно усилить, но это меняет контракт «только фактура, без генерации».
- Деплой не делал (по инструкции — тимлид патчем после ревью).