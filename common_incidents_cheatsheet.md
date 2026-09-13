# Типовые инциденты

← [README](README.md) · [логи](logs_cheatsheet.md) · [htop](htop_cheatsheet.md) · [seek and destroy](seek_and_destroy.md)

## 1. Переполненный диск

### Симптомы

- `No space left on device`
- Сервисы не пишут логи или падают

### Диагностика

```bash
df -hT
df -ih
du -xhd1 / | sort -h
du -xhd1 /var | sort -h
```

### Типовые места

- `/var/log` — большие `.log` / `.gz`
- `/tmp`, `/var/tmp`
- Кэши, бэкапы, дампы БД

### Быстрые действия (осторожно)

Не чистите journal и архивы логов на production без понимания последствий; при сомнении — `du -xhd1 /var/log | sort -h`, snapshot или бэкап ([backups_cheatsheet.md](backups_cheatsheet.md)).

```bash
sudo rm /var/log/*.gz              # только если понятно, что удаляете
sudo journalctl --vacuum-size=100M
```

---

## 2. nginx не стартует / упал

### Симптомы

- HTTP 502/504/5xx
- `systemctl status nginx` → failed

### Диагностика

```bash
systemctl status nginx
journalctl -u nginx -b --no-pager
sudo nginx -t
ss -tulpen | grep ':80'
ss -tulpen | grep ':443'
```

Часто: ошибка в конфиге или занят порт 80/443.

---

## 3. DNS

### Симптомы

- `ping 8.8.8.8` ок, `ping google.com` — нет
- Медленные запросы по доменам

### Диагностика

```bash
ping -c 4 8.8.8.8
ping -c 4 google.com
dig google.com
cat /etc/resolv.conf
```

Решения: DNS в настройках сети; правка `/etc/resolv.conf` может перезаписываться (systemd-resolved, NetworkManager). Проверить локальный резолвер.

---

## 4. Высокая нагрузка CPU / памяти

```bash
uptime
htop                            # см. htop_cheatsheet.md — F6, F9, load vs nproc
ps aux --sort=-%cpu | head -15
ps aux --sort=-%mem | head -15
```

Дальше — [seek_and_destroy.md](seek_and_destroy.md): `readlink -f /proc/PID/exe`, `dpkg -S`, оценка пакета, remove/restart.

---

## 5. Сервис не стартует после ребута

```bash
systemctl status ИМЯ_СЕРВИСА
journalctl -u ИМЯ_СЕРВИСА -b
systemctl is-enabled ИМЯ_СЕРВИСА
```

Проверить: `enabled`, конфиг, зависимости (сеть, диски, каталоги).

---

## 6. Чек‑лист при инциденте

1. **Симптом** — что именно не работает.
2. **Состояние:** `uptime`, `free -h`, `df -hT`, `systemctl --failed`, `ss -tulpen`.
3. **Логи:** `journalctl` и `/var/log` ([logs_cheatsheet.md](logs_cheatsheet.md)).
4. **Минимальное безопасное действие** (restart, vacuum, ротация).
5. **Зафиксировать** команды и результат.
