<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# Создание sandbox (Docker)

## Минимальный create

```bash
cautem sandbox create \
  --name demo \
  --workspace "$PWD" \
  --policy cautem-cli/policies/default.yaml \
  --memory 2g
```

С образом агента и credential-провайдерами:

```bash
cautem sandbox create \
  --name cursor \
  --from cursor \
  --workspace "$PWD" \
  --policy /path/to/policy.yaml \
  --provider cursor \
  --provider gh \
  --memory 2g
```

## Жизненный цикл

```bash
cautem sandbox list
cautem sandbox status demo
cautem sandbox exec demo -- uname -a
cautem sandbox connect demo          # интерактивный bash -il
cautem sandbox stop demo
cautem sandbox start demo
cautem sandbox delete demo
```

Delete удаляет sandbox, proxy sidecar, сеть и помеченные volumes.

## Что делает create

1. Резолвит образ (`--image` / `--from` / default).
2. Подтягивает образы Engine (sandbox + slim proxy).
3. Создаёт `cautem-net-<name>` (internal при включённом proxy).
4. Стартует `cautem-proxy-<name>` из `debian:bookworm-slim`
   (`CAUTEM_PROXY_IMAGE` для override).
5. Создаёт и стартует `cautem-<name>` с политикой, bind workspace,
   опциональным data volume, лимитами CPU/memory/PIDs.
6. Регистрирует sandbox в gateway при наличии.

## Шаблоны

Фиксируйте sizing в template (как в OpenShell):

```bash
cautem sandbox template create \
  --name desk \
  --from cursor \
  --memory 2g \
  --cpu 2 \
  --pids-limit 2048

cautem sandbox create --template desk --name worker --workspace "$PWD"
```

Флаги перекрывают поля template. Soft defaults из config/env заполняют только
пустые поля — см. [Ресурсы](./resources.md).

По умолчанию шаблоны хранятся локально в профиле CLI. Флаг `--gateway`
управляет шаблоном в workspace выбранного Gateway:

```bash
cautem sandbox template create shared-agent --gateway \
  --image ghcr.io/example/agent:stable --cpu 500m --memory 2Gi \
  --gpu 1 --env MODE=fast --label team=runtime
cautem sandbox template list --gateway
cautem sandbox template get shared-agent --gateway
cautem sandbox template delete shared-agent --gateway
```

Общий lifecycle шаблонов доступен Gateway API клиентам. Создание sandbox из
Gateway-шаблона пока не подключено в CLI; `sandbox create --template` сейчас
читает локальный шаблон.
