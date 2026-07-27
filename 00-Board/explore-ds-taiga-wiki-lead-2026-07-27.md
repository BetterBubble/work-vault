---
title: Разведка ДС — Taiga
type: note
status: draft
created: 2026-07-27
tags:
- board
- design-system
- explore
- taiga
- wiki
permalink: tacticum/00-board/explore-ds-taiga-wiki-lead-2026-07-27
---

# Дом ДС в трекере и вики — что нашлось

Проверял лид (`lead-ds`) лично: воркеру MCP-тулы taiga/wiki не прокинуты, поэтому трекер и вики
взяты на себя. Всё ниже — чтение, ничего не создавалось и не менялось.

## Главная находка: эпик #540 «Design (Token Layer + Studio)»

В Taiga #12 (`tacticum-tacticum_dev`) есть **отдельный эпик по дизайну**, которого не было ни в
одном разборе созвонов. Это и есть кандидат на то самое «ещё где-то» из вопроса руководителя —
только не чужой дом, а **наш собственный, замысленный и незакрытый**.

- **Эпик #540 «Design (Token Layer + Studio)»**, проект #12.
- **PRD:** `docs/superpowers/specs/2026-06-18-design-studio-prd.md`, коммит `85fe489` от
  2026-06-18, **автор — Diaret (сам руководитель)**. Статус в шапке PRD: `ready-for-agent`.
- **Дизайн-спека:** `docs/superpowers/specs/2026-06-18-design-studio-design.md` · **ADR-0046**
  `docs/adr/0046-design-studio-agent-driven-authoring.md`.

**Что в PRD написано про Figma — дословно по смыслу:** дизайнеры Tacticum создают ДС во внешней
Figma + Tokens Studio, результат вручную экспортируется в DTCG и сидится в Design BC; это даёт
**vendor-lock на закрытый облачный инструмент** (против суверенности/Реестра ПО), нет
authoring-поверхности внутри продукта. Решение — Design Studio: раздел «Design» в Legends +
новый bounded context `studio/`, агент порождает ДС, токены сразу доступны dev-агентам через `design_*`.

**Статус реализации — ноль.** Проверено в `~/tacticum/tacticum-dev` (main):
- каталога `studio/` не существует (`find . -type d -name studio` — пусто);
- `grep -rn "design_studio" --include="*.py" apps/` — ни одного вхождения, то есть
  `source_type=design_studio` (модуль M7 из PRD) не заведён;
- все US эпика в статусе **New** (id 67): S3 #543 token emission adapter, S5 #545, S6 #546,
  S7 #547, S8 #548 (роль дизайнера + premium gating), S10 #550, S11 #551 (E2E «промпт дизайнера →
  published система видна dev-агенту»), S12 #552, S13 #553.

Вывод фактом: **Design Studio — замысел руководителя, оформленный PRD и ADR, но не начатый.**

## Что по ДС в трекере закрыто

- **US #696** «Дизайн-система tacticum-web для тенанта tacticum (DTCG + KB + скилл + сид в
  dogfood)» — **Done**, finish_date 2026-07-19, теги `design-system`/`dogfood`, входит в эпик #540.
  Состав по описанию: `design-systems/tacticum-web/` (tokens.json DTCG light+dark, design-system.yaml,
  DESIGN.md, preview.html, тесты на валидацию блоба и контраст WCAG AA), KB-пак, скилл
  `tacticum-design-tokens` в профиле `tacticum-internal-dev` 0.2.0, сид в org tacticum + attach.
  Спека `docs/superpowers/specs/2026-07-19-tacticum-web-design-system-design.md`, план
  `docs/superpowers/plans/2026-07-19-tacticum-web-design-system.md`, ветка
  `feature/tacticum-web-design-system`.
  **Важно:** это ДС для НАШИХ продуктовых веб-консолей (Dev/Legends, Helm, project-hub), а не для ИВА.

## Что по ДС в трекере открыто (статус New)

- **US #429** `ui-mockup-match` — runtime UI ↔ approved mockups diff loop (P2).
- **US #720** FRQA-S19 — мокапы для аналитика (`mockup-authoring` + `design-system-discovery` в
  `iva-fr-analyst`).
- **Task #489** `[S5] design-token-resolve` оракул + UI-token golden-таск.
- **Issue #302** `[E7-pilot] Defer Figma MCP — likely supersede by tacticum-design MCP (custom)` —
  то есть решение отложить Figma MCP в пользу своего уже принималось.
- **Issue #657** `installation_id: «omit по умолчанию» ломает kb_*/design_* на командных
  phk-токенах` — задевает именно тот канал, которым профиль тянет токены.

## Чего в трекере НЕТ

Задач с номерами #132 и #134 в Taiga под этими ref по дизайн-системе **не найдено** — это номера
**PR GitHub** `TacticumApps/tacticum-dev` (поймал воркер, пруфы — в его заметке). В постановке они
были названы задачами Taiga, это была ошибка формулировки.

**Не найдено задачи на схему обновления ДС** (Figma → репозиторий → профиль). Искал по запросам:
`design-system`, `Figma`, `дизайн`. То есть маршрут, который требует руководитель, в трекере не
заведён ни в каком виде.

**Владелец DS-задач в трекере не определён:** у всех найденных US/task/issue `assigned_to: null`.
Фактический владелец кода ДС виден только по git — Дмитрий Солонко.

## Вики tacticum — пусто

- `search_pages(tenant=tacticum, "дизайн-система токены Figma")` → пустой результат.
- `search_pages(tenant=tacticum, "design system")` → пустой результат.
- `list_pages(tenant=tacticum, limit=50)` → пустой результат.

То есть **по этому доступу в вики tacticum не видно ни одной страницы** — ни про ДС, ни вообще.
Причину не проверял: это либо ограничение прав на пути, либо тенант действительно пуст. Отличить
одно от другого этим инструментом нельзя — вопрос в списке «чего мы не знаем».

**Следствие:** описания процесса обновления дизайна в вики нет. Единственное место, где маршрут
описан связно, — наша память (`11-Directions/Направление- Единая дизайн-система…`) и разборы
созвонов, то есть документ, которого команды не видят.