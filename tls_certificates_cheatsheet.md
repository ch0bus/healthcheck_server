# TLS, сертификаты, certbot

← [README](README.md) · nginx: [nginx_cheatsheet.md](nginx_cheatsheet.md) · firewall: [firewall_cheatsheet.md](firewall_cheatsheet.md) · сеть: [network_diagnostics_cheatsheet.md](network_diagnostics_cheatsheet.md)

Симптомы: **сертификат просрочен**, **NET::ERR_CERT_***, HTTPS не открывается, **certbot renew** падает.

---

## 1. Быстрая проверка с клиента

```bash
curl -vI https://example.com 2>&1 | grep -E 'subject:|issuer:|expire|SSL'
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null | openssl x509 -noout -dates -subject
```

---

## 2. Certbot + nginx (Let's Encrypt)

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
sudo certbot certificates
sudo certbot renew --dry-run
systemctl list-timers | grep certbot
```

Логи: `/var/log/letsencrypt/letsencrypt.log`

---

## 3. Типовые ошибки renew

| Причина | Что проверить |
|---------|----------------|
| Порт 80 закрыт с интернета | ufw + SG облака; HTTP-01 challenge |
| nginx misconfig | `nginx -t` |
| DNS не на этот сервер | `dig +short example.com` |
| Rate limit LE | частые delete/recreate |

---

## 4. Ручной сертификат (self-signed — только тест)

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/test.key -out /etc/ssl/certs/test.crt \
  -subj "/CN=localhost"
```

Браузер будет ругаться — для prod используйте LE или коммерческий CA.

---

## 5. Проверка цепочки и SNI

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null
openssl x509 -in /etc/letsencrypt/live/example.com/fullchain.pem -noout -dates
```

---

## 6. HSTS / редирект HTTP→HTTPS

В nginx (фрагмент):

```nginx
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}
```

После правок: `nginx -t && systemctl reload nginx` — [nginx_cheatsheet.md](nginx_cheatsheet.md).

---

## 7. Шпаргалка

```text
openssl s_client / x509 -dates
certbot --nginx    renew --dry-run
80/443 открыты    DNS → правильный IP
```
