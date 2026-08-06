---
title: Правка привязки логина и двойного granted (iva-write)
type: report
status: draft
date: 2026-07-29
tags:
- iva-write
- helm
- keystore
- oauth
- gate-fix
permalink: tacticum/00-board/impl-loginfix-2026-07-29-1
archived-at: 2026-08-06 09:37
---

# Правка привязки логина и двойного granted

Репозиторий helm, worktree `/Users/bubblemac/tacticum-worktrees/helm-iva-write-keystore`,
ветка `feat/iva-write-keystore`. Коммит **`3ac09b7`** поверх `d4eb847`. Не пушилось,
не мержилось, на серверы не ходилось.

Обе правки — по повторному гейту `gate-iva-write-final-2026-07-29.md`.

## П1 — `userKey` больше не идентификатор

**Что было.** `whoami` собирал логин по цепочке `name` → `username` → `userKey`, и он
хранится ОДИН на все системы (`bound_login` ищет по всем). `userKey` — внутренний ключ
учётной записи Confluence, а не логин: с `name` из Jira он не совпадёт никогда. Ответ
Confluence без `username` давал бы `login_mismatch` — «Согласие выдано под другой
учётной записью», то есть человека обвиняли бы в чужой учётке там, где разошлись наши
собственные поля. Тест этого не ловил: в заглушке обе системы отдавали одинаковый логин.

**Что стало.** Идентификатором считаются только `name` (Jira) и `username`
(Confluence) — сопоставимые поля общей директории пользователей. Нет ни того, ни
другого → `login=""`, то есть идентификатора нет вовсе, и дальше работают уже
существующие правила: `subject` из аутентификации → слабая запись (`identity_weak`),
названа в чате → отказ; при существующей привязке — `identity_unverified` (fail-closed,
но без обвинения).

- `src/helm/infrastructure/iva_oauth/client.py:400` — `userKey` убран из цепочки;
  докстринг `whoami` (`:374-388`) называет причину.
- `src/helm/application/iva_write.py:31-35` — в решение 2 модульного докстринга добавлено,
  что логин это только `name`/`username`, а ответ с одним `userKey` означает
  «идентификатора нет», а не «не совпал».

## П2 — одно согласие = одна строка `granted`

**Что было.** `application/iva_write.py` писал исход сверки и статус ключа одинаковым
`outcome="granted"`, различие только в `reason`. Подсчёт успехов по `outcome` дал бы
вдвое больше согласий, чем их было.

**Что стало.** Статус ключа пишется своим исходом `key_status` (`reason=key_*`).
`granted` теперь значит ровно одно успешное согласие.

- `src/helm/application/iva_write.py:351-368` — `outcome="key_status"` + комментарий, зачем.
- `src/helm/infrastructure/db/credential_repo.py:29-38` — `key_status` в `USE_OUTCOMES`
  (словарь исходов закрытый, `ValueError` на чужое) + описание.
- `src/helm/infrastructure/db/models.py:3129-3134,3150` — докстринг и комментарий колонки.

Имя исхода взято `key_status`, а не предложенное в постановке `key_note`: `key_note` уже
занято функцией в `application/tacticum_key.py`, которая рендерит фразу для человека, и
одно имя на две разные вещи путало бы читателя. Смысл тот же.

## Заглушка и тесты

`tests/support/atlassian_dc.py` — новый флаг `expose_login` (по умолчанию `True`).
`False` = сервер отдаёт только внутренний ключ учётки: у Confluence остаётся `userKey`
без `username`, у Jira — `key` без `name`. Модульный докстринг и `_confluence_stub` в
тестах обновлены.

Три новых теста в `tests/application/test_iva_write_consent.py`:

1. `test_only_user_key_means_no_identifier_not_a_foreign_login` — логин привязан на Jira,
   Confluence отдаёт только `userKey` → `identity_unverified`, не `login_mismatch`;
   запись не создаётся, привязка не тронута. Это тот случай, который назвал гейт.
2. `test_user_key_is_never_bound_as_login` — согласие с одним `userKey` не создаёт
   привязку (`external_login == ""`, `bound_login == ""`). Иначе внутренний ключ
   Confluence лёг бы идентификатором и следующее согласие в Jira отказывалось бы как
   «чужая учётка».
3. `test_one_consent_writes_exactly_one_granted` — журнал успешного согласия равен
   `[("granted", "email_verified"), ("key_status", "key_known_present")]`, строк
   `granted` ровно одна.

Плюс тестовый помощник `_entries` (пары исход+причина) рядом с существующим `_reasons`.

**Существующие тесты не удалялись и не переписывались** — правки поведения не сломали ни
одного: `reason` у ключевой строки прежний (`key_*`), а проверок на её `outcome` в
тестах не было. Оба места, где тесты смотрят на `("granted", …)`
(`tests/interface/test_iva_write_api.py:268`, `tests/interface/test_iva_internal_api.py:398`),
касаются сверки личности и продления, а не ключа, и остались зелёными как есть.

## Числа и проверки

| | до | после |
|---|---|---|
| `uv run pytest` | 2368 passed, 32 skipped | **2371 passed, 32 skipped** |

- `mypy --strict` (конфиг проекта strict=true) по всем шести затронутым файлам — чисто.
- `ruff check` по своим файлам — чисто, кроме одного **предсуществующего** E501 в
  `models.py:1947` (комментарий про статусы поставки, к этой работе отношения не имеет;
  тот же самый на `HEAD` до правок). Не трогал: это чужая строка вне объёма задачи.