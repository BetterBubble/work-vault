---
title: explore-doctranslator-gost-2026-08-06 — что из doc_translator переиспользуемо под генерацию ГОСТ-документов
type: report
status: draft
created: 2026-08-06 15:00
updated: 2026-08-06 15:00
permalink: tacticum/00-board/explore-doctranslator-gost-2026-08-06
repo: doc_translator
project: генерация ГОСТ-документации агентом
role: explorer
tags:
- board
- gost-docs
- explore
---

# Разведка `doc_translator` под задачу «агент собирает 4–5 ГОСТ-документов»

Все пути ниже — относительно `/Users/bubblemac/tacticum/doc_translator/`.

Serena подняла проект со второй попытки: первый `activate_project` привязался к чужому корню
(`/Users/bubblemac/tacticum/helm`) и `find_symbol` возвращал пустоту. После повторной активации
символьные операции работают — call-sites `reconstruct_docx` ниже получены через
`find_referencing_symbols`, не грепом.

## 1. Извлечение (`backend/app/domains/processing/extractors/`)

**Форматы.** 10 экстракторов + 3 конвертера OLE2→OOXML, зарегистрированы 9 адаптерами на
15 MIME-типов (`backend/app/domains/processing/format_registry.py:121-204`, константы MIME
`format_registry.py:12-26`).

| адаптер | форматы | библиотека под капотом |
|---|---|---|
| `word-openxml` | DOCX, DOC | `python-docx` (`extractors/docx_extractor.py:7`); DOC→DOCX через LibreOffice-subprocess (`extractors/doc_converter.py:12`) |
| `pdf` | PDF | PyMuPDF/`fitz` (`extractors/pdf_extractor.py:11`), OCR — PaddleOCR (`ocr_engine.py:19`) |
| `presentation-openxml` | PPTX, PPT | `python-pptx` (`extractors/pptx_extractor.py:29`) |
| `spreadsheet-openxml` | XLSX, XLS | `openpyxl` / `xlrd` (`extractors/xls_extractor.py:4`) |
| `rtf` | RTF ×2 mime | `striprtf` (`extractors/rtf_extractor.py:4`) |
| `odt` | ODT | голый `xml.etree` по zip (`extractors/odt_extractor.py:5`) |
| `epub` | EPUB | `ebooklib` + BeautifulSoup (`extractors/epub_extractor.py:5-7`) |
| `markdown` | MD ×2 mime | самописный парсер (`extractors/md_extractor.py`) |
| `subtitle` | SRT, VTT | самописный (`extractors/srt_extractor.py`) |

**Модель данных на выходе — и это главное ограничение.** Экстрактор отдаёт `list[TextSegment]`
(`backend/app/domains/processing/schema.py:48-67`), то есть **плоский список кусков текста, а не
дерево документа**. Поля `TextSegment`: `text`, `segment_type`, `paragraph_index`, `table_index`,
`row_index`, `col_index`, `bold`, `italic`, `font_size`, `alignment`, `color`, `bbox`, `raw_spans`,
`leading`. Плюс поверх строится IR — `DocumentIR` / `TextUnit`
(`backend/app/domains/processing/ir.py:63-87`), но это **IR переводческих юнитов** (`unit_id`,
`source_hash`, `translate_policy`, `protected_spans`), а не IR документа. Ни одного поля про
структуру документа в нём нет.

**Что из оформления сохраняется, а что нет** (`extractors/docx_extractor.py:51-107`):

- **Сохраняется:** bold, italic, underline, размер шрифта, имя шрифта, цвет run'а
  (`docx_extractor.py:62-88`, в `SpanData`), выравнивание абзаца (`docx_extractor.py:94`,
  словарь `_ALIGNMENT_MAP` — `docx_extractor.py:20-25`).
- **НЕ сохраняется:** отступы (`w:ind`), интервалы (`w:spacing`), поля страницы, размер/ориентация
  листа, **нумерация и уровни списков** (`w:numPr`/`w:numId`/`w:ilvl`), табуляции, границы таблиц,
  колонтитульная разметка. Проверено грепом по всему `backend/app/`:
  `grep -rn "numPr|numId|ilvl|numbering" --include="*.py" backend/app/` — **ноль совпадений**.
- **Имени стиля в модели нет вообще.** Стиль читается один раз в `docx_extractor.py:51` и тут же
  схлопывается в булеву классификацию по жёстко зашитому английскому набору
  `_HEADING_STYLES = {"Heading 1","Heading 2","Heading 3","Title","Subtitle"}`
  (`docx_extractor.py:27`, применение — `docx_extractor.py:52-54`). Русские стили («Заголовок 1»)
  сюда не попадут, а сам `style_name` в `TextSegment` не кладётся.
