# LVM и расширение диска

← [README](README.md) · место на диске: [common_incidents_cheatsheet.md](common_incidents_cheatsheet.md) · установка: [linux_install_and_basics.md](linux_install_and_basics.md)

Когда **`df`** показывает **100%** на `/` или `/var`, но на VPS добавили **volume** — часто нужен **LVM** или resize раздела.

---

## 1. Что установлено

```bash
lsblk -f                                  # дерево дисков, FS, UUID
df -hT
sudo pvs && sudo vgs && sudo lvs          # LVM: physical / volume group / logical
sudo fdisk -l                             # разделы (осторожно на prod)
findmnt /                                   # откуда смонтирован /
```

---

## 2. Типовой стек Ubuntu server

```text
disk → partition → PV → VG (ubuntu-vg) → LV (ubuntu-lv) → ext4 → /
```

Имена **`ubuntu-vg`** / **`ubuntu-lv`** частые на автoinstall; у вас могут отличаться.

---

## 3. Расширить LV после увеличения диска в облаке

**Порядок:** расширить **раздел** (если нужно) → **PV** → **LV** → **файловую систему**.

```bash
# 1) облако: увеличили диск в панели, затем на VM:
sudo growpart /dev/sda 3                  # номер раздела с PV — проверьте lsblk
sudo pvresize /dev/sda3
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv   # ext4 онлайн
df -h /
```

Для **xfs** вместо `resize2fs`:

```bash
sudo xfs_growfs /
```

---

## 4. Новый диск → добавить в VG

```bash
sudo pvcreate /dev/sdb
sudo vgextend ubuntu-vg /dev/sdb
sudo lvextend -l +100%FREE /dev/ubuntu-vg/ubuntu-lv
sudo resize2fs /dev/ubuntu-vg/ubuntu-lv
```

---

## 5. Без LVM (простой раздел)

Иногда только:

```bash
sudo growpart /dev/sda 1
sudo resize2fs /dev/sda1
```

Или отдельный mount **`/data`** — проще перенос данных, чем resize root.

---

## 6. Ошибки

- **`growpart` не найден** — пакет `cloud-guest-utils`.
- Расширили LV, забыли **`resize2fs`/`xfs_growfs`** — `df` не изменится.
- Snapshot/backup перед первым разом на prod: [backups_cheatsheet.md](backups_cheatsheet.md).

---

## 7. Шпаргалка

```text
lsblk    pvs/vgs/lvs    growpart → pvresize → lvextend → resize2fs
```
