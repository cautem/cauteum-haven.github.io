---
status: soon
---

<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# MicroVM

<div class="ws-soon" markdown>

![В разработке](../../assets/images/under-construction.svg){ width="420" }

**Экспериментальный режим.** Локальный драйвер работает, но Gateway пока его не включает.

</div>

## Что уже работает

Драйвер запускает отдельную libkrun VM для каждой песочницы. Linux rootfs можно
импортировать из OCI registry reference или staged tar-архива; драйвер создаёт
неизменяемый ext4 base image и writable overlay для каждой песочницы. Для
registry используются Docker/Podman credentials процесса драйвера. Выбирается
платформа `linux/<архитектура хоста>`; перед кешированием проверяются manifest
и layer digests.

Также есть аутентифицированный guest control channel с versioned v1 JSON и
length-prefixed frames; host и guest отклоняют пропущенные и неизвестные версии.
`start-workload` возвращает успех только после того, как guest agent прочитал
закрытый versioned readiness marker, созданный supervisor после запуска workload.
Путь к marker удаляется из environment дочернего процесса. Также доступны
перезапуск процесса, exec, logs, file copy, policy proxy и SSH relay. Host
вращает per-sandbox token при успешном Start и recovery; pending rotation
хранится в закрытом каталоге sandbox и повторяется с тем же rotation ID после
потери ответа.
Exec поддерживает одноразовые команды и full-duplex PTY с stdin, resize окна и
terminal-generated signals. У Gateway compatibility adapter есть отдельный VM
interactive-exec path; создание sandbox остаётся fail-closed до прохождения
workspace и target-host gates.

Образы
хранятся в кеше внутри каталога состояния VM. Квота по умолчанию — 64 GiB;
неиспользуемые образы удаляются, когда требуется место.

## Готовность к production

Linux worker запускается в собственных user и mount namespaces и делает
распространение mounts приватным до создания broker и VM. Запуск завершается
ошибкой, если namespaces недоступны или Gateway запущен от host root. Namespace
root worker пока отображается на account Gateway, это не отдельный host UID.
macOS запускает отдельный worker process и отклоняет host mounts, но этот драйвер
пока не создаёт эквивалентный host mount namespace. Workspace mounts ещё не
передаются ни одной из платформ.

Gateway отклоняет создание VM, пока разрешённые host mounts не будут открываться
и передаваться через file descriptors от отдельного host identity, не будут
приняты UID/GID mapping и persistence и не пройдут Gateway lanes для Linux KVM и
macOS ARM64.

Поля OCI image `Entrypoint`, `Cmd`, `Env`, `User` и `WorkingDir` применяются к
in-memory VM workload spec. При Gateway recovery image config загружается по
закреплённому platform manifest digest. Environment и user не сохраняются в
durable state, reserved identity variables фильтруются, root identity отклоняется.
Unit tests и macOS ARM64 live acceptance проверяют OCI config, credential-helper auth
приватного registry, восстановление по digest, одноразовый TTY exec и наследование
image env/workdir. Та же live проверка подтверждает PTY resize, stdin и Ctrl-C.
Gateway→VM и Linux KVM acceptance, отдельный host UID, workspace mounts/ownership,
GPU passthrough и PID limits остаются открытыми. Эти пробелы перечислены в
[матрице runtime acceptance](../reference/runtime-acceptance.md).

## Настройка хоста

Соберите worker, статический Linux guest agent и runtime helpers командой
`cautem-driver/scripts/build-vm-runtime.sh`. Укажите `CAUTEM_VM_WORKER`,
`CAUTEM_VM_GUEST_AGENT`, `CAUTEM_VM_RUNTIME_DIR` и `CAUTEM_VM_STATE_DIR`. Linux
требует чтение и запись в `/dev/kvm`; macOS поддерживается на Apple Silicon с
Hypervisor.framework и подписанным worker bundle. Для сборки rootfs нужен
e2fsprogs.

`CAUTEM_VM_IMAGE_CACHE_MAX_BYTES` меняет квоту кеша с 64 GiB по умолчанию.
Credentials registry берутся из Docker или Podman credential files,
доступных процессу драйвера. Загруженный rootfs tar ограничен 10 GiB в сжатом
виде и 32 GiB после распаковки.

Инструкции live acceptance драйвера находятся в разделе MicroVM в
[README драйвера](https://github.com/cautem/cautem-driver#microvm-status).
Gateway и Linux KVM acceptance остаются обязательными gates релиза.
