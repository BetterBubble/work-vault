---
title: control-tower-v02 — чеклист выравнивания (канон = спека)
type: reference
permalink: tacticum/20-architecture/control-tower-v02-cheklist-vyravnivaniia-kanon-speka
tags:
- helm
- control-tower
- canon
- alignment
- architecture
---

Якорь выравнивания Helm на канон [[control-tower-v02]]. **Правило: каждая задача воркеру цитирует конкретный §/Шаг канона; лид гейчит результат против него; любое отступление — только явным ADR, не тихим дрейфом.** Команда сверяется с этим чеклистом, а не с пересказом.

## Что канон предписывает (по разделам + статус реализации)

- [spec] **§3 Модель** `Signal → Initiative` (владелец+срок+артефакт+класс) · Блок(1 ЕОЛ) · SalesInitiative(depends_on→Initiative, decomposes_to) · генезис цель∪sales∪эпик + ручной merge · оси Блок/Продукт/Поколение · зависимости Initiative→Initiative/Блок/Сервис · Т1/Т2/Т3. → ПОСТРОЕНО. #done
- [spec] **§4 Срочно×Важно** срочно=дедлайн−lead_time≤2мес (lead_time: ЕОЛ первичен, LLM черновик, НЕ в критпути 1a); важно=Σ(сумма×вероятность). → ПОСТРОЕНО. #done
- [spec] **§5 Монитор ГД** дерево Блок→ЕОЛ→инициативы · светофор+override(аудит) · Гантт · снапшот+diff(§5.3) · недельный бриф (детерминированный в 1a, LLM-проза позже через Gateway). → ПОСТРОЕНО. #done
- [spec] **§6.1 Разрывы** цель без работы · работа без цели · работа без денег · обещание без работы · срочно+важно без владельца · дубли. → 5/6 ПОСТРОЕНО; «дубли» = 1b (semantic_dedup). #done
- [spec] **§7 Данные / §13** штатка·зарплаты·CV·Jira·git+repomix·CRM·юнит-эк·Confluence + Т1/Т2/Т3 + карты идентичности/work→product. → 1a-часть (Jira+ручные) есть; остальное 1b. #partial
- [spec] **§8.2 Субстрат** Graphiti · LightRAG · knowledge_rag(Qdrant+Meili) · LLM Gateway · RE-переиспользование (KB-Brownfield-Bootstrap: topology/dedup/sbom/use_case/profile_gen/phases). → ОТКЛОНЕНИЕ (см. ниже). #gap
- [spec] **§9 Волны / §9.1 Шаги 0–9.** 1a=Шаг0+Шаг1(Jira)+Шаг2-lite+Шаг4-lite+Шаги7–8+Шаг9. 1b=полные Шаги1–9.

## Граница отклонения (точно, а не «всё под RAG»)

- [gap] §9.1 **Шаг 4**: «граф → в **Graphiti**; код+CV/Confluence → в **LightRAG/knowledge_rag**». То есть даже граф 1a по канону — в Graphiti. Мы построили его РЕЛЯЦИОННО в Postgres → отклонение №1, исправляем (Фаза 2). #gap
- [decision] Канон сам разводит субстрат по волнам: **Graphiti (граф) — с Волны 1** (Шаг 4) → добавляем сейчас. **Postgres — операционка веб-аппа** (§9 1a «FastAPI+Postgres+Alembic», §0.10 аннотации ЕОЛ Helm ведёт сам) → ОСТАЁТСЯ, не отклонение. **LLM Gateway (Шаг 5) · LightRAG/knowledge_rag (Шаг 4, код/CV) · RE (Шаг 3)** потребляют git/код/CV = **Шаги 1b** → приходят вместе с 1b. #decision
- [decision] Значит «переписывание» = НЕ big-bang под RAG, а канон-точная последовательность: **граф → Graphiti сейчас**; Gateway/LightRAG/knowledge_rag/RE — по 1b-шагам, по мере ввода git/кода/CV. Фазы 0→5 этому соответствуют. #decision

## Внутренние натяжки канона (беру осознанно)

- [tension] §8.2 «субстрат всё сразу с Волны 1» vs §9 1a «без обязательного LLM/идентичности» → субстрат-*граф* (Graphiti) с Волны 1; *фичи* 1a без LLM/RAG (светофор по ЕОЛ, бриф детерминирован); RAG/Gateway/RE на 1b-данных. #tension
- [tension] Два хранилища (не одно): Graphiti = граф/темпоральность; Postgres = операционка. По канону, не противоречие. #tension
- [tension] Тенант-изоляция в каноне НЕ прописана (§-пробел) → закрывается ADR по итогам разведки #24 (Graphiti-connect vs Postgres+AGE). #tension

## План фаз переписывания (роли)

- [plan] Ф0 разведка platform (Graphiti/Gateway/RAG + изоляция + RE) — @explorer #24 (идёт). Ф1 решение+ADR — лид. Ф2 граф→Graphiti|AGE (Postgres остаётся операционным) — implementer(s)+verifier(изоляция). Ф3 Gateway вместо мок-LeadTimeEstimator (чистый своп) — implementer. Ф4 knowledge_rag/LightRAG (Confluence/CV/код) — implementer(s). Ф5 RE-переиспользование — implementer(s)+verifier. #plan

## Отношения
- implements [[control-tower-v02]]
- relates_to [[Helm — 1b git-ingestion: скелет готов + спека real-adapter/identity-map]]
- relates_to [[agent-team-design]]
