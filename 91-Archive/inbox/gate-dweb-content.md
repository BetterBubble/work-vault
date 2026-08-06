---
title: gate-dweb-content
type: note
permalink: tacticum/00-board/gate-dweb-content-1
tags:
- gate
- controller
- us4
- iva-web-brownfield
- GO
archived-at: 2026-08-03 11:16
---

# Гейт: контент ветки D-web (ТЗ#3 US#4, финальный проход iva-web-brownfield)

Ветка `feat/us4-passD-web` @ 80607bc (worktree `/Users/bubblemac/tacticum-worktrees/us4-passD-web`), 1 коммит поверх main 8f8287c. Вердикт read-only контролёра по КОНТЕНТУ (ребейз на PR-C — отдельный шаг перед push).

## ВЕРДИКТ: GO (контент)

контент оба гардрейла PASS, coverage зелёный, не-DB 100% зелёный, все падения DB-инфра, скоуп чист, git чист; версия 0.4.1 временная — ребейз на PR-C перед push

## По пунктам

- **1. Скоуп — PASS.** Дифф ровно 6 файлов, ВСЕ внутри `templates/iva-web-brownfield/`: brd-authoring/SKILL.md, pin-authoring/SKILL.md, tests-authoring/SKILL.md, commands/start-task.md, manifest.yaml, CHANGELOG.md. Ничего вне профиля (владелец/зеркала/другие профили/роли/ui-base не тронуты). Косметик-фикс `libs/mail`→`libs/<domain>` присутствует в примере DM-2 pin.
- **2. Полный catalog/ + классификация — PASS.** 549 passed, 2 failed, 120 errors. ВСЕ 122 непрошедших — DB-инфра: фикстура `postgres_url` (conftest.py:35) поднимает `docker run postgres:16-alpine` → exit 125 (Docker недоступен в песочнице). 2 FAILED (test_create_draft_404_unknown_profile, test_patch_profile_404) — тот же корень, DB-ошибка всплывает внутри запроса (500 вместо 404), не логика. НИ ОДНОГО реального падения.
- **3. Coverage-тест — PASS.** `test_role_covers_replaced_profile` = 10 passed (параметризован). D-web новых скиллов не добавляет — контент существующих; coverage цел.
- **4. Не-DB структурный набор — PASS (100% зелёный).** parity/coverage/role_presets/install_smoke/schemas = 303 passed, 0 fail; web-brownfield profile test = 13 passed.
- **5. mirror-sync + version-discipline — PASS.** manifest 0.4.1 == верхняя запись CHANGELOG [0.4.1]; brd/start-task — чистый синк с каноном tacticum-dev-base, web-специфика сохранена. Все mirror/version-тесты — в 549 non-DB passed. Версия 0.4.1 ВРЕМЕННАЯ (→0.5.1 после PR-C) — не сужу строго по указанию.
- **6. Git-чисто — PASS.** Автор Александр Шульга (aleksandr-shulga-0507@yandex.ru), 0 AI-подписей (совпадения "ai" — ложные из xfail/fail-open), 0 секретов/.env/ключей/мусора. Ветка feat/us4-passD-web (не main).
- **7. Строго-по-ТЗ — PASS.**
  - **brd К-1/К-5:** детект маркера `fr_skeleton` до чтения; v2 → FT-n §1.4 / UC-n §1.5 Части 1 (НЕ из Приложения) + ссылка на CT-n/DM-n/EV-n; v1 (маркер отсутствует) → П.A/П.B по-старому; ID наследуются дословно; backward-safe.
  - **pin К-2/К-3/К-4:** реализация CT-n/DM-n/EV-n по стабильному ID под web-стек (REST/@ivcs/ng-openapi-gen, TS/NgRx-signals, Centrifuge/WS/RxJS); таблица статусов; К-3 гейт D-n → blocked без имитации; К-4 расхождение проект↔код через kb_verify_api_exists/kb_get_code_context = критичное.
  - **tests К-2:** контрактные тесты `Covers: CT-n/DM-n/EV-n`, msw/HttpClient-мок/TestBed+Jest(iva-one)/Vitest(iva-connect); расхождение→xfail, blocked→нет теста; таблица статусов.
  - **start-task К-3:** гейт на ПЛАШКЕ «Предложение, требует утверждения: разработчик + CTO» + открытый Q-n → BLOCKED без fail-open даже при битом маркере; независим от маркера; v1 без плашек → гейт не срабатывает. Соответствует ТЗ§4/§2.2 п.3 (§1.9 UI plated), backward-safe, факты не выдуманы.

## Замечание (не дефект)
run-tester.md стр.11 pre-existing mail-примеры (UC-MAIL-042) — вне скоупа D-web, оставлено намеренно; не гейтил.

## Следующий шаг
Тимлид → OK Президента (через ГД). Перед push — ребейз ветки на PR-C, версия станет 0.5.1.