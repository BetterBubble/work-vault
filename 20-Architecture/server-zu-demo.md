---
title: server-zu-demo
type: config
permalink: tacticum/20-architecture/server-zu-demo
tags:
- zu_demo
- infra
- deploy
- server
---

# Сервер zu_demo (демо-стенд ЗУ)
Доступ: root по ключу через ssh-manager MCP (режим readonly). IP 159.194.212.2.
ВАЖНО — грабли, не наступать:
- ДВА дерева кода: /home/tacticum-deploy/codex-git/ — БОЕВОЕ (из него собирается контейнер). /root/codex/ — старое, мёртвое, контейнер его НЕ читает. Работать только с codex-git.
- Права: боевой код принадлежит tacticum-deploy (uid 1000). Операции с кодом/git — через sudo -u tacticum-deploy, НЕ от root (иначе ломается владелец файлов и git ругается dubious ownership — это защита, не обходить).
- Код довозится через git pull в codex-git (от tacticum-deploy). Данные (gitignored: golden-наборы, конфиги) — через rsync точечно + chown tacticum-deploy. НЕ лить rsync весь код — конфликт с git.
- Docker: команды docker compose — из /root/zu-deploy/ (там docker-compose.yml). Из другого места — "no configuration file". Compose собирает build: /home/tacticum-deploy/codex-git.
- Контейнер собирает код в ОБРАЗ (COPY), не монтирует: правка файла на диске не видна в контейнере до пересборки. Данные (golden) можно закинуть без пересборки через docker compose cp, код — только пересборкой.
- .env НЕ трогать: боевой .env в codex-git с секретами. При rsync — --exclude='.env'. Бэкапить перед изменениями.
- Секреты (Gateway/Meili/project-hub ключи) в серверном .env, их читают сами приложения через docker env_file. Claude НЕ читает и НЕ выводит значения (cat .env / printenv / env / echo $KEY — нельзя). Работа через контейнеры, которые сами подтягивают ключи.
- Системное (docker daemon, swap, пакеты) — делает пользователь сам под root, не Claude.

## Связи
- relates_to [[lightrag-в-codex]]
