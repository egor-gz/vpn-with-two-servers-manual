# Подробная инструкция: как использовать голандский VPN, подключаясь через российский сервер (для сотового оператора это будет выглядеть как российский трафик)

## Для кого эта инструкция

Эта инструкция написана для человека, который:

- почти не работал с Linux-терминалом;
- не является сетевым инженером;
- хочет повторить рабочую схему из двух серверов;
- хочет подключаться к **российскому VPN-серверу**, но выходить в интернет с **нидерландского сервера**;
- при желании хочет сделать так, чтобы **российские IP-адреса** ходили **напрямую через российский сервер**, а всё остальное — через Нидерланды.

Инструкция максимально практическая: с пояснениями, что происходит, и с готовыми командами для копирования.

---

# Что именно мы строим

## Базовая схема

```text
[iPhone / Mac / iPad / роутер / pfSense / другой клиент]
                     |
                     | WireGuard
                     v
          [Сервер в России: входная точка]
                     |
                     | отдельный WireGuard-туннель между серверами
                     v
        [Сервер в Нидерландах: выходная точка]
                     |
                     | обычный выход в интернет
                     v
                  [Интернет]
```

## Как идёт трафик в базовой версии

```text
Клиент -> Россия -> Нидерланды -> Интернет
```

То есть:

- к VPN клиент подключается к **российскому серверу**;
- сам российский сервер не является основным выходом в интернет;
- почти весь клиентский трафик он отправляет через защищённый туннель на сервер в Нидерландах;
- уже **нидерландский сервер** выпускает этот трафик наружу.

В результате:

- VPN «входит» в России;
- внешний IP в интернете обычно будет **нидерландский**.

---

# Расширенная схема: если нужно, чтобы российские IP шли напрямую через Россию

Позже можно включить более умную логику:

```text
Если IP назначения российский -> Россия -> Интернет напрямую
Если IP назначения не российский -> Россия -> Нидерланды -> Интернет
```

То есть схема станет такой:

```text
                    +---------------------> [Интернет напрямую из РФ]
                    |
Клиент -> Россия ---+
                    |
                    +--> Нидерланды -> Интернет
```

Это удобно, когда нужно:

- чтобы часть сервисов видела **российский IP**;
- а остальной трафик шёл через Нидерланды.

---

# Важное замечание перед началом

В этой инструкции есть **две части**:

1. **Часть A** — базовый вариант: всё через Нидерланды.
2. **Часть B** — дополнительная настройка: российские IP идут напрямую через РФ.

Если вы новичок, сначала поднимите **Часть A** и убедитесь, что она работает. И только потом переходите к **Части B**.

---

# Какие серверы покупать

Для повторения этой инструкции удобно взять два VPS у VDSina:

- **Россия** — локация **Moscow**. Регистрироваться и покупать тут: https://vdsina.ru/?partner=hjvsfm2z6f27
- **Нидерланды** — локация **Amsterdam**. Регистрироваться и покупать тут: https://www.vdsina.com/?partner=34ns87t837hl

Обе ссылки реферальные, т.е. я заработаю на кофе с вашей регистрации (цена для вас такая же, как без реферальной ссылки). Если этого не хочется, можете скопировать только первую часть ссылки и перейти без реферальной программы. Для вас цена в обоих случаях будет одинаковая.

Практичный стартовый тариф для обоих серверов:

- **1 CPU / 2 GB RAM / 50 GB disk**

В зависимости от того, насколько быстрый VPN для какого числа клиентов вам нужен - может, в будущем расширитесь на более дорогие сервера. Но лично я использую пока два таких и доволен. Vdsina не позволяет переключаться с более дорогих на более дешевые сервера, поэтому советую начать с этих дешевых - а проапгрейдить при желании потом сможете.

---

# Что понадобится

- email;
- аккаунт в VDSina;
- два VPS:
  - один в России;
  - один в Нидерландах;
- доступ root к обоим серверам;
- приложение **WireGuard** на iPhone / Mac / iPad / другом клиенте.

