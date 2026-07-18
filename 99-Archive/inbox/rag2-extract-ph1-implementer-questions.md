---
title: RAG#2 extract Ф1 — вопросы имплементера перед стартом
type: note
permalink: tacticum/00-inbox/rag2-extract-ph1-implementer-questions
tags:
- rag2
- ingest
- extractor
- helm
- worker-question
status: archived
updated: 2026-07-18
---

Статус: **работа НЕ начата** (0%). Ветка/worktree для этой задачи ещё не было — только что создал (`~/tacticum-worktrees/helm-rag2-extract`, ветка `feature/rag2-extract-ph1`), кода/тестов пока нет.

Прочитал канон («RAG#2 pipeline выгрузки+ингеста — канон (для реализации)») + «пилот пройден» + существующие `src/helm/infrastructure/rag2/{ingest.py,confluence.py,extractors.py}` в helm. План модуля `src/helm/ingest/rag2_extract.py` готов, жду ответы на 4 вопроса ниже — дальше пишу код/тесты без пауз.

## Вопросы

1. **Ветка/worktree** — использую `~/tacticum-worktrees/helm-rag2-extract`, ветка `feature/rag2-extract-ph1` (готового worktree под задачу не было, назвал сам). Ок?

2. **Разрыв схемы changelog.** Канон требует `ch: {author,created,items:[{field,from,to}]}` (1:1 с сырым Jira `changelog.histories[]`). А текущий `_render_changelog` в прод-`infrastructure/rag2/ingest.py` ждёт **плоские** записи `{field,from,to}` без вложенного `items` (по одной смене поля на запись, без группировки по history-событию). Т.е. между Ф1-экстрактом и текущим прод-рендером текста — разрыв формата: если завтра просто скормить `ch` из моего экстрактора в `build_document`, changelog-текст в документе будет пустым (поля `f`/`from`/`to` не найдутся на верхнем уровне записи `ch[i]`).
   - Делаю строго по канону в новом экстракторе (не трогаю `ingest.py`, как вы просили).
   - Оставляю TODO-комментарий в коде на реконсиляцию (сплющить `ch` в момент подготовки к ингесту — видимо Ф2/3).
   - Ок так, или нужно что-то поменять сейчас?

3. **Epic-link customfield Jira.** В Jira Server/DC epic обычно лежит в кастомном поле (`customfield_XXXXX`), id специфичен для инстанса ИВА — без живого доступа не знаю точный id. Делаю параметром `jira_issue_to_record(..., epic_field: str | None = None)` + fallback на `fields.epic.key` (team-managed схема). `par` (parent) — отдельно, не путаю с epic. Ок?

4. **HR/archive space keys (HRHITECH, HR1, IS, LEGALIVA, birthday, org, KEYCLT, SSFYL, integrator/Реестр контрактов...) из канона** — НЕ хардкожу список в код (бизнес-решение может измениться, не хочу, чтобы жило магией в git). Передаю через `QualityConfig.exclude_spaces`/`--exclude-spaces-file`, дефолт пустой (плюс отдельно всегда фильтрую personal-пространства по префиксу `~` и 🔴 в названии — это универсально, не список). Ок?

## Если по всем четырём — «делай как предложил» / без ответа в разумный срок
Иду по дефолтам выше, фиксирую как принятые решения в сводке `RAG2_EXTRACT_SUMMARY.md` на ветке (коммичу туда же, где код).
