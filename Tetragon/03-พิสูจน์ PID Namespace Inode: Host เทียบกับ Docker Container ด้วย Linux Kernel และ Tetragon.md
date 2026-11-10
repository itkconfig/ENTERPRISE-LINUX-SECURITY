# 🔬 พิสูจน์ PID Namespace Inode: Host เทียบกับ Docker Container ด้วย Linux Kernel และ Tetragon

## 🎯 เป้าหมาย

บทความนี้สาธิตวิธีตรวจสอบว่าโปรเซสที่ทำงานบนเครื่องแม่ (Host) อยู่ใน **PID Namespace
เดียวกับ PID 1 ของ Host** หรือไม่ และเปรียบเทียบกับโปรเซสภายใน Docker Container

แนวคิดสำคัญคือ Linux แยกโปรเซสออกเป็น PID Namespace โดยแต่ละ Namespace
มีหมายเลข inode ของตนเอง เราสามารถอ่านหมายเลขนี้ได้จาก `/proc/<PID>/ns/pid`
และใช้เป็นหลักฐานประกอบการตรวจสอบ Policy ของ Tetragon

> **ข้อควรเข้าใจ:** `host_ns` ไม่ใช่ชื่อมาตรฐานที่ Kernel กำหนดให้โดยตรง
> แต่เป็นชื่อตัวแปร/ค่าที่ผู้เขียน Policy กำหนดขึ้น โดยทั่วไปจะกำหนดให้เท่ากับ inode ของ
> PID Namespace ที่ Host ใช้งาน เช่นค่าจาก `/proc/1/ns/pid` ในมุมมองของ Host

## 1. พิสูจน์จาก Linux Kernel ผ่าน `/proc`

สมมติว่าเครื่อง Host คือ `docker-o1` และ Container ชื่อ `corp_wp_prod_app`

รันคำสั่งต่อไปนี้บน Host:

``` bash
echo "1. PID Namespace ของ PID 1 บน Host:"
sudo readlink /proc/1/ns/pid

echo
echo "2. PID Namespace ของ Shell ปัจจุบัน:"
readlink /proc/self/ns/pid

echo
echo "3. PID Namespace ของ Shell ที่ใช้รันคำสั่งบน Host:"
sh -c 'readlink /proc/$$/ns/pid'

echo
echo "4. PID Namespace ของโปรเซสหลักใน Container:"
CONTAINER_PID=$(sudo docker inspect -f '{{.State.Pid}}' corp_wp_prod_app)
echo "Container init PID on Host: ${CONTAINER_PID}"
sudo readlink "/proc/${CONTAINER_PID}/ns/pid"
```

### วิธีอ่านผลลัพธ์

ตัวอย่างผลลัพธ์ (หมายเลข inode เป็นเพียงตัวอย่าง):

``` text
1. PID Namespace ของ PID 1 บน Host:
pid:[4026531836]

2. PID Namespace ของ Shell ปัจจุบัน:
pid:[4026531836]

3. PID Namespace ของ Shell ที่ใช้รันคำสั่งบน Host:
pid:[4026531836]

4. PID Namespace ของโปรเซสหลักใน Container:
Container init PID on Host: 24871
pid:[4026532590]
```

-   ข้อ 1 คือ inode ของ PID Namespace ที่ PID 1 ของ Host ใช้งาน
-   ข้อ 2 และ 3 ควรตรงกับข้อ 1 หาก Shell และคำสั่งทำงานอยู่ใน PID Namespace
    เดียวกับ Host
-   ข้อ 4 ควรเป็น inode คนละค่า เมื่อ Container ใช้ PID Namespace แยกจาก Host

ดังนั้น หาก Policy กำหนด `host_ns` ให้เท่ากับ `4026531836` โปรเซสบน Host จะมีค่า
Namespace ตรงกับ `host_ns` ส่วนโปรเซสใน Container จะมีค่าต่างออกไป

**ข้อควรระวัง:** หากใช้ `--pid=host` หรือกำหนดให้ Container แชร์ PID Namespace
กับ Host ค่าในข้อ 4 อาจตรงกับ Host ได้ และหากคำสั่งทำงานจาก Shell ที่อยู่ใน
Namespace ซ้อนอีกชั้น ข้อ 2 ก็อาจไม่ตรงกับข้อ 1

## 2. พิสูจน์ด้วย `lsns`

คำสั่ง `lsns` แสดง Namespace ที่ระบบกำลังใช้งานอยู่ รวมถึง PID Namespace:

``` bash
sudo lsns -t pid
```

ตัวอย่างรูปแบบผลลัพธ์:

``` text
        NS TYPE NPROCS   PID USER COMMAND
4026531836 pid     ...     1 root /usr/lib/systemd/systemd
4026532590 pid     ... 24871 root /usr/bin/containerd-shim
```

คอลัมน์ `NS` คือ Namespace inode ซึ่งสามารถนำไปเทียบกับผลจาก
`readlink /proc/<PID>/ns/pid` ได้

แนวทางตรวจสอบ:

1.  หาแถวที่มี PID เป็น `1` และตรวจสอบว่าเป็น systemd หรือ init process ของ
    Host
2.  หา PID ของ Container จาก `docker inspect`
3.  ตรวจสอบว่า PID ของ Container อยู่ในแถวที่มีค่า `NS` แตกต่างจาก Host หรือไม่

คำสั่งช่วยตรวจสอบแบบเจาะจง:

``` bash
CONTAINER_PID=$(sudo docker inspect -f '{{.State.Pid}}' corp_wp_prod_app)

echo "Host:"
sudo lsns -t pid -p 1

echo
echo "Container init process (Host PID ${CONTAINER_PID}):"
sudo lsns -t pid -p "${CONTAINER_PID}"
```

> `lsns` อาจแสดงโปรเซสตัวแทนของ Namespace ไม่ใช่รายชื่อทุกโปรเซสใน Container
> และผลลัพธ์ขึ้นกับสิทธิ์ที่ใช้รัน ควรใช้ `sudo` และเทียบค่า `NS` เป็นหลัก
> ไม่ควรสรุปจากลำดับแถวหรือชื่อโปรเซสเพียงอย่างเดียว

## 3. ตรวจสอบหลักฐานจาก Tetragon

Tetragon ตรวจจับเหตุการณ์ระดับ Kernel และส่งข้อมูลเหตุการณ์ออกมาเป็น JSON
โดยข้อมูลโปรเซสอาจมีฟิลด์ เช่น binary, arguments, PID และ Docker metadata
ทั้งนี้โครงสร้าง JSON อาจแตกต่างกันตามเวอร์ชันและรูปแบบการตั้งค่า

เริ่มจากตรวจสอบว่ามีไฟล์ Log อยู่จริง:

``` bash
sudo ls -lh /var/log/tetragon/
```

หากใช้ไฟล์ `/var/log/tetragon/tetragon.log` และเป็น JSON Lines
สามารถดูเหตุการณ์ `process_exec` ล่าสุดได้ดังนี้:

``` bash
sudo jq -c '
  select(.process_exec != null) |
  {
    binary: .process_exec.process.binary,
    arguments: .process_exec.process.arguments,
    pid: .process_exec.process.pid,
    docker: .process_exec.process.docker,
    process: .process_exec.process
  }
' /var/log/tetragon/tetragon.log | tail -n 10
```

### ตรวจสอบว่ามีฟิลด์ Namespace อยู่ใน Event หรือไม่

อย่าเพิ่งสมมติว่าทุกเวอร์ชันหรือทุก Event จะมีฟิลด์ `ns.pid.inum` ในตำแหน่งเดียวกัน
ให้ตรวจสอบโครงสร้าง JSON ที่ระบบบันทึกจริงก่อน:

``` bash
sudo jq '
  select(.process_exec != null) |
  .process_exec.process |
  keys
' /var/log/tetragon/tetragon.log | tail -n 30
```

หาก Event มีข้อมูล Namespace ให้ตรวจดูโครงสร้างจริงของ Event:

``` bash
sudo jq '
  select(.process_exec != null) |
  .process_exec
' /var/log/tetragon/tetragon.log | tail -n 100
```

จากนั้นจึงเลือก JSON path ที่ตรงกับ schema ของ Tetragon เวอร์ชันที่ติดตั้งอยู่ หาก Log
ไม่มีฟิลด์ inode ของ PID Namespace โดยตรง ก็ไม่ควรสร้างข้อสรุปว่าได้พิสูจน์ inode จาก
Log แล้ว

### Docker metadata ใช้ประกอบการวิเคราะห์ ไม่ใช่หลักฐาน inode โดยตรง

ตัวอย่างการดึง Docker metadata:

``` bash
sudo jq -r '
  select(.process_exec != null) |
  "Binary: \(.process_exec.process.binary // "-") | " +
  "PID: \(.process_exec.process.pid // "-") | " +
  "Docker: \(.process_exec.process.docker // "-")"
' /var/log/tetragon/tetragon.log | tail -n 10
```

