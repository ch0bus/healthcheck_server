# Архивирование и сжатие в Linux

← [README](README.md) · бэкапы: [backups_cheatsheet.md](backups_cheatsheet.md) · логи `.gz`: [logs_cheatsheet.md](logs_cheatsheet.md)

**Архив** — один файл/поток из многих (сохраняет пути, права, иногда владельцев). **Сжатие** — уменьшает размер данных. На Linux чаще всего **вместе**: `tar` + `gzip`/`xz`/`zstd`.

| Расширение | Архив | Сжатие | Комментарий |
|------------|-------|--------|-------------|
| `.tar` | да | нет | «Склеенные» файлы, большой размер |
| `.tar.gz` / `.tgz` | tar | gzip | Универсально, быстро сжать |
| `.tar.bz2` | tar | bzip2 | Медленнее, иногда меньше gzip |
| `.tar.xz` | tar | xz | Сильное сжатие, CPU‑тяжёлое |
| `.tar.zst` | tar | zstd | Современный баланс скорость/размер |
| `.gz` | нет | gzip | Один файл: `gzip file` → `file.gz` |
| `.zip` | да | да | Обмен с Windows, `unzip` |

В блоках `bash` комментарии после `#` поясняют команду.

---

## 1. `tar` — основа архивирования

### Создание

```bash
tar -cf archive.tar /path/to/dir              # только архив, без сжатия (-c create, -f file)
tar -czf archive.tar.gz /path/to/dir          # + gzip (-z)
tar -cjf archive.tar.bz2 /path/to/dir         # + bzip2 (-j)
tar -cJf archive.tar.xz /path/to/dir          # + xz (-J, заглавная J)
tar --zstd -cf archive.tar.zst /path/to/dir   # + zstd (GNU tar 1.31+)

sudo tar -czvf backup.tar.gz -C / etc          # -v список, -C корень (в архиве будет etc/...)
tar -czf site.tar.gz --exclude='node_modules' --exclude='*.log' /var/www   # исключения
```

### Просмотр и распаковка

```bash
tar -tf archive.tar.gz                        # список содержимого (-t)
tar -tzf archive.tar.gz | less                # то же, постранично
tar -xzf archive.tar.gz                       # распаковать в текущий каталог (-x extract)
tar -xzf archive.tar.gz -C /tmp/restore       # распаковать в указанный каталог
tar -xzf archive.tar.gz path/inside/file.txt    # извлечь один файл из архива
tar -xzf archive.tar.gz --strip-components=1  # убрать первый уровень путей при extract
```

### Права и root

```bash
sudo tar -czpf backup.tar.gz /etc             # -p preserve permissions (полезно для /etc)
tar -xzf backup.tar.gz --same-owner             # восстановить владельца (нужен root)
```

Осторожно: `tar -xzf ... -C /` поверх live‑системы может перезаписать конфиги — сначала **`-C /tmp/test`**.

---

## 2. Gzip — `.gz`

```bash
gzip file.txt                                 # сжать → file.txt.gz, исходник удаляется
gzip -k file.txt                              # -k keep — оставить оригинал
gzip -9 file.txt                              # -1..-9 уровень (9 медленнее, чуть меньше)
gunzip file.txt.gz                            # распаковать
zcat file.gz                                  # содержимое на stdout (не создаёт файл)
zless file.gz                                 # просмотр как less
zgrep 'pattern' /var/log/syslog.2.gz          # grep по сжатому логу
pigz -9 backup.tar                            # параллельный gzip (пакет pigz), быстрее на CPU
```

Один поток **не сжимает** уже сжатое (jpg, mp4, `.zip`) — выигрыш минимален.

---

## 3. XZ и Bzip2

```bash
xz file.tar                                   # → file.tar.xz (исходник удаляется)
xz -k -9 file.tar                             # keep + max level
unxz file.tar.xz                              # или: xz -d file.tar.xz
xzcat file.xz | tar -tf -                     # список tar внутри xz без файла на диске

bzip2 file.tar                                # → file.tar.bz2
bunzip2 file.tar.bz2                          # распаковать
bzcat file.bz2 | less                         # просмотр
```

**Когда что:** `gzip` — по умолчанию; **`xz`** — архив «на хранение», редко распаковывают; **`bzip2`** — legacy, встречается в старых дистрибутивах.

---

## 4. Zstd — быстрый современный алгоритм

```bash
sudo apt install -y zstd                      # утилита zstd
zstd file.tar                                   # → file.tar.zst
zstd -d file.tar.zst                            # распаковать
zstd -19 file.tar                               # уровень 1–19 (выше — медленнее, меньше)
tar -I 'zstd -19' -cf archive.tar.zst /data   # tar через zstd (-I compressor)
tar -I zstd -xf archive.tar.zst                 # распаковать
zstdcat file.zst | tar -tf -                    # pipe без промежуточного файла
```

