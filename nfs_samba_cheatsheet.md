# NFS и Samba: сетевые каталоги в Linux

← [README](README.md) · монтирование: [linux_install_and_basics.md](linux_install_and_basics.md) · firewall: [firewall_cheatsheet.md](firewall_cheatsheet.md)

**NFS** и **Samba (SMB/CIFS)** — способы отдать каталог по сети другим машинам. Оба часто используют в LAN (дом, офис, homelab), реже на публичном VPS без VPN.

| | **NFS** | **Samba (SMB)** |
|---|---------|------------------|
| Протокол | NFSv3/v4 (Unix‑традиция) | SMB/CIFS (как Windows «шара») |
| Типичные клиенты | Linux, Unix, macOS (частично) | Windows, Linux, macOS |
| Аутентификация | IP + Unix UID/GID (Kerberos возможен) | Логин/пароль Samba, домен AD |
| Порты (ориентир) | 2049/tcp (v4), rpc (v3) | 445/tcp (SMB), 139 (legacy) |
| Когда выбирать | Linux↔Linux, NAS для Linux | Доступ с Windows, mixed LAN |

В блоках `bash` комментарии после `#` поясняют команду.

---

## 1. Общие понятия

- **Экспорт / share** — каталог на **сервере**, который отдают в сеть.
- **Монтирование (mount)** — подключить удалённый каталог в локальную точку (`/mnt/nas`).
- **`/etc/fstab`** — автомount при загрузке (с `_netdev`, чтобы дождаться сети).
- **Права** — на NFS сильно зависят от совпадения **UID/GID** на клиенте и сервере; у Samba — от пользователя Samba и опций `create mask` / `force user`.

---

## 2. NFS — сервер (Debian/Ubuntu)

### Установка и каталог

```bash
sudo apt update                              # индексы пакетов
sudo apt install -y nfs-kernel-server          # демон NFS (kernel server)
sudo mkdir -p /srv/nfs/data                    # каталог, который отдаём в сеть
sudo chown nobody:nogroup /srv/nfs/data        # пример; часто свой user/group
sudo chmod 755 /srv/nfs/data                   # права на экспортируемый путь
```

### `/etc/exports` — что и кому разрешено

```bash
sudoedit /etc/exports                        # правка списка экспортов
```

Пример строк (синтаксис: `путь клиент(опции)`):

```text
# /etc/exports — комментарии с #

/srv/nfs/data  192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash)
/srv/nfs/backup 192.168.1.10(ro,sync,no_subtree_check)
```

| Опция | Комментарий |
|-------|-------------|
| `rw` / `ro` | Запись / только чтение |
| `sync` | Запись на диск перед ответом клиенту (надёжнее, медленнее) |
| `no_subtree_check` | Часто на одном каталоге — меньше странностей |
| `no_root_squash` | root на клиенте = root на NFS (опасно; только доверенная LAN) |
| `root_squash` | root на клиенте мапится в nobody (безопаснее по умолчанию) |

Применить экспорты:

```bash
sudo exportfs -ra                            # перечитать exports и обновить экспорт
sudo exportfs -v                             # показать активные экспорты
showmount -e localhost                       # что видит клиент с этой машины
```

Сервис и firewall:

```bash
sudo systemctl enable --now nfs-server       # Ubuntu 22.04+; раньше иногда nfs-kernel-server
sudo systemctl status nfs-server
sudo ss -tulpen | grep 2049                  # NFSv4 слушает 2049

sudo ufw allow from 192.168.1.0/24 to any port nfs   # если ufw включён
# или точечно: sudo ufw allow from 192.168.1.0/24 to any port 2049
```

---

## 3. NFS — клиент

### Установка и ручной mount

```bash
sudo apt install -y nfs-common               # клиент NFS
sudo mkdir -p /mnt/nas                       # локальная точка монтирования

sudo mount -t nfs4 192.168.1.5:/srv/nfs/data /mnt/nas    # NFSv4 (рекомендуется)
# NFSv3 (legacy): sudo mount -t nfs 192.168.1.5:/srv/nfs/data /mnt/nas

df -hT /mnt/nas                              # проверить, что смонтировано
mount | grep nfs                             # строка mount в системе
sudo umount /mnt/nas                         # отмонтировать
```

