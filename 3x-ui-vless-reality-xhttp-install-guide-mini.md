# Мини-инструкция: 3x-ui + VLESS Reality XHTTP

## 1. Подготовка

```bash
apt update
```

## 2. Открыть базовые порты

```bash
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw status
```

## 3. Запуск установщика

```bash
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

Порт панели можно выбрать двумя способами.

Вариант 1 — задать вручную:

```text
Would you like to customize the Panel Port settings? [y/n]: y
Please set up the panel port: 58666
```

Тогда открыть порт панели:

```bash
ufw allow 58666/tcp
ufw status
```

Вариант 2 — случайный порт:

```text
Would you like to customize the Panel Port settings? [y/n]: n
```

После установки взять порт из финального вывода:

```text
Port: XXXXX
```

И открыть его:

```bash
ufw allow XXXXX/tcp
ufw status
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
ss -tulpn | grep -E '443|80|58666|XXXXX'
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
<img width="621" height="1919" alt="image" src="https://github.com/user-attachments/assets/c0bcb95c-901e-4cd8-98fa-c41ab9718921" />
<img width="619" height="1429" alt="image" src="https://github.com/user-attachments/assets/9690067b-07a3-4ee7-954e-55331a245261" />


