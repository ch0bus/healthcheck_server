# LinuxHelp

Набор шпаргалок и мини‑гайдов по администрированию Linux и быстрой диагностике проблем
на рабочих станциях и серверах (включая VPS).

## Оглавление

- [С чего начать](#с-чего-начать)
- [Структура репозитория](#структура-репозитория)
- [Быстрая проверка состояния сервера](#быстрая-проверка-состояния-сервера)
- [Расширенная диагностика VPS](#расширенная-диагностика-vps)
- [Поиск и устранение «прожорливых» процессов](#поиск-и-устранение-прожорливых-процессов)
- [Дополнительные шпаргалки](#дополнительные-шпаргалки)
- [Индекс команд](#индекс-команд)
- [Планы по развитию](#планы-по-развитию)

> Команды с `apt`/`dpkg` рассчитаны на Debian/Ubuntu и производные. На RHEL/Alma/Rocky:
> `dnf provides /путь`, `rpm -qf /путь`, `systemctl` — по-прежнему актуален.

Лицензия: [LICENSE](LICENSE) · как дополнять: [CONTRIBUTING.md](CONTRIBUTING.md).

## С чего начать

| Маршрут | Куда идти |
|---------|-----------|
| **Новый VPS** | [linux_install_and_basics.md](linux_install_and_basics.md) → [ssh_cheatsheet.md](ssh_cheatsheet.md) → [tmux_cheatsheet.md](tmux_cheatsheet.md) → [server_healthcheck_quick.md](server_healthcheck_quick.md) → [firewall_cheatsheet.md](firewall_cheatsheet.md) → [backups_cheatsheet.md](backups_cheatsheet.md) |
| **Что-то сломалось** | [server_healthcheck_quick.md](server_healthcheck_quick.md) → [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md) → [logs_cheatsheet.md](logs_cheatsheet.md) → тематическая шпаргалка (nginx, сеть, память…) |
| **Desktop / рутина** | [bashrc_cheatsheet.md](bashrc_cheatsheet.md) → [routine_automation_scripts.md](routine_automation_scripts.md) → [ranger_cheatsheet.md](ranger_cheatsheet.md) |

## Структура репозитория

### Диагностика и инциденты

- [`server_healthcheck_quick.md`](server_healthcheck_quick.md) — супер‑краткая памятка по проверке состояния сервера.
- [`server_healthcheck_full.md`](server_healthcheck_full.md) — расширенный чек‑лист для диагностики VPS/сервера.
- [`seek_and_destroy.md`](seek_and_destroy.md) — кейс `io.elementary.appcenter` и алгоритм поиска/удаления «прожорливых» процессов.
- [`common_incidents_cheatsheet.md`](common_incidents_cheatsheet.md) — типовые инциденты.
- [`logs_cheatsheet.md`](logs_cheatsheet.md) — анализ логов.
- [`memory_and_load_cheatsheet.md`](memory_and_load_cheatsheet.md) — RAM, swap, load, OOM.
- [`disk_io_cheatsheet.md`](disk_io_cheatsheet.md) — iotop, iostat, I/O wait.
- [`network_diagnostics_cheatsheet.md`](network_diagnostics_cheatsheet.md) — DNS, маршруты, порты.

### Мониторинг и ресурсы

- [`realtime_monitoring_cheatsheet.md`](realtime_monitoring_cheatsheet.md) — мониторинг в реальном времени.
- [`htop_cheatsheet.md`](htop_cheatsheet.md) — htop: экран, клавиши, F9, сценарии на сервере.
- [`lvm_disk_cheatsheet.md`](lvm_disk_cheatsheet.md) — LVM, расширение диска.

### Сервисы и инфраструктура

- [`nginx_cheatsheet.md`](nginx_cheatsheet.md) — nginx, reverse proxy, 502/504.
- [`tls_certificates_cheatsheet.md`](tls_certificates_cheatsheet.md) — TLS, certbot, срок сертификата.
- [`databases_cheatsheet.md`](databases_cheatsheet.md) — PostgreSQL / MySQL минимум.
- [`docker_vps_cheatsheet.md`](docker_vps_cheatsheet.md) — Docker, compose, порты и ufw.
- [`systemd_cheatsheet.md`](systemd_cheatsheet.md) — unit, timer, journal, override.
- [`ssh_cheatsheet.md`](ssh_cheatsheet.md) — SSH: ключи, `sshd`, `~/.ssh/config`, scp/rsync/sftp.
- [`nfs_samba_cheatsheet.md`](nfs_samba_cheatsheet.md) — NFS и Samba: экспорт, mount, fstab, права.
- [`firewall_cheatsheet.md`](firewall_cheatsheet.md) — ufw, nftables, firewalld, SG, fail2ban.
- [`backups_cheatsheet.md`](backups_cheatsheet.md) — стратегия 3-2-1, tar/rsync, БД, restic, cron.
- [`permissions_cheatsheet.md`](permissions_cheatsheet.md) — chmod, chown, umask, namei, ACL.

### Основы, shell и текст

- [`linux_install_and_basics.md`](linux_install_and_basics.md) — установка Ubuntu‑подобных систем, ФС, ориентирование в Linux.
- [`vim_cheatsheet.md`](vim_cheatsheet.md) — Vim: режимы, правка конфигов, поиск, `.vimrc`.
- [`bashrc_cheatsheet.md`](bashrc_cheatsheet.md) — `.bashrc`, профиль, PATH, aliases, функции.
- [`bash_scripts_cheatsheet.md`](bash_scripts_cheatsheet.md) — bash‑скрипты: set -euo, trap, flock, примеры.
- [`archives_compression_cheatsheet.md`](archives_compression_cheatsheet.md) — tar, gzip/xz/zstd, zip, zgrep.
- [`awk_sed_cheatsheet.md`](awk_sed_cheatsheet.md) — sed/awk: замена, поля, логи, мини‑скрипты.
- [`regex_cheatsheet.md`](regex_cheatsheet.md) — regex с нуля: grep -E, sed, awk, bash.
- [`jq_cheatsheet.md`](jq_cheatsheet.md) — JSON: journal, API, docker logs.

### Терминал и desktop

- [`tmux_cheatsheet.md`](tmux_cheatsheet.md) — tmux: сессии, окна, панели, SSH, ~/.tmux.conf.
- [`ranger_cheatsheet.md`](ranger_cheatsheet.md) — ranger: vi‑навигация, yy/dd/pp, rifle, закладки.
- [`routine_automation_scripts.md`](routine_automation_scripts.md) — готовые скрипты: сортировка файлов, фото, рутина.

---

## Быстрая проверка состояния сервера

Минимальный набор команд из [`server_healthcheck_quick.md`](server_healthcheck_quick.md):

```bash
uptime              # нагрузка и время работы
free -h             # оперативная память и swap
df -hT              # место на дисках
systemctl --failed  # сервисы с ошибками

ps aux --sort=-%cpu | head    # процессы с высокой нагрузкой CPU
ps aux --sort=-%mem | head    # процессы, использующие память

ss -tulpen          # порты и сетевые соединения
journalctl -p err -b  # ошибки текущей загрузки
```

Этого достаточно, чтобы за 1–2 минуты понять, «что болит» у сервера.

---

## Расширенная диагностика VPS

Из [`server_healthcheck_full.md`](server_healthcheck_full.md):

### Общая нагрузка

```bash
uptime
top
free -h
```

### Диски и inode

```bash
df -hT
df -ih
du -xhd1 / | sort -h
```

### Ресурсоёмкие процессы

```bash
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10
```

### Сервисы

```bash
systemctl --failed
systemctl list-units --type=service --state=running
systemctl status nginx    # заменить nginx на нужный сервис
```

### Сеть и порты

```bash
ss -tulpen
ss -s
ip -br addr
ip -s link
```

### Доступность сети и DNS

```bash
ping -c 4 1.1.1.1
curl -I --max-time 10 https://example.com
dig example.com
```

### Системные ошибки и логи

```bash
journalctl -p err -b
journalctl -p warning..alert -b
dmesg -T --level=err,warn

journalctl -u nginx --since today
journalctl -u nginx -f
tail -F /var/log/syslog
```

### Аппаратные показатели (при наличии)

```bash
sensors
sudo smartctl -a /dev/sda
```

---

## Поиск и устранение «прожорливых» процессов

На основе [`seek_and_destroy.md`](seek_and_destroy.md).

### 1. Найти тяжёлый процесс

```bash
ps aux --sort=-%mem | head -15
ps aux --sort=-%cpu | head -15
```

Пример однострочника: показать все процессы, использующие больше 5% памяти:

```bash
ps aux | awk '$4 > 5 {print $2, $4"%", $11}' | sort -k2 -rn
```

### 2. Определить источник процесса

```bash
readlink -f /proc/PID/exe          # PID из колонки ps
tr '\0' ' ' </proc/PID/cmdline      # полная командная строка
which КОМАНДА                        # если процесс — имя из PATH
dpkg -S /путь/к/файлу
apt show ПАКЕТ
```

### 3. Разобраться с зависимостями

```bash
apt-cache depends ПАКЕТ
apt-cache rdepends ПАКЕТ
apt-cache rdepends --no-recommends ПАКЕТ
```

### 4. Безопасное удаление пакета

```bash
sudo apt remove ПАКЕТ        # удалить, оставить конфиги
sudo apt purge ПАКЕТ         # удалить вместе с конфигами
sudo apt autoremove          # удалить неиспользуемые зависимости
sudo apt clean               # очистить кэш пакетов
```

### 5. Завершение зависшего процесса

```bash
kill PID
kill -9 PID
sudo kill PID
killall ИМЯ_ПРОЦЕССА
```

### 6. Проверка автозапуска

```bash
systemctl --user list-unit-files | grep enabled
systemctl list-unit-files | grep enabled

systemctl --user status СЛУЖБА
systemctl status СЛУЖБА

systemctl --user disable СЛУЖБА
sudo systemctl disable СЛУЖБА
```

### 7. Практический алгоритм «seek and destroy»

1. Выявить тяжёлый процесс:
   ```bash
   ps aux --sort=-%mem | head -15
   ```
2. Найти его источник:
   ```bash
   readlink -f /proc/PID/exe
   dpkg -S /путь/к/файлу
   ```
3. Проверить зависимости:
   ```bash
   apt-cache rdepends ПАКЕТ
   ```
4. Оценить важность: критичен ли пакет для системы.
5. Удалить пакет:
   ```bash
   sudo apt remove ПАКЕТ
   ```
6. Убить оставшийся процесс (если ещё жив):
   ```bash
   kill PID
   ```
7. Очистить систему:
   ```bash
   sudo apt autoremove && sudo apt clean
   ```
8. Проверить автозапуск, чтобы процесс не возвращался:
   ```bash
   systemctl --user list-unit-files | grep ПАКЕТ
   ```

В описанном кейсе безопасным кандидатом на удаление оказался `pop-shop` (магазин приложений),
а не критичные системные компоненты. Meta‑пакет `pop-desktop` можно удалить без вреда системе.

Перед `kill -9` и `apt purge` на сервере: по возможности `kill PID` (SIGTERM), снапshot/бэкап,
оценка `apt-cache rdepends`. Snap/Flatpak не снимаются через `dpkg -S` — смотрите `snap list` / `flatpak list`.

---

## Дополнительные шпаргалки

| Файл | Содержание |
|------|------------|
| [`logs_cheatsheet.md`](logs_cheatsheet.md) | `journalctl`, `/var/log`, nginx/postgresql, `grep`/`zgrep` |
| [`realtime_monitoring_cheatsheet.md`](realtime_monitoring_cheatsheet.md) | `htop`, `iotop`, `iftop`, `nload`, `watch` |
| [`htop_cheatsheet.md`](htop_cheatsheet.md) | htop: F1–F10, Setup, дерево, фильтр, kill, `-u`/`-p` |
| [`common_incidents_cheatsheet.md`](common_incidents_cheatsheet.md) | Диск, nginx, DNS, нагрузка, сервис после ребута, чек‑лист инцидента |
| [`linux_install_and_basics.md`](linux_install_and_basics.md) | Установка Ubuntu/Mint/Pop, разметка, ext4/swap/LVM, FHS, shell |
| [`vim_cheatsheet.md`](vim_cheatsheet.md) | Режимы, навигация, `:s`, split, sudoedit, минимальный `.vimrc` |
| [`bashrc_cheatsheet.md`](bashrc_cheatsheet.md) | `.bashrc` vs profile, `export`, aliases, функции, `bash -n` |
| [`ssh_cheatsheet.md`](ssh_cheatsheet.md) | Ключи, sshd, config, scp/sftp/rsync, туннели, отладка |
| [`nfs_samba_cheatsheet.md`](nfs_samba_cheatsheet.md) | NFSv4 exports, Samba smb.conf, CIFS mount, ufw |
| [`firewall_cheatsheet.md`](firewall_cheatsheet.md) | ufw/nft/firewalld, VPS SG, Docker/FORWARD, SSH lockout |
| [`backups_cheatsheet.md`](backups_cheatsheet.md) | 3-2-1, tar/rsync, pg_dump/mysqldump, restic/borg, restore |
| [`archives_compression_cheatsheet.md`](archives_compression_cheatsheet.md) | tar, gzip/bzip2/xz/zstd, zip, pipes, checksum |
| [`bash_scripts_cheatsheet.md`](bash_scripts_cheatsheet.md) | set -euo, функции, trap/flock, healthcheck, retry, cron |
| [`routine_automation_scripts.md`](routine_automation_scripts.md) | Сортировка, фото, HEIC/PDF, watch-inbox, dotfiles, trash |
| [`awk_sed_cheatsheet.md`](awk_sed_cheatsheet.md) | sed s///, awk $1/$NF, nginx/df/ps, awk -f скрипты |
| [`ranger_cheatsheet.md`](ranger_cheatsheet.md) | ranger: h/j/k/l, буфер, :cd, rifle, ranger_cd, sudo |
| [`tmux_cheatsheet.md`](tmux_cheatsheet.md) | Prefix, detach/attach, splits, copy mode, сценарии на VPS |
| [`regex_cheatsheet.md`](regex_cheatsheet.md) | BRE/ERE, grep/sed/awk, логи nginx/auth, мини‑скрипты |
| [`memory_and_load_cheatsheet.md`](memory_and_load_cheatsheet.md) | load vs nproc, swap, OOM, dmesg/journalctl -k |
| [`disk_io_cheatsheet.md`](disk_io_cheatsheet.md) | iotop, iostat, I/O wait, состояние D |
| [`lvm_disk_cheatsheet.md`](lvm_disk_cheatsheet.md) | pvs/lvs, growpart, lvextend, resize2fs |
| [`nginx_cheatsheet.md`](nginx_cheatsheet.md) | sites-enabled, nginx -t, proxy, 502/504 |
| [`tls_certificates_cheatsheet.md`](tls_certificates_cheatsheet.md) | certbot, openssl s_client, renew |
| [`databases_cheatsheet.md`](databases_cheatsheet.md) | postgres/mysql status, dump, логи |
| [`docker_vps_cheatsheet.md`](docker_vps_cheatsheet.md) | compose, 127.0.0.1 bind, logs, ufw |
| [`systemd_cheatsheet.md`](systemd_cheatsheet.md) | unit, timer, systemctl edit, journalctl -u |
| [`permissions_cheatsheet.md`](permissions_cheatsheet.md) | chmod, chown, namei, www-data/nginx |
| [`network_diagnostics_cheatsheet.md`](network_diagnostics_cheatsheet.md) | ip, ss, dig, curl, SG+ufw |
| [`jq_cheatsheet.md`](jq_cheatsheet.md) | JSON, journalctl -o json, docker logs |

Полный кейс и таблицы команд: [`seek_and_destroy.md`](seek_and_destroy.md).

---

## Индекс команд

Краткий список **типичных команд на VPS**; утилиты вроде `awk`, `ranger`, `restic`, ufw — в [таблице шпаргалок](#дополнительные-шпаргалки) выше.

### Системное состояние

- `uptime`
- `top`
- `htop`
- `free -h`
- `vmstat`

### Диски и файловые системы

- `df -hT`
- `df -ih`
- `du -xhd1 / | sort -h`
- `du -xhd1 /var | sort -h`
- `iostat -x`
- `ls /var/log`

### Терминал (tmux)

- `tmux`
- `tmux new -A -s ИМЯ`
- `tmux ls`
- `tmux attach -t ИМЯ`

### Мониторинг (watch)

- `watch -n 1 'ps aux --sort=-%cpu | head -15'`

### Процессы

- `ps aux --sort=-%cpu | head`
- `ps aux --sort=-%mem | head`
- `ps aux | awk '$4 > 5 {print $2, $4"%", $11}' | sort -k2 -rn`
- `kill`, `kill -9`, `killall`
- `readlink -f /proc/PID/exe`
- `which`

### Сервисы и автозапуск

- `systemctl --failed`
- `systemctl list-units --type=service --state=running`
- `systemctl status ИМЯ_СЕРВИСА`
- `systemctl is-enabled ИМЯ_СЕРВИСА`
- `systemctl --user list-unit-files | grep enabled`
- `systemctl list-unit-files | grep enabled`
- `systemctl --user disable СЛУЖБА`
- `sudo systemctl disable СЛУЖБА`

### Пакеты и зависимости (APT)

- `dpkg -S /путь/к/файлу`
- `apt show ПАКЕТ`
- `apt-cache depends ПАКЕТ`
- `apt-cache rdepends ПАКЕТ`
- `apt-cache rdepends --no-recommends ПАКЕТ`
- `sudo apt remove ПАКЕТ`
- `sudo apt purge ПАКЕТ`
- `sudo apt autoremove`
- `sudo apt clean`

### Сеть и DNS

- `ss -tulpen`
- `ss -s`
- `ip -br addr`
- `ip -s link`
- `ping -c 4 1.1.1.1`
- `ping -c 4 ДОМЕН`
- `curl -I --max-time 10 URL`
- `dig ДОМЕН`
- `iftop`
- `nload`

### Логи и диагностика

- `journalctl`
- `journalctl -b`
- `journalctl -p err -b`
- `journalctl -p warning..alert -b`
- `journalctl -u nginx`
- `journalctl -u ИМЯ_СЕРВИСА -b`
- `journalctl --since today`
- `tail -F /var/log/syslog`
- `grep`, `grep -E`, `zgrep`
- `less`
- `dmesg -T --level=err,warn`
- `logrotate -d /etc/logrotate.conf`

### Аппаратные показатели

- `sensors`
- `sudo smartctl -a /dev/sda`
- `iotop`

---

## Планы по развитию

- При необходимости — перевод основных разделов на английский для международных команд.
- Углублённые темы по желанию: Kubernetes, Git на сервере, hardening CIS, PostgreSQL tuning.
