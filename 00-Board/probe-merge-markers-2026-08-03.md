---
title: Механизм установки инструкционных фрагментов — дубль маркеров и моджибейк в
  AGENTS.md аналитика
type: note
status: draft
created: 2026-08-03
tags:
- board
- профили
- механизм
permalink: tacticum/00-board/probe-merge-markers-2026-08-03
---

# Механизм установки фрагментов: где правится дубль маркеров

Репозиторий `/Users/bubblemac/tacticum/tacticum-dev`, всё читалось на `origin/main` = `f6a33ea`
(локальный HEAD `b612fd9` отстаёт на 55 коммитов; расхождений по разобранным файлам не проверял
построчно — все цитаты ниже строго из `origin/main`).

## Главный вывод одной фразой

**Дубль маркеров — не дефект сервера и не дефект шаблона: сервер отдаёт тело, обёрнутое ровно
одной парой маркеров (это запинено тестами по байтам), а шаблоны маркеров в теле не содержат.
Всё, что происходит с файлом на диске, делает клиент (LLM-агент Codex), исполняя прозу-псевдокод
из скилла. Значит это дефект контракта установки — у нас нет ни одной строки исполняемого кода,
которая бы вставляла блок в существующий `AGENTS.md`, и нет ни одной проверки результата.**
Ручные действия пользователя как причина не исключены (телеметрии применения нет), но не нужны для
объяснения: наблюдаемая картина полностью воспроизводится ошибкой агента при исполнении нашей же
инструкции.

---

## 1. Где реализован `merge_strategy: append_section` и как используется `marker_id`

Сервер **не пишет файлы**. Он рендерит список деклараций-actions; `merge_strategy` + `marker_id`
кладутся в action как данные, а маркеры вшиваются В САМО ТЕЛО контента.

Точка правки — `apps/backend/src/backend/catalog/domain/renderer.py:121` (`_wrap_body_with_markers`):

```python
def _wrap_body_with_markers(action: dict[str, Any]) -> dict[str, Any]:
    if action.get("action") != "write_file":
        return action
    marker_id = action.get("marker_id")
    if not marker_id or action.get("merge_strategy") != "append_section":
        return action
    body = action.get("content", "")
    # Suppress only when the WRAPPER is already present — a prose mention of the
    # marker_id (e.g. go-profile copilot pack "Marker: tacticum:…") must still wrap.
    if f"<!-- {marker_id} -->" in body:
        return action
    if body and not body.endswith("\n"):
        body += "\n"
    wrapped = dict(action)
    wrapped["content"] = f"<!-- {marker_id} -->\n{body}<!-- /{marker_id} -->\n"
    return wrapped
```

Вызывается на обоих путях доставки:
- `renderer.py:244` — `render_for_claude_code` (`return [_annotate_manual_apply(_wrap_body_with_markers(a)) for a in actions]`);
- `renderer.py:364` — `_render_via_canonical` (путь Codex/Copilot), после склейки одноимённых path
  в `_dedupe_actions_by_path` (`renderer.py:367`).

Откуда берётся `marker_id`: из метаданных ингредиента, `renderer.py:209-210` (repo_config) и
`renderer.py:226-227` (instruction_pack). Модель — `catalog/domain/ingredients/instruction_pack.py:10,17,18`
(`MergeStrategy = Literal["replace", "append_section", "deep_merge", "create_if_missing"]`,
дефолт `append_section`). В манифесте профиля — `templates/iva-fr-analyst/manifest.yaml:234-237`
(`marker_id: "tacticum:iva-fr-analyst"`), `templates/iva-system-analyst/manifest.yaml:128-131`,
`templates/iva-role-analyst/manifest.yaml:93-97`. Формат маркера, который реально едет клиенту:
`<!-- tacticum:iva-fr-analyst -->` … `<!-- /tacticum:iva-fr-analyst -->`.

В манифест для клиента `marker_id` пробрасывается в `actions_meta`:
`workspace/interface/mcp/pull_installation_content_manifest.py:109` и
`catalog/interface/mcp/tacticum_init_manifest.py:124`.

