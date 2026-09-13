# Бэкапы на Linux: стратегия, инструменты, восстановление

← [README](README.md) · передача по сети: [ssh_cheatsheet.md](ssh_cheatsheet.md) · сетевое хранилище: [nfs_samba_cheatsheet.md](nfs_samba_cheatsheet.md) · место на диске: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)

**Бэкап (резервная копия)** — копия данных и конфигов, из которой можно **восстановиться** после сбоя, ошибки админа, компрометации или удаления.

Правило **3-2-1** (ориентир):

- **3** копии данных (рабочая + два бэкапа),
- **2** разных носителя/типа (диск сервера + NAS/облако),
- **1** копия **вне** площадки (другой ДЦ, другой провайдер, offline).

Бэкап без **проверки восстановления** — только надежда. Раз в квартал делайте тестовый restore на staging.

В блоках `bash` комментарии после `#` поясняют команду.

---

## 1. Что бэкапить на VPS / сервере

| Категория | Пути (примеры) | Комментарий |
|-----------|----------------|-------------|
| Конфиги | `/etc` | nginx, ssh, systemd, cron, сеть |
| Данные приложений | `/var/www`, `/srv`, `/opt` | сайты, uploads |
| Базы данных | дампы, не только файлы PGDATA | нужен consistent dump |
| Пользователи | `/home` | если не только deploy |
| Мета | список пакетов, `crontab -l`, unit‑файлы | быстрое воссоздание |

**Не полагайтесь** только на snapshot диска без понимания: БД при live‑snapshot может быть **несогласованной** без freeze/dump.

```bash
dpkg --get-selections > ~/pkg-list.txt    # список установленных пакетов (Debian/Ubuntu)
crontab -l > ~/crontab-user.txt            # cron текущего пользователя
sudo crontab -l > ~/crontab-root.txt       # cron root
systemctl list-unit-files --type=service > ~/systemd-services.txt   # обзор сервисов
```

---

## 2. Виды бэкапов

| Тип | Суть | Плюсы / минусы |
|-----|------|----------------|
| **Полный (full)** | Вся выборка целиком | Простое восстановление; долго и много места |
| **Инкрементальный** | Только изменения с прошлого бэкапа | Экономия места; цепочка restore |
| **Дифференциальный** | Изменения с последнего full | Компромисс full/incremental |
| **Snapshot** | Снимок ФС/тома (LVM, ZFS, cloud) | Быстро; не замена offsite |
| **Репликация** | Живая копия (rsync, streaming) | RPO маленький; не защита от удаления/шифровальщика без versioning |

**RPO** — сколько данных можно потерять (интервал бэкапа). **RTO** — сколько времени допустим простой.

---

## 3. Простые архивы: `tar` и `gzip`

Подходит для `/etc`, небольших сайтов, разовых снимков. Подробнее форматы и сжатие: [archives_compression_cheatsheet.md](archives_compression_cheatsheet.md).

```bash
sudo tar -czvf /backup/etc-$(date +%F).tar.gz -C / etc   # архив /etc с датой в имени
# -c создать, -z gzip, -v verbose, -f файл, -C корень для относительных путей

tar -tzf /backup/etc-2025-01-15.tar.gz | head            # список файлов в архиве
sudo tar -xzvf /backup/etc-2025-01-15.tar.gz -C /tmp/restore   # распаковать в /tmp/restore

sudo tar -czvf /backup/www.tar.gz --exclude='*/cache/*' /var/www   # исключить cache
```

Права и владельцы при restore в `/` — осторожно; для проверки распаковывайте в **`/tmp/restore`**.

```bash
du -sh /backup/*.tar.gz                    # размер архивов
```

---

## 4. `rsync` — синхронизация и «живые» копии

Идеален для каталогов на другой диск, NAS или удалённый сервер. См. [ssh_cheatsheet.md](ssh_cheatsheet.md).

```bash
sudo rsync -aAXHv --delete /etc/ /backup/etc-mirror/   # зеркало /etc (--delete убирает лишнее в mirror)
# -a archive, -A ACL, -X xattr, -H hard links (осторожно с --delete на production)

rsync -aAXHv /var/www/ user@backup-host:/backups/www/   # на удалённый хост по SSH
rsync -aAXHv -e 'ssh -p 2222' ~/data/ user@host:~/backups/data/   # нестандартный SSH порт

# dry-run — показать, что бы изменилось, без копирования:
rsync -aAXHn --delete /var/www/ /backup/www-mirror/
```

