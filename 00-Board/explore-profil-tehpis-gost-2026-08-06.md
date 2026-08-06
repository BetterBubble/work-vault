---
title: 'Разведка: профиль техписа под ГОСТ-документацию — анатомия, что уже есть,
  гэпы'
type: report
status: draft
created: 2026-08-06
tags:
- board
- gost-docs
- profiles
- explore
permalink: tacticum/00-board/explore-profil-tehpis-gost-2026-08-06-1
---

# Разведка: решается ли ГОСТ-документация профилем tacticum-dev

Репозиторий: `/Users/bubblemac/tacticum/tacticum-dev`, ветка `main`, HEAD `b612fd90`.
Serena по репо не поднималась — материал целиком markdown/yaml (манифесты, SKILL.md, ADR),
символов нет, разведка велась чтением файлов и поиском по тексту. Для этого материала
это адекватный инструмент, потери call-sites тут не возникает.

## 1. Анатомия doc-профиля — что нужно написать, чтобы появился новый

Актуальная модель — **не «профиль-монолит», а лейны + роль-пресет** (ADR-0057 →
ADR-0059, `docs/adr/0059-single-axis-process-lanes-and-role-packs.md:1`). Образец из
постановки — `templates/iva-fr-analyst` — **депрекирован по назначению**:
`templates/iva-fr-analyst/manifest.yaml:10` → `superseded_by: iva-role-analyst`.
Живой владелец его содержимого — лейн `iva-analysis-fr` (`templates/_mirrors.yaml:21-30`,
пара owner=iva-analysis-fr / mirror=iva-fr-analyst, байт-в-байт под CI-сверкой
`scripts/check_mirror_sync.py`).

**Формула роли** (ADR-0059 Решение 1): `роль = core + процесс-лейны + стек-лейны + способность-лейны`.
Пример: `templates/iva-role-analyst/manifest.yaml:20-24` →
`[tacticum-core-base, tacticum-analysis-core, iva-analysis-fr, tacticum-research-base]`.

**Файловая анатомия профиля** (на примере `templates/iva-fr-analyst`, 13 файлов):

```
templates/<profile-id>/
  manifest.yaml                      # обязателен: profile_id, version, persona, ingredients[]
  README.md, CHANGELOG.md
  ingredients/
    skills/<id>/SKILL.md             # kind: skill_spec  → .claude/skills/<id>/SKILL.md
    skills/<id>/references/*.md      # доп. файлы скилла (progressive disclosure)
    commands/<id>.md                 # kind: command_spec → слэш-команда
    agents/<id>.md                   # kind: agent_spec   → субагент (model, tools, description)
    repo-configs/claude-code/CLAUDE.md.fragment   # kind: instruction_pack (merge_strategy)
    repo-configs/codex/AGENTS.md.fragment
    repo-configs/codex/config.toml.template
```

Ингредиент объявляется в `manifest.yaml` записью с `kind` из 9 типов
(`templates/_schema/ingredient.v1.schema.json:10-16`): `instruction_pack`, `rule_set`,
`agent_spec`, `skill_spec`, `mcp_server_spec`, `command_spec`, `hook_spec`,
`permission_policy`, `repo_config`. Ключевые поля записи —
`templates/iva-fr-analyst/manifest.yaml:53-63`: `tier`, `supports: [claude-code, codex]`,
`install_scope: user|repo`, `target_path_template`, `codex_target_path`, `body_path`,
`metadata.description_trigger`.

- **MCP** объявляется тем же способом, телом-пустышкой + metadata:
  `templates/iva-fr-analyst/manifest.yaml:152-214` (helm-analyst, iva-mcp,
  iva-atlassian-write, tacticum-mcp; `transport`, `url`, `env_required`, `auth_type`).
  Скоуп тулов задаётся `metadata.allowed_tools` —
  `templates/iva-techwriter-mcp/manifest.yaml:92-95`.
- **Роль-пресет = pure-composition leaf:** `ingredients: []` +
  `depends_on` (`templates/tacticum-role-techwriter/manifest.yaml:27-30,104`).