---

# ЧАСТЬ A. БАЗОВЫЙ ВАРИАНТ: Россия -> Нидерланды

---

## Шаг 1. Зарегистрироваться в VDSina

1. Откройте сайт VDSina.
2. Зарегистрируйтесь по email.
3. Подтвердите регистрацию.
4. Войдите в панель управления.
5. Сохраните пароль от панели и доступ к email.

---

## Шаг 2. Создать VPS в России

В панели VDSina:

1. Выберите **Standard VPS/VDS**.
2. Локация: **Moscow**.
3. ОС: **Ubuntu 24.04** или **Ubuntu 22.04**.
4. Тариф: **1 CPU / 2 GB RAM / 50 GB disk**.
5. Завершите заказ.
6. После создания сохраните:
   - внешний IP сервера;
   - пароль root или способ входа.

### Назовём его так:

- **RU_SERVER_IP** — внешний IP российского сервера

---

## Шаг 3. Создать VPS в Нидерландах

Сделайте то же самое, только:

1. Локация: **Amsterdam**.
2. ОС: **Ubuntu 24.04** или **Ubuntu 22.04**.
3. Тариф: **2 CPU / 4 GB RAM / 100 GB disk**.

Сохраните:

- внешний IP сервера;
- пароль root.

### Назовём его так:

- **NL_SERVER_IP** — внешний IP нидерландского сервера

---

## Шаг 4. Подключиться к серверам по SSH

### На Mac/Linux

Подключение к российскому серверу:

```bash
ssh root@RU_SERVER_IP
```

Подключение к нидерландскому серверу:

```bash
ssh root@NL_SERVER_IP
```

Если система спросит `yes/no`, напишите:

```bash
yes
```

Потом введите пароль root.

### На Windows

Можно использовать:

- Windows Terminal / PowerShell;
- PuTTY;
- Termius.

---

## Шаг 5. Настроить нидерландский сервер

Все команды в этом разделе выполняются **на нидерландском сервере**.

### 5.1. Установить WireGuard и включить маршрутизацию

```bash
apt update
apt install -y wireguard ufw

cat >/etc/sysctl.d/99-forward.conf <<'EOF1'
net.ipv4.ip_forward=1
EOF1

sysctl --system
```

### 5.2. Сгенерировать ключ для межсерверного туннеля

```bash
umask 077
wg genkey | tee /etc/wireguard/wg-russia.key | wg pubkey > /etc/wireguard/wg-russia.pub

echo "ПУБЛИЧНЫЙ КЛЮЧ NL-СЕРВЕРА:"
cat /etc/wireguard/wg-russia.pub

echo
echo "МАРШРУТ ПО УМОЛЧАНИЮ:"
ip route show default
```

Сохраните публичный ключ — он понадобится на российском сервере.

### 5.3. Открыть порт WireGuard

```bash
ufw allow 51821/udp
ufw reload || true
```

### 5.4. Создать конфиг межсерверного туннеля на NL-сервере

Сначала пока пропустите этот шаг, если у вас ещё нет публичного ключа `wg-exit` с российского сервера. Мы вернёмся к нему после настройки российского сервера.

---

## Шаг 6. Настроить российский сервер

Все команды в этом разделе выполняются **на российском сервере**.

### 6.1. Установить необходимые пакеты

```bash
apt update
apt install -y wireguard nftables qrencode curl htop iftop nload sysstat iperf3 mtr-tiny conntrack

cat >/etc/sysctl.d/99-forward.conf <<'EOF2'
net.ipv4.ip_forward=1
EOF2

sysctl --system
```

### 6.2. Создать таблицы маршрутизации

```bash
grep -qE "^[[:space:]]*200[[:space:]]+do_exit$" /etc/iproute2/rt_tables || echo "200 do_exit" >> /etc/iproute2/rt_tables
grep -qE "^[[:space:]]*201[[:space:]]+ru_direct$" /etc/iproute2/rt_tables || echo "201 ru_direct" >> /etc/iproute2/rt_tables
```

