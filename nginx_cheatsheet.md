# Nginx: конфиг, проверка, типовые сбои

← [README](README.md) · логи: [logs_cheatsheet.md](logs_cheatsheet.md) · TLS: [tls_certificates_cheatsheet.md](tls_certificates_cheatsheet.md) · инциденты: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md) · regex: [regex_cheatsheet.md](regex_cheatsheet.md)

**Nginx** — reverse proxy и static; на VPS часто перед приложением (PHP-FPM, Node, Docker).

---

## 1. Установка и unit

```bash
sudo apt update && sudo apt install -y nginx
systemctl status nginx
systemctl enable nginx
ss -tulpen | grep -E ':80|:443'
```

Пути (Debian/Ubuntu):

| Путь | Назначение |
|------|------------|
| `/etc/nginx/nginx.conf` | Главный конфиг |
| `/etc/nginx/sites-available/` | Виртуальные хосты |
| `/etc/nginx/sites-enabled/` | Symlink на active |
| `/var/log/nginx/access.log`, `error.log` | Логи |

---

## 2. Безопасная правка

```bash
sudo nginx -t                              # синтаксис — всегда перед reload
sudo systemctl reload nginx                # мягче restart
sudo systemctl restart nginx               # если reload не помог
```

Deploy‑паттерн: [bash_scripts_cheatsheet.md](bash_scripts_cheatsheet.md) (nginx -t + reload).

Редактирование: `sudoedit /etc/nginx/sites-available/default` — [vim_cheatsheet.md](vim_cheatsheet.md).

---

## 3. Минимальный site (HTTP)

```nginx
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/example;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

```bash
sudo ln -sf /etc/nginx/sites-available/example /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 4. Reverse proxy на приложение

```nginx
location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

502/504 часто = **backend не слушает** `127.0.0.1:8080` — проверьте `ss -tlnp | grep 8080`.

---

## 5. PHP-FPM (кратко)

```nginx
location ~ \.php$ {
    include snippets/fastcgi-php.conf;
    fastcgi_pass unix:/run/php/php8.3-fpm.sock;   # версия PHP — ls /run/php/
}
```

```bash
systemctl status php8.3-fpm
```

---

## 6. HTTPS и certbot

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
sudo certbot renew --dry-run
systemctl list-timers | grep certbot
```

Подробнее: [tls_certificates_cheatsheet.md](tls_certificates_cheatsheet.md).

---

## 7. Диагностика 502 / 504 / 5xx

```bash
systemctl status nginx
journalctl -u nginx -n 50 --no-pager
sudo tail -50 /var/log/nginx/error.log
curl -I http://127.0.0.1/
curl -I -H 'Host: example.com' http://127.0.0.1/
```

| Код | Частая причина |
|-----|----------------|
| **502** | upstream мёртв / wrong socket |
| **504** | timeout до backend |
| **413** | `client_max_body_size` |
| **404** | root/alias, try_files |

---

## 8. Логи и awk/grep

```bash
tail -F /var/log/nginx/access.log
grep -E '" 5[0-9]{2} ' /var/log/nginx/access.log | tail
awk '$9 == 404 {print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head
```

[logs_cheatsheet.md](logs_cheatsheet.md), [awk_sed_cheatsheet.md](awk_sed_cheatsheet.md).

---

## 9. Firewall

```bash
sudo ufw allow 'Nginx Full'                # 80+443 профиль ufw
sudo ufw status
```

[firewall_cheatsheet.md](firewall_cheatsheet.md) — SG облака + ufw.

---

## 10. Шпаргалка

```text
nginx -t → reload    sites-enabled    journalctl -u nginx
502 → backend ss -tlnp    HTTPS → tls_certificates + certbot
```

БД за nginx: [databases_cheatsheet.md](databases_cheatsheet.md).