- **Инварианты приёмки нового профиля:** валидация JSON-схемой
  (`CONTRIBUTING.md:127`), версия новых профилей строго `0.1.0` (`CONTRIBUTING.md:135`),
  голден-снапшот установки — `apps/backend/tests/e2e_install/golden/<profile>/claude-code.json`
  и `codex.json` (карта путь→sha256 всех разложенных файлов; 22 голдена сейчас),
  и mirror-дисциплина, если ингредиент зеркалится.

## 2. Есть ли уже техпис — НЕТ гринфилда, роль собрана

`iva-tech-writer` в репозитории **не существует**. Но существует три готовых кирпича:

| Что | Путь | Версия | Содержимое |
|---|---|---|---|
| `tacticum-role-techwriter` | `templates/tacticum-role-techwriter/manifest.yaml` | 0.3.0 | роль-пресет, `ingredients: []`, `depends_on: [tacticum-core-base, tacticum-documentation-base, iva-techwriter-mcp]` |
| `tacticum-documentation-base` | `templates/tacticum-documentation-base/manifest.yaml` | 0.1.0 | лейн документации, 1 ингредиент — скилл `doc-authoring` |
| `iva-techwriter-mcp` | `templates/iva-techwriter-mcp/manifest.yaml` | 0.1.0 | write-канал `iva-atlassian-write` (Confluence create/update page + jira_add_comment) |

**План от 20.07 в этой части устарел.** Заметка
`20-Architecture/План- 4 профиля tacticum-dev … .md` помечает техписа «🔴 с нуля» и предлагает
`iva-tech-writer` по образцу `iva-fr-analyst` — с тех пор прошли ADR-0057/0058/0059, роль
техписа собрана по лейновой модели, а образец-профиль депрекирован. Остаётся верным только
пункт про гэпы: артефакт не выбран, корпуса эталонов нет.

**Но содержание doc-authoring — не то.** `templates/tacticum-documentation-base/ingredients/skills/doc-authoring/SKILL.md`,
**55 строк**, пост-dev документирование: API/контракт, README, changelog, ADR-заметка
(строки 21-33). Ни слова про ГОСТ, ЕСПД, шаблон документа, оформление. Прямо антипаттерном
записано «новый md-файл вместо правки существующего раздела» (строка 52) — то есть скилл
принципиально дописывает существующие доки, а не собирает документ с нуля по шаблону.
Это ровно противоположный ГОСТ-задаче режим.

## 3. Какие профили реально собраны и кто ближе по механике

59 профилей в `templates/` (без `_schema`), из них **16 роль-пресетов**:
`tacticum-role-internal / -platform / -techwriter`, `iva-role-analyst / -architect /
-connect-ios / -go / -ios / -ivcs / -java / -kmp / -one-mail-desktop / -qa / -qa-web / -web`,
`firebird-role-web`. Остальное — лейны (core/процесс/стек/способность/MCP) и депрекированные
монолиты переходного периода.

**Ближе всего к «собрать документ по шаблону из источников» — `iva-role-analyst`**
(лейн `iva-analysis-fr`, скилл `fr-authoring`), и с большим отрывом. Это единственный
механизм в репозитории, который: берёт сырьё из внешних источников (MCP), собирает документ
по жёсткому скелету, прогоняет механический валидатор и публикует. Роль техписа рядом не
стоит: у неё есть канал публикации, но нет авторинга документа по шаблону.

## 4. Скиллы авторинга — как задан шаблон и формат

Всего 333 файла SKILL.md, 151 уникальное имя скилла. Авторинг документов:

| Скилл | Путь | Строк | Как задан шаблон |
|---|---|---|---|
| `fr-authoring` | `templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md` (владелец — `iva-analysis-fr`) | **779** | ДВА полных скелета в fenced-блоках + валидатор + правила честности |
| `pin-authoring` | `templates/iva-rn-brownfield/ingredients/skills/pin-authoring/SKILL.md` | 207 | `## Document structure` (стр. 13) + API Verification Gate (стр. 109) |
| `tests-authoring` | `templates/tacticum-analysis-core/ingredients/skills/tests-authoring/SKILL.md` | 103 | `## Структура документа (tests.md)` (стр. 52) + обязательная BDD-форма (стр. 71) |
| `brd-authoring` | `templates/iva-rn-brownfield/ingredients/skills/brd-authoring/SKILL.md` | 85 | `## Document structure` (стр. 54) |
| `adr-authoring` | `templates/iva-kmp-brownfield/ingredients/skills/adr-authoring/SKILL.md` | 38 | `## Structure for each ADR` (стр. 13) |
| `doc-authoring` | `templates/tacticum-documentation-base/…/doc-authoring/SKILL.md` | 55 | шаблона нет — список «что документировать» |

