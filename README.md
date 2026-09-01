# LinuxHelp

Набор шпаргалок и мини‑гайдов по администрированию Linux и быстрой диагностике проблем
на рабочих станциях и серверах (включая VPS).

## Структура репозитория

- `server_helthcheck_quick.txt` — супер‑краткая памятка по проверке состояния сервера.
- `server_helthcheck_full.txt` — расширенный чек‑лист для диагностики VPS/сервера.
- `seek_and_destroy.txt` — разбор реального кейса с «прожорливым» процессом
  `io.elementary.appcenter` и общий алгоритм поиска и удаления проблемных процессов/пакетов.

---

## Быстрая проверка состояния сервера

Минимальный набор команд из `server_helthcheck_quick.txt`:

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

Из `server_helthcheck_full.txt`:

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

На основе `seek_and_destroy.txt`.

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
which ПРОЦЕСС
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
   which ПРОЦЕСС
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

---

## Планы по развитию

- Добавить отдельные файлы‑шпаргалки по:
  - анализу логов (`journalctl`, `rsyslog`, `nginx`, `postgresql` и др.);
  - мониторингу в реальном времени (`htop`, `iotop`, `iftop`, `nload`);
  - типовым инцидентам (переполненный диск, упавший nginx, проблемы с DNS и т.п.).
- Перевести основные разделы на английский для удобства использования в международных командах.
