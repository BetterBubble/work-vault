---
title: Решения по 2A iva-write-base (2026-07-21)
type: note
permalink: tacticum/00-board/resheniia-po-2-a-iva-write-base-2026-07-21
status: approved
role: lead (тимлид)
date: 2026-07-21
tags:
- decision
- iva-write
- 2a
---

# Решения пользователя по 2A iva-write-base

1. **Аутентификация — параметризуем** (сменный блок), финал — по исходу сегодняшнего грилла ADR-0058 (техучётка `iva` vs личный PAT #712). НО:
2. **Мульти-ключевая схема — обязательное требование дизайна**, продумать глубоко и реализовать: **hub-ключ (phk_*/актор) + PAT Jira/Confluence + Allure-PAT + tactic**. Прецедента в манифестах нет (`auth_type` скаляр) → нужна новая конструкция. Это ядро задачи 2A.
3. **Композиция — аддитивно:** новый лейн `iva-write-base` с отдельным `mcp_server_spec` id `iva-write`. **analysis НЕ трогаем** (guardrail).
4. **Скоуп сейчас:** дизайн-спека + **PoC на песочном Jira/Confluence**.
5. **allowed_tools — сузить до write-тулов** (confluence_create/update_page, jira_create_issue/add_comment, +transition, +Allure).

## Связано
- [[explore-iva-write-base]] · [[konflikt-2-a-iva-write-adr-0058-lichnyi-pat-razvesti-do-dizaina]] · [[plan-tri-deliveravla-iva-iva-write-3-roli-my-todo-priviazka-k-sisteme]]