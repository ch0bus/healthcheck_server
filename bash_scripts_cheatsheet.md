# Bash‑скрипты: шпаргалка с примерами

← [README](README.md) · интерактивный shell: [bashrc_cheatsheet.md](bashrc_cheatsheet.md) · cron/бэкапы: [backups_cheatsheet.md](backups_cheatsheet.md) · vim: [vim_cheatsheet.md](vim_cheatsheet.md)

**Скрипт** — файл с командами bash, который можно запускать повторно (cron, systemd, deploy). В скрипте **не работают** aliases из `.bashrc` — только полные команды и функции, объявленные в самом скрипте.

В примерах ниже — комментарии `#` внутри кода; shebang `#!/bin/bash` указывает интерпретатор.

---

## 1. Каркас «правильного» скрипта

```bash
#!/bin/bash
# my-script.sh — краткое описание

set -euo pipefail                           # см. §2
IFS=$'\n\t'                                 # безопаснее split по пробелам

readonly SCRIPT_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"   # каталог скрипта
readonly LOG_FILE="/var/log/my-script.log"

log() { printf '[%s] %s\n' "$(date '+%F %T')" "$*" | tee -a "$LOG_FILE"; }

main() {
  log "start"
  # ...
  log "done"
}

main "$@"
```

```bash
chmod +x my-script.sh                       # право на выполнение
./my-script.sh                              # запуск из текущего каталога
bash my-script.sh                           # явно через bash (без +x)
bash -n my-script.sh                        # проверка синтаксиса
shellcheck my-script.sh                     # линтер (sudo apt install shellcheck)
```

---

## 2. `set -euo pipefail` — зачем

| Опция | Эффект |
|-------|--------|
| `-e` | Выход при ненулевом коде команды (есть исключения: `if`, `\|\|`) |
| `-u` | Ошибка при обращении к несуществующей переменной |
| `-o pipefail` | Пайп возвращает ошибку, если упала **любая** команда в цепочке |

```bash
set -e                                        # включить по отдельности при обучении
set -u
set -o pipefail

# временно отключить -e для одной команды:
set +e
might_fail || true
set -e

# или:
might_fail || log "ignored failure"
```

---

## 3. Переменные, кавычки, подстановки

```bash
name="deploy"                                 # без пробелов вокруг =
path="/var/www/$name"                         # подстановка переменной
path="/var/www/${name}_site"                  # явные границы {}

now=$(date +%F)                               # вывод команды в переменную
files=$(ls /etc/*.conf 2>/dev/null | wc -l)   # stderr в /dev/null

echo "$path"                                  # всегда в кавычках при использовании
echo '${name}'                                # одинарные — без подстановки

readonly BACKUP_DIR="/backup"                 # нельзя переприсвоить
export BACKUP_DIR                             # дочерние процессы увидят

(( count = 5 + 1 ))                             # арифметика bash
(( count++ ))
```

**Массивы:**

```bash
servers=(web1 web2 db1)
echo "${servers[0]}"                          # первый элемент
echo "${#servers[@]}"                         # число элементов
for s in "${servers[@]}"; do echo "$s"; done  # кавычки обязательны
```

**Аргументы скрипта:**

```bash
echo "$0"                                     # имя скрипта
echo "$1" "$2"                                # первый и второй аргумент
echo "$#"                                     # количество аргументов
echo "$@"                                     # все аргументы как список
shift                                         # сдвинуть $1.. (осталось $#-1)

if [[ $# -lt 1 ]]; then
  echo "Usage: $0 <hostname>" >&2
  exit 1
fi
```

---

## 4. Условия: `[[ ]]`, `test`, `case`

```bash
[[ -f /etc/nginx/nginx.conf ]] && echo "file exists"    # обычный файл
[[ -d /var/www ]] && echo "dir exists"
[[ -x /usr/bin/nginx ]] && echo "executable"
[[ -z "${VAR:-}" ]] && echo "VAR empty or unset"
[[ -n "$VAR" ]] && echo "VAR non-empty"
[[ "$a" == "$b" ]]                            # строки (в [[ ]] не escape для ==)
[[ "$n" -gt 10 ]]                             # числа: -eq -ne -lt -le -gt -ge

# -e file  -f regular  -d dir  -r readable  -w writable  -s non-empty

if systemctl is-active --quiet nginx; then
  echo "nginx running"
elif systemctl is-enabled --quiet nginx; then
  echo "nginx enabled but not running"
else
  echo "nginx not configured"
fi
```

