# Установка 3x-ui + VLESS Reality + XHTTP на Ubuntu-сервер

Инструкция описывает установку **3x-ui** и создание подключения **VLESS + Reality + XHTTP** — именно того подхода, который описан в репозитории `ServerTechnologies/3x-ui-with-xhttp`.

Отличие от нашего уже рабочего варианта:

```text
наш рабочий вариант: VLESS + Reality + TCP RAW + Vision
вариант из репозитория: VLESS + Reality + XHTTP
```

База одинаковая: **3x-ui + Xray-core + VLESS + Reality**. Отличается транспорт: вместо `TCP (RAW)` выбирается `XHTTP`.

---

## 0. Что получится в итоге

Схема без CDN:

```text
Ubuntu VPS
→ 3x-ui как systemd-служба
→ Xray-core внутри 3x-ui
→ VLESS + Reality + XHTTP
→ порт VLESS: 443/tcp
→ порт панели: случайный или свой, например 17701/tcp
→ порт Let's Encrypt / ACME: 80/tcp
```

Клиенты:

```text
Windows / Linux: v2rayN
Android: v2rayNG
iOS / macOS arm64: Streisand
```

Если на сервере уже есть AmneziaWG, нормальная совместная схема:

```text
AmneziaWG: 51820/udp или 443/udp
3x-ui panel: 17701/tcp
VLESS Reality XHTTP: 443/tcp
ACME / Let's Encrypt: 80/tcp
SSH: 22/tcp
```

Конфликт будет только если другая служба уже использует тот же самый **TCP-порт**, например `443/tcp`.

---

## 1. Команды на сервере

Подключиться к серверу:

```bash
ssh root@IP_СЕРВЕРА
```

Проверить, какие порты уже заняты:

```bash
ss -tulpn
ufw status numbered || true
```

Минимальная подготовка перед установкой:

```bash
apt update
DEBIAN_FRONTEND=noninteractive apt install -y curl wget sudo ufw nano openssl ca-certificates socat cron tar tzdata
```

Эта команда установки базовых пакетов **желательна, но не всегда обязательна**: установочный скрипт 3x-ui обычно сам доставляет часть зависимостей. Но лучше выполнить её заранее, чтобы точно были `curl`, `openssl`, `socat`, `cron`, `ca-certificates` и `ufw`.

⚠️ 
**НЕ НУЖНО делать полный `apt upgrade -y` на рабочем сервере без необходимости.** Особенно если на сервере уже есть SSH-настройки, AmneziaWG или другие службы. `apt upgrade` может задавать вопросы по конфигам вроде `/etc/ssh/sshd_config` или `/etc/cloud/cloud.cfg`.

Команды ниже выполнять **только если есть проблема с apt/dpkg**: например, `dpkg was interrupted`, broken dependencies, зависшее/прерванное обновление.

```bash
dpkg --configure -a
apt install -f -y
```

Полное обновление системы — только осознанно, когда действительно хочешь обновить пакеты ОС:

```bash
apt upgrade -y
```
⚠️

```bash
apt update
```
Установить 3x-ui:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Во время установки:

```text
Would you like to customize the Panel Port settings? [y/n]: y
```

Если скрипт сам сгенерирует случайный порт панели — это нормально. Сохраните его.

На выборе SSL-сертификата в новых версиях 3x-ui лучше выбрать Let's Encrypt по IP:

```text
1. Let's Encrypt for Domain
2. Let's Encrypt for IP Address
3. Custom SSL Certificate

Choose an option: 2
```

На вопрос про IPv6:

```text
Do you have an IPv6 address to include?:
```

Нажать **Enter**, если IPv6 не нужен.

На вопрос про порт ACME:

```text
Port to use for ACME HTTP-01 listener (default 80):
```

Нажать **Enter**, чтобы оставить `80`.

После установки скрипт покажет примерно такие данные:

```text
Username: ...
Password: ...
Port: ...
WebBasePath: ...
Access URL: https://IP_СЕРВЕРА:PORT/WEBBASEPATH
```

Их нужно сохранить.

---

## 2. Открыть порты / firewall

Если `ufw status` показывает `inactive`, то панель может открываться и без этих команд: firewall Ubuntu просто не фильтрует входящие подключения. То есть для работы 3x-ui это **не всегда обязательно**.

