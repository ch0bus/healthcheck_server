# systemd: unit, timer, journal, отладка

← [README](README.md) · логи: [logs_cheatsheet.md](logs_cheatsheet.md) · bash/cron: [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md) §12 · инциденты: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)

**systemd** управляет **сервисами**, **таймерами**, **mount**, **логами** (journald) на большинстве Linux.

---

## 1. Основные команды

```bash
systemctl status nginx.service
systemctl start|stop|restart|reload nginx
systemctl enable nginx                      # автозапуск
systemctl disable nginx
systemctl is-active nginx
systemctl is-enabled nginx
systemctl list-units --type=service --state=running
systemctl --failed
systemctl daemon-reload                     # после правки unit-файлов
```

---

## 2. Где лежат unit-файлы

| Путь | Назначение |
|------|------------|
| `/lib/systemd/system/` | Пакеты (apt) |
| `/etc/systemd/system/` | Админские override |
| `~/.config/systemd/user/` | User units |

```bash
systemctl cat nginx.service                 # эффективный unit (слои)
systemctl show nginx.service -p FragmentPath -p DropInPaths
```

---

## 3. Override без правки пакета

```bash
sudo systemctl edit nginx.service           # создаст drop-in в /etc/systemd/system/
# пример drop-in:
# [Service]
# Environment=FOO=bar
# MemoryMax=512M

sudo systemctl daemon-reload
sudo systemctl restart nginx
```

`edit --full` — копия всего unit (осторожно при обновлении пакета).

---

## 4. Свой service unit (шаблон)

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My App
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/var/www/myapp
ExecStart=/usr/bin/node /var/www/myapp/server.js
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now myapp.service
journalctl -u myapp -f
```

---

## 5. Timer (вместо cron)

```ini
# /etc/systemd/system/backup.service
[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup.sh

# /etc/systemd/system/backup.timer
[Unit]
Description=Daily backup

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl enable --now backup.timer
systemctl list-timers --all
```

Сравнение с cron: [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md) §12.

---

## 6. journalctl по unit

```bash
journalctl -u nginx.service
journalctl -u nginx -b                             # текущая загрузка
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx -f
journalctl -u nginx -p err..alert
```

---

## 7. Зависимости и порядок

```bash
systemctl list-dependencies nginx.service
systemctl show nginx -p After -p Requires -p Wants
```

«Не стартует после reboot» — часто **`enabled`**, **`After=`**, каталог **`WorkingDirectory`**, права: [permissions_cheatsheet.md](permissions_cheatsheet.md).

---

## 8. User systemd (desktop)

```bash
systemctl --user status
systemctl --user enable --now myservice.service
loginctl enable-linger $USER                    # user units без login session
```

---

## 9. Отладка

```bash
systemd-analyze blame                           # кто тормозит boot
systemd-analyze critical-chain nginx.service
systemctl reset-failed
```

---

## 10. Шпаргалка

```text
status/start/enable    daemon-reload после edit
journalctl -u NAME -f    timers: list-timers
override: systemctl edit
```