**case** (удобно для аргументов CLI):

```bash
case "${1:-}" in
  start)  systemctl start myapp ;;
  stop)   systemctl stop myapp ;;
  status) systemctl status myapp ;;
  *)      echo "Usage: $0 {start|stop|status}" >&2; exit 1 ;;
esac
```

---

## 5. Циклы

```bash
for f in /var/log/*.log; do                   # glob; если нет match — один literal *.log
  [[ -e "$f" ]] || continue                   # пропустить если glob не совпал
  echo "== $f =="
  tail -n 1 "$f"
done

for (( i=0; i<5; i++ )); do echo "$i"; done

while read -r line; do                        # построчно stdin (не tail -f в pipe без осторожности)
  echo "line: $line"
done < /etc/hosts

while ! ping -c1 -W1 1.1.1.1 &>/dev/null; do  # ждать сеть
  sleep 2
done
```

---

## 6. Функции и локальные переменные

```bash
die() { echo "ERROR: $*" >&2; exit 1; }

require_root() {
  [[ $EUID -eq 0 ]] || die "run as root (sudo $0)"
}

backup_dir() {
  local src="$1"                              # local только внутри функции
  local dest="$2"
  [[ -d "$src" ]] || die "no such dir: $src"
  tar -czf "$dest" -C "$(dirname "$src")" "$(basename "$src")"
}

backup_dir /etc "/backup/etc-$(date +%F).tar.gz"
```

---

## 7. Ввод‑вывод, redirect, pipe

```bash
echo "info"                                   # stdout
echo "warn" >&2                               # stderr
echo "line" >> "$LOG_FILE"                    # append в файл
: > "$LOG_FILE"                               # обнулить файл (truncate)

cmd >out.log 2>&1                             # stdout+stderr в один файл
cmd 2>/dev/null                               # скрыть stderr
cmd | grep pattern                            # pipe
cmd1 && cmd2                                  # cmd2 если cmd1 OK
cmd1 || cmd2                                  # cmd2 если cmd1 failed

exec > >(tee -a "$LOG_FILE") 2>&1             # весь вывод скрипта в log + терминал
```

**Here document** (многострочный текст / remote script):

```bash
cat <<'EOF' > /tmp/note.txt                   # 'EOF' — без подстановки переменных
Hello
PATH is literal $PATH
EOF

ssh user@host bash -s <<EOF                   # без кавычек — подстановка локальных vars
hostname
uptime
EOF
```

---

## 8. `trap` — cleanup при выходе

```bash
#!/bin/bash
set -euo pipefail

TMP="$(mktemp -d)"                            # временный каталог
cleanup() { rm -rf "$TMP"; }
trap cleanup EXIT                             # при любом выходе
trap 'echo "Interrupted"; exit 130' INT TERM  # Ctrl+C

# работа в $TMP ...
```

**Lockfile** — не запускать два экземпляра (cron):

```bash
LOCK=/var/run/myjob.lock
exec 9>"$LOCK"                                # fd 9
flock -n 9 || { echo "already running"; exit 0; }
# ... основная работа ...
# lock снимается при закрытии fd при exit
```

---

## 9. Проверки перед работой

```bash
command -v nginx >/dev/null || { echo "nginx not installed"; exit 1; }   # есть ли в PATH

if ! systemctl is-active --quiet nginx; then
  systemctl start nginx || exit 1
fi

df -h / | awk 'NR==2 { gsub(/%/,"",$5); if ($5+0 > 90) exit 1 }' || die "disk > 90%"

curl -fsS --max-time 10 https://example.com/health || die "health check failed"
# -f fail on HTTP error, -s silent, -S show error body
```

---

## 10. Примеры «на все случаи»

К каждому примеру — **зачем** он нужен; код можно вставить в `/usr/local/bin/` и вызывать вручную или из cron.

### 10.1 Мини healthcheck (как [server_healthcheck_quick.md](server_healthcheck_quick.md))

**Что делает:** за один запуск печатает срез состояния VPS — нагрузку, память, заполнение корневого раздела, упавшие unit’ы systemd и первые строки списка слушающих портов.

**Когда использовать:** утренний осмотр, после деплоя, перед/после инцидента; вывод можно переслать в тикет или сохранить в лог (`script.sh | tee health-$(date +%F).log`).

