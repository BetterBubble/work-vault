---
title: Инфра и доступы для RAG ИВА (ответы из Taiga)
type: note
permalink: tacticum/02-architecture/infra-i-dostupy-dlia-rag-iva-otvety-iz-taiga
tags:
- iva
- rag
- инфра
- доступы
- gateway
- vllm
- confluence
- taiga
---

## Что нашлось в Taiga по нашим открытым вопросам (2026-07-13). Wiki (tacticum/cifragen) — пусто по этим темам, всё в Taiga.

## Эмбеддер / LLM / Gateway (ЗАКРЫТО в основном)
- **Эмбеддер = Gateway `tacticum/embed` = bge-m3/1024** → делегирует в **tei_service `/v1/embeddings`**, provider-fallback DeepInfra→OpenRouter. Домен **llm.cifragen.ru** (LiteLLM proxy, Traefik TLS), авторизация Bearer virtual-key. (tacticum-platform US#29 id785, PRD #65 id904).
- **Свой Qwen (chat/embedding) — НЕ разворачивают**, только упомянут как суверенный провайдер Фаза 2 (deferred, platform #28 id784). → Для RAG эмбеддер = **bge-m3 через Gateway**, не Qwen.
- **vLLM `triva` (генерация в контуре ИВА):** доступ описан — туннель `systemd SSH 127.0.0.1:<port> → adp_emb → 10.0.196.14:9034`, клиент `IvaLlm=GatewayClient(model=triva)`, секрет `HELM_IVA_LLM_*` в `/opt/helm/.env`. (iva-control-tower Task #27 id1054, ADR-0003 = Task #34 id1061).
- **Reranker:** bge-reranker self-hosted TEI через Gateway `:4000` + project-hub-ключ; шортлист Meili top-30→top-8, graceful fallback (Task #32 id1059). **Точный JSON `/rerank` в Taiga нет — смотреть код/конфиг LiteLLM.**

## Серверы / инфра (ЧАСТИЧНО — мощного хоста нет)
- **adp_emb** — транзит контура ИВА (туннель к vLLM + Bearer-PAT к Confluence).
- **tei_service / LLM Gateway VPS** — 4 CPU / 7.8 ГБ / 17 ГБ (Postgres16+Redis7+Traefik). **Маловат для нашего RAG-стенда** (Qdrant+Meili+bge). Планируется перенос MSK→SPB + ребренд tei_service→LLM Gateway (#81 id955).
- **vLLM GPU / Model Serving — Фаза 3, deferred, GPU-хоста НЕТ.** → Выделенный мощный сервер под RAG всё ещё надо просить.

## Доступы к данным ИВА
- **Confluence (IVAPROJECT) тела+вложения:** путь ЕСТЬ — S5 ConfluenceConnector (helm #68 id1175, CQL lastModified, Bearer-PAT через adp_emb, инкремент-реиндекс) + MCP-1 (mcp-atlassian на ADP) + MCP Gateway `/iva-read` (platform #95 id1330, In progress).
- **eva-mcp (EVA/IVA Desk):** MCP-4 (#5 id1296) — статус **New, не поднят**. 6634 задачи/25 проектов, cookie-сессия headless на ADP.
- **Чаты/почты (RAG#3):** прямого источника в Taiga НЕТ. Ближайшее — ServiceDesk SD-RCA #60, S3 ServiceDesk-коннектор (Naumen).
- **iva-mcp токен:** rate-limit общего VPN-канала ADP (helm #111) — беречь при массовой выгрузке.

## OIDC / контур (ЧАСТИЧНО ЗАКРЫТО)
- **tenant `iva` уже есть в project-hub** (для MCP-ключей). MCP Gateway `/iva-read` пропускает только org/tenant=iva (tacticum → 403). RBAC по capability в project-hub (PRD #65). Отдельной задачи «project-hub тенант ИВА для фронта RAG#1» нет — но tenant iva существует.

## Temporal / LightRAG
- Temporal-RAG: на **Graphiti** (Memory ADR-0006, M4 Graph Builder), SD-RCA #60, Arch-KB #89. **LightRAG в Taiga не упоминается** — решения нет (если нужен — отдельно).

## Итог: что СНЯЛОСЬ и что ОСТАЛОСЬ к руководителю
- Снялось/сузилось: эмбеддер (bge-m3 через Gateway, не Qwen), доступ к triva (туннель описан), путь к Confluence (S5/MCP-1/iva-read), tenant iva для OIDC, reranker (bge через Gateway).
- Осталось: **выделенный мощный сервер под RAG** (tei VPS мал, GPU deferred); **eva-mcp поднять** (MCP-4 New); **чаты/почты RAG#3** (источника нет); **scope RAG#2 Jira** (какие проекты); Gateway-ключ для нас; точный `/rerank` из кода.

## Отношения
- relates_to [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[RAG#2 — дизайн: оффлайн-индекс + live-MCP подгрузка]]

## Выводы по хостингу и выгрузке (2026-07-13, обсуждение с пользователем)
- **RAG-у НЕ нужен GPU-сервер.** Все тяжёлые модели за сетью: эмбеддер bge-m3 → Gateway (`llm.cifragen.ru`, **ключ У ПОЛЬЗОВАТЕЛЯ ЕСТЬ**), генерация → vLLM triva через туннель, reranker → Gateway. Локально RAG нужен только Qdrant+Meili+BFF+фронт → **скромный CPU-VPS (2-4 CPU / 4-8 ГБ / ~40 ГБ)**, не дефицитный GPU-хост.
- **helm-сервер** (замер: 2 CPU / 3.8 ГБ RAM, диск 78% занято, крутится prod control tower) — годится как **сетевой мост в контур ИВА** (туннель к triva #27, adp_emb к Jira/Confluence), но как хост RAG-стека МАЛОВАТ: RAG#1 впритык, RAG#2 не влезет, рядом prod — риск. → отдельный CPU-VPS под RAG-стек, helm как мост.
- **Контур:** RAG#1 (публичное) + RAG#2 (Jira/Confluence — НЕ строго on-prem, брали не on-prem для helm) → могут жить у нас на CPU-VPS. Строго on-prem только RAG#3 (чаты/почты).
- **EVA — снимается:** ИВА переходят полностью на Jira → eva-mcp не нужен, Jira покроет RAG#2.
- **Массовая выгрузка Confluence НЕ через MCP:** iva-mcp идёт через узкий VPN-канал adp_emb (rate-limit, helm #111) + 1× токен-приём на контент. Для массового корпуса — **прямой Confluence REST с сервисным PAT через adp_emb** (как S5 ConfluenceConnector #68), дамп на диск, не через контекст модели. MCP — только для live-подгрузки точечно (RAG#2 роутер).
- **triva-туннель (#27)** сейчас НЕ поднят на helm — нужно поднять для генерации в контуре.

## Confluence — выкачиваем САМИ (подтверждено, паттерн в проде)
Прямой Confluence REST-дамп с adp_emb сервисным PAT (минуя iva-mcp) — **осуществим, доступ уже есть:**
- Транзит **helm→adp_emb** реален: alias `adp-jump` (ProxyJump) в `/home/tacticum-deploy/.ssh/config`, через него уже гоняются helm_refresh_repos / git-зеркала IVA. helm=10.16.0.20, platform=10.16.0.19.
- PAT: `/opt/iva-mcp/env` (CONFLUENCE_URL + CONFLUENCE_PERSONAL_TOKEN, read-only).
- **Рабочий скрипт-прототип уже есть и в проде:** `/opt/helm/scripts/extract_confluence_15.py` — `GET {base}/rest/api/content/{id}?expand=body.storage` с Bearer PAT, запускается НА adp_emb. Рядом семейство extract_jira_* — тот же паттерн прямого REST.
- Метод дампа: `content?spaceKey=IVAPROJECT&expand=body.storage,version,ancestors` (пагинация) + вложения `content/{id}/child/attachment` → download. Дамп на диск adp_emb → scp к нам. Бережно (конкурентность 1-2 + паузы, VPN узкий).
- **Блокер только процессный:** нужно явное разрешение пользователя на живой прогон (auto-mode классификатор блокирует SSH-заход в контур + сетевой пробник как «эксфильтрацию» — правильно). Live-досягаемость не проверена, вывод на прод-скриптах.
→ Вопрос к руководителю про Confluence-доступ фактически СНЯТ: доступ+скрипт есть, нужно только «ок на нагрузку» + оценка объёма space.

## vLLM triva — ТУННЕЛЬ ПОДНЯТ, генерация подтверждена (2026-07-13, отчёт руководителя)
- Прогон **ассистента требований** (US#24-26, трек Шульги) на vLLM `triva` (Gemma): **15 вызовов через туннель helm→adp_emb→triva прошли** → туннель #27 РАБОТАЕТ (раньше не был активен). Генерация в контуре ИВА доступна вживую.
- Результаты генерации (правильное требование в контексте): grounded 12/12, вердикты треков 12/12, 0 ложных «не нашёл», negative 3/3 корректный отказ, 0 галлюцинаций, ~1с. **Вердикт: triva/Gemma годится для Q&A как есть, свой Qwen не нужен** (совпадает с нашим выводом).
- **Главный оставшийся риск у них = RETRIEVAL** («найдёт ли поиск нужное требование; если принесёт не то — Gemma не спасёт; риск про поиск, не LLM»). → Это ровно НАША зона: reranker/hybrid/структурный чанкинг/метаданные/answer-eval закрывают этот риск. Синергия RAG#1/#2/ассистент — один движок-ретрив.
- Открытый вопрос про triva-туннель СНЯТ (работает). Свой Qwen — не нужен (подтверждено дважды).
