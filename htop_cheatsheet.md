# htop: интерактивный монитор процессов

← [README](README.md) · другие мониторы: [realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md) · «кто жрёт CPU»: [seek_and_destroy.md](seek_and_destroy.md)

**htop** — улучшенная замена **`top`**: цветные шкалы CPU/RAM, список процессов с прокруткой, сортировка и завершение процессов с клавиатуры (и мышью). Удобен на **desktop** и по **SSH**, когда нужно быстро понять, кто грузит систему.

| | **top** | **htop** |
|---|---------|----------|
| Управление | Мало подсказок, сортировка через `M`/`P` без меню | Меню F‑клавиш, фильтр, дерево |
| CPU | Одна строка на ядро (зависит от версии) | Полоски по ядрам + средняя |
| Завершение процесса | `k` → PID | F9, выбор сигнала |
| Установка | Обычно уже есть | Пакет `htop` |

На минимальных контейнерах иногда только `top`; на VPS/desktop — `htop` ставят отдельно.

---

## 1. Установка и запуск

```bash
sudo apt update && sudo apt install -y htop   # Debian/Ubuntu
htop                                          # полный экран; выход — F10 или q
htop -h                                       # справка по ключам командной строки
```

**Запуск с ограничением:**

```bash
htop -u www-data              # только процессы пользователя www-data
htop -p 1234,5678               # только указанные PID (удобно следить за сервисом)
htop -d 20                      # обновление каждые 2.0 с (значение ×0.1 с; по умолчанию ~1.5 с)
htop -t                         # сразу дерево процессов (как F5)
htop -s PERCENT_CPU             # стартовая сортировка (имена полей — в F6 / Setup)
```

По SSH без локали иногда «ломается» псевдографика — попробуйте `export TERM=xterm-256color` или `htop` в **tmux** ([tmux_cheatsheet.md](tmux_cheatsheet.md)) / screen.

---

## 2. Что на экране

### Верхняя панель (система)

- **CPU** — полоски по ядрам (0–100% на ядро). Красное — kernel, зелёное — user, другие цвета — nice/I/O wait (зависит от темы).
- **Mem / Swp** — занято RAM и swap; следите, чтобы swap не рос постоянно (см. [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)).
- **Tasks** — сколько процессов всего, сколько running.
- **Load average** — средняя очередь на CPU (1 / 5 / 15 мин). Сравнивайте с **числом ядер** (`nproc`): load 8 на 4 ядрах — перегруз.
- **Uptime** — время с последней загрузки.

### Список процессов (низ)

Колонки по умолчанию: **PID**, **USER**, **PRI/NI** (приоритет/nice), **VIRT/RES/SHR** (память), **S** (состояние: `R` run, `S` sleep, `D` disk wait, `Z` zombie), **%CPU**, **%MEM**, **TIME+**, **Command**.

- **RES** — реальная RAM процесса; для «кто съел память» сортируйте по **MEM** (F6).
- **VIRT** — виртуальный адресный простор (часто завышен, не путать с фактическим давлением на RAM).
- **TIME+** — суммарное CPU‑время процесса с момента старта.

Состояние **`D` (uninterruptible sleep)** часто связано с **диском** — тогда смотрите ещё `iotop` ([realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md)).

---

## 3. Клавиши (основное)

| Клавиша | Действие |
|---------|----------|
| **F1** | Справка по клавишам |
| **F2** | **Setup** — колонки, метры, цвета; сохраняется в `~/.config/htop/htoprc` |
| **F3** | **Search** — поиск по имени процесса |
| **F4** | **Filter** — показать только совпадающие (узкий список) |
| **F5** | Дерево процессов (родитель → дочерние) |
| **F6** | **SortBy** — сортировка (CPU, MEM, PID, …) |
| **F7 / F8** | Уменьшить / увеличить **nice** выделенного процесса |
| **F9** | **Kill** — выбор сигнала (SIGTERM, SIGKILL, …) |
| **F10** / **q** | Выход |
| **↑↓** | Выбор строки |
| **PgUp / PgDn** | Прокрутка списка |
| **Space** | Пометить процесс (tag) — для групповых действий |
| **u** | Фильтр по пользователю (интерактивно) |
| **H** | Показать/скрыть **потоки** процессов |
| **K** | Показать/скрыть **kernel threads** |
| **t** | Дерево (как F5) |
| **c** | Показать **полную** командную строку |
| **Shift+P / M / T** | Сортировка по CPU / MEM / TIME (быстро) |

Мышь: клик по строке — выбор; клик по заголовку колонки — сортировка (если включено в Setup).

---

## 4. F2 Setup — что имеет смысл настроить

