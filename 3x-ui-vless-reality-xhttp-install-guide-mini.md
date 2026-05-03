# Мини-инструкция: 3x-ui + VLESS Reality XHTTP

## 1. Установка 3x-ui

```bash
apt update
```

## 2. Порты

Если UFW активен и на сервере есть AmneziaWG:

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw allow 51820/udp
ufw allow 51821/tcp
ufw status
```

Если AmneziaWG нет:

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw status
```

Порт панели 3x-ui можно выбрать двумя способами:

```text
Вариант 1: задать вручную, например 58666 → заранее открыть 58666/tcp
Вариант 2: нажать Enter / выбрать random → после установки открыть сгенерированный порт из финального вывода
```

Если задаёшь порт панели вручную:

```bash
ufw allow 58666/tcp
ufw status
```

## 3. Запуск установщика

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Ответы в установщике, если хочешь свой порт панели:

```text
Would you like to customize the Panel Port settings? [y/n]: y
Please set up the panel port: 58666
```

Ответы в установщике, если хочешь случайный порт панели:

```text
Would you like to customize the Panel Port settings? [y/n]: n
```

SSL:

```text
Choose SSL certificate setup method: 2
Do you have an IPv6 address to include?: Enter
Port to use for ACME HTTP-01 listener (default 80): Enter
```

Проверка:

```bash
x-ui status
systemctl status x-ui --no-pager
ss -tulpn | grep -E '443|80|51820|51821|58666'
```

Открыть панель:

```text
https://IP_СЕРВЕРА:ПОРТ_ПАНЕЛИ/WEBBASEPATH
```

## 4. Создание подключения VLESS Reality XHTTP

```text
Inbounds → Add Inbound
```

Проставить:

```text
Remark: vless-reality-xhttp-main
Protocol: vless
Port: 443
```

Клиент:

```text
Включить: ON
```

Транспорт:

```text
Transport: XHTTP
```

Reality:

```text
Security: Reality
Target: www.sony.com:443
SNI: www.sony.com
Short IDs: одно значение, например fe4fbaaf65eacb35
```

Нажать:

```text
Get New Cert
Get New Seed
```

Sniffing:

```text
Sniffing: ON
HTTP: ON
TLS: ON
QUIC: ON
FAKEDNS: OFF
```

Создать:

```text
Create / Создать
```

Проверка:

```bash
ss -tulpn | grep 443
```

## 5. Экспорт ссылки

```text
Inbounds → маленький + слева от подключения → клиент → QR / Copy Link
```

Ссылка:

```text
vless://...
```

## 6. Клиент Windows

```text
GitHub → 2dust/v2rayN → Releases
Скачать: v2rayN-windows-64-desktop.zip
Распаковать
Запустить v2rayN.exe
Configuration → Import share links from clipboard
Set as active server
Set system proxy
Routing: Global / Proxy all
Reload
```

## 7. Клиент Android

```text
GitHub → 2dust/v2rayNG → Releases
Скачать: v2rayNG_*_arm64-v8a.apk
Если не уверен: v2rayNG_*_universal.apk
+ → Scan QR code
или
+ → Import config from clipboard
```
