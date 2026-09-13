# Файерволы в Linux: ufw, nftables, firewalld

← [README](README.md) · SSH (не закрыть себе доступ): [ssh_cheatsheet.md](ssh_cheatsheet.md) · NFS/Samba порты: [nfs_samba_cheatsheet.md](nfs_samba_cheatsheet.md)

**Файервол (межсетевой экран)** фильтрует сетевой трафик: что **разрешить** (allow), что **блокировать** (deny/drop). На VPS обычно два уровня:

1. **Облако / роутер** — Security Groups, «firewall» в панели провайдера (до вашего сервера).
2. **Хост (Linux)** — `ufw`, `firewalld` или прямо **nftables** / **iptables**.

Оба уровня должны согласовываться: открыли порт 80 в ufw, но группа безопасности в облаке его режет — сайт снаружи недоступен.

В блоках `bash` комментарии после `#` поясняют команду.

---

## 1. Какие бывают и что выбрать

| Инструмент | Где | Суть |
|------------|-----|------|
| **Панель VPS (SG)** | Hetzner, DO, AWS… | ACL по IP/портам на гипервизоре |
| **ufw** | Ubuntu, Debian desktop/server | Простой фронтенд к iptables/nft |
| **firewalld** | RHEL, Alma, Rocky, Fedora | Зоны + `firewall-cmd`, динамические правила |
| **nftables** | Современный Linux (ядро) | Наследник iptables, единый `nft` |
| **iptables** | Legacy | Старые скрипты; на новых системах часто → nft backend |
| **Router OS** | Домашний роутер | NAT + фильтр WAN→LAN |

**Практика для этого репозитория (Ubuntu‑подобные VPS):**

- Включить **Security Group** в облаке (минимум: SSH, HTTP/HTTPS по необходимости).
- На сервере — **`ufw`** с политикой **deny incoming**, явные **allow** для нужных портов.
- На RHEL‑семье — **`firewalld`** (кратко в §7).

Не полагайтесь только на облако: при смене провайдера или bare metal host‑firewall обязателен.

---

## 2. Основы: порты, направления, stateful

| Термин | Комментарий |
|--------|-------------|
| **incoming / INPUT** | Кто‑то стучится **на ваш** сервер (SSH, HTTP) |
| **outgoing / OUTPUT** | Сервер сам инициирует (apt update, DNS) — обычно разрешён |
| **forward / FORWARD** | Маршрутизация между интерфейсами (роутер, Docker, VPN) |
| **Stateful** | Разрешить ответ на уже установленное исходящее соединение (ESTABLISHED) |
| **TCP / UDP** | Правило часто указывает протокол (`22/tcp`, `53/udp`) |

Типичные порты на сервере:

| Порт | Сервис |
|------|--------|
| 22/tcp | SSH |
| 80, 443/tcp | HTTP / HTTPS |
| 2049/tcp | NFSv4 |
| 445, 139/tcp | Samba |
| 53/udp | DNS (если свой резолвер) |

Проверить, что слушает система **до** настройки firewall:

```bash
ss -tulpen                     # все слушающие порты и процессы
ss -tulpen | grep LISTEN       # только LISTEN
sudo lsof -i -P -n | grep LISTEN   # альтернатива (нужен lsof)
```

---

## 3. UFW (Uncomplicated Firewall) — основной сценарий

### Установка и первый запуск

```bash
sudo apt update                              # индексы пакетов
sudo apt install -y ufw                      # часто уже установлен на Ubuntu

sudo ufw default deny incoming               # по умолчанию блокировать входящие
sudo ufw default allow outgoing              # исходящие — разрешить
sudo ufw allow OpenSSH                       # ВАЖНО: до enable, иначе потеряете SSH
sudo ufw enable                              # включить (y — подтверждение)
sudo ufw status verbose                      # правила и политики по умолчанию
```

Профиль **`OpenSSH`** — готовое правило для порта из `/etc/services` (обычно 22). Если sshd на **2222**:

```bash
sudo ufw allow 2222/tcp comment 'SSH custom' # явный порт TCP
sudo ufw status numbered                     # номера правил для delete
```

### Разрешить и запретить

```bash
sudo ufw allow 80/tcp                        # HTTP с любого IP
sudo ufw allow 443/tcp                       # HTTPS
sudo ufw allow from 192.168.1.0/24 to any port 22   # SSH только из LAN
sudo ufw deny from 203.0.113.50              # блок IP (blacklist)
sudo ufw allow from 10.0.0.0/8 to any port 5432 proto tcp   # Postgres только из VPN

sudo ufw allow 'Nginx Full'                  # профиль из /etc/ufw/applications.d/
sudo ufw app list                            # доступные профили приложений
sudo ufw app info 'Nginx Full'               # какие порты в профиле
```

