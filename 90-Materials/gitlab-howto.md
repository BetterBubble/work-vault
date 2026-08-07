---
title: gitlab-howto
type: note
permalink: tacticum/90-materials/gitlab-howto
---

# GitLab (git.cifragen.ru) — доступ, пуш, CI/CD. Кратко

## 1. Доступ (один раз)

1. **Аккаунт** — саморегистрации нет, заводит админ. Напиши в общий чат → получишь логин/пароль → смени пароль при первом входе.
2. **SSH-ключ**: `git.cifragen.ru → Preferences → SSH Keys` → вставить `~/.ssh/id_ed25519.pub`.
   Проверка: `ssh git@git.cifragen.ru` → `Welcome to GitLab, @username!`
3. **Токен для API/агентов** (по необходимости): `Preferences → Access Tokens` → scope `api`, срок ≤ 90 дней. Хранить в env-переменной, не в коде.

## 2. Как лить код

```bash
git clone git@git.cifragen.ru:tacticum/<repo>.git
git checkout -b feat/my-change     # main защищён, прямой пуш нельзя
# ... правки, коммиты ...
git push -u origin feat/my-change  # → создать MR в main через UI/ссылку из пуша
```

- Ветки: `feat/*`, `fix/*`, `ci/*`. Коммиты — Conventional Commits (`feat: …`, `fix: …`).
- Мерж MR: после зелёного пайплайна, обычным **merge-commit** (sync-ветки не squash'ить — ломает историю).
- Мержить в защищённый main может Maintainer проекта.

## 3. То же самое агентом (Claude Code / Codex)

- Агент работает **только через ветку + MR**, в main напрямую не пушит.
- Git: агент коммитит от твоего имени (`git config user.name/email`), **без AI-атрибуции** в коммитах/MR.
- API (создать MR, посмотреть пайплайн): `curl -H "PRIVATE-TOKEN: $GITLAB_TOKEN" https://git.cifragen.ru/api/v4/...` — отдай агенту токен через env.
- Типовая просьба агенту: «склонируй `tacticum/<repo>` по SSH, сделай ветку, внеси X, запушь и открой MR в main; MR не мержить».

## 4. CI/CD — как это работает

**Раннер:** общий `default-docker-runner` (docker, без privileged) → образы собираются **Kaniko**, вспомогательные БД в тестах — через `services:`.

**Типовой пайплайн** (`.gitlab-ci.yml` в корне репы):

```
lint (ruff/mypy) → test → security (gitleaks) → build (Kaniko → registry) → deploy
```

**Образы:** `registry.cifragen.ru/tacticum/<repo>/<image>`
- пуш в `main` → теги `:sha` + `:latest`
- git-тег `vX.Y.Z` → тег-образ `:vX.Y.Z` (релиз)

**Деплой (CD):**
- **re** — авто на **каждый коммит в main**: CI по SSH накатывает код и рестартует сервис сначала на `vps1` (38.180.236.39), при успехе+healthz — на `vps2` (31.129.105.187). С фич-веток — ручная кнопка `deploy:vps1` (обкатка). Откат = revert-коммит в main.
- **dev** — в работе: образы из registry + деплой через Portainer (portainer.cifragen.ru), авто на main с авто-откатом по healthz.

**Мониторинг:** дашборд CI/CD в grafana.cifragen.ru + Telegram-алерты: красный пайплайн в main, экспортёр down, диск registry >80%, образ >1GB. Красный деплой = алерт прилетит сам.

**Если пайплайн красный:** открой джобу → лог. Частое: ruff/mypy (чини код), gitleaks (не коммить секреты; FP — в `.gitleaks.toml` allowlist), сборка (Dockerfile). Тесты в re advisory (не блокируют), в dev/platform — блокирующие.