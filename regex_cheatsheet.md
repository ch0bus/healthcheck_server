# Регулярные выражения (regex) для Linux

← [README](README.md) · grep/sed/awk: [awk_sed_cheatsheet.md](awk_sed_cheatsheet.md) · логи: [logs_cheatsheet.md](logs_cheatsheet.md) · vim: [vim_cheatsheet.md](vim_cheatsheet.md)

**Регулярное выражение (regex)** — это **шаблон строки**, а не точный текст. Программа смотрит: «подходит ли эта строка под картинку?» — и находит, фильтрует или заменяет.

Пример без магии:

- Текст: `error: disk full`
- Шаблон: `error` — **подходит** (подстрока есть).
- Шаблон: `^error` — **не подходит** (`error` не в **начале** строки; `^` = «начало»).
- Шаблон: `disk.*full` — **подходит** (`.` = любой символ, `*` = «сколько угодно раз»).

Regex не отдельная программа — его понимают **`grep`**, **`sed`**, **`awk`**, **`vim`**, **`find`**, bash **`[[ =~ ]]`**. Синтаксис чуть **различается** (ниже — про это).

---

## 1. Где это нужно на сервере

| Задача | Инструмент |
|--------|------------|
| Найти строки в логе | `grep`, `journalctl \| grep` |
| Вырезать/заменить в файле | `sed` |
| Разобрать колонки + условие | `awk` |
| Поиск в конфиге в vim | `/шаблон` |
| Проверить ввод в скрипте | bash `[[ $x =~ ... ]]` |

Подробнее по sed/awk: [awk_sed_cheatsheet.md](awk_sed_cheatsheet.md).

---

## 2. Два «диалекта»: BRE и ERE

На Linux в терминале чаще всего:

| Режим | Как включить | Запомнить |
|-------|--------------|-----------|
| **ERE** (extended) | `grep -E`, `egrep`, `sed -E` | `+`, `?`, `\|`, `()` **без** лишних `\` |
| **BRE** (basic) | `grep` без флагов, старый `sed` | `(`, `)`, `+` часто нужно писать как `\(` `\)` `\+` |

**Совет новичку:** везде, где можно, используйте **`-E`** — читается проще.

```bash
grep -E 'error|warn|fail' app.log          # ERE: OR через |
grep 'error\|warn' app.log                   # BRE: | экранирован
grep -E 'foo(bar|baz)' file.txt               # группа: foobar или foobaz
```

**`grep -P`** — PCRE (ещё богаче, `\b` границы слов). Есть в **GNU grep** на Linux; в минимальных образах может не быть. Ниже, где нужно `-P`, указано явно.

---

## 3. Символы: буквально vs «спецсмысл»

Многие символы в regex — **не** «как на клавиатуре», а **команды**. Чтобы искать **точку** или **вопрос**, их **экранируют** обратным слэшем `\`.

| Пишете | Значение |
|--------|----------|
| `a`, `9`, `-` (в середине класса) | Обычный символ |
| `.` | **Любой** один символ (кроме перевода строки в большинстве режимов) |
| `*` | Предыдущий фрагмент **0 или больше** раз |
| `+` | **1 или больше** (в **ERE**, т.е. `grep -E`) |
| `?` | **0 или 1** раз (ERE) |
| `^` | **Начало** строки |
| `$` | **Конец** строки |
| `\|` | **Или** (ERE): `cat\|dog` |
| `( … )` | **Группа** (ERE) |
| `[ … ]` | **Один** символ из набора |
| `[^ … ]` | **Один** символ **не** из набора |
| `\` | «Следующий символ — буквально»: `\.` → точка |

**Жадность:** `.*` «съедает» строку до конца, насколько может. Для логов часто хватает **`[^ ]*`** (до пробела) или **`[0-9]+`**.

### Мини‑разбор

```text
Строка:   GET /api/users HTTP/1.1" 404

^GET          — да (начинается с GET)
404$          — нет (после 404 ещё ничего… если строка ровно до 404 — да)
 GET          — нет (^ не совпало — пробел в начале)