Но для безопасности лучше включить UFW и явно разрешить только нужные порты. Замените `17701` на фактический порт панели из вывода установки, например `58666`.

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 17701/tcp
ufw status numbered
```

После этого UFW можно включить:

```bash
ufw --force enable
ufw status verbose
```

Почему `--force`: иногда интерактивное `ufw enable` в SSH-сессии отвечает `Aborted`, даже если вводить `y`. `--force` включает UFW без вопроса.

Если SSH работает не на `22`, сначала откройте свой SSH-порт, иначе можно потерять доступ к серверу.

Если на сервере уже стоит AmneziaWG, **не выполняйте `ufw reset` и не удаляйте существующие правила**. Обычно AmneziaWG использует UDP-порт, например `51820/udp` или `443/udp`, а VLESS/XHTTP использует `443/tcp`, поэтому они не конфликтуют.

Если используете subscription-ссылки 3x-ui, может понадобиться порт подписки, например `2096/tcp`:

```bash
ufw allow 2096/tcp
```

Если обычная `vless://` ссылка/QR копируется напрямую, `2096/tcp` обычно не нужен.

---

## 3. Проверить работу 3x-ui

```bash
x-ui status
systemctl status x-ui --no-pager
```

Посмотреть настройки панели, если забыли URL, логин, порт или base path:

```bash
x-ui settings
```

Перезапуск 3x-ui:

```bash
x-ui restart
```

Логи:

```bash
x-ui log
journalctl -u x-ui -n 100 --no-pager
```

---

## 4. Зайти в панель

Открыть в браузере:

```text
https://IP_СЕРВЕРА:PORT/WEBBASEPATH
```

Пример:

```text
https://45.38.23.54:17701/aqxVQON0xFQ5lE9P9w
```

После первого входа желательно сразу поменять:

```text
username
password
WebBasePath
```

---

## 5. Создать VLESS Reality XHTTP inbound без CDN

В панели:

```text
Inbounds / Подключения → Add Inbound / Создать подключение
```

Верх формы:

```text
Remark / Примечание: vless-reality-xhttp-main
Protocol / Протокол: vless
Listen IP / Мониторинг IP: оставить пустым
Port / Порт: 443
Total Traffic / Общий расход: 0
Reset Traffic / Сброс трафика: Never / Никогда
Expiry Time / Дата окончания: не задавать
```

Блок клиента:

```text
Email / Name: laptop
UUID / ID: Generate / сгенерировать
Flow: Пусто / empty
Total Traffic / Общий расход: 0
Expiry Time / Дата окончания: не задавать
Enable / Включить: ON
Authentication: None / пусто
Encryption: none
Decryption: none
```

Важно: для XHTTP не ставим `xtls-rprx-vision`. Это было нужно для TCP RAW + Vision. Для XHTTP поле `Flow` лучше оставить пустым.

Блок транспорта для простого XHTTP без CDN:

```text
Transmission / Транспорт: XHTTP
Host / Хост: пусто
Path / Путь: /
Request Header / Заголовок запроса: не добавлять
Mode: auto
Padding Bytes: 100-1000
Padding Obfs Mode: OFF
Uplink HTTP Method: Default (POST)
Session Placement: Default (path)
Sequence Placement: Default (path)
No SSE Header: OFF
Sockopt: OFF
TCP Masks: не добавлять
QUIC Params: OFF
External Proxy: OFF
```

`Host` пустой — это нормально для простого варианта без CDN. `Path: /` тоже нормально. Расширенные XHTTP-поля в разных версиях 3x-ui могут выглядеть немного по-разному; если сомневаетесь, оставляйте дефолтные значения как выше.

Блок безопасности:

```text
Security / Безопасность: Reality
```

Reality-настройки:

```text
Xver: 0
uTLS / Fingerprint: chrome
Target / Dest: www.sony.com:443
SNI / Server Names: www.sony.com
Max Time Diff: 0
Min Client Ver: пусто
Max Client Ver: пусто
SpiderX: /
Short IDs: fe4fbaaf65eacb35
```

`Target` и `SNI` должны совпадать по домену. Можно использовать и другой доступный HTTPS-сайт, например:

```text
www.apple.com:443
www.apple.com

www.nvidia.com:443
www.nvidia.com
```

После `Get New Cert` и `Get New Seed` **не меняйте** `Target`, `SNI`, `Short IDs`, `Public Key`, `Private Key`, `mldsa65 Seed` и `mldsa65 Verify` без повторного экспорта новой ссылки в клиент. Если эти поля поменять, старая ссылка в v2rayN/v2rayNG станет неактуальной.

Нажать:

```text
Get New Cert
```

После этого должны заполниться:

```text
Public Key
Private Key
```

Для XHTTP в репозитории также предлагается нажать:

```text
Get New Seed
```

После этого должны заполниться:

```text
mldsa65 Seed
mldsa65 Verify
```

Для варианта из репозитория нажимаем `Get New Seed`. Если какой-то клиент плохо импортирует ссылку или не поддерживает ML-DSA/seed, можно попробовать вариант без `mldsa65 Seed / Verify`, но сначала лучше следовать репозиторию буквально.

Sniffing:

```text
Sniffing: ON
HTTP: ON
TLS: ON
QUIC: ON
Metadata Only: OFF
Route Only: OFF
```

Нажать:

```text
Create / Создать
```

---

## 6. Проверить, что Xray слушает порт 443

После создания inbound на сервере:

```bash
ss -tulpn | grep 443
```

Нормальный результат должен быть примерно такой:

```text
tcp LISTEN 0 4096 *:443 *:* users:(("xray-linux-amd64",pid=...,fd=...))
```

Если порт `443` слушает `xray`, значит inbound поднялся.

Логи при проблемах:

```bash
x-ui log
journalctl -u x-ui -n 100 --no-pager
```

---

## 7. Создать отдельного клиента для телефона

Один QR можно использовать и на ПК, и на телефоне, но лучше делать отдельного клиента на каждое устройство.

В уже созданном inbound:

```text
Edit / Изменить → Clients / Клиенты → Add Client
```

Для телефона:

```text
Email / Name: phone
UUID: Generate
Flow: пусто
Limit IP: пусто или 0
Total GB: 0
Expiry Time: не задавать
Enable: ON
```

Сохранить.

Итоговая структура:

```text
1 inbound XHTTP на 443
client laptop → отдельная ссылка/QR
client phone  → отдельная ссылка/QR
```

---

## 8. Где взять QR или ссылку

В списке Inbounds нажать маленький `+` слева от строки inbound. Раскроется список клиентов.

У конкретного клиента можно открыть QR или скопировать ссылку.

Если нажать на QR, в некоторых версиях 3x-ui ссылка просто копируется в буфер обмена. Это нормально.

Ссылка начинается так:

```text
vless://...
```

Для XHTTP в ссылке должны быть примерно такие параметры:

```text
type=xhttp
security=reality
pbk=...
fp=chrome
sni=www.apple.com
sid=fe4fbaaf65eacb35
spx=%2F
```

Если после любых изменений нажимали `Get New Cert`, меняли `Short IDs`, `SNI`, `Target` или `Seed`, нужно заново экспортировать ссылку и импортировать её в клиент. Старая ссылка уже может не работать.

---

## 9. Клиент для Windows

Рекомендуемый клиент:

```text
v2rayN
```

Скачивать:

```text
GitHub → 2dust/v2rayN → Releases
```

Для обычного Windows 10/11 x64:

```text
v2rayN-windows-64-desktop.zip
```

Это portable-приложение, установщика нет.

Как использовать:

```text
1. Распаковать архив.
2. Перенести папку, например, в C:\Tools\v2rayN\
3. Запустить v2rayN.exe.
4. Создать ярлык на рабочий стол или закрепить на панели задач.
```

Импорт ссылки:

```text
Configuration → Import share links from clipboard
```

или:

```text
Servers → Import bulk URL from clipboard
```

После импорта:

```text
1. Выбрать импортированный сервер.
2. Правой кнопкой → Set as active server.
3. Внизу включить Set system proxy.
4. Режим маршрутизации поставить Global / Proxy all.
5. Нажать Reload.
```

Для теста открыть:

```text
https://google.com
https://ifconfig.me
```

