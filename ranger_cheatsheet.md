# Ranger: файловый менеджер в терминале

← [README](README.md) · навигация shell: [linux_install_and_basics.md](linux_install_and_basics.md) · редактор: [vim_cheatsheet.md](vim_cheatsheet.md) · архивы: [archives_compression_cheatsheet.md](archives_compression_cheatsheet.md)

**Ranger** — **TUI**‑файловый менеджер в терминале с **vi‑подобными** клавишами: три колонки (родитель / текущий каталог / превью), быстрый обход дерева без мыши. Удобен на **SSH**, в **tmux** и на desktop, если привычны `h/j/k/l` из Vim.

| | **GUI (Nautilus, Dolphin)** | **mc (Midnight Commander)** | **ranger** |
|---|------------------------------|-----------------------------|------------|
| Интерфейс | Окно, мышь | Две панели, F‑клавиши | Колонки, vi‑keys |
| Превью | Да | Ограничено | Текст, изображения (с утилитами) |
| Скрипты / rifle | — | User menu | **rifle.conf**, `:shell` |
| Зависимости | Desktop | Обычно есть в repos | Python, терминал |

**nnn**, **lf** — альтернативы с похожей идеей; ranger — самый «vim‑like» из популярных.

---

## 1. Установка и первый запуск

```bash
sudo apt update && sudo apt install -y ranger   # Debian/Ubuntu
ranger                                          # открыть в текущем каталоге
ranger /var/log                                 # старт в заданном пути
ranger --help                                   # опции CLI
```

**Полезные пакеты для превью и архивов** (не обязательны, но расширяют возможности):

```bash
sudo apt install -y atool unrar-free poppler-utils mediainfo highlight \
  w3m-img img2txt                             # архивы, PDF, медиа, подсветка кода, картинки в терминале
```

Скопировать дефолтные конфиги для правки:

```bash
ranger --copy-config=all                      # ~/.config/ranger/{rc.conf,rifle.conf,commands.py,...}
```

Выход: **`:q`** или **`:quit`**, либо **Ctrl+D** на пустом каталоге (зависит от настроек). Клавиша **`Q`** — quit без подтверждения в стандартном keymap.

---

## 2. Экран: три колонки

- **Слева** — содержимое **родительского** каталога (контекст «откуда пришли»).
- **Центр** — **текущий** каталог (здесь основная работа).
- **Справа** — **превью** выделенного файла (текст, метаданные, миниатюра — если настроено).

Внизу — **строка пути**, **статус** (права, размер, свободное место на ФС), иногда подсказки по клавишам.

Скрытые файлы (`.`): **`zh`** — toggle, или в консоли `:set show_hidden true`.

---

## 3. Навигация (как в Vim)

| Клавиша | Действие |
|---------|----------|
| **h** / **←** | Вверх на уровень (родительский каталог) |
| **l** / **→** / **Enter** | Войти в каталог или открыть файл (**rifle**) |
| **j** / **↓** | Вниз по списку |
| **k** / **↑** | Вверх по списку |
| **gg** | В начало списка |
| **G** | В конец списка |
| **Ctrl+F** / **Ctrl+B** | Страница вниз / вверх |
| **[** / **]** | Предыдущий / следующий каталог на том же уровне |
| **H** / **L** | Назад / вперёд по **истории** каталогов |
| **-** (минус) | Перейти в каталог, из которого запустили ranger |
| **`** (backtick) | Меню **закладок** (см. ниже) |
| **/** | Поиск по имени в текущем каталоге |
| **n** / **N** | Следующее / предыдущее совпадение поиска |

**Быстрый переход по пути** — консоль **`:`**:

```text
:cd /etc/nginx
:cd ~/Projects
:cd -                         # как в shell — предыдущий каталог
```

---

## 4. Закладки

| Клавиша | Действие |
|---------|----------|
| **m** затем **буква** | Сохранить текущий каталог в закладку (например `ml` → bookmark `l`) |
| **`** затем **буква** | Перейти на закладку `l` |
| **um** + буква | Удалить закладку (через `:delbookmark` / keymap) |

Закладки хранятся в **`~/.config/ranger/bookmarks`**.

---

## 5. Файлы: выделение, копирование, удаление

Операции **vi‑style**: сначала «вырезать/копировать» в буфер, потом **paste**.

