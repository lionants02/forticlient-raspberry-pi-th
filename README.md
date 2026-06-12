# คู่มือติดตั้ง FortiClient VPN บน Raspberry Pi

คู่มือนี้อธิบายวิธีใช้งาน VPN ของ Fortinet/FortiGate บน Raspberry Pi พร้อมวิธีสร้างคำสั่งและปุ่มบนหน้าจอสำหรับกดเปิด/ปิด VPN

> หมายเหตุสำคัญ: Raspberry Pi ส่วนใหญ่ใช้สถาปัตยกรรม ARM/ARM64 เช่น `armv7l` หรือ `aarch64` แต่ FortiClient VPN แบบ official สำหรับ Linux มักเป็นแพ็กเกจสำหรับเครื่อง x86_64/amd64 จึงติดตั้ง `.deb` ของ FortiClient โดยตรงบน Raspberry Pi ไม่ได้ในหลายกรณี แนวทางที่ใช้ได้จริงบน Raspberry Pi คือใช้ `openfortivpn` ซึ่งเป็น client แบบ command line ที่รองรับ Fortinet SSL VPN

## สารบัญ

- [คู่มือติดตั้ง FortiClient VPN บน Raspberry Pi](#คู่มือติดตั้ง-forticlient-vpn-บน-raspberry-pi)
  - [สารบัญ](#สารบัญ)
  - [ภาพรวม](#ภาพรวม)
  - [สิ่งที่ต้องเตรียม](#สิ่งที่ต้องเตรียม)
  - [ติดตั้งแพ็กเกจ](#ติดตั้งแพ็กเกจ)
  - [สร้างไฟล์ config VPN](#สร้างไฟล์-config-vpn)
    - [กรณี URL ที่ได้รับเป็น `https://ip-address:10443/vpn`](#กรณี-url-ที่ได้รับเป็น-httpsip-address10443vpn)
    - [กรณี certificate ของ FortiGate ยังไม่ถูก trust](#กรณี-certificate-ของ-fortigate-ยังไม่ถูก-trust)
  - [ทดสอบเชื่อมต่อแบบ command line](#ทดสอบเชื่อมต่อแบบ-command-line)
  - [วิธีที่ 1: สร้างคำสั่งเปิด/ปิด VPN แบบ systemd](#วิธีที่-1-สร้างคำสั่งเปิดปิด-vpn-แบบ-systemd)
  - [วิธีที่ 2: สร้างสคริปต์เปิด/ปิด VPN แบบ command](#วิธีที่-2-สร้างสคริปต์เปิดปิด-vpn-แบบ-command)
  - [สร้างปุ่มบนหน้าจอ Desktop](#สร้างปุ่มบนหน้าจอ-desktop)
  - [ปุ่ม Desktop แบบใช้ systemd service](#ปุ่ม-desktop-แบบใช้-systemd-service)
  - [ทำให้กดปุ่มแล้วไม่ต้องถาม sudo password](#ทำให้กดปุ่มแล้วไม่ต้องถาม-sudo-password)
  - [ใช้งานกับ SAML/SSO หรือ OTP](#ใช้งานกับ-samlsso-หรือ-otp)
  - [Troubleshooting](#troubleshooting)
    - [`Gateway certificate validation failed`](#gateway-certificate-validation-failed)
  - [ถอนการติดตั้ง](#ถอนการติดตั้ง)
  - [อ้างอิง](#อ้างอิง)
  - [เครดิต](#เครดิต)

## ภาพรวม

- เหมาะกับ Raspberry Pi OS / Debian / Ubuntu บน Raspberry Pi
- ใช้กับ Fortinet SSL VPN ผ่าน `openfortivpn`
- มีตัวอย่างทั้งแบบ command line, systemd service, และปุ่ม Desktop
- ถ้าองค์กรบังคับใช้ FortiClient EMS, ZTNA, IPsec เฉพาะ client, หรือ SSO/SAML ที่ซับซ้อนมาก อาจต้องใช้เครื่อง Linux x86_64 หรือสอบถามผู้ดูแลระบบ VPN เพิ่มเติม

## สิ่งที่ต้องเตรียม

ขอข้อมูล VPN จากผู้ดูแลระบบก่อนเริ่ม:

- VPN host เช่น `vpn.example.com`
- Port เช่น `443` หรือ `10443`
- Username
- Password หรือวิธี OTP/2FA
- Realm ถ้ามี เช่น `employees`
- ค่า trusted certificate ถ้าองค์กรกำหนดไว้

ตรวจสอบสถาปัตยกรรมของ Raspberry Pi:

```bash
uname -m
```

ถ้าได้ `aarch64`, `armv7l`, หรือ `armhf` ให้ใช้แนวทาง `openfortivpn` ในคู่มือนี้

## ติดตั้งแพ็กเกจ

อัปเดตระบบและติดตั้งเครื่องมือที่จำเป็น:

```bash
sudo apt update
sudo apt install -y openfortivpn ppp lxterminal
```

ถ้าใช้ Raspberry Pi OS แบบ Desktop อยู่แล้ว มักมี `lxterminal` อยู่แล้ว แต่คำสั่งด้านบนใส่ไว้ให้ครบสำหรับปุ่ม Desktop

ตรวจสอบว่า `openfortivpn` ติดตั้งสำเร็จ:

```bash
openfortivpn --version
```

## สร้างไฟล์ config VPN

สร้างโฟลเดอร์ config:

```bash
sudo install -d -m 700 /etc/openfortivpn
```

สร้างไฟล์ config:

```bash
sudo vim /etc/openfortivpn/company.conf
```

ใส่ค่าตัวอย่างนี้ แล้วแก้ให้ตรงกับระบบของคุณ:

```ini
host = vpn.example.com
port = 443
username = your_username

# ถ้ามี realm ให้เอา # ออก แล้วแก้ค่า
# realm = employees

# แนะนำให้ไม่ใส่ password ในไฟล์นี้ เพื่อความปลอดภัย
# โปรแกรมจะถาม password/OTP ตอน connect
# password = your_password

# ถ้า VPN แจ้ง certificate digest ให้ใส่ค่า trusted-cert ที่นี่
# trusted-cert = 0123456789abcdef0123456789abcdef0123456789abcdef0123456789abcdef
```

### กรณี URL ที่ได้รับเป็น `https://ip-address:10443/vpn`

อย่าใส่ `https://`, port, หรือ path ลงในค่า `host` เพราะ `openfortivpn` จะเอาค่า `host` ไป resolve เป็นชื่อเครื่องโดยตรง ถ้าใส่ทั้ง URL จะเจอ error เช่น `getaddrinfo: Name or service not known`

ให้แยกค่าแบบนี้:

```ini
host = ip-address
port = 10443
username = my-vpn-user
realm = vpn
```

ถ้า `ip-address` ในตัวอย่างคือ IP จริง ให้ใส่เฉพาะ IP เช่น `192.0.2.10` หรือถ้าเป็นชื่อ DNS ให้ใส่เฉพาะชื่อ เช่น `vpn.example.com` ส่วน `/vpn` มักเป็น realm ของ FortiGate SSL VPN จึงใส่เป็น `realm = vpn`

### กรณี certificate ของ FortiGate ยังไม่ถูก trust

ถ้าเชื่อมต่อแล้วเจอ `Gateway certificate validation failed` ให้ตรวจสอบ certificate กับผู้ดูแลระบบก่อน ถ้าแน่ใจว่าเป็น FortiGate ขององค์กรจริง ให้คัดลอกค่า sha256 digest ที่ `openfortivpn` แสดง แล้วเพิ่มลงใน config:

```ini
trusted-cert = aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

ตัวอย่าง config สำหรับ URL `https://ip-address:10443/vpn` พร้อม certificate digest:

```ini
host = ip-address
port = 10443
username = my-vpn-user
realm = vpn
trusted-cert = aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

ค่า `aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa` เป็นค่า mock สำหรับตัวอย่างเท่านั้น ใช้งานจริงไม่ได้ ให้ใช้ digest ที่ `openfortivpn` แสดงจากเครื่องของคุณ

ถ้า certificate ของ FortiGate ถูกเปลี่ยนหรือต่ออายุในอนาคต ค่า digest จะเปลี่ยนด้วย ต้องแก้ `trusted-cert` ในไฟล์ config ให้เป็นค่าใหม่

ตั้ง permission ให้ไฟล์ config:

```bash
sudo chmod 600 /etc/openfortivpn/company.conf
```

## ทดสอบเชื่อมต่อแบบ command line

เชื่อมต่อ VPN:

```bash
sudo openfortivpn -c /etc/openfortivpn/company.conf
```

ถ้าเชื่อมต่อสำเร็จ ให้เปิด terminal นี้ค้างไว้ การตัด VPN ทำได้โดยกด:

```text
Ctrl+C
```

ถ้าครั้งแรกเจอ error เรื่อง certificate และคุณยืนยันกับผู้ดูแลระบบแล้วว่า certificate ถูกต้อง ให้คัดลอกค่า digest ที่โปรแกรมแสดง แล้วใส่ใน `trusted-cert` ของไฟล์ `/etc/openfortivpn/company.conf`

ตรวจสอบ tunnel หลังเชื่อมต่อ:

```bash
ip addr show ppp0
ip route
```

## วิธีที่ 1: สร้างคำสั่งเปิด/ปิด VPN แบบ systemd

วิธีนี้เหมาะกับการเปิด/ปิดด้วยคำสั่งสั้น ๆ และทำปุ่ม Desktop ได้ง่าย แต่ไม่เหมาะกับ VPN ที่ต้องกรอก OTP แบบ interactive ทุกครั้ง เว้นแต่คุณตั้งค่าการยืนยันตัวตนให้ service ใช้งานได้เอง

สร้าง service:

```bash
sudo vim /etc/systemd/system/fortivpn.service
```

ใส่เนื้อหานี้:

```ini
[Unit]
Description=Fortinet SSL VPN via openfortivpn
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
ExecStart=/usr/bin/openfortivpn -c /etc/openfortivpn/company.conf
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

โหลด service ใหม่:

```bash
sudo systemctl daemon-reload
```

เปิด VPN:

```bash
sudo systemctl start fortivpn.service
```

ปิด VPN:

```bash
sudo systemctl stop fortivpn.service
```

ดูสถานะ:

```bash
systemctl status fortivpn.service
```

ดู log:

```bash
journalctl -u fortivpn.service -f
```

ถ้าต้องการให้ VPN ติดเองตอนเปิดเครื่อง:

```bash
sudo systemctl enable fortivpn.service
```

ยกเลิกการติดเองตอนเปิดเครื่อง:

```bash
sudo systemctl disable fortivpn.service
```

## วิธีที่ 2: สร้างสคริปต์เปิด/ปิด VPN แบบ command

วิธีนี้เหมาะกับการใช้งานจาก terminal หรือทำปุ่ม Desktop ที่เปิดหน้าต่าง terminal ให้กรอก password/OTP ได้

สร้างสคริปต์เปิด VPN:

```bash
sudo vim /usr/local/bin/vpn-up
```

ใส่เนื้อหานี้:

```bash
#!/usr/bin/env bash
set -euo pipefail

CONFIG="/etc/openfortivpn/company.conf"

if pgrep -f "openfortivpn.*${CONFIG}" >/dev/null; then
  echo "VPN is already running."
  exit 0
fi

sudo /usr/bin/openfortivpn -c "$CONFIG"
```

สร้างสคริปต์ปิด VPN:

```bash
sudo vim /usr/local/bin/vpn-down
```

ใส่เนื้อหานี้:

```bash
#!/usr/bin/env bash
set -euo pipefail

if pgrep -x openfortivpn >/dev/null; then
  sudo pkill -INT -x openfortivpn
  echo "VPN disconnect signal sent."
else
  echo "VPN is not running."
fi
```

ทำให้สคริปต์รันได้:

```bash
sudo chmod 755 /usr/local/bin/vpn-up /usr/local/bin/vpn-down
```

ใช้งาน:

```bash
vpn-up
vpn-down
```

หมายเหตุ: `vpn-up` จะเปิดค้างจนกว่าจะตัด VPN ถ้าต้องกรอก password/OTP ให้กรอกใน terminal ที่เปิดสคริปต์นี้

## สร้างปุ่มบนหน้าจอ Desktop

ส่วนนี้สร้างไอคอนบน Desktop 2 ปุ่ม คือ Connect และ Disconnect

ถ้ายังไม่มีโฟลเดอร์ Desktop ให้สร้างก่อน:

```bash
mkdir -p ~/Desktop
```

สร้างไฟล์ปุ่ม Connect:

```bash
vim ~/Desktop/VPN-Connect.desktop
```

ใส่เนื้อหานี้:

```ini
[Desktop Entry]
Type=Application
Name=VPN Connect
Comment=Connect Fortinet SSL VPN
Exec=lxterminal -e bash -lc "vpn-up; echo; read -p 'Press Enter to close...'"
Icon=network-vpn
Terminal=false
Categories=Network;
```

สร้างไฟล์ปุ่ม Disconnect:

```bash
vim ~/Desktop/VPN-Disconnect.desktop
```

ใส่เนื้อหานี้:

```ini
[Desktop Entry]
Type=Application
Name=VPN Disconnect
Comment=Disconnect Fortinet SSL VPN
Exec=lxterminal -e bash -lc "vpn-down; echo; read -p 'Press Enter to close...'"
Icon=network-vpn
Terminal=false
Categories=Network;
```

ตั้ง permission:

```bash
chmod +x ~/Desktop/VPN-Connect.desktop ~/Desktop/VPN-Disconnect.desktop
```

ถ้า Desktop ถามว่า Trust หรือ Allow Launching ให้คลิกอนุญาตก่อนใช้งาน

## ปุ่ม Desktop แบบใช้ systemd service

ถ้าคุณเลือกวิธี systemd และไม่ต้องกรอก OTP ใน terminal สามารถใช้ปุ่มแบบนี้แทนได้

ปุ่ม Connect:

```ini
[Desktop Entry]
Type=Application
Name=VPN Connect
Comment=Start Fortinet SSL VPN service
Exec=lxterminal -e bash -lc "sudo systemctl start fortivpn.service; systemctl status fortivpn.service --no-pager; echo; read -p 'Press Enter to close...'"
Icon=network-vpn
Terminal=false
Categories=Network;
```

ปุ่ม Disconnect:

```ini
[Desktop Entry]
Type=Application
Name=VPN Disconnect
Comment=Stop Fortinet SSL VPN service
Exec=lxterminal -e bash -lc "sudo systemctl stop fortivpn.service; echo 'VPN stopped.'; echo; read -p 'Press Enter to close...'"
Icon=network-vpn
Terminal=false
Categories=Network;
```

## ทำให้กดปุ่มแล้วไม่ต้องถาม sudo password

ถ้าต้องการให้ user ปัจจุบันสั่งเปิด/ปิด service ได้โดยไม่ต้องกรอก sudo password ให้แก้ sudoers อย่างระวัง

ดูชื่อ user:

```bash
whoami
```

เปิดไฟล์ sudoers เฉพาะ VPN:

```bash
sudo visudo -f /etc/sudoers.d/fortivpn
```

ใส่บรรทัดนี้ โดยเปลี่ยน `pi` เป็นชื่อ user ของคุณ:

```text
pi ALL=(root) NOPASSWD: /bin/systemctl start fortivpn.service, /bin/systemctl stop fortivpn.service, /bin/systemctl status fortivpn.service
```

ตรวจสอบ path ของ `systemctl` ถ้าไม่แน่ใจ:

```bash
command -v systemctl
```

ถ้าได้ `/usr/bin/systemctl` ให้ใช้ path นั้นแทน `/bin/systemctl` ใน sudoers

## ใช้งานกับ SAML/SSO หรือ OTP

ถ้า VPN ใช้ SAML/SSO ให้ลองเชื่อมต่อแบบนี้:

```bash
sudo openfortivpn -c /etc/openfortivpn/company.conf --saml-login
```

บางองค์กรใช้หน้า login ที่ต้องเปิด browser หรือมี JavaScript ซับซ้อน ถ้า `openfortivpn` ไม่สามารถ login ได้ ให้สอบถามผู้ดูแลระบบว่ารองรับ SSL VPN แบบ username/password/OTP หรือมีวิธีสำหรับ Linux ARM หรือไม่

## Troubleshooting

ถ้าเชื่อมต่อไม่ได้ ให้ลองตรวจสอบตามนี้

ถ้าเจอ error นี้:

```text
ERROR:  getaddrinfo: Name or service not known
```

ให้ตรวจสอบว่า `host` ไม่มี `https://`, ไม่มี port, และไม่มี path เช่น `/vpn`

ตัวอย่างที่ผิด:

```ini
host = https://ip-address:10443/vpn
port = 10443
```

ตัวอย่างที่ถูก:

```ini
host = ip-address
port = 10443
realm = vpn
```

ถ้าแก้ config แล้วขึ้น `connect: Connection refused` ให้ตรวจสอบว่า IP/port ถูกต้องและ FortiGate เปิด SSL VPN port นั้นจริง:

```bash
curl -vk https://ip-address:10443/vpn
```

### `Gateway certificate validation failed`

ถ้าเจอ error ลักษณะนี้:

```text
ERROR:  Gateway certificate validation failed, and the certificate digest is not in the local whitelist.
ERROR:  or add this line to your configuration file:
ERROR:      trusted-cert = aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

สาเหตุคือ certificate ของ FortiGate ยังไม่ได้อยู่ใน local whitelist ของ `openfortivpn` วิธีแก้คือเพิ่ม `trusted-cert` ลงใน `/etc/openfortivpn/company.conf`

เปิดไฟล์ config:

```bash
sudo vim /etc/openfortivpn/company.conf
```

เพิ่มบรรทัดนี้ โดยใช้ค่า digest ที่เครื่องของคุณแสดง:

```ini
trusted-cert = aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

หรือทดสอบแบบชั่วคราวโดยส่งค่า digest ผ่าน command line:

```bash
sudo openfortivpn -c /etc/openfortivpn/company.conf --trusted-cert aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

ค่า digest ในตัวอย่างเป็นค่า mock ให้เปลี่ยนเป็นค่าจริงที่โปรแกรมแสดงหลังคำว่า `trusted-cert =`

เมื่อใส่ `trusted-cert` ใน config แล้ว คำสั่ง `vpn-up`, ปุ่ม Desktop, และ `fortivpn.service` จะใช้ค่า certificate นี้อัตโนมัติ

ตรวจสอบ log:

```bash
journalctl -u fortivpn.service -n 100 --no-pager
```

ตรวจสอบว่ามี process VPN หรือไม่:

```bash
pgrep -a openfortivpn
```

ตรวจสอบ interface:

```bash
ip addr show ppp0
```

ถ้าเจอปัญหา PPP module:

```bash
sudo modprobe ppp_generic
sudo modprobe ppp_async
```

ถ้าต้องการโหลด module ทุกครั้งตอน boot:

```bash
printf "ppp_generic\nppp_async\n" | sudo tee /etc/modules-load.d/ppp.conf
```

ถ้าเชื่อมต่อแล้วเข้าเว็บภายในไม่ได้:

```bash
ip route
cat /etc/resolv.conf
```

ตรวจสอบกับผู้ดูแลระบบว่า VPN ต้อง push route/DNS แบบใด และมี split tunnel หรือไม่

## ถอนการติดตั้ง

หยุด service:

```bash
sudo systemctl disable --now fortivpn.service
```

ลบ service และสคริปต์:

```bash
sudo rm -f /etc/systemd/system/fortivpn.service
sudo rm -f /usr/local/bin/vpn-up /usr/local/bin/vpn-down
sudo systemctl daemon-reload
```

ลบ package:

```bash
sudo apt remove -y openfortivpn ppp
```

ลบ config เฉพาะเมื่อแน่ใจว่าไม่ต้องใช้แล้ว:

```bash
sudo rm -rf /etc/openfortivpn
```

## อ้างอิง

- `openfortivpn`: <https://github.com/adrienverge/openfortivpn>

## เครดิต

เอกสารนี้สร้างโดย Codex ใช้โมเดล GPT-5