---

## 10. Клиент для Android

Рекомендуемый клиент:

```text
v2rayNG
```

Скачивать:

```text
GitHub → 2dust/v2rayNG → Releases
```

Для большинства современных телефонов:

```text
v2rayNG_*_arm64-v8a.apk
```

Если не уверены в архитектуре телефона:

```text
v2rayNG_*_universal.apk
```

Импорт:

```text
+ → Scan QR code
```

или:

```text
+ → Import config from clipboard
```

Потом выбрать профиль и нажать кнопку подключения.

---

## 11. XHTTP через CDN Cloudflare

Это продвинутый вариант. Его стоит делать только если простой XHTTP без CDN уже понятен.

Нужны:

```text
домен или поддомен
Cloudflare
DNS-запись домена на сервер
```

В 3x-ui создать отдельный inbound:

```text
Protocol: VLESS
Port: любой свободный порт, кроме 80, 443, 22, например 8443
Transport: XHTTP
Host: ваш домен или поддомен
Path: любой путь, например /xhttp
Security: Пусто / None
Sniffing: OFF
```

В Cloudflare:

```text
SSL/TLS → Flexible
SSL/TLS → Edge Certificates → TLS 1.3 OFF
Network → gRPC ON
Rules → Origin Rule
Hostname equals ваш домен/поддомен
Destination Port → Rewrite to порт inbound, например 8443
```

Клиентская настройка после импорта ссылки:

```text
Address: ваш домен/поддомен
Port: 443
Security: TLS
SNI: ваш домен/поддомен
```

Этот вариант сложнее и может снижать скорость, но скрывает IP сервера за CDN.

---

## 12. Полезные команды управления

Статус службы:

```bash
systemctl status x-ui --no-pager
```

Перезапуск:

```bash
systemctl restart x-ui
```

Остановить:

```bash
systemctl stop x-ui
```

Запустить:

```bash
systemctl start x-ui
```

Меню управления:

```bash
x-ui
```

Логи:

```bash
x-ui log
journalctl -u x-ui -n 100 --no-pager
```

Проверить порты:

```bash
ss -tulpn | grep -E '443|17701|80'
```

---

## 13. Удаление 3x-ui

Обычно достаточно:

```bash
x-ui uninstall
```

Если нужно вручную подчистить:

```bash
systemctl stop x-ui || true
systemctl disable x-ui || true
rm -f /etc/systemd/system/x-ui.service
systemctl daemon-reload
systemctl reset-failed
rm -rf /usr/local/x-ui
rm -rf /etc/x-ui
rm -rf /root/cert/ip
```

Если использовался Let's Encrypt-сертификат по IP:

```bash
~/.acme.sh/acme.sh --remove -d IP_СЕРВЕРА --ecc || true
rm -rf ~/.acme.sh/IP_СЕРВЕРА_ecc
```

Закрывать только те порты, которые точно больше не нужны:

```bash
ufw delete allow 17701/tcp
ufw delete allow 443/tcp
ufw delete allow 80/tcp
ufw status
```

`22/tcp` не удалять, если через него работает SSH. Если на сервере стоит AmneziaWG, не удаляйте его UDP-порты, например `51820/udp` или `443/udp`.

---

## 14. Короткий итог

Простой XHTTP без CDN:

```text
Protocol: VLESS
Port: 443
Transport: XHTTP
Security: Reality
Flow: пусто
uTLS: chrome
Target/SNI: www.apple.com
Get New Cert: да
Get New Seed: по инструкции репозитория да, но при проблемах можно тестировать без seed
Sniffing: ON
```

Клиенты:

```text
Windows: v2rayN-windows-64-desktop.zip
Android: v2rayNG_arm64-v8a.apk
iOS / macOS arm64: Streisand
```

Для постоянного использования лучше делать отдельный клиент/QR на каждое устройство:

```text
laptop
phone
tablet
```
<img width="621" height="1919" alt="image" src="https://github.com/user-attachments/assets/fa9ebb2e-7ca4-469e-ae5b-b21f12c0e848" />
<img width="619" height="1429" alt="image" src="https://github.com/user-attachments/assets/45db3e54-67fa-4d74-a087-83c05806dd9a" />

