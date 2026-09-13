# Готовые скрипты: рутина, файлы, изображения

← [README](README.md) · основы bash: [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md) · архивы: [archives_compression_cheatsheet.md](archives_compression_cheatsheet.md)

Скрипты для **домашнего ПК** и «грязной» папки `Downloads/`, `Pictures/`. На production‑сервере большинство не нужны; на VPS — только осознанно.

**Перед первым запуском:** сделайте копию каталога или `DRY_RUN=1`; читайте блок **«Что делает»**; под каждым скриптом — **«Запуск»** с примерами команд и комментариями; путь передавайте аргументом, не хардкодьте `/`.

```bash
mkdir -p ~/bin                             # личные скрипты
export PATH="$HOME/bin:$PATH"              # добавить в ~/.bashrc
chmod +x ~/bin/*.sh
```

---

## 0. Зависимости (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y imagemagick webp jpegoptim exiftool fdupes ffmpeg   # по нужде; fdupes/ffmpeg опционально
sudo apt install -y libheif-examples poppler-utils img2pdf trash-cli inotify-tools   # для §15
convert -version                           # ImageMagick
```

| Пакет | Зачем |
|-------|--------|
| **imagemagick** | `convert`, `mogrify` — resize, формат |
| **webp** | `cwebp`, `dwebp` |
| **jpegoptim** | сжатие JPEG без сильной потери |
| **libimage-exiftool-perl** | `exiftool` — метаданные EXIF |
| **fdupes** | поиск дубликатов |
| **ffmpeg** | превью видео, конверт |
| **libheif-examples** | `heif-convert` — HEIC с iPhone |
| **poppler-utils** | `pdfseparate` — страницы PDF |
| **img2pdf** | PDF из изображений (альтернатива ImageMagick) |
| **trash-cli** | `trash-put` — «корзина» в CLI |
| **inotify-tools** | `inotifywait` — watch папки |

---

## 1. Сортировка по расширению

**Что делает:** в каталоге `$1` создаёт подпапки `pdf/`, `jpg/`, `zip/` … и **перемещает** файлы (не трогает подкаталоги на первом уровне без `-mindepth`).

```bash
#!/bin/bash
# ~/bin/sort-by-ext.sh
set -euo pipefail

DIR="${1:-.}"
DRY_RUN="${DRY_RUN:-0}"
cd "$DIR"

shopt -s nullglob
for f in *; do
  [[ -f "$f" ]] || continue
  ext="${f##*.}"
  [[ "$f" == *.* && "$ext" != "$f" ]] || ext="no_extension"
  ext_lower=$(printf '%s' "$ext" | tr '[:upper:]' '[:lower:]')
  mkdir -p "$ext_lower"
  if [[ "$DRY_RUN" == 1 ]]; then
    echo "mv -n -- '$f' '$ext_lower/'"
  else
    mv -n -- "$f" "$ext_lower/"    # -n не перезаписывать
  fi
done
```

### Запуск

```bash
# Пробный прогон: только печать команд mv, файлы не трогает
DRY_RUN=1 ~/bin/sort-by-ext.sh ~/Downloads

# Разобрать папку загрузок
~/bin/sort-by-ext.sh ~/Downloads

# Текущий каталог (если забыли указать путь — в скрипте DIR="${1:-.}")
cd ~/Desktop/misc && ~/bin/sort-by-ext.sh

# Через bash без chmod +x
bash ~/bin/sort-by-ext.sh /mnt/usb/inbox
```

---

## 2. Сортировка по дате изменения (год‑месяц)

**Что делает:** раскладывает файлы в `2025-03/`, `2025-09/` по **mtime** (время последнего изменения; «дата создания» в Linux часто = mtime или birth через `stat`).

```bash
#!/bin/bash
# ~/bin/sort-by-month.sh
set -euo pipefail

SRC="${1:?usage: sort-by-month.sh DIR}"
DRY_RUN="${DRY_RUN:-0}"

find "$SRC" -maxdepth 1 -type f -print0 | while IFS= read -r -d '' f; do
  ym=$(date -r "$f" +%Y-%m)                  # дата mtime файла
  dest="$SRC/$ym"
  mkdir -p "$dest"
  base=$(basename "$f")
  if [[ "$DRY_RUN" == 1 ]]; then
    echo "mv -n -- '$f' '$dest/$base'"
  else
    mv -n -- "$f" "$dest/$base"
  fi
done
```

**По дате из EXIF (фото):** если `exiftool` видит `DateTimeOriginal`, можно заменить `date -r` на:

```bash
ym=$(exiftool -s3 -DateTimeOriginal "$f" 2>/dev/null | cut -d: -f1-2 | tr ':' '-')
[[ -z "$ym" ]] && ym=$(date -r "$f" +%Y-%m)
```

### Запуск

```bash
# Сначала dry-run — увидеть, в какие папки YYYY-MM уедут файлы
DRY_RUN=1 ~/bin/sort-by-month.sh ~/Pictures/inbox

