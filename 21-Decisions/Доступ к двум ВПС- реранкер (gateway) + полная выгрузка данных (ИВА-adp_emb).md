---
title: 'Доступ к двум ВПС: реранкер (gateway) + полная выгрузка данных (ИВА/adp_emb)'
type: note
permalink: tacticum/21-decisions/dostup-k-dvum-vps-reranker-gateway-polnaia-vygruzka-dannykh-iva-adp-emb
tags:
- access
- rag2
- rerank
- gateway
- iva
- adp_emb
- blocker
---

## Контекст
2026-07-15: руководитель даёт доступ к двум ВПС, чтобы снять оба остаточных блокера RAG#2 разом — реранкер и полную выгрузку данных. До этого оба висели как внешние блокеры (см. [[session-state]]).

## ВПС 1 — LLM Gateway
- **Хост:** `llm.cifragen.ru` → **155.212.134.20** (litellm, endpoint `/v1`). Живой (401 без ключа).
- **Задача:** реранкер `tacticum/rerank` — сейчас 403 (не в allowlist нашего ключа), `HELM_DOCS_RERANK_ENABLED=false`.
- **Что сделать:** в конфиге litellm (`config.yaml`/compose) проверить, что модель `tacticum/rerank` (бэкенд — TEI `bge-reranker-v2-m3`, репо `tei_service`) есть в `model_list`, и добавить её в разрешённые модели ключа/команды. Проверка: `curl https://llm.cifragen.ru/v1/rerank` с ключом → скоры, не 403. Затем в helm `HELM_DOCS_RERANK_ENABLED=true` + замер eval.

## ВПС 2 — ИВА / adp_emb (прод-прокси во внутреннюю сеть ИВА)
- **Хост:** `adp-jump` = **194.36.208.242** (user `tacticum`). Это точка входа во внутреннюю сеть ИВА (jira.iva.ru DC 10.3.15, wiki.iva.ru, vLLM triva 10.0.196.14:9034).
- **Задача:** полная выгрузка Jira+Confluence (сейчас частичный экспорт от 03.07: Jira ≤300 свежих/проект, Confluence — только индекс без тел). Индекс в проде держится на сэмпле: Qdrant `iva_confluence` = 9 точек, Jira-коллекции нет (fail-soft).
- **Форматы входа ingest** (отличаются от аналитических CSV в `data/real`):
  - **Jira** → `HELM_RAG2_TASKS_DIR`, компактный JSONL: `{k,type,st,cat,pr,an,al,comp,ep,rep,sum,desc,cr,up,links[]}`. Полностью — пагинация `startAt`/`maxResults=100` по 14 проектам.
  - **Confluence** → `HELM_RAG2_CONFLUENCE_DIR`, схема iva-mcp: `metadata{id,title,url,space{key,name},version,attachments[],content{value(markdown),format}}`. Тела через `expand=body.storage` + докачка вложений (CSV требований).
- Guardrail секретов (`redact.py`, merge #43) маскирует ключи/токены на ингесте — данные пойдут замаскированными.
- После выгрузки: `python -m helm.ingest.rag2_index` → сверка схемы → golden-eval на живом индексе. Опц.: полный `helm_mgmt_index` (SD ~43k) вместо 400.

## Порядок
Реранкер быстрее (~полчаса), выгрузка снимает главный блокер RAG#2. Доступы добавить в конфиг `ssh-manager` по аналогии с `helm`/`zu_demo`.

Связано: [[Инфра и доступы для RAG ИВА (ответы из Taiga)]], генерация — см. [[Gemini→Gemma (triva) — локальная генерация в контуре ИВА]].
