# Шпаргалка по анализу логов

← [README](README.md) · awk/sed: [awk_sed_cheatsheet.md](awk_sed_cheatsheet.md) · инциденты: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)

## 1. journalctl (systemd)

### Базовые команды

```bash
journalctl                # все логи с начала ведения
journalctl -b             # логи только текущей загрузки
journalctl -b -1          # предыдущая загрузка
journalctl -f             # "tail" для journalctl (онлайн‑просмотр)
journalctl --since today  # с начала текущего дня
journalctl --since "2024-01-01" --until "2024-01-02 03:00"
```

### Фильтрация по сервису, юниту, приоритету

```bash
journalctl -u nginx             # логи сервиса nginx
journalctl -u nginx -f          # следить за логами nginx в реальном времени
journalctl -u ssh --since -1h   # ssh за последний час

journalctl -p err -b            # только ошибки текущей загрузки
journalctl -p warning..alert -b # предупреждения и выше
```

### Фильтрация по полям

```bash
journalctl _PID=1234
journalctl _UID=1000
journalctl SYSLOG_IDENTIFIER=cron
```

---

## 2. Классические текстовые логи

Обычно в `/var/log`:

- `/var/log/syslog` или `/var/log/messages` — общесистемные
- `/var/log/auth.log` — авторизация и SSH
- `/var/log/kern.log` — ядро

### Просмотр

```bash
tail -n 100 /var/log/syslog
tail -F /var/log/syslog
less +G /var/log/syslog
```

### Поиск

```bash
grep -i "error" /var/log/syslog
zgrep -i "oom" /var/log/syslog.1.gz   # см. archives_compression_cheatsheet.md
grep -i -C3 "failed" /var/log/auth.log
```

---

## 3. nginx

Пути по умолчанию (могут отличаться):

- `/var/log/nginx/access.log`
- `/var/log/nginx/error.log`

```bash
tail -F /var/log/nginx/error.log

# 10 самых частых 404
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head

# 10 самых "шумных" IP
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

---

## 4. PostgreSQL

Логи часто в `/var/log/postgresql/` или в `PGDATA`.

```bash
ls /var/log/postgresql
tail -F /var/log/postgresql/postgresql-*.log
grep -i "ERROR" /var/log/postgresql/postgresql-*.log
```

Полезные параметры в `postgresql.conf`: `log_min_duration_statement`, `log_statement`, `log_line_prefix`.

---

## 5. rsyslog и ротация

- `/etc/rsyslog.conf`, `/etc/rsyslog.d/*.conf`
- `/etc/logrotate.conf`, `/etc/logrotate.d/*`

```bash
logrotate -d /etc/logrotate.conf   # тестовый прогон, без изменений
```