### `/etc/fstab` — автomount

```bash
sudoedit /etc/fstab
```

Строка (подставьте IP и путь):

```text
192.168.1.5:/srv/nfs/data  /mnt/nas  nfs4  defaults,_netdev,timeo=14,retrans=2  0  0
```

```bash
sudo mount -a                                # проверить fstab (все строки)
findmnt /mnt/nas                             # дерево mount
```

Опции **`_netdev`** — не mount до поднятия сети. При проблемах загрузки — `nofail` (не падать, если NAS offline).

---

## 4. NFS — права, UID/GID, диагностика

На NFS «владелец» файла на сервере — числовой **UID**. Если на клиенте другой пользователь с тем же UID — увидит чужие права.

```bash
id deploy                                    # uid/gid на клиенте
ls -ln /mnt/nas                              # числовые uid/gid на смонтированном каталоге
sudo chown 1000:1000 /srv/nfs/data/file    # на сервере — явный uid (если нужно)
```

```bash
rpcinfo -p 192.168.1.5                       # RPC (для NFSv3; на v4 может быть минимально)
sudo journalctl -u nfs-server -n 30 --no-pager   # логи сервера
dmesg | tail                                 # ошибки mount на клиенте
```

Типичные ошибки: **permission denied** (exports/ro, root_squash), **stale file handle** (перезапуск сервера — remount), firewall блокирует 2049.

---

## 5. Samba — сервер (Debian/Ubuntu)

Samba даёт **Windows‑совместимые шары** и может участвовать в домене Active Directory (здесь — простой standalone).

### Установка

```bash
sudo apt update
sudo apt install -y samba                    # smbd, nmbd, smbclient
sudo systemctl enable --now smbd nmbd        # smbd — файлы; nmbd — NetBIOS (legacy)
sudo systemctl status smbd
sudo ss -tulpen | grep -E ':445|:139'        # SMB порты
```

### Пользователь Samba (отдельно от Unix)

Samba‑пароль привязывают к **существующему** Unix‑пользователю:

```bash
sudo adduser --no-create-home smbuser        # пример системного user (или используйте deploy)
sudo smbpasswd -a deploy                     # задать SMB‑пароль для Unix user deploy
sudo smbpasswd -e deploy                     # включить учётку Samba
sudo pdbedit -L                              # список пользователей Samba
```

### `/etc/samba/smb.conf`

```bash
sudoedit /etc/samba/smb.conf
sudo testparm                                # проверка синтаксиса и итоговый конфиг
sudo testparm -s                             # вывести effective config
sudo systemctl reload smbd                   # применить после правок
```

Минимальный пример **в конец файла** (глобальная секция `[global]` обычно уже есть):

```ini
# /etc/samba/smb.conf — фрагмент

[data]
   comment = Shared data
   path = /srv/samba/data
   browseable = yes
   read only = no
   guest ok = no
   valid users = deploy
   create mask = 0664
   directory mask = 0775
```

```bash
sudo mkdir -p /srv/samba/data
sudo chown deploy:deploy /srv/samba/data
sudo chmod 2775 /srv/samba/data              # setgid — новые файлы группы deploy
```

Публичная **guest**‑шара (только в доверенной LAN, осторожно):

```ini
[public]
   path = /srv/samba/public
   guest ok = yes
   read only = yes
```

Firewall:

```bash
sudo ufw allow samba                         # профиль Ubuntu для 445/139
sudo ufw status
```

---

## 6. Samba — клиент Linux

### Просмотр шар и интерактив

```bash
sudo apt install -y cifs-utils smbclient     # mount.cifs и клиент smb
smbclient -L //192.168.1.5 -U deploy         # список шар на сервере (-U user)
smbclient //192.168.1.5/data -U deploy       # интерактив: ls, get, put, exit
```

