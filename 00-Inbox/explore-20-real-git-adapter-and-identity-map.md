---
title: explore-20-real-git-adapter-and-identity-map
type: report
permalink: tacticum/00-inbox/explore-20-real-git-adapter-and-identity-map
tags:
- helm
- explore
- 1b
- git
- identity
---

# Разведка #20 — RealGitAdapter + identity-map (follow-on к #19)

Черновик explorer. Каноническую запись делает лид.

## A. RealGitAdapter — маппинг схем на контракт RawGitCommit (заводит #19)

Синтетика `git_commits.csv`: `hash,repo,author_email,date,message,branch` — jira-ключ ВНУТРИ message («MAIL-201 add gost cipher»).
Реал `data/repos/commits.csv`: `repo,hash,author_email,author_name,date,branch,jira_keys,subject` — jira_keys отдельной колонкой (IVAONE-12113; может быть несколько), + author_name.
Реал `data/repos/merges.csv`: `repo,hash,author_email,date,mr_ref(!NNN),source_branch,target_branch,jira_keys,subject`.
`repos_manifest.csv`: repo→product/generation (M3).

Маппинг на RawGitCommit (общий контракт для обоих источников):
| поле RawGitCommit | синтетика | реал |
|---|---|---|
| hash | hash | hash |
| repo | repo | repo |
| author_email | author_email | author_email |
| author_name | — (None) | author_name |
| date | date | date |
| branch | branch | branch |
| jira_keys (tuple) | **регулярка** из message (`[A-Z][A-Z0-9]+-\d+`, все совпадения) | **split колонки** jira_keys (по ; / пробелу), пусто→() |
| subject | message | subject |

Итог: две реализации одного протокола (как IssueSource). Синтетика извлекает ключ регуляркой из message; реал берёт готовую колонку + fallback-регулярка по subject/branch (в branch есть `feature/IVAONE-12066-...`). csv_io уже глотает BOM (utf-8-sig).

merges → PR-прокси: отдельный `RawGitMerge` (или commits с флагом is_merge). Несёт mr_ref (!NNN, есть у 46/145), source_branch→target_branch (ребро зависимости/интеграции), jira_keys. Полноценный PR (title/reviewers/status/approvals) — НЕ здесь (блокер PAT). Т.е. merges = прокси PR/MR: mr_ref как external_id PR, статус «merged» подразумевается (это merge-коммит).

## B. Identity-map слой

Опора уже есть: `teams.csv` несёт на человека колонки `repos` и `jira_projects`; модель `Person` (models.py:47-66) имеет JSON-поля `repos`/`jira_projects` + 1:N `PersonEmail` (email PK, is_primary). Т.е. person↔email 1:N уже заложен без миграции.

Что показал реальный `commits.csv` (author_email, ~40 уникальных):
- **Мульти-идентичность одного человека**: `a.rodionov@iva.ru` ↔ `a.rodionov@iva-tech.ru`; `sergey.sukhov@iva.ru` ↔ `@ivcs.su`; `v.bolshunov@iva.ru` ↔ `v.bolshunov@MacBook-Pro-Bolsunov.local`. → это ровно кейс person 1:N email.
- **Машинно-локальные/мусорные**: `v.bolshunov@MacBook-Pro-Bolsunov.local`, `ev@EVMed.local` — git user.email по умолчанию хоста. Нужна нормализация/фильтр.
- **Боты/CI**: `jenkins@ivcs.su` — не человек, исключать из Person-загрузки.
- **Внешние подрядчики**: `@gmail.com` (andrey.o.vorobiev, golempanda), `@mail.ru` (maksimrm1342), `@iva-tech.ru`, `@nwire.ru` — не в teams.
- **Свои (Tacticum) в контуре ИВА**: `taktikum_a.berezin@iva.ru`, `taktikum_d.parshakov@iva.ru`.

Дизайн (1b):
1. **Резолвер** `resolve_person(author_email) -> person_id | None` через PersonEmail-lookup (точное совпадение). Сшивка seed: teams.csv → Person + PersonEmail (primary) + Person.repos/jira_projects.
2. **Алиасы** (несколько email на человека): НЕ авто-мержить разные домены (iva.ru vs gmail небезопасно). Эвристика-кандидат по совпадению local-part + оператор подтверждает → доп. строки PersonEmail. Задел под `identity_alias`/аннотацию оператора.
3. **Бакет «внешний/неизвестный»**: неразрезолвленный author → помечается external/unknown (не крашить). Отдельный отчёт «unresolved git authors» (аналог разрыва) — оператору на разметку.
4. **Фильтр не-людей**: боты (jenkins@, *@ci*), machine-local (`*.local`, содержит имя хоста) — исключать/флагать до Person-загрузки.
5. **Нормализация**: lower-case, trim; опц. отбрасывать `+suffix`.

Блокеры:
1. **Т1-сшивка**: реальные Jira-проекты git (IVAONE/VCSWEB/P8) ≠ курированные (ONE/MAIL...). Без реального Jira-экспорта IVA git jira_keys повиснут → массово «работа без цели». (Уже в списке запросов данных.)
2. **Алиасинг требует человека**: авто-слияние разных доменов небезопасно → оператор-подтверждение, не полностью автоматом.
3. **PR/MR полнота**: title/reviewers/status/approvals — только GitLab PAT (на VPS нет). merges = прокси.

Объём: RealGitAdapter (+RawGitMerge) поверх готового #19-контракта ≈ небольшой адаптер. Identity-резолвер + seed из teams + unresolved-отчёт ≈ отдельный слой (модель уже готова, нужна логика резолва + отчёт). Алиас-подтверждение оператором — задел UI/аннотаций.
