<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# Переименование Quarshell и план релиза

Название продукта и собственные внутренние имена переходят на **Quarshell**.
На странице совместимости можно сказать, что Quarshell поддерживает OpenShell
v0.1.4; upstream-название остаётся только там, где оно обозначает wire contract,
dependency, attribution, fixtures или границы совместимости.

Эта карта — источник точных имён для миграции. GitHub owner остаётся `cautem`.
Продуктовые repos уже переименованы на месте в `quarshell-*`; смены Go module
path находятся в `main` для core, proto, providers, runtime, driver, proxy,
gateway, SDK, CLI, display и slogx. Старые module tags не изменялись; новые
tags и каскад зависимостей ещё не опубликованы. CLI names, package metadata,
config paths, environment variables и runtime resource identifiers сохраняют
имена Cautem, пока миграция и откат не реализованы и не проверены.

Runtime compatibility gates остаются открыты: текущая gap-проверка сообщает о
12 из 18 требований со статусом open или blocked. Это план миграции и отчёт о
состоянии, а не заявление о полной совместимости или готовности релиза.
Собственные symbols продукта должны использовать Quarshell; OpenShell остаётся
только там, где обозначает wire contract, dependency, attribution, fixtures
или границы совместимости.

## Статус миграции — 2026-10-16

- **GitHub repositories:** продуктовые repos теперь называются `quarshell-*`
  у owner `cautem`. Зарезервированный `cautem/.github` не менялся.
- **Go module declarations:** новые paths находятся в `main` для перечисленных
  выше одиннадцати Go-модулей. Опубликованные tags всё ещё указывают на commits
  со старой module identity, поэтому потребители продолжают использовать старые
  paths.
- **Новые releases:** ещё не опубликованы. Обновить dependency pins и imports в
  порядке зависимостей, затем собрать внешние consumers с `GOWORK=off` до
  публикации каждой группы тегов.
- **Переход продукта:** binary aliases, Python/npm package identity, config и
  data directories, environment variables, images, volumes, networks и labels
  ещё не мигрировали. Сохранить данные и Cautem resources до явной и проверенной
  процедуры переноса.
- **Runtime gates:** evidence Docker, Podman и MicroVM остаётся частичным.
  Gateway сохраняет fail-closed для MicroVM; имеющиеся проверки не позволяют
  заявлять полный parity с OpenShell.

## Карта имён и миграции данных

| Сейчас | Цель | Поведение при миграции |
|---|---|---|
| `cautem/cautem-core`, `cautem/cautem-proto`, `cautem/cautem-driver`, `cautem/cautem-runtime` | `cautem/quarshell-core`, `cautem/quarshell-proto`, `cautem/quarshell-driver`, `cautem/quarshell-runtime` | Repositories и Go module declarations переименованы. Сохранить старые tags; выпустить и проверить новые paths после каскада зависимостей. |
| `cautem/cautem-proxy`, `cautem/cautem-providers`, `cautem/cautem-gateway` | `cautem/quarshell-proxy`, `cautem/quarshell-providers`, `cautem/quarshell-gateway` | Repositories и Go module declarations переименованы. Imports потребителей, image refs и gateway config ещё требуют миграции в порядке зависимостей. |
| `cautem/cautem-sdk`, `cautem/cautem-python` | `cautem/quarshell-sdk`, `cautem/quarshell-python` | Repositories переименованы. Нужно выпустить новые package paths и проверить Go/Python consumers; публикация PyPI остаётся отдельной задачей. |
| `cautem-cli`, `cmd/cautem`, `cautem-console` | `quarshell-cli`, `cmd/quarshell`, browser app в `quarshell-ui` | Оставить `cautem` как warning alias на один release cycle. |
| `cautem/cautem-ui`, `cautem/cautem-display` | `cautem/quarshell-ui`, `cautem/quarshell-display` | Repositories переименованы. Нужно обновить package metadata, ссылки и workflows. |
| `cautem/slogx` (локальный checkout `cautem-slogx`) | `cautem/quarshell-slogx` | Repository и Go module declaration переименованы; новый tag и миграция consumers ещё не готовы. |
| `cautem/sandbox.dev` (локальный checkout `cautem-docs`) | `cautem/quarshell-docs` | Repository переименован; обновить метаданные сайта, ссылки и release-документацию. |
| `cautem/.github` (локальный checkout `cautem-org-profile`) | `cautem/.github` | Оставить специальное имя GitHub profile repo; обновить его содержимое под Quarshell. |
| `cautem.control.v1`, `@cautem/control-client` | `quarshell.control.v1`, `@quarshell/control-client` | Версионировать новый API package. Старый endpoint оставить как adapter на время sunset. |
| Переменные `CAUTEM_*` | `QUARSHELL_*` | Новые имена имеют приоритет. Старое читается, только если нового нет; значение переносится под новым ключом с предупреждением. Конфликтующие значения останавливают запуск. |
| `~/.config/cautem`, `~/.local/share/cautem`, `/etc/cautem` | эквиваленты Quarshell | Скопировать до первой записи, сохранить permissions, не удалять старую директорию автоматически и хранить backup для rollback. |
| `cautem-*` images, volumes, networks | `quarshell-*` | Находить и принимать ресурсы по сохранённым IDs/labels. Не удалять ресурс только из-за старого имени. |
| `cautem.internal/*`, `host.cautem.internal` | `quarshell.internal/*`, `host.quarshell.internal` | При миграции распознавать оба label/alias. Новые ресурсы создавать с Quarshell labels; сохранять старые IDs и endpoints до явного удаления. |
| `cautem` package symbols, help и собственные config keys | Имена Quarshell | Переименовывать по инвентарю владельцев, не глобальной заменой. Generated типы и строки протокола сохранять. |
| OpenShell service/package names и `*.openshell.localhost` | без изменения на compatibility endpoint | Это wire и compatibility identifiers. Собственные API Quarshell используют имена Quarshell. |
| SPDX, copyright, сторонние module names, исторический changelog | сохранить attribution/provenance | Менять только актуальные продуктовые ссылки; не переписывать авторство лицензий и upstream history. |

