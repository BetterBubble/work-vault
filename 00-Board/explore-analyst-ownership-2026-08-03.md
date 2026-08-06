---
title: 'Разведка: опора по «смене владельца записи» в наших кодовых базах и знаниях'
type: note
status: draft
created: 2026-08-03
tags:
- board
- analyst
- разведка
- ownership
permalink: tacticum/00-board/explore-analyst-ownership-2026-08-03
---

# Разведка: есть ли опора по владению записью и его передаче

Проверяемое утверждение команды аналитики: требование «смена владельца записи»
«нигде никем не было сделано и даже ни разу не обдумано».

**Короткий факт: утверждение неверно.** Опора есть в обоих смыслах слова «владение»,
и их два разных — их надо разводить (см. §0).

## 0. Дисциплина терминов: три разных «владельца», не путать

| Что | Где живёт | Смысл |
|---|---|---|
| **A. Владение в продукте ИВА** (владелец чата/канала/мероприятия/записи ВКС) | продуктовые требования ИВА, вики `wiki.iva.ru`, Jira `IVAONE*`/`VCSWEB*`/`IVCS` | **это и есть то, что спрашивали аналитики** |
| **B. Владение в HELM** (кто отвечает за область/продукт/контейнер/заказчика) | репо `helm`, ADR-0008, `ownership_assignment` | управленческий слой нашего же инструмента |
| **C. Владелец документа/профиля** | `tacticum-dev` (ADR-метаданные `Владелец: mr.diaret@ya.ru`, `profiles.owner_organization_id`) | почти пусто, к делу не относится |

Jira-«владелец задачи» (assignee) — четвёртое, к делу не относится вообще.

## 1. Сущность владельца/владения в helm (контур B) — ЕСТЬ, полноценная

Единый слой владения, ADR-0008 «Единый слой владения HELM», статус **Принят 2026-07-26**:
`/Users/bubblemac/tacticum/helm/docs/adr/0008-helm-ownership-model.md`

Модель данных — одна таблица вместо полей-владельцев на сущностях:

- `/Users/bubblemac/tacticum/helm/src/helm/infrastructure/db/models.py:1475` —
  `__tablename__ = "ownership_assignment"`; колонки `scope_kind`, `scope_ref`, `role`,
  `person_id` (FK `person.person_id`), `primary` (физически `is_primary`), `active`,
  `granted_at`, `granted_by_person_id`, `tenant` (строки 1501–1514).
- Индексы (`models.py:1476-1499`): RBAC по ячейке, обратный по человеку, partial-unique
  среди активных — повторный add идемпотентен, soft-remove не мешает пере-назначению.

Домен: `/Users/bubblemac/tacticum/helm/src/helm/domain/ownership.py`

- `SCOPE_KINDS` (`:23`) = `client · product · area · container · area_product`;
- `ROLES` (`:27`) = `sales · presale · po · tpo · competence_lead` (CPO — не роль назначения,
  а портфельный full-доступ);
- матрица допустимости роль×измерение `_SCOPE_ROLES` (`:31`), валидация `validate_scope_role` (`:79`);
- предикаты RBAC: `has_role` (`:113`), `primary_owner` (`:134`), `scopes_owned_by` (`:159`),
  `effective_owners` с наследованием ячейки матрицы (`:180`).

Репозиторий: `/Users/bubblemac/tacticum/helm/src/helm/infrastructure/db/ownership_repo.py`
— `add_assignment` (`:158`, реактивирует soft-removed строку), `remove_assignment` (`:232`,
soft `active=false`, история сохраняется), `set_primary` (`:246`).

Кроме единого слоя есть **поля-владельцы прямо на записях**:
- `arch_node.owner_person_id` — владелец контейнера C4 (`models.py:2466`);
- `process_task.owner_person_id` / `owner_email` — владелец стадийной задачи (`models.py:2949-2952`);
- `product.owner_email` (`models.py:59`), `initiative.owner_email` (`models.py:853`),
  `sales_initiative.owner_email` (`models.py:900`).

## 2. Операция СМЕНЫ владельца — по слоям

### (а) Модель/БД — ЕСТЬ, тремя разными способами

1. **Единый слой (assignment)**: смена = `add_assignment` + `remove_assignment` +
   `set_primary`. **Атомарной операции «передать владение» (transfer/reassign) НЕТ** —
   это не одна ручка, а комбинация. История держится `active`+`granted_at`+`granted_by_person_id`.
   Use-cases: `/Users/bubblemac/tacticum/helm/src/helm/application/ownership.py:97`
   (`add_owner`), `:137` (`remove_owner`), `:144` (`set_primary_owner`).
