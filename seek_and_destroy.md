# Seek and destroy: «прожорливый» процесс

← [README](README.md) · типовой сценарий нагрузки: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md)

## Кейс: `io.elementary.appcenter`

**Проблема:** процесс занимал ~12,8% памяти (~1 ГБ) и замораживал систему.

**Решение:** удаление пакета `pop-shop` и завершение процесса.

`pop-shop` — некритичный магазин приложений; meta‑пакет `pop-desktop` можно снять без вреда системе.

---

## Шаг 1: поиск процессов

| Команда | Описание |
|--------|----------|
| `ps aux --sort=-%mem` + `head -15` | По памяти |
| `ps aux --sort=-%cpu` + `head -15` | По CPU |
| `top -b -n 1` + `head -20` | Снимок top |
| `htop` | Интерактивно (если установлен) |

## Шаг 2: источник процесса

| Команда | Описание |
|--------|----------|
| `readlink -f /proc/PID/exe` | Путь к бинарнику (PID из `ps`) |
| `tr '\0' ' ' </proc/PID/cmdline` | Командная строка |
| `which команда` | Если процесс — имя из PATH |
| `dpkg -S /путь/к/файлу` | Пакет Debian/Ubuntu |
| `apt show пакет` | Метаданные пакета |

## Шаг 3: зависимости

| Команда | Описание |
|--------|----------|
| `apt-cache depends пакет` | Прямые зависимости |
| `apt-cache rdepends пакет` | Обратные зависимости |
| `apt-cache rdepends --no-recommends пакет` | Без recommends |

## Шаг 4: удаление пакета

| Команда | Описание |
|--------|----------|
| `sudo apt remove пакет` | Без конфигов |
| `sudo apt purge пакет` | С конфигами |
| `sudo apt autoremove` | Лишние зависимости |
| `sudo apt clean` | Кэш apt |

## Шаг 5: завершение процесса

| Команда | Описание |
|--------|----------|
| `kill PID` | SIGTERM |
| `kill -9 PID` | SIGKILL (крайний случай) |
| `sudo kill PID` | От root |
| `killall имя` | Все процессы по имени |

## Шаг 6: автозапуск

| Команда | Описание |
|--------|----------|
| `systemctl --user list-unit-files`, затем `grep enabled` | User units |
| `systemctl list-unit-files`, затем `grep enabled` | System units |
| `systemctl --user status служба` | Статус (user) |
| `systemctl status служба` | Статус (system) |
| `systemctl --user disable служба` | Отключить (user) |
| `sudo systemctl disable служба` | Отключить (system) |

---

## Бонус: однострочники

Процессы с памятью > 5%:

```bash
ps aux | awk '$4 > 5 {print $2, $4"%", $11}' | sort -k2 -rn
```

Быстрая диагностика по PID (подставьте PID из `ps`):

```bash
PID=1234
EXE=$(readlink -f /proc/$PID/exe)
echo "=== exe ===" && echo "$EXE"
echo "=== cmdline ===" && tr '\0' ' ' </proc/$PID/cmdline && echo
echo "=== package ===" && dpkg -S "$EXE"
echo "=== rdepends ===" && apt-cache rdepends "$(dpkg -S "$EXE" | cut -d: -f1)"
```

---

## Алгоритм

1. `ps aux --sort=-%mem | head -15` — найти тяжёлый процесс.
2. `readlink -f /proc/PID/exe` → `dpkg -S путь` — источник.
3. `apt-cache rdepends пакет` — кто зависит.
4. Оценить, нужен ли пакет системе.
5. `sudo apt remove пакет`.
6. `kill PID`, если процесс ещё жив.
7. `sudo apt autoremove && sudo apt clean`.
8. `systemctl --user list-unit-files | grep пакет` — автозапуск.

Snap/Flatpak не определяются через `dpkg -S` — смотрите `snap list`, `flatpak list`.
