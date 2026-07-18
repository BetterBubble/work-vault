---
title: rag2-code-audit
type: report
permalink: tacticum/04-sessions/rag2-code-audit
tags:
- rag2
- audit
- helm
- analyst-mcp
---

# RAG#2 / analyst-MCP — независимый код-аудит Р-1…Р-5

Read-only аудит реализаций требований Р-1…Р-5 в `~/tacticum/helm` (main, всё смержено).
Метод: символьная разведка (Serena) + чтение кода/тестов. Сервис не нагружался (retrieval/rerank не гонялись).

Связано: [[rag2-ab-measurements]]

## Итог
- Дефектов: **1** (medium-high).
- Рисков: **2** (1 medium — фиктивное покрытие парсера; 1 low — взаимодействие).
- Заметок/coverage-оговорок: 3.
- Критичных (роняющих сервис) — нет.

Ранжировано по серьёзности.

## Observations
- [defect] Р-2×дотяжка: noise_floor=drop (прод τ=0.5, commit 71b64a3) роняет запиненную present-but-low exact-key задачу, если реранк дал confidence<τ #rag2 #noise-floor #exact-key
- [risk] Р-4: парсер Sessions.html (HTML→команды) покрыт только синтетикой; гейт имени команды может терять команды на реальном pandoc-HTML #rag2 #contract-registry #test-gap
- [risk] Р-3: _is_nl_query считает токены по РАСШИРЕННОМУ запросу → code-expand может понизить ft-вес #rag2 #code-term
- [clean] Р-1 api_registry: token-boundary + конъюнкция корректны; тесты явно проверяют присутствие adversarial-операций #rag2 #api-registry
- [clean] Дотяжка exact-key: tenant fail-closed железный, тест инспектирует реальное тело Qdrant #rag2 #tenant-isolation
- [clean] Р-5 effort_hint: active_days=None при пустом changelog, не 0.0; тесты честные #rag2 #effort-hint

---

## 🔴 [ДЕФЕКТ, medium-high] Р-2×дотяжка: noise_floor=drop роняет запиненную exact-key задачу
**Где:** `application/rag2.py:295-324` (порядок pin→merge→floor), `domain/rag2.py:467-487` (`apply_noise_floor`), `infrastructure/rag2/reranker.py:54-57` (реранк ставит confidence на ВСЕ доки).

Порядок в `answer`: `boost_exact_keys` (пин) → `merge` → `apply_noise_floor(floor, action)` → `[:k]`. Floor стоит корректно (последним, по confidence). НО `JiraReranker.rerank` проставляет `confidence` КАЖДОМУ реранкнутому доку, включая present-but-low exact-key. Если реранк оценил названную задачу ниже τ и `noise_action="drop"` — задача **выкидывается из выдачи, несмотря на пин**.

В проде включено: commit `71b64a3` включил noise-floor τ=0.5 drop → дефект живой.

Repro: запрос «статус IVAONE-7752?», задача present-but-low, кросс-энкодер по короткому key-запросу даёт ~0.3 (<0.5). Пин ставит #1 → floor дропает. Аналитик спросил задачу по ключу и НЕ получил её.

Почему не поймали тесты: в `test_rag2_orchestrator.py` все exact-key доки идут с `confidence=None` → floor их никогда не режет (None-never-noise). Совмещённого кейса «pin + drop + низкий confidence» нет; ветки exact-key и noise_floor-drop тестируются раздельно.

Нюанс для фикса: retrieval-miss путь БЕЗОПАСЕН (задача из `fetch_by_keys` приходит с confidence=None, реранка нет). Уязвим ТОЛЬКО present-but-low путь. Направление: floor-иммунитет для запиненных exact-key (или tag вместо drop).

## 🟠 [РИСК, medium] Р-4: реальный парсинг Sessions.html не покрыт — тихая потеря команд
**Где:** `ingest/jump_contracts.py:115-207`; тесты `tests/ingest/test_jump_contracts.py`, `tests/domain/test_contract_registry_golden.py`.

Матчер контрактов тестируется на РЕАЛЬНЫХ командах, НО они берутся из рукотворного `data/contracts/jump-subset.commands.json`, а не из парсинга живого pandoc-`Sessions.html`. Сам парсер HTML→команды тестируется только на синтетической чистой фикстуре `_HTML`.

Дыра: гейт `_COMMAND_NAME_RE = ^[A-Za-z][A-Za-z0-9]*$` требует, чтобы `<h3>` был ровно одним camelCase-токеном. Реальный pandoc часто кладёт в h3 якорь `<a class="anchor">`, `¶`, номер раздела, nbsp → title склеится → `_is_command_name`=False → **команда молча теряется**. Полный golden (101 команда) — skipif без артефакта, в CI не идёт.

Доп: дедуп по имени (`title in seen`) схлопнет две разные команды с одинаковым h3-текстом. Направление: прогнать парсер на реальном срезе, сверить command_count=101.

## 🟡 [РИСК, low] Р-3×hybrid: NL-гейт по расширенному запросу
**Где:** `infrastructure/rag2/search.py:192-204, 245-274`.

При `expand_code_terms=on` в `_hybrid` уходит уже расширенный `text`, `_is_nl_query` считает по нему. Code-expand добавляет формы → короткий запрос переваливает `NL_MIN_TOKENS=6` → ft_weight понижается (при `hybrid_ft_weight<1`) → ослабляет exact-match ровно для code-запросов. Реранк использует исходный query (ок), гейт — нет. Стоит гейтить по исходному query.

