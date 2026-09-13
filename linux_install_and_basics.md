# Ubuntu‑подобные системы: установка, файловые системы, ориентирование

← [README](README.md) · после установки: [server_healthcheck_quick.md](server_healthcheck_quick.md)

Краткий фундамент для Debian/Ubuntu и производных (Linux Mint, Pop!_OS, elementary OS, Kubuntu, Xubuntu и т.д.). На сервере и на ноутбуке логика одна; в основном отличаются установщик и набор GUI.

В блоках `bash` у каждой строки есть комментарий `# …`. В терминал копируйте только часть **до** `#` (или всю строку — shell проигнорирует хвост, если `#` внутри кавычек нет).

---

## 1. Что значит «Ubuntu‑подобная» система

| Семейство | Пакеты | Типичное использование |
|-----------|--------|-------------------------|
| Debian / Ubuntu / Mint / Pop!_OS | `apt`, `dpkg` | Десктоп, VPS, серверы |
| Fedora / RHEL / Alma | `dnf`, `rpm` | Серверы, корпоративные стенды |

Дальше — про **Debian/Ubuntu‑линию**: одинаковые команды обновления, похожая структура каталогов (FHS), systemd.

---

## 2. Подготовка к установке

### Откуда брать образ

- Официальный сайт дистрибутива (Ubuntu, Debian, Mint…).
- Для VPS — образ от провайдера (часто уже установлен minimal/server).

Проверка контрольной суммы (пример для ISO):

```bash
sha256sum ubuntu-24.04-desktop-amd64.iso   # хеш ISO; сравнить с суммой на сайте загрузки
```

### Загрузочная флешка (ПК / ноутбук)

1. Записать ISO на USB: **Rufus** (Windows), **Etcher**, **Ventoy**, или `dd` / **GNOME Disks** в Linux.
2. В BIOS/UEFI: включить загрузку с USB; при dual boot — отключить **Fast Boot** (Windows), освободить место или отдельный диск.
3. **UEFI vs Legacy (BIOS):** современные дистрибутивы ожидают UEFI + таблица **GPT**. Старый BIOS — **MBR** (сегодня реже).

```bash
# ОСТОРОЖНО: of=/dev/sdX — ваша флешка, не системный диск
sudo dd if=./ubuntu.iso of=/dev/sdX bs=4M status=progress conv=fsync  # запись ISO на USB
# if — образ; of — устройство флешки; bs — размер блока; conv=fsync — сброс буфера на диск
```

### Минимальные требования (ориентир)

- **Desktop:** 4+ ГБ RAM, 25+ ГБ диск, 64‑bit CPU.
- **Server / VPS:** 1–2 ГБ RAM для минимального сервера; больше — под Docker/БД.

---

## 3. Установка: типовые шаги установщика

1. Язык, раскладка, сеть (Wi‑Fi на ноутбуке).
2. **Диск:**
   - **Стереть диск и установить Ubuntu** — один диск только под Linux.
   - **Рядом с Windows** — shrink NTFS, создать разделы под Linux (нужен backup).
   - **Вручную (Something else)** — полный контроль (см. ниже).
3. Часовой пояс, пользователь (логин + пароль; на сервере часто включают **OpenSSH**).
4. Перезагрузка, извлечь USB.

### Разметка диска (desktop, простой случай)

Установщик Ubuntu часто предлагает один раздел `/` на **ext4** + **swap** (файл или раздел). Этого достаточно для обучения и домашнего ПК.

### Разметка (server / «вручную»)

| Точка монтирования | Размер (ориентир) | Зачем |
|--------------------|-------------------|--------|
| `/boot` или `/boot/efi` | 512 МБ – 1 ГБ | Загрузчик (EFI — FAT32) |
| `/` | 20–40+ ГБ | Система и `/usr` |
| `/var` | 10–50+ ГБ | Логи, кэши, БД (на сервере выделяют отдельно) |
| `/home` | остальное | Данные пользователей (переустановка без потери home) |
| swap | 0–8 ГБ | Подкачка; на RAM ≥ 16 ГБ часто swap‑file или небольшой swap |

**LVM** — удобно менять размер томов позже; **шифрование (LUKS)** — для ноутбука с чувствительными данными (пароль при каждой загрузке).

### VPS «с нуля»

Обычно один диск, раздел `/` ext4, swap по желанию провайдера. Первый вход — SSH ключ или пароль из панели; сразу:

```bash
sudo apt update              # обновить индексы пакетов из репозиториев
sudo apt upgrade -y          # установить доступные обновления (-y — без вопроса)
```

---

## 4. Первые шаги после установки

```bash
sudo apt update              # списки пакетов с зеркал
sudo apt upgrade -y          # обновить установленные пакеты
sudo apt install -y curl wget git vim htop   # сеть, git, редактор, монитор процессов

whoami                       # имя текущего пользователя
hostname                     # имя машины в сети
uname -a                     # ядро, архитектура, версия ОС
```

