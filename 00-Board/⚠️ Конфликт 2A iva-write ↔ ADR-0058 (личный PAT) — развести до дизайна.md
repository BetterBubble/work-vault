---
title: ⚠️ Конфликт 2A iva-write ↔ ADR-0058 (личный PAT) — развести до дизайна
type: note
permalink: tacticum/00-board/konflikt-2-a-iva-write-adr-0058-lichnyi-pat-razvesti-do-dizaina
status: parked-needs-decision
role: lead (тимлид)
date: 2026-07-21
tags:
- flag
- iva-write
- adr-0058
- escalation
---

# Флаг: 2A iva-write конфликтует с ADR-0058

2A **запаркован** по решению пользователя (фокус на деплой 2C). Перед возобновлением дизайна 2A — развести с ГД/пользователем:

**Конфликт (нашла разведка [[explore-iva-write-base]]):**
- План/хендофф зафиксировали: **личный PAT сотрудника** + мульти-ключевая схема (3-4 ключа).
- **ADR-0058** (docs/adr/0058-*, Proposed, грилл сегодня 2026-07-21), Решение 5: **отменяет личный PAT** из Taiga #712 → PAT **техучётки `iva`** + принудительная подпись актора + scope `iva-req-write`. Личный PAT явно отвергнут как немасштабируемый.

**Вопрос к ГД:** дизайн 2A строим по #712 (личный PAT) или по ADR-0058 (техучётка)? От этого зависит вся аутентификация лейна iva-write-base.

**Плюс:** мульти-ключевой схемы (2+ ключа на сервер) в манифестах прецедента НЕТ — будет новая конструкция (`auth_type` один на сервер; `required_scopes` в схеме есть, но нигде не используется).

## Связано
- [[explore-iva-write-base]] · [[plan-tri-deliveravla-iva-iva-write-3-roli-my-todo-priviazka-k-sisteme]]