**Versioning:** plain rsync с `--delete` **перезапишет** удаление на источнике. Для защиты от ransomware смотрите **restic/borg** (immutable snapshots) или snapshots на NAS.

---

## 5. Дампы баз данных

### PostgreSQL

```bash
sudo -u postgres pg_dump mydb > /backup/mydb-$(date +%F).sql      # логический дамп одной БД
sudo -u postgres pg_dumpall > /backup/pg-all-$(date +%F).sql       # все БД + роли (globals)

gzip /backup/mydb-$(date +%F).sql                                  # сжать
zcat /backup/mydb-2025-01-15.sql.gz | sudo -u postgres psql mydb   # restore в БД (осторожно, перезапишет)

sudo -u postgres pg_dump -Fc mydb -f /backup/mydb.dump             # custom format (-Fc), удобен pg_restore
sudo -u postgres pg_restore -d mydb /backup/mydb.dump              # restore из custom
```

### MySQL / MariaDB

```bash
mysqldump -u root -p --single-transaction mydb > /backup/mydb-$(date +%F).sql   # InnoDB без длинной блокировки
mysqldump -u root -p --all-databases > /backup/mysql-all-$(date +%F).sql

mysql -u root -p mydb < /backup/mydb-2025-01-15.sql              # restore (проверьте на копии!)
```

Пароли в cron не храните в plain text — **`~/.my.cnf`** с `chmod 600` или socket auth.

---

## 6. Restic и Borg — dedup, шифрование, offsite

Для регулярных бэкапов на **S3**, **B2**, SFTP, локальный каталог.

### Restic (пример)

```bash
sudo apt install -y restic                    # установка
export RESTIC_REPOSITORY=sftp:user@backup:/restic/repo   # или s3:s3.amazonaws.com/bucket
export RESTIC_PASSWORD='strong-repo-password' # пароль репозитория (не терять!)
restic init                                   # один раз — создать repo
restic backup /etc /var/www /home/deploy      # backup путей
restic snapshots                              # список снимков
restic restore latest --target /tmp/restic-restore   # восстановить последний в каталог
restic check                                  # проверка целостности repo
restic forget --keep-daily 7 --keep-weekly 4 --prune   # retention + очистка
```

### BorgBackup (кратко)

```bash
sudo apt install -y borgbackup
borg init --encryption=repokey /backup/borg-repo   # инициализация
export BORG_PASSPHRASE='...'                       # passphrase
borg create /backup/borg-repo::{hostname}-{now:%Y-%m-%d} /etc /var/www
borg list /backup/borg-repo
borg extract /backup/borg-repo::archive-name       # в текущий каталог
```

Выбор: **restic** — проще multi‑platform и S3; **borg** — часто быстрее dedup на своём сервере.

---

## 7. Снимки томов и облако VPS

```bash
# LVM snapshot (краткоживущий, для consistent backup большого тома)
sudo lvcreate -L 5G -s -n snap_home /vg/home   # snapshot 5G (нужен свободный VG)
sudo mount /dev/vg/snap_home /mnt/snap
sudo tar -czf /backup/home-from-snap.tar.gz -C /mnt/snap .
sudo umount /mnt/snap
sudo lvremove -f /vg/snap_home                   # удалить snapshot после копирования
```

**Snapshot провайдера** (Hetzner, DO, AWS EBS) — образ диска на момент времени. Плюс: быстрый откат всей VM. Минус: не заменяет географически удалённый бэкап; стоимость; consistent DB — по-прежнему дампы.

---

## 8. Расписание: cron и systemd timer

### Cron

```bash
crontab -e                                   # редактор расписания пользователя
```

```cron
# /etc/cron.d/mybackup — или crontab -e
# m h dom mon dow user command
0 3 * * * root /usr/local/bin/backup-etc.sh >> /var/log/backup-etc.log 2>&1
```

Пример скрипта **`/usr/local/bin/backup-etc.sh`**:

```bash
#!/bin/bash
set -euo pipefail                            # выход при ошибке, неисп. переменные — ошибка
BACKUP_DIR=/backup
mkdir -p "$BACKUP_DIR"
tar -czf "$BACKUP_DIR/etc-$(date +%F).tar.gz" -C / etc
find "$BACKUP_DIR" -name 'etc-*.tar.gz' -mtime +14 -delete   # хранить 14 дней
```

