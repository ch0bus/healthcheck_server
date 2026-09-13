# tmux: терминальный мультиплексор

← [README](README.md) · SSH: [ssh_cheatsheet.md](ssh_cheatsheet.md) · shell: [bashrc_cheatsheet.md](bashrc_cheatsheet.md) · vim‑keys: [vim_cheatsheet.md](vim_cheatsheet.md)

**tmux** (terminal multiplexer) держит **сессии** shell на сервере: окна и панели живут, даже если **оборвался SSH**, закрыли ноутбук или переключили Wi‑Fi. Повторный вход — **`tmux attach`**, работа продолжается там, где остановились.

| | **Обычный SSH** | **SSH + tmux на сервере** |
|---|-----------------|---------------------------|
| Обрыв связи | Shell и foreground‑команды часто **убиваются** | Сессия tmux **остаётся**, `attach` снова |
| Несколько задач | Много вкладок терминала или `&` | Окна/панели в **одной** SSH‑сессии |
| Долгий deploy / `tail -F` | Риск потерять вывод | Лог в панели, history scrollback |

**GNU screen** — аналог старого поколения; на новых гайдах чаще **tmux**. Идея та же: detach / attach.

**Где запускать:** tmux ставят и запускают **на той машине, где должна жить работа** — обычно **на VPS** после `ssh user@host`, не вместо SSH на ноутбуке (локальный tmux тоже полезен, но для админки VPS — серверный).

---

## 1. Иерархия: server → session → window → pane

```text
tmux server (демон на хосте)
  └── session "prod"          # именованная сессия (проект, сервер, задача)
        ├── window 0: shell
        ├── window 1: logs
        └── window 2: editor
              ├── pane (слева)
              └── pane (справа)
```

- **Session** — отдельный «рабочий стол» в терминале.
- **Window** — как вкладка (полный экран).
- **Pane** — деление окна на части (split).

---

## 2. Установка и первые команды

```bash
sudo apt update && sudo apt install -y tmux   # Debian/Ubuntu
tmux                                          # новая анонимная сессия
tmux new -s work                              # сессия с именем work
tmux ls                                       # список сессий
tmux attach -t work                           # подключиться к work
tmux attach -d -t work                        # attach, отцепить других клиентов (-d)
tmux kill-session -t work                     # убить сессию work
```

**Создать или подключиться одной командой** (tmux 1.7+):

```bash
tmux new -A -s main                           # -A: attach если есть, иначе new
```

Выход из shell **внутри** tmux (`exit` или Ctrl+D) закрывает **pane**; когда не останется окон — сессия завершится. Чтобы **оставить** сессию работающей — **detach** (см. §4).

---

## 3. Префикс: все «горячие» клавиши через него

По умолчанию **prefix** = **`Ctrl+b`**. Схема: нажали **Ctrl+b**, отпустили, затем **одну** букву/клавишу.

В тексте ниже: **`Prefix`** = **`Ctrl+b`**.

Справка внутри tmux: **`Prefix`** **`?`** — список bindings; **`q`** — номера панелей (кратко).

---

## 4. Сессии

| Действие | Клавиши / команда |
|----------|-------------------|
| Отсоединиться (detach), сессия **жива** | **`Prefix`** **`d`** или `tmux detach` |
| Переименовать сессию | **`Prefix`** **`$`** |
| Меню выбора сессии | **`Prefix`** **`s`** |
| Новая сессия (из shell) | `tmux new -s ИМЯ` |
| Убить текущую сессию | `tmux kill-session` или **`Prefix`** **`:kill-session`** |

После **`ssh`** на VPS типичный ритуал:

```bash
ssh vps
tmux new -A -s admin
# работа…
# Prefix d — закрыли ноутбук
# позже:
ssh vps
tmux attach -t admin
```

Keepalive SSH (NAT не рвёт TCP) — в [ssh_cheatsheet.md](ssh_cheatsheet.md) (`ServerAliveInterval`); **tmux** страхует уже **оборванную** сессию.

---

## 5. Окна (windows)

