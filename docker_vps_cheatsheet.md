# Docker на VPS

← [README](README.md) · firewall: [firewall_cheatsheet.md](firewall_cheatsheet.md) §6 · nginx: [nginx_cheatsheet.md](nginx_cheatsheet.md) · systemd: [systemd_cheatsheet.md](systemd_cheatsheet.md) · память: [memory_and_load_cheatsheet.md](memory_and_load_cheatsheet.md)

**Docker** упаковывает приложения; на VPS типичная схема: **nginx на хосте** → контейнер на `127.0.0.1`, не публиковать лишние порты наружу.

---

## 1. Установка (Ubuntu)

```bash
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo tee /etc/apt/keyrings/docker.asc > /dev/null
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo ${VERSION_CODENAME}) stable" | sudo tee /etc/apt/sources.list.d/docker.list
sudo apt update && sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
sudo usermod -aG docker "$USER"              # перelogin
docker run --rm hello-world
```

---

## 2. Ежедневные команды

```bash
docker ps -a
docker logs -f --tail 100 CONTAINER
docker exec -it CONTAINER bash
docker stats --no-stream
docker compose ps
docker compose logs -f
docker compose up -d
docker compose down
```

---

## 3. Compose (минимум)

```yaml
# docker-compose.yml
services:
  app:
    image: myapp:latest
    restart: unless-stopped
    ports:
      - "127.0.0.1:8080:8080"
    environment:
      - NODE_ENV=production
    volumes:
      - appdata:/data

volumes:
  appdata:
```

```bash
docker compose up -d
curl -I http://127.0.0.1:8080/
```

**`- "127.0.0.1:8080:8080"`** — не светить порт в интернет; снаружи только nginx :443.

---

## 4. Firewall и порты

`-p 80:80` **обходит ufw** для published ports — см. [firewall_cheatsheet.md](firewall_cheatsheet.md).

```bash
ss -tulpen | grep docker
sudo ufw status verbose
```

Рекомендация: bind **127.0.0.1**, reverse proxy на хосте.

---

## 5. Диск и логи

```bash
docker system df
docker system prune -f                       # осторожно: неиспользуемое
du -sh /var/lib/docker/
journalctl -u docker --since today
```

Логи контейнера → **json-file** растёт: `max-size` в `/etc/docker/daemon.json` (осторожно на prod).

---

## 6. OOM и лимиты

```bash
docker update --memory 512m --memory-swap 512m CONTAINER
docker inspect CONTAINER --format '{{.HostConfig.Memory}}'
```

[memory_and_load_cheatsheet.md](memory_and_load_cheatsheet.md).

---

## 7. JSON-логи и jq

```bash
docker logs CONTAINER 2>&1 | tail -5
docker logs CONTAINER 2>&1 | jq -R 'fromjson? | .message' 2>/dev/null | tail
```

[jq_cheatsheet.md](jq_cheatsheet.md).

---

## 8. Шпаргалка

```text
compose up -d    logs -f    127.0.0.1:port
ufw + Docker → firewall_cheatsheet
nginx reverse proxy → nginx_cheatsheet
```
