# Память, swap, load average и OOM

← [README](README.md) · htop: [htop_cheatsheet.md](htop_cheatsheet.md) · инциденты: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md) · логи: [logs_cheatsheet.md](logs_cheatsheet.md)

Когда сервер **тормозит**, **swap растёт**, процессы **пропадают** или в логах **Out of memory** — этот гайд дополняет [htop_cheatsheet.md](htop_cheatsheet.md) (интерактив) и healthcheck в README.

---

## 1. Load average — что значит число

```bash
uptime                              # load average: 1 / 5 / 15 минут
nproc                               # сколько логических CPU
grep -c ^processor /proc/cpuinfo    # то же альтернативой
```

**Load** — среднее число процессов в очереди на CPU **или** в uninterruptible sleep (**I/O wait**). Правило большого пальца:

| Ситуация | Оценка |
|----------|--------|
| load **<** `nproc` | CPU обычно справляется |
| load **≈** `nproc` | Почти полная загрузка |
| load **>>** `nproc` (например 16 на 4 ядрах) | Очередь, лаг, возможен I/O |

Load **высокий**, а CPU в htop **низкий** — смотрите **диск** ([disk_io_cheatsheet.md](disk_io_cheatsheet.md)), состояние **`D`** в ps/htop.

---

## 2. Память: free и /proc

```bash
free -h                             # total, used, free, available, swap
cat /proc/meminfo | head -20
vmstat 1 5                          # si/so — swap in/out (ненулевые постоянно = больно)
```

- **`available`** (в `free -h`) — сколько RAM реально можно отдать новым процессам (учитывает cache).
- **Swap used** постоянно растёт — система **давит** RAM; производительность падает.

---

## 3. Кто съел память

```bash
ps aux --sort=-%mem | head -15
ps -eo pid,user,rss,cmd --sort=-rss | head -15   # RSS в KB
smem -rkt 2>/dev/null | head                    # если установлен smem
```

В htop: **Shift+M**, колонка **RES** — [htop_cheatsheet.md](htop_cheatsheet.md) § утечки.

---

## 4. OOM killer — симптомы и логи

Ядро убивает процесс, когда RAM+swap исчерпаны.

```bash
sudo dmesg -T | grep -iE 'oom|killed process|out of memory'
sudo journalctl -k --no-pager | grep -i oom
sudo journalctl -b | grep -i 'out of memory'
zgrep -i oom /var/log/syslog* 2>/dev/null | tail -20
```

Типичная строка dmesg: `Killed process 1234 (java) total-vm:...`.

**После OOM:** понять **кто** и **почему** (лимит контейнера, утечка, мало RAM на VPS). Не только «увеличить RAM» — иногда **лимит** (`systemd` `MemoryMax`, cgroup Docker).

---

## 5. systemd: лимиты памяти

```bash
systemctl show nginx.service -p MemoryCurrent -p MemoryMax -p MemoryHigh
systemctl cat nginx.service | grep -i memory
```

Фрагмент unit (пример):

```ini
[Service]
MemoryMax=512M
MemoryHigh=400M
```

Подробнее units: [systemd_cheatsheet.md](systemd_cheatsheet.md).

---

## 6. Docker и OOM

```bash
docker stats --no-stream
docker inspect CONTAINER --format '{{.HostConfig.Memory}}'
journalctl -u docker --since "1 hour ago" | grep -i oom
```

Контейнер с лимитом RAM может быть убит **внутри** cgroup без общего OOM на хосте — смотрите `docker inspect` и логи.

[docker_vps_cheatsheet.md](docker_vps_cheatsheet.md).

---

## 7. Что делать (осторожно)

| Действие | Когда |
|----------|--------|
| Перезапустить прожорливый **сервис** | Понятный leak / пик |
| **Увеличить RAM** VPS | Постоянный дефицит, много swap |
| **swap file** на малых VPS | Краткосрочно; не замена RAM |
| `systemctl restart` после OOM | Если unit в failed |
| `echo 3 > /proc/sys/vm/drop_caches` | **Не** на prod без причины (только диагностика cache) |

Добавить swap (пример — проверьте диск):

```bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
grep -q '/swapfile' /etc/fstab || echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 8. Мини‑скрипт: алерт по памяти и load

```bash
#!/usr/bin/env bash
# ~/bin/mem-load-check.sh — exit 1 если RAM или load «плохие»
set -euo pipefail
LOAD=$(awk '{print int($1+0.5)}' /proc/loadavg)
CPUS=$(nproc)
AVAIL=$(awk '/MemAvailable/ {print int($2/1024)}' /proc/meminfo)   # MB
MIN_MB="${1:-256}"
if (( LOAD > CPUS * 2 )); then
  echo "WARN: load $LOAD > 2*cpus ($CPUS)" >&2
  exit 1
fi
if (( AVAIL < MIN_MB )); then
  echo "WARN: MemAvailable ${AVAIL}MB < ${MIN_MB}MB" >&2
  exit 1
fi
echo "OK: load=$LOAD cpus=$CPUS avail=${AVAIL}MB"
```

**Запуск:** `chmod +x ~/bin/mem-load-check.sh && ~/bin/mem-load-check.sh 512`

---

## 9. Шпаргалка

```text
uptime + nproc    free -h    vmstat 1
dmesg/journalctl -k + oom    ps --sort=-rss
htop Shift+M    disk_io если load без CPU
```
