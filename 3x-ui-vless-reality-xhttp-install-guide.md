═══════════════════════════════════════════
     Panel Installation Complete!
═══════════════════════════════════════════
Username:    smCHnAyh7i
Password:    5IdNlAoLkk
Port:        60870
WebBasePath: AyHKwEGM4F2VYcCEMx
Access URL:  https://31.59.114.59:60870/AyHKwEGM4F2VYcCEMx
═══════════════════════════════════════════

# Установка 3x-ui + VLESS Reality + XHTTP на Ubuntu-сервер

Инструкция описывает установку **3x-ui** и создание подключения **VLESS + Reality + XHTTP** — именно подхода из репозитория `ServerTechnologies/3x-ui-with-xhttp`.

Отличие от варианта `TCP RAW + Vision`:

```text
TCP RAW + Vision: VLESS + Reality + TCP RAW + Flow xtls-rprx-vision
XHTTP:            VLESS + Reality + XHTTP, Flow пустой
```

База одинаковая: **3x-ui + Xray-core + VLESS + Reality**. Отличается транспорт.

---

## 0. Что получится в итоге

Схема без CDN:

```text
Ubuntu VPS
→ 3x-ui как systemd-служба
→ Xray-core внутри 3x-ui
→ VLESS + Reality + XHTTP
→ порт VLESS/XHTTP: 443/tcp
→ порт панели 3x-ui: лучше задать вручную, например 58666/tcp
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
AmneziaWG VPN:       51820/udp
AmneziaWG panel:     51821/tcp
3x-ui panel:         58666/tcp или другой выбранный порт
VLESS Reality XHTTP: 443/tcp
ACME / Let's Encrypt: 80/tcp
SSH:                 22/tcp
```

Конфликт будет только если другая служба уже использует тот же самый порт и протокол, например `443/tcp`.

---

## 1. Проверить текущие порты

Перед установкой полезно посмотреть, что уже работает на сервере:

```bash
ss -tulpn
ufw status numbered || true
docker ps || true
```

Если стоит AmneziaWG через Docker, обычно будет что-то вроде:

```text
51820/udp — AmneziaWG VPN
51821/tcp — панель amnezia-wg-easy
```

Их не трогаем.

---

## 2. Минимальная установка

Минимально для установки 3x-ui обычно достаточно:

```bash
apt update
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

`apt update` желательно оставить: он не обновляет систему, а только обновляет список пакетов.

---

## 3. Желательная подготовка

Эта команда **желательна, но не всегда обязательна**: установочный скрипт 3x-ui обычно сам доставляет часть зависимостей. Но лучше выполнить её заранее, чтобы точно были `curl`, `openssl`, `socat`, `cron`, `ca-certificates` и `ufw`.

```bash
apt update
DEBIAN_FRONTEND=noninteractive apt install -y curl wget sudo ufw nano openssl ca-certificates socat cron tar tzdata
```

---

## ⚠️ Необязательный аварийный блок

**Этот блок НЕ является обязательным для установки 3x-ui.**

**НЕ НУЖНО делать полный `apt upgrade -y` на рабочем сервере без необходимости.** Особенно если на сервере уже есть SSH-настройки, AmneziaWG или другие службы. `apt upgrade` может задавать вопросы по конфигам вроде `/etc/ssh/sshd_config` или `/etc/cloud/cloud.cfg`.

Команды ниже выполнять **только если есть проблема с apt/dpkg**: например, `dpkg was interrupted`, broken dependencies, зависшее или прерванное обновление.

```bash
dpkg --configure -a
apt install -f -y
```

Полное обновление системы выполнять только осознанно, когда действительно хочешь обновить пакеты ОС:

```bash
apt upgrade -y
```

---

## 4. Порт панели: вручную или случайный

Есть два варианта.

### Вариант A — рекомендованный: задать порт панели вручную

Проще и понятнее заранее выбрать порт панели, например:

```text
58666
```

Плюсы:

```text
1. заранее понятно, какой порт открывать в UFW;
2. проще записать URL панели;
3. меньше риска забыть открыть случайный порт;
4. меньше путаницы при повторной установке.
```

Во время установки:

```text
Would you like to customize the Panel Port settings? [y/n]: y
Please set up the panel port: 58666
```

Важно: если выбрал `y`, на втором вопросе **нельзя просто нажимать Enter**. Нужно ввести число. Иначе будет ошибка вроде:

```text
invalid value "" for flag -port
Access URL: https://IP:/WEBBASEPATH
```

### Вариант B — случайный порт

Если хочешь, чтобы скрипт сам сгенерировал порт:

```text
Would you like to customize the Panel Port settings? [y/n]: n
```

Тогда в конце установки смотри строку:

```text
Port: 60870
Access URL: https://IP:60870/WEBBASEPATH
```

Если UFW активен, этот сгенерированный порт нужно открыть уже после установки:

```bash
ufw allow 60870/tcp
ufw status
```

---

## 5. Firewall / UFW

Если `ufw status` показывает `inactive`, панель может открываться и без `ufw allow`: firewall Ubuntu просто ничего не фильтрует. Но если UFW уже активен, нужно явно открыть нужные порты.

### Если заранее выбрал порт панели 58666

Для чистого сервера:

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 58666/tcp
ufw status
```