## 2. Проверяется ли наличие маркера перед вставкой — идемпотентна ли повторная установка

**На сервере — нечего проверять: он не видит диск.** Идемпотентность целиком на клиенте, и
предписана она ПРОЗОЙ, а не кодом. Единственное место с алгоритмом — псевдокод в скилле
`catalog/domain/builtins/tacticum_context_skill.md:160-181`:

```
#      - "append_section" (comes with `marker_id`): edit ONLY the block
#        between `<!-- {marker_id} -->` and `<!-- /{marker_id} -->` —
#        replace it if present, append it (with markers) if absent.
    elif strategy == "append_section" and exists(full["path"]):
        replace_or_append_marked_section(full["path"], full["marker_id"], full["content"])
```

Функции `replace_or_append_marked_section` в репозитории **не существует** — грепом по `origin/main`
она встречается только в трёх markdown-файлах (сам скилл и две его копии в шаблонах
`templates/tacticum-core-base/…/SKILL.md:175`, `templates/tacticum-dev-base/…/SKILL.md:178`).
Исполняет это LLM своими руками.

Единственная работающая реализация — тестовый оракул `tests/e2e_install/oracles.py:74-88`:

```python
if strategy == "append_section" and action.get("marker_id") and dest.exists():
    marker = action["marker_id"]
    start, end = f"<!-- {marker} -->", f"<!-- /{marker} -->"
    section = action["content"]          # уже обёрнут маркерами рендерером
    text = dest.read_text(encoding="utf-8")
    if start in text and end in text:
        text = text.split(start)[0] + section + text.split(end, 1)[1]
    else:
        text = text.rstrip("\n") + "\n\n" + section
```

Регулярки нет — поиск подстрокой. Поведение при уже существующем блоке: **заменяет** (по ПЕРВОМУ
вхождению открывающего и ПЕРВОМУ закрывающему; всё, что между вторым и последующими маркерами,
остаётся справа от вставки — отсюда и берётся эффект «вложенности»). Не падает никогда. Но это
код тестов, на машине аналитика он не исполняется.

**Два наших документа противоречат друг другу и скиллу** — это отдельный источник дефекта:
- `docs/agents/codex-init.md:272` описывает маркеры формата `<!-- tacticum:iva-brownfield-mail:start -->` /
  `:end` — это формат старого PowerShell-инсталлятора (`templates/*/scripts/apply.ps1:126-127`), а не
  того, что сейчас отдаёт сервер. Агент, идущий по этому доку, своего блока НЕ найдёт и допишет второй.
- `docs/agents/installing-new-profile.md:193-195` даёт `upsert_section(path, marker=…, body=body)` и
  НИГДЕ не говорит, что body уже обёрнут маркерами. Агент, который «оборачивает сам», получает ровно
  наблюдаемое: два открывающих подряд и два закрывающих подряд вокруг одного блока.

## 3. Что происходит при установке ВТОРОГО профиля в ту же папку

Блоки просто конкатенируются (ветка `else` выше — append в конец файла). **Проверки на конфликт
профилей в одной директории нет нигде и быть не может: сервер вообще не знает про директории.**
`Installation` уникален по `(workspace_id, profile_id)` — см.
`workspace/application/provision_installation.py` (док-строка про `uq_installations_workspace_profile`
и ветка идемпотентности в конце функции). Ни поля с путём, ни привязки к папке в модели нет.
Единственный локальный след установки — `.tacticum/context.yaml` с ОДНИМ `installation_id`; при
установке второго профиля в ту же папку он молча перезаписывается (шаг 4 «Correct flow» в скилле).

Именно это и произошло у аналитика: `iva-fr-analyst` (`templates/iva-fr-analyst/manifest.yaml:10`)
и `iva-system-analyst` (`manifest.yaml:10`) оба помечены `superseded_by: iva-role-analyst`, то есть
папка проходила МИГРАЦИЮ на роль. Процедура миграции —
`tacticum_context_skill.md:296-395`, и её **шаг 6 CLEANUP прямо требует** удалить legacy-секции:

```
#    Then in CLAUDE.md / AGENTS.md: remove the legacy marked section
#    <!-- {legacy marker_id} --> ... <!-- /{legacy marker_id} -->
```

