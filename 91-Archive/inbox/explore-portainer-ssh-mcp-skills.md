---
title: explore-portainer-ssh-mcp-skills
type: note
permalink: tacticum/00-board/explore-portainer-ssh-mcp-skills
status: draft
role: explorer
tags:
- explore
- portainer
- mcp
- ssh-mcp
- skills
- draft
archived-at: 2026-07-29 18:12
---

# explore-portainer-ssh-mcp-skills

Разведка скиллов Portainer/SSH-MCP из репо `TacticumApps/agents` (по запросу ГД для рекомендации пользователю).

## Доступ
Прочитано из **локального клона**: `/Users/bubblemac/tacticum/agents` (git remote `git@github.com:TacticumApps/agents.git`). GitHub API/raw не понадобились (репо приватный — API отдал 404 без токена, но клон есть локально).

Реальный состав папки скилла (`/Users/bubblemac/tacticum/agents/.claude/skills/portainer/`):
- `SKILL.md`
- `portainer-mcp-hygiene.md`

**Важные расхождения с постановкой ГД:**
- **Отдельного SSH-MCP скилла НЕТ.** SSH-MCP описан только внутри `portainer/SKILL.md` как шаг подключения.
- **systemd-юнитов в репо НЕТ вообще.** `grep -rli 'systemd|ExecStart|WantedBy|.service'` по всему репо — ноль совпадений. Автозапуск в скилле решается через `docker run --restart always`, а не systemd.
- Рядом лежит ещё скилл `install-openviking` — он про плагин памяти OpenViking, к Portainer/SSH отношения не имеет.
- `.mcp.json` в самом репо `agents` отсутствует (скилл предполагает, что он в проекте, откуда запускают Claude Code).

## Portainer-MCP — назначение и операции
Управление Docker-окружениями через Portainer HTTP API из Claude Code. Инстанс: UI/API `https://portainer.cifragen.ru` (self-signed TLS, `PORTAINER_TLS_VERIFY=0`). Тулы вида `mcp__portainer__*`.

Пакет MCP-сервера: `uvx --from "mcp-portainer~=2.43.0" mcp-portainer`. Env: `PORTAINER_URL`, `PORTAINER_API_KEY` (`<ptr_...>` из My Account → Access tokens), `PORTAINER_TLS_VERIFY=0`, опц. `PORTAINER_EXPOSE_ENV_VALUES=1` (иначе значения env всегда `[REDACTED]`).

Ключевой тул `get_guidance` — вызывать в начале любой задачи, отдаёт полный гайд в сессию.

Операции (символ → назначение), файл `SKILL.md`:
- Окружения: `EndpointList`, `EndpointCreate` (добавить сервер как Agent, type=2), `EndpointDelete`. Типы окружений: 1=Local Docker, 2=Agent, 4=Edge, 5=Local K8s, 6=KubeConfig. Статус прямых агентов: 1=up, 2=down; edge — по `heartbeat`.
- Стеки: `StackList`, `StackCreateDockerStandaloneString` (создать compose-стек из строки), `StackUpdate` (Prune/PullImage), `StackStop`/`StackStart`, `StackDelete`, `StackFileInspect`, `endpointForceUpdateService` (форс-пул образа сервиса). Типы стеков: 1=Swarm, 2=Compose, 3=K8s.
- Контейнеры/сети/volume/образы — через generic `docker_proxy(environment_id, method, path, body, select)` (проксирует Docker API: `/containers/json`, `/containers/<id>/logs|restart|json`, `/networks`, `/volumes`, `/images`). Плюс `dockerImagesList`. Для K8s — `kubernetes_proxy`.

Что требуется на целевом сервере: запущенный **Portainer Agent** `portainer/agent:2.43.0` на порту **9001** (`docker run -d --restart always -p 9001:9001 -v /var/run/docker.sock -v /var/lib/docker/volumes ...`). Между сервером-агентом и Portainer-core должен быть открыт порт 9001.

## Portainer как таковой (из скилла)
- Portainer-core — веб/API инстанс на `portainer.cifragen.ru` (в скилле как разворачивать core НЕ описано — он считается уже поднятым; core API отвечает на `:9443/api/status`).
- Новые серверы подключаются моделью **Agent (type=2)**: на сервере поднимается `portainer/agent` контейнером, в Portainer создаётся Environment с `URL=tcp://<IP>:9001`, `TLS=True, TLSSkipVerify=True, TLSSkipClientVerify=True`.
- Существующие compose-стеки: скилл создаёт/обновляет их как Portainer Stacks из compose-строки (`StackCreateDockerStandaloneString` / `StackUpdate`), билд/пул образов — на стороне сервера через Docker API.
- Про сборку образов on-server отдельного раздела нет; управление образами — только list/delete + force-update сервиса.