- Настроить **SSH‑ключи** на сервере (`~/.ssh/authorized_keys`), отключить вход по паролю root (когда ключи работают).
- **Firewall** (если используете ufw):

```bash
sudo ufw enable              # включить фильтрацию (подтвердить правило для SSH)
sudo ufw allow OpenSSH       # разрешить вход по SSH (порт 22)
sudo ufw status verbose      # проверить правила
```
- Резервная копия важных данных — до экспериментов с разделами и purge пакетов.

Дальнейшая эксплуатация: [server_healthcheck_quick.md](server_healthcheck_quick.md), [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md).

---

## 5. Файловые системы и диски

### Основные ФС в Linux

| ФС | Где встречается | Кратко |
|----|-----------------|--------|
| **ext4** | Ubuntu по умолчанию | Универсальный выбор для `/` |
| **xfs** | RHEL, некоторые серверы | Хорош на больших томах |
| **btrfs** | Fedora, опционально | Снимки, сжатие; сложнее администрировать |
| **vfat (FAT32)** | `/boot/efi` | EFI‑раздел |
| **swap** | раздел или файл | Подкачка; см. `free -h` ниже |

```bash
free -h                      # оперативная память и swap (раздел или swapfile)
```

Просмотр дисков:

```bash
lsblk -f                     # диски, ФС, UUID, LABEL, точки монтирования
df -hT                       # занятое место и тип ФС (human-readable)
sudo blkid                   # UUID, TYPE и PARTUUID разделов
findmnt                      # дерево смонтированных файловых систем
```

### Монтирование

- **`/etc/fstab`** — что монтировать при загрузке (UUID надёжнее имён `/dev/sda1`).

```bash
sudo mount /dev/sdX1 /mnt    # временно смонтировать раздел в /mnt (sdX1 — ваш раздел)
sudo umount /mnt             # отмонтировать, когда закончили
```

Пример строки fstab (ext4):

```text
# UUID=...  точка_монтирования  тип_ФС  опции  dump  fsck_pass
UUID=xxxx-xxxx  /data  ext4  defaults  0  2
# defaults — стандартные опции; последняя 2 — порядок проверки fsck при загрузке
```

UUID узнать:

```bash
sudo blkid                   # строка UUID=... для нужного раздела
```

### Inode

Диск может быть «полным» по **числу файлов**, а не по байтам:

```bash
df -ih                       # использование inode (лимит «числа файлов», не только места)
```

Много мелких файлов (почта, кэш, `node_modules`) — типичная причина.

### LVM (кратко)

Физические тома (PV) → группа (VG) → логические тома (LV) → ext4/xfs сверху.

```bash
sudo pvs                     # physical volumes (физические тома LVM)
sudo vgs                     # volume groups (группы томов)
sudo lvs                     # logical volumes (логические тома под mount)
```

Расширение LV — отдельная тема; перед правками — backup/snapshot.

---

## 6. Иерархия каталогов (FHS): где что лежит

| Путь | Назначение |
|------|------------|
| `/` | Корень; все пути начинаются отсюда |
| `/home/USER` | Файлы пользователя |
| `/root` | Home суперпользователя |
| `/etc` | Конфиги (nginx, ssh, сеть) |
| `/var` | Меняющиеся данные: логи, очереди, БД |
| `/var/log` | Логи ([logs_cheatsheet.md](logs_cheatsheet.md)) |
| `/usr` | Программы и библиотеки (большая часть пакетов) |
| `/bin`, `/sbin` | Базовые команды (часто симлинки в `/usr`) |
| `/tmp` | Временные файлы (очищаются при перезагрузке) |
| `/opt` | Сторонний софт вне apt |
| `/dev` | Устройства |
| `/proc` | Информация о ядре и процессах (`/proc/PID/`) |
| `/sys` | Интерфейс к ядру (sysfs) |
| `/run` | Runtime‑данные (PID‑файлы, sockets) |
| `/boot` | Ядро и initramfs |
| `/mnt`, `/media` | Точки для временного монтирования |

Один «файл» в `/proc` или `/sys` — не обычный файл на диске, а интерфейс ядра.

---

## 7. Ориентирование в системе: shell и пути

### Запуск терминала

- Desktop: Terminal, Konsole, GNOME Terminal.
- Server: SSH:

```bash
ssh user@host                # вход на удалённый хост (логин user, имя или IP host)
ssh -p 2222 user@host        # нестандартный порт SSH
```

### Пути

- **Абсолютный:** `/etc/passwd` — от корня `/`.
- **Относительный:** `cd Documents` — от текущего каталога.
- `.` — текущий каталог; `..` — родитель; `~` — home (`/home/user`).
- Пробелы в именах: кавычки `"My File.txt"` или экранирование `\ `.

### Базовые команды навигации

