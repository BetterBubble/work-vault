---
title: recon-demo-roskosmos-rag2
type: report
permalink: tacticum/00-board/recon-demo-roskosmos-rag2
tags:
- recon
- demo
- rag2
- analyst
- roskosmos
archived-at: 2026-07-29 18:12
---

# Recon: демо-готовность RAG#2 /analyst (Роскосмос)

- Снято: 2026-07-22 (день, МСК), read-only, прод не тронут
- repo: helm · ветка: main · сервер: helm (159.194.233.33, /opt/helm)
- Контейнеры: helm-helm-1 (Up 15h), traefik (12d), postgres (healthy). Qdrant/Meili — внешние (10.16.0.19).

## 1. /analyst — ЖИВ
- Страница https://helm.tacticum.ru/analyst → HTTP 200. API /api/rag2/context up, auth-gated (401 без bearer — так и надо; AUTH_REQUIRED=true, tenant=iva).
- Бэкенд rag2 проверен через MCP analyst_context (тот же in-process handler): degraded=false, mode=index, answerable=true. Демо-сценарий «статус фичи SSO» отдал корректный цитируемый контекст (Confluence-требования + закрытая Jira VCSWEB2-4799 + helm_mgmt requirement).
- ⚠️ ГЛАВНОЕ: AnalystChat.tsx показывает `a.context` НАПРЯМУЮ (retrieval-блок с цитатами [n]), БЕЗ LLM-синтеза. Веб /analyst — это «цитируемый поиск», НЕ «бот выдаёт план прозой». Генерация ответа по контексту делается внешним агентом (MCP-клиент), не веб-мордой. triva в путь /api/rag2/context НЕ подключён.

## 2. Что проиндексировано (Qdrant bge_m3_1024)
- iva_jira 325 155 чанков (~8119 задач) ✅
- iva_confluence 102 226 ✅
- helm_mgmt 400 · helm_requirements 1465 · iva_docs 8272 (RAG#1) · knowledge 80 274
- Граф зависимостей (networkx по Jira-линкам) — включён (rag2_graph_enabled) ✅
- Allure — в Qdrant НЕТ; живёт как отдельный MCP-тул requirement_tests (снапшот as_of 2026-07-22, свежий) — но в веб /analyst НЕ входит ❌
- Топология/C4 — в Qdrant НЕТ; отдельный MCP-тул arch_map (systems/owners/risk/edges, работает) — в веб /analyst НЕ входит ❌
- Веб-контекст = Jira + Confluence + helm_mgmt + граф Jira-линков. as_of Jira/Confluence = 2026-07-10 (~12 дн).

## 3. triva/Gemma — HEALTHY (риск №1 сейчас зелёный)
- Туннель 127.0.0.1:8790 / 172.18.0.1:8790 жив (ssh pid 75397). /v1/models OK.
- Тест chat/completions → «Работает.», HTTP 200, латентность 0.33s. Модель /models/triva_llm_instruct, ctx 65536.
- Нюанс: triva нужен для /docs/ask (RAG#1) и для синтеза MCP-агентом; в веб /analyst не задействован.

## 4. Docs (RAG#1) фолбэк — ЖИВ
- Страница переехала с /docs на **/iva-docs** → HTTP 200 (/docs=404 это Swagger, скрыт в проде — норма). Публичный /docs/ask существует (мой POST дал 405 — метод/путь; эндпоинт есть).

## 5. Гэпы до демо
- **Экспектейшн-мисматч (критично):** веб /analyst отдаёт цитируемый retrieval, а не сгенерированный «план: сделано/не сделано/кому задача». Для «бот даёт план» нужен LLM-шаг, которого в веб-морде нет. Варианты без правки прода: (а) демить retrieval-с-цитатами и проговаривать вывод вслух; (б) демить через MCP-клиента (Claude синтезирует) — но это не «сайт».
- **Топология + Allure не в веб-ответе.** Если Роскосмосу обещан «корпоративный RAG с топологией и тестами» — в веб /analyst их НЕТ (только Jira+Confluence+граф). Доступны лишь как MCP-тулы arch_map/requirement_tests.
- **Auth:** нужен рабочий вход в tenant=iva для презентующего (страница за логином дэша). Проверить креды до 16:00.
- **Свежесть:** Jira/Confluence as_of 2026-07-10 (дисклеймер показывается) — для демо ок.

## Вердикт
GO — если демо позиционируется как «цитируемый поиск по Jira+Confluence с графом зависимостей» (все зависимости зелёные: triva, Qdrant, Meili, корпуса).
NO-GO как «бот синтезирует план с топологией и Allure» без нарратора/MCP — веб-морда этого не делает.

- [ ] to:director from:recon GO как retrieval-с-цитатами (все деп-сы зелёные, triva 0.33s); NO-GO как «бот-план с топологией/Allure» — веб /analyst не синтезирует и не включает C4/Allure, нужен нарратор или MCP-клиент
