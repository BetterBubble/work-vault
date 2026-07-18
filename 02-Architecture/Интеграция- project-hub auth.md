---
title: 'Интеграция: project-hub auth'
type: note
permalink: tacticum/02-architecture/integratsiia-project-hub-auth
tags:
- integration
- auth
- project-hub
- oidc
- codex
---

# Интеграция: Tacticum IdP (project-hub auth) в своём сервисе

**Источник на диске:** `~/tacticum/fast_task/01-using-project-hub-auth.md`
**Тип:** инструкция/интеграция. Как аутентифицировать пользователей в своём backend через project-hub (`https://master.cifragen.ru`).

## Главный принцип (architecture.md §7)
Приложение НЕ делает свой OIDC, не хранит ключи провайдеров, не изобретает аутентификацию. Единственный источник идентичности — project-hub. Сервис только **проверяет** токен у hub и применяет права.

## Модель токенов
project-hub выдаёт **непрозрачные (opaque)** токены `phk_<hex>` (НЕ JWT, локально по подписи не проверить — валидация всегда запросом к hub):
- **User token** `phk_…` — от лица пользователя (scope `mcp` или узкие read/write), приходит в `Authorization: Bearer`.
- **Service key** `phk_…` (`user_id IS NULL`, scope `resolve`) — ключ сервиса; сервис предъявляет его hub'у, чтобы валидировать user-токены.
- **Session cookie** `phub_session` — только для web-UI hub, не для service-to-service.
Хранение: argon2id-хэш, plaintext показывается один раз.

## Relations
- part_of [[02-Architecture]]
- relates_to [[Интеграция: тенанты project-hub]]
- relates_to [[ADR-0001 — Демо-стенд RAG ЗУ (Codex)]]
- relates_to [[План расширения M2–M4]]
- relates_to [[glossary]]
