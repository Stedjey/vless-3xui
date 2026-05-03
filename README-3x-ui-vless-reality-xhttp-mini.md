# Мини-инструкция: 3x-ui + VLESS Reality XHTTP

## 1. Установка 3x-ui

```bash
apt update
```

Если UFW уже активен и на сервере есть AmneziaWG:

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 58666/tcp
ufw allow 51820/udp
ufw allow 51821/tcp
ufw status
```

Если AmneziaWG нет:

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 58666/tcp
ufw status
```

Установка:

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Ответы в установщике:

```text
Would you like to customize the Panel Port settings? [y/n]: y
Please set up the panel port: 58666

Choose SSL certificate setup method: 2

Do you have an IPv6 address to include?:
Enter

Port to use for ACME HTTP-01 listener (default 80):
Enter
```

Проверка:

```bash
x-ui status
systemctl status x-ui --no-pager
ss -tulpn | grep -E '58666|443|80|51820|51821'
```

Открыть панель:

```text
https://IP_СЕРВЕРА:58666/WEBBASEPATH
```

---

## 2. Создание подключения VLESS Reality XHTTP

Панель:

```text
Inbounds → Add Inbound
```

Основное:

```text
Remark: vless-reality-xhttp
Protocol: vless
Listen IP: пусто
Port: 443
Total Traffic: 0
Expiry Time: не задавать
```

Клиент:

```text
Email: laptop
UUID: Generate
Flow: пусто
Encryption: none
Decryption: none
```

Транспорт:

```text
Transport: XHTTP
Host: пусто
Path: /
Mode: auto
Padding Bytes: 100-1000
Padding Obfs Mode: OFF
Uplink HTTP Method: Default (POST)
Session Placement: Default (path)
Sequence Placement: Default (path)
No SSE Header: OFF
Sockopt: OFF
External Proxy: OFF
```

Безопасность:

```text
Security: Reality
Xver: 0
uTLS / Fingerprint: chrome
Target / Dest: www.sony.com:443
SNI / Server Names: www.sony.com
Max Time Diff: 0
Min Client Ver: пусто
Max Client Ver: пусто
SpiderX: /
Short IDs: сгенерировать или указать одно значение
```

Нажать:

```text
Get New Cert
Get New Seed
```

Должны заполниться:

```text
Public Key
Private Key
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

Проверка на сервере:

```bash
ss -tulpn | grep 443
```

---

## 3. Экспорт ссылки

```text
Inbounds → маленький + слева от подключения → клиент → QR / Copy Link
```

Ссылка должна начинаться с:

```text
vless://
```

И содержать:

```text
type=xhttp
security=reality
sni=www.sony.com
sid=...
pbk=...
```

---

## 4. Клиент Windows

Скачать:

```text
GitHub → 2dust/v2rayN → Releases
v2rayN-windows-64-desktop.zip
```

Установка:

```text
Распаковать архив
Запустить v2rayN.exe
```

Импорт:

```text
Configuration → Import share links from clipboard
Set as active server
Set system proxy
Routing: Global / Proxy all
Reload
```

---

## 5. Клиент Android

Скачать:

```text
GitHub → 2dust/v2rayNG → Releases
v2rayNG_*_arm64-v8a.apk
```

Если не уверен в архитектуре:

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
