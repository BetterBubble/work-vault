---
title: 'Тех-долг QA: живая синхронизация ivaqa/kit GitLab → наш QA-профиль'
type: note
permalink: tacticum/00-board/tekh-dolg-qa-zhivaia-sinkhronizatsiia-ivaqa-kit-git-lab-nash-qa-profil
created: '2026-07-24'
updated: '2026-07-24'
project: tacticum-dev / профили
status: open
tags:
- tech-debt
- qa
- iva-role-qa
- kit
- gitlab
- sync
- upstream
---

# Тех-долг: подтягивать и обновлять инфу из их GitLab в наш QA-профиль

**Записано:** 2026-07-24 (решение президента). **Тип:** тех-долг QA. **Статус:** open (не блокер текущей передачи, нужен для актуальности + мульти-стэка).

## Проблема
Наш QA-профиль (`iva-role-qa`: 9 скиллов + 3 субагента + стек `pytest-playwright-canvas`) — **byte-copy из upstream `ivaqa/kit`** команды Жени (Брейкина). Сейчас источник взят **разовым снапшотом** `kit-main.zip` (marketplace v1.5.1, репо создан 2026-07-19). **Живого доступа к `git.hi-tech.org/ivaqa/kit` у нас НЕТ** ([[recon-aqa-kit-zhenya]]) — значит наш профиль со временем **расходится** с upstream (kit обновляется командой, мы — нет).

## Что нужно (суть тех-долга)
Механизм, чтобы **нужная инфа из их GitLab (`ivaqa/kit`) подтягивалась и обновлялась в наш профиль** — а не жила разовым слепком. Держать наш QA-профиль в актуальном состоянии относительно upstream.

## Что для этого требуется
1. **Живой доступ к `git.hi-tech.org/ivaqa`** — от Жени (владелец группы): PAT `read_repository` / деплой-ключ (на adp_emb или helm) / регулярное зеркало. Общий доступ к GitLab ИВА — через DevO, но приватную группу `ivaqa` открывает Женя.
2. **Модель обновления** (выбрать): (a) периодическое зеркало/архив → ре-импорт; (b) submodule/adapter на upstream kit; (c) ручной ре-снапшот «по петлям» (как kit сам обновляет своих потребителей — каналы `main`/`stable`/пин-тег).
3. **Скоуп синка** — что именно тянем: craft-стек (`pytest-playwright-canvas`), 3 субагента, скиллы write/batch/fix + оркестраторы, KMP-линия. (iOS в kit нет — не тянуть, отдельный вопрос.)

## Контекст мульти-стэка
- **KMP** — ядро kit, тестируется как web (Playwright + canvas accessibility-overlay + testTag'и, стенд `kmp-stage`, репо `one-web-kmp`); нативного пути нет.
- **iOS** — в kit **ничего нет**; для iOS-профиля нужны отдельные вводные от команды Жени.

## Связано
[[napravlenie-profili-qa-profil-iva-role-qa-aqa-toolkit-iva]] · [[recon-aqa-kit-zhenya]] · [[recon-kit-full-qa-dorabotka]] · [[qa-profile-model-opis-multi-stek-model-qa-leinov]]