**Важно:** `systemctl --failed` с `|| true`, чтобы скрипт не оборвался, если failed‑юнитов нет (при `set -e` иначе был бы exit 1).

```bash
#!/bin/bash
set -euo pipefail

echo "=== uptime ===" && uptime
echo "=== memory ===" && free -h
echo "=== disk ===" && df -hT /
echo "=== failed units ===" && systemctl --failed --no-pager || true
echo "=== listening ===" && ss -tulpen | head -20
```

### 10.2 Бэкап `/etc` с ротацией

**Что делает:** создаёт сжатый архив `etc-ГГГГ-ММ-ДД.tar.gz` с деревом `/etc` (конфиги nginx, ssh, systemd и т.д.) и **удаляет** архивы старше `KEEP_DAYS` дней.

**Когда использовать:** ежедневный cron на сервере, где конфиги меняют руками; перед `apt upgrade` или правкой `/etc`.

**Не заменяет:** дампы БД и `/var/www` — добавьте отдельные tar/rsync ([backups_cheatsheet.md](backups_cheatsheet.md), [archives_compression_cheatsheet.md](archives_compression_cheatsheet.md)).

```bash
#!/bin/bash
set -euo pipefail
BACKUP_DIR="/backup"
KEEP_DAYS=14
mkdir -p "$BACKUP_DIR"
file="$BACKUP_DIR/etc-$(date +%F).tar.gz"
tar -czf "$file" -C / etc
find "$BACKUP_DIR" -name 'etc-*.tar.gz' -mtime +"$KEEP_DAYS" -delete
echo "OK $file"
```

### 10.3 Retry команды (сеть, API)

**Что делает:** функция `retry` вызывает переданную команду снова (до 5 раз, пауза 3 с), пока она не завершится с кодом 0; иначе возвращает ошибку.

**Когда использовать:** нестабильный интернет, скачивание артефактов deploy, обращение к API при кратковременных 5xx/timeout.

**Пример внизу:** качает файл по HTTPS; при обрыве повторит, а не упадёт с первой же ошибки curl.

```bash
retry() {
  local n=1 max=5 delay=3
  until "$@"; do
    (( n++ )) || true
    [[ $n -le $max ]] || return 1
    sleep "$delay"
  done
}

retry curl -fsS --max-time 30 -O https://example.com/file.tar.gz
```

### 10.4 Перезапуск сервиса только если конфиг OK

**Что делает:** сначала проверяет конфиг nginx (`nginx -t`); при синтаксической ошибке **не** трогает работающий процесс. Если OK — `reload` (подхват настроек без обрыва всех соединений) и проверка, что unit active.

**Когда использовать:** после правки `/etc/nginx/` в deploy‑скрипте; безопаснее, чем слепой `systemctl restart`.

**Аналог:** для других демонов — их `-t` / `configtest` (apache2ctl, sshd -t).

```bash
#!/bin/bash
set -euo pipefail
nginx -t                                    # проверка синтаксиса
systemctl reload nginx                      # мягче чем restart
systemctl is-active --quiet nginx
```

### 10.5 Чтение простого конфига `KEY=VALUE`

**Что делает:** загружает пары `KEY=VALUE` из файла в переменные shell — либо через `source` (весь файл как bash‑код), либо построчно через `read` (пропуск `#` и пустых строк).

**Когда использовать:** свой `/etc/myapp/settings.conf` без JSON/YAML; скрипт читает `BACKUP_DIR`, `RETENTION` и т.п.

**Осторожно:** `source` и `printf -v "$key"` опасны, если конфиг может править чужой человек — там можно подсунуть выполнение команд. Для недоверенных файлов парсьте только известные ключи через `case`.

```bash
CONFIG=/etc/myapp/settings.conf
[[ -f "$CONFIG" ]] || exit 1
# shellcheck source=/dev/null
source "$CONFIG"                            # осторожно: это выполнение bash-кода!
# безопаснее парсить вручную:
while IFS='=' read -r key value; do
  [[ "$key" =~ ^#.*$ || -z "$key" ]] && continue
  printf -v "$key" '%s' "$value"            # export to variable named $key — только если доверяете файлу
done < "$CONFIG"
```

### 10.6 Параллельный запуск (осторожно с нагрузкой)

**Что делает:** на каждом хосте из списка запускает **фоном** (`&`) одну и ту же SSH‑команду (здесь — reload nginx), затем `wait` ждёт завершения всех.

