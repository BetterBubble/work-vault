---
title: gate-deploy-set-prod
type: note
permalink: tacticum/00-board/gate-deploy-set-prod
tags:
- gate
- controller
- prod-seed
- deploy-set
- combined
- tz1
- tz2
- tz3
---

# Гейт контролёра: DEPLOY-SET прод-сида всех 3 ТЗ — ВЕРДИКТ GO

> Независимая read-only проверка репо-стороны против **main `d70c68a`** (HEAD подтверждён). Живой прод-каталог/коллизии — DB-сторона, закрывал lead-modes запросом к БД. Контролёр ничего не правил.

## ВЕРДИКТ: GO

22-пакетный deploy-set = ровно наши 3 ТЗ (версии совпадают с main d70c68a), чужого лага нет, C-исключение безопасно, orphan нет, роль-дедуп ок.

## Пункт за пунктом

**1. Версии 22 пакетов (manifest == заявленное) — ПРОШЛО.** Все 22 прочитаны из `d70c68a:templates/<pkg>/manifest.yaml`, совпадают 1-в-1: research-base 0.1.0, lite-base 0.1.3, bugfix-base 0.1.3, development-core 0.1.1, iva-web-development-base 0.1.2, kmp-development-base 0.7.0, iva-web-brownfield 0.5.1, dev-base 0.2.7, fr-analyst 0.1.12, brownfield-mail 0.7.4, rn-brownfield 0.5.4, ios-brownfield 0.1.3, firebird-web-brownfield 0.1.3, analysis-base 0.1.7, kmp-brownfield 0.5.2, role-go 0.2.1, role-{ios,java,kmp,mail,web} 0.1.1, role-analyst 0.1.2. Несоответствий нет.

**2. Версии-трейс (чужого лага нет) — ПРОШЛО.** Пройдены CHANGELOG:
- `iva-kmp-development-base` 0.1.0→0.7.0: все накопленные 0.2.0/0.3.0/0.4.0/0.5.0/0.6.0/0.7.0 = ТЗ#1 Сц.4 «web→kmp screen port» (одна задача, итерации). Чужого нет.
- `iva-web-brownfield` 0.2.1→0.5.1: 0.3.0 (ТЗ#1 DS-скиллы), 0.4.0 (ТЗ#1 Figma numeric-compare), 0.5.0 (ТЗ#1 iva-core-design-system), 0.5.1 (ТЗ#3 FR v2). Чужого нет.
- `iva-analysis-base` 0.1.3→0.1.7: 0.1.4 (ТЗ#2 гейт классификации), 0.1.5 (ТЗ#3 FR v2 + формат 3.1), 0.1.6 (ТЗ#3 анализаторы data-model/events), 0.1.7 (ТЗ#2 ADR-first + 2-й слой). Смесь = только наши ТЗ#2+ТЗ#3.
- `iva-kmp-brownfield` 0.4.5→0.5.2: 0.5.0 (ТЗ#2 cross-repo ось-2), 0.5.1/0.5.2 (ТЗ#3 pin/tests/BRD под FR v2 + гейт D-n). Смесь = только наши ТЗ. Версии ≤0.4.5 уже в проде, в дельту не входят.

**3. C-исключение (architect/techwriter) безопасно — ПРОШЛО.** Grep depends_on по всем 22: ни один не ссылается на `iva-architect-mcp` / `iva-role-architect` / `iva-techwriter-mcp` / `tacticum-role-techwriter`. Упоминания «architect»/«techwriter» в 22 — только в комментариях и `description_trigger` (ADR-authoring прозой; boundary-note «пост-dev документирование — роль техписа») — это НЕ рёбра depends_on. Все 4 C-пакета существуют (0.1.0/0.3.0/0.1.0/0.3.0), это новые профили чужих направлений.

**4. Зависимости (orphan нет) — ПРОШЛО.** У 7 ролей 13 уникальных лейнов depends_on. 7 из них в deploy-set (development-core, bugfix-base, lite-base, research-base, kmp-development-base, web-development-base, analysis-base); 6 не в сете (tacticum-core-base, tacticum-ui-base, iva-{go,ios,java,mail}-development-base) — все существуют в репо как шаблоны и, по DB-дельте lead-modes, уже актуальны в проде (их нет в дельте = версии main==прод). Лейна, которого нет ни в 22, ни в проде, — нет. Пинов версий в depends_on нет (роль тянет последнюю активную) → порядок «лейны→роли» достаточен.
- Наблюдение (не блокер): руководный рунбук в §4 пишет «8 лейнов уже в проде» и включает туда kmp/web-development-base, которые фактически деплоятся (обновляются). Точный подсчёт: 6 лейнов вне сета. На состав сида и orphan-статус не влияет (профили в проде существуют, поднимается версия) — косметическая неточность формулировки.

**5. Роль-дедуп — ПРОШЛО.** Каждая из 7 ролей — один шаблон/один манифест/одна финальная версия. Дублей одной роли на разные ТЗ нет.

**6. Полнота — ПРОШЛО (репо-сторона).** Дельта main↔прод = 26 (DB, lead-modes). 22 ours + 4 C (foreign) = 26. Все 22 подтверждены как наши (пп.1–2), 4 исключённых подтверждены как чужое направление. Ничего нашего не выпало, чужого в 22 нет (осознанные «смешанные» = разные наши ТЗ). Полнота против ПОЛНОЙ дельты каталога — DB-сторона (закрыл lead-modes).

**7. mirror-sync + version-discipline на d70c68a — ПРОШЛО.** `check_mirror_sync.py` → «OK — 64 зеркальных ингредиента в 6 парах синхронны». `check_profile_version_discipline.py` → «OK — 48 profile(s) clean». Совпадает с ожидаемым 64/6, 48 clean.

## Границы проверки
- Репо-сторона (main d70c68a) — закрыл контролёр. Живой каталог `tacticum_catalog`, коллизии пар (профиль,версия), фактическое присутствие 6 not-in-set лейнов в проде — DB-сторона, закрывал lead-modes.
- HEAD=d70c68a, ветка main, чисто; единственный untracked `docs/adr/0060-*.md` (вне templates/, на сид не влияет).

## Связано
[[prep-combined-prod-seed-all3]] · [[night-final-all3-2026-07-25]] · [[prod-seed-iva-role-qa-prep]]
