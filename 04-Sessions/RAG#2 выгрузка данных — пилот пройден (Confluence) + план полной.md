---
title: RAG#2 выгрузка данных — пилот пройден (Confluence) + план полной
type: note
permalink: tacticum/04-sessions/rag-2-vygruzka-dannykh-pilot-proiden-confluence-plan-polnoi
tags:
- rag2
- ingest
- confluence
- jira
- adp_emb
- pilot
- document-processing
- done
---

## Пилот (2026-07-15) — задача 2, доступ через adp_emb
Цель — полная выгрузка Jira/Confluence на helm для RAG#2. Пилот на Confluence **пройден end-to-end**.

## Решения по scope/контуру (руководитель)
- Целимся в **полное продуктовое/портфельное покрытие** (все продукты + вся Jira), фазами.
- **Исключаем:** personal-пространства (111 из 225) + HR/🔴/архив (HRHITECH, HR1, birthday, IS, Legal/LEGALIVA, integrator/Реестр контрактов, KEYCLT, SSFYL, org и т.п.).
- **Контур:** Jira/Confluence наружу в DeepInfra на эмбеддингах — руководитель: «плохо, но можно». **Чаты/почту — нельзя** (строго on-prem).

## Масштаб (замерено)
- Confluence: **31 038 страниц** в 225 пространствах (114 global + 111 personal). In-scope (product/process global минус HR) — ориентир ~15–20k.
- Jira: 14 проектов, до ~14k задач в крупных (VCSMOB), суммарно ~50k.

## Доступ/креды
- adp_emb: VPN ESTABLISHED; `jira.iva.ru`/`wiki.iva.ru` → 10.22.0.10.
- **Confluence:** `/var/lib/tacticum/env/confluence.env` (`CONFLUENCE_BASE_URL`+`CONFLUENCE_TOKEN`) — работает.
- **Jira: токена НЕТ** в env adp_emb (Confluence-токен для Jira = 401). Нужен Jira-PAT (или через iva-mcp). **Блокер Jira-пилота.**

## Пилот Confluence (пространство IVADS, 51 стр) — результат
- Экстрактор `/root/rag2_pilot/conf_pilot.py` на adp_emb: REST `expand=body.storage,version,ancestors,metadata.labels,space` + вложения. Пишет per-page JSON (формат helm) + `manifest.json` + `items.jsonl` (id/version/updated/hash — для дельт).
- **51 стр + 18 вложений (15 png, 3 zip), 7.6 МБ, 36с → 1.42 стр/с.**
- Перенос **сервер→сервер** (helm тянет с adp_emb по ключу `adp-jump`, НЕ через ноут — deny на локальную копию ИВА-данных).
- **Качество (парсер helm в контейнере):** 51/51 load, 86 чанков, markdown чистый (таблицы→пайпы, heading-путь сохранён). Формат 100% совместим.

## Оценка времени (экстраполяция)
- Confluence in-scope ~15–20k × 1.42 стр/с ≈ **~3–4 ч** аккуратным темпом (можно ускорить снижением паузы 0.3→0.1с). + Jira (нужен токен + замер).

## document_processing (reuse — решение)
`rag_eval_service/document_processing` (`doc_extract`) — сервис `POST /extract` → `list[TextSegment]`: **pdf/docx/xlsx/rtf/pptx + png/jpg с OCR (Tesseract rus+eng)**, ядро из `doc_translator`. **Берём его** для вложений вместо helm-овского TODO-extractor (закрывает pdf/docx + OCR картинок). Схема: экстрактор тянет файл → `/extract` → сегменты → индекс как `*_attachment`.

## Осталось
1. **Jira-пилот:** получить Jira-PAT (или iva-mcp) → вытянуть 1 проект полно (desc+комменты+changelog+links+attachments, helm ест `com/ch/links/att`) → замер.
2. **Вложения:** поднять/подключить `document_processing`, прогнать csv/xlsx/pdf требований.
3. **Полный прогон:** экстрактор по всем in-scope пространствам/проектам, resumable + дельты, ночью; → helm → `rag2_index` → golden-eval.

Связано: [[Доступ к двум ВПС- реранкер (gateway) + полная выгрузка данных (ИВА-adp_emb)]], [[План RAG#2 (аналитики+поддержка)- Jira+Confluence+helm-данные]].

## ОБНОВЛЕНИЕ (2026-07-15, позже) — Jira-блокер СНЯТ + Jira-пилот пройден
- **Jira-доступ решён:** REST принимает **Basic-auth** `monakhov-tech:<EVA-пароль>` → HTTP 200. Аккаунт в группе `ivatech-jira-browse-allprojects` → видит **все** проекты (в отличие от старого урезанного PAT). Отдельный Jira-PAT не нужен. (В env adp_emb Jira-токена нет; используем Basic-auth тем же логином, что EVA/Confluence-SSO.)
- **Jira-пилот (IVAONE, 50 задач)** через `/root/rag2_pilot/jira_pilot.py` (search `fields=*all&expand=changelog`): **50 задач за 18.2с → 2.75 задачи/с**. Пример IVAONE-12517: 4 коммента, 145 записей changelog, 21 связь — полнота подтверждена.
- **Формат под helm:** `k/sum/desc/com/ch/links/att` + payload. helm `build_document` собрал текст со всеми секциями (**Комментарии/История изменений/Связи** — есть). Формат совместим 1:1.

## Итоговые замеры для полной выгрузки
- Confluence ~1.42 стр/с → in-scope ~15–20k → **~3–4 ч**.
- Jira ~2.75 задачи/с → ~50k → **~5 ч**.
- Итого **~8–9 ч** аккуратным темпом (пауза 0.3/0.25с), resumable, ночью. Ускоряемо снижением пауз/параллелизмом (~вдвое).

## Placement doc_processing (обсуждение, не решено)
Вариант A (рекомендую) — извлечение вложений на adp_emb (не грузить маленький helm-хост OCR/PDF); B — на helm ingest-компонентом. Всё оформляем **через git** (ветка/PR), пилот-скрипты — черновики.