## SSH-MCP — назначение и настройка (внутри SKILL.md, отдельного скилла нет)
Назначение: доступ к серверам по SSH из Claude Code для подготовки сервера (поставить agent, проверить `docker ps`, узнать IP-интерфейсы). Тул: `mcp__ssh-mcp-<name>__exec`.

Пакет: `npx ssh-mcp --host=<IP> --user=<user> --password=<password>`. Запись в `.mcp.json`:
```
"ssh-mcp-<name>": {"type":"stdio","command":"npx","args":["ssh-mcp","--host=<IP>","--user=root","--password=<pwd>"],"env":{}}
```
Один MCP-инстанс на сервер (`ssh-mcp-<name>`). После правки `.mcp.json` — перезапуск сессии Claude Code. Конкретные серверы в скилле не перечислены (host/user/password подставляет пользователь).

## systemd
В репо/скилле systemd-юнитов НЕТ. Автозапуск Portainer-agent обеспечивается флагом Docker `--restart always`, не systemd. Если ГД ждал примеры юнитов — их в этом источнике не существует (нужно уточнить, откуда взялось ожидание).

## Шаги внедрения (workflow «добавить новый сервер», из SKILL.md)
1. Онбординг: проверить `.mcp.json` — подключены ли `portainer` и `ssh-mcp-*`; отсутствующие добавить (вариант А — агент вставляет токен/креды сам; вариант Б — пользователь через `claude mcp add`). После — перезапуск сессии.
2. Подключиться к серверу по SSH (`ssh-mcp-<name>`), проверить `docker ps`.
3. Определить IP для агента: предпочитать внутреннюю подсеть (10.x/172.x/192.168.x) — проверка `curl -sk https://<portainer-core-internal-ip>:9443/api/status`; если общей подсети нет — публичный IP core.
4. Поставить Portainer Agent контейнером (`portainer/agent:2.43.0`, порт 9001, `--restart always`), проверить `docker ps --filter name=portainer_agent`.
5. `EndpointCreate(EndpointCreationType=2, URL="tcp://<IP>:9001", TLS=True, TLSSkipVerify=True, TLSSkipClientVerify=True)`.
6. Верифицировать: `EndpointList([?name=='new-server'])` → `status:1` = агент виден; `status:2` → firewall/порт 9001.

## Риски / оговорки (из текста скилла)
- **Токены/креды в открытую.** Скилл предлагает агенту вписывать `PORTAINER_API_KEY` и SSH `--password` прямо в `.mcp.json` (в args/env, plaintext). SSH обычно под `--user=root`. Это чувствительно для гита/шаринга (в самом репо `.mcp.json` нет — и хорошо).
- **TLS не проверяется** (`PORTAINER_TLS_VERIFY=0`, `TLSSkipVerify/TLSSkipClientVerify=True`) — self-signed, MITM-риск на публичных каналах.
- **Мутации не гарантированы ответом.** Write-операции часто возвращают пустое тело/`{"Output":""}` — это НЕ подтверждение; hygiene-док требует read-проверку после каждой мутации (напр. `StackList([?name=='...'])` → `[]`).
- **env всегда `[REDACTED]`** — чтобы увидеть значения, надо `PORTAINER_EXPOSE_ENV_VALUES=1` (доп. экспозиция секретов).
- **Тяжёлые ответы** — без JMESPath `select` список окружений со снэпшотами 20K+ токенов; hygiene требует всегда передавать `select`.
- **K8s-рестарт** — через scale replicas 0→1, не delete pod.
- **Открытый порт 9001** между серверами — сетевой периметр (в скилле firewall упомянут только как причина `status:2`).
- **Drift** как термин в скилле НЕ упоминается (постановка ГД спрашивала — прямо про drift текста нет).

## Файлы-источники (абсолютные пути)
- `/Users/bubblemac/tacticum/agents/.claude/skills/portainer/SKILL.md`
- `/Users/bubblemac/tacticum/agents/.claude/skills/portainer/portainer-mcp-hygiene.md`
- (не по теме) `/Users/bubblemac/tacticum/agents/.claude/skills/install-openviking/SKILL.md`