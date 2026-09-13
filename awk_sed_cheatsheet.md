# sed и awk: обработка текста в командной строке

← [README](README.md) · regex с нуля: [regex_cheatsheet.md](regex_cheatsheet.md) · логи: [logs_cheatsheet.md](logs_cheatsheet.md) · bash: [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md)

**sed** (stream editor) и **awk** — стандартные утилиты Unix для **потоковой** обработки текста: фильтрация, замена, разбор колонок. Есть на любом Linux/VPS без установки.

| | **sed** | **awk** |
|---|---------|---------|
| Сильная сторона | Замена, удаление, вставка **по шаблону строки** | Разбор **полей/колонок**, суммы, отчёты |
| Модель | «Каждая строка → правила sed» | «Каждая строка → поля $1 $2 … → действие» |
| Язык | Регулярные выражения + команды `s///` | Мини‑язык (C‑подобный) |
| Когда вместо grep | Нужна **правка** текста или удаление строк | Нужна **математика** или несколько полей |

**grep** — только найти строки. **sed/awk** — преобразовать или посчитать. Для тяжёлых JSON/XML чаще **jq** / **xmlstarlet**; для таблиц CSV — **csvkit** или Python.

В примерах комментарии `#` в bash-блоках; внутри `awk '...'` комментарий — `#` до конца строки программы awk.

---

## 1. sed — основы

### Синтаксис

```bash
sed 'команды' файл                        # вывод в stdout, файл не меняется
sed -i 'команды' файл                     # правка на месте (GNU sed; бэкап: sed -i.bak)
sed -n 'команды' файл                     # -n — молчать; печатать только явный p
sed -E 'команды' файл                     # -E extended regex (GNU; на macOS иногда -r)
```

### Замена `s/что/на_что/флаги`

```bash
sed 's/foo/bar/' file.txt                 # первое вхождение foo→bar в каждой строке
sed 's/foo/bar/g' file.txt                # g — все вхождения в строке
sed 's|/usr/local|/opt|g' file.txt        # другой разделитель | если в пути /
sed 's/^#//' file.txt                     # убрать # в начале строки
sed 's/[[:space:]]*$//' file.txt          # убрать хвостовые пробелы
sed -i.bak 's/old/new/g' /etc/app.conf    # бэкап app.conf.bak перед правкой
```

### Адреса строк

```bash
sed -n '10p' file.txt                     # только 10-я строка
sed -n '10,20p' file.txt                  # строки 10–20
sed -n '1,5d' file.txt                    # d — удалить (без -n: печатает остальное)
sed '/error/Id' log.txt                   # удалить строки с error (I — без регистра)
sed '/^$/d' file.txt                      # удалить пустые строки
sed '1d' file.txt                         # удалить первую строку
```

### Вставка и append

```bash
sed '/listen 80/a \\    server_name example.com;' nginx.snippet   # после совпадения
sed '1i # managed by script' file.conf      # i — insert перед 1-й строкой
```

### Практика sed

```bash
# закомментировать все строки с Port (не трогая #Port)
sed -i.bak -E 's/^Port /#Port /' /etc/ssh/sshd_config

# только строки с ERROR на stderr в пайпе
grep . app.log | sed -n '/ERROR/p'

# заменить табы на два пробела (осторожно в Makefile)
sed 's/\t/  /g' file.txt

# удалить UTF-8 BOM в первой строке
sed '1s/^\xEF\xBB\xBF//' file.txt
```

---

## 2. awk — основы

**awk** читает строки, режет на **поля** (по умолчанию пробелы/табы), даёт `$1`, `$2`, … `$NF` (последнее), `$0` — вся строка.

```bash
awk '{ print $1 }' file                   # первый столбец
awk '{ print $NF }' file                  # последний столбец
awk '{ print $(NF-1) }' file              # предпоследний
awk -F: '{ print $1 }' /etc/passwd        # -F разделитель полей — двоеточие
awk -F'[ ,]+' '{ print $2 }' file          # FS — regex: пробел или запятая
```

### Шаблон (pattern) и действие

```bash
awk '/error/ { print $0 }' log            # только строки, где есть error
awk '$3 > 100 { print $1, $3 }' data.txt  # сравнение поля (число)
awk 'NR==1 { next } { print }' file       # NR — номер строки; пропустить заголовок
awk 'NF > 0 { print }' file               # не пустые строки
```

### BEGIN / END

