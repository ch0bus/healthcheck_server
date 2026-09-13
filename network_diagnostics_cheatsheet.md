# Сеть: диагностика DNS, маршруты, порты

← [README](README.md) · SSH: [ssh_cheatsheet.md](ssh_cheatsheet.md) · firewall: [firewall_cheatsheet.md](firewall_cheatsheet.md) · nginx: [nginx_cheatsheet.md](nginx_cheatsheet.md) · инциденты: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)

**Ping есть, сайт нет** / **curl по IP работает, по домену нет** / **порт закрыт** — пошаговая диагностика.

---

## 1. Интерфейсы и адреса

```bash
ip -br addr
ip route show
ip -s link
hostname -I
```

---

## 2. Порты и сокеты

```bash
ss -tulpen
ss -tulpen | grep ':443'
ss -s
sudo lsof -iTCP -sTCP:LISTEN -P -n
```

Сервис слушает **127.0.0.1** only — снаружи не доступен (часто правильно за nginx).

---

## 3. Достижимость

```bash
ping -c 4 1.1.1.1                           # ICMP (может быть заблокирован — не единственный тест)
curl -I --max-time 10 http://127.0.0.1/
curl -I --max-time 10 https://example.com/
nc -zv example.com 443                      # TCP (если есть nc)
traceroute -n 8.8.8.8                       # или mtr
```

---

## 4. DNS

```bash
dig example.com +short
dig example.com A
dig @1.1.1.1 example.com                    # другой резолвер
resolvectl status                           # systemd-resolved
cat /etc/resolv.conf
```

Симптомы DNS: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md) §3.

---

## 5. «Локально OK, снаружи нет»

1. `ss -tulpen` — слушает ли на `0.0.0.0` / `*`?
2. `sudo ufw status` — [firewall_cheatsheet.md](firewall_cheatsheet.md)
3. Security Group / firewall **облака**
4. Docker publish — [docker_vps_cheatsheet.md](docker_vps_cheatsheet.md)
5. nginx `server_name` / TLS — [nginx_cheatsheet.md](nginx_cheatsheet.md), [tls_certificates_cheatsheet.md](tls_certificates_cheatsheet.md)

```bash
curl -I http://YOUR_PUBLIC_IP/              # с самого сервера (hairpin может отличаться)
```

---

## 6. MTU / VPN (кратко)

```bash
ip link show | grep mtu
ping -M do -s 1472 8.8.8.8                  # фрагментация; уменьшать -s при «не пингуется большой пакет»
```

---

## 7. Мини‑скрипт check-endpoint

```bash
#!/usr/bin/env bash
# ~/bin/check-url.sh URL
set -euo pipefail
url="${1:?url}"
code=$(curl -o /dev/null -s -w '%{http_code}' --max-time 15 "$url")
echo "$url → HTTP $code"
[[ "$code" =~ ^[23] ]] || exit 1
```

**Запуск:** `~/bin/check-url.sh https://example.com`

---

## 8. Шпаргалка

```text
ip a / ip route    ss -tulpen    dig
127.0.0.1 vs 0.0.0.0    ufw + cloud SG
curl -vI    nc -zv host port
```