# Разложить фото из «входящих»
~/bin/sort-by-month.sh ~/Pictures/inbox

# Скриншоты с рабочего стола (только файлы на верхнем уровне каталога)
~/bin/sort-by-month.sh ~/Pictures/Screenshots

# После правки скрипта под EXIF (см. выше) — те же команды; дата возьмётся из камеры
~/bin/sort-by-month.sh ~/Pictures/camera-import
```

---

## 3. Inbox «разобрать всё» (расширение + год‑месяц)

**Что делает:** `~/Inbox/file.jpg` → `~/Sorted/2025-09/jpg/file.jpg`.

```bash
#!/bin/bash
# ~/bin/inbox-triage.sh
set -euo pipefail

IN="${1:?inbox dir}"
OUT="${2:?output base dir}"
DRY_RUN="${DRY_RUN:-0}"

find "$IN" -maxdepth 1 -type f -print0 | while IFS= read -r -d '' f; do
  ym=$(date -r "$f" +%Y-%m)
  name=$(basename "$f")
  ext="${name##*.}"
  [[ "$name" == *.* && "$ext" != "$name" ]] || ext="none"
  ext=$(printf '%s' "$ext" | tr '[:upper:]' '[:lower:]')
  dest="$OUT/$ym/$ext"
  mkdir -p "$dest"
  if [[ "$DRY_RUN" == 1 ]]; then
    echo "mv -n '$f' '$dest/$name'"
  else
    mv -n -- "$f" "$dest/$name"
  fi
done
```

### Запуск

```bash
# План без перемещения
DRY_RUN=1 ~/bin/inbox-triage.sh ~/Inbox ~/Archive/Sorted

# Inbox в домашнем каталоге → общее хранилище
~/bin/inbox-triage.sh ~/Inbox ~/Archive/Sorted

# Явные пути: скачанное → структурированный архив
~/bin/inbox-triage.sh ~/Downloads/to-sort ~/data/files

# Результат: ~/Archive/Sorted/2025-09/pdf/док.pdf и т.д.
ls -R ~/Archive/Sorted | head -40
```

---

## 4. Пакетное переименование (дата + порядковый номер)

**Что делает:** `IMG_1234.jpg` → `2025-09-13_001.jpg` в одном каталоге (только файлы, не рекурсия).

```bash
#!/bin/bash
# ~/bin/prefix-date-rename.sh
set -euo pipefail

DIR="${1:?dir}"
PREFIX="${2:-$(date +%F)}"
DRY_RUN="${DRY_RUN:-0}"
cd "$DIR"

n=1
shopt -s nullglob
for f in *; do
  [[ -f "$f" ]] || continue
  ext="${f##*.}"
  if [[ "$f" == *.* && "$ext" != "$f" ]]; then
    new=$(printf '%s_%03d.%s' "$PREFIX" "$n" "$ext")
  else
    new=$(printf '%s_%03d' "$PREFIX" "$n")
  fi
  ((n++))
  [[ "$f" == "$new" ]] && continue
  if [[ "$DRY_RUN" == 1 ]]; then
    echo "mv -n -- '$f' '$new'"
  else
    mv -n -- "$f" "$new"
  fi
done
```

### Запуск

```bash
# Просмотр переименований (сегодняшняя дата в префиксе по умолчанию)
DRY_RUN=1 ~/bin/prefix-date-rename.sh ~/Pictures/to_rename

# Переименовать всё в каталоге: 2025-09-13_001.jpg, _002.jpg, …
~/bin/prefix-date-rename.sh ~/Pictures/to_rename

# Свой префикс (второй аргумент): vacation_001.jpg
~/bin/prefix-date-rename.sh ~/Pictures/to_rename vacation

# Сортировка имён зависит от порядка glob * — при необходимости сначала sort-by-month
```

---

## 5. Поиск дубликатов

**Что делает (простой):** `fdupes` находит одинаковые файлы по содержимому.

```bash
fdupes -r ~/Pictures                       # рекурсивно
fdupes -rdN ~/Downloads                    # -d удалить дубликаты, оставить первый (-N без вопросов — ОПАСНО)
```

**Что делает (скрипт по MD5, только вывод):** список групп с одинаковым хешем — удаление вручную.

```bash
#!/bin/bash
# ~/bin/find-dupes-md5.sh
set -euo pipefail

