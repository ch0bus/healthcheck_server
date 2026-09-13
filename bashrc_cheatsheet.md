# Bash: `.bashrc`, профиль и настройка shell

← [README](README.md) · основы: [linux_install_and_basics.md](linux_install_and_basics.md) · SSH: [ssh_cheatsheet.md](ssh_cheatsheet.md) · vim: [vim_cheatsheet.md](vim_cheatsheet.md)

**Bash** — стандартная оболочка на Ubuntu/Debian и многих VPS. Персональные настройки живут в домашнем каталоге; системные — в `/etc`.

В блоках `bash` комментарии после `#` — пояснения; в терминал можно вставлять всю строку (хвост после `#` shell игнорирует).

---

## 1. Какие файлы за что отвечают

| Файл | Когда читается | Назначение |
|------|----------------|------------|
| `~/.bashrc` | **Интерактивный** non-login shell (новый терминал, `ssh user@host`, `bash`) | Aliases, `PS1`, `PATH`, функции |
| `~/.bash_profile` или `~/.profile` | **Login** shell (консоль до GUI, `ssh` с login, `su - user`) | Часто подключает `.bashrc` или задаёт `PATH` |
| `~/.bash_logout` | Выход из login shell | Редко используется |
| `/etc/bash.bashrc` | Все интерактивные bash (Debian/Ubuntu) | Общие настройки дистрибутива |
| `/etc/profile` | Login shell | Системный профиль |
| `/etc/profile.d/*.sh` | При login (если profile их подключает) | Пакеты добавляют `nodejs.sh`, `python3.sh` и т.д. |

На Ubuntu в desktop‑терминале обычно грузится **`~/.bashrc`**. На сервере по SSH — тоже часто `.bashrc` (интерактивный non-login).

Проверить, login ли shell:

```bash
shopt -q login_shell && echo login || echo non-login   # login_shell включён — login
echo $0                    # -bash часто login; bash — non-login
```

---

## 2. Открыть, применить, проверить

```bash
vim ~/.bashrc              # правка (или nano ~/.bashrc)
source ~/.bashrc           # применить в текущем терминале без перезапуска
. ~/.bashrc                # то же, что source
exec bash                  # перезапустить shell (aliases подхватятся)
type myalias               # показать: alias, function или путь к файлу
alias                      # список всех alias текущей сессии
```

После правок всегда **`source ~/.bashrc`** или новое окно терминала. Ошибка синтаксиса в `.bashrc` может вывести сообщение при каждом входе — правьте через `bash -n`:

```bash
bash -n ~/.bashrc          # проверка синтаксиса без выполнения (quiet = OK)
```

---

## 3. Минимальный шаблон `~/.bashrc`

```bash
# ~/.bashrc — пример каркаса (скопируйте и дополняйте)

# если shell не интерактивный — не настраивать prompt/aliases
case $- in
  *i*) ;;                    # интерактивный — продолжаем
  *) return;;                # скрипт/cron — выходим
esac

# история команд
HISTCONTROL=ignoreboth       # без дубликатов и с пробелом в начале — «секретные»
HISTSIZE=5000                # строк в памяти
HISTFILESIZE=10000           # строк в ~/.bash_history
shopt -s histappend          # дописывать историю, не перезаписывать файл

# удобства bash
shopt -s checkwinsize        # обновлять LINES/COLUMNS после каждой команды
shopt -s globstar            # **/*.log работает (bash 4+)

# безопасность по умолчанию для rm (опционально, по вкусу)
# alias rm='rm -i'

# свой PATH (локальные bin раньше системных)
export PATH="$HOME/bin:$HOME/.local/bin:$PATH"

# prompt: user@host:каталог$
export PS1='\[\033[01;32m\]\u@\h\[\033[00m\]:\[\033[01;34m\]\w\[\033[00m\]\$ '

# подключить доп. файл, если есть
[ -f ~/.bash_aliases ] && . ~/.bash_aliases
```

---

## 4. Переменные окружения (`export`)

```bash
export EDITOR=vim            # редактор для crontab, visudo, git commit
export VISUAL=vim              # «полноэкранный» редактор по умолчанию
export LANG=en_US.UTF-8        # локаль (или ru_RU.UTF-8)
export LC_ALL=                 # пусто — не переопределять все категории насильно

export PATH="$PATH:$HOME/go/bin"   # добавить в конец PATH
export PATH="$HOME/go/bin:$PATH"   # добавить в начало (приоритет)

# одна сессия, без .bashrc:
MYVAR=value command          # переменная только для одной команды
export MYVAR=value           # до закрытия терминала
```

Просмотр:

```bash
printenv PATH                # значение PATH
env | grep -i proxy          # найти proxy-переменные
set                          # все переменные и shell options (много вывода)
```

---

## 5. Aliases: сокращения команд

```bash
alias ll='ls -alF'           # длинный список с типами файлов
alias la='ls -A'             # все кроме . и ..
alias ..='cd ..'               # на уровень вверх
alias ...='cd ../..'           # два уровня
alias gs='git status'          # git
alias gc='git commit'
alias please='sudo'            # мем; на сервере лучше не привыкать

# с аргументами alias неудобен — используйте функцию (см. ниже)
unalias ll                   # убрать alias до конца сессии
```