2. **Владелец контейнера C4** — прямая смена поля с аудитом (см. (б)).
3. **Владелец стадийной задачи** — прямая смена с записью события:
   `/Users/bubblemac/tacticum/helm/src/helm/application/stage_owners.py:215`
   `assign_stage_owner(...)`; `person_id=None` = снятие; смена пишет `owner_assigned`
   в `process_event` (`stage_owners.py:293-301`); повтор того же владельца — no-op без события.

### (б) REST API — ЕСТЬ

- `/Users/bubblemac/tacticum/helm/src/helm/interface/api/routers/ownership.py:26` —
  роутер `/api/ownership`; `GET ""` (`:90`), `GET /scopes` (`:114`), `GET /matrix` (`:139`),
  `POST ""` (`:159`), `DELETE /{assignment_id}` (`:183`), `POST /{assignment_id}/primary` (`:195`).
  Гейт правок — `require_cpo` (`:29`), только full_access, иначе 403.
- **Буквальная «смена владельца записи» с аудитом:**
  `/Users/bubblemac/tacticum/helm/src/helm/interface/api/routers/cio.py:4561`
  `PATCH /api/cio/arch-nodes/{node_id}/owner` → `set_arch_owner`, докстринг дословно
  «Сменить владельца (техлида/CTO) контейнера — с аудитом; переживает реингест C4».
  Пишет `RefdataAudit(entity="arch_node", kind="update", value="owner_person_id=… (via=…)", actor=…)`
  (`cio.py:4576-4580`). Поле `via` = `manual|recommended` (`cio.py:4558`).
- `/Users/bubblemac/tacticum/helm/src/helm/interface/api/routers/process.py:487`
  `PUT /api/process/requirement/{rid}/stages/{stage_key}/owner`; рядом
  `GET .../stage-candidates/{stage_key}` (`process.py:461`) — владелец выбирается только
  из справочника кандидатов, свободного ввода нет.

### (в) Веб-UI — ЕСТЬ

- `/Users/bubblemac/tacticum/helm/web/src/components/OwnerEditor.tsx:1` — переиспользуемый
  «редактор ответственных» одной ячейки (добавить/снять/сделать основным), правки под CPO;
  пусто → бейдж «не назначен».
- `/Users/bubblemac/tacticum/helm/web/src/screens/OwnershipMatrix.tsx:1` — экран матрицы
  «Область × Продукт» с наследованием и override; смонтирован в настройках:
  `/Users/bubblemac/tacticum/helm/web/src/screens/Settings.tsx:633`.
- `/Users/bubblemac/tacticum/helm/web/src/screens/ArchOwners.tsx:221` — кнопка с
  `title="сменить владельца (рекомендации + поиск)"`, компонент `OwnerPicker` (`ArchOwners.tsx:192`)
  с ранжированными рекомендациями и typeahead по ростеру.
- Клиент API: `/Users/bubblemac/tacticum/helm/web/src/api.ts:794` (`cioSetArchOwner`),
  `:888-908` (`ownershipList/Scopes/Add/Remove/SetPrimary/Matrix`), `:882` (PUT владельца стадии).

### (г) MCP-слой — НЕТ, ни чтения назначений, ни смены

- `/Users/bubblemac/tacticum/helm/src/helm/interface/mcp/analyst_server.py` — полный список
  тулов: `analyst_search` (`:371`), `analyst_context` (`:389`), `related_tasks` (`:406`),
  `docs_ask` (`:430`), `api_registry_check` (`:447`), `contract_check` (`:464`),
  `requirement_tests` (`:481`), `arch_map` (`:532`), `arch_container` (`:584`),
  `arch_drift` (`:612`), `affected_systems` (`:733`), `requirement_coverage` (`:801`),
  `nearest_spec` (`:1091`), `who_to_involve` (`:1135`), `effort_hint` (`:1199`),
  `gap_questions` (`:1329`), `constraints` (`:1465`), `contradiction_check` (`:1501`),
  `my_todo` (`:1558`). **Ни одного тула по `ownership_assignment`.** Владелец виден
  только косвенно: `who_to_involve` отдаёт `owners` — владельцев узлов C4 (`:1152-1189`).
