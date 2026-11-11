# ติดตั้งและจัดการ Tetragon บน Linux Host

คู่มือนี้อธิบายการติดตั้ง Tetragon v1.8.0 จาก Binary Release บนเครื่อง Host การติดตั้ง
CLI `tetra` การจัดการ TracingPolicy ด้วยสคริปต์ `tp-ctl` การตรวจสอบเหตุการณ์
SIGKILL และการ Reload Policy

> **หมายเหตุ:** ตัวอย่างคำสั่งใช้สถาปัตยกรรม `amd64` และติดตั้งบนระบบที่รองรับแพ็กเกจ
> RPM เช่น Rocky Linux โดยควรตรวจสอบ Release และเอกสารทางการก่อนใช้งานจริง

เอกสารอ้างอิง: - [Tetragon
Releases](https://github.com/cilium/tetragon/releases) - [Tetragon
Package Installation](https://tetragon.io/docs/installation/package/)

------------------------------------------------------------------------

## 1. ติดตั้ง Tetragon (Terminal 1)

ดาวน์โหลด Binary ของ Tetragon และติดตั้งลงบนเครื่อง Host โดยตรง

``` bash
cd /opt

sudo dnf install -y wget

sudo wget https://github.com/cilium/tetragon/releases/download/v1.8.0/tetragon-v1.8.0-amd64.tar.gz

sudo tar -xvf tetragon-v1.8.0-amd64.tar.gz

cd tetragon-v1.8.0-amd64/

# ติดตั้ง Tetragon ลงในระบบ
sudo ./install.sh
```

ตรวจสอบสถานะ Service:

``` bash
sudo systemctl status tetragon
```

------------------------------------------------------------------------

## 2. ติดตั้งเครื่องมือ Tetra CLI

`tetra` เป็น CLI สำหรับตรวจสอบและจัดการข้อมูลจาก Tetragon เช่น TracingPolicy
และ Event stream

``` bash
mkdir -p "$HOME/tetra-cli-v1.8.0"
cd "$HOME/tetra-cli-v1.8.0"

sudo curl -fLO \
  https://github.com/cilium/tetragon/releases/download/v1.8.0/tetra-linux-amd64.tar.gz

sudo tar -xzf tetra-linux-amd64.tar.gz

sudo install -m 0755 tetra /usr/local/bin/tetra
sudo restorecon -v /usr/local/bin/tetra
```

ตรวจสอบ Policy ที่โหลดอยู่ในระบบ:

``` bash
sudo /usr/local/bin/tetra tracingpolicy list
```

------------------------------------------------------------------------

## 3. ตรวจสอบและเปลี่ยนโหมด Policy

### 3.1 ดูรายการ Policy ที่โหลดอยู่

``` bash
sudo /usr/local/bin/tetra tracingpolicy list
```

### 3.2 เปลี่ยน Static Policy เป็น Monitor mode

ตัวอย่างเปลี่ยน Policy `14-file-protect-sensitive-data` เป็นโหมด Monitor:

``` bash
sudo /usr/local/bin/tetra tp set-mode \
  --domain static \
  14-file-protect-sensitive-data monitor
```

### 3.3 เปลี่ยน Static Policy เป็น Enforce mode

``` bash
sudo /usr/local/bin/tetra tp set-mode \
  --domain static \
  14-file-protect-sensitive-data enforce
```

> **ข้อควรระวัง:** ควรเริ่มทดสอบ Policy ในโหมด Monitor ก่อน
> เพื่อประเมินผลกระทบและลดความเสี่ยงจากการบล็อกการทำงานที่จำเป็น
> จากนั้นจึงพิจารณาเปลี่ยนเป็น Enforce

------------------------------------------------------------------------

## 4. สร้าง Directory สำหรับจัดเก็บ Policy

กำหนดให้ Directory นี้เป็นคลังเก็บไฟล์ Policy ที่ยังไม่เปิดใช้งาน

``` bash
sudo install -d -m 0700 /etc/tetragon/disabled-policies
```

นำไฟล์ Policy `.yaml` ไปเก็บไว้ที่:

``` text
/etc/tetragon/disabled-policies/
```

เมื่อจะเปิดใช้งาน Policy จะสร้าง Symlink ไปยัง Directory:

``` text
/etc/tetragon/tetragon.tp.d/
```

------------------------------------------------------------------------

## 5. สร้างสคริปต์ tp-ctl สำหรับจัดการ Policy

สคริปต์ `tp-ctl` ช่วยให้เปิดใช้งาน ปิดใช้งาน และตรวจสอบ Policy ได้สะดวกขึ้น โดยใช้
Symlink ระหว่าง Directory คลัง Policy กับ Directory ที่ Tetragon ใช้โหลด
Policy

สร้างสคริปต์:

``` bash
sudo tee /usr/bin/tp-ctl > /dev/null <<'EOF'
#!/bin/bash

AVAILABLE_DIR="/etc/tetragon/disabled-policies"
ENABLED_DIR="/etc/tetragon/tetragon.tp.d"

ACTION=$1
POLICY=$2

if [[ -z "$ACTION" ]]; then
    echo "Usage: tp-ctl {enable|disable|status|list} [policy_name|all]"
    exit 1
fi

case "$ACTION" in
    enable)
        if [[ "$POLICY" == "all" ]]; then
            echo "[INFO] Enabling ALL policies..."
            ln -sf "$AVAILABLE_DIR"/*.yaml "$ENABLED_DIR"/
            systemctl reset-failed tetragon 2>/dev/null
            systemctl restart tetragon
            echo "[OK] All policies enabled and Tetragon restarted."
            exit 0
        fi

        POLICY_FILE="${POLICY%.yaml}.yaml"

        if [[ ! -f "$AVAILABLE_DIR/$POLICY_FILE" ]]; then
            echo "[ERROR] Policy '$POLICY_FILE' not found in $AVAILABLE_DIR"
            exit 1
        fi

        ln -sf "$AVAILABLE_DIR/$POLICY_FILE" "$ENABLED_DIR/$POLICY_FILE"
        systemctl reset-failed tetragon 2>/dev/null
        systemctl restart tetragon
        echo "[OK] Enabled '$POLICY_FILE'"
        ;;

    disable)
        if [[ "$POLICY" == "all" ]]; then
            echo "[INFO] Disabling ALL policies..."
            rm -f "$ENABLED_DIR"/*.yaml
            systemctl reset-failed tetragon 2>/dev/null
            systemctl restart tetragon
            echo "[OK] All policies disabled and Tetragon restarted."
            exit 0
        fi

        POLICY_FILE="${POLICY%.yaml}.yaml"

        if [[ ! -L "$ENABLED_DIR/$POLICY_FILE" && ! -f "$ENABLED_DIR/$POLICY_FILE" ]]; then
            echo "[WARN] Policy '$POLICY_FILE' is not currently enabled."
            exit 1
        fi

        rm -f "$ENABLED_DIR/$POLICY_FILE"
        systemctl reset-failed tetragon 2>/dev/null
        systemctl restart tetragon
        echo "[OK] Disabled '$POLICY_FILE'"
        ;;

    status|list)
        echo "=== Enabled Policies ($ENABLED_DIR) ==="
        ls -1 "$ENABLED_DIR"/*.yaml 2>/dev/null | xargs -n 1 basename 2>/dev/null || echo "(None)"
        echo ""
        echo "=== Active in Tetragon Engine ==="
        /usr/local/bin/tetra tracingpolicy list
        ;;

    *)
        echo "Usage: tp-ctl {enable|disable|status|list} [policy_name|all]"
        exit 1
        ;;
esac
EOF
```

กำหนด Permission ให้สคริปต์:

``` bash
sudo chmod 755 /usr/bin/tp-ctl
```

> **ตรวจสอบ Path ให้ตรงกัน:** สคริปต์ถูกสร้างไว้ที่ `/usr/bin/tp-ctl` ดังนั้นให้กำหนด
> Permission ที่ `/usr/bin/tp-ctl` ไม่ใช่ `/usr/local/bin/tp-ctl`

------------------------------------------------------------------------

## 6. เปิดใช้งานและจัดการ Policy

### 6.1 เปิดใช้งาน Policy ทั้งหมดผ่าน Symlink

ก่อนใช้งาน ตรวจสอบว่า Directory ปลายทางมีอยู่แล้ว:

``` bash
sudo install -d -m 0755 /etc/tetragon/tetragon.tp.d
```

สร้าง Symlink จาก Policy ทั้งหมดในคลังไปยัง Directory ที่ Tetragon ใช้โหลด:

``` bash
sudo ln -sf /etc/tetragon/disabled-policies/*.yaml /etc/tetragon/tetragon.tp.d/
```

รีเซ็ตสถานะ Failed และ Restart Service:

``` bash
sudo systemctl reset-failed tetragon
sudo systemctl restart tetragon
```

### 6.2 ใช้ tp-ctl จัดการ Policy

เปิดใช้งาน Policy ทั้งหมด:

``` bash
sudo tp-ctl enable all
```

ตรวจสอบสถานะ Policy:

``` bash
sudo tp-ctl status
```

ปิดใช้งาน Policy ทั้งหมด:

``` bash
sudo tp-ctl disable all
```

ตรวจสอบรายการ Policy ที่ Tetragon โหลดอยู่:

``` bash
sudo /usr/local/bin/tetra tracingpolicy list
```

ตัวอย่างการเปิดหรือปิด Policy รายตัว:

``` bash
sudo tp-ctl enable 14-file-protect-sensitive-data
sudo tp-ctl disable 14-file-protect-sensitive-data
```

> **ข้อควรระวัง:** คำสั่ง `disable all` จะลบไฟล์ `.yaml` ใน Directory
> `tetragon.tp.d` ตามรูปแบบที่สคริปต์กำหนด จึงควรใช้ Directory นี้สำหรับ Symlink
> ของ Policy ที่จัดการด้วย `tp-ctl` เท่านั้น

------------------------------------------------------------------------

## 7. ตรวจสอบเหตุการณ์ SIGKILL แบบ Real-time

ตัวอย่างนี้ใช้ `tetra getevents` เพื่อรับ Event จาก Tetragon แล้วส่งต่อให้ `jq`
กรองเฉพาะเหตุการณ์ที่มี Action เกี่ยวข้องกับ `SIGKILL`

### 7.1 เตรียม jq

ติดตั้ง `jq` หากเครื่องยังไม่มี:

``` bash
sudo dnf install -y jq
```

### 7.2 เริ่มรับ Event

``` bash
sudo -v

printf '\033[1;33mListening for SIGKILL events... Press Ctrl+C to stop.\033[0m\n'

sudo /usr/local/bin/tetra getevents -o json \
  | jq --unbuffered -r '
      def txt:
        if . == null or . == "" then "-"
        else tostring
        end;

      (.process_kprobe // .process_tracepoint // .process_uprobe) as $ev
      | select($ev != null)
      | select(($ev.action // "" | tostring | contains("SIGKILL")))

      | (
          .time as $raw
          | try (
              $raw
              | sub("\\.[0-9]+Z$"; "Z")
              | fromdateiso8601
              | . + 25200
              | strftime("%Y-%m-%d %H:%M:%S")
            ) catch ($raw // "-")
        ) as $time

      | ($ev.policy_name // "-") as $policy
      | ($ev.process.docker // "") as $cid

      | (
          [
            $ev.args[]?
            | (
                .linux_binprm_arg.path?,
                .file_arg.path?,
                .string_arg?,
                (
                  if (.sock_arg.daddr? // "") != "" then
                    "\(.sock_arg.daddr):\(.sock_arg.dport // "-")"
                  else
                    empty
                  end
                )
              )
            | select(. != null and . != "")
            | tostring
          ]
          | unique
          | if length == 0 then "-" else join(" ; ") end
        ) as $target

      | (
          if ($policy | startswith("31-")) then "\u001b[36m"
          elif ($policy | startswith("33-")) then "\u001b[35m"
          else "\u001b[32m"
          end
        ) as $color

      | "\u001b[0m" as $reset
      | "\u001b[1;33m" as $yellow

      | [
          "",
          "\($color)======================================================================\($reset)",
          "Time (+07) : \($time)",
          "Node       : \(.node_name | txt)",
          "Location   : \(if $cid != "" then "CONTAINER " + $cid[0:12] else "HOST / UNKNOWN" end)",
          "Policy     : \($color)\($policy)\($reset)",
          "Hook       : \(($ev.function_name // $ev.call // $ev.subsystem) | txt)",
          "Action     : \($yellow)\($ev.action | txt)\($reset)",
          "----------------------------------------------------------------------",
          "Process    : \($ev.process.binary | txt)",
          "PID / UID  : \($ev.process.pid | txt) / \($ev.process.uid | txt)",
          "Arguments  : \($ev.process.arguments | txt)",
          "Parent     : \($ev.parent.binary | txt)",
          "Parent PID : \($ev.parent.pid | txt)",
          "Parent args: \($ev.parent.arguments | txt)",
          "Target     : \($yellow)\($target)\($reset)",
          "ContainerID: \($cid | txt)",
          "Exec ID    : \($ev.process.exec_id | txt)",
          "\($color)======================================================================\($reset)"
        ]
      | .[]
    '

stream_status=("${PIPESTATUS[@]}")
printf '\nStream ended: tetra=%s | jq=%s\n' \
  "${stream_status[0]}" "${stream_status[1]}"
```

กด `Ctrl+C` เพื่อหยุดการรับ Event

**หมายเหตุเกี่ยวกับ Timezone:** ค่า `+25200` คือการบวก 7 ชั่วโมงเพื่อแสดงเวลา
UTC+7 (เวลาไทย) โดยสมมติว่า Timestamp ต้นทางเป็น UTC

------------------------------------------------------------------------

## 8. Reload Policy ใน Tetragon

ในบางกรณีต้องการลบ Policy ออกจาก Tetragon Engine ก่อนเริ่ม Service ใหม่ เช่น
เมื่อมี Policy ค้างอยู่หรือจำเป็นต้องโหลด Policy ใหม่

ตัวอย่างลบ Policy `31-proc-prevent-shell-in-container`:

``` bash
sudo /usr/local/bin/tetra tracingpolicy delete \
  31-proc-prevent-shell-in-container 2>/dev/null

sudo systemctl reset-failed tetragon
sudo systemctl restart tetragon

sleep 3

sudo /usr/local/bin/tetra tracingpolicy list
```

> **ข้อควรระวัง:** การลบ Policy ด้วย `tracingpolicy delete` จะมีผลกับ Policy
> ที่กำลังอยู่ใน Engine ส่วนไฟล์ต้นฉบับใน Directory คลังยังคงอยู่ หากไฟล์ยังอยู่ใน
> Directory ที่ Tetragon เฝ้าดู อาจถูกโหลดกลับมาได้ตามกลไกการทำงานของ Service
> และการตั้งค่าในระบบ

------------------------------------------------------------------------

## 9. สรุปคำสั่งที่ใช้บ่อย

``` bash
# ตรวจสอบ Service
sudo systemctl status tetragon

# ดู Policy ใน Engine
sudo tetra tracingpolicy list

# เปิดใช้งาน Policy ทั้งหมด
sudo tp-ctl enable all

# ปิดใช้งาน Policy ทั้งหมด
sudo tp-ctl disable all

# ตรวจสอบ Policy ที่เปิดใช้งาน
sudo tp-ctl status

# เปิดใช้งาน Policy รายตัว
sudo tp-ctl enable 14-file-protect-sensitive-data

# ปิดใช้งาน Policy รายตัว
sudo tp-ctl disable 14-file-protect-sensitive-data

# Restart Tetragon
sudo systemctl restart tetragon
```

------------------------------------------------------------------------

## 10. แนวทางปฏิบัติที่แนะนำ

-   ตรวจสอบความเข้ากันได้ของ Tetragon กับ Kernel และ Linux Distribution
    ก่อนติดตั้ง
-   เริ่มจาก Monitor mode ก่อนใช้ Enforce mode
-   เก็บไฟล์ Policy ต้นฉบับไว้ใน `/etc/tetragon/disabled-policies/` และใช้
    Symlink เพื่อควบคุมการเปิดใช้งาน
-   ตรวจสอบ Log และรายการ Policy ใน Engine หลัง Restart ทุกครั้ง
-   ทดสอบ Policy ใน Lab ก่อนนำไปใช้กับ Production
-   สำรองไฟล์ Policy และสคริปต์ `tp-ctl` ก่อนแก้ไข
