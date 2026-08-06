---
title: Best practices — Java-бэкенд (источники)
type: note
status: current
created: 2026-08-03
updated: 2026-08-03
tags: [board, bestpractices, java, quarkus]
permalink: tacticum/00-board/bp-java
---

# Java-бэкенд — официальные источники best practices

> **Коротко:** в профиле версия языка не зафиксирована — манифест `iva-java-development-base` требует только `java`, ветки названы словами (MCU: CXF + checkstyle; Quarkus/disk-storage: CDI + Mutiny); актуальное на 2026-08-03 — JDK 25 LTS (GA 16.09.2025, поддержка до сентября 2028) при не-LTS JDK 26 и 27 на подходе, для Quarkus-ветки — Quarkus 3.33 LTS (до 25.03.2027, минимум для всей 3.x — JDK 17), для MCU — CXF 4.2.x и Checkstyle 13.x (требует JRE 21+); всё названное в профиле поддерживается, но риск не в версиях, а в удалённом: Jakarta EE 11 выкинул SOAP with Attachments и XML Binding — это прямо по SOAP-части MCU.

## Стек по профилю

Дословно из quickstart-а профиля `iva-role-java` (`TacticumApps/dev`, `docs/user_manuals/iva-role-java-profile-quickstart.md`): «стек Java, две ветки: MCU (signalling, checkstyle, CXF) и Quarkus/disk-storage (CDI, Mutiny, suite-дисциплина тестов, хрупкие зоны storage/NATS/cluster и media)». Манифест `templates/iva-java-development-base/manifest.yaml` требует `java`.

Отсюда два контура источников:

- **MCU** — классический Java-сервис: JDK + Apache CXF (JAX-WS/JAX-RS) + checkstyle как формальный гейт стиля. Всё «платформенное» (JDK, javac, линтеры, JUnit, Maven) применимо здесь в полном объёме.
- **Quarkus / disk-storage** — Quarkus + CDI (Jakarta) + Mutiny (реактивщина поверх Vert.x), NATS, кластер, медиа. Сюда добавляются quarkus.io, smallrye.io и спецификации Jakarta EE.

Общая база на 2026-08-03: **JDK 25 LTS**, **Quarkus 3.33 LTS** (минимум для 3.x — JDK 17), **Maven 3.9.x** (4.0 всё ещё RC), **JUnit 6.x**, **Checkstyle 13.x** (требует JRE 21+).

## Источники