| Клавиши | Действие |
|---------|----------|
| **`Prefix`** **`c`** | Новое окно |
| **`Prefix`** **`n`** / **`p`** | Следующее / предыдущее окно |
| **`Prefix`** **`0`** … **`9`** | Перейти на окно по номеру |
| **`Prefix`** **`,`** | Переименовать **текущее** окно |
| **`Prefix`** **`&`** | Закрыть окно (подтверждение) |
| **`Prefix`** **`w`** | Список окон (выбор интерактивно) |
| **`Prefix`** **`l`** | Вернуться к **последнему** окну (toggle) |

Имена окон (`logs`, `htop`, `deploy`) экономят время при **`Prefix w`**.

---

## 6. Панели (panes)

| Клавиши | Действие |
|---------|----------|
| **`Prefix`** **`%`** | Split **вертикально** (две колонки) |
| **`Prefix`** **`"`** | Split **горизонтально** (две строки) |
| **`Prefix`** **стрелки** | Фокус на соседнюю панель |
| **`Prefix`** **`o`** | Следующая панель по кругу |
| **`Prefix`** **`x`** | Закрыть текущую панель |
| **`Prefix`** **`z`** | **Zoom** — развернуть панель на всё окно (повтор — вернуть) |
| **`Prefix`** **`{`** / **`}`** | Поменять панели местами |
| **`Prefix`** **`!`** | Текущая панель → отдельное окно |
| **`Prefix`** **`q`** | Показать номера панелей (для jump) |
| **`Prefix`** **`;`** | Перейти к **последней** активной панели |

**Размер:** **`Prefix`** удерживать + **стрелки** (resize) — в стандартном keymap; в `~/.tmux.conf` часто переназначают на **`Prefix`** **`H/J/K/L`**.

**Синхронный ввод во все панели** (осторожно на prod):

```text
Prefix :setw synchronize-panes on
Prefix :setw synchronize-panes off
```

Удобно для одинаковой команды на нескольких staging‑хостах в split — на production легко сделать лишнее.

---

## 7. Прокрутка и copy mode

Вывод в pane **прокручивается** только в **copy mode**:

| Клавиши | Действие |
|---------|----------|
| **`Prefix`** **`[`** | Войти в copy mode |
| **PgUp** / **PgDn** или стрелки | Прокрутка |
| **`q`** | Выйти из copy mode |

С **`setw -g mode-keys vi`** в конфиге (см. §9): в copy mode **`v`** — выделение, **`y`** — копировать в tmux buffer; вставка в pane: **`Prefix`** **`]`**.

Буфер tmux ≠ системный clipboard; для интеграции с X/Wayland нужны **`tmux-yank`**, **`xclip`**, **`wl-copy`** — по желанию на desktop.

---

## 8. Командная строка tmux

**`Prefix`** **`:`** — prompt (как `:` в vim):

```text
:new-window -n logs
:kill-window
:respawn-pane -k                          # перезапустить shell в pane (-k kill)
:select-window -t logs
:select-pane -t 2
:set-option -g mouse on                   # до перезагрузки конфига
:source-file ~/.tmux.conf                 # перечитать конфиг
```

Полезно **`respawn-pane`** после «зависшего» shell без закрытия layout.

---

## 9. `~/.tmux.conf` (минимум)

```bash
nano ~/.tmux.conf
```

Пример разумного старта:

```tmux
# ~/.tmux.conf — после правок: Prefix :  →  source-file ~/.tmux.conf  или tmux source-file ~/.tmux.conf

set -g default-terminal "screen-256color"   # цвета для htop, ranger, vim
set -g history-limit 50000                  # длинный scrollback логов
set -g mouse on                             # мышь: панели, resize, scroll (tmux 2.1+)
set -g base-index 1                         # окна с 1, не с 0
setw -g pane-base-index 1
setw -g mode-keys vi                        # copy mode как vi

# splits привычнее символов | и -
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
bind c new-window -c "#{pane_current_path}" # новое окно в том же каталоге

bind r source-file ~/.tmux.conf \; display "tmux.conf reloaded"
```

**`#{pane_current_path}`** — новые pane/окно открываются в **текущем каталоге** pane (tmux 1.9+).

Reload из shell **вне** tmux:

```bash
tmux source-file ~/.tmux.conf
```

Dotfiles: [bashrc_cheatsheet.md](bashrc_cheatsheet.md), бэкап конфигов — [routine_automation_scripts.md](routine_automation_scripts.md) §15.12.

---

## 10. Практические сценарии

### Деплой и лог в одной SSH‑сессии