DIR="${1:?dir}"
find "$DIR" -type f -print0 | xargs -0 md5sum | sort | awk '
  { hash=$1; $1=""; sub(/^ /,""); path=$0
    if (hash in seen) { if (!seen[hash]) { print "---"; print prev[hash]; seen[hash]=1 } print path }
    else prev[hash]=path
  }'
```

### Запуск (fdupes)

```bash
# Только показать группы одинаковых файлов (рекурсивно по Pictures)
fdupes -r ~/Pictures

# Сравнить два каталога между собой
fdupes -r ~/Pictures/2024 ~/Pictures/backup-2024

# Удалить дубликаты, оставить первый в каждой группе — ОПАСНО: сначала без -d
# fdupes -r ~/Downloads   # проверить список
# fdupes -rdN ~/Downloads # -N без вопросов; сделайте бэкап
```

### Запуск (find-dupes-md5.sh)

```bash
# Отчёт по MD5: блоки «---» — группы с одинаковым содержимым; rm вручную
~/bin/find-dupes-md5.sh ~/Downloads

# Только один уровень не поддерживается скриптом — ищет рекурсивно; для shallow используйте fdupes
~/bin/find-dupes-md5.sh ~/Pictures/2025-09 | less
```

---

## 6. Изображения: уменьшить длинную сторону

**Что делает:** все JPG/PNG в каталоге — max 1920 px по большей стороне, **перезапись на месте** (`mogrify`).

```bash
#!/bin/bash
# ~/bin/resize-images.sh
set -euo pipefail

DIR="${1:?dir}"
MAX="${2:-1920}"
shopt -s nullglob nocaseglob
shopt -s extglob

cd "$DIR"
for f in *.{jpg,jpeg,png,JPG,JPEG,PNG}; do
  [[ -f "$f" ]] || continue
  mogrify -resize "${MAX}x${MAX}>" "$f"   # только если больше MAX; > = уменьшать, не увеличивать
  echo "resized $f"
done
```

### Запуск

```bash
# Уменьшить длинную сторону до 1920 px (перезапись файлов на месте)
~/bin/resize-images.sh ~/Pictures/export

# Миниатюры для веба: max 1280 px (второй аргумент)
~/bin/resize-images.sh ~/Pictures/export 1280

# Перед mass mogrify — копия каталога или бэкап ([backups_cheatsheet.md](backups_cheatsheet.md))
cp -a ~/Pictures/export ~/Pictures/export.bak
~/bin/resize-images.sh ~/Pictures/export
```

---

## 7. Изображения: конвертация в WebP

**Что делает:** рядом с каждым `photo.jpg` создаёт `photo.webp` (оригинал не удаляет).

```bash
#!/bin/bash
# ~/bin/to-webp.sh
set -euo pipefail

DIR="${1:?dir}"
QUALITY="${2:-85}"
find "$DIR" -maxdepth 1 -type f \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' \) -print0 |
  while IFS= read -r -d '' f; do
    out="${f%.*}.webp"
    [[ -f "$out" ]] && continue
    cwebp -q "$QUALITY" "$f" -o "$out" && echo "$out"
  done
```

### Запуск

```bash
# WebP рядом с JPG/PNG, качество по умолчанию 85
~/bin/to-webp.sh ~/Pictures/export

# Выше сжатие (меньше файл, хуже качество)
~/bin/to-webp.sh ~/Pictures/export 75

# Только новые: уже существующие .webp пропускаются (повторный запуск безопасен)
~/bin/to-webp.sh ~/Pictures/export

