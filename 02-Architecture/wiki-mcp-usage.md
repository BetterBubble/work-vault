---
title: wiki-mcp-usage
type: note
permalink: tacticum/02-architecture/wiki-mcp-usage
tags:
- wiki
- mcp
- architecture
- conventions
---

Правила работы с Wiki MCP (cifragen):

- Путь к странице ОБЯЗАН начинаться с префикса тенанта: "tacticum/..." или "cifragen/...". Относительные пути (например "platform/architecture") и лишний префикс "t/..." дают ошибку дизамбигуации ("ambiguous").
- get_page работает только с полным путём с префиксом тенанта.
- list_pages в auth-режиме возвращает пустой список (баг auth-фильтра по hub-membership) — НЕ полагаться на него.
- Обходной путь для поиска страниц: использовать search_pages (находит страницы корректно), не list_pages.
- При работе по платформе тенант = tacticum; по Залог Успеха = cifragen.

## Relations
- relates_to [[Tacticum Platform Architecture]]
- part_of [[02-Architecture]]
