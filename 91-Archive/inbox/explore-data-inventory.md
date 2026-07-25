---
title: explore-data-inventory
type: report
permalink: tacticum/91-archive/inbox/explore-data-inventory
tags:
- helm
- data
- inventory
- explorer
status: archived
updated: 2026-07-18
---

# explore-data-inventory — разведка дропов `~/tacticum/helm/data/`

> Черновик воркера-разведчика (read-only). Снимок источников 2026-07-03. Канон — `control-tower-v02`.
> M-теги = US Taiga #32 (M0 подготовка входов · M1 ingestion · M2 сшивка Т1-Т3 · M3 RE · M4 граф · M5 скоринг · M6 разрывы · M7 Монитор ГД · M8 Gateway · M9 governance · M10 веб-апп).

## Кодировка/формат
Все CSV — UTF-8 **с BOM** (`utf-8-sig`), loader `csv_io` BOM проглатывает. `wc -l` включает строку-заголовок (строк данных = lines−1).

## A. Инвентарь (по дропам)

### D0. loose `data/` (корень) — ДУБЛИ/доки
| Файл | Строк | Real/Синт | 🔴 | Модуль | Волна | CQ | Вердикт |
|---|---|---|---|---|---|---|---|
| jira_issues.csv | 3072 | real | | M1 | 1a | A1/A4/A5 | **ДУБЛЬ** real/jira (md5 идентичен) → _superseded |
| jira_epics.csv | 101 | real | | M1 | 1a | A4 | **ДУБЛЬ** → _superseded |
| jira_manifest.csv | 15 | real | | M1 | 1a | — | **ДУБЛЬ** → _superseded |
| README.md, competency-questions.md | — | док | | — | — | — | оставить в корне |

