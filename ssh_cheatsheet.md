# SSH: подключение, ключи, конфиги, передача файлов

← [README](README.md) · основы: [linux_install_and_basics.md](linux_install_and_basics.md) · shell: [bashrc_cheatsheet.md](bashrc_cheatsheet.md)

**SSH (Secure Shell)** — зашифрованный протокол для удалённого входа в shell, выполнения команд и передачи файлов. На VPS и серверах это основной способ администрирования.

- **Клиент** — команда `ssh`, `scp`, `sftp`, `rsync` на вашем ПК.
- **Сервер** — демон **`sshd`** (OpenSSH), слушает порт **22** по умолчанию.
- **Аутентификация:** пароль пользователя Linux или **ключевая пара** (рекомендуется для серверов).

В блоках `bash` комментарии после `#` поясняют строку; shell игнорирует хвост после `#`.

---

## 1. Установка OpenSSH (Debian/Ubuntu)

```bash
sudo apt update                              # обновить индексы пакетов
sudo apt install -y openssh-client             # клиент: ssh, scp, sftp (часто уже есть)
sudo apt install -y openssh-server             # сервер sshd (на VPS обычно предустановлен)

systemctl status ssh                         # Ubuntu: unit часто называется ssh
systemctl status sshd                        # на некоторых дистрибутивах — sshd
sudo systemctl enable --now ssh              # автозапуск сервера SSH
ss -tulpen | grep ':22'                      # слушает ли кто-то порт 22
```

Клиент нужен на **вашем** ноутбуке; **`openssh-server`** — на машине, **куда** вы заходите.

---

## 2. Первое подключение

```bash
ssh user@203.0.113.10                        # user — логин на сервере; IP или домен
ssh -p 2222 user@example.com                 # если sshd слушает порт 2222, не 22
ssh user@example.com 'uptime; free -h'       # одна удалённая команда без интерактива
```

При **первом** входе клиент покажет **fingerprint** ключа сервера и спросит `yes/no`. Это нормально — вы подтверждаете, что подключаетесь к нужной машине. Отпечаток сохраняется в **`~/.ssh/known_hosts`**.

```bash
ssh-keygen -F example.com                    # показать сохранённый ключ хоста для имени
ssh-keygen -R example.com                    # удалить старую запись (после переустановки VPS)
```

Если видите **«WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!»** — IP могли выдать другому серверу, или сервер переустановили. Не игнорируйте: сверьте с панелью провайдера, затем `ssh-keygen -R host` и подключитесь снова.

---

## 3. Ключи: генерация и установка на сервер

Ключевая пара: **приватный** ключ остаётся у вас (`~/.ssh/id_ed25519`), **публичный** кладётся на сервер (`~/.ssh/authorized_keys`).

```bash
ssh-keygen -t ed25519 -C "you@laptop"        # -t тип; -C метка (email/комментарий)
# сохранить в ~/.ssh/id_ed25519 (Enter) или своё имя файла
# passphrase — пароль на ключ (рекомендуется); можно оставить пустым на dev-only

ls -la ~/.ssh/                               # id_ed25519 (600), id_ed25519.pub (644)
chmod 700 ~/.ssh                             # каталог только для вас
chmod 600 ~/.ssh/id_ed25519                  # приватный ключ — строго 600
chmod 644 ~/.ssh/id_ed25519.pub              # публичный можно 644

ssh-copy-id -i ~/.ssh/id_ed25519.pub user@203.0.113.10   # скопировать ключ на сервер
# альтернатива без ssh-copy-id:
cat ~/.ssh/id_ed25519.pub | ssh user@host 'mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys'
```

На **сервере** для пользователя `user`:

```bash
# права (иначе sshd может отказать)
chmod 700 ~                                  # home не должен быть world-writable
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys             # только владелец читает/пишет
```

Проверка входа по ключу:

```bash
ssh -i ~/.ssh/id_ed25519 user@203.0.113.10   # явно указать ключ
ssh -o PreferredAuthentications=publickey user@host   # только ключ, без пароля (тест)
```

---

## 4. Клиентский конфиг `~/.ssh/config`

Сокращает длинные команды; права **`chmod 600 ~/.ssh/config`**.

```bash
vim ~/.ssh/config                            # создать или править
chmod 600 ~/.ssh/config                      # обязательная рекомендация OpenSSH
```

Пример содержимого:

