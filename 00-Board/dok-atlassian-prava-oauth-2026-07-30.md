---
title: dok-atlassian-prava-oauth-2026-07-30
type: note
permalink: tacticum/00-board/dok-atlassian-prava-oauth-2026-07-30
status: draft
lead: explorer
date: 2026-07-30
method: read-only, официальная документация Atlassian (confluence.atlassian.com,
  developer.atlassian.com, jira.atlassian.com REST-поиск). WebSearch/WebFetch/basic-memory
  в этом контексте недоступны — источники получены curl'ом по официальным доменам,
  поиск через публичный Confluence Search REST на confluence.atlassian.com.
tags:
- board
- iva-write
- oauth
- atlassian
- explorer
---

# Права для incoming OAuth 2.0 link: Jira DC 10.3 и Confluence DC 9.2

**Главный вывод:** **System Administrator НЕ обязателен.** По документации обоих продуктов
регистрация incoming-ссылки типа OAuth относится к «обычному» админу: в Jira — `Jira administrators`
(ADMINISTER), в Confluence — глобальное право `Confluence Administrator`. Оба утверждения
опираются на **прямые формулировки** в доках, но обе формулировки написаны в терминах «OAuth»
без уточнения версии (1.0/2.0) — это единственная дыра, см. §2 и §3, «риск».

Версии документации взяты ровно под заказчика: пространство `ADMINJIRASERVER103` (Jira DC 10.3)
и `CONF92` (Confluence DC 9.2). Не «Latest», не Cloud.

---

## 1. Где регистрируется incoming OAuth 2.0 link (UI)

**Jira DC 10.3 — написано прямо:**
> Go to **Administration > Applications > Application links**. Select **Create link**.
> Select **External application**, and then choose **Incoming** as the direction.

Отдельного раздела «OAuth 2.0» в меню нет — это подтип application link. Тип называется
**External application / Incoming**. Далее: Redirect URL (Callback URL) → выбор scopes →
Jira генерирует Client ID и Client secret. Посмотреть повторно: Application links →
**More actions > View credentials**.

Источник: https://confluence.atlassian.com/spaces/ADMINJIRASERVER103/pages/1489807047/Configure+an+incoming+link

**Confluence DC 9.2 — написано прямо:**
> Go to **Administration menu**, then **General Configuration > Application links**.
> Select **Create link**. Select **External application**, and then choose **Incoming** as the direction.

Источник: https://confluence.atlassian.com/spaces/CONF92/pages/1477577012/Configure+an+incoming+link