| Клавиша | Действие |
|---------|----------|
| **Space** | Выделить / снять выделение с файла |
| **v** | Умное выделение (toggle) |
| **V** | Выделить / снять **все** в каталоге |
| **uv** | Сбросить выделение |
| **yy** | **Копировать** (yank) файлы в буфер |
| **dd** | **Вырезать** (cut) — для перемещения |
| **pp** | **Вставить** в текущий каталог |
| **po** | Вставить **перезаписью** (overwrite) |
| **D** | Удалить (или в корзину — см. `rc.conf`) |
| **cw** | Переименовать (rename) |
| **/c** | Создать файл (`:touch`) |
| **F7** или **:mkdir** | Новый каталог |

**Права и владелец** (нужны права на ФС):

| Клавиша | Действие |
|---------|----------|
| **+** / **-** | chmod +x / убрать x (на выделенном) |
| **=** | Диалог **chmod** |
| **d** затем **m** | **chown** (если есть в keymap) |

Консоль: `:chmod 644 file`, `:rename newname`.

**Теги** (отдельно от выделения Space):

- **t** — пометить файл тегом; операции **`dT`**, **`yy`** с тегами — см. **F1** / `:help` в вашей версии.

---

## 6. Открытие, просмотр, редактирование

| Клавиша | Действие |
|---------|----------|
| **l** / **Enter** | Открыть через **rifle** (программа по типу файла) |
| **r** | Меню «открыть с помощью…» |
| **i** | **Просмотр** (pager, без запуска GUI) |
| **E** | Редактировать в **`$EDITOR`** (часто vim/nano) |
| **du** | Размер каталога (du) |
| **`:preview`** | Настройки превью |

Редактирование конфигов на сервере: **E** → правки в vim → `:wq` → в ranger файл обновится после **`:reload`** или **Ctrl+R**.

Связка с Vim: [vim_cheatsheet.md](vim_cheatsheet.md), **`sudoedit`** для `/etc`:

```bash
export EDITOR='vim'
sudo -E ranger /etc/nginx          # осторожно: удаление в /etc без undo
sudoedit /etc/nginx/sites-available/default   # для одного файла часто безопаснее
```

---

## 7. Консоль `:` и shell

Нажмите **`:`** — командная строка ranger (как ex‑режим).

```text
:help                         # список команд
:shell ls -la                 # выполнить shell-команду, вернуться в ranger
:shell sudo systemctl status nginx
:pwd
:rename newname.conf
:mkdir subdir
:delete                       # удалить выделенное
:flat 0                       # «плоский» вид: 0 = только текущий уровень
:flat -1                      # все файлы рекурсивно в одном списке (осторожно на больших деревьях)
:filter regex                 # показать только совпадающие
:filter
:reload                       # перечитать каталог с диска
```

| Клавиша | Действие |
|---------|----------|
| **S** | **Shell** — `$SHELL` в **текущем каталоге** (выйти из shell → снова ranger) |
| **s** | Shell с подстановкой **имён выделенных** файлов |
| **!** | Произвольная команда с `%s` / `%f` (см. `:help`) |
| **@** | Просмотр **man** / help по выделенному (если настроено) |

---

## 8. rifle — чем открывать файлы

Файл **`~/.config/ranger/rifle.conf`**: правила «расширение / MIME → команда».

Пример фрагмента (после `--copy-config` правьте копию):

```ini
# PDF — просмотр в терминале
ext pdf, has pdftotext, terminal = pdftotext "$1" - | ${PAGER:-less}

# Архивы — список (см. archives_compression_cheatsheet.md)
ext tar|gz|bz2|xz, has atool, terminal = atool -l "$1" | ${PAGER:-less}

# Текст / конфиги — $EDITOR
ext txt|md|conf|yaml|yml|json, terminal = "$EDITOR" -- "$1"
```

После правок **rifle** перезапуск ranger не всегда нужен — достаточно **`:reload`**.

---

## 9. Настройка `rc.conf` (кратко)

```bash
nano ~/.config/ranger/rc.conf
```

Имеет смысл:

```python
set preview_images true
set preview_directories true
set show_hidden false
set sort natural
set confirm_on_delete multiple
# set delete_to_trash true   # нужен пакет trash-cli
```

```bash
sudo apt install -y trash-cli
```

В **`rc.conf`**: `set delete_to_trash true` — тогда **D** отправляет в корзину (где поддерживается), а не unlink навсегда.

---

## 10. Интеграция с bash: выйти из ranger — остаться в каталоге

По умолчанию `cd` в ranger **не** меняет каталог родительского shell. Паттерн **`--choosedir`**:

```bash
# функция в ~/.bashrc — см. bashrc_cheatsheet.md
ranger_cd() {
  local tmp
  tmp="$(mktemp)"
  if ranger --choosedir="$tmp" "${1:-.}"; then
    if [[ -f "$tmp" ]]; then
      cd -- "$(cat "$tmp")" || return
      rm -f "$tmp"
    fi
  fi
}
# alias r='ranger_cd'
```

