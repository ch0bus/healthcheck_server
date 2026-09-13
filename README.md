# LinuxHelp

Набор шпаргалок и мини‑гайдов по администрированию Linux и быстрой диагностике проблем
на рабочих станциях и серверах (включая VPS).

## Оглавление

- [Структура репозитория](#структура-репозитория)
- [Быстрая проверка состояния сервера](#быстрая-проверка-состояния-сервера)
- [Расширенная диагностика VPS](#расширенная-диагностика-vps)
- [Поиск и устранение «прожорливых» процессов](#поиск-и-устранение-прожорливых-процессов)
- [Дополнительные шпаргалки](#дополнительные-шпаргалки)
- [Индекс команд](#индекс-команд)
- [Планы по развитию](#планы-по-развитию)

> Команды с `apt`/`dpkg` рассчитаны на Debian/Ubuntu и производные. На RHEL/Alma/Rocky:
> `dnf provides /путь`, `rpm -qf /путь`, `systemctl` — по-прежнему актуален.

## Структура репозитория

- [`server_healthcheck_quick.md`](server_healthcheck_quick.md) — супер‑краткая памятка по проверке состояния сервера.
- [`server_healthcheck_full.md`](server_healthcheck_full.md) — расширенный чек‑лист для диагностики VPS/сервера.
- [`seek_and_destroy.md`](seek_and_destroy.md) — кейс `io.elementary.appcenter` и алгоритм поиска/удаления «прожорливых» процессов.
- [`logs_cheatsheet.md`](logs_cheatsheet.md) — анализ логов.
- [`realtime_monitoring_cheatsheet.md`](realtime_monitoring_cheatsheet.md) — мониторинг в реальном времени.
- [`common_incidents_cheatsheet.md`](common_incidents_cheatsheet.md) — типовые инциденты.
- [`linux_install_and_basics.md`](linux_install_and_basics.md) — установка Ubuntu‑подобных систем, ФС, ориентирование в Linux.
- [`vim_cheatsheet.md`](vim_cheatsheet.md) — Vim: режимы, правка конфигов, поиск, `.vimrc`.
- [`bashrc_cheatsheet.md`](bashrc_cheatsheet.md) — `.bashrc`, профиль, PATH, aliases, функции.
- [`ssh_cheatsheet.md`](ssh_cheatsheet.md) — SSH: ключи, `sshd`, `~/.ssh/config`, scp/rsync/sftp.

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
| [`common_incidents_cheatsheet.md`](common_incidents_cheatsheet.md) | Диск, nginx, DNS, нагрузка, сервис после ребута, чек‑лист инцидента |
| [`linux_install_and_basics.md`](linux_install_and_basics.md) | Установка Ubuntu/Mint/Pop, разметка, ext4/swap/LVM, FHS, shell |
| [`vim_cheatsheet.md`](vim_cheatsheet.md) | Режимы, навигация, `:s`, split, sudoedit, минимальный `.vimrc` |
| [`bashrc_cheatsheet.md`](bashrc_cheatsheet.md) | `.bashrc` vs profile, `export`, aliases, функции, `bash -n` |
| [`ssh_cheatsheet.md`](ssh_cheatsheet.md) | Ключи, sshd, config, scp/sftp/rsync, туннели, отладка |

Полный кейс и таблицы команд: [`seek_and_destroy.md`](seek_and_destroy.md).

---

## Индекс команд

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
- `grep`, `zgrep`
- `less`
- `dmesg -T --level=err,warn`
- `logrotate -d /etc/logrotate.conf`

### Аппаратные показатели

- `sensors`
- `sudo smartctl -a /dev/sda`
- `iotop`

---

## Планы по развитию

- Раздел по OOM и нехватке памяти (`dmesg`, `journalctl -k`, `grep -i oom`).
- Краткий блок про firewall (`ufw`, `nft`, `iptables -L`) и «не могу зайти по SSH».
- Интерпретация load average относительно `nproc`.
- При необходимости — перевод основных разделов на английский для международных команд.