**Когда использовать:** несколько однотипных web‑нод; rolling вручную без Ansible.

**Риск:** одновременный reload на всех — краткий всплеск нагрузки; для production часто делают по одному хосту или используют orchestration. Нужен SSH по ключу ([ssh_cheatsheet.md](ssh_cheatsheet.md)).

```bash
for host in web1 web2 web3; do
  ssh "$host" 'sudo systemctl reload nginx' &
done
wait                                          # дождаться всех фоновых jobs
```

### 10.7 Dry-run режим через переменную

**Что делает:** обёртка `run` либо **выполняет** команду, либо только **печатает**, что было бы выполнено, если `DRY_RUN=1`.

**Когда использовать:** скрипты с `rm`, `mv`, mass‑deploy — сначала `DRY_RUN=1 ./script.sh`, проверить вывод, потом боевой запуск.

**Пример:** `run rm -f /tmp/old-cache/*` при dry-run не удалит файлы, покажет `[dry-run] rm -f ...`.

```bash
DRY_RUN="${DRY_RUN:-0}"
run() {
  if [[ "$DRY_RUN" == 1 ]]; then
    echo "[dry-run] $*"
  else
    "$@"
  fi
}
run rm -f /tmp/old-cache/*
# DRY_RUN=1 ./script.sh
```

### 10.8 Лог с уровнями

**Что делает:** функция `log LEVEL сообщение` пишет в stderr строку с временем и уровнем; уровень окружения `LOG_LEVEL` отсекает шум (при `INFO` не показывается `DEBUG`).

**Когда использовать:** длинные скрипты и cron — в лог попадает важное, отладку включают `LOG_LEVEL=DEBUG ./script.sh`.

**Уровни в примере:** `DEBUG` — всё; `INFO` — без DEBUG; `WARN` — только предупреждения и выше (логику можно расширить под `ERROR`).

```bash
LOG_LEVEL="${LOG_LEVEL:-INFO}"
log() {
  local level="$1"; shift
  case "$LOG_LEVEL" in
    DEBUG) ;;
    INFO)  [[ "$level" == DEBUG ]] && return 0 ;;
    WARN)  [[ "$level" == DEBUG || "$level" == INFO ]] && return 0 ;;
  esac
  printf '[%s] [%s] %s\n' "$(date +%T)" "$level" "$*" >&2
}
log INFO "started"
log DEBUG "verbose detail"
```

---

## 11. Аргументы: `getopts` (кратко)

### Что это

**`getopts`** — встроенная команда bash для разбора **коротких опций** командной строки в стиле Unix: `-v`, `-h`, `-f /path/file`. Один проход по `$1`, `$2`, …; не нужно вручную писать `if [[ "$1" == "-v" ]]; then shift; fi` для каждого флага.

Служебные переменные:

| Переменная | Значение |
|------------|----------|
| `opt` | буква текущей опции (в цикле `while getopts ... opt`) |
| `OPTARG` | аргумент опции, если она с параметром (`-f file` → `OPTARG=file`) |
| `OPTIND` | индекс следующего `$n` для разбора; после цикла — сдвинуть `shift` |

Строка опций **`":vh"`** в `getopts ":vh" opt`:

- **`v`**, **`h`** — допустимые флаги без аргумента;
- **двоеточие в начале** (`:`) — при опции, которой **не хватает** аргумента или неизвестной букве, getopts кладёт ошибку в `opt` (`\?` или `:`), а не пишет в stderr сам; удобно выводить свой текст.

Опция **с аргументом** задаётся двоеточием после буквы, например **`"f:vh"`** — `-f /etc/nginx.conf`.

### Зачем

- Единый **`Usage: ./script.sh [-v] [-f file] host...`**, как у `tar`, `grep`, системных утилит.
- Скрипт из cron/systemd можно вызывать с флагами: `./deploy.sh -v` только для отладки.
- Ошибки пользователя (`./script.sh -x`) обрабатываются предсказуемо.

**Когда достаточно без getopts:** один‑два positional аргумента (`$1` — host, `$2` — port) — см. §3; подкоманды `start|stop` — **`case "$1" in`** (§10.4).

**Длинные опции** `--verbose`, `--config=/path` — в чистом bash не через `getopts`; нужен **`getopt`** (GNU) или ручной разбор; для админ‑скриптов часто хватает коротких `-v`/`-h`.

### Пример

