# Права, владельцы, umask

← [README](README.md) · основы: [linux_install_and_basics.md](linux_install_and_basics.md) · nginx: [nginx_cheatsheet.md](nginx_cheatsheet.md) · NFS: [nfs_samba_cheatsheet.md](nfs_samba_cheatsheet.md)

**Permission denied**, nginx «не видит» файл, скрипт не **executable** — почти всегда **user/group/mode** или **path** (родительские каталоги).

---

## 1. ls и chmod

```bash
ls -la file
stat file
chmod 644 file                              # rw-r--r--
chmod 755 dir                                 # rwx для owner, rx для остальных
chmod u+x script.sh
chmod -R o-w /var/www                         # осторожно -R
```

| Цифра | Права |
|-------|--------|
| 7 | rwx |
| 6 | rw- |
| 5 | r-x |
| 4 | r-- |

---

## 2. chown

```bash
sudo chown user:group file
sudo chown -R www-data:www-data /var/www/site
id www-data
groups user
```

---

## 3. umask

```bash
umask                                       # текущая маска (часто 0022)
touch newfile && ls -l newfile              # default file 644 при umask 022
```

В `~/.bashrc` или `/etc/profile` — [bashrc_cheatsheet.md](bashrc_cheatsheet.md).

---

## 4. namei — права на всём пути

```bash
namei -l /var/www/html/index.html
```

Каждый каталог в пути должен быть **executable (x)** для пользователя, который идёт к файлу.

---

## 5. Типовые кейсы

### Nginx 403 / Permission denied

```bash
namei -l /var/www/site/public/index.html
# home /var/www часто должны быть o+x для «others traverse»
sudo chmod o+x /var/www
sudo chown -R www-data:www-data /var/www/site
```

### Скрипт «Permission denied» при ./run.sh

```bash
chmod +x run.sh
head -1 run.sh                              # shebang #!/usr/bin/env bash
```

### SSH «Permissions too open»

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519 ~/.ssh/authorized_keys
```

[ssh_cheatsheet.md](ssh_cheatsheet.md).

---

## 6. ACL (кратко)

```bash
sudo apt install -y acl
getfacl file
setfacl -m u:www-data:rx dir
setfacl -d -m u:www-data:rx dir              # default ACL для новых файлов
```

На NFS/Samba ACL сложнее — [nfs_samba_cheatsheet.md](nfs_samba_cheatsheet.md).

---

## 7. Sudo и root

```bash
sudo -u www-data cat /var/www/private/file
sudo find /var/www -type f ! -perm -g+r
```

---

## 8. Шпаргалка

```text
ls -la    stat    namei -l PATH
chmod/chown    umask    SSH 700/600
www-data + o+x на родительских dir
```
