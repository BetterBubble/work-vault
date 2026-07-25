---
status: draft
role: lead-fr
topic: ТЗ#3 US#4 Проходы D (web) + E (kmp) — карты (для окон ГД/ds)
repo: /Users/bubblemac/tacticum/tacticum-dev
date: 2026-07-24
permalink: tacticum/00-board/map-us4-passesDE
---

# US#4 D (web) + E (kmp) — карты раскатки

Оба — «полный профильный проход» (brd К-1/5 + pin/tests К-2/4 + start-task К-3/5) в один монолит. Модель НЕ mirror (ручной синк из канона + стек-специфика pin/tests/start-task). Дифф+версии ГД перед push. HOLD push до освобождения окна.

## Проход D — iva-web-brownfield (0.2.1, Angular/Nx)
| К-item | Файл | Действие |
|---|---|---|
| К-1/К-5 | brd-authoring/SKILL.md | синк канонического brd (как B1 mail/rn) — читает FR v2 §1.4/1.5 + маркер |
| К-2/К-4 | pin-authoring/SKILL.md | web-стек (Angular/Nx, angular-*, openapi-codegen): реализация CT/DM/EV + FR↔KB расхождения |
| К-2 | tests-authoring/SKILL.md | контрактные тесты `Covers: CT-n` под Angular/Karma/Jest |
| К-3/К-5 | commands/start-task.md | синк канонического start-task (plate-based гейт D-n + fr_skeleton) |
| версия | manifest 0.2.1→0.2.2 + CHANGELOG | |
**⚠️ ОКНО:** lead-ds активен на iva-web-brownfield (Сц.1/2) — по git его PR-ы копятся. **Строить D — из СНИМКА main после освобождения окна ds** (иначе moving-target/ребейз на чужие правки). Финальный push D — после окна ds (через ГД). Пока — карта готова, код по сигналу ГД «окно web свободно».

## Проход E — iva-kmp-brownfield (0.5.0, KMP)
| К-item | Файл | Действие |
|---|---|---|
| К-1/К-5 | brd-authoring/SKILL.md | ⚠️ kmp brd УЖЕ РАЗОШЁЛСЯ (axis-2, md5≠канон) → НЕ clobber; аккуратный merge дельты К-1/5 в diverged brd |
| К-2/К-4 | pin-authoring/SKILL.md | kmp-стек (Compose/KMP): CT/DM/EV + FR↔KB |
| К-2 | tests-authoring/SKILL.md | контрактные тесты под KMP |
| К-3/К-5 | commands/start-task.md | синк канонического start-task |
| версия | manifest 0.5.0→0.5.1 + CHANGELOG | |
**⚠️ ОКНО:** двойная контенция (lead-modes #145 axis-2 правил тело + lead-ds бампит kmp-dev-base). brd diverged. **Согласовать окно с ГД ДО правки** (kmp-brd делать ПОСЛЕ стабилизации axis-2, отдельным аккуратным проходом). Пока — карта готова, жду разводку ГД.

## Порядок (после C-канон+B2+C-монолиты)
D и E независимы друг от друга (разные профили), но каждый ждёт своё окно. Как окна освободятся (через ГД) — гоню профильный проход → батарея → дифф ГД → push. Композиты ios/firebird brd/start-task уже покрыты наследованием из канона (не в D/E).

## Связано
[[explore-us4-conveyor-scope]] · [[map-us4-passB-rollout]] · [[plan-us4-conveyor-execution]]