| Источник | Тип | Чей | Что именно берём | Применимо к | Как часто обновляется | Сверено |
|---|---|---|---|---|---|---|
| [dev.java/learn](https://dev.java/learn/) | доки/туториалы | Oracle (офиц. сайт Java) | Канонические туториалы по языку и API: pattern matching, records, Stream API, **Virtual Threads**, FFM API, коллекции, инструменты JVM. Основной источник «как писать на современной Java» | обоим | постоянно, привязано к текущему JDK | 2026-08-03 |
| [openjdk.org/projects/jdk/25](https://openjdk.org/projects/jdk/25/) | release page | OpenJDK | Полный список JEP-ов JDK 25 LTS + подтверждение статуса LTS. GA 16.09.2025 | обоим | раз в релиз | 2026-08-03 |
| [openjdk.org/projects/jdk/26](https://openjdk.org/projects/jdk/26/) | release page | OpenJDK | Список JEP-ов JDK 26. GA 17.03.2026 | обоим | раз в релиз | 2026-08-03 |
| [openjdk.org/projects/jdk/27](https://openjdk.org/projects/jdk/27/) | release page | OpenJDK | Что едет в JDK 27 (GA 15.09.2026): G1 по умолчанию везде, post-quantum TLS 1.3, Lazy Constants (3-й preview) | обоим | в процессе, до GA | 2026-08-03 |
| [openjdk.org/jeps/0](https://openjdk.org/jeps/0) | индекс JEP | OpenJDK | Полный индекс JEP по состояниям (Draft / Candidate / Integrated / Delivered) и по релизам. **Главный источник «куда едет язык»** | обоим | непрерывно | 2026-08-03 |
| [openjdk.org/jeps/1](https://openjdk.org/jeps/1) | процесс | OpenJDK | Сам JEP-процесс: как фича проходит путь preview → final. Нужно, чтобы отличать preview от стабильного и не тащить preview в прод | обоим | редко | 2026-08-03 |
| [Consolidated JDK 26 Release Notes](https://www.oracle.com/java/technologies/javase/26all-relnotes.html) · [26 relnote issues](https://www.oracle.com/java/technologies/javase/26-relnote-issues.html) | release notes | Oracle | Removed / Deprecated APIs, изменения поведения. Единственное место, где полный список удалённого | обоим | раз в релиз + CPU | 2026-08-03 |
| [Consolidated JDK 25 Release Notes](https://www.oracle.com/java/technologies/javase/25all-relnotes.html) | release notes | Oracle | То же для текущей LTS — базовая точка отсчёта для рабочего рантайма | обоим | раз в релиз + CPU | 2026-08-03 |
| [JDK 25 Migration Guide](https://docs.oracle.com/en/java/javase/25/migrate/index.html) | migration guide | Oracle | Официальный гайд миграции на 25 — что ломается при переезде с 17/21 | обоим | раз в LTS | 2026-08-03 |
| [javac tool spec (25)](https://docs.oracle.com/en/java/javase/25/docs/specs/man/javac.html) · [(26)](https://docs.oracle.com/en/java/javase/26/docs/specs/man/javac.html) | reference | Oracle | Полный список `-Xlint` категорий. **Встроенные предупреждения компилятора — самая дешёвая практика, выраженная машиной**; `-Xlint:all -Werror` в CI закрывает целый класс замечаний до линтера | обоим | раз в релиз | 2026-08-03 |
| [inside.java](https://inside.java/) · [Newscast](https://inside.java/newscast) | блог/видео | команда Java в Oracle | Разборы фич от авторов, Quality Outreach (предупреждения об ломающих изменениях в EA-сборках), «почему сделали именно так». Тут раньше всего видно смену рекомендации | обоим | несколько постов в неделю | 2026-08-03 |
| [OpenJDK Quality Group](https://openjdk.org/groups/quality/) | процесс | OpenJDK | Quality Outreach — программа раннего оповещения о поломках в библиотеках на новых JDK | обоим | по мере релизов | 2026-08-03 |
| [quarkus.io/guides](https://quarkus.io/guides/) | доки | Quarkus (Red Hat) | Корень всей документации. Разбита на concepts / how-to / tutorials / references — рекомендации живут в concepts и how-to | Quarkus | непрерывно, по версиям | 2026-08-03 |
| [quarkus.io/releases](https://quarkus.io/releases/) | release policy | Quarkus | Какие версии LTS и до какой даты поддерживаются. **Первое, что сверять** перед решением об апгрейде | Quarkus | раз в релиз | 2026-08-03 |
| [quarkus.io/blog](https://quarkus.io/blog/) · [тег lts](https://quarkus.io/blog/tag/lts/) | блог релизов | Quarkus | Анонсы релизов и LTS, объяснение смены дефолтов | Quarkus | ~ежемесячно | 2026-08-03 |
| [Migration Guides (wiki)](https://github.com/quarkusio/quarkus/wiki/Migration-Guides) | migration guides | Quarkus | Гайд на каждый минорный релиз от 1.1 до 4.0. **Именно здесь пишут «этот способ больше не рекомендуется»** — самый плотный источник изменений практик | Quarkus | каждый релиз | 2026-08-03 |
| [Migration Guide 4.0](https://github.com/quarkusio/quarkus/wiki/Migration-Guide-4.0) | migration guide | Quarkus | Ломающие изменения будущей мажорной: Jackson 2 → 3 (переезд пакетов в `tools.jackson.*`), Jakarta REST 4, Jakarta Data | Quarkus | пополняется до GA | 2026-08-03 |
| [The Road to Quarkus 4 (discussion #52020)](https://github.com/quarkusio/quarkus/discussions/52020) | roadmap | Quarkus | Официальная дорожная карта: беты сен–окт 2026, GA ноябрь 2026, LTS март 2027; база Java 21/25, Jakarta EE 11, JPMS-архитектура, Vert.x 5, Netty 4.2. 3.x после 4.0 — только maintenance ~год | Quarkus | по мере движения | 2026-08-03 |
| [Virtual Threads guide](https://quarkus.io/guides/virtual-threads) | how-to | Quarkus | `@RunOnVirtualThread`: когда виртуальные потоки вместо Mutiny, а когда нет. Разбор pinning (до Java 24 — `synchronized`, после — только натив), монополизации карьера, деградации пулов на `ThreadLocal` (Jackson, Netty). **Ключевой документ для решения «реактивно или блокирующе»** | Quarkus | по релизам | 2026-08-03 |
| [Quarkus Reactive Architecture](https://quarkus.io/guides/quarkus-reactive-architecture) | concept | Quarkus | Модель исполнения: event loop, worker threads, где нельзя блокировать | Quarkus | по релизам | 2026-08-03 |
| [Mutiny primer](https://quarkus.io/guides/mutiny-primer) | concept | Quarkus | Введение в Mutiny в контексте Quarkus | Quarkus | по релизам | 2026-08-03 |
| [Duplicated context / context propagation](https://quarkus.io/guides/duplicated-context) | concept | Quarkus | Как переносить контекст через асинхронные границы — типовой источник багов в реактивном коде | Quarkus | по релизам | 2026-08-03 |
| [CDI Reference](https://quarkus.io/guides/cdi-reference) | reference | Quarkus | Как ArC (реализация CDI в Quarkus) отличается от полного CDI: что работает в build-time, что не поддерживается | Quarkus | по релизам | 2026-08-03 |
| [Testing your application](https://quarkus.io/guides/getting-started-testing) · [Continuous Testing](https://quarkus.io/guides/continuous-testing) | how-to | Quarkus | `@QuarkusTest` vs `@QuarkusIntegrationTest`, тестовые профили и ресурсы, непрерывное тестирование. **База для suite-дисциплины** | Quarkus | по релизам | 2026-08-03 |
| [Measuring Performance](https://quarkus.io/guides/performance-measure) | how-to | Quarkus | Как корректно мерить время старта и RSS — вместо «на глаз» | Quarkus | по релизам | 2026-08-03 |
| [Tips for writing native applications](https://quarkus.io/guides/writing-native-applications-tips) | how-to | Quarkus | Ограничения native-image: reflection, ресурсы, динамические прокси | Quarkus | по релизам | 2026-08-03 |
| [Configuration Reference](https://quarkus.io/guides/config-reference) | reference | Quarkus | SmallRye Config / MicroProfile Config: приоритеты источников, профили, `@ConfigMapping` | Quarkus | по релизам | 2026-08-03 |
| [quarkus.io/security](https://quarkus.io/security/) | security | Quarkus | CVE и какие ветки получают патчи | Quarkus | по мере CVE | 2026-08-03 |
| [SmallRye Mutiny docs](https://smallrye.io/smallrye-mutiny/latest/) | доки | SmallRye | Каноническая документация Mutiny (текущая — 3.3.0). Tutorials / guides / reference | Quarkus | по релизам | 2026-08-03 |
| [Going reactive: a few pitfalls](https://smallrye.io/smallrye-mutiny/latest/reference/going-reactive-a-few-pitfalls/) | reference | SmallRye | Прямой список антипаттернов реактивного кода от авторов библиотеки. **Самый концентрированный «как не надо» по Mutiny** | Quarkus | редко | 2026-08-03 |
| [Uni and Multi](https://smallrye.io/smallrye-mutiny/latest/reference/uni-and-multi/) | reference | SmallRye | Когда `Uni`, когда `Multi` — базовое решение, которое агенты чаще всего принимают неправильно | Quarkus | редко | 2026-08-03 |
| [Testing Mutiny code](https://smallrye.io/smallrye-mutiny/latest/guides/testing/) | guide | SmallRye | `UniAssertSubscriber` / `AssertSubscriber` — как тестировать асинхронный код без `Thread.sleep` | Quarkus | редко | 2026-08-03 |
| [Migrating to Mutiny 2](https://smallrye.io/smallrye-mutiny/latest/reference/migrating-to-mutiny-2/) | migration | SmallRye | Что менялось между мажорами Mutiny (в Quarkus 3.33 уже ядро Mutiny 3.1.1) | Quarkus | редко | 2026-08-03 |
| [Jakarta CDI](https://jakarta.ee/specifications/cdi/) · [CDI 4.1](https://jakarta.ee/specifications/cdi/4.1/) | спецификация | Eclipse Foundation | Нормативный текст CDI 4.1 (Jakarta EE 11). Источник истины по скоупам, событиям, интерцепторам — выше любого гайда | Quarkus | раз в мажор EE | 2026-08-03 |
| [Jakarta EE 11 Release](https://jakarta.ee/release/11/) | release | Eclipse Foundation | Что нового и что удалено в EE 11: Jakarta Data 1.0, Persistence 3.2 (records), Concurrency 3.1 (virtual threads), удалены Managed Beans, SOAP with Attachments и XML Binding | обоим | раз в мажор | 2026-08-03 |
| [Jakarta RESTful WS 4.0](https://jakarta.ee/specifications/restful-ws/4.0/) | спецификация | Eclipse Foundation | Нормативный REST-API для EE 11 — база для JAX-RS в CXF 4.2 и в Quarkus 4 | обоим | раз в мажор | 2026-08-03 |
| [cxf.apache.org](https://cxf.apache.org/) | release/новости | Apache CXF | Текущие версии и живые ветки. На 2026-08-03: 4.2.2 (10.06.2026), 4.1.7, 4.0.11, 3.6.11 (20.05.2026); 3.5.x закрыта | MCU | по релизам | 2026-08-03 |
| [CXF User Guide](https://cxf.apache.org/docs/index.html) | доки | Apache CXF | JAX-WS и JAX-RS фронтенды, Bus configuration, транспорты (HTTP/2, JMS, WebSocket, SSE), WS-Security, DataBindings, инструменты кодогенерации из WSDL | MCU | по релизам | 2026-08-03 |
| [CXF Security Advisories](https://cxf.apache.org/security-advisories.html) | security | Apache CXF | CVE по веткам — обязательный вход при выборе версии | MCU | по мере CVE | 2026-08-03 |
| [checkstyle.org](https://checkstyle.org/) · [Release Notes](https://checkstyle.org/releasenotes.html) | линтер | Checkstyle | Текущая версия 13.9.0 (27.07.2026), требует JRE 21+. Ноты — источник новых проверок | MCU (гейт есть в ветке), полезно обоим | ~раз в месяц | 2026-08-03 |
| [Style Configurations](https://checkstyle.org/style_configs.html) · [Google's Style](https://checkstyle.org/google_style.html) · [Sun's Style](https://checkstyle.org/sun_style.html) | правила | Checkstyle | Три эталонных набора в поставке: Sun, Google, **OpenJDK**. Базу проекта берём из них, а не сочиняем | MCU | вместе с релизами | 2026-08-03 |
| [Checks reference](https://checkstyle.org/checks.html) | reference | Checkstyle | Полный каталог проверок по категориям — сверять, не отключён ли нужный | MCU | вместе с релизами | 2026-08-03 |
| [errorprone.info](https://errorprone.info/) · [Bug Patterns](https://errorprone.info/bugpatterns) | линтер | Google | Плагин к javac, ловит ошибки времени компиляции, которых не видит javac. Текущая 2.50.0 (10.06.2026). Каталог bug patterns читается как список антипаттернов | обоим | ~раз в квартал | 2026-08-03 |
| [SpotBugs docs](https://spotbugs.readthedocs.io/en/stable/) · [Bug Descriptions](https://spotbugs.readthedocs.io/en/latest/bugDescriptions.html) | линтер | SpotBugs | Байткод-анализ (наследник FindBugs). Текущая 4.10.3 (12.07.2026). Bug Descriptions — каталог дефектов с объяснением | обоим | ~раз в квартал | 2026-08-03 |
| [JUnit User Guide](https://docs.junit.org/current/user-guide/) · [Release Notes](https://docs.junit.org/current/release-notes/) | доки | JUnit Team | Каноника JUnit 5/6: жизненный цикл, параметризация, расширения, теги и suite. Текущая 6.1.2 (12.07.2026) | обоим | по релизам | 2026-08-03 |
| [Maven Releases History](https://maven.apache.org/docs/history.html) · [What's new in Maven 4](https://maven.apache.org/whatsnewinmaven4.html) | доки | Apache Maven | История версий и что меняет Maven 4 (рантайм Java 17, upgrade-tool для pom.xml). **На 2026-08-03 Maven 4.0.0 всё ещё RC (rc-5 от 13.11.2025), в проде — 3.9.x (последняя 3.9.16 от 13.05.2026)** | обоим | по релизам | 2026-08-03 |

Итого **44 ссылки**, все открыты и сверены 2026-08-03.

## Что изменилось за последний год

### Обеим веткам

1. **JDK 25 — текущая LTS (GA 16.09.2025), поддержка до сентября 2028.** Финализированы и стали рекомендованным дефолтом: **Scoped Values** (JEP 506) — замена `ThreadLocal` для передачи контекста, особенно в связке с виртуальными потоками; **Module Import Declarations** (JEP 511); **Compact Source Files and Instance Main Methods** (JEP 512); **Flexible Constructor Bodies** (JEP 513) — код до `super()`; **Key Derivation Function API** (JEP 510); **Compact Object Headers** (JEP 519) — теперь стабильная фича, ощутимая экономия heap. Плюс **AOT Command-Line Ergonomics** (514) и **AOT Method Profiling** (515) — ускорение старта, и **Generational Shenandoah** (521).

   Остаются **preview / incubator, в прод не тащить**: Structured Concurrency (пятый preview), Primitive Types in Patterns (третий preview), Stable Values, PEM Encodings, Vector API (десятый инкубатор).

2. **JDK 26 (GA 17.03.2026) — не LTS.** Главное для нас — не фичи, а **ломающие изменения**:
   - **`sun.misc.Unsafe`: memory-access методы теперь бросают исключение по умолчанию** (дорожная карта JEP 471/498; в JDK 26 дефолтом стало `--sun-misc-unsafe-memory-access=deny`, отдельного JEP на этот шаг нет). Это бьёт по низкоуровневым библиотекам — Netty и всё, что вокруг него. Для ветки Quarkus/NATS/media это первое, что ломается при попытке запуститься на 26.
   - **JEP 500 «Prepare to Make Final Mean Final»** — предупреждения на мутацию `final`-полей через глубокую рефлексию. Готовят запрет. Затрагивает старые фреймворки инъекции и самописные хаки в тестах.
   - **Удалено:** Applet API (JEP 504), `Thread.stop()` (перестал компилироваться), поддержка InfiniBand SDP, XML-интерчейндж в `DescriptorSupport`, модуль `jdk.jsobject`, инструмент `jrunscript`, алгоритмы DESede/PKCS1Padding из требований. `SQLPermission` помечен deprecated for removal.
   - Полезное: **HTTP/3 в HttpClient** (JEP 517), AOT-кеширование объектов с любым GC (516), улучшения throughput G1 (522).

3. **JDK 27 (GA 15.09.2026, не LTS) — уже видно, что придёт:** G1 становится GC по умолчанию во всех окружениях (JEP 523), post-quantum гибридный обмен ключами для TLS 1.3 (JEP 527). Практический вывод: явные флаги выбора GC в конфигурации сервисов стоит пересмотреть заранее.

4. **Jakarta EE 11**: база Java SE 17+, с отдельными возможностями на 21+. CDI 4.1, Jakarta REST 4.0, Persistence 3.2 (поддержка records), Validation 3.1 (records), Concurrency 3.1 (виртуальные потоки), новая спека **Jakarta Data 1.0**. **Удалены**: Managed Beans, все ссылки на SecurityManager, все optional-спеки, а также **Jakarta SOAP with Attachments и Jakarta XML Binding** — прямо касается SOAP-части MCU.

5. **Инструменты подтянули базовую Java:** Checkstyle 13.x требует **JRE 21+** (было ниже) — это ломающее изменение января 2026 для CI-образов ветки MCU. Maven 4 требует Java 17 в рантайме, но **до GA не дошёл** — на 03.08.2026 актуален rc-5 (13.11.2025), в проде остаётся 3.9.x. JUnit ушёл в 6.x.

6. **Виртуальные потоки перестали быть «новинкой» и стали дефолтным ответом для I/O-нагрузки** — при условии Java 21+. Важный сдвиг: до Java 24 блокирующая операция внутри `synchronized` пинила карьер-поток, с Java 24 (JEP 491) — уже нет. Значит рекомендация «замените `synchronized` на `ReentrantLock` ради виртуальных потоков» на JDK 25 **устарела** и её больше не надо тиражировать.

### Только Quarkus / disk-storage

7. **Сменилась LTS: Quarkus 3.33 (25.03.2026, поддержка до 25.03.2027)** — это то, на что надо целиться. Предыдущая **3.27 LTS** (24.09.2025) поддерживается до 24.09.2026, **3.20 LTS умерла 28.03.2026**. 3.33 — прямое продолжение 3.32, миграционный гайд пустой, апгрейд с 3.27 механический (`quarkus update`). Текущий feature-релиз — 3.38 (29.07.2026).

8. **Java 25 поддерживается Quarkus «из коробки» с 3.31.** В 3.30 генерация AOT-кеша перешла на упрощённый workflow JEP 514 и **эта фича теперь требует Java 25**. Минимум для всей ветки 3.x при этом остаётся **JDK 17**.

9. **Mutiny переехал на 3.x внутри 3.x-линейки Quarkus** — в BOM Quarkus 3.33.2 ядро `io.smallrye.reactive:mutiny` версии 3.1.1, документация smallrye — 3.3.0. Формального «Migrating to Mutiny 3» на сайте нет (есть только гайд по переезду на 2) — это дыра, см. ниже.

10. **Quarkus 4.0 — на горизонте ноября 2026, и он ломающий.** Беты сентябрь–октябрь, GA ноябрь 2026, LTS 4.x — март 2027. База Java 21/25 (Java 17, скорее всего, отвалится), полное выравнивание по Jakarta EE 11, JPMS-архитектура, Vert.x 5, Netty 4.2 и **Jackson 2 → 3 с переездом пакетов**: `com.fasterxml.jackson.core` → `tools.jackson.core`, `com.fasterxml.jackson.databind` → `tools.jackson.databind` (аннотации остаются на месте). Ветка 3.x после выхода 4.0 живёт ещё примерно год, но только на исправлениях. Для агентных профилей это значит: код, который пишется сейчас, лучше писать так, чтобы импорты Jackson были локализованы.

### Только MCU

11. **Apache CXF 4.2 принёс поддержку Jakarta EE 11** (4.2.2 от 10.06.2026). Напоминание по базовым линиям: 4.1.0 был первым релизом с Jakarta EE 10 и базой JDK 17. Ветка 3.5.x закрыта окончательно (3.5.11 — последний релиз), живут 4.2.x / 4.1.x / 4.0.x / 3.6.x. Если MCU сидит на 3.x — это уже долг, а не выбор.

12. **Checkstyle: помимо перехода на JRE 21+, за год добавились новые проверки в эталонные наборы** — в google_checks приехали `WhitespaceBeforeEmptyBody` и `TextBlockGoogleStyleFormattingCheck`, в sun_checks — проверки на «Beginning Comments» и убраны дублирующиеся срабатывания `WhitespaceAfter`/`WhitespaceAround`. Проект, который скопировал конфиг год назад и заморозил, уже отстал от эталона.

## Что переписать в профиле прямо сейчас

- **(обе ветки) Убрать совет «замените `synchronized` на `ReentrantLock` ради виртуальных потоков»** — с Java 24 блокировка внутри `synchronized` больше не пинит карьер-поток (JEP 491), на JDK 25 рекомендация устарела и тиражировать её нельзя.
- **(обе ветки) Заменить `ThreadLocal` как способ протаскивания контекста на Scoped Values** — JEP 506 финализирован в JDK 25 и это рекомендованный дефолт, особенно в связке с виртуальными потоками; в Quarkus-ветке отдельно оговорить, что `ThreadLocal` в библиотеках (Jackson, Netty) деградирует пулы под `@RunOnVirtualThread`.
- **(MCU) Вычеркнуть Jakarta XML Binding и SOAP with Attachments из «идёт с платформой»** — в Jakarta EE 11 обе спеки удалены; для SOAP-части MCU их надо тянуть явными зависимостями и фиксировать это в тексте профиля, иначе сборка на EE 11 просто не соберётся.
- **(MCU) Зафиксировать живые ветки CXF вместо безверсионного «CXF»** — 4.2.x (первая с поддержкой Jakarta EE 11) / 4.1.x / 4.0.x / 3.6.x; 3.5.x закрыта окончательно, и сидение на 3.x надо называть долгом, а не выбором (cxf.apache.org, ветки на 2026-08-03).
- **(MCU) Поднять требование к CI-образу до JRE 21+ и перечитать эталонный конфиг checkstyle** — Checkstyle 13.x не запустится ниже 21, а в google_checks за год приехали `WhitespaceBeforeEmptyBody` и `TextBlockGoogleStyleFormattingCheck`: замороженная год назад копия конфига уже разошлась с эталоном.
- **(Quarkus) Целиться на 3.33 LTS и локализовать импорты Jackson** — 3.20 LTS умерла 28.03.2026, 3.27 доживает до 24.09.2026, апгрейд на 3.33 механический (`quarkus update`); в Quarkus 4.0 (GA ноябрь 2026) пакеты Jackson переезжают `com.fasterxml.jackson.*` → `tools.jackson.*`, поэтому код лучше писать с изолированными импортами уже сейчас (Migration Guide 4.0).

## Чего не нашёл / где источник слабый

- **Нет официального документа «Quarkus best practices».** quarkus.io так свой раздел не называет: рекомендации размазаны по guides категории *concepts* и *how-to* (реактивная архитектура, виртуальные потоки, duplicated context) и по **migration guides**. Для регулярной сверки практик самый плотный вход — именно migration guides, а не документация: там прямым текстом пишут, что перестало быть рекомендованным.
- **Нет «Migrating to Mutiny 3».** Есть только `reference/migrating-to-mutiny-2/`, при том что в Quarkus 3.33 уже ядро Mutiny 3.1.1. Что менялось между Mutiny 2 и 3 — по официальным страницам восстановить не удалось; придётся идти в GitHub releases smallrye/smallrye-mutiny, а это уже release notes, не гайд.
- **Не удалось подтвердить по официальной странице, какая версия будет следующей LTS после JDK 25.** На странице проекта JDK 27 пометки LTS нет (в отличие от JDK 25, где она есть явно). По двухлетнему циклу 17 → 21 → 25 следующая должна быть JDK 29 (сентябрь 2027), но **это вывод из закономерности, а не цитата** — Oracle Java SE Support Roadmap машинно прочитать не получилось.
- **Apache CXF документирован заметно хуже остальных.** User Guide — вики-страницы без версионирования и дат правки; понять, какая рекомендация к какой ветке относится, часто невозможно. «Что изменилось» по CXF приходится собирать из новостной ленты на главной, а не из внятного migration guide.
- **Расширение Quarkus CXF (`quarkus-cxf`) живёт в Quarkiverse — это community, а не Quarkus core.** Если MCU и Quarkus-ветка когда-нибудь сойдутся, источник для стыка будет не такого же уровня официальности, как остальные в таблице. В список намеренно не включил.
- **Официальной страницы «правила javac по умолчанию» не существует** — есть только tool spec с перечислением категорий `-Xlint`. Практику «включить `-Xlint:all -Werror`» ни один официальный документ прямо не рекомендует; это вывод, а не цитата, и в скилле его надо помечать как соглашение проекта.
- **Error Prone и SpotBugs — де-факто стандарты, но не «официальные» в смысле стека:** это Google и независимое сообщество, не OpenJDK и не Red Hat. Как источник best practices их каталоги отличные, но статус у них ниже, чем у dev.java или quarkus.io, и в вики это стоит оговорить.

## Владелец ревью

| Роль | Кто | Дата сверки |
|---|---|---|
| Владелец стека | _не назначен — впишите себя_ | 2026-08-03 |

Правьте эту страницу под свой стек: вычёркивайте источники, которыми не пользуетесь, дописывайте свои, поправьте описание стека, если оно разошлось с реальностью.
