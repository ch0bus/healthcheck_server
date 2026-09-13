# Быстрая проверка состояния сервера (1–2 минуты)

← [README](README.md) · подробнее: [полный чек‑лист](server_healthcheck_full.md), [seek and destroy](seek_and_destroy.md)

```bash
uptime              # нагрузка и время работы
free -h             # оперативная память и swap
df -hT              # место на дисках
systemctl --failed  # сервисы с ошибками

ps aux --sort=-%cpu | head    # процессы с высокой нагрузкой CPU
watch -n 1 'ps aux --sort=-%cpu | head -15'   # то же онлайн (Ctrl+C)

ps aux --sort=-%mem | head    # процессы, использующие память

ss -tulpen          # порты и сетевые соединения
journalctl -p err -b   # ошибки текущей загрузки
```
