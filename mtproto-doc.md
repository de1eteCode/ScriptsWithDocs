# Установка MTProto

## Узнать свободный порт

Выберите любой свободный порт.

Проверить список популярных портов можно следующим скриптом:

```bash
for p in 2053 2083 2087 2096 8444 10443 11443 12443 15443 18443 20443; do
  if ss -tulpn | grep -q ":$p "; then
    echo "$p занят"
  else
    echo "$p свободен"
  fi
done
```

В дальнейшем в примерах используется порт **18443**.

---

## Открыть порт в firewall, если используется UFW

```bash
ufw allow 18443/tcp
ufw reload
```

Проверить статус UFW:

```bash
ufw status
```

> Если у VPS-провайдера есть собственный firewall или Security Group, порт необходимо открыть и там.

---

## Установка Docker

```bash
apt update
apt install -y docker.io
systemctl enable --now docker
```

Проверка установки:

```bash
docker --version
```

Проверка работы службы:

```bash
systemctl status docker
```

---

## Сгенерировать Secret для MTProto

```bash
SECRET=$(head -c 16 /dev/urandom | xxd -ps)
echo $SECRET
```

Сохраните значение `SECRET` — оно понадобится для подключения в Telegram.

---

## Запустить контейнер MTProto

```bash
docker run -d \
  --name mtproto-proxy \
  --restart unless-stopped \
  -p 18443:443 \
  -e SECRET=$SECRET \
  telegrammessenger/proxy:latest
```

---

## Проверить работу контейнера

Список контейнеров:

```bash
docker ps
```

Просмотр логов:

```bash
docker logs mtproto-proxy
```

Просмотр потребления ресурсов:

```bash
docker stats mtproto-proxy
```

---

## Получить IP-адрес сервера

```bash
curl -4 ifconfig.me
```

---

## Подключение в Telegram

Ссылка для подключения:

```text
tg://proxy?server=YOUR_SERVER_IP&port=18443&secret=YOUR_SECRET
```

Пример:

```text
tg://proxy?server=1.2.3.4&port=18443&secret=abcdef1234567890abcdef1234567890
```

Также можно добавить прокси вручную:

- Тип: MTProto
- Сервер: IP сервера
- Порт: 18443
- Secret: значение переменной `SECRET`

---

## Управление контейнером

Остановить:

```bash
docker stop mtproto-proxy
```

Запустить:

```bash
docker start mtproto-proxy
```

Перезапустить:

```bash
docker restart mtproto-proxy
```

Удалить:

```bash
docker rm -f mtproto-proxy
```

---

## Получить Secret после установки

```bash
docker inspect mtproto-proxy | grep SECRET
```

---

## Обновление MTProto

Сначала убедитесь, что переменная `SECRET` доступна в текущей сессии.

Если переменная уже есть:

```bash
echo $SECRET
```

Если переменной нет, можно получить Secret из текущего контейнера:

```bash
docker inspect mtproto-proxy | grep SECRET
```

После этого обновите образ и пересоздайте контейнер:

```bash
docker pull telegrammessenger/proxy:latest

docker stop mtproto-proxy
docker rm mtproto-proxy

docker run -d \
  --name mtproto-proxy \
  --restart unless-stopped \
  -p 18443:443 \
  -e SECRET=$SECRET \
  telegrammessenger/proxy:latest
```

---

## Полезные команды

Проверить, какой процесс использует порт:

```bash
ss -tulpn | grep :18443
```

Посмотреть все контейнеры:

```bash
docker ps -a
```

Посмотреть последние 50 строк логов:

```bash
docker logs --tail 50 mtproto-proxy
```

Перезапустить контейнер после изменения настроек:

```bash
docker restart mtproto-proxy
```

---

## Примечания

- Не используйте порт, который уже занят Nginx, Xray, 3x-ui или другими сервисами.
- Для личного использования MTProto обычно потребляет менее 100 МБ оперативной памяти.
- Контейнер использует официальный Docker-образ Telegram: `telegrammessenger/proxy`.
- Если Telegram не подключается к прокси, проверьте:
  - открыт ли порт в UFW;
  - открыт ли порт у VPS-провайдера;
  - корректно ли указан Secret;
  - запущен ли контейнер.