Хорош для **больших бэкапов**, когда важны скорость и разумное сжатие ([backups_cheatsheet.md](backups_cheatsheet.md)).

---

## 5. ZIP — совместимость с Windows

```bash
sudo apt install -y zip unzip                 # если не установлены
zip -r archive.zip folder/                    # рекурсивно (-r)
zip -r archive.zip folder/ -x '*.git*'        # исключить .git
zip -9 archive.zip file1 file2                # -0 без сжатия, -9 max

unzip -l archive.zip                          # список без распаковки
unzip archive.zip -d /tmp/extract             # распаковать в каталог
unzip -j archive.zip '*.txt' -d /tmp/         # -j без путей, только имена файлов
```

**7z** (опционально): `sudo apt install p7zip-full`, `7z a archive.7z dir`, `7z x archive.7z`.

---

## 6. Пipes: сжатие без временного файла

```bash
tar -c /etc | gzip > etc.tar.gz               # tar на stdout → gzip в файл
tar -c /var/log | zstd -19 | ssh user@host 'cat > /backup/logs.tar.zst'   # по SSH на другой сервер

curl -s URL/file.tar.xz | tar -xJ -C /tmp/    # скачать и распаковать (-J = xz)
```

Флаги tar для авто‑формата по расширению: **`-a`** (GNU tar) — подбирает gzip/xz по `-f`.

```bash
tar -caf backup.tar.xz /data                    # -a auto compression по .xz
tar -xaf backup.tar.xz -C /tmp/restore
```

---

## 7. Сравнение (ориентир)

На одном и том же `tar` архиве текст/логи — **gzip** быстрее, **xz** меньше, **zstd** часто лучший компромисс. Бинарники и уже сжатое — разница мала.

| Алгоритм | Скорость сжатия | Степень сжатия | Типичное использование |
|----------|-----------------|-------|-------------------------|
| gzip | высокая | средний | `.tar.gz`, logrotate |
| bzip2 | низкая | средний+ | legacy |
| xz | низкая | высокий | release ISO, долгое хранение |
| zstd | высокая | хороший | бэкапы, современные дистрибутивы |

```bash
# грубое сравнение на одном каталоге (осторожно с местом на диске)
time tar -cf - /etc | gzip   > /tmp/etc.tgz
time tar -cf - /etc | xz     > /tmp/etc.txz
time tar -cf - /etc | zstd   > /tmp/etc.tzst
ls -lh /tmp/etc.t*
```

---

## 8. Целостность и проверка

```bash
gzip -t file.gz                               # test — код 0 если OK
xz -t file.xz
zstd -t file.zst
tar -tzf archive.tar.gz > /dev/null           # «прочитать» весь архив — проверка tar+gzip

sha256sum archive.tar.gz > archive.tar.gz.sha256   # контрольная сумма
sha256sum -c archive.tar.gz.sha256            # проверить (-c check)
gpg --detach-sign archive.tar.gz              # подпись (если настроен GPG)
```

После бэкапа — **размер + checksum** в runbook ([backups_cheatsheet.md](backups_cheatsheet.md)).

---

## 9. Логи и ротация

Logrotate часто создаёт **`*.gz`**:

```bash
ls -lh /var/log/*.gz
zgrep -i error /var/log/nginx/error.log.*.gz
zcat /var/log/syslog.2.gz | tail -100
```

Не удаляйте единственные архивы логов на production без политики — [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md).

```bash
sudo logrotate -d /etc/logrotate.conf           # dry-run ротации
```

---

## 10. Типичные ошибки

| Ошибка | Решение |
|--------|---------|
| `tar: Removing leading '/'` | Норма при `tar -czf x.tgz /abs/path`; лучше `-C /` и относительный путь |
| Архив пустой / не тот путь | Проверить `-C`, trailing `/`, `--exclude` |
| «Unexpected EOF» | обрезанный файл, не докачанный transfer |
| Распаковали поверх `/` | restore в `/tmp`, diff, затем копирование |
| Двойное сжатие `.gz.gz` | понять pipeline; один раз gzip |

```bash
file archive.tar.gz                           # тип файла (magic bytes)
ls -lh archive.tar.gz                         # подозрительно маленький размер?
```

---

## 11. Мини‑шпаргалка

```bash
tar -czf backup.tar.gz -C / etc               # архив /etc
tar -xzf backup.tar.gz -C /tmp/restore        # распаковать безопасно
tar -tzf backup.tar.gz | head                 # содержимое
gzip -k large.log                             # сжать, оставить оригинал
zstd -19 -k dump.sql                          # zstd max level
zip -r share.zip project/                       # для Windows
sha256sum backup.tar.gz > backup.tar.gz.sha256 # checksum
```

Передача архива: [ssh_cheatsheet.md](ssh_cheatsheet.md) (`scp`, `rsync`). Редактор: [vim_cheatsheet.md](vim_cheatsheet.md).