**Как обеспечивается формат — механика fr-authoring, четыре слоя.** Это и есть готовый
ответ на «ГОСТ требует жёсткой структуры»:

1. **Скелет verbatim в fenced-блоке.** Не описание структуры, а буквальный каркас с
   заголовками, плейсхолдерами и инлайн-инструкциями в угловых скобках. Два жанра:
   «Скелет FR — ДВЕ ЧАСТИ» (`fr-authoring/SKILL.md:420-599`, 18 разделов Части 1 + П.C…П.J)
   и «Скелет ТЗ» (`:603-718`, 18 разделов). Выбор жанра — таблица `:409-412`,
   согласуется с человеком на гейте до сборки.
2. **Механический валидатор структуры перед публикацией.** `:178-205` — детерминированный
   чек-лист (i)…(iv) с явным pass/fail по каждому пункту; `:295-299` — гейт шага 9:
   хотя бы один *fail* = дефект документа, публикацию НЕ выполнять. Прямой аналог
   нормоконтроля, только машинный.
3. **Стабильные серии идентификаторов** (`:32-58`): FT-n/UC-n/Q-n/D-n/CT-n/DM-n/EV-n,
   номер выдан однажды и не переиспользуется; сквозная трассировка вниз по конвейеру.
4. **Правила честности** (`:720-779`, 12 пунктов): каждый факт с цитатой `[n]`, запрет
   выдуманных wire-имён, отрицательный результат — тоже факт, деградация всегда явная
   («молчаливый пропуск запрещён» — `:148`, `:289`).

**Few-shot образцы — почти нет.** Единственная зацепка: скелет ТЗ выверен по реальному
документу — «образец: „Техническое задание IVA Disk 1.5“, page 211846538»
(`fr-authoring/SKILL.md:412`, `:605`), но сам документ в репозиторий не вложен, только
ссылка на Confluence. Механизм для приложенных образцов существует —
`ingredients/skills/<id>/references/*.md` (напр.
`templates/iva-qa-autotest-base/ingredients/skills/write-autotest/references/feature-mapping.template.md`,
`.../batch-autotest/references/phases.md`), в схеме есть `metadata.assets`
(`templates/_schema/ingredient.v1.schema.json:81`). То есть класть эталоны ГОСТ-документов
в профиль есть куда — просто этого пока никто не делал.

## 5. Выходной формат — markdown + публикация в Confluence. DOCX нет

- **fr-authoring:** документ живёт в Confluence, не файлом. Шаг 9 (`:277-333`): адрес
  публикации из `.tacticum/context.yaml` → блок `fr_publish` (`space_key`, `parent_page_id`);
  запись тулами `confluence_create_page` / `confluence_update_page` (повторный прогон —
  update, не новая страница: история версий = аудит); вложения через
  `confluence_upload_attachment`; диаграммы drawio и якоря — сырыми
  `<ac:structured-macro>` Confluence Storage Format (`:307`, `:440`); компактный блок
  в Jira комментарием + label `ready-for-dev`/`needs-info`.
- **doc-authoring:** файлы в репозитории (README, changelog), markdown.
- **DOCX/ODT/pandoc/python-docx — в tacticum-dev нет нигде.** Поиск по всему репозиторию
  (`*.md`, `*.yaml`, `*.py`, `*.ts`, минус node_modules) дал 5 файлов, все — ложные
  срабатывания на quarkus-media-conversion и текст доков. Генерации бинарных офисных
  форматов в конвейере не существует.

## ВЕРДИКТ

**Решается профилем — авторинговая часть, но НЕ целиком: выходной формат придётся строить
отдельно, и это не «профиль vs сервис», а профиль ПЛЮС конвертер.**