**Запуск:**

```bash
source ~/.bashrc
ranger_cd /var/www
# по выходу из ranger pwd = последний каталог в ranger
```

---

## 11. Практические сценарии

### Разбор логов в `/var/log`

```bash
sudo ranger /var/log
# j/k — файл, i — просмотр, E — если нужно править logrotate snippet
# / error — поиск по имени в каталоге
# :flat -1 и filter — только при необходимости (может быть медленно)
```

Для огромных логов **i** лучше заменить на **`!tail -n 100 %f`** или смотреть [logs_cheatsheet.md](logs_cheatsheet.md).

### Деплой: залить выделенные файлы в каталог

Выделить (**Space**) нужные файлы → **yy** → **l** в целевой каталог → **pp**. Для перезаписи конфигов — сначала бэкап (**yy** в `backup/`).

### Поиск «где лежит конфиг nginx»

```bash
ranger /
# :cd /etc
# / nginx — поиск имени в текущем каталоге; для глубокого поиска — :shell sudo find /etc -name 'nginx.conf' 2>/dev/null
```

### Несколько однотипных переименований

Выделить файлы → **`:bulkrename`** (если включено в keymap, часто **A** на выделении) — шаблон `{p}{s}{r}` в духе `{p}{s}{r|foo|bar|}`.

### Архив «как в файловом менеджере»

Выделить каталоги → **`:shell tar -czvf backup.tgz %s`** (синтаксис плейсхолдеров уточните `:help shell` — `%s` selection). Или **yy** + **pp** в `/tmp/backup/`, затем tar из [archives_compression_cheatsheet.md](archives_compression_cheatsheet.md).

### Права на скрипт после копирования

Скопировали **`deploy.sh`** → выделить → **+** (chmod +x) или **`=`** → `755`.

### Работа по SSH с узким терминалом

```bash
export TERM=xterm-256color
ranger --cmd="set preview_appearance none"   # или отключить превью в rc.conf на время
```

Уменьшите ширину превью в **rc.conf** (`column_ratio`) или **Ctrl+W** (toggle preview в стандартном map).

### NFS / Samba mount

Открыть **`/mnt/nas/share`** как обычный каталог; при **Stale file handle** — **`:reload`**, снаружи remount ([nfs_samba_cheatsheet.md](nfs_samba_cheatsheet.md)).

### Не удалить лишнее в `/` или `/etc`

- Включите **`confirm_on_delete`**.
- Не запускайте **`sudo ranger /`** без необходимости; для точечных правок — **`sudoedit`** или **`sudo ranger /etc/nginx`**.
- **D** без корзины на сервере **неотвратимо** — проверьте **`delete_to_trash`**.

### Быстро открыть проект из закладки

В `~/Projects/myapp`: **`m`** затем **`p`** (закладка `p`) → в следующий раз **`'`** + **`p`** из любого места.

---

## 12. CLI без полного UI

```bash
ranger --choosefile=/tmp/out.txt /path    # выбрать один файл, записать путь в out.txt
ranger --choosedir=/tmp/dir.txt           # записать каталог при выходе (см. ranger_cd)
ranger --selectfile=/etc/hosts            # старт с выделением на файле
```

Удобно в скриптах «выберите файл вручную».

---

## 13. Частые ошибки

- **Превью «ломает» SSH** — отключить `preview_images` или поставить `w3m-img` / использовать **i** вместо **l** на бинарниках.
- **yy/pp «ничего не происходит»** — забыли **выделить** (Space) или вставляете не в тот каталог (смотрите центральную колонку).
- **Нет прав на запись** — `E`/`pp`/`D` молча fail; смотрите статус или `:shell touch test`.
- **Кодировка имён** — locale UTF-8 (`locale`); битые имена — `:rename` или `mv` из shell.

---

## 14. Мини‑шпаргалка

```text
h/j/k/l  навигация   gg/G   H/L история   m / `  закладки
yy dd pp   копировать/вырезать/вставить   Space выделение   cw rename   D delete
E edit   i view   r open with   S shell в каталоге   :cd :q
zh скрытые   / поиск   :reload   :flat -1 осторожно
ranger --copy-config=all   ~/.config/ranger/{rc.conf,rifle.conf}
```

Desktop‑рутина с файлами: [routine_automation_scripts.md](routine_automation_scripts.md). SSH и копирование: [ssh_cheatsheet.md](ssh_cheatsheet.md) (scp/rsync).
