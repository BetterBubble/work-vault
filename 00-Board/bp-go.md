---
title: Best practices — Go-бэкенд (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, go]
permalink: tacticum/00-board/bp-go
---

# Go-бэкенд — официальные источники best practices

> **Коротко:** в профиле версия языка не зафиксирована — описание стека называет только «Go» и набор библиотек; актуальные на 2026-08-03 — Go 1.26.5 и 1.25.12 (обе поддерживаются, 1.24 уже вне поддержки), 1.27 в статусе черновика; то, что названо в профиле, поддерживается, но нижняя граница де-факто уже 1.25 — её требуют goose v3.27.0 и grpc-go.

## Стек по профилю

Дословно из профиля (`TABLE_BLURBS` в `scripts/wiki_sync_manuals.py`, репозиторий TacticumApps/dev):
«Go; NATS + gRPC микросервисы; внешний proto-репозиторий; Postgres через sqlc + goose; samber/do DI; golangci-lint».
Плюс слоистая архитектура delivery → usecase → repository → entity и OTel.
Агентные профили: `iva-role-go`, `iva-go-backend-brownfield`.

Что из этого имеет живой официальный источник обновлений: язык и тулчейн Go, линтеры (golangci-lint,
staticcheck, vet), gRPC-Go, protobuf-go, nats.go, sqlc, goose, samber/do, opentelemetry-go.
Слоистая архитектура официального источника не имеет вообще — это наша внутренняя норма, сверять не с чем.

## Источники

### Язык, стиль, процесс