### 6.3. Сгенерировать ключи WireGuard

```bash
umask 077
wg genkey | tee /etc/wireguard/wg-clients.key | wg pubkey > /etc/wireguard/wg-clients.pub
wg genkey | tee /etc/wireguard/wg-exit.key    | wg pubkey > /etc/wireguard/wg-exit.pub

echo "ПУБЛИЧНЫЙ КЛЮЧ wg-clients:"
cat /etc/wireguard/wg-clients.pub

echo
echo "ПУБЛИЧНЫЙ КЛЮЧ wg-exit:"
cat /etc/wireguard/wg-exit.pub

echo
echo "МАРШРУТ ПО УМОЛЧАНИЮ:"
ip route show default
```

Сохраните:

- публичный ключ `wg-clients.pub`
- публичный ключ `wg-exit.pub`

### 6.4. Открыть клиентский порт WireGuard

Если используете UFW на российском сервере:

```bash
ufw allow 51820/udp
ufw reload || true
```

### 6.5. Создать конфиги `wg-exit` и `wg-clients`

Ниже замените:

- `NL_SERVER_IP` — на внешний IP нидерландского сервера
- `NL_WG_RUSSIA_PUB` — на публичный ключ `/etc/wireguard/wg-russia.pub` с NL-сервера

```bash
WAN_IF=$(ip route show default | awk '/default/ {print $5; exit}')
WAN_GW=$(ip route show default | awk '/default/ {print $3; exit}')

cat >/etc/wireguard/wg-exit.conf <<EOF3
[Interface]
Address = 10.200.0.1/30
PrivateKey = $(cat /etc/wireguard/wg-exit.key)
Table = 200

PostUp = ip rule add fwmark 0x1 table 201 priority 100
PostUp = ip rule add from 10.100.0.0/24 table 200 priority 200
PostUp = ip route replace default via ${WAN_GW} dev ${WAN_IF} table 201
PostUp = iptables -A FORWARD -i wg-clients -o %i -j ACCEPT
PostUp = iptables -A FORWARD -i %i -o wg-clients -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

PreDown = ip rule delete fwmark 0x1 table 201 priority 100
PreDown = ip rule delete from 10.100.0.0/24 table 200 priority 200
PreDown = ip route delete default via ${WAN_GW} dev ${WAN_IF} table 201
PreDown = iptables -D FORWARD -i wg-clients -o %i -j ACCEPT
PreDown = iptables -D FORWARD -i %i -o wg-clients -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT

[Peer]
PublicKey = NL_WG_RUSSIA_PUB
Endpoint = NL_SERVER_IP:51821
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF3

cat >/etc/wireguard/wg-clients.conf <<EOF4
[Interface]
Address = 10.100.0.1/24
ListenPort = 51820
PrivateKey = $(cat /etc/wireguard/wg-clients.key)
EOF4

chmod 600 /etc/wireguard/wg-exit.conf /etc/wireguard/wg-clients.conf
```

После этого откройте конфиг и замените руками placeholders, если не подставили сразу:

```bash
nano /etc/wireguard/wg-exit.conf
```

Нужно, чтобы блок `[Peer]` выглядел так:

```ini
[Peer]
PublicKey = РЕАЛЬНЫЙ_ПУБЛИЧНЫЙ_КЛЮЧ_НИДЕРЛАНДСКОГО_СЕРВЕРА
Endpoint = РЕАЛЬНЫЙ_IP_НИДЕРЛАНДСКОГО_СЕРВЕРА:51821
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

---

## Шаг 7. Вернуться на нидерландский сервер и закончить его конфиг

Теперь у вас уже есть публичный ключ `wg-exit.pub` с российского сервера.

На нидерландском сервере замените:

- `RU_WG_EXIT_PUB` — на публичный ключ `/etc/wireguard/wg-exit.pub` с RU-сервера

```bash
WAN_IF=$(ip route show default | awk '/default/ {print $5; exit}')