```bash
pwd                          # полный путь текущего каталога
cd ~                         # перейти в домашний каталог ($HOME)
cd /var/log                  # перейти в каталог (абсолютный путь)
cd ..                        # на уровень вверх (родительский каталог)
cd -                         # вернуться в предыдущий каталог
ls -la                       # список файлов, включая скрытые (имена с точки)
ls -lh                       # размеры в K/M/G и права доступа
tree -L 2 /etc/nginx         # дерево каталогов, глубина 2 (нужен пакет tree)
```

### Просмотр и поиск

```bash
cat /etc/os-release          # название и версия дистрибутива (PRETTY_NAME)
less /var/log/syslog         # постраничный просмотр (/ — поиск, q — выход)
head -n 20 file              # первые 20 строк файла
tail -n 50 file              # последние 50 строк
tail -F /var/log/syslog      # «живой» хвост лога (новые строки по мере записи)
file ./script.sh             # тип содержимого (скрипт, бинарник, текст)

find /var/log -name "*.log" -type f    # найти обычные файлы *.log под /var/log
find ~ -maxdepth 2 -name "*.md"        # *.md в home, не глубже 2 уровней
locate nginx.conf            # быстрый поиск по индексу (пакет mlocate, updatedb)
```

### Копирование, перемещение, права

```bash
cp -a src dst                # копия с правами, владельцем и метками времени (-a = archive)
mv old new                   # переименовать или перенести файл/каталог
mkdir -p project/src         # создать вложенные каталоги (-p — родителей, если нет)
rm file                      # удалить файл (каталоги — rm -r, осторожно с rm -rf)
chmod u+x script.sh          # добавить право на выполнение владельцу (u+x)
chmod 644 file               # права в octal: rw-r--r--
chown user:group file        # сменить владельца и группу (обычно sudo)
```

### Пользователи и sudo

- **root** — полные права; в Ubuntu повседневно — свой user + **`sudo`**.

```bash
sudo command                 # выполнить одну команду от root
sudo -i                      # интерактивная shell root (осторожно)
groups                       # группы текущего пользователя
id                           # uid, gid и все группы

sudo adduser deploy            # создать пользователя (интерактивно задаст пароль)
sudo usermod -aG sudo deploy   # добавить в группу sudo (право на sudo на Ubuntu)
```

### Дистрибутив и версия

```bash
cat /etc/os-release          # ID, VERSION_ID, PRETTY_NAME дистрибутива
lsb_release -a               # то же в формате LSB (пакет lsb-release)
hostnamectl                  # имя хоста, ОС, ядро, архитектура
```

### Пакеты (связь «команда ↔ система»)

```bash
which nginx                  # путь к исполняемому файлу nginx в PATH
dpkg -S /usr/bin/nginx       # какой .deb‑пакет установил этот файл
apt search htop              # поиск пакета по имени/описанию
apt show htop                # описание, версия, зависимости пакета
sudo apt install htop        # установить пакет
sudo apt remove htop         # удалить пакет, конфиги оставить
```

Подробнее про «чужой» процесс и пакет: [seek_and_destroy.md](seek_and_destroy.md).

### Справка

```bash
man ls                       # руководство по команде ls (q — выход)
man systemd.unit             # документация по unit-файлам systemd
command -v ls                # путь или имя команды (учитывает alias)
type cd                      # builtin, alias или file для cd
help cd                      # справка по встроенной команде bash
```

Редактор **vim** на сервере: [vim_cheatsheet.md](vim_cheatsheet.md). Настройка shell: [bashrc_cheatsheet.md](bashrc_cheatsheet.md).

---

## 8. Режимы системы (ориентир)

| Режим | Как попасть | Зачем |
|-------|-------------|--------|
| **multi-user.target** | обычная загрузка | Сервер / консоль |
| **graphical.target** | desktop | GUI |
| **recovery** | GRUB → Advanced → recovery | root shell, fsck, сеть |
| **Live USB** | загрузка с флешки | починка диска, chroot, backup |

```bash
systemctl get-default        # default target (graphical.target или multi-user.target)
systemctl list-units --type=target   # список targets (режимов загрузки)
```

---

## 9. Мини‑чек‑лист «я только что установил Linux»

```bash
sudo apt update              # индексы пакетов
sudo apt upgrade -y          # обновления системы

cat /etc/os-release          # какая ОС установлена
lsblk -f                     # диски и разделы
df -hT                       # свободное место

pwd                          # навигация: где вы в файловой системе
cd ~ && ls -la               # домашний каталог и его содержимое
less /etc/hosts              # пример постраничного просмотра

journalctl -p err -b         # ошибки с текущей загрузки (systemd)
ls /var/log                  # классические файлы логов
```

Привычка на VPS: раз в неделю [server_healthcheck_quick.md](server_healthcheck_quick.md).

---

## 10. Что изучать дальше в этом репозитории

| Тема | Файл |
|------|------|
| Быстрая диагностика | [server_healthcheck_full.md](server_healthcheck_full.md) |
| Инциденты | [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md) |
| Логи | [logs_cheatsheet.md](logs_cheatsheet.md) |
| Мониторинг | [realtime_monitoring_cheatsheet.md](realtime_monitoring_cheatsheet.md) |