| Источник | Тип | Чей | Что именно берём | Обновляется | Сверено |
|---|---|---|---|---|---|
| [go.dev/doc/effective_go](https://go.dev/doc/effective_go) | доки | Go team | База идиом языка. **Важно:** в шапке официальная пометка — документ написан в 2009, активно не обновляется, не покрывает дженерики и модули (issue 28782). Как источник «современного стиля» брать нельзя | не обновляется | 2026-08-03 |
| [go.dev/wiki/CodeReviewComments](https://go.dev/wiki/CodeReviewComments) | доки | Go team (вики) | 35+ разделов типовых замечаний на ревью: именование, ошибки, receivers, контексты, интерфейсы. Правки требуют предварительного обсуждения — страница стабильна | редко | 2026-08-03 |
| [google.github.io/styleguide/go](https://google.github.io/styleguide/go/) | доки | Google | Три уровня: Style Guide (канон) → Style Decisions (нормативно, с обоснованиями) → Best Practices (рекомендательно). Самый детальный из живых стайл-гайдов | без дат версий | 2026-08-03 |
| [go.dev/doc/comment](https://go.dev/doc/comment) | доки | Go team | Норма doc-комментариев: пакеты, типы, функции, doc links, директивы. То, что проверяет `gofmt` с Go 1.19 | редко | 2026-08-03 |
| [go.dev/doc/modules/gomod-ref](https://go.dev/doc/modules/gomod-ref) | доки | Go team | Справочник директив `go.mod`: `go`, `require`, `replace`, `retract`, `tool`, `godebug`, `ignore` | с релизами | 2026-08-03 |
| [go.dev/doc/security/best-practices](https://go.dev/doc/security/best-practices) | доки | Go team | govulncheck в CI, race detector (`go test -race`), fuzzing, регулярное обновление тулчейна, подписка на golang-announce | редко | 2026-08-03 |
| [github.com/golang/proposal](https://github.com/golang/proposal) | proposal | Go team | Процесс предложений: issue → обсуждение → design doc → решение. Статусы Incoming → Active → Likely Accept/Decline → Accepted/Declined, ревью еженедельно. Активные — в [проекте Proposals](https://github.com/orgs/golang/projects/17) | еженедельно | 2026-08-03 |

### Релизы Go

| Источник | Тип | Чей | Что именно берём | Обновляется | Сверено |
|---|---|---|---|---|---|
| [go.dev/doc/devel/release](https://go.dev/doc/devel/release) | релизы | Go team | Полная история версий и политика поддержки: версия живёт до выхода двух следующих мажорных. Текущее: 1.26.5 (07.07.2026), 1.25.12 (07.07.2026), 1.24 уже вне поддержки | два мажора в год + патчи ~ежемесячно | 2026-08-03 |
| [go.dev/doc/go1.25](https://go.dev/doc/go1.25) | релизы | Go team | Релиз 12.08.2025: `testing/synctest`, container-aware GOMAXPROCS, flight recorder, новые vet-анализаторы | зафиксирован | 2026-08-03 |
| [go.dev/doc/go1.26](https://go.dev/doc/go1.26) | релизы | Go team | Релиз 10.02.2026: Green Tea GC по умолчанию, `new(expr)`, переписанный `go fix`, `errors.AsType` | зафиксирован | 2026-08-03 |
| [go.dev/doc/go1.27](https://go.dev/doc/go1.27) | релизы | Go team | **Черновик**, релиз ожидается в августе 2026: generic methods, `encoding/json/v2` как стандарт, удаление старых GODEBUG. К релизу текст может измениться | до релиза меняется | 2026-08-03 |
| [go.dev/blog](https://go.dev/blog/) | блог | Go team | Разборы «как правильно» от авторов языка. За последний год: `go fix` (17.02.2026), `//go:fix inline` (10.03.2026), Green Tea GC (29.10.2025), Flight Recorder (26.09.2025), результаты опроса разработчиков (21.01.2026) | ~1–2 поста в месяц | 2026-08-03 |
| [go.dev/blog/gofix](https://go.dev/blog/gofix) | блог | Go team (Alan Donovan) | Конкретика по модернизации кода: `go fix -diff ./...`, фиксеры `minmax`, `rangeint`, `stringscut`, `newexpr`. Это и есть машинно выраженный ответ «пиши современно» | зафиксирован | 2026-08-03 |

### Линтеры — практика, выраженная машиной

| Источник | Тип | Чей | Что именно берём | Обновляется | Сверено |
|---|---|---|---|---|---|
| [pkg.go.dev/cmd/vet](https://pkg.go.dev/cmd/vet) | линтер | Go team | Список встроенных анализаторов (printf, errorsas, loopclosure, timeformat…). `go tool vet help` — полный перечень | с релизами Go | 2026-08-03 |
| [golangci-lint.run/docs/product/changelog](https://golangci-lint.run/docs/product/changelog/) | линтер | golangci | История версий. Последняя — v2.12.2 (06.05.2026); v2.0.0 вышла 24.03.2025 | ~ежемесячно | 2026-08-03 |
| [golangci-lint.run/docs/welcome/quick-start](https://golangci-lint.run/docs/welcome/quick-start/) | линтер | golangci | Набор по умолчанию — ровно пять: `errcheck`, `govet`, `ineffassign`, `staticcheck`, `unused` | с версиями | 2026-08-03 |
| [golangci-lint.run/docs/product/migration-guide](https://golangci-lint.run/docs/product/migration-guide/) | линтер | golangci | Что изменилось в v2: `version: "2"` в конфиге, форматтеры (`gci`, `gofmt`, `gofumpt`, `goimports`) вынесены в секцию `formatters`, пресеты удалены, `disable-all/enable-all` → `linters.default: none/all`. Есть команда `golangci-lint migrate` | зафиксирован | 2026-08-03 |
| [.golangci.reference.yml](https://raw.githubusercontent.com/golangci/golangci-lint/main/.golangci.reference.yml) | линтер | golangci | Эталонный конфиг со всеми опциями. `linters.default` принимает `standard` (дефолт) / `all` / `none` / `fast` | с версиями | 2026-08-03 |
| [staticcheck.dev/changes](https://staticcheck.dev/changes/) | линтер | dominikh | Последняя — 2026.2 (v0.8.0): поддержка Go 1.27 (generic methods), новая проверка SA9010 (`defer foo()`, где foo возвращает функцию), **SA5011 отключена** — вместо неё рекомендован анализатор `nilness` в gopls | ~2 релиза в год | 2026-08-03 |
| [x/tools/gopls/…/modernize](https://pkg.go.dev/golang.org/x/tools/gopls/internal/analysis/modernize) | линтер | Go team | Анализатор, предлагающий современные конструкции; работает и через gopls, и как подкоманда с `-fix`. Тот же движок, что за `go fix` в 1.26 | с релизами gopls | 2026-08-03 |

### Библиотеки стека

| Источник | Тип | Чей | Что именно берём | Обновляется | Сверено |
|---|---|---|---|---|---|
| [github.com/grpc/grpc-go/releases](https://github.com/grpc/grpc-go/releases) | релизы | gRPC | Последний — v1.83.0 (30.07.2026). Релизы каждые ~6 недель, в каждом behavior changes и security fixes. В [README](https://github.com/grpc/grpc-go/blob/master/README.md) политика: поддерживаются «any one of the two latest major» релизов Go | ~раз в 6 недель | 2026-08-03 |
| [grpc.io/docs/languages/go](https://grpc.io/docs/languages/go/) | доки | gRPC | Гайды по метаданным, интерсепторам, статус-кодам. **Внимание:** страница помечена «last modified October 30, 2023» — концептуально годится, за новинками сюда не ходить | редко | 2026-08-03 |
| [protobuf.dev/reference/go/opaque-migration](https://protobuf.dev/reference/go/opaque-migration/) | доки | Google | Opaque API: поля не экспортируются, доступ через аксессоры. Для edition 2024 и новее — **по умолчанию**. Миграция через `open2opaque` (HYBRID → rewrite → OPAQUE), нужны protoc ≥29.0 и protoc-gen-go ≥1.36.0. Прямо касается внешнего proto-репозитория | по мере изменений | 2026-08-03 |
| [github.com/protocolbuffers/protobuf-go/releases](https://github.com/protocolbuffers/protobuf-go/releases) | релизы | Google | Последний — v1.36.11 (12.12.2025). Значимое за год: поддержка edition 2024 (v1.36.9) | нерегулярно | 2026-08-03 |
| [github.com/nats-io/nats.go/releases](https://github.com/nats-io/nats.go/releases) | релизы | Synadia/NATS | Последний — v1.52.0 (07.05.2026), под фичи сервера 2.14 (batch publish, message scheduling). Ритм — раз в 1–2 месяца | ~ежемесячно | 2026-08-03 |
| [nats.go/jetstream/MIGRATION.md](https://github.com/nats-io/nats.go/blob/main/jetstream/MIGRATION.md) | доки | Synadia/NATS | **Ключевое для нас:** старый JetStream API (`nc.JetStream()`, `nats.JetStreamContext`) официально deprecated. Новый путь — пакет `jetstream`: `jetstream.New(nc)`, явное создание стримов/консьюмеров, обязательный `context.Context`, pull-консьюмеры (`Consume()`/`Messages()`) вместо `Subscribe()` | по мере изменений | 2026-08-03 |
| [github.com/nats-io/nats-server/releases](https://github.com/nats-io/nats-server/releases) | релизы | Synadia/NATS | Последние — v2.14.4 и v2.12.14 (обе 30.07.2026). Читаем ради security-фиксов и того, какие фичи вообще доступны клиенту | ~ежемесячно | 2026-08-03 |
| [docs.sqlc.dev/…/changelog](https://docs.sqlc.dev/en/latest/reference/changelog.html) | релизы | sqlc | Последняя — v1.31.1 (22.04.2026). В v1.31.0: форматирование SQL AST, разворачивание `SELECT *`, нативная БД для тестов без Docker, подкоманда `parse` | несколько релизов в год | 2026-08-03 |
| [pressly.github.io/goose](https://pressly.github.io/goose/) | доки | pressly | [Provider API](https://pressly.github.io/goose/documentation/provider/) — точка входа без глобального состояния, `embed.FS`, блокировки БД для нескольких процессов (актуально для k8s). Для встроенных в сервис миграций — рекомендуемый путь вместо глобальных функций | нерегулярно | 2026-08-03 |
| [github.com/pressly/goose/releases](https://github.com/pressly/goose/releases) | релизы | pressly | Последний — v3.27.3 (22.07.2026). В v3.27.0 (22.02.2026) минимальная версия Go поднята до 1.25 и изменены шаблоны SQL-миграций; в v3.26.0 — `*slog.Logger`, `WithTableName`, интерфейс `Locker` | ~раз в 1–2 месяца | 2026-08-03 |
| [do.samber.dev](https://do.samber.dev/) | доки | samber | Документация v2: Scope-дерево (RootScope + дочерние), transient-сервисы, health-checks, graceful shutdown | активно | 2026-08-03 |
| [do.samber.dev/docs/upgrading/from-v1-x-to-v2](https://do.samber.dev/docs/upgrading/from-v1-x-to-v2) | доки | samber | Гайд миграции (обновлён 28.07.2026): импорт `samber/do/v2`, `*do.Injector` → `do.Injector` (стал интерфейсом), `ShutdownOnSIGTERM()` удалён, `Shutdown()` возвращает `map[string]error` и неблокирующий, хуки стали слайсами с параметром ошибки, имена сервисов включают полный путь пакета | по мере изменений | 2026-08-03 |
| [github.com/samber/do/releases](https://github.com/samber/do/releases) | релизы | samber | v2.0.0 — 21.09.2025 (стабильная, до этого два года beta/rc), последняя v2.1.0 — 20.07.2026 (оптимизация аллокаций и DAG) | редко | 2026-08-03 |
| [github.com/open-telemetry/opentelemetry-go/releases](https://github.com/open-telemetry/opentelemetry-go/releases) | релизы | OTel | Последний — v1.44.0 (27.05.2026). Breaking: лимит кардинальности метрик 2000 по умолчанию; лимит OTLP-запроса 64 MiB; self-observability экспортёров под `OTEL_GO_X_SELF_OBSERVABILITY` | ~ежемесячно | 2026-08-03 |
| [opentelemetry.io/docs/languages/go](https://opentelemetry.io/docs/languages/go/) | доки | OTel | Статусы сигналов и правила ручного инструментирования. На странице: traces — stable, metrics — stable, logs — **beta** | активно | 2026-08-03 |

## Что изменилось за последний год

Окно: август 2025 — август 2026. Что реально меняет то, как надо писать код сегодня.

**1. Тестирование конкурентности перестало быть кустарным (Go 1.25, 12.08.2025).**
`testing/synctest` — стабильный пакет с виртуализированным временем: часы двигаются мгновенно, когда все
горутины заблокированы. Всё, что раньше писалось через `time.Sleep` и флейки в CI, теперь пишется через
`synctest.Test`. Профилям разработчиков надо перестать генерировать sleep-based тесты.

**2. GOMAXPROCS больше не надо чинить руками (Go 1.25).**
Рантайм на Linux сам учитывает CPU-лимиты cgroup и периодически их перечитывает. Внешние обвязки вроде
`uber-go/automaxprocs` в новых сервисах избыточны. Отключение — `GODEBUG=containermaxprocs=0`.

**3. `go fix` из исторического рудимента стал инструментом модернизации (Go 1.26, 10.02.2026).**
Команда переписана целиком: десятки фиксеров под современные идиомы (`min`/`max`, `for range n`,
`strings.Cut`, `new(expr)`), встроенный source-level inliner с директивой `//go:fix inline`, автоудаление
неиспользуемых импортов. Практический вывод: `go fix -diff ./...` — это машинная проверка «код в стиле
2025 или в стиле 2026», её можно гонять регулярно, а не спорить о стиле словами.

**4. Green Tea GC включён по умолчанию (Go 1.26).**
В 1.25 был экспериментом (`GOEXPERIMENT=greenteagc`), в 1.26 стал дефолтом: ожидаемое снижение накладных
расходов GC на 10–40%, ещё ~10% на Ice Lake / Zen 4+. Плюс ускорение cgo-вызовов ~на 30%. Откат —
`GOEXPERIMENT=nogreenteagc`. Для нас это значит: старые «оптимизации под GC» в коде могут быть уже не нужны.

**5. `encoding/json/v2` перестаёт быть экспериментом (Go 1.27, черновик, август 2026).**
В 1.25–1.26 пакет существовал только под `GOEXPERIMENT=jsonv2` — на pkg.go.dev до сих пор написано «most
users should use encoding/json». В черновике 1.27 он входит в стандартную библиотеку вместе с
`encoding/json/jsontext`, откат — через `GOEXPERIMENT=nojsonv2`. Более строгие умолчания и заметно более
быстрый unmarshal. Это самое крупное изменение, к которому стоит готовиться заранее.

**6. Generic methods (Go 1.27, черновик).**
Метод сможет объявлять собственные типовые параметры — то, чего в языке не было с момента появления
дженериков. Ограничение: методы интерфейсов типовых параметров не объявляют и не реализуются
generic-методами. Плюс обобщённый вывод типов функций и произвольные field-селекторы в ключах структурных
литералов. Staticcheck 2026.2 это уже поддерживает.

**7. golangci-lint v2 — другой конфиг, чем годом раньше.**
v2.0.0 вышла 24.03.2025 (сейчас v2.12.2, 06.05.2026). Конфиг обязан начинаться с `version: "2"`; пресеты
удалены; форматтеры (`gofmt`, `gofumpt`, `goimports`, `gci`) переехали в отдельную секцию `formatters`,
появилась команда `golangci-lint fmt`; `disable-all`/`enable-all` заменены на `linters.default`. Есть
`golangci-lint migrate`. Набор по умолчанию — пять линтеров: errcheck, govet, ineffassign, staticcheck,
unused. Дополнительно за год появились линтеры `modernize` и `godoclint`.

**8. samber/do v2 стал стабильным (21.09.2025) — и это ломающий переход.**
После двух лет beta/rc: `Injector` из структуры стал интерфейсом, появилось дерево Scope (RootScope +
дочерние с локальными или наследуемыми сервисами), transient-сервисы, `Shutdown()` теперь неблокирующий и
возвращает `map[string]error`, `ShutdownOnSIGTERM()` удалён. Если наш код на v1 — это отдельная задача, а
не автозамена импортов.

**9. Старый JetStream API в nats.go официально deprecated.**
`nc.JetStream()` / `JetStreamContext` заменяются пакетом `jetstream`: явное создание стримов и консьюмеров,
обязательный контекст в каждом вызове, pull-консьюмеры как основной способ потребления. Push-консьюмеры
в новом пакете появились (v1.44.0) именно ради миграции. Старый API пока работает, но новый код на нём
писать нельзя.

**10. Мелочи, которые всё равно попадают в код.**
`errors.AsType` — типобезопасная замена `errors.As` (1.26). `log/slog.NewMultiHandler` (1.26). `io.ReadAll`
вдвое быстрее (1.26). В 1.25 починен баг компилятора с отложенной проверкой nil — код вида
`f, err := os.Open(...)` с обращением к `f` до проверки `err` теперь честно паникует. Минимальная версия
Go у goose — 1.25 (v3.27.0), у grpc-go — две последних мажорных, то есть на август 2026 это 1.25 и 1.26 (release notes grpc-go 1.81.0: «Minimum supported Go version is now 1.25»). Go 1.27 на дату сверки ещё не вышел.

## Что переписать в профиле прямо сейчас

- **Убрать Effective Go из списка источников современного стиля** — оставить его только как введение в базовые идиомы, а канон стиля взять из Google Go Style Guide + CodeReviewComments: в шапке Effective Go прямая пометка, что документ написан в 2009 и не покрывает дженерики и модули (issue 28782).
- **Заменить в инструкциях по тестам конкурентности `time.Sleep`-паттерн на `testing/synctest`** — с Go 1.25 это стабильный пакет с виртуальным временем, и sleep-based тесты больше не нужно ни писать, ни принимать на ревью.
- **Убрать рекомендацию тащить `uber-go/automaxprocs` в новые сервисы** — с Go 1.25 рантайм на Linux сам читает CPU-лимиты cgroup и перечитывает их периодически; обвязка избыточна.
- **Заменить во всех примерах JetStream `nc.JetStream()` / `nats.JetStreamContext` на пакет `jetstream`** (`jetstream.New(nc)`, явное создание стримов и консьюмеров, обязательный `context.Context`, pull-консьюмеры) — старый API официально deprecated (jetstream/MIGRATION.md).
- **Переписать требования к golangci-lint под v2 и добавить `go fix -diff ./...` в чеклист** — конфиг обязан начинаться с `version: "2"`, форматтеры переехали в секцию `formatters`, `disable-all`/`enable-all` заменены на `linters.default`, пресеты удалены (migration guide v2); `go fix` из Go 1.26 — это машинная проверка «код современный», вместо словесных споров о стиле.
- **Привести примеры DI к samber/do v2** — `do.Injector` стал интерфейсом (был `*do.Injector`), `ShutdownOnSIGTERM()` удалён, `Shutdown()` неблокирующий и возвращает `map[string]error`; гайд v1 → v2 (обновлён 28.07.2026) прямо перечисляет это как ломающие изменения, автозаменой импортов не решается.

## Чего не нашёл / где источник слабый

- **Effective Go официально мёртв.** В шапке прямая пометка: написан в 2009, активно не обновляется, не
  покрывает дженерики и модули. Держать его в профиле как «источник стиля» — ошибка; он годится только как
  введение в базовые идиомы.
- **Google Go Style Guide без дат и версий.** Ни даты обновления, ни changelog, ни версионирования. Понять
  «что изменилось с прошлой сверки» нельзя иначе как диффом самих страниц. Для регулярной сверки это
  плохой источник — придётся хранить снимок.
- **Состав набора `standard` у golangci-lint зафиксирован только в квик-старте.** В эталонном конфиге
  `.golangci.reference.yml` — лишь ссылка на сайт, в справочнике по линтерам списка тоже нет. Единственный
  надёжный локальный способ — `golangci-lint linters` на своей конфигурации.
- **grpc.io/docs/languages/go помечен «last modified October 30, 2023».** Концепции (интерсепторы,
  метаданные, статус-коды) не устарели, но всё новое живёт только в GitHub-релизах. Единого документа
  «что в gRPC-Go устарело» не существует — только чтение release notes подряд.
- **Расхождение по статусу логов OTel Go.** На странице opentelemetry.io logs помечены как beta, при этом
  отдельные log-компоненты в релизах выглядят стабилизированными. Детально не разбирал — перед тем как
  писать в профиль «логи стабильны», надо проверить конкретный модуль, который мы используем.
- **У sqlc и goose нет раздела «что устарело».** Есть changelog изменений, но нет деклараций
  deprecation — узнать, что перестало быть рекомендованным, можно только по коду и по тому, что исчезло
  из документации.
- **Go 1.27 — черновик.** Всё из пункта про generic methods и json/v2 — из документа со статусом DRAFT.
  К моменту релиза (август 2026) формулировки могут поменяться; перед внесением в профиль пересверить.
- **Даты релизов с HTML-страниц GitHub читаются неверно** (там относительное время). Для release notes
  пользоваться `api.github.com/repos/<owner>/<repo>/releases` — там `published_at` в ISO. Замечание
  техническое, но при регулярной автосверке критичное.
- **Слоистая архитектура (delivery → usecase → repository → entity) официального источника не имеет.**
  Это внутренняя норма ИВА. Сверять её не с чем, ревизия — только своя.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.
