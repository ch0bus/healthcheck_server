# Мониторинг в реальном времени

← [README](README.md) · [расширенный healthcheck](server_healthcheck_full.md)

## 1. htop

```bash
htop
```

- F6 — сортировка (CPU, память)
- F3 — поиск процесса
- F9 — завершить процесс

## 2. iotop — диск

```bash
sudo iotop
sudo iotop -o   # только процессы с ненулевой I/O
```

## 3. iftop — трафик по соединениям

```bash
sudo iftop -i eth0
```

Интерфейс подставьте свой (`eth0`, `ens3`, …). Клавиши: `t`, `S` / `D`.

## 4. nload — трафик по интерфейсу

```bash
sudo nload
sudo nload eth0
```

## 5. Дополнительно

### iostat (sysstat)

```bash
iostat -x 1 10
```

### vmstat

```bash
vmstat 1 10
```

### dstat / atop / glances

Комбинированные мониторы нескольких метрик.

---

## Минимальный набор «смотреть прямо сейчас»

| Задача | Утилита |
|--------|---------|
| CPU / память / процессы | `htop` |
| Диск | `iotop`, `iostat` |
| Сеть (соединения) | `iftop` |
| Сеть (общий трафик) | `nload` |
