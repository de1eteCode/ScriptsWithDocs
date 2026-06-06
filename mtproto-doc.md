# Установка mtproto

## Узнай свободный порт
Выбери любой, где будет свободен

Скрипт:

for p in 2053 2083 2087 2096 8444 10443 11443 12443 15443 18443 20443; do
  if ss -tulpn | grep -q ":$p "; then
    echo "$p занят"
  else
    echo "$p свободен"
  fi
done

Дальше пример идет для порта - 18443

## Открой порт в firewall (Если используешь ufw)

Узнать статус ufw - ufw status

ufw allow 18443/tcp
ufw reload

## Docker
apt update
apt install -y docker.io
systemctl enable --now docker

Проверка - docker --version

## Сгенерируй secret для MTProto
SECRET=$(head -c 16 /dev/urandom | xxd -ps)
echo $SECRET

## Запусти MTProto контейнер
docker run -d \
  --name mtproto-proxy \
  --restart unless-stopped \
  -p 18443:443 \
  -e SECRET=$SECRET \
  telegrammessenger/proxy:latest
  
## Проверка что контейнер работает
docker ps
docker logs mtproto-proxy

## Ссылка для подключения в Telegram
Ссылка будет такого вида: tg://proxy?server=YOUR_SERVER_IP&port=18443&secret=YOUR_SECRET

## Управление
Остановить - docker stop mtproto-proxy
Запустить - docker start mtproto-proxy
Перезапустить - docker restart mtproto-proxy
Удалить - docker rm -f mtproto-proxy
Посмотреть secret, если забыл - docker inspect mtproto-proxy | grep SECRET