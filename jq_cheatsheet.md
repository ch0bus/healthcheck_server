# jq: JSON в командной строке

← [README](README.md) · regex: [regex_cheatsheet.md](regex_cheatsheet.md) · логи: [logs_cheatsheet.md](logs_cheatsheet.md) · docker: [docker_vps_cheatsheet.md](docker_vps_cheatsheet.md)

**jq** разбирает **JSON** — удобнее, чем regex для `journalctl -o json`, API, docker logs.

```bash
sudo apt install -y jq
jq --version
```

---

## 1. Основы

```bash
echo '{"name":"app","port":8080}' | jq .
echo '{"a":1}' | jq '.a'
echo '{"items":[1,2,3]}' | jq '.items[]'
echo '{"users":[{"n":"x"},{"n":"y"}]}' | jq '.users[].n'
```

---

## 2. Фильтры и select

```bash
curl -s https://api.example.com/status | jq '.status'
journalctl -u nginx -n 5 -o json | jq -s '.[].MESSAGE' -r
docker logs CONTAINER 2>&1 | jq -R 'fromjson? | select(. != null) | .log' -r 2>/dev/null | tail
```

```bash
# только строки где priority >= 4 (пример структуры journal)
journalctl -b -p warning -o json | jq -s '.[] | select(._SYSTEMD_UNIT) | {unit: ._SYSTEMD_UNIT, msg: .MESSAGE}' -r
```

---

## 3. Массивы и map

```bash
echo '[{"id":1},{"id":2}]' | jq 'map(.id)'
echo '[1,2,3,4]' | jq '[.[] | select(. > 2)]'
```

---

## 4. Файлы и потоки

```bash
jq '.servers[].host' config.json
jq -r '.version' package.json
cat big.json | jq -c '.events[] | {type, ts}' | head
```

**`-r`** — raw string без кавычек. **`-c`** — compact одна строка на объект.

---

## 5. Когда jq, когда grep/awk

| Данные | Инструмент |
|--------|------------|
| JSON | **jq** |
| nginx text log | awk/grep — [awk_sed_cheatsheet.md](awk_sed_cheatsheet.md) |
| Смешанный текст | grep; не «regex на весь JSON» — [regex_cheatsheet.md](regex_cheatsheet.md) §14 |

---

## 6. Мини‑скрипт

```bash
#!/usr/bin/env bash
# ~/bin/jq-field.sh KEY < file.json
set -euo pipefail
key="${1:?key}"
jq -r --arg k "$key" '.[$k] // empty'
```

**Запуск:** `~/bin/jq-field.sh port < app.json`

---

## 7. Шпаргалка

```text
jq .    jq '.field'    jq -r    map/select
journalctl -o json | jq    docker logs | jq
```
