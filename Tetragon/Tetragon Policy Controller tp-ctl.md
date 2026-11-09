# Tetragon Policy Controller (`tp-ctl`)
## จัดการ TracingPolicy บน Rocky Linux / Docker

เอกสารนี้อธิบายการสร้างสคริปต์ `tp-ctl` สำหรับจัดการ Tetragon TracingPolicy (`.yaml`) โดยแยกไฟล์ Policy ที่ปิดใช้งานออกจาก Policy ที่เปิดใช้งาน และใช้ **Symbolic Link (Symlink)** เชื่อมไฟล์เข้าสู่โฟลเดอร์ที่ Tetragon ใช้งาน

---

## 1. เตรียมโฟลเดอร์เก็บ Policy

สร้างโฟลเดอร์สำหรับเก็บ Policy ที่ยังไม่เปิดใช้งาน และโฟลเดอร์ Policy ที่ Tetragon ใช้งาน

```bash
sudo install -d -m 0700 /etc/tetragon/disabled-policies
sudo install -d -m 0755 /etc/tetragon/tetragon.tp.d
```

แนวทางการจัดเก็บไฟล์:

- `/etc/tetragon/disabled-policies/` — เก็บไฟล์ Policy ต้นฉบับที่ยังไม่เปิดใช้งาน
- `/etc/tetragon/tetragon.tp.d/` — โฟลเดอร์ที่ Tetragon ใช้โหลด Policy โดยไฟล์ที่จัดการผ่าน `tp-ctl` จะเป็น Symlink ไปยังไฟล์ต้นฉบับ

> **หมายเหตุ:** ตรวจสอบให้แน่ใจว่า Tetragon บนเครื่องของคุณถูกกำหนดให้โหลด Policy จาก `/etc/tetragon/tetragon.tp.d/` จริง ก่อนนำสคริปต์นี้ไปใช้

---

## 2. สร้างสคริปต์ `tp-ctl`

สคริปต์นี้รองรับคำสั่งหลัก 3 กลุ่ม:

- `enable` — เปิดใช้งาน Policy โดยสร้าง Symlink และ Restart Tetragon
- `disable` — ปิดใช้งาน Policy โดยลบ Symlink หรือย้ายไฟล์ตรงออกจากโฟลเดอร์ทำงาน
- `status` — แสดง Policy ที่มีอยู่ สถานะ Symlink และสถานะ Policy จาก Tetragon

สร้างไฟล์ `/usr/bin/tp-ctl`:

```bash
sudo tee /usr/bin/tp-ctl > /dev/null << 'EOF'
#!/bin/bash
set -euo pipefail
export PATH="/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"

AVAIL_DIR="/etc/tetragon/disabled-policies"
ENABLED_DIR="/etc/tetragon/tetragon.tp.d"
TETRA_BIN="/usr/local/bin/tetra"
SOCK_FILE="/var/run/tetragon/tetragon.sock"

if [[ "${EUID}" -ne 0 ]]; then
    echo "[ERROR] Please run with sudo: sudo tp-ctl status"
    exit 1
fi

mkdir -p "$AVAIL_DIR" "$ENABLED_DIR"

# Restart Tetragon และรอจนกว่า Socket จะพร้อมใช้งาน (สูงสุด 15 วินาที)
restart_and_wait() {
    echo "[INFO] Restarting tetragon service..."
    systemctl restart tetragon

    local i=0
    while [[ ! -S "$SOCK_FILE" ]]; do
        sleep 1
        i=$((i + 1))

        # หาก Service หยุดทำงานระหว่างรอ ให้แสดง Log
        if ! systemctl is-active --quiet tetragon; then
            echo "[ERROR] Tetragon failed to start! Last 15 log lines:"
            journalctl -u tetragon -n 15 --no-pager
            exit 1
        fi

        if [[ "$i" -ge 15 ]]; then
            echo "[ERROR] Timeout waiting for $SOCK_FILE (15s)."
            journalctl -u tetragon -n 15 --no-pager
            exit 1
        fi
    done

    # รอให้ gRPC server พร้อมรับคำสั่ง
    sleep 1
}

ACTION="${1:-status}"
POLICY="${2:-}"

if [[ -n "$POLICY" && "$POLICY" != *.yaml ]]; then
    POLICY="${POLICY}.yaml"
fi

case "$ACTION" in
    enable|en)
        if [[ -z "$POLICY" ]]; then
            echo "Usage: sudo tp-ctl enable <policy-name>"
            exit 1
        fi

        # ย้ายไฟล์ตรงที่อยู่ในโฟลเดอร์ทำงานกลับไปเก็บเป็นไฟล์ต้นฉบับ
        if [[ -f "$ENABLED_DIR/$POLICY" && ! -L "$ENABLED_DIR/$POLICY" ]]; then
            mv "$ENABLED_DIR/$POLICY" "$AVAIL_DIR/$POLICY"
        fi

        if [[ ! -f "$AVAIL_DIR/$POLICY" ]]; then
            echo "[ERROR] Policy not found: $AVAIL_DIR/$POLICY"
            exit 1
        fi

        ln -sf "$AVAIL_DIR/$POLICY" "$ENABLED_DIR/$POLICY"
        echo "[OK] Enabled symlink: $POLICY"

        restart_and_wait
        "$TETRA_BIN" tracingpolicy list
        ;;

    disable|dis)
        if [[ -z "$POLICY" ]]; then
            echo "Usage: sudo tp-ctl disable <policy-name>"
            exit 1
        fi

        if [[ -f "$ENABLED_DIR/$POLICY" && ! -L "$ENABLED_DIR/$POLICY" ]]; then
            mv "$ENABLED_DIR/$POLICY" "$AVAIL_DIR/$POLICY"
            echo "[OK] Moved direct file to $AVAIL_DIR/$POLICY"
        elif [[ -L "$ENABLED_DIR/$POLICY" ]]; then
            rm -f "$ENABLED_DIR/$POLICY"
            echo "[OK] Removed symlink: $POLICY"
        else
            echo "[WARN] Policy '$POLICY' is not currently enabled."
            exit 0
        fi

        restart_and_wait
        "$TETRA_BIN" tracingpolicy list
        ;;

    status|list|ls)
        echo "=== 1. Available Policies ($AVAIL_DIR) ==="

        for f in "$AVAIL_DIR"/*.yaml; do
            [[ -e "$f" ]] || { echo "  (empty)"; break; }
            fname=$(basename "$f")

            if [[ -L "$ENABLED_DIR/$fname" ]]; then
                echo "  [ENABLED]  $fname (Symlink)"
            else
                echo "  [DISABLED] $fname"
            fi
        done

        echo ""
        echo "=== 2. Direct Files in $ENABLED_DIR ==="
        DIRECT_COUNT=0

        for f in "$ENABLED_DIR"/*.yaml; do
            [[ -e "$f" ]] || break

            if [[ ! -L "$f" ]]; then
                echo "  [DIRECT]   $(basename "$f")"
                DIRECT_COUNT=$((DIRECT_COUNT + 1))
            fi
        done

        [[ "$DIRECT_COUNT" -eq 0 ]] && echo "  (none - all managed via symlinks)"

        echo ""
        echo "=== 3. Kernel Status (tetra tracingpolicy list) ==="

        if [[ -S "$SOCK_FILE" ]]; then
            "$TETRA_BIN" tracingpolicy list
        else
            echo "[ERROR] Tetragon socket ($SOCK_FILE) not found. Service status:"
            systemctl status tetragon --no-pager || true
        fi
        ;;

    *)
        echo "Usage: sudo tp-ctl {enable|disable|status} [policy-name]"
        exit 1
        ;;
esac
EOF
```

กำหนด Permission ให้สามารถเรียกใช้งานได้:

```bash
sudo chmod 755 /usr/bin/tp-ctl
```

ตรวจสอบว่าไฟล์ถูกสร้างเรียบร้อย:

```bash
ls -l /usr/bin/tp-ctl
```

---

## 3. ทดสอบสคริปต์

เรียกดูสถานะ Policy และ Tetragon:

```bash
sudo tp-ctl status
```

ตัวอย่างการเรียกใช้งาน:

```text
=== 1. Available Policies (/etc/tetragon/disabled-policies) ===
  [DISABLED]  10-block-docker-socket.yaml
  [ENABLED]   20-monitor-process.yaml (Symlink)

=== 2. Direct Files in /etc/tetragon/tetragon.tp.d ===
  (none - all managed via symlinks)

=== 3. Kernel Status (tetra tracingpolicy list) ===
...
```

ผลลัพธ์จริงขึ้นอยู่กับ Policy และสถานะของเครื่อง

---

## 4. เพิ่ม Policy ใหม่เข้าสู่ระบบ

เมื่อมีไฟล์ Policy ใหม่ เช่น `10-block-docker-socket.yaml` ให้นำไฟล์ไปเก็บในโฟลเดอร์ Disabled ก่อน

```bash
sudo cp 10-block-docker-socket.yaml /etc/tetragon/disabled-policies/
```

ตรวจสอบไฟล์:

```bash
sudo ls -l /etc/tetragon/disabled-policies/
```

### เปิดใช้งานทีละไฟล์

```bash
sudo tp-ctl enable 10-block-docker-socket.yaml
```

หรือไม่ต้องระบุนามสกุล `.yaml` ก็ได้:

```bash
sudo tp-ctl enable 10-block-docker-socket
```

สคริปต์จะสร้าง Symlink ใน `/etc/tetragon/tetragon.tp.d/` จากนั้น Restart Tetragon และแสดงรายการ Policy

### เปิดใช้งานทุกไฟล์ด้วย `tp-ctl`

```bash
for f in /etc/tetragon/disabled-policies/*.yaml; do
    [[ -e "$f" ]] || continue
    sudo tp-ctl enable "$(basename "$f")"
done
```

**ข้อควรระวัง:** วิธีนี้จะ Restart Tetragon ทุกครั้งที่เปิดใช้งานหนึ่งไฟล์ หากมีหลาย Policy จะทำให้ Service Restart หลายรอบ

---

## 5. เปิดใช้งานทุก Policy ด้วย Symlink แล้ว Restart ครั้งเดียว

หากต้องการเปิดใช้งานทุกไฟล์พร้อมกัน สามารถสร้าง Symlink ทั้งหมดโดยตรง:

```bash
sudo ln -sf /etc/tetragon/disabled-policies/*.yaml /etc/tetragon/tetragon.tp.d/
```

ตรวจสอบ Symlink:

```bash
ls -l /etc/tetragon/tetragon.tp.d/
```

จากนั้น Restart Tetragon เพียงครั้งเดียว:

```bash
sudo systemctl reset-failed tetragon
sudo systemctl restart tetragon
sleep 3
sudo /usr/local/bin/tetra tracingpolicy list
```

> วิธีนี้เหมาะกับการเปิดใช้งานหลาย Policy พร้อมกัน แต่จะไม่ผ่านการตรวจสอบและการจัดการรายไฟล์ของ `tp-ctl` และอาจมี Symlink ค้างอยู่หากไฟล์ต้นฉบับถูกลบหรือเปลี่ยนชื่อ

---

## 6. ปิดใช้งาน Policy ทีละไฟล์

ใช้คำสั่ง `disable` ตามด้วยชื่อ Policy:

```bash
sudo tp-ctl disable 10-block-docker-socket.yaml
```

หรือ:

```bash
sudo tp-ctl disable 10-block-docker-socket
```