Оба legacy-блока на месте → шаг 6 не выполнен. Шаг 8 (чек-лист приёмки, пункт «(4) CLAUDE.md /
AGENTS.md contain markers of the role ONLY») тоже не выполнен либо выполнен формально. Это проза
без единой автоматической проверки: сервер верифицирует только `check_updates`, то есть факт
скачивания, а не состояние файла.

## 4. Кодировка — не наша

Бэкенд отдаёт контент как Python `str` из БД в JSON-ответе MCP (FastMCP streamable-http,
`platform/app_factory.py`); своих `Content-Type`/`charset` мы нигде не выставляем — грепа по
`charset`/`media_type` в `apps/backend/src` даёт только `application/json` в
`catalog/interface/admin/render_preview.py:64` и заголовки исходящих запросов к эмбеддеру.
**Записи файлов на диск в бэкенде нет вообще.** Всё файловое чтение — с явным `encoding="utf-8"`:
`catalog/domain/builtins/builtins.py:52`, `design/application/seed_runner.py:41-42`,
`platform/stats.py:177`, `platform/observability.py:46`, и сидирование тел ингредиентов
`apps/backend/scripts/seed_community.py:66,87` (`read_text(encoding="utf-8")`). Файлов без явной
кодировки не найдено ни в `apps/backend/src`, ни в `apps/backend/scripts`, ни в `scripts`.

Природа моджибейка определена точно. Проверка:

```
'# Tacticum — роль «Аналитик ИВА»'.encode('utf-8').decode('latin-1')
→ '# Tacticum â ÑÐ¾Ð»Ñ Â«ÐÐ½Ð°Ð»Ð¸ÑÐ¸Ðº ÐÐÐÂ»'
```

— совпадает с наблюдаемым посимвольно. `cp1251` и `cp1252` на этих байтах падают, `cp866` даёт
другую картинку. То есть UTF-8 байты были **декодированы как latin-1/ISO-8859-1 и записаны обратно
в UTF-8**. Классическая причина — HTTP-клиент, применяющий к телу без явного `charset` дефолт
`ISO-8859-1` из RFC 2616 (для `text/event-stream` это живая грабля), либо запись файла клиентом с
кодировкой платформы.

Ключевая деталь, сужающая место поломки: **битый только новый блок `iva-role-analyst`, а старые
блоки читаемы**. Если бы агент читал существующий `AGENTS.md` в latin-1 и писал в utf-8, поехал бы
ВЕСЬ файл. Значит испорчен именно доставленный контент — на участке «MCP-ответ → буфер клиента»,
уже после нашего процесса. Первая строка битого блока —
`templates/iva-role-analyst/ingredients/repo-configs/codex/AGENTS.md.fragment:1`
(`# Tacticum — роль «Аналитик ИВА» (iva-role-analyst)`), в шаблоне она в нормальном UTF-8.

**Вывод: с нас пункт снимается по коду.** Что стоит проверить у себя (я не мог): нет ли моджибейка
уже в БД в `body` ингредиента — тогда виноват не клиент, а конкретный прогон сидирования.

## 5. Тесты

Тесты на байты доставки есть, на состояние папки после повторной установки — **нет**.

Что есть:
- `tests/catalog/domain/test_marker_wrap.py` — обёртка ровно одной парой маркеров
  (`test_append_section_pack_body_is_wrapped_in_markers_exactly_once:47`), отсутствие двойной
  обёртки для уже обёрнутого донора (`test_already_html_wrapped_body_is_not_double_wrapped:91`),
  неприменение обёртки к `create_if_missing`, регресс на прозаическое упоминание marker_id.
- `tests/e2e_install/test_incident_677_claude_md.py` — установка на РАНЕЕ СУЩЕСТВУЮЩИЙ
  `CLAUDE.md`: контент репозитория выживает, секция ровно одна (`assert_markers_once`,
  `oracles.py:188`).
- `tests/e2e_install/test_install_flow.py:1216-1227` — то же для `AGENTS.md`/Codex.