```bash
awk 'BEGIN { sum=0 } { sum+=$1 } END { print sum }' numbers.txt   # сумма 1-го столбца
awk 'BEGIN { FS=","; OFS="\t" } { print $1, $2 }' data.csv       # CSV → tab
```

### Переменные и printf

```bash
awk '{ cnt++ } END { print cnt }' file    # число строк
awk '{ printf "%-20s %8.2f\n", $1, $2 }' report.txt
```

---

## 3. sed + awk в пайпах

```bash
ps aux | awk '$4 > 5.0 {print $2, $4"%", $11}'     # память > 5% (как в seek_and_destroy)
df -hT | awk 'NR>1 {print $7, $6}'                  # mount point и Use%
journalctl -b --no-pager | sed -n '/Failed/p'       # только строки с Failed
ss -tulpen | awk 'NR>1 {print $5}' | sed 's/.*://'  # порты из Local Address
```

---

## 4. Логи и nginx

См. также [logs_cheatsheet.md](logs_cheatsheet.md).

```bash
# топ IP в access.log
awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head

# все 404 и URL
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head

# ошибки за сегодня — вырезать timestamp (пример формата зависит от log)
grep "$(date +%d/%b/%Y)" /var/log/syslog | awk 'tolower($0) ~ /error|fail/ {print}'

# заменить в логе IP на REDACTED (копию, не оригинал!)
sed 's/[0-9]\{1,3\}\(\.[0-9]\{1,3\}\)\{3\}/REDACTED/g' access.log.copy > access.anon
```

---

## 5. Конфиги и системное администрирование

```bash
# пользователи с shell не nologin
awk -F: '$7 !~ /nologin|false/ {print $1, $7}' /etc/passwd

# размеры из du -h (поле 1 — размер, $NF — путь)
du -h /var/log | awk 'NR<=10 {print}'

# проверка: диск > 90% (поле $5 в df -P)
df -P / | awk 'NR==2 { gsub(/%/,"",$5); if ($5+0 >= 90) exit 1 }'
echo $?   # 1 если переполнен

# вытащить значение из KEY=value
grep -E '^PORT=' .env | sed 's/^PORT=//' | tr -d '"'

awk -F= '/^PORT=/ {print $2; exit}' .env   # то же через awk
```

---

## 6. Практические однострочники

```bash
# уникальные значения 3-го столбца
awk '{print $3}' file | sort -u

# сумма 2-го столбца если 1-й столбец == "apple"
awk '$1=="apple" {s+=$2} END {print s}' sales.txt

# номера строк как grep -n
awk '{print NR": "$0}' file

# заменить в 2-м поле только строк где $1=="active"
awk '$1=="active" {$2="yes"; print}' OFS='\t' data.txt

# CSV: второе поле в кавычках
awk -F',' '{gsub(/^"|"$/, "", $2); print $2}' data.csv
```

---

## 7. Готовые мини‑скрипты

### 7.1 Подсветка диска в df

**Что делает:** печатает строки df, где использование ≥ порога (только текст, без цветов).

```bash
#!/bin/bash
# ~/bin/df-warn.awk — вызывается: df -P | awk -f ~/bin/df-warn.awk
# или inline:
df -P | awk 'NR==1 {print; next} { gsub(/%/,"",$5); if ($5+0 >= 90) print "WARN:", $0; else print }'
```

### Запуск

```bash
df -P | awk 'NR==1 {print; next} { gsub(/%/,"",$5); if ($5+0 >= 90) print "HIGH:", $0 }'
THRESH=80 df -P / | awk -v t="${THRESH:-90}" 'NR==2 { gsub(/%/,"",$5); if ($5+0 >= t) exit 1 }'
```

---

### 7.2 Нормализация `/etc/hosts` (убрать лишние пробелы)

**Что делает:** сжимает пробелы между полями, не трогая комментарии после `#`.

```bash
#!/bin/bash
# ~/bin/normalize-hosts.sh
set -euo pipefail
FILE="${1:?/etc/hosts}"
sed -i.bak -E \
  -e 's/[[:space:]]+/ /g' \
  -e 's/^ //' \
  "$FILE"
echo "normalized $FILE (backup ${FILE}.bak)"
```

### Запуск

```bash
sudo cp /etc/hosts /etc/hosts.bak.manual
sudo ~/bin/normalize-hosts.sh /etc/hosts
```

---