```sshconfig
# ~/.ssh/config — комментарии начинаются с #

Host vps                                     # короткое имя: ssh vps
    HostName 203.0.113.10                    # реальный адрес
    User deploy                              # логин по умолчанию
    Port 22                                  # порт sshd
    IdentityFile ~/.ssh/id_ed25519           # какой ключ использовать
    IdentitiesOnly yes                       # не перебирать все ключи подряд

Host github.com                              # типичный кейс Git
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_github

Host jump * !jump                            # через bastion (ProxyJump)
    HostName %h
    User admin

Host internal-db
    HostName 10.0.0.5
    User dbadmin
    ProxyJump jump                           # сначала ssh на jump, затем на internal
```

Использование:

```bash
ssh vps                                      # как ssh deploy@203.0.113.10 с нужным ключом
ssh -G vps | grep -E '^(hostname|user|port|identityfile) '   # что реально применит клиент
```

Полезные **опции клиента** (в командной строке или в `config`):

| Опция | Комментарий |
|-------|-------------|
| `-v` / `-vv` / `-vvv` | Подробный лог (отладка) |
| `-o ConnectTimeout=10` | Обрыв, если нет ответа 10 с |
| `-o ServerAliveInterval=60` | Keepalive, чтобы NAT не рвал сессию |
| `-N` | Без удалённой shell (только туннель) |
| `-L` / `-R` | Локальный / удалённый port forward (см. §8) |

---

## 5. Серверный конфиг `/etc/ssh/sshd_config`

Правки — через root; после изменений **перезагрузка конфига** (сессии не обрывает текущих):

```bash
sudoedit /etc/ssh/sshd_config                # безопасная правка с sudo
sudo sshd -t                                 # проверка синтаксиса (test)
sudo systemctl reload ssh                    # применить конфиг (Ubuntu: unit ssh)
# или: sudo systemctl reload sshd
```

Частые директивы (раскомментировать и задать значение):

| Директива | Смысл | Пример для VPS |
|-----------|--------|----------------|
| `Port` | Порт прослушивания | `22` или `2222` (не забудьте firewall) |
| `PermitRootLogin` | Вход root | `prohibit-password` (только ключ) или `no` |
| `PasswordAuthentication` | Вход по паролю | `no` после настройки ключей |
| `PubkeyAuthentication` | Вход по ключу | `yes` |
| `AllowUsers` / `AllowGroups` | Кому разрешён SSH | `AllowUsers deploy backup` |
| `MaxAuthTries` | Попыток авторизации | `3` |
| `ClientAliveInterval` | Keepalive с сервера | `120` |

```bash
grep -E '^(Port|PermitRootLogin|PasswordAuthentication|PubkeyAuthentication)' /etc/ssh/sshd_config
sudo journalctl -u ssh -n 50 --no-pager     # ошибки sshd после reload
sudo tail -f /var/log/auth.log              # попытки входа (Debian/Ubuntu)
```

**Важно:** перед `PasswordAuthentication no` откройте **вторую** SSH‑сессию и убедитесь, что вход по ключу работает.

---

## 6. SSH‑agent (хранение passphrase)

Agent держит расшифрованный ключ в памяти, чтобы не вводить passphrase каждый раз.

```bash
eval "$(ssh-agent -s)"                       # запустить agent в текущей shell
ssh-add ~/.ssh/id_ed25519                    # добавить ключ (спросит passphrase)
ssh-add -l                                   # список ключей в agent
ssh-add -D                                   # удалить все ключи из agent
```

На desktop часто agent стартует автоматически (GNOME Keyring и т.п.). На сервере agent обычно **не** нужен — вы **подключаетесь к** серверу, а не с него наружу.

Проброс agent на удалённую машину (**AgentForwarding**) — риск; включайте только доверенным хостам в `config`:

```sshconfig
Host trusted-bastion
    ForwardAgent yes
```

---

## 7. Передача файлов

### `scp` — простое копирование

```bash
scp file.txt user@203.0.113.10:~/           # локальный → удалённый home
scp user@203.0.113.10:~/logs/app.log .       # удалённый → текущий каталог (.)
scp -r ./project user@host:/var/www/         # каталог рекурсивно (-r)
scp -P 2222 file user@host:                  # -P порт (у scp заглавная P)
scp -i ~/.ssh/id_ed25519 file user@host:     # явный ключ
```

### `sftp` — интерактивный FTP поверх SSH

```bash
sftp user@203.0.113.10                       # интерактивная сессия
# команды внутри sftp: ls, cd, get remote local, put local remote, mkdir, rm
sftp -P 2222 user@host
```

Пример пакетных команд:

```bash
sftp user@host <<EOF
get /var/log/nginx/access.log ./access.log
put ./index.html /var/www/html/
bye
EOF
```

### `rsync` — синхронизация (удобно для бэкапов и деплоя)

```bash
rsync -avz ./local/ user@host:/remote/path/  # -a archive, -v verbose, -z сжатие
rsync -avz --delete user@host:/remote/ ./backup/   # зеркало; --delete убирает лишнее локально
rsync -avz -e 'ssh -p 2222' file user@host:   # нестандартный порт через -e
rsync -avz --progress ./big.tar user@host:~/   # прогресс больших файлов
```

Trailing **`/`** важен: `local/` — содержимое каталога; `local` — каталог целиком.

---

## 8. Туннели (кратко)

**Локальный forward** — порт на вашем ПК ведёт на сервис **за** SSH‑сервером:

```bash
ssh -L 8080:127.0.0.1:80 user@203.0.113.10   # localhost:8080 → nginx на VPS:80
# открыть браузер http://127.0.0.1:8080
```

**Удалённый forward** — порт на **удалённой** машине ведёт к вашему localhost (админ‑панели, CI):

```bash
ssh -R 9000:127.0.0.1:3000 user@host         # на host:9000 → ваш локальный :3000
```

**ProxyJump** — цепочка через bastion (предпочтительнее старых `-ProxyCommand`):

```bash
ssh -J user@bastion:22 user@10.0.0.5         # сначала bastion, затем internal
```

---

## 9. Безопасность: минимальный чек‑лист VPS

1. Вход по **ключу**; `PasswordAuthentication no` после проверки.
2. **Root:** `PermitRootLogin no` (работать под user + `sudo`).
3. **Firewall:** разрешить только нужный порт SSH (`ufw allow OpenSSH` или `2222/tcp`).
4. **Fail2ban** (опционально) — бан IP после bruteforce.
5. Обновления: `sudo apt upgrade` для патчей OpenSSH.
6. Не коммитить **приватные ключи** в git; не слать `.pem` в мессенджеры.

```bash
sudo ufw status verbose                      # правила firewall
sudo apt install -y fail2ban                 # опционально
sudo fail2ban-client status sshd             # jail ssh (имя может отличаться)
```

---

## 10. Диагностика типичных ошибок

| Сообщение | Частая причина | Что проверить |
|-----------|----------------|---------------|
| `Connection refused` | sshd не запущен / другой порт / firewall | `ss -tulpen`, `systemctl status ssh`, ufw |
| `Connection timed out` | неверный IP, сеть, SG cloud | ping, маршрут, security group провайдера |
| `Permission denied (publickey)` | нет ключа в `authorized_keys`, права `~/.ssh` | `ssh-copy-id`, chmod 700/600 |
| `Permission denied (password)` | пароль неверный или пароли отключены | панель VPS, `PasswordAuthentication` |
| `Too many authentication failures` | клиент шлёт много ключей | `IdentitiesOnly yes` в config |
| Host key changed | новый сервер на старом IP | `ssh-keygen -R`, сверить fingerprint |

```bash
ssh -vvv user@host 2>&1 | tail -40             # подробный лог последних шагов
nc -zv 203.0.113.10 22                         # достижим ли TCP порт 22 (если есть nc)
curl -v telnet://203.0.113.10:22               # грубая проверка порта (альтернатива)
```

Логи на сервере: `journalctl -u ssh`, `/var/log/auth.log` — см. [logs_cheatsheet.md](logs_cheatsheet.md).

---

## 11. SSH и Git (напоминание)

```bash
git clone git@github.com:user/repo.git         # SSH URL; ключ должен быть в GitHub/GitLab
ssh -T git@github.com                          # тест: "Hi username! You've successfully authenticated..."
```

Отдельный ключ для Git — отдельный `IdentityFile` в `~/.ssh/config` для `Host github.com`.

---

## 12. Мини‑шпаргалка команд

```bash
ssh user@host                                # интерактивная shell
ssh vps                                      # через ~/.ssh/config Host vps
scp -r dir user@host:~/                      # копировать каталог на сервер
rsync -avz dir/ user@host:~/dir/             # синхронизировать
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@host   # установить ключ
sudo sshd -t && sudo systemctl reload ssh     # проверить и применить sshd_config
```

После настройки сервера: [server_healthcheck_quick.md](server_healthcheck_quick.md). Правка конфигов: [vim_cheatsheet.md](vim_cheatsheet.md).