- **Иерархия заголовков не строится.** `SegmentType.HEADING` — плоская метка без уровня
  (`schema.py:8`). Уровня раздела, дерева разделов, сквозной нумерации пунктов в модели нет.

`SegmentType` покрывает 20 видов кусков (`schema.py:6-25`): paragraph, heading, table_cell, caption,
header, footer, footnote, endnote, comment, text_box, hyperlink, 6 слайдовых, 2 табличных.

## 2. Реконструкция (`backend/app/domains/processing/reconstructors/`)

**Ответ однозначный: только «взять исходный файл и заменить текст». Собрать документ с нуля или по
шаблону нельзя — такого кода в репозитории нет.**

Это видно из сигнатуры, а не из чтения между строк. Единственная точка входа принимает исходные
байты обязательным первым аргументом:

```python
def reconstruct_docx(
    original_bytes: bytes,
    original_segments: list[TextSegment],
    translated_texts: list[str],
) -> bytes:
```

`reconstructors/docx_reconstructor.py:27-31`. Докстрока прямо описывает стратегию: «Load the
original DOCX → build mapping paragraph_index → translated text → walk all paragraphs and tables,
replace text» (`docx_reconstructor.py:32-38`). Тот же контракт протянут через весь слой:
`FormatAdapter.reconstruct` типизирован как `Callable[[bytes, list[TextSegment], list[str]], bytes]`
(`format_registry.py:29`), диспетчер — `reconstructors/dispatcher.py:22-34`.

Подтверждения, что создания документа нет:

- `Document(...)` (python-docx) во всём `backend/app/` вызывается **только** от готовых байтов:
  `extractors/docx_extractor.py:36` и `reconstructors/docx_reconstructor.py:39` — оба
  `Document(io.BytesIO(...))`. Вызова `Document()` без аргументов (создание пустого документа) нет.
- API создания структуры (`add_paragraph`, `add_heading`, `add_table`, `add_section`) **не
  используется нигде**. Есть только три вызова `add_run` — и все внутри уже существующего абзаца,
  при замене его текста: `docx_reconstructor.py:349`, `:368`, `:372`.
- Call-sites `reconstruct_docx` (через `find_referencing_symbols`): один продовый —
  `format_registry.py:354,356`; остальные 13 — тесты (`backend/tests/unit/test_roundtrip.py:73,97,120`,
  `test_docx_reconstructor.py:71,446,566`, `test_docx_formatting_regression.py:62,207,221,256`,
  `test_large_file_scale.py:87,101`, `test_metadata_preservation.py:38`,
  `test_sprint1_gaps.py:710`). Все — round-trip, ни одного «сборка с нуля».

Что переиспользуемо из этого файла: механика посегментной подмены текста с сохранением форматирования
run'ов — `_replace_paragraph_text` (`docx_reconstructor.py:325-389`), `_apply_span_formatting`
(`:304-322`), пропорциональное разрезание переведённого текста по исходным run'ам
(`_split_text_proportionally`, вызов на `:366`), сохранение полей PAGE/DATE/NUMPAGES
(`:357-361`, `_is_field_related_run`). Это ровно то, что нужно для сценария **«взять
ГОСТ-шаблон .docx и залить в него содержимое»** — но не для «сгенерировать документ».

Побочный факт: `md_reconstructor.py` и `srt_reconstructor.py` импортируют собственные экстракторы
(`md_reconstructor.py:5`, `srt_reconstructor.py:3`) — то есть даже текстовые форматы пересобираются
из исходника, а не пишутся с нуля.

## 3. `docx_fidelity_report.py` — годится ли как машинный нормоконтроль

**Не годится. Он проверяет не оформление, а переводимость.**

Это фиче-детектор: разбирает XML-части пакета (`word/*.xml`, фильтр `_should_scan_part` —
`docx_fidelity_report.py:198-199`), считает вхождения элементов по локальному имени тега
(`_build_feature_finding:172-195`) и сверяет со списком из **7 захардкоженных фич**
(`_FEATURE_SPECS`, `docx_fidelity_report.py:91-145`):