- **Meters** — оставить CPU + Memory; при нехватке места на маленьком терминале можно убрать лишние счётчики.
- **Display options** — «Highlight large numbers», «Leave margin»; **Tree view by default** — если всегда смотрите иерархию.
- **Columns** — добавить **IO** / **RDWR** (если сборка htop поддерживает I/O колонки), **Command** vs короткое имя.
- **Colors** — тема для тёмного/светлого терминала.

После **F10** в Setup настройки пишутся в **`~/.config/htop/htoprc`** — можно переносить между машинами (dotfiles).

---

## 5. Завершение процесса (F9)

1. Выделите процесс **↑↓** (или найдите **F3**).
2. **F9** → выберите сигнал:
   - **15 SIGTERM** — «вежливо» попросить завершиться ( **сначала всегда он** ).
   - **9 SIGKILL** — немедленное убийство; процесс не успевает сохранить данные. На сервере — только если SIGTERM не помог.
   - **1 SIGHUP** — перечитать конфиг (актуально для некоторых демонов, не универсально).

Для **systemd‑сервиса** правильнее снаружи htop:

```bash
sudo systemctl stop nginx.service    # остановить сервис штатно
sudo systemctl status nginx.service  # убедиться, что не перезапустился (Restart=always)
```

Убийство только «дочернего» worker может быть бессмысленным — master породит новый. См. [seek_and_destroy.md](seek_and_destroy.md).

---

## 6. Практические сценарии

### Высокий CPU за минуту

```bash
htop
# F6 → PERCENT_CPU (или Shift+P)
# F4 → filter: имя бинарника или часть cmdline
```

Сверка с CLI (без htop):

```bash
ps aux --sort=-%cpu | head -15
```

### Кто съел память

```bash
htop
# F6 → PERCENT_MEM (Shift+M)
# c — полная командная строка (docker, java -jar …)
```

### Много мелких процессов одного родителя

**F5** / **t** — дерево; раскройте **systemd**, **containerd**, **python** parent.

### Следить только за веб‑стеком

```bash
htop -u www-data
# или
htop -p "$(pgrep -d, -f 'php-fpm|nginx')"
```

`pgrep -d,` даёт список PID через запятую для `-p`.

### Zombie (состояние Z)

Zombie **не лечится kill** — нужно, чтобы **родитель** вызвал `wait()`. Найдите родителя в дереве (**F5**), перезапустите **службу** или исправьте баг; kill zombie бесполезен.

### Низкая скорость SSH

```bash
htop -d 50                      # реже обновлять экран
# F2 → уменьшить число CPU‑метров на экране
export TERM=xterm-256color
```

### Load высокий, а CPU в htop «не забит»

Часто виноваты процессы в **`D`** (ожидание диска) или **I/O wait** (красные/оранжевые доли на CPU‑метрах — зависит от темы). В списке отсортируйте по **STATE** (F6, если колонка включена в F2) или ищите **`D`** глазами.

```bash
# параллельно с htop — кто держит диск
sudo iotop -o
iostat -x 1 5
```

Подробнее: [realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md).

### Один процесс на 100% одного ядра

На 4‑ядерном VPS «100% CPU» в htop для одного процесса — это **одно ядро**, не вся машина. **F6 → PERCENT_CPU**, **c** — полная cmdline (часто `python script.py`, `node`, `php`, компиляция).

```bash
# узнать, сколько ядер сравнивать с load
nproc
uptime
```

Подозрение на майнер/ботнет: незнакомый бинарник в `/tmp`, случайное имя, пользователь `www-data` — сверка с [seek_and_destroy.md](seek_and_destroy.md) (`readlink -f /proc/PID/exe`).

### Снизить приоритет, не убивая (бэкап, архив, rsync)

Выделите процесс → **F7** (nice +, менее приоритетно) несколько раз. Удобно, когда **tar**/**zstd**/**restic** конкурирует с продом на маленьком VPS.

Проверка снаружи:

```bash
ps -o pid,ni,cmd -p PID    # NI — nice; выше = «вежливее» к другим
renice +10 -p PID          # то же из shell (нужны права на чужой процесс → sudo)
```

### Разогнался cron / systemd timer

Симптом: каждые N минут всплеск CPU и исчезновение процесса. В htop **F4** → `backup`, `certbot`, `php`, `ansible` — поймать в момент пика. Дерево (**F5**): дочерний процесс часто висит под **`systemd`**, **`cron`**, **`run-parts`**.

```bash
systemctl list-timers --all | head -20
grep -r . /etc/cron.d /etc/cron.daily 2>/dev/null | head
journalctl -u ИМЯ.service --since "10 min ago" --no-pager
```

### Docker / podman: кто внутри контейнеров

Имена в колонке Command часто урезаны — **c** показывает полный путь. **F4** → имя образа, `docker-compose`, `containerd-shim`. Дерево: под **`containerd`** / **`dockerd`** пачка shim.