- `/Users/bubblemac/tacticum/helm/src/helm/interface/mcp/process_server.py` — тулы
  `start_process`, `get_process`, `request_approval`, `approve_gate`, `reject_gate`,
  `advance_tracks`, `my_process_tasks`. `owner_person_id`/`owner_email` только **отдаются**
  на чтение (`:230-231`, `:546-547`); **тула назначения/смены владельца стадии нет**.
- `/Users/bubblemac/tacticum/helm/src/helm/interface/mcp/hrd_server.py:411` —
  `offboarding_impact` читает ячейки владения (`ownership`, `sole_owner`, `:432-451`),
  `bus_factor` (`:476`). Только чтение, только HRD-контур.

**Картина совпадает с прецедентом `needs_clarification`:** флаг есть в
`src/helm/interface/api/routers/product.py`, `models.py`, `web/src/api.ts`, `types.ts`,
`ProductPage.tsx`, `ReleaseWindows.tsx` — и **отсутствует в `src/helm/interface/mcp/`**
целиком (grep по каталогу: 0 совпадений). То же и с владением: REST+UI есть, MCP — нет.

## 3. История (helm) — всё ДО 31.07

| Хеш | Дата | Коммит |
|---|---|---|
| `4931c99` | 2026-07-11 | Add C4 container owners and repo overrides (`arch_node.owner_person_id`, CIO-эндпоинты смены владельца + аудит, экран «Контейнеры и владельцы») |
| `447d681` | 2026-07-11 | Add smart owner candidate recommendations (**US #127**) — ранжированные кандидаты в владельцы, `OwnerPicker`, поле `via` в аудит |
| `eeebb8e` | 2026-07-26 | Add stage owner assignment and CTO refdata (**задача #186**) — назначение владельцев стадий, `cto_level`, событие `owner_assigned`, дропдауны в UI |
| `c8ddbf7` | 2026-07-27 | Add ownership assignment layer and API — модель `ownership_assignment` + миграция + `/api/ownership` |
| `87642fc` | 2026-07-27 | Unify area ownership with assignments — бэкфилл из `product_area.{po,tpo}_person_id`, `GET /api/ownership/scopes`, `OwnerEditor` |
| `dd9ebf5` | 2026-07-27 | Add area-product ownership matrix (**US #191**) — композитный scope `area_product`, `/api/ownership/matrix`, экран матрицы |
| `d6cd8ef` | 2026-07-27 | Add product priority and matrix hide action |
| `a02793f` | 2026-07-27 | Normalize granted_at timezone in ownership repo |

US, названные в ADR/коде: эпик **#188**, US **#189**, **#190**, **#191** (ADR-0008),
US **#127** (владельцы контейнеров), задача **#186** (владельцы стадий).

Для сверки прецедента: `b44bd32` | 2026-07-28 | `feat: needs_clarification flag on requirements (US #204)`.

Вся реализация владения в helm — **11–27 июля 2026**, то есть **до 31.07**, когда
аналитики ставили эксперимент.

## 4. Что есть в знаниях, доступных аналитику (контур A — продукт ИВА)

Здесь опора **прямая и точная**, вплоть до дословного совпадения формулировки.

### 4.1 Требование «Смена владельца записи ВКС» УЖЕ в каталоге требований — 4 раза

`/Users/bubblemac/tacticum/helm/data/iva/wiki_reqs_10.json`:
- `№168`, `"ВКС: Смена владельца записи ВКС"`, `status_text: "Отсутствует"`,
  `page_title: "Требования к IVA One 1.0 (Клиенты)"`, `rn_track: "Да"`;
- `№191`, тот же заголовок, `page_title: "Требования к IVA One 1.0 (Полный перечень)"`, `rn_track: "Да"`.

`/Users/bubblemac/tacticum/helm/data/iva/wiki_reqs_15.json` (generation `1.5`, 957 строк):
- `"ВКС: Смена владельца записи ВКС"`, `status_text: "Не реализовано"`, `client: ["Мосбиржа"]`,
  `page_title: "IVA 1.5"`;
- он же, `status_text: "Отсутствует"`, `component_or_area: "ВКС"`, `page_title: "ВКС 1.5"`.

`/Users/bubblemac/tacticum/helm/data/iva/req_realization.json` — строка
`{"title": "ВКС: Смена владельца записи ВКС", "generation": "1.0", "verdict": "absent"}`.

Тело страницы Confluence:
`/Users/bubblemac/tacticum/helm/data/real/confluence/bodies/IVAPROJECT_205489583.md:469-471`
(страница «[IVAPROJECT] ВКС 1.5», `pageId=205489583`, версия v28), строка каталога №59:

> `ВКС: Смена владельца записи ВКС` … `Отсутствует` `Высокая` `В клиентском коде не
> реализована смена владельца записи ВКС; найдены только просмотр/навигация записей и
> смежное storage-management.`

То есть требование не только сформулировано, но и **проверено по коду** с вердиктом и
достоверностью.

### 4.2 По этому требованию ЕСТЬ живая задача в Jira

`/Users/bubblemac/tacticum/helm/data/iva/tasks_rich/task_007650.jsonl`:

```
IVAONEHALF-121 · "[BlockedByBack][ВКС] ВКС: Смена владельца записи ВКС"
статус "Приостановлено" · cat not_started · приоритет Низкий · компонент Desktop
репортер Корнюшин Пётр · assignee Unassigned
создана 2026-07-07 · обновлена 2026-07-10
labels: draft_KMP_DRAFT_118, kmp_import_2026_07_07, kmp_task_import
описание ссылается на строку каталога № 58 (Область: Функциональные требования, Компонент: ВКС)
```

### 4.3 Смежная смена владельца в ИВА РЕАЛИЗОВАНА и имеет готовые постановки

Готовые постановки в вики (`/Users/bubblemac/tacticum/helm/data/real/wiki/pages_index.csv`):

| Строка | Спейс | page_id | Заголовок | Автор, дата |
|---|---|---|---|---|
| `:3145` | IM | 124084755 | Передача прав владельца и администратора в чатах и каналах | Тарасова Надежда, 2025-03-26 |
| `:7811` | IVCS | 49742869 | Отображение информации о владельце демонстрации в мероприятии | petr.sukhov, 2021-02-24 |
| `:9532` | IVCS | 72583825 | **Изменение владельца мероприятия** | Сухов Сергей, 2022-04-07 |
| `:22121` | IVCS | 155454413 | Назначение владельцем чата удаленного пользователя | Касьяненко Анастасия, 2025-07-31 |
| `:23342` | IVCS | 165709304 | **Просмотр информации о чате и смена владельца в чате** | Воронин Артём, 2026-01-29 |
| `:24002` | IVCS | 171345358 | Развитие владельцев фич и модулей | Кожанов Егор, 2026-04-06 |

Закрытые Jira-задачи «передача прав владельца» по всем платформам
(`/Users/bubblemac/tacticum/helm/data/iva/tasks_rich/`):

- `IVAONE-1520` «[BACK] Добавить функциональность передачи прав владельца канала/чата», Закрыт, 2024-04-16
- `IVAONE-1524` «[Backend] Передача прав владельца чата», Закрыт, 2024-04-16
- `IVAONE-3356` «Передача прав владельца чата», Закрыт, 2024-10-21
- `IVAONE-3727` iOS · `IVAONE-3728` Android · `IVAONE-3729` WEB · `IVAONE-3730` Desktop —
  «Добавить функциональность передачи прав владельца канала/чата», все Закрыт, 2024-11-18
- `IVAONE-8752` «После передачи прав владельца "слетает" аватар чата…», Закрыт, 2025-10-20
- `IVAONE-11690` «[iOS][Чаты] Можно передать права владельца чата удалённому пользователю», Закрыт, 2026-04-22
- `IVAONE-11440` «[iOS][Чаты] Не закрывается экран участники чата при передаче прав владельца», Закрыт, 2026-04-10
- `VCSWEB-7743` «Запретить назначение владельцем чата remote пользователей», Закрыт, 2026-06-22
  (`/Users/bubblemac/tacticum/helm/data/real/jira/jira_issues.csv:472`)
- `IVAUC2-685` «Создать тест кейс "можно назначить владельцем чата пользователя которого удалили"», Закрыт, 2025-09-04

## 5. tacticum-dev — по владению записи практически пусто

- `Владелец: mr.diaret@ya.ru` в шапке ~20 ADR — это метаданные документа, не модель владения.
- `/Users/bubblemac/tacticum/tacticum-dev/apps/backend/alembic/versions/0019_profile_owner_organization.py:1`
  — `profiles.owner_organization_id` (ADR-0010, Issue #289, 2026-05-28): direct-FK ownership
  профиля организацией. Строка `:13` упоминает «or reassign ownership first» **как ручное
  действие оператора — операции переназначения не реализовано**.
- `/Users/bubblemac/tacticum/tacticum-dev/docs/adr/0051-project-hub-as-single-idp-and-rbac-authority.md`
  — RBAC/identity, но не владение записью.
- Поиск по `смена владельца|передача владения|reassign|transfer_owner|change_owner` по всему
  репо даёт **одно** попадание — тот самый докстринг миграции.

## 6. Вывод фактом

**ЕСТЬ:**
1. В самом продукте ИВА требование **дословно «ВКС: Смена владельца записи ВКС»** зарегистрировано
   в четырёх каталогах требований (1.0 Клиенты №168, 1.0 Полный перечень №191, IVA 1.5 с заказчиком
   Мосбиржа, ВКС 1.5 №59), имеет вердикт реализации `absent`/«Отсутствует» с обоснованием по коду
   и достоверностью «Высокая», и по нему заведена задача **IVAONEHALF-121** (создана 2026-07-07,
   «Приостановлено», BlockedByBack).
2. Смежная смена владельца в ИВА **реализована и задокументирована**: передача прав владельца
   чата/канала закрыта по всем пяти платформам в 2024, изменение владельца мероприятия
   специфицировано в 2022, смена владельца в чате — постановка Воронина от 2026-01-29.
   Это готовые постановки-аналоги ровно того класса, который просит `nearest_spec`.
3. В helm есть **полный слой владения** (ADR-0008, принят 2026-07-26) с моделью, RBAC, REST и UI,
   включая эндпоинт, чей докстринг буквально «Сменить владельца … — с аудитом»
   (`cio.py:4561`), редактор ответственных, матрицу владения и назначение владельцев стадий
   с событием `owner_assigned`. Всё это существовало **до 31.07**.

**НЕТ:**
1. В продукте ИВА **смена владельца записи ВКС действительно не реализована** — это
   зафиксированный статус требования, а не пробел в знаниях.
2. В helm **нет единой атомарной операции «передать владение»** (transfer/reassign):
   в assignment-слое смена собирается из add + soft-remove + set_primary; ADR-0008 сценарий
   передачи владения как таковой не описывает (в тексте ADR нет ни «смена владельца»,
   ни «передача владения» — только «история/замы» в §Последствия, строка 119).
3. **В MCP-слое владения нет вообще** — ни в аналитическом сервере (19 тулов, ни одного по
   `ownership_assignment`), ни возможности сменить владельца в process-сервере (только чтение
   `owner_person_id`). Через профиль аналитика этот слой невидим — та же картина, что с
   `needs_clarification`.
4. В tacticum-dev опоры по смене владельца записи нет.

**Что могло бы служить референсом аналитику:** вики-страницы IM 124084755, IVCS 165709304,
IVCS 72583825, IVCS 155454413; закрытые задачи IVAONE-1520/1524/3727-3730; строка каталога
ВКС 1.5 №59 с формулировкой и вердиктом; задача IVAONEHALF-121.

## Чего я НЕ смог установить

- **Реально ли эти материалы находятся в индексе RAG**, который читают `analyst_search`,
  `analyst_context`, `nearest_spec`. Локально в `data/real/confluence/bodies/` лежит **6 тел
  страниц** при 3154 строках в `pages_index.csv`; `data/real/wiki/` содержит **только индекс,
  без тел**. Проверить наполнение Qdrant без поднятого стенда я не могу. То есть «материал
  существует» я доказал, «профиль аналитика мог его достать» — нет.
- **Содержимое реестров OpenAPI ИВА** (`api_registry_check`): каталог `data/real/api`
  (`config.py:433`, реестры `clients,integration,bot`) локально отсутствует — наполняется джобой
  `helm.ingest.api_index` с `beta.hi-tech.org`. Есть ли там операция смены владельца записи —
  не установлено.
- **Тела вики-страниц** IM 124084755 и IVCS 165709304/72583825 — в репозитории их нет, судить
  можно только по заголовкам из индекса.
- **Что именно видел агент-аналитик в эксперименте 31.07** — лога у меня нет; по прямому
  запрету не гадаю.
- **Есть ли требование «Смена владельца записи ВКС» как запись `Requirement` в БД helm** —
  проверял только файлы `data/`, к работающей БД не обращался.