สำหรับ Policy ที่จัดการด้วย Symlink สคริปต์จะลบ Symlink ออกจาก `tetragon.tp.d` โดยไม่ลบไฟล์ต้นฉบับใน `disabled-policies`

---

## 7. ปิดใช้งานทุก Policy

ใช้ลูปเรียก `tp-ctl disable` สำหรับทุกไฟล์ `.yaml` ที่อยู่ในโฟลเดอร์ทำงาน:

```bash
for f in /etc/tetragon/tetragon.tp.d/*.yaml; do
    [[ -e "$f" || -L "$f" ]] || continue
    sudo tp-ctl disable "$(basename "$f" .yaml)"
done
```

**ข้อควรระวัง:** `tp-ctl disable` จะ Restart Tetragon ทุกครั้งที่ปิดหนึ่ง Policy หากมีหลายไฟล์จะเกิดการ Restart หลายรอบ และไฟล์ Policy ที่เป็นไฟล์ตรง (ไม่ใช่ Symlink) จะถูกย้ายกลับไปยัง `disabled-policies`

---

## 8. คำสั่งสรุปสำหรับใช้งานประจำวัน

| งาน | คำสั่ง |
|---|---|
| ดูสถานะ | `sudo tp-ctl status` |
| เปิด Policy | `sudo tp-ctl enable <ชื่อไฟล์>` |
| ปิด Policy | `sudo tp-ctl disable <ชื่อไฟล์>` |
| ดู Symlink | `ls -l /etc/tetragon/tetragon.tp.d/` |
| ดู Policy ที่ Tetragon โหลด | `sudo /usr/local/bin/tetra tracingpolicy list` |
| ดู Log ของ Service | `sudo journalctl -u tetragon -n 50 --no-pager` |

---

## 9. ข้อควรระวังด้านการดูแลระบบ

1. ตรวจสอบ YAML syntax และเนื้อหา Policy ก่อนเปิดใช้งานจริง
2. Policy บางรายการอาจมีผลต่อการทำงานของ Container หรือ Host ควรทดสอบใน Lab ก่อน Production
3. สำรองไฟล์ Policy ก่อนแก้ไข และควบคุมสิทธิ์การเขียนใน `/etc/tetragon/`
4. การใช้ `ln -sf` โดยตรงจะข้ามขั้นตอนตรวจสอบของ `tp-ctl`
5. `tetra tracingpolicy list` ใช้ตรวจสอบรายการ Policy ที่ Tetragon รายงานว่ากำลังใช้งาน ไม่ใช่การตรวจสอบว่า Policy ทุกข้อทำงานตรงตามวัตถุประสงค์แล้ว
6. หาก Tetragon ไม่พร้อมหลัง Restart ให้ตรวจสอบ `systemctl status tetragon` และ `journalctl -u tetragon` ก่อนสั่งงานซ้ำ

---

## Mindmap: แนวคิดการจัดการ Policy

```text
Tetragon Policy Management
│
├── disabled-policies/
│   └── เก็บไฟล์ YAML ต้นฉบับ
│       ├── 10-block-docker-socket.yaml
│       └── 20-monitor-process.yaml
│
├── tetragon.tp.d/
│   └── Symlink ไปยัง Policy ที่ต้องการเปิดใช้
│       ├── 10-block-docker-socket.yaml -> ../disabled-policies/
│       └── 20-monitor-process.yaml -> ../disabled-policies/
│
└── tp-ctl
    ├── enable  → สร้าง Symlink + Restart
    ├── disable → ลบ Symlink + Restart
    └── status  → ตรวจสอบไฟล์และสถานะ Tetragon
```

**แนวคิดสำคัญ:** เก็บไฟล์ Policy ต้นฉบับไว้ใน `disabled-policies` และควบคุมการเปิดใช้งานผ่าน Symlink ทำให้แยก Policy ที่มีอยู่ในระบบออกจาก Policy ที่ต้องการให้ Tetragon โหลดได้อย่างชัดเจน
