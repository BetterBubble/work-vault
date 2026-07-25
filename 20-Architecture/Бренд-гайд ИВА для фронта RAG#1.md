---
title: Бренд-гайд ИВА для фронта RAG#1
type: note
permalink: tacticum/20-architecture/brend-gaid-iva-dlia-fronta-rag-1
tags:
- iva
- rag
- rag1
- frontend
- brand
- design-tokens
---

## Источник
Боевой CSS `iva.ru` (`variables.css`/`stylesheet.css`, логотипы SVG). Локальный корпус — стоковая тема Antora, бренда не несёт. Система Material-3 (тональные шкалы).

## Палитра (ключевое)
- **Primary green** `#21b175` (кнопки/ссылки/акцент), hover `#189662`, active `#148959`.
- **Secondary emerald** `#30b28b`; **Accent azure/cyan** `#32b9b9` (градиенты, «свечение»).
- Текст на primary `#fafcfb`.
- Фон: основной `#f0f0f4`, карточки `#ffffff`, low `#f8f8f9`, разделитель `#e7e8ef`.
- Тёмный UI: `#161618` / low `#1f1f21` (footer/dark-режим).
- Текст: основной `#161618`, вторичный rgba(22,22,24,.8), muted .6.
- Границы `#d6d7e1` / светлая `#e7e8ef`.
- Success = primary `#21b175`; **Danger** `#de3730`, danger-bg `#fcebea`.

## Логотип / «лепестки»
- Вордмарк «IVA Technologies»: `iva.ru/assets/templates/main/assets/img/logo.svg` (монохром, перекраска через fill/currentColor).
- Марка «IVA»: `.../logo-small.svg` (75×56) — **это и есть «лепестки»**: буквы I-V-A из заострённых миндалевидных форм-лепестков. Отдельного декоративного цветка на сайте НЕТ (мотив живёт в логомарке).
- Favicon/apple-touch/safari-pinned (brand `#21b175`).
- Применение лепестка в UI: акцент-глиф в hero, маркер пустых состояний чата, водяной знак фона (#21b175 @ 6-10%).

## Второй бренд-приём
«Зелёный glow на тёмном»: тёмный фон `#161618` + неоновое свечение `#21b175`→`#32b9b9` (градиент). `--glow-green: 0 0 48px rgba(33,177,117,.35)`.

## Шрифт
Основной сайта — **Guaruja Neue** (проприетарный, лицензии на веб у нас нет). Рекомендация: **Inter/Manrope** (геометрический гротеск, близкая метрика); Guaruja — если достанем лицензию.

## Скругления
0.4/0.6/0.8/1.0/1.2/1.6rem (≈4-16px), pill 9999px. Частые — 0.6 и 1.0rem. Тон: минимализм, воздух, мягкие тени, светлый фон + зелёный акцент; альт — тёмный с glow.

## design-tokens
Готовый CSS/`:root` и Tailwind `@theme` блок — в отчёте воркера (палитра+radius+font+glow). Primary `#21b175`, accent `#32b9b9`, danger `#de3730`, radius-lg 1rem, font Inter.

## Чего не хватает (уточнить скринами)
Живой скрин карточек/кнопок (тени/hover/паддинги), отдельный SVG-лепесток для декора (вырезать из logo-small), лицензия Guaruja, dark-mode маппинг поверхностей чата.

## Отношения
- part_of [[Концепт: три RAG для ИВА на общем движке]]
- relates_to [[RAG#1 — чеклист улучшений vs ЗУ (нюансы codex)]]