### Удаление, нумерация, лог

```bash
sudo ufw status numbered                     # [ 1] правило …
sudo ufw delete 3                            # удалить по номеру
sudo ufw delete allow 80/tcp                 # удалить по тексту правила
sudo ufw logging on                          # лог в /var/log/ufw.log
sudo ufw logging medium                      # уровень (off, low, medium, high)
sudo tail -F /var/log/ufw.log                # смотреть блокировки (нужны права)
```

### Сброс и отключение

```bash
sudo ufw disable                             # выключить firewall (осторожно)
sudo ufw reset                               # удалить все правила (осторожно на production)
sudo ufw reload                              # применить после правок /etc/ufw/*.rules
```

Конфиги: **`/etc/ufw/ufw.conf`**, **`/etc/ufw/before.rules`**, **`/etc/ufw/user.rules`** — правьте через `ufw` CLI когда возможно; после ручных правок — `ufw reload`.

### IPv6

```bash
grep IPV6 /etc/ufw/ufw.conf                  # IPV6=yes по умолчанию
sudo ufw status verbose                      # отдельные v6 правила при необходимости
sudo ufw allow from 2001:db8::/32 to any port 22   # пример v6
```

---

## 4. Безопасная последовательность на новом VPS

1. Подключиться по SSH (ключ уже работает).
2. `sudo ufw allow OpenSSH` (или ваш порт).
3. `sudo ufw default deny incoming` + `allow outgoing`.
4. Добавить 80/443 или другие сервисы.
5. **`sudo ufw enable`**.
6. **Не закрывать** текущую SSH‑сессию — открыть **вторую** и проверить вход.
7. В панели облака — те же порты в Security Group.

```bash
# проверка с другой машины (не с самого сервера):
nc -zv YOUR_IP 22                            # TCP порт доступен?
nc -zv YOUR_IP 80
# или: nmap -Pn -p 22,80,443 YOUR_IP
```

Заблокировали себя: консоль провайдера (VNC/serial), recovery, временно отключить SG в панели — см. §10.

---

## 5. nftables (современная основа netfilter)

На Ubuntu 22.04+ ufw часто управляет **nftables** backend. Прямое управление — когда нужен тонкий контроль.

```bash
sudo apt install -y nftables                 # если не установлен
sudo nft list ruleset                        # все таблицы, цепочки, правила
sudo nft list table inet filter              # если таблица есть (имя может отличаться)
sudo systemctl enable --now nftables         # загрузка ruleset при старте (Debian)
```

Пример **минимального** standalone ruleset (не смешивайте вслепую с ufw — выберите один менеджер):

```bash
sudoedit /etc/nftables.conf                  # или отдельный файл + include
```

```nft
# /etc/nftables.conf — пример; комментарии #

flush ruleset

table inet filter {
  chain input {
    type filter hook input priority 0; policy drop;
    ct state established,related accept    # ответы на исходящие
    iif "lo" accept                        # localhost
    tcp dport 22 accept                      # SSH
    tcp dport { 80, 443 } accept             # web
    # icmp type echo-request accept          # ping (опционально)
  }
  chain forward { type filter hook forward priority 0; policy drop; }
  chain output { type filter hook output priority 0; policy accept; }
}
```

```bash
sudo nft -c -f /etc/nftables.conf            # проверка синтаксиса (-c check)
sudo nft -f /etc/nftables.conf                 # загрузить (если ufw выключен!)
sudo systemctl reload nftables                 # применить unit (дистрибутив‑зависимо)
```

**iptables** (legacy) — только просмотр на многих системах:

```bash
sudo iptables -L -n -v                       # цепочки filter
sudo iptables -S                             # правила в виде команд
sudo iptables-nft -L -n -v                   # iptables через nft backend
```

Новые правила через `iptables` на системах с ufw **не рекомендуются** — ufw перезапишет или будет конфликт.

---

## 6. Docker, VPN и FORWARD (важно)

**Docker** по умолчанию манипулирует iptables/nft и может **обходить ufw** для опубликованных портов (`-p 8080:80`). Решения — отдельная тема (bind на localhost, reverse proxy, `ufw-docker`, custom DOCKER-USER chain).

**VPN / роутер** — трафик идёт через **FORWARD**; ufw по умолчанию настраивает в основном **INPUT**.

```bash
grep DEFAULT_FORWARD_POLICY /etc/default/ufw   # often DROP
sudo ufw route allow in on tun0 out on eth0    # пример маршрутизации (OpenVPN)
```