### 7.3 Отчёт: топ процессов по памяти (awk)

**Что делает:** из `ps aux` — PID, MEM%, COMMAND для строк с MEM > порога.

```bash
#!/bin/bash
# ~/bin/ps-mem-report.sh
THRESH="${1:-5.0}"
ps aux | awk -v t="$THRESH" '
  NR==1 { print; next }
  $4+0 > t { printf "%-8s %6s%% %s\n", $2, $4, $11 }
' | head -20
```

### Запуск

```bash
~/bin/ps-mem-report.sh          # > 5% RAM
~/bin/ps-mem-report.sh 10         # > 10%
```

---

### 7.4 Парсинг `ss` / `netstat`: список слушающих портов

```bash
#!/bin/bash
# ~/bin/listening-ports.sh
ss -tulpen | awk '
  NR==1 { next }
  {
    split($5, a, ":");
    port=a[length(a)];
    print port, $1
  }
' | sort -n | uniq
```

### Запуск

```bash
~/bin/listening-ports.sh
~/bin/listening-ports.sh | grep -E '^(22|80|443) '
```

---

### 7.5 Бulk replace в дереве каталогов (sed)

**Что делает:** в файлах `*.conf` под `$DIR` заменяет строку (осторожно: проверьте DRY_RUN).

```bash
#!/bin/bash
# ~/bin/sed-tree-replace.sh
set -euo pipefail
DIR="${1:?dir}"
FROM="${2:?from}"
TO="${3:?to}"
DRY_RUN="${DRY_RUN:-0}"
export FROM TO
find "$DIR" -type f -name '*.conf' -print0 | while IFS= read -r -d '' f; do
  if [[ "$DRY_RUN" == 1 ]]; then
    sed -n "s/${FROM}/${TO}/gp" "$f" | sed "s|^|${f}: |"
  else
    sed -i.bak "s/${FROM}/${TO}/g" "$f"
  fi
done
```

### Запуск

```bash
DRY_RUN=1 ~/bin/sed-tree-replace.sh ~/project/config OLD_URL NEW_URL
~/bin/sed-tree-replace.sh ~/project/config localhost 127.0.0.1
```

---

### 7.6 CSV → сводка (awk)

**Что делает:** сумма продаж по 2-му столбцу, сгруппировано по 1-му (категория).

```bash
#!/usr/bin/awk -f
# ~/bin/sum-by-category.awk
# usage: awk -f sum-by-category.awk sales.csv   (CSV: category,amount)
BEGIN { FS=","; OFS="\t" }
NR==1 && $1 ~ /cat/i { next }               # пропуск заголовка
{
  gsub(/^"|"$/, "", $1); gsub(/^"|"$/, "", $2)
  sum[$1] += $2
}
END { for (k in sum) print k, sum[k] }
```

### Запуск

```bash
awk -f ~/bin/sum-by-category.awk ~/data/sales.csv | sort -k2 -nr
```

---

## 8. sed vs awk — что выбрать

| Задача | Инструмент |
|--------|------------|
| Заменить текст в файле | `sed -i` |
| Удалить строки по regex | `sed '/pattern/d'` |
| Сумма/среднее по столбцу | `awk` |
| Разбор `/etc/passwd`, логов с полями | `awk -F` |
| Только показать совпадения | `grep` / `rg` |
| JSON API | `jq` |

---

## 9. Частые ошибки

| Ошибка | Решение |
|--------|---------|
| `sed -i` без бэкапа на production | `sed -i.bak` или копия файла |
| В `sed` спецсимволы `/` в URL | разделитель `\|` или `#` |
| awk сравнивает строки как числа | `$3+0` или `int()` |
| Пробелы в полях | `FS` или `$1` в кавычках в csv |
| macOS sed vs GNU | на Mac `sed -i ''` и `sed -E` |

```bash
# GNU sed явно (Debian)
sed --version | head -1
```

---

## 10. Мини‑шпаргалка

```bash
sed 's/a/b/g' file                          # replace all
sed -n '5,10p' file                         # print lines 5-10
awk '{print $1, $3}' file                   # columns
awk -F: '{print $1}' /etc/passwd            # delimiter :
awk '/pat/ {c++} END {print c}' file        # count matches
ps aux | awk 'NR>1 && $4>10 {print $11}'    # process names high mem
```

Пайпы в скриптах: [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md). Правка конфигов: [vim_cheatsheet.md](vim_cheatsheet.md).