cat >/etc/wireguard/wg-russia.conf <<EOF5
[Interface]
Address = 10.200.0.2/30
ListenPort = 51821
PrivateKey = $(cat /etc/wireguard/wg-russia.key)

PostUp = iptables -A FORWARD -i %i -o ${WAN_IF} -s 10.100.0.0/24 -j ACCEPT
PostUp = iptables -A FORWARD -i ${WAN_IF} -o %i -d 10.100.0.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PostUp = iptables -t nat -A POSTROUTING -s 10.100.0.0/24 -o ${WAN_IF} -j MASQUERADE

PreDown = iptables -D FORWARD -i %i -o ${WAN_IF} -s 10.100.0.0/24 -j ACCEPT
PreDown = iptables -D FORWARD -i ${WAN_IF} -o %i -d 10.100.0.0/24 -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
PreDown = iptables -t nat -D POSTROUTING -s 10.100.0.0/24 -o ${WAN_IF} -j MASQUERADE

[Peer]
PublicKey = RU_WG_EXIT_PUB
AllowedIPs = 10.100.0.0/24, 10.200.0.1/32
PersistentKeepalive = 25
EOF5

chmod 600 /etc/wireguard/wg-russia.conf
```

Если нужно, вручную откройте и замените placeholder:

```bash
nano /etc/wireguard/wg-russia.conf
```

---

## Шаг 8. Поднять межсерверный туннель

### На нидерландском сервере

```bash
wg-quick up wg-russia
systemctl enable wg-quick@wg-russia
wg show wg-russia
```

### На российском сервере

```bash
wg-quick up wg-exit
wg-quick up wg-clients
systemctl enable wg-quick@wg-exit
systemctl enable wg-quick@wg-clients
wg show
ip rule show
ip route show table 200
ip route show table 201
```

Если всё хорошо, вы увидите:

- `wg-clients`
- `wg-exit`
- недавний handshake с NL-сервером

---

## Шаг 9. Настроить базовую политику nftables на российском сервере

На базовом этапе мы пока делаем простую структуру, которая позже позволит добавлять логику «российские IP напрямую».

```bash
WAN_IF=$(ip route show default | awk '/default/ {print $5; exit}')

cat >/etc/nftables.conf <<EOF6
flush ruleset

table inet vpn_policy {
    set ru_country4 {
        type ipv4_addr
        flags interval
    }

    set manual_direct4 {
        type ipv4_addr
        flags interval
    }

    chain prerouting {
        type filter hook prerouting priority mangle; policy accept;

        iifname "wg-clients" ip daddr @ru_country4 meta mark set 0x1
        iifname "wg-clients" ip daddr @manual_direct4 meta mark set 0x1
    }

    chain forward {
        type filter hook forward priority filter; policy accept;

        ct state established,related accept

        iifname "wg-clients" oifname "wg-exit" accept
        iifname "wg-clients" oifname "${WAN_IF}" accept
        iifname "wg-exit" oifname "wg-clients" accept
        iifname "${WAN_IF}" oifname "wg-clients" ct state established,related accept
    }
}

table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;

        oifname "${WAN_IF}" ip saddr 10.100.0.0/24 masquerade
    }
}
EOF6

systemctl enable --now nftables
nft -f /etc/nftables.conf
nft list ruleset
```

На этом этапе даже если `ru_country4` и `manual_direct4` пока пустые — это нормально.

---

## Шаг 10. Сделать удобную команду для добавления новых клиентов

На российском сервере:

```bash
mkdir -p /etc/wireguard/clients
chmod 700 /etc/wireguard/clients

cat >/usr/local/sbin/wg-add-client <<'EOF7'
#!/usr/bin/env bash
set -euo pipefail

