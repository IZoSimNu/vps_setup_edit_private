<h1 align="center">Настройка VPS с нуля + Marzban + VLESS TCP REALITY + nginx steal_oneself</h1>

## Сетевые порты

Для внешнего доступа к серверу в этой инструкции нужны:
- TCP 22 — SSH (лучше ограничить доступ своим IP в firewall провайдера).
- TCP 80 — выпуск и продление сертификата Let's Encrypt через HTTP-01.
- TCP 443 — HTTPS-вход.

Порты 8000, 1443 и 8443 используются внутренними компонентами схемы. Не добавляйте их в список публично открытых портов. Перед запуском проверьте, что панель Marzban не опубликована Docker на внешнем интерфейсе: это отдельный пункт проверки после установки.

Если используете firewall в панели VPS-провайдера, разрешите только необходимые входящие подключения. Не включайте UFW вслепую на удалённом сервере: сначала разрешите SSH и убедитесь, что доступ не потеряется.

### Требования:
- VPS
- Ubuntu 22.04 и выше
- Доступ по SSH, если еще нет
- Домен fallback, например `example.ru`, направленный A-записью на IP VPS.
- Доменное имя для REALITY SNI, например `www.example.ru`. Оно должно совпадать с `serverNames` в Xray и с именем в `stream map`.
- Для REALITY SNI не обязательно создавать отдельную DNS-запись на IP VPS: проверьте требования выбранной схемы и используйте домен, который действительно обслуживает TLS на целевой стороне. Не указывайте несуществующее имя.
### Зависимости:
`docker fail2ban mc net-tools cron socat curl wget iperf3 nginx libnginx-mod-stream`

Для проверки DNS-записей установите `dnsutils`:
```bash
sudo apt install -y dnsutils
dig +short example.ru A
```

## Настройка сервера
Апдейт системы, если еще не делали
```bash
sudo apt update && sudo apt upgrade -y
```
### Установка нужного ПО
Может и не очень нужного, однако я его ставлю в большинстве случаев
```bash
sudo apt install -y fail2ban mc net-tools cron socat curl wget iperf3
```
### Включение bbr 
Это поможет увеличить пропускную способность сетевого интерфейса
```bash
echo "net.core.default_qdisc=fq" | sudo tee -a /etc/sysctl.conf
echo "net.ipv4.tcp_congestion_control=bbr" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```


### Установка docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
rm -f get-docker.sh
```
Скрипт устанавливает стабильный канал Docker, но Docker отмечает этот способ как удобный вариант для быстрого развёртывания, а не предпочтительный метод для production. Для строгой production-установки используйте установку Docker из официального APT-репозитория. Не запускайте установщик повторно без необходимости.

Проверяем:
```bash
docker ps -a
```
Если в выводе есть пустая таблица докера - победа.
### Установка Marzban
- Запустим установочный скрипт
```bash
sudo bash -c "$(curl -sL https://github.com/Gozargah/Marzban-scripts/raw/master/marzban.sh)" @ install
```
- Создадим суперпользователя для администрирования системы
```bash
marzban cli admin create --sudo
```
### Настройка SSL для Marzban

В этом примере `example.ru` — домен fallback и адрес подписок. У него должна быть корректная DNS A-запись на IP VPS. Не добавляйте `www.example.ru` в сертификат, если для него нет DNS-записи или он не используется.

Проверить DNS:
```bash
sudo apt install -y dnsutils
dig +short example.ru A
```

Установим acme.sh от имени root (это важно, поскольку сертификаты будут размещены в системной директории):
```bash
curl https://get.acme.sh | sh -s email=ВАШ_EMAIL
```

Создадим директорию:
```bash
mkdir -p /var/lib/marzban/certs
```

Для выпуска сертификата в режиме standalone порт 80 должен быть доступен из интернета и свободен на самом VPS. Если Nginx уже установлен и запущен, временно остановите его. Если Nginx ещё не установлен, команду остановки пропустите:
```bash
systemctl is-active --quiet nginx && systemctl stop nginx || true
/root/.acme.sh/acme.sh --set-default-ca --server letsencrypt
/root/.acme.sh/acme.sh --issue --standalone -d example.ru
```

Установим сертификат в постоянные пути, которые будут использовать Marzban и Nginx:
```bash
/root/.acme.sh/acme.sh --install-cert -d example.ru \
  --key-file /var/lib/marzban/certs/key.pem \
  --fullchain-file /var/lib/marzban/certs/fullchain.pem \
  --reloadcmd "marzban restart; if systemctl is-active --quiet nginx; then systemctl reload nginx; fi"