| фича | искомые теги | статус |
|---|---|---|
| `docx_comments` | `comment` | translated / info |
| `docx_hyperlinks` | `hyperlink` | translated / info |
| `docx_bookmarks` | `bookmarkStart` | preserved / info |
| `docx_text_boxes` | `txbxContent` | translated / info |
| `docx_content_controls` | `sdt` | **unsupported / blocker** |
| `docx_tracked_changes` | `ins`, `del`, `moveFrom`, `moveTo` | **unsupported / blocker** |
| `docx_fields` | `fldSimple`, `instrText` | preserved / info |

Метрики на выходе (`DOCXFidelityReport`, `:38-78`): `unsupported_count`, `blocking_count`, список
`findings` (feature, status, severity, count, parts, message), `parse_failures`, булев `ok`
(`:67-69` — `ok` ⟺ нет unsupported и нет ошибок парсинга).

Ни шрифта, ни кегля, ни полей, ни отступов, ни нумерации, ни состава документа он не смотрит. Смысл
модуля — fail-closed перед переводом: «в файле есть tracked changes → мы не гарантируем верность,
блокируем». Вызывается из экстрактора (`extractors/docx_extractor.py:10`).

**Что реально годится в заготовку машинного нормоконтроля** — не сам отчёт, а его каркас и соседние
модули:
- **Форма отчёта** — `_FeatureSpec` (`:81-88`) → finding с severity info/warning/blocker
  (`:16-19`) → агрегат с `ok` и счётчиками. Под правила ГОСТ («поле слева 30 мм», «шрифт 14 pt»,
  «нумерация пунктов сквозная») эта форма ложится один-в-один: меняется набор спеков, каркас
  остаётся. 211 строк, чистый standalone-код без зависимостей кроме `read_ooxml_parts`.
- **Доступ к сырым частям OOXML** — `read_ooxml_parts(package_bytes) -> dict[str, bytes]`
  (`ooxml_structural_diff.py:108`). Через него достаётся `word/styles.xml`, `word/numbering.xml`,
  `w:sectPr` — то, где живут реальные ГОСТ-атрибуты.
- **Рендер в PDF для визуальной проверки** — `render_ooxml_to_pdf_with_libreoffice`
  (`ooxml_libreoffice_renderer.py:22`) + попиксельное сравнение страниц
  (`ooxml_visual_diff.py:28`, `_pixel_similarity:164`).
- Оговорка: `diff_ooxml_structure` (`ooxml_structural_diff.py:32`, 395 строк) — это сравнение
  **двух пакетов между собой** (исходник vs перевод), а не проверка одного пакета против правила.
  Для нормоконтроля напрямую не подходит; полезен как «эталон vs результат», если эталонный документ
  есть.

Аналогичные отчёты есть для PDF (`pdf_fidelity_report.py`, 365 стр.), XLSX (344), PPTX (273).

## 4. Пригодность как библиотеки — можно ли выдернуть без стека

**Можно. Слой `processing` инфраструктурой практически не заражён.**

Проверено грепом по всему домену: `grep -rn "sqlalchemy|minio|celery|redis|session|Depends"
backend/app/domains/processing/` → **единственное совпадение — строка регулярки в
`pdf_reconstructor.py:76`** (`\b(systemctl|journalctl|nginx|postgres|redis|docker|...)\b` — паттерн
для распознавания технических терминов в тексте, к инфраструктуре отношения не имеет). Ни ORM, ни
объектного хранилища, ни брокера, ни FastAPI-зависимостей внутри домена нет.

Внешние зависимости слоя (полный список `app.*`-импортов):
- `app.domains.processing.*` — внутри себя, замкнуто;
- `app.core.settings` — **только в `pdf_extractor.py`**, 6 обращений и все — фича-флаги OCR
  и классификатора PDF: `PDF_CLASSIFIER_FORCE_MODE` (`pdf_extractor.py:476`),
  `PDF_CLASSIFIER_ENABLED` (`:480`), `PDF_PADDLE_OCR_FALLBACK_ENABLED` (`:2494`),
  `OCR_MAX_RETRIES` (`:2513`), `:3754-3755`;
- `structlog` — сквозной логгер.

Границу с инфраструктурой держит `translation`-домен: реконструкция вызывается через Protocol
`ReconstructDocument` (`backend/app/domains/translation/pipeline.py:130`, инъекция —
`:207,220`), а celery-таск лишь подставляет реализацию
(`backend/app/tasks/translation_tasks.py:191-192,1617-1633`). То есть развязка уже сделана.