WG_CONF="/etc/wireguard/wg-clients.conf"
WG_IF="wg-clients"
CLIENTS_DIR="/etc/wireguard/clients"
SERVER_PUB="$(cat /etc/wireguard/wg-clients.pub)"
SERVER_PORT="51820"
VPN_SUBNET_BASE="10.100.0"
DEFAULT_DNS="1.1.1.1"

read -rp "Имя клиента: " CLIENT_NAME
CLIENT_NAME="$(echo "$CLIENT_NAME" | tr -cd "a-zA-Z0-9._-")"
if [[ -z "$CLIENT_NAME" ]]; then
  echo "Имя клиента не может быть пустым."
  exit 1
fi

read -rp "DNS для клиента [${DEFAULT_DNS}]: " DNS
DNS="${DNS:-$DEFAULT_DNS}"

read -rp "Публичный IP российского сервера: " ENDPOINT
if [[ -z "$ENDPOINT" ]]; then
  echo "IP сервера обязателен."
  exit 1
fi

if grep -q "# ${CLIENT_NAME}$" "$WG_CONF"; then
  echo "Клиент с таким именем уже есть."
  exit 1
fi

USED_IPS="$(grep -E "AllowedIPs *= *10\.100\.0\.[0-9]+/32" "$WG_CONF" | sed -E "s/.*10\.100\.0\.([0-9]+)\/32.*//" | sort -n || true)"
NEXT_IP=""
for i in $(seq 2 254); do
  if ! echo "$USED_IPS" | grep -qx "$i"; then
    NEXT_IP="$i"
    break
  fi
done

if [[ -z "$NEXT_IP" ]]; then
  echo "Свободные адреса в 10.100.0.0/24 закончились."
  exit 1
fi

CLIENT_IP="${VPN_SUBNET_BASE}.${NEXT_IP}"

umask 077
CLIENT_PRIV="$(wg genkey)"
CLIENT_PUB="$(printf "%s" "$CLIENT_PRIV" | wg pubkey)"
CLIENT_PSK="$(wg genpsk)"

cat >>"$WG_CONF" <<EOCFG

# ${CLIENT_NAME}
[Peer]
PublicKey = ${CLIENT_PUB}
PresharedKey = ${CLIENT_PSK}
AllowedIPs = ${CLIENT_IP}/32
PersistentKeepalive = 25
EOCFG

wg syncconf "$WG_IF" <(wg-quick strip "$WG_IF")

CLIENT_CONF="${CLIENTS_DIR}/${CLIENT_NAME}.conf"
cat >"$CLIENT_CONF" <<EOCLIENT
[Interface]
PrivateKey = ${CLIENT_PRIV}
Address = ${CLIENT_IP}/32
DNS = ${DNS}

[Peer]
PublicKey = ${SERVER_PUB}
PresharedKey = ${CLIENT_PSK}
Endpoint = ${ENDPOINT}:${SERVER_PORT}
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOCLIENT

chmod 600 "$CLIENT_CONF"

echo
echo "Client created: ${CLIENT_NAME}"
echo "VPN IP: ${CLIENT_IP}"
echo "Config file: ${CLIENT_CONF}"
echo
echo "----- BEGIN CLIENT CONFIG -----"
cat "$CLIENT_CONF"
echo "----- END CLIENT CONFIG -----"
echo
echo "QR code:"
qrencode -t ansiutf8 < "$CLIENT_CONF"
EOF7

