---
title: PR#48 feat/analyst-mcp-server — разрешение конфликта pyproject/uv.lock (merge
  origin/main)
tags:
- helm
- rag2
- mcp
- merge
- worker-report
date: 2026-07-15
permalink: tacticum/00-inbox/rebase-worker-mcp-resolve-1
status: archived
updated: 2026-07-18
---

# Задача

`origin/main` = `94fb29d` (PR#46 rag2-extract-ph1 и PR#47 rag2-extract-pdf-docx уже смержены). Остался PR#48 `feat/analyst-mcp-server` (origin `8471fc7`) с конфликтом в `pyproject.toml`/`uv.lock` (main принёс `pypdf`/`python-docx`, ветка добавляет `mcp[cli]`). Задача: влить `origin/main` мержем (не rebase), разрешить конфликт, НЕ пушить.

# Где сделано

Рабочее место: `~/tacticum-worktrees/helm-analyst-mcp-rebase`, ветка `migration/analyst-mcp-rebase-tmp` (её tip до мержа `8471fc7` == `origin/feat/analyst-mcp-server`, проверено `git rev-parse`). Использовал этот уже существующий temp-worktree вместо создания нового — содержимое идентично `origin/feat/analyst-mcp-server`. Основной репо `~/tacticum/helm` не трогал.

# Ход работы

1. `git fetch origin` → `origin/main`=`94fb29d`, `origin/feat/analyst-mcp-server`=`8471fc7`.
2. `git merge origin/main` — конфликт в `pyproject.toml` и `uv.lock`, как и предупреждал лид.
3. `pyproject.toml`: объединил зависимости вручную — оставил `mcp[cli]>=1.2` (из ветки) + `pypdf>=4.3`, `python-docx>=1.1` (из main). Ничего не потеряно.
4. `uv.lock`: конфликт не мержил руками — взял версию origin/main как базу (`git show origin/main:uv.lock > uv.lock`), затем `uv lock` — перегенерировал чисто поверх объединённого pyproject.toml. Добавились: `attrs`, `cffi`, `cryptography`, `httpx-sse`, `jsonschema(+specifications)`, `markdown-it-py`, `mcp==1.28.1`, `mdurl`, `pycparser`, `pyjwt`, `python-multipart`, `pywin32`, `referencing`, `rich`, `rpds-py`, `shellingham`, `sse-starlette`, `typer` (mcp-стек), при этом pdf/docx-депы (`pypdf`, `python-docx`, `lxml`) из main сохранились.
5. `git add pyproject.toml uv.lock` + `git commit --no-edit` (стандартное merge-сообщение).
6. `uv sync` — ок, доустановил `lxml`, `pypdf`, `python-docx` (4 пакета).

# Финальный SHA

`cde58ef` (merge-коммит `migration/analyst-mcp-rebase-tmp` ← `origin/main`)

Родители: `8471fc7` (MCP-ветка) + `94fb29d` (origin/main после PR#46+#47).

# `git diff origin/main..HEAD --stat`

Только новое из MCP-задачи, никаких удалений чужого:
```
ANALYST_MCP_SUMMARY.md                   | 123 +++++++++
pyproject.toml                           |   1 +
src/helm/interface/mcp/__init__.py       |   5 +
src/helm/interface/mcp/analyst_server.py | 396 +++++++++++++++++++++++++++
src/helm/main.py                         |  35 ++-
tests/interface/test_analyst_mcp.py      | 407 ++++++++++++++++++++++++++++
uv.lock                                  | 450 ++++++++++++++++++++++++++++++-
7 files changed, 1410 insertions(+), 7 deletions(-)
```

# Тесты / линтеры

- `uv run pytest -q`: **1 failed, 1362 passed, 12 skipped**.
- Упавший тест: `tests/ingest/test_rag2_harness.py::test_audit_attachments_classifies_statuses` — ожидает, что вложение `spec.pdf` классифицируется как `skipped_unsupported` (комментарий в тесте: `# skipped_unsupported (TODO)`), но фактически возвращается `empty_or_missing`.
- **Это НЕ следствие моего мержа и НЕ связано с MCP-веткой.** Перепроверил на чистом `origin/main` (94fb29d, отдельный detached worktree, без каких-либо моих изменений) — тест падает **точно так же**. Причина: PR#46 (`feature/rag2-extract-ph1`, содержит `test_rag2_harness.py` с ожиданием `skipped_unsupported` для pdf, т.к. на момент написания extractors.py pdf не поддерживал) и PR#47 (`feat/rag2-extract-pdf-docx`, добавил в `extractors.py` реальную поддержку pdf) — оба смержены в main по отдельности, и их пересечение осталось непротестированным: раз extractors.py теперь "умеет" pdf, `audit_attachments` для несуществующего pdf-файла классифицирует его как `empty_or_missing` (файла физически нет), а не `skipped_unsupported` (та ветка кода для "не умеем" больше не срабатывает на pdf). Это баг рассинхронизации ожиданий теста vs новой реализации, живущий уже в `origin/main` независимо от PR#48.
- Я его **не чинил** — вне периметра задачи (только pyproject/uv.lock конфликт по MCP), и правка теста/логики audit_attachments — решение не моё. Флагирую лиду отдельно.
- `ruff check` на изменённых MCP-файлах (`analyst_server.py`, `mcp/__init__.py`, `main.py`, `test_analyst_mcp.py`, `pyproject.toml`): чисто.
- `mypy` на изменённых MCP-src-файлах: чисто (`Success: no issues found in 3 source files`).

# Что НЕ сделано / требует внимания лида

1. **Push не делал** — как и просили, коммит `cde58ef` лежит только локально в `~/tacticum-worktrees/helm-analyst-mcp-rebase` на ветке `migration/analyst-mcp-rebase-tmp`. Пушить в `feat/analyst-mcp-server` на origin — задача лида.
2. ~~Pre-existing баг в origin/main~~ — **исправлено следующим коммитом** (см. апдейт ниже).

---

# Апдейт: фикс стейл-теста прямо в ветку #48

По просьбе лида — доложил фикс `test_audit_attachments_classifies_statuses` прямо в `migration/analyst-mcp-rebase-tmp`, чтобы мерж #48 сразу дал зелёный main (отдельный PR для этого было бы долго).

**Диагноз подтверждён**: логика `audit_attachments`/`extract_attachment`/`TODO_SUFFIXES` в коде корректна и не менялась — `TODO_SUFFIXES = {".doc", ".pptx", ".ppt", ".rtf"}` (pdf/docx там больше нет после #47). Баг был только в тесте: он использовал `.pdf` как пример «неподдерживаемого» типа, что устарело после #47.

**Правка теста** (`tests/ingest/test_rag2_harness.py`, только эта строка ожиданий):
- `spec.pdf` (пример unsupported) → заменил на `spec.pptx` (`.pptx` всё ещё в `TODO_SUFFIXES`, реально даёт `skipped_unsupported`).
- Добавил отдельный кейс `missing.pdf` → `empty_or_missing`, чтобы не терять покрытие: показывает, что pdf теперь **поддержан** extractors.py, но при отсутствии файла корректно классифицируется как `empty_or_missing` (не как `extracted` и не как `skipped_unsupported`).
- Прод-логику `audit_attachments`/`extract_attachment` **не трогал** — она верна, конфликта между «что тест ожидает» и «что код делает» после правки ожиданий больше нет.

**Коммит**: `217cf77` — `RAG#2 test-fix: audit_attachments — pdf теперь поддержан extractors.py (#47)`.

**Новый финальный SHA ветки**: `217cf77` (родитель — merge-коммит `cde58ef`).

## `git diff origin/main..HEAD --stat` (после фикса теста)

```
ANALYST_MCP_SUMMARY.md                   | 123 +++++++++
pyproject.toml                           |   1 +
src/helm/interface/mcp/__init__.py       |   5 +
src/helm/interface/mcp/analyst_server.py | 396 +++++++++++++++++++++++++++
src/helm/main.py                         |  35 ++-
tests/ingest/test_rag2_harness.py        |  11 +-
tests/interface/test_analyst_mcp.py      | 407 ++++++++++++++++++++++++++++
uv.lock                                  | 450 ++++++++++++++++++++++++++++++-
8 files changed, 1419 insertions(+), 9 deletions(-)
```

Только MCP-новое + точечный фикс стейл-теста в `test_rag2_harness.py`. Ничего чужого не удалено.

## Тесты / линтеры после фикса

- `uv run pytest -q`: **1363 passed, 12 skipped, 0 failed**. ✅ полностью зелёный.
- `ruff check tests/ingest/test_rag2_harness.py`: чисто (был E501 line-too-long на промежуточном варианте, поправил, теперь чисто).
- `mypy` (полный прогон по конфигу проекта, `files=["src","tests"]`, 440 файлов): **31 ошибка** в 7 файлах — все pre-existing и не в моих файлах (`scripts/seed_req_matrix.py`, `src/helm/application/req_matrix.py`, `src/helm/interface/api/routers/cio.py`, `tests/ingest/test_seed_req_matrix.py`, `tests/application/test_req_matrix_build.py`, `tests/llm/test_gateway.py`, `tests/interface/test_req_matrix_api.py`) — весь этот техдолг живёт в req-matrix/cio/gateway коде, унаследованном из main, вне периметра моей задачи. `tests/ingest/test_rag2_harness.py` и все MCP-файлы в списке ошибок **отсутствуют** — мой фикс чист.

Push не делал — коммит `217cf77` лежит локально на `migration/analyst-mcp-rebase-tmp` в `~/tacticum-worktrees/helm-analyst-mcp-rebase`.