# Один каталог для сайта: оригиналы остаются, заливаете на хостинг *.webp
ls ~/Pictures/export/*.webp | head
```

---

## 8. Изображения: сжать JPEG и убрать EXIF

**Что делает:** `jpegoptim` уменьшает размер файла; `exiftool` удаляет метаданные (геолокация и т.д.) перед публикацией.

```bash
#!/bin/bash
# ~/bin/jpeg-clean.sh
set -euo pipefail

DIR="${1:?dir}"
find "$DIR" -maxdepth 1 -type f \( -iname '*.jpg' -o -iname '*.jpeg' \) -print0 |
  while IFS= read -r -d '' f; do
    jpegoptim --strip-all --max=85 "$f"      # --strip-all = убрать метаданные в optim
    # или отдельно: exiftool -all= -overwrite_original "$f"
    echo "done $f"
  done
```

### Запуск

```bash
# Сжать JPEG в каталоге и убрать EXIF (геолокация и т.д.)
~/bin/jpeg-clean.sh ~/Pictures/for-public

# Сначала копия — скрипт перезаписывает файлы
cp -a ~/Pictures/for-public ~/Pictures/for-public-orig
~/bin/jpeg-clean.sh ~/Pictures/for-public

# Проверить, что GPS убран
exiftool -GPS:all ~/Pictures/for-public/sample.jpg
```

---

## 9. Изображения: водяной знак (простой)

**Что делает:** накладывает полупрозрачный текст в угол (нужен ImageMagick).

```bash
#!/bin/bash
# ~/bin/watermark.sh
set -euo pipefail

IN="${1:?image}"
OUT="${2:-${IN%.*}_wm.${IN##*.}}"
TEXT="${3:-© me}"

convert "$IN" -gravity SouthEast -fill 'rgba(255,255,255,0.6)' \
  -pointsize 24 -annotate +20+20 "$TEXT" "$OUT"
echo "$OUT"
```

### Запуск

```bash
# Новый файл photo_wm.jpg, текст по умолчанию «© me»
~/bin/watermark.sh ~/Pictures/photo.jpg

# Явный выходной файл и подпись
~/bin/watermark.sh ~/Pictures/photo.jpg ~/Pictures/photo_marked.jpg "My Name"

# Пакетно в цикле (оригиналы не трогаем)
for f in ~/Pictures/export/*.jpg; do ~/bin/watermark.sh "$f" "${f%.jpg}_wm.jpg"; done
```

---

## 10. Видео: кадр‑превью

**Что делает:** из `clip.mp4` — `clip.jpg` на отметке 5 секунд.

```bash
#!/bin/bash
# ~/bin/video-thumb.sh
set -euo pipefail

IN="${1:?video file}"
SEC="${2:-5}"
OUT="${3:-${IN%.*}.jpg}"

ffmpeg -y -ss "$SEC" -i "$IN" -frames:v 1 -q:v 2 "$OUT"
echo "$OUT"
```

### Запуск

```bash
# Кадр на 5-й секунде → clip.jpg рядом с clip.mp4
~/bin/video-thumb.sh ~/Videos/clip.mp4

# Кадр на 30 s, своё имя превью
~/bin/video-thumb.sh ~/Videos/clip.mp4 30 ~/Pictures/clip-cover.jpg

# Обложки для всех mp4 в каталоге
for v in ~/Videos/*.mp4; do ~/bin/video-thumb.sh "$v" 3 "${v%.mp4}.jpg"; done
```

---

## 11. Очистка старых файлов в Downloads

**Что делает:** удаляет **файлы** (не каталоги) старше N дней в `$1`.

```bash
#!/bin/bash
# ~/bin/clean-old.sh
set -euo pipefail

DIR="${1:?dir}"
DAYS="${2:-90}"
DRY_RUN="${DRY_RUN:-0}"

find "$DIR" -maxdepth 1 -type f -mtime +"$DAYS" -print0 |
  while IFS= read -r -d '' f; do
    if [[ "$DRY_RUN" == 1 ]]; then
      echo "rm -- '$f'"
    else
      rm -v -- "$f"
    fi
  done
```

### Запуск

```bash
# Список файлов старше 90 дней (по умолчанию), без удаления
DRY_RUN=1 ~/bin/clean-old.sh ~/Downloads

# Удалить файлы в Downloads старше 90 дней (не трогает подпапки)
~/bin/clean-old.sh ~/Downloads 90

# Агрессивнее: старше 30 дней
DRY_RUN=1 ~/bin/clean-old.sh ~/Downloads 30

# Вместо rm — trash-cli: trash-put $(find ... )  (см. §15)
```

---

## 12. Скопировать с флешки / DCIM по дате

**Что делает:** копирует новые файлы с `/media/user/CANON/DCIM` в `~/Pictures/import/YYYY-MM/` (не удаляет с флешки).

```bash
#!/bin/bash
# ~/bin/import-dcim.sh
set -euo pipefail

SRC="${1:?mount point DCIM or card root}"
DEST="${2:-$HOME/Pictures/import}"

find "$SRC" -type f \( -iname '*.jpg' -o -iname '*.cr2' -o -iname '*.mp4' \) -print0 |
  while IFS= read -r -d '' f; do
    ym=$(date -r "$f" +%Y-%m)
    mkdir -p "$DEST/$ym"
    cp -an -- "$f" "$DEST/$ym/"    # -n skip if exists, -a preserve
    echo "imported $f"
  done
```

### Запуск

```bash
# Путь к смонтированной флешке (подставьте user и метку тома)
ls /media/"$USER"/                                 # найти точку монтирования
~/bin/import-dcim.sh /media/"$USER"/CANON/DCIM

# Второй аргумент — куда складывать (по умолчанию ~/Pictures/import)
~/bin/import-dcim.sh /media/"$USER"/CANON/DCIM ~/Pictures/archive

# Повторный импорт: cp -n не перезаписывает уже скопированные имена
~/bin/import-dcim.sh /media/"$USER"/CANON/DCIM

# После импорта можно sort-by-month на ~/Pictures/import/2025-09
```

---

## 13. Отчёт: что занимает место в каталоге

**Что делает:** топ‑20 подкаталогов и топ‑20 файлов.

```bash
#!/bin/bash
# ~/bin/dir-report.sh
set -euo pipefail

DIR="${1:-.}"
echo "=== dirs ==="
du -xhd1 "$DIR" 2>/dev/null | sort -hr | head -20
echo "=== large files ==="
find "$DIR" -xdev -type f -printf '%s %p\n' 2>/dev/null | sort -rn | head -20 | awk '{print $1/1048576 " MB", $2}'
```

### Запуск

```bash
# Отчёт по текущему каталогу
~/bin/dir-report.sh

# Что раздувает домашний Downloads
~/bin/dir-report.sh ~/Downloads

# Перед clean-old или sort — куда деть место
~/bin/dir-report.sh ~/Videos | tee ~/dir-report.txt
```

---

## 14. Синхронизация «рабочей» папки на NAS / второй диск

**Что делает:** зеркало через `rsync` (см. [ssh_cheatsheet.md](ssh_cheatsheet.md) для remote).

```bash
#!/bin/bash
# ~/bin/sync-to-backup-disk.sh
set -euo pipefail

SRC="${1:?source dir}"
DST="${2:?backup dir}"
rsync -aAXHv --delete-delay --exclude '.cache' --exclude 'node_modules' "$SRC/" "$DST/"
```

`--delete-delay` — удаляет на приёмнике лишнее **после** успешной передачи; без `--delete` безопаснее для первого знакомства.

### Запуск

```bash
# Локальный диск или каталог на NAS (смонтирован в /mnt/backup)
~/bin/sync-to-backup-disk.sh ~/Documents /mnt/backup/Documents-mirror

# Первый раз — без --delete в скрипте безопаснее; в файле можно заменить на rsync -aAXHv без delete
# Проверка: diff -rq после sync (медленно) или размер du

# Удалённый сервер по SSH (если заменить тело на rsync -avz user@host:...)
# rsync -avz --delete-delay ~/Projects/ user@nas:/share/Projects/

# По cron еженочно — [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md) §12
# 0 2 * * * /home/user/bin/sync-to-backup-disk.sh /home/user/Documents /mnt/backup/Documents
```

---

## 15. Дополнительные скрипты (бывший «каталог идей»)

### 15.1 Скриншоты → по месяцам

**Что делает:** файлы с `Screenshot`, `scrot`, `Spectacle` в имени (или все файлы в каталоге) раскладывает в `DEST/YYYY-MM/`.

```bash
#!/bin/bash
# ~/bin/sort-screenshots.sh
set -euo pipefail
SRC="${1:?dir, e.g. ~/Pictures/Screenshots}"
DEST="${2:-$SRC/sorted}"
DRY_RUN="${DRY_RUN:-0}"
shopt -s nullglob nocaseglob
cd "$SRC"
for f in *[Ss]creenshot* *.png; do
  [[ -f "$f" ]] || continue
  ym=$(date -r "$f" +%Y-%m)
  mkdir -p "$DEST/$ym"
  if [[ "$DRY_RUN" == 1 ]]; then echo "mv -n '$f' '$DEST/$ym/'"; else mv -n -- "$f" "$DEST/$ym/"; fi
done
```

### Запуск

```bash
DRY_RUN=1 ~/bin/sort-screenshots.sh ~/Pictures/Screenshots
~/bin/sort-screenshots.sh ~/Pictures/Screenshots ~/Pictures/Screenshots/sorted
```

---

### 15.2 HEIC → JPEG (iPhone)

**Что делает:** конвертирует `*.HEIC` / `*.heic` в `.jpg` (оригинал не удаляет).

```bash
#!/bin/bash
# ~/bin/heic-to-jpg.sh
set -euo pipefail
DIR="${1:?dir}"
find "$DIR" -maxdepth 1 -type f \( -iname '*.heic' \) -print0 | while IFS= read -r -d '' f; do
  out="${f%.*}.jpg"
  [[ -f "$out" ]] && continue
  heif-convert "$f" "$out" && echo "$out"
done
```

### Запуск

```bash
~/bin/heic-to-jpg.sh ~/Pictures/inbox
~/bin/heic-to-jpg.sh ~/Downloads && ~/bin/sort-by-ext.sh ~/Downloads
```

---

### 15.3 Картинки → один PDF

**Что делает:** собирает JPG/PNG из каталога в один PDF (`img2pdf`; порядок — sorted by name).

```bash
#!/bin/bash
# ~/bin/images-to-pdf.sh
set -euo pipefail
DIR="${1:?dir with images}"
OUT="${2:-${DIR%/}.pdf}"
mapfile -t files < <(find "$DIR" -maxdepth 1 -type f \( -iname '*.jpg' -o -iname '*.jpeg' -o -iname '*.png' \) | sort)
((${#files[@]})) || { echo "no images" >&2; exit 1; }
img2pdf -o "$OUT" "${files[@]}"
echo "$OUT"
```

### Запуск

```bash
~/bin/images-to-pdf.sh ~/Documents/scan-pages
~/bin/images-to-pdf.sh ~/Documents/scan-pages ~/Documents/scan-2025-09-13.pdf
# альтернатива: convert ~/Documents/scan-pages/*.jpg ~/out.pdf
```

---

### 15.4 PDF → отдельные страницы

**Что делает:** `pdfseparate` — `doc.pdf` → `doc-1.pdf`, `doc-2.pdf`, … в указанный каталог.

```bash
#!/bin/bash
# ~/bin/pdf-split.sh
set -euo pipefail
PDF="${1:?file.pdf}"
OUTDIR="${2:-${PDF%.*}-pages}"
mkdir -p "$OUTDIR"
pdfseparate "$PDF" "$OUTDIR/page-%d.pdf"
echo "pages in $OUTDIR"
ls "$OUTDIR" | head
```

### Запуск

```bash
~/bin/pdf-split.sh ~/Documents/manual.pdf
~/bin/pdf-split.sh ~/Documents/manual.pdf /tmp/manual-pages
```

---

### 15.5 Бэкап каталогов с git‑репозиториями

**Что делает:** находит под `$1` каталоги с `.git` и пакует каждый в `backup/git-имя-ДАТА.tar.gz`.

```bash
#!/bin/bash
# ~/bin/backup-git-projects.sh
set -euo pipefail
ROOT="${1:-$HOME/Workspace}"
DEST="${2:-$HOME/Backups/git}"
mkdir -p "$DEST"
find "$ROOT" -maxdepth 3 -type d -name .git -print0 | while IFS= read -r -d '' g; do
  proj=$(dirname "$g")
  name=$(basename "$proj")
  tar -czf "$DEST/${name}-$(date +%F).tar.gz" \
    --exclude='.git/objects' --exclude='node_modules' \
    -C "$(dirname "$proj")" "$name"
  echo "$DEST/${name}-$(date +%F).tar.gz"
done
```

### Запуск

```bash
~/bin/backup-git-projects.sh ~/Workspace ~/Backups/git
# см. [archives_compression_cheatsheet.md](archives_compression_cheatsheet.md), [backups_cheatsheet.md](backups_cheatsheet.md)
```

---

### 15.6 Уведомление: диск заполнен

**Что делает:** если использование раздела ≥ порога (по умолчанию 90%), выводит сообщение и шлёт `notify-send` (GUI).

```bash
#!/bin/bash
# ~/bin/disk-space-notify.sh
set -euo pipefail
MOUNT="${1:-/}"
THRESH="${2:-90}"
used=$(df -P "$MOUNT" | awk 'NR==2 { gsub(/%/,"",$5); print $5 }')
msg="Disk $MOUNT at ${used}% (threshold ${THRESH}%)"
if (( used >= THRESH )); then
  echo "$msg" >&2
  command -v notify-send >/dev/null && notify-send "Disk space" "$msg" || true
  exit 1
fi
echo "OK $msg"
```

### Запуск

```bash
~/bin/disk-space-notify.sh /
~/bin/disk-space-notify.sh /home 85
# cron каждый час: 0 * * * * /home/user/bin/disk-space-notify.sh / || true
```

---

### 15.7 Музыка: сортировка по расширению

**Что делает:** как §1, но только `mp3`, `flac`, `ogg`, `m4a`, `wav` в каталоге `Music/inbox`.

```bash
#!/bin/bash
# ~/bin/sort-music.sh
set -euo pipefail
DIR="${1:-$HOME/Music/inbox}"
DRY_RUN="${DRY_RUN:-0}"
cd "$DIR"
shopt -s nullglob nocaseglob
for ext in mp3 flac ogg m4a wav; do
  mkdir -p "$ext"
  for f in *."$ext"; do
    [[ -f "$f" ]] || continue
    if [[ "$DRY_RUN" == 1 ]]; then echo "mv '$f' '$ext/'"; else mv -n -- "$f" "$ext/"; fi
  done
done
```

### Запуск

```bash
DRY_RUN=1 ~/bin/sort-music.sh ~/Music/inbox
~/bin/sort-music.sh ~/Music/inbox
# теги и альбомы — отдельно beets; здесь только разложить по формату
```

---

### 15.8 EXIF/IPTC: ключевое слово «альбом»

**Что делает:** добавляет keyword всем JPEG в каталоге (`exiftool`).

```bash
#!/bin/bash
# ~/bin/exif-keyword.sh
set -euo pipefail
DIR="${1:?dir}"
TAG="${2:?keyword e.g. vacation}"
exiftool -IPTC:Keywords+="$TAG" -ext jpg -ext jpeg -overwrite_original "$DIR"
echo "tagged $TAG in $DIR"
```

### Запуск

```bash
~/bin/exif-keyword.sh ~/Pictures/2025-vacation vacation
~/bin/exif-keyword.sh ~/Pictures/2025-vacation "Summer 2025"
exiftool -IPTC:Keywords ~/Pictures/2025-vacation/sample.jpg
```

---

### 15.9 Сортировка фото по экспозиции (тёмные / светлые)

**Что делает:** по `ExposureCompensation` или Brightness из EXIF кладёт файлы в `dark/`, `normal/`, `bright/` (упрощённо).

```bash
#!/bin/bash
# ~/bin/sort-by-exposure.sh
set -euo pipefail
DIR="${1:?dir}"
DRY_RUN="${DRY_RUN:-0}"
find "$DIR" -maxdepth 1 -type f \( -iname '*.jpg' -o -iname '*.jpeg' \) -print0 |
  while IFS= read -r -d '' f; do
    ev=$(exiftool -s3 -ExposureCompensation "$f" 2>/dev/null | head -1)
    ev=${ev%% *}
    ev=${ev:-0}
    bucket=normal
    awk -v e="$ev" 'BEGIN { if (e+0 < -0.3) exit 1; if (e+0 > 0.3) exit 2; exit 0 }'
    r=$?
    case $r in 1) bucket=dark ;; 2) bucket=bright ;; esac
    mkdir -p "$DIR/$bucket"
    base=$(basename "$f")
    if [[ "$DRY_RUN" == 1 ]]; then echo "mv '$f' '$DIR/$bucket/'"; else mv -n -- "$f" "$DIR/$bucket/$base"; fi
  done
```

### Запуск

```bash
DRY_RUN=1 ~/bin/sort-by-exposure.sh ~/Pictures/test-batch
~/bin/sort-by-exposure.sh ~/Pictures/test-batch
ls ~/Pictures/test-batch/dark ~/Pictures/test-batch/bright 2>/dev/null
```

---

### 15.10 Watch‑папка → inbox-triage

**Что делает:** ждёт новые файлы в каталоге и вызывает `inbox-triage.sh` (нужен `inotify-tools`).

```bash
#!/bin/bash
# ~/bin/watch-inbox.sh
set -euo pipefail
WATCH="${1:?watch dir}"
OUT="${2:?sorted base}"
TRIAGE="${3:-$HOME/bin/inbox-triage.sh}"
[[ -x "$TRIAGE" ]] || { echo "missing $TRIAGE" >&2; exit 1; }
echo "Watching $WATCH → $OUT (Ctrl+C stop)"
while inotifywait -e close_write -e moved_to "$WATCH"; do
  "$TRIAGE" "$WATCH" "$OUT"
done
```

### Запуск

```bash
~/bin/watch-inbox.sh ~/Inbox ~/Archive/Sorted
# в фоне: nohup ~/bin/watch-inbox.sh ~/Inbox ~/Archive/Sorted >> ~/watch-inbox.log 2>&1 &
# systemd user service — см. [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md) §12
```

---

### 15.11 Еженедельный разбор Inbox

**Что делает:** обёртка для cron — triage + опционально sort-screenshots.

```bash
#!/bin/bash
# ~/bin/weekly-inbox.sh
set -euo pipefail
export PATH="/usr/local/bin:/usr/bin:/bin:$HOME/bin"
"$HOME/bin/inbox-triage.sh" "$HOME/Inbox" "$HOME/Archive/Sorted"
[[ -x "$HOME/bin/sort-screenshots.sh" ]] && "$HOME/bin/sort-screenshots.sh" "$HOME/Pictures/Screenshots" || true
```

### Запуск

```bash
~/bin/weekly-inbox.sh
# crontab -e — воскресенье 10:00:
# 0 10 * * 0 /home/USER/bin/weekly-inbox.sh >> /home/USER/logs/weekly-inbox.log 2>&1
```

---

### 15.12 Бэкап dotfiles

**Что делает:** архив `.bashrc`, `.profile`, `.config/git`, `.ssh/config` (без ключей — только config).

```bash
#!/bin/bash
# ~/bin/backup-dotfiles.sh
set -euo pipefail
DEST="${1:-$HOME/Backups}"
mkdir -p "$DEST"
out="$DEST/dotfiles-$(date +%F).tar.gz"
tar -czf "$out" -C "$HOME" \
  --ignore-failed-read \
  .bashrc .profile .bash_aliases \
  .config/git .gitconfig \
  .ssh/config 2>/dev/null || true
[[ -f "$out" ]] && echo "$out" || { echo "nothing archived" >&2; exit 1; }
```

### Запуск

```bash
~/bin/backup-dotfiles.sh
~/bin/backup-dotfiles.sh ~/Backups
tar -tzf ~/Backups/dotfiles-$(date +%F).tar.gz | head
```

---

### 15.13 RAW + JPEG: одно имя из RAW

**Что делает:** для каждого `.CR2`/`.NEF` переименовывает парный `.jpg` в то же basename, что у RAW (если JPEG лежит рядом).

```bash
#!/bin/bash
# ~/bin/align-raw-jpeg-names.sh
set -euo pipefail
DIR="${1:?dir}"
DRY_RUN="${DRY_RUN:-0}"
shopt -s nullglob nocaseglob
cd "$DIR"
for raw in *.CR2 *.cr2 *.NEF *.nef; do
  [[ -f "$raw" ]] || continue
  base="${raw%.*}"
  for j in "$base".jpg "$base".JPEG "$base".jpeg; do
    [[ -f "$j" ]] || continue
    target="${base}.jpg"
    [[ "$j" == "$target" ]] && break
    if [[ "$DRY_RUN" == 1 ]]; then echo "mv '$j' '$target'"; else mv -n -- "$j" "$target"; fi
    break
  done
done
```

### Запуск

```bash
DRY_RUN=1 ~/bin/align-raw-jpeg-names.sh ~/Pictures/raw-import
~/bin/align-raw-jpeg-names.sh ~/Pictures/raw-import
# метаданные RAW→JPEG: exiftool -tagsFromFile foo.CR2 -ext jpg .
```

---

### 15.14 Удалить пустые каталоги

**Что делает:** снизу вверх удаляет пустые папки после sort/triage.

```bash
#!/bin/bash
# ~/bin/remove-empty-dirs.sh
set -euo pipefail
DIR="${1:?root dir}"
DRY_RUN="${DRY_RUN:-0}"
if [[ "$DRY_RUN" == 1 ]]; then
  find "$DIR" -depth -type d -empty -print
else
  find "$DIR" -depth -type d -empty -delete
  echo "removed empty under $DIR"
fi
```

### Запуск

```bash
DRY_RUN=1 ~/bin/remove-empty-dirs.sh ~/Downloads
~/bin/remove-empty-dirs.sh ~/Archive/Sorted
~/bin/sort-by-ext.sh ~/Downloads && ~/bin/remove-empty-dirs.sh ~/Downloads
```

---

### 15.15 «Корзина»: старые файлы в trash

**Что делает:** как §11, но `trash-put` вместо `rm` (восстановление через `trash-list` / файловый менеджер).

```bash
#!/bin/bash
# ~/bin/trash-old-files.sh
set -euo pipefail
DIR="${1:?dir}"
DAYS="${2:-90}"
DRY_RUN="${DRY_RUN:-0}"
command -v trash-put >/dev/null || { echo "install trash-cli" >&2; exit 1; }
find "$DIR" -maxdepth 1 -type f -mtime +"$DAYS" -print0 |
  while IFS= read -r -d '' f; do
    if [[ "$DRY_RUN" == 1 ]]; then echo "trash-put '$f'"; else trash-put -- "$f"; fi
  done
```

### Запуск

```bash
DRY_RUN=1 ~/bin/trash-old-files.sh ~/Downloads 90
~/bin/trash-old-files.sh ~/Downloads 90
trash-list | head
# восстановление: trash-restore (интерактивно)
```

---

## 16. Безопасность и привычки

1. **`DRY_RUN=1`** — для всего, что двигает/удаляет ([bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md) §10.7).
2. **`mv -n` / `cp -n`** — не затирать одноимённые файлы молча.
3. **Бэкап** перед массовым `mogrify` и `fdupes -d` ([backups_cheatsheet.md](backups_cheatsheet.md)).
4. **`shellcheck`** и `bash -n` перед установкой в cron.
5. Не запускать на **`/`** или **`$HOME`** целиком без `maxdepth` и теста.

---

## 17. Быстрая установка всех примеров

Скопируйте нужные блоки в отдельные файлы в `~/bin/`, не обязательно все сразу:

```bash
mkdir -p ~/bin
# вставить скрипт через vim/nano, затем:
chmod +x ~/bin/sort-by-ext.sh ~/bin/sort-by-month.sh   # и т.д.
DRY_RUN=1 ~/bin/sort-by-ext.sh ~/Downloads             # проверка
```

Общие паттерны (функции `log`, `flock`, `getopts`): [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md).