Если на сервере уже есть AmneziaWG:

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 58666/tcp
ufw allow 51820/udp
ufw allow 51821/tcp
ufw status
```

Если UFW выключен и хочешь его включить:

```bash
ufw --force enable
ufw status verbose
```

`--force` полезен, потому что обычный `ufw enable` в SSH-сессии иногда отвечает `Aborted`.

### Зачем нужен 80/tcp

`80/tcp` нужен только для выпуска/автообновления сертификата панели через Let's Encrypt.

Если `80/tcp` закрыт, установка с вариантом **2 — Let's Encrypt for IP Address** может упасть с ошибкой:

```text
Timeout during connect (likely firewall problem)
Please ensure port 80 is reachable
```

В таком случае открой `80/tcp` и повтори установку/выпуск сертификата.

### Зачем нужен 2096/tcp

3x-ui может поднять subscription server на `2096/tcp`. Если используешь обычную `vless://` ссылку или QR, `2096/tcp` обычно не нужен. Если будешь использовать subscription-ссылки 3x-ui, открой:

```bash
ufw allow 2096/tcp
```

---

## 6. Установка 3x-ui

Запуск установщика:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Рекомендуемый вариант с фиксированным портом панели:

```text
Would you like to customize the Panel Port settings? [y/n]: y
Please set up the panel port: 58666
```

SSL-сертификат для панели:

```text
1. Let's Encrypt for Domain
2. Let's Encrypt for IP Address
3. Custom SSL Certificate
```

Если `80/tcp` открыт, выбирай:

```text
2
```

На IPv6:

```text
Do you have an IPv6 address to include? (leave empty to skip):
```

Если IPv6 не нужен, просто Enter.

На ACME listener:

```text
Port to use for ACME HTTP-01 listener (default 80):
```

Просто Enter, чтобы оставить `80`.

После установки сохрани:

```text
Username
Password
Port
WebBasePath
Access URL
```

Пример:

```text
Access URL: https://31.59.114.59:58666/WEBBASEPATH
```

Если логин/пароль где-то засветились, после входа в панель лучше сразу поменять пароль и WebBasePath.

---

## 7. Проверить работу 3x-ui

```bash
x-ui status
systemctl status x-ui --no-pager
```

Проверить, что панель слушает свой порт:

```bash
ss -tulpn | grep -E '58666|60870|2053|2096|443'
```

Посмотреть настройки панели:

```bash
x-ui settings
```

Логи:

```bash
journalctl -u x-ui -n 100 --no-pager
```

Если запустил `x-ui log`, там надо выбирать пункты меню цифрами `1`, `2`, `0`. Команду `journalctl ...` нужно запускать в обычной консоли, не внутри меню `x-ui log`.

---

## 8. Зайти в панель

