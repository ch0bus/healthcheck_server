# Расширенная проверка VPS / сервера

← [README](README.md) · [быстрая проверка](server_healthcheck_quick.md) · [мониторинг в реальном времени](realtime_monitoring_cheatsheet.md)

## Общая нагрузка

```bash
uptime
top
free -h
```

`uptime` — load average за 1, 5 и 15 минут; `top` — процессы и CPU; `free -h` — RAM и swap.

## Диски и inode

```bash
df -hT
df -ih
du -xhd1 / | sort -h
```

Проверка заполнения ФС и inode; при нехватке места — поиск крупных каталогов через `du`.

## Ресурсоёмкие процессы

```bash
ps aux --sort=-%cpu | head -10
ps aux --sort=-%mem | head -10
```

## Сервисы

```bash
systemctl --failed
systemctl list-units --type=service --state=running
systemctl status nginx    # заменить на нужный сервис
```

`systemctl --failed` быстро показывает сломанные юниты.

## Сеть и порты

```bash
ss -tulpen
ss -s
ip -br addr
ip -s link
```

## Доступность сети и DNS

```bash
ping -c 4 1.1.1.1
curl -I --max-time 10 https://example.com
dig example.com
```

## Системные ошибки и логи

```bash
journalctl -p err -b
journalctl -p warning..alert -b
dmesg -T --level=err,warn

journalctl -u nginx --since today
journalctl -u nginx -f
tail -F /var/log/syslog
```

## Аппаратные показатели (при наличии)

```bash
sensors
sudo smartctl -a /dev/sda
```

На VPS команды могут быть недоступны из‑за виртуализации.

## Практическая быстрая проверка (одним блоком)

```bash
uptime
free -h
df -hT
systemctl --failed
ss -tulpen
journalctl -p err -b --no-pager
```

## Мониторинг в реальном времени

```bash
htop
iotop
iftop
nload
```

См. [realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md).

## Минимальный набор (как отдельная памятка)

Совпадает с [server_healthcheck_quick.md](server_healthcheck_quick.md):

```bash
uptime
free -h
df -hT
systemctl --failed
ps aux --sort=-%cpu | head
ps aux --sort=-%mem | head
ss -tulpen
journalctl -p err -b
```