/[a-z]+/      — да (/api/ — буквы между слэшами)
/[a-z]+       — подстрока /api
```

---

## 4. Классы символов (удобнее «вручную»)

Внутри **`[ ]`**:

| Шаблон | Смысл |
|--------|--------|
| `[0-9]` | Цифра |
| `[a-z]` | Строчная латиница |
| `[A-Za-z0-9_]` | «Слово» в стиле программирования |
| `[._-]` | Точка, подчёркивание, дефис (дефис в конце — буквально) |
| `[^0-9]` | Не цифра |

**POSIX‑классы** (работают в `[ ]`, читаются легче):

```bash
grep -E '[[:digit:]]+' file              # одна и более цифр
grep -E '[[:alpha:]]+' file              # буквы
grep -E '[[:space:]]' file               # пробельный символ
grep -E '[[:alnum:]_-]+' file            # «токен» для имён
```

---

## 5. Повторы: `{n,m}`

В **ERE** (`grep -E`, `sed -E`):

```bash
grep -E '[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}' access.log   # грубо IPv4
grep -E '([0-9]{4}-){2}[0-9]{2}' file    # дата вида 2026-09-13 (упрощённо)
grep -E '.{0,10}error.{0,10}' app.log    # error с контекстом до 10 символов с каждой стороны
```

| Запись | Смысл |
|--------|--------|
| `{3}` | Ровно 3 |
| `{2,5}` | От 2 до 5 |
| `{4,}` | 4 и больше |

---

## 6. grep — главный инструмент «найти строку»

```bash
grep 'error' /var/log/syslog               # строки с подстрокой error (регистр важен)
grep -i 'error' /var/log/syslog              # -i без учёта регистра
grep -v '127.0.0.1' access.log              # -v инверсия: строки БЕЗ шаблона
grep -c 'Failed password' /var/log/auth.log  # -c только число совпадений
grep -n 'listen' /etc/nginx/nginx.conf       # -n номера строк
grep -E 'error|warn|crit' app.log          # несколько слов через OR
grep -E '^#|^Port' /etc/ssh/sshd_config     # комментарий ИЛИ Port в начале… (^#|^Port)
grep -w 'root' /etc/passwd                 # -w целое «слово», не substring в другом слове
grep -r --include='*.conf' -E 'ssl_' /etc/nginx/   # рекурсивно только *.conf
```

**Не путать:** `grep pattern` ищет **regex**, не «glob как в ls». Звёздочка `*` в regex — **не** «любое имя файла».

---

## 7. sed — замена и удаление по шаблону

```bash
# убрать пробелы в конце строк
sed -E 's/[[:space:]]+$//' file.txt

# закомментировать строки, начинающиеся с Port (не #Port)
sed -E 's/^Port /#Port /' /etc/ssh/sshd_config

# удалить пустые строки
sed -E '/^$/d' file.txt

# заменить первое вхождение; g — все в строке
sed -E 's/foo/bar/g' file.txt

# только строки с ERROR напечатать
sed -n '/ERROR/p' app.log
```

**Группы и обратные ссылки** (ERE, GNU sed):

```bash
# поменять местами два слова через пробел (упрощённо)
echo 'hello world' | sed -E 's/([a-z]+) ([a-z]+)/\2 \1/'

# вытащить значение после KEY=
echo 'PORT=8080' | sed -E 's/^PORT=([0-9]+)/\1/'
```

Больше примеров: [awk_sed_cheatsheet.md](awk_sed_cheatsheet.md).

---

## 8. awk — когда regex + поля

В awk шаблон **`/regex/`** проверяет **всю строку** `$0`:

```bash
awk '/error|fail/ { print $0 }' app.log
awk '/^[0-9]{4}-/ { print $1, $3 }' data.tsv
awk -F'[= ]+' '$1 ~ /^PORT$/ { print $2 }' .env
```

Разделитель полей **`-F`** тоже может быть regex: `-F'[ ,]+'`.

---

## 9. Bash: проверка переменной

```bash
email='user@example.com'
if [[ $email =~ ^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$ ]]; then
  echo 'похоже на email (упрощённая проверка)'