ฟิลด์ `docker` ใช้ช่วยบ่งชี้ว่า Event เกี่ยวข้องกับ Container หรือไม่ แต่การไม่มีค่า
`docker` **ไม่ได้พิสูจน์ด้วยตัวมันเอง** ว่าโปรเซสอยู่ใน Host PID Namespace
และการมีค่า `docker` ก็ไม่ใช่การแสดง inode โดยตรง

หลักฐานที่ชัดเจนสำหรับการเทียบ inode คืออ่านค่าจาก `/proc` ของโปรเซสที่เกี่ยวข้อง
แล้วเทียบกับค่า `host_ns` ที่ Policy ใช้จริง

## 4. เชื่อมโยงกับ Tetragon Policy: `NotIn: ["host_ns"]`

สมมติว่า Policy มีเงื่อนไขลักษณะนี้:

``` yaml
- operator: NotIn
  values:
    - "host_ns"
```

ต้องตรวจสอบให้แน่ใจว่า `host_ns` ใน Policy ถูกแทนด้วยค่า inode จริงตามกลไกและ
syntax ที่ Policy เวอร์ชันนั้นรองรับ ไม่ใช่ข้อความตัวอักษร `"host_ns"`
ที่ถูกนำไปเปรียบเทียบตรง ๆ

เมื่อกำหนดค่า Host Namespace ถูกต้อง หลักการทำงานคือ:

``` text
Host PID Namespace
        |
        | inode = host_ns
        |
        +---- Process: nc
        |       Namespace inode ตรงกับ host_ns
        |       ไม่เข้าเงื่อนไข NotIn
        |
        +---- Container Process: nc
                Namespace inode ต่างจาก host_ns
                เข้าเงื่อนไข NotIn
```

**สำคัญ:** การเข้าเงื่อนไข `NotIn` ไม่ได้แปลว่า Kernel จะบล็อกโปรเซสโดยอัตโนมัติ
ผลลัพธ์ว่าจะส่งสัญญาณ ปฏิเสธการทำงาน หรือดำเนินการอย่างไร ขึ้นอยู่กับ hook, return
action และรายละเอียดอื่นใน TracingPolicy ที่ใช้งานจริง ต้องตรวจสอบ Policy
ทั้งฉบับประกอบด้วย

## 5. สรุปผลการพิสูจน์

  -----------------------------------------------------------------------
  วิธีตรวจสอบ                           สิ่งที่พิสูจน์ได้
  ----------------------------------- -----------------------------------
  `/proc/<PID>/ns/pid`                inode ของ PID Namespace
                                      ของโปรเซสที่ระบุ

  `lsns -t pid`                       ภาพรวม Namespace และ inode
                                      ที่ระบบมองเห็น

  Tetragon JSON Log                   เหตุการณ์และ metadata ของโปรเซสตาม
                                      schema ที่ Event ส่งออก

  ตรวจ Policy                         ค่า `host_ns` และเงื่อนไขที่ใช้ตัดสินใจจริง
  -----------------------------------------------------------------------

หัวใจสำคัญคือ **อย่าใช้ Docker metadata แทน PID Namespace inode** ให้ใช้
`/proc` และ `lsns` เป็นหลักฐานระดับ Linux Kernel ส่วน Log ของ Tetragon
ใช้ยืนยันว่า Event ใดเกิดขึ้นกับโปรเซสใด และใช้ตรวจสอบข้อมูลที่ Policy ส่งออกมา

เมื่อหลักฐานทั้งสามส่วนสอดคล้องกัน เราจึงอธิบายได้อย่างมีเหตุผลว่าเหตุใดโปรเซสบน Host
จึงไม่เข้าเงื่อนไขที่ตั้งใจใช้กับ Container แต่โปรเซสภายใน Container
จึงเข้าเงื่อนไขดังกล่าว

------------------------------------------------------------------------

📌 **หมายเหตุสำหรับการทดสอบใน Lab**

-   ตรวจสอบชื่อ Container ให้ตรงกับระบบจริง
-   ตรวจสอบว่า Container ไม่ได้ใช้ `--pid=host`
-   รันคำสั่งตรวจสอบจาก Host namespace
-   ตรวจสอบค่า `host_ns` ที่ Policy ใช้จริงก่อนเปรียบเทียบ
-   ทดสอบกับ Container ที่ไม่ใช่งาน Production ก่อนเสมอ
-   อย่าสรุปว่า Tetragon บล็อกโปรเซสแล้วจากการเห็น Event เพียงอย่างเดียว
    ต้องตรวจสอบ action และผลลัพธ์จริงด้วย

#Tetragon #LinuxKernel #PIDNamespace #DockerSecurity #RockyLinux
#ContainerSecurity #eBPF #ThreatHunting