### Mount CIFS

```bash
sudo mkdir -p /mnt/smb
sudo mount -t cifs //192.168.1.5/data /mnt/smb -o username=deploy,uid=1000,gid=1000,file_mode=0664,dir_mode=0775
# спросит пароль Samba; или credentials file (см. ниже)

df -hT /mnt/smb
sudo umount /mnt/smb
```

Файл учётных данных (права **600**):

```bash
sudoedit /root/.smbcredentials               # или ~/.smbcredentials для user mount
```

```text
username=deploy
password=SECRET
domain=WORKGROUP
```

```bash
sudo chmod 600 /root/.smbcredentials
sudo mount -t cifs //192.168.1.5/data /mnt/smb -o credentials=/root/.smbcredentials,uid=1000,gid=1000
```

### fstab для CIFS

```text
//192.168.1.5/data  /mnt/smb  cifs  credentials=/root/.smbcredentials,uid=1000,gid=1000,_netdev,nofail  0  0
```

```bash
sudo mount -a                                # проверка
```

---

## 7. Samba — доступ с Windows (кратко)

На Windows: **Проводник → \\192.168.1.5\data** → логин `deploy` и Samba‑пароль.

Имя NetBIOS сервера (если нужно): `server string` / имя хоста в `[global]`:

```ini
[global]
   workgroup = WORKGROUP
   server min protocol = SMB2
```

---

## 8. NFS vs Samba — что выбрать

| Сценарий | Выбор |
|----------|--------|
| Несколько Linux‑серверов, один NAS | NFSv4 |
| Windows + Linux в одной сети | Samba |
| Виртуалки монтируют одно хранилище | NFS (производительность, Unix права) |
| «Как сетевая папка Windows» | Samba |
| Только передача файлов разово | [ssh_cheatsheet.md](ssh_cheatsheet.md) (`rsync`/`scp`) |

**Не выставляйте** NFS и Samba в интернет без VPN и жёсткой фильтрации — bruteforce и misconfig частая причина инцидентов.

---

## 9. Безопасность (минимум)

```bash
# ограничить NFS подсеть в /etc/exports — не * без крайней нужды
# Samba: guest ok = no для чувствительных данных
# отключить SMB1 (старые клиенты): server min protocol = SMB2_02 в smb.conf

sudo testparm -v | grep -i protocol          # какие SMB версии разрешены
grep -v '^#' /etc/exports | grep -v '^$'     # активные строки exports
```

- Регулярные обновления: `sudo apt upgrade` (samba, nfs-kernel-server).
- Бэкапы **на сервере шар**, не только через сеть.
- Для production — рассмотреть **Kerberos NFS**, **AD‑интеграцию Samba**, отдельный VLAN для storage.

---

## 10. Диагностика

| Симптом | NFS | Samba |
|---------|-----|--------|
| mount висит | firewall, неверный IP exports | 445 закрыт, неверное имя шары |
| Permission denied | UID/GID, ro, root_squash | valid users, SMB пароль, права path |
| Нет в списке шар | `exportfs -v` | `testparm`, `smbclient -L` |

```bash
# NFS
sudo exportfs -v
showmount -e 192.168.1.5

# Samba
sudo testparm
sudo journalctl -u smbd -n 40 --no-pager
sudo smbstatus                               # кто подключён (на сервере)
```

При переполнении диска на шаре — [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md).

---

## 11. Мини‑чек‑лист

**NFS server:** `nfs-kernel-server` → `/etc/exports` → `exportfs -ra` → ufw → client `mount -t nfs4`.

**NFS client:** `nfs-common` → `mount` / fstab с `_netdev`.

**Samba server:** `samba` → каталог + права → `smb.conf` → `testparm` → `smbpasswd -a` → `reload smbd`.

**Samba client:** `cifs-utils` → `mount -t cifs` или `smbclient`.

Правка конфигов: [vim_cheatsheet.md](vim_cheatsheet.md).