fi

port='8080'
if [[ $port =~ ^[0-9]+$ ]] && (( port >= 1 && port <= 65535 )); then
  echo 'порт OK'
fi
```

**Справа от `=~`** — regex **без** кавычек (или в кавычках `"..."` — иначе bash режет `\`). Для сложных шаблонов иногда проще **`grep -E`** или **`case`**.

Скрипты: [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md).

---

## 10. Полезные шаблоны «с копипастой»

| Нужно | ERE (grep -E / sed -E) |
|-------|-------------------------|
| IPv4 (грубо) | `[0-9]{1,3}(\.[0-9]{1,3}){3}` |
| Строка только цифры | `^[0-9]+$` |
| Пустая строка | `^$` |
| Пробелы по краям (замена) | `^[[:space:]]+` / `[[:space:]]+$` |
| UUID | `[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}` |
| Домен (упрощённо) | `[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}` |
| HTTP код 4xx/5xx в nginx | `" [45][0-9]{2} ` |
| ssh Failed password | `Failed password` |
| Раскомментировать `#` в начале | `^#` → замена в sed |

**Граница слова** без `-P`: используйте **`grep -w`** для целых слов или **`[^a-zA-Z]root[^a-zA-Z]`** (грубо).

---

## 11. Практика: логи и админка

### Неудачные входы SSH

```bash
sudo grep -E 'Failed password|Invalid user' /var/log/auth.log | tail -20
sudo journalctl -u ssh --since today | grep -Ei 'failed|invalid'
```

### Nginx: 404 и медленные запросы (формат зависит от log_format)

```bash
grep -E '" 404 ' /var/log/nginx/access.log | tail -20
grep -E '" 5[0-9]{2} ' /var/log/nginx/access.log | wc -l
```

### Топ IP (часто 1‑е поле или awk; regex для «похоже на IP»)

```bash
grep -oE '[0-9]{1,3}(\.[0-9]{1,3}){3}' /var/log/nginx/access.log | sort | uniq -c | sort -rn | head
```

`-oE` — только совпавший кусок, extended regex.

### Найти строки без даты в начале (битые строки)

```bash
grep -Ev '^[0-9]{4}-[0-9]{2}-[0-9]{2}' app.log | head
```

Подробнее: [logs_cheatsheet.md](logs_cheatsheet.md).

---

## 12. Мини‑скрипты

### 12.1 Подсветка «опасных» строк в конфиге

```bash
#!/usr/bin/env bash
# ~/bin/grep-danger.sh — пример: что проверить глазами в конфиге
set -euo pipefail
file="${1:?usage: grep-danger.sh FILE}"
patterns='password|secret|api[_-]?key|BEGIN (RSA|OPENSSH) PRIVATE'
grep -Eni "$patterns" "$file" || echo 'OK: явных совпадений нет (это не гарантия безопасности)'
```

**Запуск:**

```bash
chmod +x ~/bin/grep-danger.sh
~/bin/grep-danger.sh ~/.env
~/bin/grep-danger.sh /etc/nginx/nginx.conf
```

### 12.2 Извлечь email из текста

```bash
#!/usr/bin/env bash
# ~/bin/extract-emails.sh
set -euo pipefail
grep -ohE '[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}' "${@:--}" | sort -u   # без аргументов — stdin (или явно -)
```

**Запуск:**

```bash
chmod +x ~/bin/extract-emails.sh
~/bin/extract-emails.sh /var/log/mail.log
journalctl -u postfix --no-pager | ~/bin/extract-emails.sh
```

### 12.3 Нормализовать пробелы в строках

```bash
#!/usr/bin/env bash
# ~/bin/squeeze-space.sh — сжать повторяющиеся пробелы в каждой строке
set -euo pipefail
sed -E 's/[[:space:]]+/ /g; s/^[[:space:]]+//; s/[[:space:]]+$//' "${1:?file}"
```

