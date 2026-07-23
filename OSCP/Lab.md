# 01
## service
- 22 ssh: ssh
- 80 http: apache2
## Công nghệ
- apache 
- Adminer 5.4.2
-  WordPress version 7.0
## Flow
- login db: http://192.168.88.155/db/
- nikto scan: nikto -h http://{{target_ip}}/blog
- hoặc dir scan extension **.php.bak, bak** => user, password
- SSH hoặc làm theo cách đăng nhập vào CMS
---
# 02
## Flow
- source có đường dẫn của aws
- Truy cập theo đường link bucket
	- Tìm được file notice của admin
	- Brute Force ssh
		- `hydra -l ubuntu -P /usr/share/wordlists/rockyou.txt ssh://192.168.1.50`
	- ssh vào hệ thống
- sudo -l
	- https://gtfobins.org/gtfobins/tar/#shell
---
# 03
## Info
- CTF Download Portal
	- Files:6•Directory:/opt/ctf_downloads
- Crack zip
	- `frackzip -D -p [wordlist] -u file.zip`
- Tìm format của user pattern kèm password file
- Sử dụng **pdfinfo** hoặc **exiftool** để extract thông tin file
- ssh bằng tk và mk đã có (hoặc brute force)
- Quay trở về root folder của thư mục */opt/ctf_downloads* => tìm thấy file db keypass
	- Đăng nhập `kpcli --kdb=[tên_file_db]` local với mk 123456
```bash
Brute force keepass
keepass2john MyDatabase.kdbx > john_hash.txt

john --wordlist=rockyou.txt john_hash.txt
```
-  Câu lệnh keypass
	- ls => cd
	- show 1 => show -f 1
	