```

Команда `marzban restart` используется штатным CLI-скриптом Marzban. Убедитесь, что команда `marzban` доступна в PATH и что Nginx установлен и запущен к моменту продления сертификата.

Проверим сертификат и его установку:
```bash
/root/.acme.sh/acme.sh --list
ls -l /var/lib/marzban/certs/
```

Запустим Nginx:
```bash
systemctl enable nginx --now
```

acme.sh автоматически добавляет задачу продления сертификата. При успешном продлении выполнится `reloadcmd`: перезапустится Marzban, чтобы он перечитал сертификат, и перечитается конфигурация Nginx. Убедитесь, что указанный в `reloadcmd` контейнер существует.

Откроем файл окружения Marzban (путь зависит от способа установки; в стандартной установке скриптом обычно используется `/opt/marzban/.env`):
```bash
nano /opt/marzban/.env
```

Проверьте/добавьте параметры:
```env
UVICORN_HOST = "127.0.0.1"
UVICORN_PORT = 8000
UVICORN_SSL_CERTFILE = "/var/lib/marzban/certs/fullchain.pem"
UVICORN_SSL_KEYFILE = "/var/lib/marzban/certs/key.pem"
XRAY_SUBSCRIPTION_URL_PREFIX = "https://example.ru"
```

Параметр `UVICORN_HOST = "127.0.0.1"` принципиален: он ограничивает панель Marzban локальным интерфейсом. Не меняйте его на `0.0.0.0`, иначе порт 8000 станет доступен на внешних интерфейсах VPS (если это не ограничено отдельно firewall).

После изменения `.env` перезапустите Marzban штатной командой:
```bash
marzban restart
```

### Поднимаем Nginx для fallback и маскировки

Установим Nginx и Stream-модуль:
```bash
apt update
apt install -y nginx libnginx-mod-stream
```

В Ubuntu пакет обычно создаёт файл загрузки модуля:
```bash
cat /etc/nginx/modules-enabled/50-mod-stream.conf
```

В нём должна быть директива:
```nginx
load_module modules/ngx_stream_module.so;
```

Если файла нет, проверьте наличие модуля и создайте его:
```bash
ls -l /usr/lib/nginx/modules/ngx_stream_module.so
printf '%s\n' 'load_module modules/ngx_stream_module.so;' > /etc/nginx/modules-enabled/50-mod-stream.conf
```

Откроем основной конфиг:
```bash
nano /etc/nginx/nginx.conf
```

В конфигурации должны быть секции `events`, `http` и `stream`. Не добавляйте второй блок `http` или `events`, если они уже есть. Пример полной конфигурации для чистой установки Ubuntu:

```nginx
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 1024;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}

stream {
    map $ssl_preread_server_name $backend {
        www.example.ru reality;
        default web;
    }

    upstream reality {
        server 127.0.0.1:1443;
    }

    upstream web {
        server 127.0.0.1:8443;
    }

    server {
        listen 443;
        proxy_pass $backend;
        ssl_preread on;
    }
}
```

В строке `www.example.ru reality;` укажите то же имя, которое прописано в `serverNames` в настройках REALITY Xray. Это имя SNI для REALITY, а не обязательно домен fallback/подписок. Значение `default web` отправляет остальные TLS-подключения на веб-сервер.

Создадим конфигурацию сайта:
```bash
nano /etc/nginx/sites-available/skufee
```

Вставим конфигурацию. Она отдаёт статическую заглушку на главной странице, а остальные запросы, включая ссылки подписок, передаёт в Marzban. Важный момент: Marzban в этой схеме слушает HTTPS на порту 8000, поэтому в `proxy_pass` используется именно `https://`, а не `http://`.