```bash
sudo chmod +x /usr/local/bin/backup-etc.sh   # исполняемый
sudo /usr/local/bin/backup-etc.sh            # ручной прогон
```

### Systemd timer (альтернатива cron)

```bash
sudo systemctl list-timers --all             # активные timers
# unit: backup-etc.service + backup-etc.timer в /etc/systemd/system/
```

---

## 9. Куда складывать бэкапы

| Место | Комментарий |
|-------|-------------|
| Отдельный раздел `/backup` | Лучше, чем тот же диск, что `/` — но не offsite |
| NAS по [NFS/Samba](nfs_samba_cheatsheet.md) | Homelab, LAN |
| Второй VPS + `rsync` по SSH | Простой offsite |
| Object storage (S3, B2) + restic | Классика cloud offsite |
| Внешний USB | Offline; не на production 24/7 |

```bash
df -hT /backup                               # место под бэкапы
du -sh /backup/* | sort -h                   # что занимает больше всего
```

Перед `apt purge` и массовым `rm` — [seek_and_destroy.md](seek_and_destroy.md): сначала бэкап или snapshot.

---

## 10. Шифрование и секреты

```bash
# GPG шифрование архива (ключ заранее создать: gpg --gen-key)
tar -czf - /etc | gpg -e -r admin@example.com > /backup/etc.tar.gz.gpg
gpg -d /backup/etc.tar.gz.gpg | tar -xzf - -C /tmp/restore

# restic/borg шифруют repo сами — храните RESTIC_PASSWORD/BORG_PASSPHRASE в безопасном месте
```

Не коммитьте пароли бэкапов в git. Для cron — root‑only файлы `600`, vault, или переменные из **systemd credentials**.

---

## 11. Проверка восстановления (обязательно)

```bash
# 1) распаковать в изолированный каталог
mkdir -p /tmp/restore-test && tar -xzf /backup/etc-DATE.tar.gz -C /tmp/restore-test

# 2) сравнить контрольные файлы
diff -u /tmp/restore-test/etc/nginx/nginx.conf /etc/nginx/nginx.conf

# 3) для БД — поднять тестовый инстанс или БД mydb_restore
sudo -u postgres createdb mydb_test
zcat /backup/mydb.sql.gz | sudo -u postgres psql mydb_test
```

Зафиксируйте в runbook: **кто**, **как often**, **сколько времени** занял restore.

---

## 12. Типичные ошибки

| Ошибка | Последствие |
|--------|-------------|
| Бэкап на том же диске, что данные | Потеря всего при отказе диска |
| `--delete` rsync без versioning | Ransomware/ошибка удалит и зеркало |
| Live copy файлов БД без dump | Битая или неподнимаемая БД |
| Никогда не тестировали restore | «Бэкапы были, но не открывались» |
| Диск `/backup` забит | cron молча не пишет — мониторить место |

```bash
tail -50 /var/log/backup-etc.log             # логи cron-скрипта
journalctl -u backup-etc.service -n 30         # если systemd
```

---

## 13. Мини‑чек‑лист VPS

1. Определить **RPO/RTO** (хотя бы грубо).
2. **Ежедневно:** дампы БД + `tar` или `restic` для `/etc` и данных.
3. **Offsite:** второй сервер, S3, или snapshot провайдера + копия дампов вне площадки.
4. **Retention:** 7 daily / 4 weekly (пример) — `find -mtime` или `restic forget`.
5. **Мониторинг:** место на диске ([common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)), успех cron (log/mail).
6. **Раз в квартал:** тест restore.

---

## 14. Шпаргалка команд

```bash
sudo tar -czvf /backup/etc-$(date +%F).tar.gz -C / etc     # архив конфигов
rsync -aAXHv /var/www/ /backup/www/                        # зеркало без --delete (безопаснее для начала)
sudo -u postgres pg_dump mydb | gzip > /backup/mydb.sql.gz  # PostgreSQL
restic backup /etc /var/www && restic snapshots             # restic (repo настроен)
crontab -l                                                  # текущее расписание
df -hT /backup                                              # место
```

Правка скриптов: [vim_cheatsheet.md](vim_cheatsheet.md).