**Практический вывод:** под DOCX можно взять `schema.py` (66 стр.) + `docx_extractor.py` (488) +
`docx_reconstructor.py` (661) + `docx_fidelity_report.py` (211) + `docx_hyperlink_text.py` (235) +
`ooxml_namespace.py` (145) + `ooxml_structural_diff.py` ради `read_ooxml_parts` (395) и получить
работающую библиотеку на одном `python-docx` + `lxml`. Ничего от Postgres/Redis/MinIO/Meilisearch/
LibreTranslate/Celery не тянется. Единственный внешний бинарь на DOCX-пути — LibreOffice, и только
для конвертации legacy `.doc` (`extractors/doc_converter.py:12`, subprocess) и для рендера в PDF.
PaddleOCR/PyMuPDF нужны только PDF-ветке — её можно не брать.

Косвенное подтверждение развязки: 23 из 60 unit-тестов (`backend/tests/unit/`) гоняют
extraction/reconstruction напрямую, импортируя функции без всякого приложения — см. call-sites в §2.

## 5. ГОСТ, шаблоны, нумерация разделов

**Ничего нет. Ни строки.**

- `grep -rin "гост|нормоконтр|ЕСКД|ЕСПД" --include="*.py" --include="*.md" --include="*.ts"
  --include="*.tsx" --include="*.yml" .` (без node_modules/`__pycache__`) → **одно совпадение, и оно
  ложное**: `reconstructors/pdf_reconstructor.py:348` — регулярка `\bГостевой\s+доступ\s+\(259...`,
  то есть слово «Гостевой» из доменного текста заказчика.
