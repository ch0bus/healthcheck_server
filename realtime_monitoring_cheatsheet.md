# Мониторинг в реальном времени

← [README](README.md) · [расширенный healthcheck](server_healthcheck_full.md) · [htop](htop_cheatsheet.md) · [tmux](tmux_cheatsheet.md)

## 1. htop

```bash
htop
```

Клавиши, Setup, kill, дерево процессов, сценарии: **[htop_cheatsheet.md](htop_cheatsheet.md)**.

Кратко: **F6** — сортировка, **F3** — поиск, **F9** — сигнал (сначала SIGTERM), **F10** — выход.

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

## 6. watch — периодический вывод команды

```bash
watch -n 1 'ps aux --sort=-%cpu | head -15'   # каждую 1 с — топ CPU (-n интервал в секундах)
watch -n 2 df -hT                             # место на дисках
watch -n 5 'ss -s'                            # сводка сокетов
```

**Ctrl+C** — выход. **`watch -d`** — подсветка изменившихся строк (diff).

Ограничение: при **обрыве SSH** `watch` завершится. Долгий мониторинг на VPS — лучше окно в **tmux** ([tmux_cheatsheet.md](tmux_cheatsheet.md)) или `htop` / `iotop` в интерактиве.

Тот же приём в [server_healthcheck_quick.md](server_healthcheck_quick.md).

---

## Минимальный набор «смотреть прямо сейчас»

| Задача | Утилита |
|--------|---------|
| CPU / память / процессы | `htop` ([шпаргалка](htop_cheatsheet.md)) |
| Диск | `iotop`, `iostat` |
| Сеть (соединения) | `iftop` |
| Сеть (общий трафик) | `nload` |
| Периодически одна команда | `watch` |
| Сессия переживает disconnect | `tmux` на сервере |