```text
# окно deploy
cd /var/www/app && ./deploy.sh

# Prefix c — новое окно "logs"
sudo tail -F /var/log/nginx/error.log

# Prefix 1 / Prefix 2 — переключение
# обрыв SSH → tmux attach — tail и deploy не пропали, если deploy в pane всё ещё идёт
```

Долгие задачи лучше в **`screen`/`tmux`** + **`nohup`**/`systemd`, если скрипт не переживает SIGHUP — см. [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md).

### Мониторинг: htop + journalctl

```text
Prefix %     # split
htop         # слева — htop_cheatsheet.md
journalctl -f -u nginx   # справа
Prefix z     # временно развернуть htop
```

### Правка конфига и проверка сервиса

```text
# pane 1
sudoedit /etc/nginx/sites-available/site

# pane 2
sudo nginx -t && sudo systemctl reload nginx
```

Vim в tmux: [vim_cheatsheet.md](vim_cheatsheet.md). **`Prefix`** **`b`** — отправить **break** в foreground (редко нужно).

### Несколько проектов на одном VPS

```bash
tmux new -s blog
tmux new -s api
tmux new -s db-tunnel
tmux ls
tmux attach -t api
```

Не смешивайте prod и эксперименты в одной session без необходимости.

### Файлы: ranger в одной панели, shell в другой

```text
Prefix "
ranger       # верх — ranger_cheatsheet.md
# низ — git status, docker compose ps
```

### Перезагрузка сервера

**tmux не переживает reboot** — сессии исчезнут. После ребута снова `ssh` и новая session; долгие job — **systemd unit**, **cron**, **at**, не только tmux.

### Два администратора, одна session

Оба могут **`attach -t shared`** — видят одни и те же pane (удобно для парной отладки, легко мешать друг другу). **`attach -d`** отцепит чужой клиент.

### Локальный tmux + SSH в pane

На **ноутбуке** `tmux`, в pane — `ssh vps`. При обрыве **локального** tmux SSH внутри pane умрёт; для **устойчивости деплоя** tmux должен работать **на VPS**, а не только локально.

---

## 11. CLI для скриптов

```bash
tmux has-session -t backup 2>/dev/null || tmux new-session -d -s backup 'restic backup ...'
tmux capture-pane -t backup:0.0 -p -S -100    # последние ~100 строк pane в stdout
tmux send-keys -t backup 'echo done' Enter    # отправить команду в pane
tmux list-windows -t backup -F '#{window_index} #{window_name}'
```

Осторожно с **`send-keys`** на production — легко отправить не туда.

---

## 12. tmux vs screen (кратко)

| | **tmux** | **screen** |
|---|----------|------------|
| Конфиг | `~/.tmux.conf` | `~/.screenrc` |
| Splits | Встроены | `-S` / regions |
| Скрипты / API | `send-keys`, `capture-pane` | есть аналоги |
| На minimal VPS | часто нужен `apt install tmux` | иногда уже установлен |

Знать **screen** полезно на legacy‑хостах: **`Ctrl+a`** prefix, **`d`** detach.

---

## 13. Частые ошибки

- **«Нет tmux после reconnect»** — забыли **`tmux new`** на сервере; работали в «голом» SSH.
- **Сессия пропала после reboot** — tmux **не переживает** перезагрузку.
- **Prefix дважды подряд** — нужно **отпустить** Ctrl+b, потом клавиша команды.
- **Мышь мешает** — `set -g mouse off` в conf.
- **Цвета htop/vim блеклые** — `default-terminal`, **`export TERM=screen-256color`** внутри tmux ([htop_cheatsheet.md](htop_cheatsheet.md)).

---

## 14. Мини‑шпаргалка

```text
Prefix = Ctrl+b

Prefix d     detach          Prefix $     rename session
Prefix c     new window      Prefix ,     rename window
Prefix n/p   next/prev win   Prefix w     window list
Prefix % "   split           Prefix z     zoom pane
Prefix [     scroll/copy     Prefix :     command prompt

tmux new -A -s NAME    tmux ls    tmux attach -t NAME
```

SSH на сервер → **`tmux new -A -s admin`** → работа → **`Prefix d`**. Логи: [logs_cheatsheet.md](logs_cheatsheet.md). Инциденты: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md).
