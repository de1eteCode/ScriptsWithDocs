# Локальная инфраструктура

Набор PowerShell-скриптов и Docker Compose-конфигурация для запуска локального окружения разработки:

- MongoDB с односерверным replica set;
- RabbitMQ с веб-интерфейсом управления;
- Microsoft SQL Server 2022 Developer;
- Redis;
- RedisInsight.

## Требования

- Docker Desktop или Docker Engine с поддержкой команды `docker compose`;
- PowerShell;
- свободные локальные порты `1433`, `5672`, `6379`, `15672`, `27017` и `5540`.

Все команды следует выполнять из этой папки, чтобы Docker Compose автоматически использовал файл `docker-compose.yml`.

## Быстрый запуск

```powershell
.\start.ps1
```

Скрипт запустит все сервисы в фоновом режиме. После запуска состояние контейнеров можно проверить командой:

```powershell
.\check.ps1
```

## Скрипты

### `start.ps1`

Выполняет:

```powershell
docker compose up -d
```

Создаёт и запускает сервисы из `docker-compose.yml` в фоновом режиме. При повторном запуске Docker Compose обновит контейнеры, если конфигурация или образы изменились.

### `check.ps1`

Выполняет:

```powershell
docker compose ps
```

Показывает контейнеры текущего Compose-проекта, их состояние, результаты healthcheck и опубликованные порты.

### `check-rs-state.ps1`

Выполняет `rs.status()` в MongoDB-контейнере:

```powershell
docker exec -it $(docker compose ps -q mongo) mongosh --eval "rs.status()"
```

Команда получает идентификатор контейнера сервиса `mongo`, запускает внутри него `mongosh` и выводит состояние replica set `rs0`. Скрипт полезен для проверки, что replica set был инициализирован и MongoDB готова к операциям, которым требуются транзакции или change streams.

## Подключение к сервисам

| Сервис | Адрес | Учётные данные |
|---|---|---|
| MongoDB | `mongodb://localhost:27017/mydb?replicaSet=rs0` | не требуются |
| RabbitMQ | `amqp://dev:dev@localhost:5672` | `dev` / `dev` |
| RabbitMQ Management UI | <http://localhost:15672> | `dev` / `dev` |
| Microsoft SQL Server | `localhost,1433` | `sa` / `Qwerty123456!` |
| Redis | `redis://localhost:6379` | не требуются |
| RedisInsight | <http://localhost:5540> | задаются при первом подключении |

> [!WARNING]
> Учётные данные записаны непосредственно в `docker-compose.yml` и предназначены только для локальной разработки. Не используйте эту конфигурацию в production и не открывайте опубликованные порты во внешнюю сеть. Для других окружений храните секреты вне репозитория.

## Сервисы Docker Compose

### MongoDB

- Образ: `mongo:latest`.
- Имя контейнера: `dev-mongo`.
- Порт: `27017`.
- Постоянные данные: том `mongo_data`, подключённый к `/data/db`.
- Ограничение памяти: `768 МБ`, память вместе со swap: `1 ГБ`.
- Политика перезапуска: `unless-stopped`.

MongoDB запускается с replica set `rs0`, принимает подключения на всех интерфейсах контейнера и использует кэш WiredTiger размером `0.25 ГБ`:

```text
mongod --replSet rs0 --bind_ip_all --wiredTigerCacheSizeGB 0.25
```

Healthcheck выполняется каждые 10 секунд. Он запрашивает состояние replica set, а если replica set ещё не создан, инициализирует односерверную конфигурацию с узлом `mongo:27017`. На проверку отводится 5 секунд, допускается 10 неудачных попыток.

### RabbitMQ

- Образ: `rabbitmq:4-management`.
- Имя контейнера: `dev-rabbitmq`.
- AMQP-порт: `5672`.
- Веб-интерфейс управления: `15672`.
- Пользователь и пароль по умолчанию: `dev` / `dev`.
- Постоянные данные: том `rabbitmq_data`, подключённый к `/var/lib/rabbitmq`.
- Ограничение памяти: `512 МБ`, память вместе со swap: `768 МБ`.
- Политика перезапуска: `unless-stopped`.

Healthcheck запускает `rabbitmq-diagnostics -q ping` каждые 15 секунд. На проверку отводится 5 секунд, допускается 5 неудачных попыток.

### Microsoft SQL Server

- Образ: `mcr.microsoft.com/mssql/server:2022-latest`.
- Имя контейнера: `dev-mssql`.
- Порт: `1433`.
- Редакция: Developer.
- Администратор: `sa`.
- Пароль: `Qwerty123456!`.
- Лимит памяти SQL Server: `1536 МБ`.
- Постоянные данные: том `mssql_data`, подключённый к `/var/opt/mssql`.
- Ограничение памяти контейнера: `2 ГБ`, память вместе со swap: `3 ГБ`.
- Политика перезапуска: `unless-stopped`.

Healthcheck выполняет запрос `SELECT 1` через `sqlcmd` каждые 20 секунд. Проверки начинаются после стартового периода 45 секунд; на одну проверку отводится 5 секунд, допускается 10 неудачных попыток.

### Redis

- Образ: `redis:8`.
- Имя контейнера: `dev-redis`.
- Порт: `6379`.
- Постоянные данные: том `redis_data`, подключённый к `/data`.
- Ограничение памяти контейнера: `256 МБ`, память вместе со swap: `384 МБ`.
- Политика перезапуска: `unless-stopped`.

Redis запускается с AOF-персистентностью. Для данных выделено не более `128 МБ`; при достижении лимита удаляются наименее недавно использованные ключи по политике `allkeys-lru`.

Healthcheck выполняет `redis-cli ping` каждые 10 секунд. На проверку отводится 3 секунды, допускается 5 неудачных попыток.

### RedisInsight

- Образ: `redis/redisinsight:latest`.
- Имя контейнера: `dev-redis-insight`.
- Веб-интерфейс: `5540`.
- Постоянные данные: том `redis_insight_data`, подключённый к `/data`.
- Ограничение памяти: `512 МБ`, память вместе со swap: `768 МБ`.
- Политика перезапуска: `unless-stopped`.

RedisInsight запускается после успешного healthcheck сервиса Redis. При первом открытии интерфейса добавьте подключение к Redis. При подключении из контейнера RedisInsight используйте имя хоста `redis` и порт `6379`, а не `localhost`.

## Постоянные данные

Compose-файл создаёт именованные Docker-тома:

| Том | Назначение |
|---|---|
| `mongo_data` | файлы базы данных MongoDB |
| `rabbitmq_data` | очереди, сообщения и настройки RabbitMQ |
| `mssql_data` | базы данных и системные файлы SQL Server |
| `redis_data` | AOF-файл и данные Redis |
| `redis_insight_data` | настройки RedisInsight |

Обычная остановка контейнеров не удаляет эти данные.

## Управление окружением

Остановить и удалить контейнеры и Compose-сеть, сохранив данные:

```powershell
docker compose down
```

Остановить и удалить контейнеры вместе с именованными томами и всеми данными:

```powershell
docker compose down -v
```

Посмотреть журналы всех сервисов:

```powershell
docker compose logs -f
```

Посмотреть журналы отдельного сервиса:

```powershell
docker compose logs -f mongo
```

Вместо `mongo` можно указать `rabbitmq`, `mssql`, `redis` или `redis-insight`.