- Шаблонов документов нет. Слово `template` в `backend/app/` встречается в двух файлах —
  `domains/translation/capabilities.py` и `reconstructors/pptx_reconstructor.py` (слайдовые
  layout'ы PowerPoint), к шаблонам документов отношения не имеет.
- Нумерации разделов/пунктов нет — см. §1, нулевой греп по `numPr|numId|ilvl|numbering`.
- В `docs/specs/` (34 файла) — спринт-отчёты по fidelity перевода, PDF-качеству и UX. Про
  генерацию документов и ГОСТ — ничего.

Репозиторий решает ровно одну задачу, и README её называет прямо: «document translation platform
for Russian source documents with layout-preserving reconstruction» (`README.md:3-5`).

---

## ВЕРДИКТ

**ВЕРДИКТ:** переиспользуемо как есть — (1) чтение DOCX и OOXML-пакета: `python-docx`-обвязка
`extractors/docx_extractor.py` и `read_ooxml_parts` (`ooxml_structural_diff.py:108`), плюс 9
адаптеров на 15 форматов, если материалы отдела разношёрстные; (2) механика **посегментной заливки
текста в готовый .docx с сохранением форматирования run'ов** — `_replace_paragraph_text` +
`_apply_span_formatting` (`docx_reconstructor.py:304-389`), это ключ к сценарию «ГОСТ-шаблон +
наполнение»; (3) каркас fidelity-отчёта (спека → finding с severity → агрегат `ok`,
`docx_fidelity_report.py:81-145`) как форма будущего машинного нормоконтроля; (4) рендер в PDF и
попиксельное сравнение (`ooxml_libreoffice_renderer.py:22`, `ooxml_visual_diff.py:28`); (5) весь
слой выдирается в библиотеку без инфраструктуры.

Писать придётся — **всё, что отличает генерацию от перевода**, а это ядро задачи: (а) модель
документа с иерархией разделов, уровнями заголовков и нумерацией пунктов — сейчас модель плоская
(`schema.py:48-67`), `SegmentType.HEADING` без уровня, стиль абзаца в модель вообще не кладётся;
(б) сборка DOCX с нуля/по шаблону — единственная точка входа требует `original_bytes` первым
аргументом (`docx_reconstructor.py:27-31`), `add_paragraph`/`add_heading`/`add_table` не
используются нигде; (в) захват и запись атрибутов ГОСТ-оформления — отступы, интервалы, поля
страницы, `w:numPr`/`numId`/`ilvl` не читаются и не пишутся (нулевой греп по всему `backend/app/`);
(г) сами правила ГОСТ и состав из 4–5 документов — в репозитории отсутствуют полностью.

Формулировка «переиспользуем doc_translator» корректна для **заливки в шаблон**, но не для
**генерации**: разворот из «плоский список сегментов + исходный файл» в «дерево документа +
шаблон» — это новая модель данных, а не доработка существующей.

**Проверено:** `backend/app/domains/processing/` целиком — `schema.py` (66), `ir.py` (392),
`format_registry.py` (550), `service.py` (113), `docx_fidelity_report.py` (211), `ocr_engine.py`;
`extractors/` (13 файлов, 5519 строк) — полностью прочитан `docx_extractor.py:1-120`, остальные по
импортам и сигнатурам; `reconstructors/` (12 файлов, 10694 строк) — прочитаны
`docx_reconstructor.py:1-160` и `:300-389`, `dispatcher.py`, `__init__.py`; публичные API
`ooxml_structural_diff.py`, `ooxml_visual_diff.py`, `ooxml_render_diff.py`,
`ooxml_libreoffice_renderer.py`; `backend/pyproject.toml`; `README.md`; листинг `docs/specs/`;
call-sites `reconstruct_docx` через Serena `find_referencing_symbols`.

**Данные:** 10 экстракторов + 3 OLE2-конвертера · 10 реконструкторов · 9 format-адаптеров ·
15 MIME-типов · слой `processing` = 5045 строк верхнего уровня + 5519 extractors + 10694
reconstructors ≈ 21 258 строк · `TextSegment` — 17 полей, из них про оформление 7 (bold, italic,
font_size, alignment, color, bbox, leading) · `SegmentType` — 20 значений · `docx_fidelity_report` —
7 проверяемых фич, 211 строк · DOCX-подмножество для выноса в библиотеку ≈ 2201 строка ·
инфраструктурных импортов в домене — 0 · импортов `app.core.settings` — 6, все в `pdf_extractor.py`
· совпадений по «ГОСТ/нормоконтроль» — 1, и оно ложное · совпадений по `numPr|numId|ilvl|numbering`
— 0 · 23 из 60 unit-тестов работают с extraction/reconstruction напрямую.

**Подтверждение:**
- `grep -rn "numPr\|numId\|ilvl\|numbering" --include="*.py" backend/app/` → пусто
- `grep -rin "гост\|нормоконтр\|ЕСКД\|ЕСПД"` → только `pdf_reconstructor.py:348` («Гостевой доступ»)
- `grep -rn "sqlalchemy\|minio\|celery\|redis\|session\|Depends" backend/app/domains/processing/` →
  только `pdf_reconstructor.py:76` (регулярка)
- `grep -rn "add_paragraph\|add_heading\|add_table\|new_document" backend/app/` → пусто;
  `add_run` → только `docx_reconstructor.py:349,368,372`
- `grep -rn "Document(" backend/app/` → все вызовы от байтов
- Serena `find_referencing_symbols reconstruct_docx` → 1 продовый call-site + 13 тестовых
- Ключевые файлы: `backend/app/domains/processing/schema.py:48-67`,
  `backend/app/domains/processing/reconstructors/docx_reconstructor.py:27-38`,
  `backend/app/domains/processing/extractors/docx_extractor.py:27,51-54`,
  `backend/app/domains/processing/docx_fidelity_report.py:91-145`,
  `backend/app/domains/processing/format_registry.py:29,121-204`

**НЕ проверено:**
- `pdf_extractor.py` (3831 стр.) и `pdf_reconstructor.py` (8370 стр.) — прочитаны только импорты,
  обращения к `settings` и публичные сигнатуры. Это 75% кода слоя, но PDF-ветка к генерации
  ГОСТ-DOCX отношения не имеет, а полное чтение съело бы разведку целиком.
- Ничего не запускалось: ни docker, ни тесты, ни импорт модулей — по прямому ограничению задачи.
  Значит вывод «слой выдирается как библиотека» получен **статически, по анализу импортов**;
  фактическую сборку изолированного пакета никто не пробовал, скрытые зависимости через рантайм
  (например, конфиг `structlog` или ленивые импорты) могут вылезти при попытке.
- `frontend/` не смотрел — вне вопроса.
- Как ведёт себя `_HEADING_STYLES` на русских стилях («Заголовок 1») — вывод сделан по чтению
  константы `docx_extractor.py:27` и сравнения `style_name in _HEADING_STYLES`
  (`docx_extractor.py:52-54`), эмпирически на русском документе не воспроизводил.
- Экстракторы ODT/RTF/EPUB/MD — только импорты и размер, глубину сохранения оформления в них не
  разбирал: под ГОСТ-DOCX не профильны.
- Есть второй, упрощённый путь чтения DOCX — `backend/app/domains/documents/text_extractor.py:75,83`
  (для поиска/индексации). В зону вопроса не входил, детально не смотрел.