За «профиль» — четыре факта. (1) Механика «источники → документ по жёсткому скелету →
механический валидатор → публикация» уже реализована и обкатана в `fr-authoring`
(779 строк, два жанра, валидатор с pass/fail-гейтом). ГОСТ отличается от ТЗ IVA Disk
содержанием скелета, а не природой задачи. (2) Роль техписа уже существует и собрана
правильно — `tacticum-role-techwriter` 0.3.0 с готовым write-каналом; добавление ГОСТ-лейна
в `depends_on` — правка одной строки манифеста. (3) Слой валидатора структурно ложится на
человеческую ступень нормоконтроля из постановки: машинный чек-лист сначала, нормоконтролёр
поверх. (4) Стоимость входа низкая: новый лейн — это каталог из manifest.yaml + SKILL.md +
references, без единой строки кода в бэкенде.

Против «сервиса» — сервис не даёт ничего, чего не даёт лейн, кроме одной вещи: **DOCX**.
ГОСТ-оформление (отступы, нумерация пунктов, титульный лист, поля) в markdown и в Confluence
Storage Format невыразимо, а генерации DOCX в tacticum-dev нет вообще. Это единственный
кусок задачи, который профилем не закрывается: нужен либо шаг конвертации (docx-шаблон +
рендер), либо решение владельца, что MVP отдаёт markdown, а оформление делает человек.
Отдельный вопрос — источники: MCP-каналы профиля смотрят в Confluence/Jira ИВА, а сырьё
смежного отдела лежит «в папках и на носителях», часть вики удалена. Чтение локальных папок
профилем возможно (Read/Glob), но фактуру придётся куда-то класть.

**Проверено:** `templates/` целиком (59 профилей, 16 ролей); манифесты `iva-fr-analyst`,
`tacticum-role-techwriter`, `tacticum-documentation-base`, `iva-techwriter-mcp`,
`iva-role-analyst`; скиллы `fr-authoring` (779 стр. полностью), `doc-authoring`,
`tests-authoring`, `brd-authoring`, `pin-authoring`, `adr-authoring`;
`_schema/ingredient.v1.schema.json`, `_mirrors.yaml`, `CONTRIBUTING.md`,
`docs/adr/0059`; голдены `apps/backend/tests/e2e_install/golden/`;
поиск по репо: `tech-writer|техпис|doc-authoring`, `ГОСТ|ЕСПД|конструкторск`,
`docx|pandoc|python-docx|odt`.

**Данные:** 59 профилей (16 роль-пресетов, остальное лейны и депрекированные монолиты);
333 файла SKILL.md, 151 уникальный скилл; 9 типов ингредиентов; 22 голден-снапшота
установки; образец `iva-fr-analyst` = 13 файлов, из них скилл-документщик 779 строк;
`doc-authoring` = 55 строк; 0 упоминаний ГОСТ/ЕСПД в tacticum-dev; 0 генераторов DOCX.

**Подтверждение:** `templates/iva-fr-analyst/manifest.yaml:10,53-63,152-214`;
`templates/tacticum-role-techwriter/manifest.yaml:27-30,104`;
`templates/tacticum-documentation-base/ingredients/skills/doc-authoring/SKILL.md:21-33,52`;
`templates/iva-fr-analyst/ingredients/skills/fr-authoring/SKILL.md:178-205,277-333,404-418,420-599,603-718,720-779`;
`templates/_schema/ingredient.v1.schema.json:10-16,81`; `templates/_mirrors.yaml:21-30`;
`docs/adr/0059-single-axis-process-lanes-and-role-packs.md`; `CONTRIBUTING.md:127,135`.

**НЕ проверено:**
- Бэкенд-рендерер (`apps/backend/…/renderer`) — как ингредиент физически раскладывается,
  и поддерживает ли он `metadata.assets` / доставку `references/*` наравне с SKILL.md.
  Смотрел только схему и голдены-хэши, код рендерера не читал.
- Реальный ГОСТ-корпус смежного отдела: какие именно 4–5 документов, какие ГОСТы (ЕСПД
  19.xxx vs конструкторские 2.xxx), есть ли у них docx-шаблоны с готовым оформлением.
  В обоих репозиториях этого нет.
- Существует ли где-то ещё в `~/tacticum` (вне tacticum-dev) генерация DOCX — искал
  только внутри tacticum-dev.
- Скилл `research` / лейн `tacticum-research-base` — может оказаться релевантен для сбора
  сырья из разрозненных источников, не смотрел.