### D1. `repos-1/` — КАНОН git+код (552M)
| Файл/дир | Объём | Модуль | Волна | CQ |
|---|---|---|---|---|
| commits.csv | 24040 (24039 коммитов) | M1·M2(Т1) | 1a | B4/B8/B5 |
| merges.csv | 345 (344; 133 несут !MR) | M1(PR-прокси) | 1b | — |
| repos_manifest.csv | 9 (8 репо) | M3 | 1a | D2 |
| repos_registry.csv | 10 (9 репо: commits/authors/языки/top_committers) | M3·§6 | 1a | B8/D2 |
| repomix/*.xml | 9 файлов, 187M (код-снимки, --compress) | RAG §8.3B | 1b | D3 |
| scip/ | 8/9 репо, ~365M (SCIP protobuf); **Kotlin(kmp)+Java(ivcs-server) НЕ покрыты** | M3/RE code_index | 1b | D2/D3 |
| topology/ | 8 репо (compact-index.json + review.md); **trust_tier auto-low, needs-review** | §3 work→product (черновик) | 1b | — |

### D2. `Обезличенные данные/` — СИНТЕТИКА (@iva.example), безопасный прогон 1a (64K)
Все выдуманные (люди/клиенты/суммы/зарплаты/CV). Продукты/поколения — реальная структура. Проходит M0-валидаторы + gate S7 (0 gaps).
| Файл | Строк | Модуль | Узел графа |
|---|---|---|---|
| reference_products.csv / reference_generations.csv | 6/6 | M0/S8 | Продукт/Поколение |
| goals.csv | 6 | M0·генезис | Цель |
| blocks.csv | 7 | M0·M4·M7 | Блок→ЕОЛ |
| teams.csv | 11 | M0·M4 | Человек→Команда |
| sales_initiatives.csv | 8 | M0·генезис | Клиент·Сделка |
| jira_issues.csv | 16 | M1 | Задача·эпик |
| git_commits.csv / git_prs.csv | 9/6 | M1·M2 | Репо·PR |
| crm_deals.csv | 6 | M1/M5 | Сделка |
| product_economics.csv | 6 | M5 | Продукт→маржа |
| sensitive/staff.csv / salary.csv / cv.csv | 11/11/11 | M5 🔴(образец) | Человек |

### D3. `data (2)/` — РЕАЛЬНЫЙ дроп ИВА 🔒 (16M, не обезличено)
**task management/jira/**
| Файл | Строк | Модуль | Волна | CQ |
|---|---|---|---|---|
| jira_issues.csv | 3072 | M1 | 1a | A1/A4/A5/B4 |
| jira_epics.csv | 101 | M1·генезис | 1a | A4 |
| jira_manifest.csv | 15 (14 проектов) | M1 | 1a | — |
| jira_issue_links.csv | 5513 (Blocks-рёбра) | M4 зависимости | 1a/1b | A3/D5/C3 |
| active_blockers.csv | 221 (живые open→open) | M4 | 1a | A3 |
| ivaone/epics_ivaone.csv | 68 | M7 продукт-монитор | 1b | A4 |
| ivaone/tasks_open_ivaone.csv | 1410 | M7 | 1b | B5 |
| ivaone/monitor_ivaone.html + brief.md | — | T0-артефакт | 1b | — |

**task management/statuses/** (Монитор ГД, ФИО реальны)
| ceo_directives.csv | 16 (15 поручений; owner_email верифиц. по identity) | M0·M7 | 1a | A1/A2 |
| monitor_gd_real.html + brief.md | — | T0-артефакт | 1a | — |
| 2026-07-02/*.docx | 7 файлов (Поручения/Итоги/Стенограммы встреч RND 26.06+02.07) | M0 сырьё | 1a | — |

**sales/** (⚠ коммерческое)
| crm_deals.csv | 2179 (2178 сделок; live-БД d10task_crm) | M5 важность | 1a | C1/C2/C3 |
| top_projects.csv | 65 | M5 | 1b | C1 |
| top_projects_pipeline.csv | 253 | M5 | 1b | C1 |

**service/** (ServiceDesk = сигналы)
| sd_tasks.csv | 42278 (42277 обращений; d10task_sd) | M1 сигналы | 1a | C4/C5 |
| sd_summary.csv | 53 | M1 | 1a | C4 |

**identity/** (⚠ PII, риск №1)
| identity_map.csv | 956 (955 идентичн., 219 мульти-системных) | сшивка M2/M4 | 1b | D1/D6/B4 |
| identity_gaps.csv | 148 (25 team без email + 122 git-кластера) | сшивка | 1b | D-gaps |

**data_room/economics/** (⚠ финансы)
| product_economics_37m.csv | 35 (маржа+решение Инвест/Сократить) | M5 деньги | 1b | B3 |
| optimistic_scenario.csv | 16 | M5 | 1b | B3 |
| portfolio_decisions.csv | 71 (pivot merged-cells, «ступеньки») | M5 | 1b | B3 |

**data_room/product_mgmt/** | product_owners.csv | 14 (продукт→владелец, реальные ЕОЛ-кандидаты) | M0·M7 blocks/ЕОЛ | 1a | E-роли |

**data_room/support/** (сигналы инцидент/FR)
| crm_open_requests.csv | 3305 (сырьё open) | M1 | 1a/1b | C4/C5 |
| fr_backlog.csv | 51 | M1 | 1b | C5 |
| backlog_snapshot.csv | 27 | M1 | 1b | C4 |
| executive_summary.csv | 23 | M1 | 1b | — |
| critical_cases.csv | 15 | M1 | 1b | C5 |
> support/* — pivot-таблицы с merged-ячейками, требуют доочистки.

**data_room/team/** (⚠ PII, реальные ФИО; ФОТ по ФИО НЕ переносили)
| target_team.csv | 118 (117 чел: трек/продукт/левел/статус Target-Gold) | M0·M4 teams | 1b | B-team |
| track_owners.csv | 6 (5 контуров-ЕОЛ-кандидатов) | M0 blocks | 1b | — |

**data_room/sensitive/** 🔴 (узкий контур, real)
| by_person.csv | 293 (292 чел: person_email·ФИО·ставка_fte·статус·трек·продукт·роль·подразделение) | M5 стоимость | 1b | B1/B2 |
| fot_by_product.csv | 317 (ФОТ по продуктам; pivot merged) | M5 | 1b | B1 |
| team_detail.csv | 350 (план/факт JIRA-часы; pivot merged) | M5 | 1b | — |
> ⚠ Расхождение: README называет файл `salary_by_person.csv` с гросс-доходом (кол.P), фактический файл `by_person.csv` в заголовке салари-колонки не показывает (person_email,ФИО,ставка_fte,статус_отбора,gold,трек,продукт,роль,левел,подразделение,компания). B1/B2 (ФОТ по ФИО) зависят от наличия рублёвого столбца — надо проверить содержимое.

**wiki/confluence/** (индекс, без тел)
| pages_index.csv | 3155 (3154 стр, 35 пространств) | M1/RAG контекст | 1b | D4 |
| spaces.csv | 118 (117 пространств) | M1 оргкарта | 1b | D4 |

**data (2)/repos/** — commits.csv(24040)/merges.csv(345)/repos_manifest.csv(9) — **байт-в-байт == repos-1 CSV** (md5 идентичны), но БЕЗ repomix/scip/topology/registry → **ДУБЛЬ (обеднённый)** → _superseded.

**Пустые placeholder-каталоги:** `data (2)/task management/eva.iva.ru/` (доступ eva не получен), `data (2)/wiki/wiki.iva.ru/` — пустые.

### D4. loose `repos/` — РАННИЙ мелкий снимок (92K)
commits.csv 436 (≤60/репо), merges.csv 146 — все 435 хешей ∈ repos-1 (0 missing) → **ПОДМНОЖЕСТВО** repos-1 → _superseded.

### D5. loose `confluence/` — ТЕЛА 3 страниц Confluence (208K)
131664812(IVAADP), 205489583+205489587(IVAPROJECT) — .json (REST body) + .md (рендер). Все 3 page_id присутствуют в индексе `data (2)/wiki/confluence/pages_index.csv`. **Комплементарно** индексу (тела vs индекс), НЕ дубль. M1/RAG, 1b, D4.

## B. Дубликаты → вердикт
1. loose `jira_*.csv` == `data (2)/task management/jira/{jira_issues,jira_epics,jira_manifest}.csv` — **md5 идентичны** → канон = real/jira, loose → `_superseded/jira-loose/`.
2. loose `repos/` (436) ⊂ `repos-1/` (24039) — **0 хешей отсутствует** → канон = repos-1, loose → `_superseded/repos-shallow/`.
3. `data (2)/repos/` CSV == `repos-1/` CSV — **md5 идентичны**, но без обогащения (repomix/scip/topology/registry) → канон = repos-1, → `_superseded/repos-csv-only/`.
4. Канон git-источника = **repos-1** (единственный с repomix+scip+topology+registry).
5. `confluence/` (тела) ≠ дубль индекса — оставить как `real/confluence/bodies/`.

## C. Целевая раскладка (по источникам; real ⟂ синтетика; 🔴 изолирован)
```
data/
  README.md · competency-questions.md          # доки (обновить README отдельно)
  example/                                       # 🟢 СИНТЕТИКА @iva.example — безопасный прогон 1a
    (всё из «Обезличенные данные/», включая sensitive/ — образец)
  real/                                          # 🔒 РЕАЛЬНЫЕ ИВА (gitignore)
    jira/            ← data(2)/task management/jira/ (+ ivaone/)
    monitor-gd/      ← data(2)/task management/statuses/ (ceo_directives, html/brief, 2026-07-02 docx)
    git/             ← repos-1/ (commits/merges/manifest/registry + repomix/ scip/ topology/)
    crm/             ← data(2)/sales/
    service/         ← data(2)/service/
    identity/        ← data(2)/identity/                 # ⚠ PII
    confluence/      ← data(2)/wiki/confluence/ (индекс)
      bodies/        ← loose confluence/ (3 тела)
    data-room/       ← data(2)/data_room/{economics,product_mgmt,support,team}
    sensitive/       🔴 ← data(2)/data_room/sensitive/   # узкий контур CEO/CPO/HR
  _superseded/                                   # ничего не теряем
    jira-loose/      ← loose jira_*.csv
    repos-shallow/   ← loose repos/
    repos-csv-only/  ← data(2)/repos/
```

## E. Пробелы данных
1. **Тела Confluence** — есть 3 из 3154 (D4/D5); D4 «где документация» почти не отвечается. Нужно `/rest/api/content/{id}?expand=body.storage` по нужным пространствам.
2. **GitLab PAT** — MR API (title/author/status/approvals) отсутствует; MR = merge-коммит-прокси (133/344 с !NNN). Нужен PAT на `iva/* ivcs/* imail/* rn/* terra/*`.
3. **Полная история Jira** — ≤300 свежих/проект (VCSMOB 14345 всего). Нужна пагинация startAt/maxResults.
4. **jira_login пуст** в identity_map (Jira group-API 403, PAT без права листать юзеров). Логины — по админ-доступу.
5. **SCIP Kotlin+Java** — kmp(Kotlin) и ivcs-server(Java-сервер), 2 крупнейших репо, символьно не покрыты. Нужна build-среда с приватными Maven/Gradle registry ИВА (Волна 2).
6. **Карта идентичности хвост** — 25 team без email + 122 git-кластера не сшиты (ядро остаточного риска №1); email не сверен с IdP.
7. **eva.iva.ru** — доступ не получен (пустой placeholder).
8. **topology** — черновик auto-low, needs-review; не источник истины без ручной валидации.
9. **Реальные операторские входы 1a** — реальные goals/blocks/ЕОЛ ещё не курированы (открытый Q0 канона); есть кандидаты product_owners/track_owners, но не финальная нарезка. Синтетика есть только в example/.
10. **ФОТ по ФИО (B1/B2)** — проверить, есть ли рублёвый столбец в sensitive/by_person.csv (README обещает гросс-доход, заголовок не показывает).