```bash
docker stats --no-stream              # лимиты и % без htop
docker top ИМЯ_КОНТЕЙНЕРА              # PID процессов контейнера → htop -p ...
htop -p "$(docker inspect -f '{{.State.Pid}}' ИМЯ_КОНТЕЙНЕРА)"
```

### База данных: postgres / mysqld

**F4** → `postgres:` или `mysqld`. Много однотипных worker — норма; **один** процесс с огромным **TIME+** и **%CPU** — долгий запрос или autovacuum/VACUUM FULL.

```bash
# PostgreSQL — активные запросы (параллельно htop)
sudo -u postgres psql -c "SELECT pid, usename, state, left(query,80) FROM pg_stat_activity WHERE state <> 'idle';"
```

Не убивайте PID postgres через **SIGKILL** без понимания — лучше `pg_cancel_backend` / `pg_terminate_backend`.

### Несколько процессов — одно действие (tag)

**Space** на нужных строках → **F9** (или **F7**/**F8** для nice) применится к помеченным. Удобно завершить пачку зависших **curl**/**wget** после ошибочного скрипта.

Снять все метки: **Shift+Space** (в некоторых версиях — снова Space на помеченных; см. **F1** Help).

### Shared‑сервер: чужие процессы

**u** → выберите пользователя (например только **`deploy`**). Или сразу:

```bash
htop -u deploy
```

Если видите чужой прожорливый процесс — **не** `kill -9` без договорённости; зафиксируйте **PID**, **USER**, **c** (cmdline) и `readlink -f /proc/PID/exe`.

### После деплоя: следить за одним сервисом

```bash
MAIN=$(systemctl show -p MainPID --value nginx.service)
htop -p "$MAIN"
# или все worker'ы php-fpm:
htop -p "$(pgrep -d, php-fpm)"
```

Обновление списка PID: выйти и запустить снова (после reload unit PID master может смениться).

### Память утекает (растёт RES минутами)

**Shift+M**, **c**, запишите PID. Через 5–10 минут RES тот же процесс вырос — утечка в приложении или кэш без лимита (Java heap, Elasticsearch).

```bash
watch -n 5 'ps -o pid,rss,cmd -p PID'   # RSS в KB; в htop RES — похожая метрика
```

На VPS с малым RAM смотрите **Swp** вверху: рост swap + высокий **%MEM** → риск OOM killer ([common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)).

### Компиляция / CI на той же машине, что и сайт

**F4** → `gcc`, `cc1plus`, `rustc`, `cargo`, `npm run build`. **F7** на всей группе (tag + nice) или перенесите сборку в CI/off‑peak. Load может быть >> `nproc` из‑за параллельного `make -j`.

### «Внезапно всё тормозит» — быстрый порядок в htop

1. **Shift+M** — кто по памяти.  
2. **Shift+P** — кто по CPU.  
3. **F5** — не размножился ли один parent (fork bomb, runaway script).  
4. Колонка **S** — есть ли **`D`** или много **`Z`**.  
5. Если виновник — systemd unit: **F9 → 15**, затем `systemctl stop`, не только kill в htop.

---

## 7. htop и top: когда что

| Задача | Инструмент |
|--------|------------|
| Быстро покликать, убить, nice | **htop** |
| Скрипт / одна строка в чат | **`top -b -n 1 \| head`** |
| Нет htop в образе | **top** или `ps` + `watch` |
| Диск, а не CPU | **iotop**, `iostat` — [realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md) |
| Разбор «прожорливого» пакета | [seek_and_destroy.md](seek_and_destroy.md) |

Снимок для лога:

```bash
top -b -n 1 | head -40 > /tmp/top-snapshot.txt
# htop в batch‑режиме классически не заменяет top -b; для отчётов — top или ps
ps aux --sort=-%cpu | head -20 >> /tmp/top-snapshot.txt
```

---

## 8. Частые ошибки

- **SIGKILL первым делом** — риск битых файлов БД и corrupted session; сначала **15**, подождать, потом **9**.
- **Load average высокий, CPU пустой** — часто **I/O** или **D** state; htop не заменяет `iotop`/`iostat`.
- **Убили процесс — он снова появился** — смотрите **systemd** / **cron** / supervisor; отключите unit, а не только PID.
- **Фильтр F4 забыли снять** — «пропали» процессы; сброс фильтра — снова **F4** и очистить или выйти/зайти.

---

## 9. Мини‑шпаргалка

```text
F3 поиск   F4 фильтр   F5 дерево   F6 сортировка   F9 kill   F10 выход
Shift+P CPU   Shift+M MEM   u пользователь   c полная cmd   H потоки
htop -u USER   htop -p PIDs   htop -t   htop -d 20
```

Диагностика сервера в связке: [server_healthcheck_quick.md](server_healthcheck_quick.md) → htop → при I/O [realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md) § iotop.