**Что делает:** `-v` включает `set -x` (трассировку); `-h` печатает справку и выходит; неизвестная опция — ошибка в stderr и exit 1. После цикла **`shift $((OPTIND - 1))`** — в `$@` остаются не‑опции (например имена файлов или хосты).

```bash
verbose=0
while getopts ":vh" opt; do
  case $opt in
    v) verbose=1 ;;
    h) echo "Usage: $0 [-v] [args...]"; exit 0 ;;
    \?) echo "Invalid option: -$OPTARG" >&2; exit 1 ;;
  esac
done
shift $((OPTIND - 1))                         # positional args: $@
[[ $verbose -eq 1 ]] && set -x                # трассировка при -v

# ./myscript.sh -v server1 server2  →  set -x + server1 server2 в $@
```

Пример с файлом: `getopts ":f:vh" opt` и в `case` ветка `f) config="$OPTARG" ;;`.

---

## 12. Cron и systemd

### Что это

| | **Cron** | **Systemd timer** |
|---|----------|-------------------|
| Суть | Демон **`cron`** запускает команды **по расписанию** (минуты, часы, дни) | **`timer.unit`** будит **`service.unit`** — тоже расписание, но через systemd |
| Где задаётся | `crontab -e`, `/etc/cron.d/*`, `/etc/cron.{daily,hourly}` | `/etc/systemd/system/*.timer` + `.service` |
| Логи | часто только ваш redirect в файл; mail root при ошибках (если настроен) | `journalctl -u имя.service` |
| Типичный Linux | везде | Ubuntu/Debian/RHEL с systemd |

Оба отвечают на вопрос: **«запускать этот bash‑скрипт автоматически ночью / каждые 5 минут»** без ручного SSH.

### Зачем скрипт + планировщик

- **Бэкапы**, ротация логов, healthcheck, `certbot renew`, очистка `/tmp`.
- Один и тот же скрипт: руками `./backup.sh`, из cron — с redirect в log.
- В скрипте уже есть **`set -euo pipefail`**, **`flock`** (§8) — cron не даст два overlap, если второй запуск придёт раньше конца первого.

**Cron** — быстрее записать одну строку. **Timer** — удобнее, если всё остальное на systemd (зависимости `After=network-online.target`, `Persistent=true` после простоя машины).

---

### Cron: формат строки

```cron
# мин час день_месяца месяц день_недели команда
# m   h   dom      mon   dow
0     3   *        *     *     /usr/local/bin/backup-etc.sh >> /var/log/backup-etc.log 2>&1
*/5   *   *        *     *     /usr/local/bin/health.sh
0     0   *        *     0     /usr/local/bin/weekly-report.sh
```

| Поле | Значение | Пример |
|------|----------|--------|
| мин | 0–59 | `0`, `*/15` каждые 15 мин |
| час | 0–23 | `3` — в 03:00 |
| dom | 1–31 | `*` каждый день |
| mon | 1–12 | `*` |
| dow | 0–7 (0 и 7 = воскресенье) | `1-5` будни |

```bash
crontab -e                                    # расписание текущего пользователя
crontab -l                                    # показать
sudo crontab -u root -e                       # root crontab
ls -la /etc/cron.d/                           # системные задания (часто с полем user)
grep -r . /etc/cron.d/ /etc/cron.daily/ 2>/dev/null | head

# проверить, что cron жив:
systemctl status cron                         # Debian/Ubuntu часто unit "cron"
grep CRON /var/log/syslog | tail              # последние срабатывания
```

**Окружение cron** — урезанное: нет вашего `.bashrc`, мало переменных. В начале скрипта:

```bash
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
export LANG=C.UTF-8
```

Или в crontab **перед** строками задания: `PATH=...`, `MAILTO=admin@example.com` (письма об ошибках, если настроена почта — на VPS часто **не** настроена, поэтому **`>> log 2>&1`** обязателен).

Пример **/etc/cron.d/mybackup** (нужны права root, в строке указан user):

```cron
SHELL=/bin/bash
PATH=/usr/sbin:/usr/bin:/sbin:/bin
0 3 * * * root /usr/local/bin/backup-etc.sh >> /var/log/backup-etc.log 2>&1
```

---

### Systemd: service + timer

**Service** — *что* запустить (oneshot = «отработал и вышел»). **Timer** — *когда*.

Файл **`/etc/systemd/system/backup-etc.service`**:

```ini
[Unit]
Description=Backup /etc to /backup
After=network-online.target

[Service]
Type=oneshot
ExecStart=/usr/local/bin/backup-etc.sh
# User=root  — по умолчанию root; для deploy: User=deploy
StandardOutput=journal
StandardError=journal
```

Файл **`/etc/systemd/system/backup-etc.timer`**:

```ini
[Unit]
Description=Daily backup-etc at 03:00

[Timer]
OnCalendar=*-*-* 03:00:00
Persistent=true
Unit=backup-etc.service

[Install]
WantedBy=timers.target
```

```bash
sudo systemctl daemon-reload                  # после создания/правки unit
sudo systemctl enable --now backup-etc.timer  # включить timer
systemctl list-timers --all                   # когда следующий запуск
sudo systemctl start backup-etc.service       # ручной прогон без ожидания timer
journalctl -u backup-etc.service -n 30 --no-pager   # лог последнего run
sudo systemctl status backup-etc.timer
```

**`Persistent=true`** — если сервер был выключен в 03:00, job выполнится после включения (cron так не гарантирует).

Аналог «каждые 5 минут» в timer: `OnCalendar=*:0/5` или `OnUnitActiveSec=5min` (смотрите `systemd.time(7)`).

---

### Cron или timer — что выбрать

| Ситуация | Выбор |
|----------|--------|
| Один скрипт, раз в ночь, быстро | cron / `/etc/cron.d` |
| Уже мониторите через `journalctl` | systemd timer |
| Нужен `After=network-online.target` | timer проще |
| Legacy, без systemd | только cron |
| Пользовательский job без root | `crontab -e` от своего user |

Пример скрипта бэкапа: §10.2 и [backups_cheatsheet.md](backups_cheatsheet.md) §8.

---

### Типичные проблемы

| Симптом | Причина |
|---------|---------|
| Скрипт «молчит», ничего не произошло | Нет `>> log 2>&1`; ошибка в mail, почты нет |
| `command not found` в cron | PATH; полный путь или `export PATH` в скрипте |
| Работает в SSH, не в cron | `.bashrc` не читается; aliases не работают |
| Два экземпляра наслаиваются | `flock` в скрипте (§8) |
| Не тот час | TZ сервера: `timedatectl`; cron использует local time |

```bash
timedatectl                                   # timezone сервера
ls -l /usr/local/bin/backup-etc.sh            # chmod +x и shebang
/usr/local/bin/backup-etc.sh                    # ручной тест от того же user, что в cron
```

---

## 13. Отладка

```bash
bash -x script.sh                             # трассировка каждой команды
bash -xv script.sh                            # + verbose (set -v)

# внутри скрипта точечно:
set -x
fragile_command
set +x

PS4='+ ${BASH_SOURCE}:${LINENO}: '            # префикс строки в trace
```

---

## 14. Частые ошибки

| Ошибка | Как правильно |
|--------|----------------|
| `$var` без кавычек при пробелах в пути | `"$var"` |
| `[` vs `[[` | в скриптах предпочитайте `[[ ]]` |
| `for i in $(seq 1 10)` с пробелами в данных | while read или массив |
| Забыли shebang, запуск `sh script.sh` | `sh` может быть dash без bash‑фич |
| `cd` без проверки | `cd "$dir" || exit 1` |
| Парсинг `ls` | `find`, glob `for f in *.log` |

---

## 15. Мини‑чек‑лист перед production

1. `#!/bin/bash` + `set -euo pipefail` (или обоснованные исключения).
2. Все пути и переменные в **кавычках**.
3. `bash -n` и **shellcheck**.
4. Логирование и ненулевой `exit` при ошибке.
5. `trap` / `flock` для cron.
6. Тест с **`DRY_RUN=1`** где есть destructive команды.

---

## 16. Шпаргалка одной строкой

```bash
[[ -f file ]] && echo ok                      # тест файла
cmd1 && cmd2 || die "fail"                    # цепочка
$(command)                                    # capture output
"${var:-default}"                             # default если unset
exec 9>lock; flock -n 9 || exit               # single instance
mktemp -d; trap 'rm -rf $d' EXIT               # temp dir
```

Интерактивный shell (aliases, PS1): [bashrc_cheatsheet.md](bashrc_cheatsheet.md).

Готовые скрипты для файлов и фото: [routine_automation_scripts.md](routine_automation_scripts.md).