Открыть в браузере:

```text
https://IP_СЕРВЕРА:PORT/WEBBASEPATH
```

Например:

```text
https://31.59.114.59:58666/WEBBASEPATH
```

---

## 9. Создать VLESS Reality XHTTP inbound без CDN

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
Expiry Time / Дата окончания: не задавать
Enable / Включить: ON
Authentication: None / пусто
Encryption: none
Decryption: none
```

Для XHTTP **не ставим** `xtls-rprx-vision`. Это было нужно для `TCP RAW + Vision`. Для XHTTP поле `Flow` оставляем пустым.

Блок транспорта:

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

`Host` пустой — это нормально для простого варианта без CDN. `Path: /` тоже нормально. Если часть расширенных XHTTP-полей отсутствует в твоей версии 3x-ui, оставляй дефолты.

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

`Target` и `SNI` должны совпадать по домену. Можно использовать другой доступный HTTPS-сайт, например:

```text
www.apple.com:443
www.apple.com

www.nvidia.com:443
www.nvidia.com
```

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

Для варианта из репозитория нажимаем `Get New Seed`. Если какой-то клиент плохо импортирует ссылку или не поддерживает ML-DSA/seed, можно потом тестировать без `mldsa65 Seed / Verify`, но сначала лучше следовать репозиторию.

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

После `Get New Cert` и `Get New Seed` **не меняй** `Target`, `SNI`, `Short IDs`, `Public Key`, `Private Key`, `mldsa65 Seed` и `mldsa65 Verify` без повторного экспорта новой ссылки в клиент. Если эти поля поменять, старая ссылка в v2rayN/v2rayNG станет неактуальной.

---

## 10. Проверить, что Xray слушает 443

```bash
ss -tulpn | grep 443
```

Нормально, если увидишь примерно:

```text
tcp LISTEN 0 4096 *:443 *:* users:(("xray-linux-amd64",pid=...,fd=...))
```

---

## 11. Создать отдельного клиента для телефона

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

Итог:

```text
1 inbound XHTTP на 443
client laptop → отдельная ссылка/QR
client phone  → отдельная ссылка/QR
```

---

## 12. Где взять QR или ссылку

В списке Inbounds нажать маленький `+` слева от строки inbound. Раскроется список клиентов.

У конкретного клиента можно открыть QR или скопировать ссылку. Если нажать на QR, в некоторых версиях 3x-ui ссылка просто копируется в буфер обмена. Это нормально.

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
sni=www.sony.com
sid=fe4fbaaf65eacb35
spx=%2F
```

Если после любых изменений нажимал `Get New Cert`, менял `Short IDs`, `SNI`, `Target` или `Seed`, нужно заново экспортировать ссылку и импортировать её в клиент.

---

## 13. Клиент для Windows

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

## 14. Клиент для Android

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

Если не уверен в архитектуре телефона:

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

## 15. XHTTP через CDN Cloudflare

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

## 16. Полезные команды управления

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
journalctl -u x-ui -n 100 --no-pager
```

Проверить порты:

```bash
ss -tulpn | grep -E '443|58666|60870|2096|x-ui|xray'
```

---

## 17. Удаление 3x-ui

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
ufw delete allow 58666/tcp
ufw delete allow 60870/tcp
ufw delete allow 443/tcp
ufw delete allow 80/tcp
ufw status
```

`22/tcp` не удалять, если через него работает SSH. Если на сервере стоит AmneziaWG, не удаляй его порты:

```text
51820/udp
51821/tcp
```

---

## 18. Короткий итог

Простой XHTTP без CDN:

```text
Protocol: VLESS
Port: 443
Transport: XHTTP
Security: Reality
Flow: пусто
uTLS: chrome
Target/SNI: www.sony.com или другой доступный HTTPS-сайт
Get New Cert: да
Get New Seed: да, если следуем репозиторию
Sniffing: ON
```

Порт панели:

```text
Лучше задать вручную, например 58666.
Если выбрал random, после установки открыть именно сгенерированный порт.
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