## 🟢 Р-1 api_registry — ЧИСТО
`domain/api_registry.py`, `api_terms.py`, `ingest/api_openapi.py`.
- Token-boundary `_token_hit` (api_registry.py:46-54) — `mail⊂email` устранён. Конъюнкция + identifier-гейт корректны. Дедуп по (method,path,operationId). `parse_openapi` не теряет операции (opId=None → матч по токенам пути).
- Тесты честные: `test_recall_email_not_found_despite_email_ops_present` ЯВНО проверяет присутствие `{checkEmail,resendEmail,revokeAttentionRequest}` (строки 73-74) перед not_found.
- Заметки: порог 2.0 → голый object-концепт поднимает все email-операции (шумно, не неверно); fallback identifier-only→1.0 мёртвый (безвредно); ascii-подтокены без множ.числа; object-only требует ВСЕ объекты (точность>полнота, by design); полный golden 419 skipif в CI.

## 🟢 Р-4 матчер, Дотяжка (tenant), Р-2 floor изолированно, Р-5 — ЧИСТО
- **Р-4 матчер:** overlay `_expand_for_contracts` (письмо⇒message JUMP) изолирован, api_terms не трогает — регресса Р-1 нет.
- **Дотяжка tenant fail-closed железный:** `vectorstore.fetch_by_keys` кладёт tenant_id в must рядом с key; пустой tenant→[] без сети. Тест `test_fetch_by_keys_filters_tenant_and_keys` инспектирует реальное тело Qdrant-scroll. Обхода нет. KNOWN_PROJECTS — длинные префиксы первыми.
- **Р-2 floor изолированно:** normalize_confidence/apply_noise_floor корректны; None-never-noise; federate сохраняет confidence поверх RRF. Честные юниты (test_rag2.py:287-348).
- **Р-5 effort_hint:** `active_days=None` при пустом changelog (не 0.0), исключается из медиан, coverage-телеметрия + note; 0.0 только при реальных переходах без dev-времени. Тесты прямо проверяют null-not-zero и живой баг. Мелочь (недостижимо): mixed tz-aware/naive datetime без guard, но compact-формат витрины единообразен.

## Сводка вердиктов
| Требование | Вердикт |
|---|---|
| Р-1 api_registry | чисто |
| Р-4 contract_registry (матчер) | чисто; **риск: парсер HTML не покрыт** |
| Р-2 floor+confidence | чисто изолированно; **дефект взаимодействия pin×floor** |
| Дотяжка exact-key (tenant) | чисто (fail-closed железный) |
| Р-3 code-term | чисто; low-риск NL-гейт по расширенному запросу |
| Р-5 effort_hint changelog | чисто |

## Relations
- relates_to [[rag2-ab-measurements]]

---

## 🟠 [РИСК precision, medium] Р-1/Р-4: weak+weak co-occurrence = ложное срабатывание матчера
Добавлено при строгом доп-проходе по приоритету «ложные срабатывания» (детерминированные матчеры идут аналитикам как «зелёные»).

**Где:** `domain/api_registry.py:79-119` (`_OpIndex.score`), зеркально `domain/contract_registry.py:91-127` (`_CmdIndex.score`). `_W_STRONG=2.0`, `_W_WEAK=1.0`, `DEFAULT_MIN_SCORE=2.0`.

**Суть.** Скоринг НЕ требует ни одного STRONG-совпадения (opId/путь/имя команды). Для action+object запроса гейт конъюнкции проходит при СЛАБЫХ совпадениях обоих (weak-action 1.0 + weak-object 1.0 = 2.0 = порог). Значит операция, у которой глагол и объект встретились ТОЛЬКО в `summary`/`tags` (weak-поле) — случайно, не по смыслу — попадает в выдачу как found.

**Repro (концепт).** Запрос «удалить сообщение» → action(remove,delete)+object(message). Операция `getInbox` с summary «Send and delete are not supported here for messages»: `delete` → weak 1.0, `messages`→`message` (плюрал-handling `_token_hit`) → weak 1.0, итого 2.0 ≥ порог → `getInbox` в matches, хотя операция не про удаление сообщений.

**Последствия.**
- В основном это ЗАСОРЕНИЕ списка matches (правильная операция с strong-хитом = 4.0 ранжируется выше, топ обычно верный).
- Но для negative-запросов может ПЕРЕВЕРНУТЬ `found` False→True (ложный «нашлось»), если у случайной операции summary содержит оба точных токена. Именно это подрывает доверие к «детерминированному not_found».

**Тест-гэп.** Ни один тест не проверяет, что чисто-weak (summary-only) co-occurrence отклоняется. Все позитивные golden-кейсы матчат по strong (opId/path). То есть precision по weak-полю не покрыта — тест «зелёный», но реальную границу weak-фолс-позитива не щупает.

**Направление (не реализую):** требовать ≥1 strong-хит для found, либо поднять порог до 2.5+ (weak+weak не проходит, weak+strong=3.0 проходит), либо давать found по weak только когда нет ни одного strong-кандидата. Под целевой тест на summary-only ложное срабатывание.

**Оговорка честности:** вектор data-dependent (нужны оба точных токена в summary неродственной операции). На текущих спеках ИВА golden-негативы (`отозвать письмо`, `забронировать переговорку`) не триггерятся — но это удача словаря, а не гарантия конструкции.

### Доп-подтверждение приоритета (b) tenant fail-closed — обхода НЕТ
Строгая перепроверка: tenant фиксируется при wiring (`self._tenant`), аналитик его не инъектит; `filters` только ДОБАВляются в `must` (не вместо tenant); все три точки (`search`/`fetch_by_keys` стора + `JiraIndexSearch.fetch_by_keys`) fail-closed на пустой tenant → `[]`. Тест инспектирует реальное тело Qdrant. Единственная мелочь: `JiraVectorStore.count(tenant_id=None)` считает по всем тенантам — но это debug/CLI-метод, не в пути аналитика (не утечка данных, только счётчик).