**Запуск:**

```bash
chmod +x ~/bin/squeeze-space.sh
~/bin/squeeze-space.sh messy.txt > clean.txt
```

### 12.4 Фильтр «только мои приложения» в ps

```bash
#!/usr/bin/env bash
# ~/bin/ps-apps.sh — процессы по regex имени (не точное совпадение)
set -euo pipefail
pat="${1:?usage: ps-apps.sh 'nginx|postgres|redis'}"
ps aux | grep -E "$pat" | grep -v 'grep -E'
```

**Запуск:**

```bash
chmod +x ~/bin/ps-apps.sh
~/bin/ps-apps.sh 'nginx|php-fpm|mysql'
```

### 12.5 Проверка, что строка — «безопасное» имя файла

```bash
#!/usr/bin/env bash
# ~/bin/safe-filename.sh — отклонить path traversal и странные символы
set -euo pipefail
name="${1:?filename}"
if [[ $name =~ ^[A-Za-z0-9._-]+$ ]] && [[ $name != *..* ]]; then
  echo "OK: $name"
else
  echo "REJECT: $name" >&2
  exit 1
fi
```

**Запуск:**

```bash
chmod +x ~/bin/safe-filename.sh
~/bin/safe-filename.sh 'report-2026.pdf'
~/bin/safe-filename.sh '../etc/passwd' || true
```

---

## 13. Как читать regex вслух (тренировка)

| Шаблон | По‑русски |
|--------|-----------|
| `^#` | «Строка **начинается** с `#`» |
| `error$` | «Строка **заканчивается** на error» |
| `colou?r` | «color или colour» (ERE) |
| `file(s)?` | «file или files» |
| `a+b` | «Один или больше `a`, потом `b`» |
| `a.*b` | «`a`, потом что угодно, потом `b`» |
| `[Ff]ail` | «Fail или fail» |
| `(19\|20)[0-9]{2}` | «Год 1900–2099 грубо» |

Проверять шаблон удобно на [regex101.com](https://regex101.com) (выберите flavor **PCRE** или **ECMAScript** для близости к `grep -E`; для sed смотрите GNU).

---

## 14. Частые ошибки новичков

1. **Забыли `-E`** — скобки и `+` не работают как ожидаете; используйте `grep -E` / `sed -E`.
2. **Точка как точка** — в regex `.` = «любой символ»; для IP/имени файла ищите **буквальную** точку: `\.` или `[.]`.
3. **`*` в shell vs regex** — в **кавычках** `'...'` или `"..."` regex не раздувается shell; без кавычек `*` — glob.
4. **Слишком общий `.*`** — вытащили лишнее; сузьте: `[0-9]+`, `[^"]*` (до кавычки).
5. **Ждали «полное совпадение файла»** — `grep` ищет **подстроку в строке**; «вся строка только X» = **`^...$`**.
6. **Один regex на JSON** — для JSON используйте **`jq`**, не героический regex.

---

## 15. Связка с vim

В vim поиск **`/шаблон`** — свой regex (близок к ERE, есть `\`, `\>` граница слова):

```text
/error\|warn          поиск error или warn
/\<root\>             слово root целиком
:%s/old/new/g         замена (см. vim_cheatsheet.md)
```

---

## 16. Мини‑шпаргалка

```text
grep -E 'a|b'     OR          grep -i     без регистра
^ начало  $ конец   . любой символ   * 0+   + 1+   ? 0-1
[0-9] [a-z] [[:digit:]]+     sed -E 's/from/to/g'     sed -n '/pat/p'
[[ $x =~ ^[0-9]+$ ]]          grep -oE 'шаблон'   только совпавший кусок
```

Дальше: текущая обработка текста — [awk_sed_cheatsheet.md](awk_sed_cheatsheet.md); интерактивный поиск по файлам — [ranger_cheatsheet.md](ranger_cheatsheet.md) (`/`).