chmod +x /usr/local/sbin/wg-add-client
```

Теперь новый клиент добавляется командой:

```bash
wg-add-client
```

---

## Шаг 11. Создать первого клиента

На российском сервере:

```bash
wg-add-client
```

Сервер спросит:

- имя клиента (`iphone`, `macbook`, `ipad`, `router`, `pfsense` и т.д.);
- DNS;
- публичный IP российского сервера.

После этого он:

- создаст peer;
- покажет QR-код;
- сохранит конфиг в `/etc/wireguard/clients/`.

### Для iPhone

1. Установите WireGuard из App Store.
2. Нажмите **Add Tunnel**.
3. Выберите **Create from QR code**.
4. Сканируйте QR-код из терминала.

### Для Mac

1. Установите WireGuard.
2. Скопируйте текст из файла:

```bash
cat /etc/wireguard/clients/ИМЯ_КЛИЕНТА.conf
```

3. Сохраните как `.conf` и импортируйте в приложение.

---

## Шаг 12. Проверить, что базовая схема работает

### На клиенте

Подключитесь к VPN и откройте сервис проверки IP.

Ожидаемо:

- клиент подключён к **российскому серверу**;
- внешний IP будет **нидерландский**.

### На российском сервере

```bash
wg show
ip rule show
ip route show table 200
```

### На нидерландском сервере

```bash
wg show wg-russia
```

### Проверить туннель Россия -> Нидерланды

На российском сервере:

```bash
ping 10.200.0.2
```

---

# ЧАСТЬ B. ДОПОЛНИТЕЛЬНО: ПУСКАТЬ РОССИЙСКИЕ IP НАПРЯМУЮ ЧЕРЕЗ РФ

Эта часть включается **после того**, как базовый вариант уже работает.

Логика будет такая:

```text
Если клиент идёт на российский IP -> трафик выходит из России напрямую
Если клиент идёт на любой другой IP -> трафик идёт через Нидерланды
```

Важно понимать:

- IP-адрес сам по себе не «говорит», что он российский;
- для этого нужен внешний список российских префиксов;
- дополнительно можно вести ручной список доменов/IP, которые всегда надо пускать напрямую через РФ.

---

## Шаг 13. Создать файлы политики и скрипты обновления на российском сервере

```bash
mkdir -p /etc/vpn-policy
mkdir -p /var/lib/vpn-policy

cat >/etc/vpn-policy/manual-direct.txt <<EOF8
# One entry per line.
# Supported:
# - IPv4 address, for example:
#   1.1.1.1
# - IPv4 CIDR, for example:
#   203.0.113.0/24
# - Hostname, for example:
#   example.com
EOF8

cat >/usr/local/sbin/vpn-ru-country-sync.sh <<'EOF9'
#!/usr/bin/env bash
set -euo pipefail

URL="https://www.ipdeny.com/ipblocks/data/aggregated/ru-aggregated.zone"
CACHE="/var/lib/vpn-policy/ru-aggregated.zone"
TMP_DL="$(mktemp)"
TMP_NFT="$(mktemp)"
trap 'rm -f "$TMP_DL" "$TMP_NFT"' EXIT

curl -fsSL "$URL" -o "$TMP_DL"

grep -Eq '^[0-9]+\.[0-9]+\.[0-9]+\.[0-9]+/[0-9]+$' "$TMP_DL"

mv "$TMP_DL" "$CACHE"

echo "flush set inet vpn_policy ru_country4" > "$TMP_NFT"

batch=()
count=0
while IFS= read -r cidr; do
    [[ "$cidr" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}/[0-9]{1,2}$ ]] || continue
    batch+=("$cidr")
    count=$((count + 1))
    if (( count >= 1000 )); then
        printf "add element inet vpn_policy ru_country4 { %s }\n" "$(IFS=,; echo "${batch[*]}")" >> "$TMP_NFT"
        batch=()
        count=0
    fi
done < "$CACHE"

if (( count > 0 )); then
    printf "add element inet vpn_policy ru_country4 { %s }\n" "$(IFS=,; echo "${batch[*]}")" >> "$TMP_NFT"
fi

nft -f "$TMP_NFT"
EOF9

cat >/usr/local/sbin/vpn-manual-direct-sync.sh <<'EOF10'
#!/usr/bin/env bash
set -euo pipefail

LIST="/etc/vpn-policy/manual-direct.txt"
TMP_NFT="$(mktemp)"
trap 'rm -f "$TMP_NFT"' EXIT

declare -a elements=()

