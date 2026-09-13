# Диск и I/O: iotop, iostat, состояние D

← [README](README.md) · load/RAM: [memory_and_load_cheatsheet.md](memory_and_load_cheatsheet.md) · htop: [htop_cheatsheet.md](htop_cheatsheet.md) · мониторинг: [realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md)

**Высокий load**, **тормоза**, процессы в **`D`** — часто **диск**, а не CPU.

---

## 1. Установка

```bash
sudo apt install -y sysstat iotop
sudo systemctl enable --now sysstat    # iostat история (опционально)
```

---

## 2. iotop — кто читает/пишет

```bash
sudo iotop                              # интерактив, как top для I/O
sudo iotop -o                           # только процессы с активным I/O
sudo iotop -ao                          # накопительно, только активные
sudo iotop -b -n 3 -d 2                 # batch: 3 снимка, интервал 2 с
```

Клавиши в интерактиве: **o** — only active, **a** — accumulated, **←/→** — сортировка.

---

## 3. iostat — устройства и очередь

```bash
iostat -x 1 5                           # каждую 1 с, 5 раз; расширенные поля
iostat -dm 1 5                          # MB/s
```

Смотреть (имена полей зависят от версии):

| Поле | Смысл |
|------|--------|
| **%util** | Занятость диска (~100% = узкое место) |
| **await** | Среднее время I/O (мс) |
| **r/s, w/s** | Операций чтения/записи в сек |

---

## 4. Быстрая диагностика «диск или нет»

```bash
uptime                                  # load
vmstat 1 5                              # wa — I/O wait (%)
ps aux | awk '$8 ~ /D/ {print}'         # процессы в D state
sudo lsof +D /var/log 2>/dev/null | head   # кто держит файлы (может быть медленно)
```

---

## 5. Место на диске vs I/O

Переполнение → другие симптомы: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md) §1.  
LVM, расширение: [lvm_disk_cheatsheet.md](lvm_disk_cheatsheet.md).

```bash
df -hT
df -ih                                  # inode
iotop -o                                  # кто пишет при «место есть, но медленно»
```

---

## 6. Практика: nginx/postgres логи забивают диск

```bash
du -xhd1 /var/log | sort -h
sudo lsof /var/log/nginx/access.log     # кто держит открытым
sudo truncate -s 0 /var/log/nginx/access.log   # осторожно: потеря лога; лучше logrotate
```

---

## 7. Шпаргалка

```text
sudo iotop -o     iostat -x 1 5     vmstat → wa
load высокий + CPU низкий → disk_io + memory_and_load
```
