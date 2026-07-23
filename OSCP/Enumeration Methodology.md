### Initial Scan [[Essential Tools#Enumeration|Tìm kiếm nmap]]
[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#initial-scan)
1. [x]  Full port scan: `nmap -p- --min-rate 1000 <target>` 
2. [x]  Service scan on open ports: `nmap -sC -sV -p <ports> <target>`
3. [ ]  Check for low-hanging fruit: anonymous FTP, SMB null sessions, default creds
---
### Web (80/443) [[Essential Tools#Web|Web tools]]
[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#web-80443)
1. [ ]  Browse manually first and check source code
2. [ ]  Directory bust with feroxbuster/gobuster
3. [ ]  Check for CMS (WordPress, Drupal, etc.) and run specific scanners
4. [ ]  Test for SQLi, LFI, file upload vulns
5. [ ]  Check `/robots.txt`, `/.git/`, /backup/, and /api
---
### SMB (139/445) [[Essential Tools#SMB|SMB Tools]] 
[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#smb-139445)
1. [ ]  Null session: `netexec smb <target> -u '' -p ''`
2. [ ]  List shares: `smbclient -L //<target>/ -N`
3. [ ]  Check for read/write access on shares
4. [ ]  Enum users: `enum4linux -a <target>`
---
### Active Directory [[Essential Tools#Impacket Suite (Know These)|Impacket]] [[Essential Tools#LDAP & RPC|LDAP]] [[Essential Tools#RDP & winrm (you will see this.)|RDP]]  
[](https://github.com/jeffaf/oscp-prep-checklist?utm_source=chatgpt.com#active-directory)
1. [ ]  Get domain info: `ldapsearch` or `enum4linux`
2. [ ]  Find users: kerbrute [https://github.com/ropnop/kerbrute](https://github.com/ropnop/kerbrute)
3. [ ]  Find SPNs for Kerberoasting
4. [ ]  Check for AS-REP roastable users
5. [ ]  BloodHound if you have creds
---
# Tips
- **21: FTP** (anonymous, brute)
- **22: SSH** (libssh, bypass auth, brute)
- **445: SMB** (smbclient, smbmap)
- **3389: RDP** (bluekeep)
- **111: RPCinfo** (mount nfshare)
- **80/443: Web**
- **Port lạ:** 6543, 1234