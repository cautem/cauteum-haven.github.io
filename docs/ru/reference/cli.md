<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# CLI

```text
cautem sandbox create|attach|list|get|stop|start|delete|exec|connect|upload|download|…
cautem cp <local-path> <sandbox>:/path | <sandbox>:/path <local-path>
cautem sandbox template create|list|get|delete
cautem sandbox provider …
cautem provider create|list|get|refresh|update|delete|profile …
cautem profile …
cautem gateway add|select|ensure|info|login|…
cautem policy get|set|…
cautem rule get|approve|reject|…
cautem workspace|forward|service|settings|inference|whoami|logs|term
cautem health|init|doctor|status|version|install|completions|ssh-proxy
```

## Create (compute)

| Флаг | Смысл |
|------|-------|
| `--name` | Имя sandbox |
| `--from` / `--image` | Alias или OCI reference |
| `--build DIR` | Собрать локальный образ из Docker-compatible build context |
| `--dockerfile PATH` | Dockerfile относительно `--build` (по умолчанию `Dockerfile`) |
| `--workspace` | Каталог хоста → `/workspace` |
| `--policy` | Базовый YAML политики |
| `--cpu` / `--memory` / `--pids-limit` | Runtime-лимиты |
| `--template` | Именованный workload template |
| `--provider` | Credential-провайдер (повтор) |
| `--no-proxy` | Без egress sidecar (dev) |
| `--display novnc` | GUI |
| `--gpu` | NVIDIA CDI |

`cautem sandbox create --help` показывает полный список flags, включая
`--remote`, `--gateway`, `--gateway-workspace`, `--env`, `--label`, `--forward`,
`--upload`, `--restart-policy`, `--approval-mode`, `--editor`, параметры display
и явные переключатели hardening/volume/provider. У каждого sandbox leaf есть
собственная справка: `cautem sandbox <leaf> --help`.

`--remote` создаёт sandbox на compute backend выбранного Gateway. Для выбора
workspace укажите `--gateway-workspace NAME`. При создании из образа доступны
CPU, memory, количество GPU, CDI device IDs через `--cdi`, PID limit через
`--pids-limit`, environment, labels и JSON конфигурации driver.
Общий шаблон содержит свои workload-параметры и не принимает inline overrides.
Если передана главная команда без `--detach`, CLI подключится к canonical
процессу и вернёт его exit code. Повторно подключиться можно командой
`cautem sandbox attach NAME`; для процесса, созданного с PTY, укажите `--tty`.

SSH helper из сгенерированных настроек `sandbox ssh-config` поддерживает
текущий Gateway, именованный Gateway и явный URL Gateway с токеном sandbox.
Точные варианты показывает `cautem ssh-proxy --help`.

Локальный `--build DIR` собирает образ через Docker Engine или Podman до
создания sandbox. Файлы build context фильтруются по `.dockerignore`; Dockerfile
должен находиться внутри context. Локальная сборка недоступна с `--remote`.

Добавляйте `--gateway` к команде `sandbox template`, чтобы управлять шаблоном
в workspace выбранного Gateway. Без флага используются локальные шаблоны.

## Подкоманды sandbox

| Команда | Формат |
|---------|--------|
| `sandbox list` | `--limit N`, `--offset N`, `--selector EXPR`, `-o table\|json`, `--ids`, `--names`, `--all-workspaces` |
| `sandbox get` | `NAME` |
| `sandbox start` / `sandbox stop` | Необязательное `NAME` (по умолчанию последний sandbox) |
| `sandbox delete` | `NAME...` либо отдельный `--all` |
| `sandbox upload` | `NAME LOCAL_PATH [SANDBOX_DEST] [--no-git-ignore]` |
| `sandbox download` | `NAME SANDBOX_PATH [LOCAL_DEST]` |
| `sandbox ssh-config` | `NAME [--install]` |
| `sandbox connect` | `[NAME] [--editor vscode\|cursor]` либо `[NAME] -- COMMAND...` |
| `sandbox provider list` | `SANDBOX` |
| `sandbox provider attach` / `detach` | `SANDBOX PROVIDER` |
| `sandbox template create` | `NAME`; image/from/policy, CPU/memory/PIDs/GPU, env, labels, driver config и provider flags |
| `sandbox template list` | Необязательный `--gateway` |
| `sandbox template get` / `delete` | `NAME [--gateway]` |

`--editor` нельзя сочетать с командой. `--no-git-ignore` доступен только для
upload; каталог по умолчанию для upload — `/workspace`, для download — текущий
каталог.

## Команды workspace

| Команда | Формат |
|---------|--------|
| `workspace create` | `--name NAME`, повторяемый `--label KEY=VALUE` |
| `workspace list` | `--page-size N`, `--page-token TOKEN`, `--label-selector KEY=VALUE[,KEY=VALUE...]`, `-o table\|yaml\|json` |
| `workspace get` | `NAME` |
| `workspace delete` | `NAME...` |

Команда списка возвращает одну страницу. Передайте её `next_page_token` в
`--page-token`, чтобы запросить следующую страницу. Условия label selector
объединяются через AND.

## Soft defaults

Если флаги (и template) не задают sizing, config/env могут заполнить пробелы —
жёсткого default memory нет. См. [Ресурсы Docker](../providers/docker/resources.md).
