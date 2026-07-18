---
title: 'Интеграция: тенанты project-hub'
type: note
permalink: tacticum/02-architecture/integratsiia-tenanty-project-hub
tags:
- integration
- tenants
- project-hub
- isolation
- rbac
---

# Интеграция: работа с тенантами Tacticum в своём сервисе

**Источник на диске:** `~/tacticum/fast_task/02-using-tenants.md`
**Тип:** инструкция/интеграция. Как использовать мульти-тенантную модель project-hub (тенанты, проекты, роли, изоляция). Предполагает настроенную auth (см. [[Интеграция: project-hub auth]]).

## Жёсткое правило платформы
Данные между клиентами (тенантами) строго изолированы и никогда не передаются между тенантами. Изоляция — ответственность каждого сервиса: hub отдаёт права, применяет их твой код.

## Модель данных
```
Tenant (клиент-организация)   slug уникален, immutable; soft-delete
  └── Project (проект)        slug уникален в рамках тенанта
        └── Membership (user↔scope↔role)
User — ГЛОБАЛЬНЫЙ (один email = один пользователь на всю платформу)
```
Тенант и права сервис получает из ответа `POST /api/internal/resolve`. Это прямой механизм fail-closed `tenant_id` в Codex (ADR-0001 D7/D11).

## Relations
- part_of [[02-Architecture]]
- relates_to [[Интеграция: project-hub auth]]
- relates_to [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- relates_to [[glossary]]