trim() {
    local s="$1"
    s="${s#"${s%%[![:space:]]*}"}"
    s="${s%"${s##*[![:space:]]}"}"
    printf "%s" "$s"
}

is_ipv4_or_cidr() {
    [[ "$1" =~ ^([0-9]{1,3}\.){3}[0-9]{1,3}(/[0-9]{1,2})?$ ]]
}

while IFS= read -r raw || [[ -n "$raw" ]]; do
    line="${raw%%#*}"
    line="$(trim "$line")"
    [[ -z "$line" ]] && continue

    if is_ipv4_or_cidr "$line"; then
        elements+=("$line")
        continue
    fi

    mapfile -t resolved < <(getent ahostsv4 "$line" | awk '{print $1}' | sort -u)
    if [[ ${#resolved[@]} -eq 0 ]]; then
        echo "Warning: could not resolve $line" >&2
        continue
    fi

    for ip in "${resolved[@]}"; do
        elements+=("$ip")
    done
done < "$LIST"

{
    echo "flush set inet vpn_policy manual_direct4"
    if [[ ${#elements[@]} -gt 0 ]]; then
        printf "add element inet vpn_policy manual_direct4 { "
        first=1
        for e in "${elements[@]}"; do
            if [[ $first -eq 1 ]]; then
                printf "%s" "$e"
                first=0
            else
                printf ", %s" "$e"
            fi
        done
        echo " }"
    fi
} > "$TMP_NFT"

nft -f "$TMP_NFT"
EOF10

chmod +x /usr/local/sbin/vpn-ru-country-sync.sh
chmod +x /usr/local/sbin/vpn-manual-direct-sync.sh
```

---

## Шаг 14. Создать systemd-сервисы и таймеры

```bash
cat >/etc/systemd/system/vpn-ru-country-sync.service <<EOF11
[Unit]
Description=Refresh RU country prefixes into nftables
After=network-online.target nftables.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/vpn-ru-country-sync.sh
EOF11

cat >/etc/systemd/system/vpn-ru-country-sync.timer <<EOF12
[Unit]
Description=Refresh RU country prefixes daily

[Timer]
OnBootSec=1min
OnUnitActiveSec=24h
Unit=vpn-ru-country-sync.service

[Install]
WantedBy=timers.target
EOF12

cat >/etc/systemd/system/vpn-manual-direct-sync.service <<EOF13
[Unit]
Description=Refresh manual direct-via-Russia destinations into nftables
After=network-online.target nftables.service
Wants=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/sbin/vpn-manual-direct-sync.sh
EOF13

cat >/etc/systemd/system/vpn-manual-direct-sync.timer <<EOF14
[Unit]
Description=Refresh manual direct-via-Russia destinations every 5 minutes

[Timer]
OnBootSec=30s
OnUnitActiveSec=5min
Unit=vpn-manual-direct-sync.service

[Install]
WantedBy=timers.target
EOF14

systemctl daemon-reload
systemctl enable --now vpn-ru-country-sync.timer
systemctl enable --now vpn-manual-direct-sync.timer
systemctl start vpn-ru-country-sync.service
systemctl start vpn-manual-direct-sync.service
```

---

## Шаг 15. Проверить, что списки загрузились

```bash
nft list set inet vpn_policy ru_country4 | sed -n '1,20p'
nft list set inet vpn_policy manual_direct4
systemctl status vpn-ru-country-sync.service --no-pager
systemctl status vpn-manual-direct-sync.service --no-pager
```

Если всё хорошо:

- `ru_country4` будет содержать много российских подсетей;
- `manual_direct4` будет пустым или содержать ваши ручные записи.

---

## Шаг 16. Добавить ручные direct-исключения

Откройте файл:

```bash
nano /etc/vpn-policy/manual-direct.txt
```

Примеры:

```text
mos.ru
gosuslugi.ru
yandex.ru
203.0.113.0/24
198.51.100.10
```

Потом перезагрузите ручной список:

```bash
systemctl start vpn-manual-direct-sync.service
nft list set inet vpn_policy manual_direct4
```

---

## Шаг 17. Проверить, куда пойдёт трафик — напрямую через РФ или через Нидерланды

### Смотреть трафик через Нидерланды

На российском сервере:

```bash
tcpdump -tttt -qni wg-exit 'src net 10.100.0.0/24 and not dst net 10.100.0.0/24'
```

### Смотреть трафик напрямую из России

```bash
tcpdump -tttt -qni ens3 'src net 10.100.0.0/24 and not dst net 10.100.0.0/24'
```

### Проверить, входит ли конкретный IP в RU-set

```bash
nft get element inet vpn_policy ru_country4 { 77.88.44.55 }
```

Если nft возвращает элемент — значит этот IP считается российским и должен идти напрямую через РФ.

---

# Повседневные команды эксплуатации

## Добавить нового клиента

```bash
wg-add-client
```

## Посмотреть всех клиентов

```bash
wg show wg-clients
```

## Показать QR-код существующего клиента

```bash
qrencode -t ansiutf8 < /etc/wireguard/clients/ИМЯ_КЛИЕНТА.conf
```

## Динамика трафика по клиентскому интерфейсу

```bash
nload wg-clients
```

## Динамика трафика по туннелю в Нидерланды

```bash
nload wg-exit
```

## Загрузка CPU и памяти

```bash
htop
```

## CPU по ядрам

```bash
mpstat -P ALL 1
```

## Память и swap

```bash
vmstat 1
free -h
```

## Состояние туннелей

```bash
wg show
```

## Маршрутизация

```bash
ip rule show
ip route show table 200
ip route show table 201
```

## Проверка правил split-routing

```bash
nft list chain inet vpn_policy prerouting
nft list chain inet vpn_policy forward
```

## Пинг до Нидерландов по межсерверному туннелю

```bash
ping 10.200.0.2
```

## Тест скорости до Нидерландов

На NL-сервере:

```bash
iperf3 -s
```

На RU-сервере:

```bash
iperf3 -c 10.200.0.2 -P 4
iperf3 -c 10.200.0.2 -P 4 -R
```

---

# Как понимать, что всё работает правильно

## Базовый вариант

Если всё настроено правильно, то:

- клиент подключается к **российскому** серверу;
- почти весь трафик идёт **через Нидерланды**;
- внешний IP у клиента — **нидерландский**.

## Вариант с direct для российских IP

Если всё настроено правильно, то:

- трафик к российским IP и ручным direct-исключениям идёт **напрямую через Россию**;
- весь остальной трафик идёт **через Нидерланды**.

---

# Что чаще всего ломается

## 1. Неверно вставлен публичный ключ
Проверяйте `PublicKey` в конфиге с обеих сторон.

## 2. Неверно вставлен внешний IP сервера
Проверяйте `Endpoint = ...`

## 3. Забыли открыть порт в UFW/файрволе

- Россия: UDP `51820`
- Нидерланды: UDP `51821`

## 4. Интерфейс не поднялся из-за ошибки в конфиге
Проверяйте:

```bash
wg show
systemctl status wg-quick@wg-clients --no-pager
systemctl status wg-quick@wg-exit --no-pager
systemctl status wg-quick@wg-russia --no-pager
```

## 5. Сервис считается «российским», но реально сидит на не-российских IP
В таком случае он может уходить через Нидерланды. Добавьте домен/IP в `manual-direct.txt`.

---

# Итог

В результате этой инструкции у вас получается двухсерверная VPN-система:

## Вариант 1 — базовый

```text
Клиент -> Россия -> Нидерланды -> Интернет
```

## Вариант 2 — расширенный

```text
Если цель российская -> Клиент -> Россия -> Интернет
Если цель не российская -> Клиент -> Россия -> Нидерланды -> Интернет
```

Это удобная и гибкая архитектура, потому что:

- входная точка остаётся в России;
- основной выход в интернет можно держать за рубежом;
- при необходимости можно часть трафика отправлять напрямую через РФ.