Если «в ufw открыто, снаружи не заходит» — проверьте **облако**, **Docker**, **FORWARD**.

---

## 7. firewalld (RHEL / Alma / Rocky) — кратко

```bash
sudo systemctl enable --now firewalld        # сервис firewalld
sudo firewall-cmd --state                    # running / not running
sudo firewall-cmd --get-default-zone         # обычно public
sudo firewall-cmd --list-all                 # зона, сервисы, порты

sudo firewall-cmd --permanent --add-service=ssh      # SSH
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --reload                   # применить permanent

sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.1.0/24" service name="ssh" accept'
```

Аналог ufw на Debian‑VPS — **не** firewalld, а **ufw** (§3).

---

## 8. Облачные Security Groups

Логика везде похожа:

- **Inbound** — к серверу (SSH, 443).
- **Outbound** — часто «всё разрешено» (обновления, DNS).
- Привязка правил к **интерфейсу / тегу / группе** VM.

Проверяйте **и** SG, **и** ufw. Типичная ошибка: ufw `allow 80`, в панели открыт только 22.

---

## 9. Fail2ban (дополнение, не замена firewall)

Fail2ban читает логи (SSH, nginx) и **временно банит IP** через iptables/nft **поверх** ufw.

```bash
sudo apt install -y fail2ban                 # установка
sudo systemctl enable --now fail2ban
sudo fail2ban-client status                  # активные jail
sudo fail2ban-client status sshd             # баны ssh (имя jail может отличаться)
sudo fail2ban-client set sshd unbanip 203.0.113.50   # разбанить IP
```

Конфиг: **`/etc/fail2ban/jail.local`** (не правьте только `jail.conf` — перезапишется при update).

```bash
sudoedit /etc/fail2ban/jail.local
```

```ini
# jail.local — пример
[sshd]
enabled = true
maxretry = 5
bantime = 1h
findtime = 10m
```

```bash
sudo systemctl restart fail2ban              # после правок
```

---

## 10. «Не могу зайти по SSH» после firewall

| Причина | Действие |
|---------|----------|
| ufw enable без allow SSH | Консоль провайдера → `ufw disable` или `ufw allow 22` |
| Неверный порт | `ufw allow 2222/tcp`; SG в облаке |
| Fail2ban забанил ваш IP | `fail2ban-client set sshd unbanip YOUR_IP` с другого IP |
| sshd не слушает | `systemctl status ssh`, `ss -tulpen \| grep 22` |
| Только ключ, ключ не тот | [ssh_cheatsheet.md](ssh_cheatsheet.md) |

```bash
# на сервере через out-of-band консоль:
sudo ufw status verbose
sudo ufw allow OpenSSH
sudo systemctl status ssh
sudo journalctl -u ssh -n 30 --no-pager
sudo fail2ban-client status sshd
```

---

## 11. Диагностика и тесты

```bash
sudo ufw status numbered                     # что разрешено на хосте
sudo nft list ruleset | less                 # низкоуровневые правила (если nft)
sudo journalctl -k | grep -i 'UFW\|DROP\|REJECT'   # ядро (может быть шумно)
sudo dmesg -T | tail -20                     # последние сообщения ядра

curl -I --max-time 5 http://127.0.0.1/       # сервис локально (минует внешний FW?)
curl -I --max-time 5 http://YOUR_PUBLIC_IP/  # с самого сервера — hairpin может отличаться
```

С **другого** хоста в интернете — `nmap`, `nc`, онлайн‑проверки портов.

---

## 12. Мини‑чек‑лист VPS

1. `ss -tulpen` — что реально слушает система.
2. Security Group — минимум нужные inbound.
3. `ufw default deny incoming`, `allow OpenSSH`, сервисы, `ufw enable`.
4. Вторая SSH‑сессия для проверки.
5. Fail2ban для sshd (опционально).
6. Логи: `/var/log/ufw.log`, [logs_cheatsheet.md](logs_cheatsheet.md).

---

## 13. Шпаргалка команд ufw

```bash
sudo ufw allow OpenSSH                      # SSH перед enable
sudo ufw allow 80,443/tcp                   # web
sudo ufw allow from 192.168.0.0/16          # доверенная подсеть (осторожно)
sudo ufw deny from 198.51.100.0/24          # блок подсети
sudo ufw status verbose                     # состояние
sudo ufw delete allow 80/tcp                # убрать правило
sudo ufw reload                             # применить
```

Правка системных конфигов: [vim_cheatsheet.md](vim_cheatsheet.md).