Приоритет конфигурации: `QUARSHELL_*` → перенесённый Quarshell file → старый
`CAUTEM_*` / legacy file → встроенное значение. Если оба пространства имён задают
разные значения одного параметра, запуск останавливается с именем поля и
источниками; выбирать молча нельзя. Будущий migration tool должен создать
backup и поддерживать dry run, отчёт, apply и rollback. Эти возможности — release
gates; пока они не реализованы и не протестированы, считать их доступными нельзя.

## Последовательность релиза

1. **Baseline и контракт совместимости.** Закрепить OpenShell v0.1.4 tag и
   commit. Проверять proto drift, полный RPC inventory и все известные gaps.
   Оставить `GetSandboxProviderEnvironment` fail-closed до реализации
   endpoint-bound credential path.
2. **Backend SPI.** Оставить переходный adapter в driver module. Переводить
   Docker и Podman независимо; capabilities проверять до durable write. Сначала
   добавить backend contract tests, затем переключать Gateway dispatch.
3. **Docker и Podman acceptance.** Закрыть lifecycle, egress, ownership,
   transfer, TTY, daemon recovery и published-artifact gates на поддерживаемых
   хостах. Вести отдельные evidence для Docker, Podman rootful и rootless.
4. **MicroVM acceptance.** Завершить rootfs import/digest binding, host
   isolation, guest protocol, policy broker, recovery и target-host tests.
   Убирать Gateway fail-closed только после прохождения всех обязательных gates.
5. **Совместимые релизы по текущим путям.** Core/proto →
   runtime/proxy/providers → driver → gateway → SDKs → CLI/UI. До смены module
   path прогнать published consumer gate с `GOWORK=off`.
6. **Завершить релизы Quarshell modules.** Repositories и одиннадцать Go module
   declarations уже переименованы. Мигрировать межмодульные imports, workspace
   pins, workflow checkouts, package metadata и artifacts в порядке
   зависимостей. Публиковать неизменяемые `github.com/cautem/quarshell-*` tags
   с SBOM и подписями только после тестов каждого модуля и consumer build без
   локальных замен. Старые tags не перемещать и не переписывать.
7. **Migration candidate.** Выпустить migration tool для binary/config/resources
   и release candidate Cautem-to-Quarshell. Проверить свежую установку, upgrade,
   старый alias, конфликт config, adoption ресурсов и rollback с последнего
   Cautem release.
8. **Документация и заключительные gates.** Обновить EN/RU install, quickstart,
   architecture, security, backend matrix, config reference, CLI help и
   completions, SDK examples и compatibility statement. Запустить проверки
   ссылок, strict docs build, полный test matrix, vulnerability scans и
   published-artifact smoke.
9. **Релиз.** Публиковать каждый module после собственных gates и downstream
   consumer build. Указать точную версию OpenShell и оставшиеся ограничения.
   Нельзя заявлять полный parity с незакрытыми методами, режимами или security gates.

## Проверки переключения и отката

| Проверка | Условие успеха |
|---|---|
| Config migration | Новые значения имеют приоритет, одинаковые старые мигрируют, конфликт останавливает запуск с понятной ошибкой. |
| CLI transition | Работает `quarshell`; `cautem` передаёт аргументы и environment, показывает warning и сохраняет exit code. |
| Существующие ресурсы | Старые IDs, labels, volumes и networks находятся. Adoption явный, cleanup проверяет сохранённое владение. |
| Новые consumers | Go, Python, npm и binary consumers разрешают опубликованные Quarshell artifacts без локальных replacements. |
| Rollback | Восстановить старый config из backup, запустить старый binary/dependencies, подключиться к ресурсам и сохранить gateway state. |
| Заявление совместимости | В нём указаны версия, проверенные поверхности, backends и открытые gaps. |

Repositories и Go module declarations уже переведены на новые имена у owner
`cautem`; репозитории переименованы на месте, без переноса в другую организацию.
Profile repo сохраняет зарезервированное имя `.github`. Новые module/package
releases и миграция данных требуют прохождения перечисленных gates.