**Опасные alias на production:** `alias cp='cp -i'` полезен на desktop; на скриптах в cron alias **не работают** (non-interactive).

---

## 6. Функции (лучше alias с параметрами)

Добавьте в `~/.bashrc`:

```bash
# mkcd — создать каталог и перейти
mkcd() { mkdir -p "$1" && cd "$1"; }

# быстрый бэкап файла перед правкой
bak() { cp -- "$1" "$1.bak.$(date +%Y%m%d%H%M%S)"; }

# cd + ls
cdls() { cd "$@" && ls -la; }
```

Проверка:

```bash
type mkcd                     # function mkcd () { ... }
mkcd /tmp/test-dir            # вызов
```

---

## 7. История и поиск по ней

```bash
history                       # номера и команды
history 20                    # последние 20
!!                            # повторить последнюю команду
!53                           # выполнить команду №53 из history
!?nginx                       # последняя команда, содержащая nginx
Ctrl+r                        # интерактивный reverse search (повтор Ctrl+r)

# настройки в .bashrc:
shopt -s histappend
export HISTTIMEFORMAT='%F %T '   # метки времени в history (опционально)
```

Файл истории: `~/.bash_history` (может подгружаться с задержкой при выходе).

---

## 8. Связка `.profile` и `.bashrc` (login + SSH)

Типичный паттерн Debian/Ubuntu в **`~/.profile`**:

```bash
# если интерактивный bash — подтянуть .bashrc
if [ -n "$BASH_VERSION" ]; then
  if [ -f "$HOME/.bashrc" ]; then
    . "$HOME/.bashrc"
  fi
fi
```

И наоборот: в некоторых системах `.bashrc` в конце проверяет и не дублирует. Смотрите **существующий** файл дистрибутива — не затирайте блоки с `# If not running interactively`.

```bash
grep -n bashrc ~/.profile ~/.bash_profile 2>/dev/null   # что откуда подключается
```

---

## 9. Системные настройки (осторожно)

```bash
ls /etc/profile.d/           # скрипты, которые может читать login
cat /etc/bash.bashrc | head    # общий bashrc (нужны права чтения)

# свой скрипт для всех пользователей — только через админку:
sudo vim /etc/profile.d/mycompany.sh
sudo chmod 644 /etc/profile.d/mycompany.sh
```

Не правьте `/etc/bash.bashrc` без необходимости — достаточно `~/.bashrc`.

---

## 10. Полезно для админа VPS

```bash
# сжатый статус после входа (лёгкий motd; не перегружать SSH)
# в ~/.bashrc — только если нужно:
# uptime
# df -hT / | tail -1

# быстрые ссылки на шпаргалки репозитория (если клонировали LinuxHelp)
# alias linuxhelp='cd ~/LinuxHelp && ls *.md'

# SSH agent (desktop) — см. ssh_cheatsheet.md §6
# eval "$(ssh-agent -s)"
# ssh-add ~/.ssh/id_ed25519

# меньше шума при tab-completion (опционально)
# bind 'set bell-style none'
```

Секреты (**пароли, API‑ключи**) не кладите в `.bashrc` — файл часто в бэкапах и dotfiles‑репозиториях; используйте `~/.config/`, vault, переменные из systemd/cron с ограниченными правами.

---

## 11. Пример `~/.bash_aliases` (Ubuntu)

На Ubuntu часто есть строка в `.bashrc`: `[ -f ~/.bash_aliases ] && . ~/.bash_aliases`. Вынесите туда только aliases:

```bash
# ~/.bash_aliases
alias update='sudo apt update && sudo apt upgrade -y'
alias ports='ss -tulpen'
alias failed='systemctl --failed'
```

```bash
source ~/.bash_aliases       # после правки
```

См. диагностику: [server_healthcheck_quick.md](server_healthcheck_quick.md).

---

## 12. Отладка «сломался bash»

| Симптом | Что сделать |
|---------|-------------|
| Странный prompt, ошибки при входе | `bash -n ~/.bashrc`; временно `mv ~/.bashrc ~/.bashrc.bak` и новый терминал |
| Alias не работает в скрипте | В скрипте alias по умолчанию выключены; вызывайте полную команду |
| `PATH` «съел» системные команды | `export PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin` |
| Изменения не видны | `source ~/.bashrc` или `exec bash`; проверьте, правили ли тот же user (`whoami`) |
| Другой shell (zsh/fish) | Ищите `~/.zshrc`, `~/.config/fish/config.fish` |

```bash
echo $SHELL                    # login shell по умолчанию (/bin/bash)
ps -p $$ -o comm=              # текущая оболочка этого терминала
chsh -l                        # список разрешённых shell (если chsh установлен)
```

---

## 13. Мини‑чек‑лист

1. Правки только в **`~/.bashrc`** / **`~/.bash_aliases`**, не в `/etc` без причины.
2. **`bash -n ~/.bashrc`** перед `source`.
3. **`export PATH`** — дубли не плодить; локальные bin в **начало**.
4. **`export EDITOR=vim`** — согласовано с [vim_cheatsheet.md](vim_cheatsheet.md).
5. На сервере — минимум aliases; критичные операции — полные команды в документации/runbook.
