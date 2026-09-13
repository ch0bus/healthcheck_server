# PostgreSQL и MySQL/MariaDB — минимум для админа

← [README](README.md) · бэкапы: [backups_cheatsheet.md](backups_cheatsheet.md) · логи: [logs_cheatsheet.md](logs_cheatsheet.md) · nginx: [nginx_cheatsheet.md](nginx_cheatsheet.md)

Кратко: **старт**, **логи**, **место на диске**, **дамп** — не тюнинг performance.

---

## 1. PostgreSQL

### Установка и статус

```bash
sudo apt install -y postgresql postgresql-contrib
systemctl status postgresql
sudo -u postgres psql -c 'SELECT version();'
```

### Подключение

```bash
sudo -u postgres psql                    # локально peer auth
psql -h 127.0.0.1 -U appuser -d appdb  # TCP
```

### Логи и диагностика

```bash
ls /var/log/postgresql/
sudo tail -F /var/log/postgresql/postgresql-*-main.log
sudo -u postgres psql -c "SELECT pid, usename, state, wait_event_type, left(query,80) FROM pg_stat_activity WHERE state <> 'idle';"
```

### Место (WAL, таблицы)

```bash
sudo du -sh /var/lib/postgresql/*/
sudo -u postgres psql -c "SELECT pg_size_pretty(pg_database_size('postgres'));"
```

Диск полон → [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md), [lvm_disk_cheatsheet.md](lvm_disk_cheatsheet.md).

### Дамп (см. также backups)

```bash
sudo -u postgres pg_dump mydb | gzip > mydb-$(date +%F).sql.gz
sudo -u postgres pg_dumpall --globals-only > globals.sql
```

---

## 2. MySQL / MariaDB

### Установка и статус

```bash
sudo apt install -y mariadb-server      # или mysql-server
sudo systemctl status mariadb
sudo mysql -e 'SELECT VERSION();'
```

### Безопасность (первый запуск)

```bash
sudo mysql_secure_installation
```

### Логи

```bash
sudo tail -F /var/log/mysql/error.log
sudo mysql -e "SHOW FULL PROCESSLIST;"
```

### Дамп

```bash
mysqldump -u root -p --single-transaction mydb | gzip > mydb-$(date +%F).sql.gz
```

---

## 3. Типовые инциденты

| Симптом | Действие |
|---------|----------|
| «Too many connections» | `pg_stat_activity` / `SHOW PROCESSLIST`, лимиты в конфиге |
| Диск `/var/lib` полон | `du`, vacuum/maintenance, архив WAL (PG) |
| Не стартует после reboot | `journalctl -u postgresql`, права на data dir [permissions_cheatsheet.md](permissions_cheatsheet.md) |
| Не пускает с nginx | слушает только localhost? `ss -tlnp | grep 5432` |

---

## 4. Шпаргалка

```text
systemctl status    psql / mysql    pg_dump / mysqldump → backups_cheatsheet
логи + pg_stat_activity / PROCESSLIST
```