Чего НЕТ (прямо, для ТЗ):
1. **Нет ни одного теста, применяющего actions ДВАЖДЫ в одну и ту же директорию.** Все 13 вызовов
   `apply_actions` бьют в свежие `tmp_path` (`tree_a`, `tree_b` — это сравнение bulk-пути с
   manifest-путём, а не повторная установка). Идемпотентность повторной установки не покрыта.
2. **Нет теста на два разных профиля в одной директории** — ни на конкатенацию, ни на конфликт.
3. **Нет теста на сценарий миграции целиком** (шаг 6 CLEANUP: удаление legacy-секции при наличии
   блока роли). Есть только `superseded`-блок в `check_updates` на уровне API.
4. **Нет теста на кодировку доставленного тела** (кириллица в `AGENTS.md` после применения).
5. `assert_markers_once` (`oracles.py:188-212`) считает маркеры ОДНОГО профиля и только в
   `CLAUDE.md`/`AGENTS.md` — картину «три профиля в одном файле» она бы не поймала.

## 6. Механизм запрета/предупреждения о двух профилях в одной директории

**Запрета нет. Предупреждения о ДИРЕКТОРИИ нет.** Есть ровно одно смежное:
`workspace/interface/mcp/check_updates.py:108-149` — если у профиля выставлен `superseded_by`,
ответ несёт блок `superseded` с текстом «предложи миграцию одной фразой и продолжай текущую
задачу; запускай процедуру ТОЛЬКО по явной просьбе пользователя». Это подсказка агенту, не гейт.

`provision_installation` (`workspace/application/provision_installation.py:60`) валидирует только
видимость профиля, наличие активной версии и разрешение воркспейса; коды ошибок —
`profile_not_found`, `no_active_version`, `workspace_not_found`, `workspace_ambiguous`,
`workspace_forbidden`, `installation_archived`. Ни `superseded_by`, ни совместимость лейнов, ни
директория в валидацию не входят: **можно спокойно провижнить и legacy-профиль, и роль, и вторую
роль — сервер не возразит ни разу.**

---

## Какие наблюдения различили бы версии (если нужен вердикт «агент против рук»)

1. `git log -p` по `AGENTS.md` в папке аналитика (если она под git): один коммит с готовым битым
   блоком = машина; серия правок = руки.
2. Наличие BOM / `\r\n` только вокруг нового блока: PowerShell `Set-Content -Encoding utf8` в
   PS 5.1 пишет BOM — след старого `apply.ps1`-пути.
3. Формат маркеров у legacy-блоков: `:start`/`:end` = ставил `apply.ps1` или агент по устаревшему
   `docs/agents/codex-init.md`; `<!-- id -->`/`<!-- /id -->` = серверный путь MCP.
4. `.tacticum/context.yaml` в папке: на какой `installation_id` он указывает сейчас — если на роль,
   миграция дошла до шага 7, но пропустила шаг 6; если на legacy — оборвалась раньше.
5. Логи сервера `installation_content_manifest_pulled` (`pull_installation_content_manifest.py:135`)
   по трём installation — сколько раз и когда тянули каждый профиль.

## Чего я НЕ смог установить

- **Не видел сам файл `AGENTS.md` аналитика** (Windows-машина) — всё выше строится на описании
  симптомов из постановки. Точный порядок и формат маркеров в файле не проверял.
- **Не смотрел в БД**: не могу сказать, лежит ли в `ingredients.body` для `iva-role-analyst`
  нормальный UTF-8 или уже битый. Это единственная оставшаяся гипотеза, при которой пункт 4
  возвращается к нам.
- **Не проверял, какую именно версию скилла `tacticum-context` держит клиент аналитика.** Ветка
  миграции появилась в `df8986b`; если на диске лежит скилл старее — агент исполнял другой текст,
  и часть выводов про шаг 6 к нему не относится.
- **Не установил, каким клиентом и когда ставились legacy-профили** (MCP-путь или `apply.ps1`) —
  различающий признак назван выше (формат маркера), но проверить его без файла нельзя.
- **Не проверял построчно расхождение локального `b612fd9` с `origin/main`** по разобранным файлам;
  всё читалось через `git show origin/main:<путь>`.