Практическая инструкция с живым примером (KB, помечена «Data Center Only», среда 8.22/9.x/**10.x**):
https://confluence.atlassian.com/display/JIRAKB/How+to+test+the+Jira+Incoming+Link+OAuth+2.0+API
Там же прямое **пререквизит-требование: «Jira must run on SSL»**.

---

## 2. Какое право нужно в Jira DC — ADMINISTER или SYSTEM_ADMIN

**Написано прямо (Jira 10.3, «Managing global permissions»):** в списке задач, которые может
делать **только** `Jira System administrators` (и не может `Jira administrators`), стоит пункт:

> Configure Application Links that use **an authentication type other than OAuth**.

Читается однозначно: application links **с** OAuth-аутентификацией сисадмина НЕ требуют →
достаточно глобального права `Jira administrators` (ADMINISTER).

Источник: https://confluence.atlassian.com/spaces/ADMINJIRASERVER103/pages/1489807379/Managing+global+permissions
(раздел «About Jira System administrators and Jira administrators»).

Там же прямо: «For all of the following procedures, you must be logged in as a user with the
**Jira administrators or Jira System administrators** global permission».

**Риск (следует из контекста, не написано):** формулировка «other than OAuth» появилась в эпоху
OAuth 1.0 (Trusted Apps / Basic Access / OAuth). Что incoming-link **OAuth 2.0** попадает в ту же
категорию «OAuth» — это вывод по смыслу, а не прямая цитата. Отдельного предложения
«для OAuth 2.0 incoming link достаточно Jira administrators» в документации **нет**.
Проверяется за минуту живьём: дать тестовому пользователю только ADMINISTER и открыть
`/plugins/servlet/applinks/listApplicationLinks`.

**Явного требования SYSTEM_ADMIN нигде не найдено.** Ни на странице настройки incoming link,
ни в KB, ни в Jira-трекере.

---

## 3. Какое право нужно в Confluence DC 9.2

**Написано прямо** — в таблице «System Administrator and Confluence Administrator permissions
compared», строка **Administration**:

| Admin console | Confluence administrator | System administrator |
|---|---|---|
| Administration | … License Details, **Application Links (OAuth only)**, Application Navigator | … License Details, Logging and Profiling, **Application Links**, Application Navigator, Analytics … |

То есть носитель глобального права **Confluence Administrator** получает доступ к Application
Links **в объёме OAuth**; полный доступ к Application Links — у System Administrator.

Источник: https://confluence.atlassian.com/spaces/CONF92/pages/1477576887/Global+Permissions+Overview

Важное различение оттуда же (написано прямо): глобальное право `Confluence Administrator` и группа
`confluence-administrators` — **не одно и то же**. Группа `confluence-administrators` даёт
System Administrator по умолчанию плюс видимость всего контента. Для нашей задачи членство в
`confluence-administrators` **не требуется** — хватает глобального права Confluence Administrator.

**Риск тот же, что в §2:** «(OAuth only)» не уточняет версию протокола.

---

## 4. REST API для регистрации incoming OAuth-клиента

**Однозначного ответа в документации нет — но косвенно: только UI.**

Что проверено:
- Обе страницы «Configure an incoming link» (Jira 10.3, Confluence 9.2) описывают **только UI-путь**.
  Ни одного упоминания REST для создания клиента.
- Страницы «Jira OAuth 2.0 provider API» и «Confluence OAuth 2.0 provider API» описывают
  **только сторону потребителя** (authorize / token / refresh) и прямо говорят:
  > Register your application in Jira by **creating an incoming link in application links**.
- Старый `/rest/applinks/1.0/` умеет читать и удалять классические applinks
  (`GET /rest/applinks/1.0/listApplicationlinks`, `DELETE /rest/applinks/1.0/applicationlink/{id}`;
  KB прямо пишет, что для удаления нужен профиль с правом `jira-administrators`) — источник:
  https://confluence.atlassian.com/display/JIRAKB/How+to+remove+Application+Link+%28AppLinks%29+via+REST+API
  **Создания** incoming OAuth 2.0 клиента там не описано.
- Поиск по трекеру Atlassian (JRASERVER + CONFSERVER, REST-поиск по jira.atlassian.com)
  не дал ни фич-реквеста, ни документированного эндпоинта на эту тему.

Вывод: **документированного REST для регистрации клиента нет; считаем, что только UI.**
Формулировка «в документации не описано» точнее, чем «не существует».

---

## 5. Альтернативы без администратора

### (а) PAT — пользователь выпускает себе сам, админ не нужен. Написано прямо.

> **All users are allowed to create their own PATs**, which will match their current permission level.

- **UI Jira:** аватар → **Profile** → в левом меню **Personal access tokens** → Create token.
- **UI Confluence:** аватар → **Settings** → **Personal access tokens** → Create token.
- **REST есть:** `POST {baseUrl}/rest/pat/latest/tokens`, тело `{"name": "...", "expirationDuration": 90}`,
  аутентификация — Basic Auth своим логином/паролем или уже имеющимся PAT.
  Прямо сказано: **«you cannot create PATs on behalf of someone else»** — только себе.
- Использование: заголовок `Authorization: Bearer <token>`.
- Доступно с Jira Core/Software 8.14+, JSM 4.15+, Confluence 7.9+ → **обе версии заказчика подходят**.
- **Что может закрыть админ:** системные свойства `-Datlassian.pats.enabled` (глобальный выключатель),
  `-Datlassian.pats.eternal.tokens.enabled`, лимиты и правила истечения.

Источник: https://confluence.atlassian.com/display/ENTERPRISE/Using+Personal+Access+Tokens

### (б) «Пользователь сам регистрирует OAuth-приложение» — в DC такого нет

Аналога Cloud-овского developer console в Data Center **в документации не описано**. Единственный
описанный способ получить client_id/client_secret — админская страница Application links (§1).
Следует из контекста, а не написано прямым отрицанием: Atlassian нигде не пишет «в DC пользователь
не может зарегистрировать приложение», просто **никакого не-админского пути не документировано**.

### (в) Прочие пути получить токен от имени пользователя с его согласием

- **Уже существующий incoming link.** Если у заказчика хоть один incoming OAuth 2.0 link уже заведён,
  админ не нужен — только его credentials. (Наши живые пробы `/rest/oauth2/latest/authorize`
  показали, что провайдер включён, но клиент не зарегистрирован.)
- **Basic Auth логином/паролем** — работает, но это не «согласие», а передача пароля; отдельно
  ломается при SSO/2SV.
- **OAuth 1.0a incoming link** — тоже настраивается в Application links и тоже админская операция,
  выигрыша по правам не даёт.
- Больше ничего документация не предлагает. **Реально не-админский путь ровно один: PAT.**

---

## 6. Cloud vs Data Center — что к чему относится

**Всё в §1–§5, кроме подпункта ниже, — это Data Center**, версии подтверждены пространствами
`ADMINJIRASERVER103` и `CONF92`. KB по тестированию incoming link помечена плашкой
«Platform Notice: **Data Center Only**».

**К Cloud (не применимо к заказчику):**
- OAuth 2.0 (3LO) в Cloud регистрируется в **developer console**
  (`https://developer.atlassian.com/console`) — «Create → OAuth 2.0 integration». Это принципиально
  другая модель: приложение заводится в аккаунте разработчика, **администратор сайта для создания
  интеграции не нужен**. Источник: https://developer.atlassian.com/cloud/jira/platform/oauth-2-3lo-apps/
- Cloud-скоупы (`read:jira-work`, `write:jira-work`, granular scopes) **не имеют отношения к DC**.
  В DC скоупы другие — см. §7.

Практический смысл: любой найденный в интернете совет «просто заведи приложение в developer
console» — про Cloud и у заказчика **не работает**.

---

## 7. Scope'ы OAuth 2.0 в DC

**Jira DC 10.3** (написано прямо; «Implied scopes» — из provider API, каскад включения):

| Scope | Что даёт | Включает |
|---|---|---|
| `READ` | View projects and issues | READ |
| `WRITE` | **Create, update, and delete projects and issues** (+ дашборды, фильтры, вложения, комментарии, свой профиль) | READ, WRITE |
| `ADMIN` | Administer Jira (кроме бэкапов/импортов/инфраструктуры) | READ, WRITE, ADMIN |
| `SYSTEM_ADMIN` | Administer Jira system (включая бэкапы, импорты, инфраструктуру) | READ, WRITE, ADMIN, SYSTEM_ADMIN |
| `READ_ALL` | Read all content, users, groups, permissions **игнорируя права**, без учётки — только для service-to-service | — |
| `MANAGE_SUBSCRIPTIONS` | подписки incremental sync + всё из READ_ALL | — |
| `JSM_KB` | связка JSM ↔ Confluence KB | — |

Источники:
https://confluence.atlassian.com/spaces/ADMINJIRASERVER103/pages/1489807049/OAuth+2.0+scopes+for+incoming+links
https://confluence.atlassian.com/spaces/ADMINJIRASERVER103/pages/1489807050/Jira+OAuth+2.0+provider+API

**Confluence DC 9.2:** `READ`, **`WRITE`** (Create, update, and delete content: spaces, pages,
blog posts, custom content, attachments, comments, templates), `ADMIN`, `SYSTEM_ADMIN`,
`READ_ALL`, `MANAGE_SUBSCRIPTIONS`.
Источник: https://confluence.atlassian.com/spaces/CONF92/pages/1477577013/OAuth+2.0+scopes+for+incoming+links

**Нам нужен ровно `WRITE`** — он покрывает создание/изменение задач в Jira и страниц в Confluence.
ADMIN/SYSTEM_ADMIN запрашивать не нужно и вредно.

Ключевая гарантия, написана прямо в обоих продуктах:
> Even if you grant higher permissions, the application won't be able to do more than the user
> authorizing it.

То есть выданный админом scope — это **потолок**, фактические права режутся правами того, кто нажал
«разрешить». Аргумент в разговоре с ИБ заказчика.

---

## 8. PKCE в DC

**Поддерживается в обоих. Написано прямо.**

Jira DC 10.3, «Supported OAuth 2.0 flows»:
> We support the following OAuth 2.0 flows: **Authorization code with Proof Key for Code Exchange (PKCE)**;
> Authorization code.
> We don't support Implicit Grant and Resource Owner Password Credentials flows.

Источник: https://confluence.atlassian.com/spaces/ADMINJIRASERVER103/pages/1489807050/Jira+OAuth+2.0+provider+API

Confluence DC — тот же текст, страница вынесена на developer.atlassian.com (ссылка стоит прямо
со страницы CONF92 «Configure an incoming link»):
https://developer.atlassian.com/server/confluence/confluence-oauth2-provider-api/

**Расхождение в самих доках, отметить:** таблица параметров говорит
`code_challenge_method` — «Can be **plain** or **sha256**», а пример запроса в той же статье
использует `code_challenge_method=**S256**`. Оба варианта в тексте Atlassian; какое значение
реально принимает 10.3/9.2 — проверять на живом стенде. Однозначного ответа в документации нет.

---

## 9. Риски и грабли, найденные попутно

- **HTTPS обязателен.** Провайдер требует HTTPS и для base URL, и для redirect URI в проде.
  Обходится системными свойствами `atlassian.oauth2.provider.skip.base.url.https.requirement` и
  `atlassian.oauth2.provider.skip.redirect.url.https.requirement` — но их выставляет админ в
  `setenv.sh` и только для stage/dev. KB прямо: «Jira must run on SSL».
  Источник: https://confluence.atlassian.com/spaces/ADMINJIRASERVER103/pages/1489807053/OAuth+2.0+provider+system+properties
- **Известный баг DC:** JRASERVER-71779 «Creating an incoming application link for OAuth can corrupt
  previously configured links» — статус *Long Term Backlog*, не исправлен, 8 голосов.
  https://jira.atlassian.com/browse/JRASERVER-71779
  Заказчику это может быть весомее прав: создание нашей ссылки теоретически задевает существующие
  applinks. Стоит предупредить до просьбы.
- **Время жизни токена:** в примере KB `expires_in: 3600`, в provider API — `7200`; настраивается
  `atlassian.oauth2.provider.access.token.expiration.seconds`. Refresh token переиспользуется,
  при refresh старая пара инвалидируется.
- **Пользователь может отозвать доступ сам:** профиль → **Authorized applications** (написано прямо
  на обеих страницах scopes). Полезно для аргумента «это обратимо».

---

## 10. Что осталось непроверенным (честно)

1. Нет прямой цитаты «OAuth **2.0** incoming link доступен Jira administrators / Confluence
   Administrator». Есть только «Application Links that use an authentication type other than OAuth»
   (Jira) и «Application Links (OAuth only)» (Confluence). Вывод корректный, но по смыслу.
2. Нет подтверждения, `plain`/`sha256` или `S256` принимает `code_challenge_method` в 10.3/9.2.
3. Не проверено, включён ли у заказчика `atlassian.pats.enabled` (PAT может быть выключен глобально).
4. community.atlassian.com как вторичный источник **не использовался** — поиск по нему в этом
   окружении недоступен (404 на поисковой ручке, WebSearch выключен). Всё выше — первичные
   источники Atlassian.
