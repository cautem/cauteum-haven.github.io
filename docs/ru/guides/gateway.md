<!--
SPDX-FileCopyrightText: Copyright (c) 2026 cautem
SPDX-License-Identifier: Apache-2.0
-->

# Gateway

Gateway — опциональный control plane: encrypted store секретов, registry
sandbox, observation log rings и workflow policy proposals.

## Запуск

```bash
cautem gateway ensure
cautem gateway add http://127.0.0.1:7443 --local --name local
cautem gateway select local
cautem gateway info
```

Compose: `cautem-gateway/compose/docker-compose.yml`.

Адрес по умолчанию: `127.0.0.1:7443`.
При `CAUTEM_GATEWAY_ALLOW_UNAUTHENTICATED=1` или флаге
`--allow-unauthenticated-users` gateway запускается только на loopback-адресе.
На публичном или LAN-адресе запуск завершается ошибкой. Поле
`allow_unauthenticated` видно в `cautem gateway info`, а `cautem status`
показывает предупреждение. Reverse proxy и проброс порта могут открыть
loopback-сервис другим машинам — учитывайте это при развёртывании.

## Зоны ответственности

| Поверхность | Роль |
|-------------|------|
| Secrets | AES-GCM store; KEK через `CAUTEM_SECRETS_KEK` или `secrets.kek` |
| Providers | Именованные инстансы + метаданные composition |
| Sandboxes | Управление реестром и жизненным циклом через RPC; ограниченный буфер логов |
| Sidecar | Resolve секретов через `host.cautem.internal` |

## Справка по HTTP API

[Спецификация OpenAPI](https://github.com/cautem/cautem-gateway/blob/main/api/openapi.yaml)
описывает только health и bootstrap-аутентификацию. Операции управления
используют `cautem.control.v1` или закреплённый OpenShell RPC контракт. SSH
потоки и supervisor relay используют HTTP upgrade transport и не являются
REST API ресурсов.

## Client API для управления

Gateway также обслуживает версионированный Connect API
`cautem.control.v1` для будущей панели управления и генерируемых SDK.
Текущая read-часть включает viewer/capabilities, сводки workspace и sandbox,
сервисов и шаблонов, а также логи и streams sandbox. Операции create/start/stop/
delete используют идемпотентный `request_id`; потерянный ответ можно сверить
через `GetOperation`. История операций и аудита доступна авторизованным admin.
Proto-схема и приватный TypeScript client находятся в `api/` репозитория
gateway. Первая read-only панель находится в `ui/`: она использует OIDC
authorization code с PKCE, хранит access token только в памяти и показывает
доступный пользователю список sandbox и ограниченный хвост логов. Настройте
публичный OIDC client, callback URL и audience токена, совпадающий с настройкой
gateway. Раздача production-файлов и same-origin proxy ещё требуют настройки
развёртывания. Панель показывает статус реестра как данные реестра и не заявляет
о runtime health. Браузер никогда не должен получать gateway owner или sandbox
supervisor credentials.

Команды CLI `sandbox list/get`, методы SDK для списка и деталей sandbox,
фильтрованные снимки и потоковые логи используют этот API через native gRPC в
Go SDK. `logs --all` перечисляет доступные пользователю workspace и одновременно
открывает не более 24 потоков.
Сначала нужно выпустить beta gateway со сгенерированным Go-контрактом, затем
соответствующую beta SDK.

Список, просмотр, импорт, обновление и удаление provider profiles, частичное
обновление credentials, чтение и запись глобальной и sandbox policy, а также
история policy используют `cautem.control.v1` через native gRPC. Ответы
профилей сохраняют полную YAML-схему cautem. Эти REST-маршруты и записи
OpenAPI удалены. Settings, services, workspaces, inference, identity, SSH
session management и command execution также перешли на RPC; по HTTP остались
bootstrap, health и relay transport.

## Когда обязателен

| Операция | Gateway |
|----------|---------|
| Attach `--provider` | Обязателен |
| Egress proxy + rewrite | Обязателен |
| Create `--no-proxy` без providers | Опционален (dev) |

## Конфиг

Gateways пишутся в `~/.config/cautem/config.yaml`:

```yaml
current: local
gateways:
  local:
    url: http://127.0.0.1:7443
```

Поля OIDC и токены появляются после `cautem gateway login`.

## Связанное

- [Credentials](./credentials.md)
- [Политика](./policy.md)
- Compose-файлы: `cautem-gateway/compose/`

## Gateway TOML OpenShell

Daemon читает `--config gateway.toml`, затем `OPENSHELL_GATEWAY_CONFIG`,
затем необязательный `$XDG_CONFIG_HOME/openshell/gateway.toml`
(по умолчанию `~/.config/openshell/gateway.toml`). Явно указанный отсутствующий
файл вызывает ошибку. Для поддержанных settings приоритет: flag > env > file.

Startup применяет адрес основного listener, имя установки в логах,
простые log levels, SSH session TTL, local auth, OIDC/JWKS, TLS server/client
certificates, external SNI certificates, mTLS identity и выбор builtin/user
`provider_profile_sources`. Referenced paths сохраняются буквально и
вычисляются относительно рабочего каталога процесса. `disable_tls=true`
игнорирует server certificates; одновременно заданный client CA отклоняется.

Поддержка частичная. Полные deploy files OpenShell ещё требуют consumers
для drivers, storage, gateway JWT, OTLP, rate limits, middleware, interceptors
и полной readiness/metrics instrumentation. Указанные неподдержанные settings блокируют startup
до создания state и открытия listeners. Loader также отклоняет unknown/duplicate
keys, отсутствующие required fields, неверные enums и database URL внутри TOML.
Parser принимает schema version 2 из закреплённого v0.1.4 и его singular
selector `compute_driver`; существующий список `compute_drivers` оставлен для
локального multi-backend формата. Парсинг upstream fixture не означает, что
все настройки его drivers уже запускаются: неподдержанные backend settings
по-прежнему fail closed на startup.
Без файла OpenShell пока действуют прежние defaults cautem.
Необязательные `health_bind_address` и `metrics_bind_address`, а также
`OPENSHELL_HEALTH_PORT` / `OPENSHELL_METRICS_PORT` и соответствующие флаги
запускают отдельные listeners. Health routes доступны на `/healthz`, `/readyz`
и `/health`; metrics сейчас отдаёт только gauge состояния gateway. Мониторинг
готовности БД и полная Prometheus instrumentation пока не реализованы.

Настройки OpenShell OIDC issuer, audience, JWKS cache TTL и role claim
передаются JWT validator. Admin role может изменять gateway resources; user role
имеет read-only доступ; admin также удовлетворяет user role. При указанном
`scopes_claim` Control RPC использует группы `sandbox:read/write`,
`provider:read/write` и `config:read/write`; конкретные handlers также проверяют
workspace и admin permissions. Полная per-method descriptor matrix ещё не
реализована. Поддерживаются RSA, ECDSA P-256/P-384 и Ed25519 signing keys.

TLS загружает client CA и external cert/key pair из исходного gateway TOML.
При включённом OIDC клиентский сертификат проверяется, если он предъявлен;
mTLS-only authentication требует проверенный client certificate. External
certificate выбирается по настроенному точному или wildcard SNI имени.
Автоматическая local PKI и hot reload сертификатов ещё не реализованы.

По умолчанию используются встроенный и сохранённый user profile catalogs.
Исходный список позволяет выбрать только нужные источники, например:

```toml
[openshell.gateway]
provider_profile_sources = [{ type = "builtin" }]
```

Interceptor source отклоняется, пока не появятся gateway interceptor runtime
и его catalog protocol.
