# Установка 3x-ui + VLESS Reality на Ubuntu-сервер
source: https://github.com/ServerTechnologies/3x-ui-with-xhttp
Инструкция описывает установку **3x-ui** на Ubuntu VPS, создание подключения **VLESS + Reality + TCP + Vision** и подключение клиентов на Windows и Android.

Подходит для сценария, когда на сервере уже могут быть другие службы, например **AmneziaWG**. Главное — не пересекаться по портам и не сбрасывать firewall.

---

## 0. Что получится в итоге

Схема:

```text
Ubuntu VPS
→ 3x-ui как systemd-служба
→ Xray-core внутри 3x-ui
→ VLESS + Reality + TCP + Vision
→ порт VLESS: 443/tcp
→ порт панели: случайный или свой, например 17701/tcp
→ порт Let's Encrypt / ACME: 80/tcp
```

Клиенты:

```text
Windows: v2rayN
Android: v2rayNG
```

Если на сервере уже есть AmneziaWG, нормальная совместная схема:

```text
AmneziaWG: 51820/udp или 443/udp
3x-ui panel: 17701/tcp
VLESS Reality: 443/tcp
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

Обновить систему и восстановить `dpkg`, если ранее обновление прерывалось:

```bash
apt update
dpkg --configure -a
apt install -f -y
apt upgrade -y
```

Установить базовые пакеты:

```bash
DEBIAN_FRONTEND=noninteractive apt install -y curl wget sudo ufw nano openssl ca-certificates socat cron tar tzdata
```

Установить 3x-ui:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Во время установки отвечать так:

```text
Would you like to customize the Panel Port settings? [y/n]: y
```

Если скрипт сам сгенерирует случайный порт панели — это нормально. Сохраните его.

На выборе SSL-сертификата:

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

## 2. Открыть порты

Замените `17701` на фактический порт панели из вывода установки.

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 17701/tcp
ufw enable
ufw status
```

Если SSH работает не на `22`, сначала откройте свой SSH-порт, иначе можно потерять доступ к серверу.

Если на сервере уже стоит AmneziaWG, не выполняйте `ufw reset` и не удаляйте существующие правила.

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

## 5. Создать VLESS Reality inbound

В панели:

```text
Inbounds → Add Inbound
```

Верх формы:

```text
Remark / Примечание: vless-reality-main
Protocol / Протокол: vless
Listen IP / Мониторинг IP: оставить пустым
Port / Порт: 443
Total Traffic / Общий расход: 0
Reset Traffic / Сброс трафика: Never
Expiry Time / Дата окончания: не задавать
```

Блок клиента:

```text
Email / Name: laptop
Authentication: None
UUID: Generate / сгенерировать
Flow: xtls-rprx-vision
Encryption: none
Decryption: none
```

Если есть кнопка генерации UUID — нажать её.

Блок транспорта:

```text
Transmission / Транспорт: TCP (RAW)
Proxy Protocol: OFF
HTTP Masking / HTTP Маскировка: OFF
Sockopt: OFF
External Proxy: OFF
```

Блок безопасности:

```text
Security / Безопасность: Reality
```

Reality-настройки:

```text
Xver: 0
uTLS / Fingerprint: chrome
Target / Dest: www.nvidia.com:443
SNI / Server Names: www.nvidia.com
Max Time Diff: 0
Min Client Ver: пусто
Max Client Ver: пусто
SpiderX: /
Short IDs: a1b2c3d4e5f6a7b8
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

Поля ниже оставить пустыми:

```text
mldsa65 Seed
mldsa65 Verify
```

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
Flow: xtls-rprx-vision
Limit IP: пусто или 0
Total GB: 0
Expiry Time: не задавать
Enable: ON
```

Сохранить.

Итоговая структура:

```text
1 inbound на 443
client laptop → отдельная ссылка/QR
client phone  → отдельная ссылка/QR
```

Так проще смотреть трафик, отключать отдельное устройство и перевыпускать ссылку, если она утекла.

---

## 8. Где взять QR или ссылку

В списке Inbounds нажать маленький `+` слева от строки inbound. Раскроется список клиентов.

У конкретного клиента можно открыть QR или скопировать ссылку.

Если нажать на QR, в некоторых версиях 3x-ui ссылка просто копируется в буфер обмена. Это нормально.

Ссылка начинается так:

```text
vless://...
```

В ней должны быть примерно такие параметры:

```text
type=tcp
security=reality
fp=chrome
sni=www.nvidia.com
sid=a1b2c3d4e5f6a7b8
flow=xtls-rprx-vision
```

Если `flow=xtls-rprx-vision` отсутствует, проверьте поле `Flow` у клиента в панели.

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

Если в v2rayN выбран китайский routing mode типа `V4-绕过大陆`, для проверки лучше переключить на `Global`.

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

## 11. Полезные команды управления

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

## 12. Удаление 3x-ui

Так как 3x-ui установлен как `systemd`-служба, удаление обычно простое. Перед удалением полезно посмотреть текущие порты, чтобы случайно не трогать чужие службы, например AmneziaWG:

```bash
ss -tulpn
ufw status numbered
```

### Вариант 1: штатное удаление

Обычно достаточно встроенной команды:

```bash
x-ui uninstall
```

После этого проверь, что служба исчезла или остановлена:

```bash
systemctl status x-ui --no-pager
ss -tulpn | grep -E '443|17701|xray|x-ui'
```

Если `x-ui` не найден, а `443/tcp` и порт панели больше не слушаются `xray` / `x-ui`, значит удаление прошло нормально.

### Вариант 2: ручная очистка

Если штатная команда не сработала или нужно подчистить вручную:

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

Если использовался Let's Encrypt-сертификат по IP, можно удалить сертификат из `acme.sh`. Замени `IP_СЕРВЕРА` на реальный IP:

```bash
~/.acme.sh/acme.sh --remove -d IP_СЕРВЕРА --ecc || true
rm -rf ~/.acme.sh/IP_СЕРВЕРА_ecc
```

### Закрытие портов

Закрывай только те порты, которые точно использовались 3x-ui и больше не нужны. Например, если порт панели был `17701`:

```bash
ufw delete allow 17701/tcp
ufw delete allow 443/tcp
ufw delete allow 80/tcp
ufw status
```

Важно: `22/tcp` не удалять, если через него работает SSH. Если на сервере стоит AmneziaWG, не удаляй его UDP-порты, например `51820/udp` или `443/udp`.

Финальная проверка:

```bash
systemctl status x-ui --no-pager
ss -tulpn | grep -E '443|17701|xray|x-ui'
ufw status numbered
```

Если команды по `x-ui` показывают, что службы нет, а `ss` ничего не выводит по `xray/x-ui`, 3x-ui удалён.

---

## 13. Короткий итог

На сервере:

```text
3x-ui как systemd-служба
Xray-core внутри 3x-ui
VLESS + Reality + TCP + Vision
Порт VLESS: 443/tcp
Порт панели: например 17701/tcp
Порт сертификата: 80/tcp
```

На устройствах:

```text
Windows: v2rayN-windows-64-desktop.zip
Android: v2rayNG_arm64-v8a.apk
```

Для постоянного использования лучше делать отдельный клиент/QR на каждое устройство:

```text
laptop
phone
tablet
```

<img width="520" height="1383" alt="image" src="https://github.com/user-attachments/assets/4899f6f4-e8e1-4ac4-ab01-74794677e9da" />
<img width="525" height="1201" alt="image" src="https://github.com/user-attachments/assets/b2945fbf-2dab-436d-a445-2787e41b4a2f" />