```nginx
server {
    listen 127.0.0.1:8443 ssl http2;
    server_name example.ru;

    ssl_certificate /var/lib/marzban/certs/fullchain.pem;
    ssl_certificate_key /var/lib/marzban/certs/key.pem;

    root /var/www/html;
    index index.html;

    location = / {
        try_files /index.html =404;
    }

    location / {
        proxy_pass https://127.0.0.1:8000;

        proxy_ssl_server_name on;
        proxy_ssl_name example.ru;
        proxy_ssl_verify on;
        proxy_ssl_trusted_certificate /etc/ssl/certs/ca-certificates.crt;
        proxy_ssl_verify_depth 2;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

Nginx проверяет сертификат Marzban и имя `example.ru` при локальном HTTPS-соединении. Для этого сертификат должен быть выпущен доверенным центром сертификации и содержать домен `example.ru`.

Создадим тестовую страницу:
```bash
mkdir -p /var/www/html
echo "Hello, world" > /var/www/html/index.html
```

Включим сайт. Если стандартная конфигурация `default` включена и конфликтует с этой схемой, отключим её:
```bash
ln -sfn /etc/nginx/sites-available/skufee /etc/nginx/sites-enabled/skufee
rm -f /etc/nginx/sites-enabled/default
nginx -t
systemctl enable nginx --now
systemctl reload nginx
```

### Настройка протокола VLESS
Заходим в настройки и заменяем конфиг следующим образом (добавляется секция VLESS TCP REALITY, можно из старого все убрать и скопировать отсюда):
```json
{
  "log": {
    "loglevel": "warning"
  },
  "routing": {
    "rules": [
      {
        "ip": [
          "geoip:private"
        ],
        "outboundTag": "BLOCK",
        "type": "field"
      }
    ]
  },
  "inbounds": [
    {
      "tag": "VLESS TCP REALITY",
      "listen": "127.0.0.1",
      "port": 1443,
      "protocol": "vless",
      "settings": {
        "clients": [],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "tcpSettings": {},
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "127.0.0.1:8443",
          "xver": 0,
          "serverNames": [
            "www.example.ru"
          ],
          "privateKey": "ключ",
          "shortIds": [
            "айди"
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": [
          "http",
          "tls",
          "quic"
        ]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "DIRECT"
    },
    {
      "protocol": "blackhole",
      "tag": "BLOCK"
    }
  ]
}
```
Для того, чтобы сгенерировать ключ (параметр `privateKey`) нужно воспользоваться командой
```bash
docker exec marzban_marzban_1 xray x25519
```
Контейнер Marzban'a может создаться с именем marzban-marzban-1, в таком случае команда будет:
```bash
docker exec marzban-marzban-1 xray x25519
```
Отсюда нас интересует значение, собственно, PrivateKey. Копируем его и вставляем в конфиг. Далее генерируем айди параметр `shortIds`
```bash
openssl rand -hex 8
```
Также копируем его и вставляем в параметр `shortIds`
![vless](https://github.com/user-attachments/assets/5721705c-b4fd-4c53-81d6-f7f9c52bc85f)

Имя в `serverNames` должно совпадать с именем, направленным в upstream `reality` в Nginx `stream map`. После этого нажать сохранить и перезагрузить ядро и перезагрузить страницу.
Теперь можно добавить пользователя, выбрать протокол VLESS, указать имя пользователя, развернуть на три точки настройки и указать Flow xtls-rprx-vision.
После создания, в основном меню можно будет скопировать ссылку на подписку и ссылку конфигурации (типа vless//).

Проверим локальные порты:
```bash
curl -k https://127.0.0.1:8000/
curl -k https://127.0.0.1:8443/
```

Первый запрос должен обращаться к HTTPS-интерфейсу Marzban. Второй — к Nginx на локальном порту 8443. После настройки DNS и Stream проверим внешний домен:
```bash
curl -v https://example.ru/
```

На главной странице ожидается `Hello, world`. Ссылку подписки проверяйте по реальному URL с токеном, который выдал Marzban; не публикуйте токен в логах, скриншотах и сообщениях. Если подписки не открываются, проверьте, что `XRAY_SUBSCRIPTION_URL_PREFIX` в `.env` совпадает с внешним HTTPS-доменом.

### Добавление телеграм бота
- Создаем телеграм бота в @BotFather (при создании будет сообщение с АПИ)
- Получаем свой айди в @userinfobot
- Редактируем конфигурационный файл
```bash
sudo nano /opt/marzban/.env
```
- Раскомментируем TELEGRAM_API_TOKEN и TELEGRAM_ADMIN_ID
- Копируем API из сообщения о создании бота и свой айди
- Сохраняем файл: `Ctrl+O`, затем `Enter`; выходим: `Ctrl+X`.
- Перезапускаем Marzban штатной командой CLI
```bash
docker restart marzban-marzban-1
```
- Открываем бота (по ссылке из сообщения ботфазера) и запускаем.
Теперь можно создавать пользователей и мониторить их через телеграм бота. Также отсюда можно сразу копировать ссылки для подключений.
