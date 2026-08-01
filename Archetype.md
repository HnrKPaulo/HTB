
Machine Name

- Platform: Hack The Box
- Difficulty: Introductory
- Operating System: Windows

Objective

Machine with a misconfigured Microsoft SQL server, exposed SMB shares and sensitive data, leading to initial access through MSSQL and privilege escalation to SYSTEM.

---

Attack Path

SMB enumeration ->
Exposed backup share ->
Credentials found ->
MSSQL Authentication ->
Sysadmin privileges ->
xp_cmdshell enabled ->
User access ->
Administrator credentials from history file ->
SYSTEM Access.

---

1. Initial Enumeration
2. Service Enumeration
3. Initial Access
4. Privilege Escalation
5. Lessons Learned
6. Final Thoughts


---

1. Initial Enumeration

Port Enumeration

sudo nmap -Pn -sS --top-ports 1000 $target -v
- Obs: During the initial scan, Nmap reported multiple dropped probes and increased the send delay. The cause could be related to network latency, packet loss, or filtering mechanisms. To improve the scan stability, I reduced the timing template to -T2, resulting in less dropped probes and more consistent scan results.

sudo nmap -T2 -Pn -sS --top-ports 1000 $target -v

	135/tcp  open  msrpc
	139/tcp  open  netbios-ssn
	445/tcp  open  microsoft-ds
	1433/tcp open  ms-sql-s
	5985/tcp open  wsman

2. Service Enumeration

Port Scan

sudo nmap -T2 -sV -p 135,139,445,1433,5985 $target -v

	135/tcp  Microsoft Windows RPC
	139/tcp  Microsoft Windows netbios-ssn
	445/tcp  Microsoft Windows Server 2008 R2 - 2012 microsoft-ds
	1433/tcp Microsoft SQL Server 2017 14.00.1000
	5985/tcp Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)

3. Initial Access

HTTP

- Error 404

firefox $target:5985 &

SMB

- 445
- Exposed shares
- Sensitive information disclosure

smbclient -L //$target
password = ''

ADMIN$     Disk      Remote Admin
backups      Disk      
C$                 Disk      Default share
PC$               IPC       Remote IPC

smbclient -U '' //$target/backups/
password= ''

I discovered a file named 'prod.dtsConfig' containing credentials for the sql_svc user. With that credentials I tried to use them to access more shares, but it returned 'Access Denied', then I aimed at the RPC and MSSQL services.

RPC

- evil-winrm
- xfreerdp
- All dead ends

MSSQL

- Impacket
- Privilleges
- Data Manipulation
- Inicial access

Impacket is a collection of tools to work with network protocols, especially those in windows ecosystems. 

- python3 /usr/share/doc/python3-impacket/examples/mssqlclient.py 'user':'passw'@'$target' -windows-auth

After taking a look at the help menu, the 'enum_logins' command revealed that the current login belongs to the 'sysadmin' server role.
Another interesting command that I found was 'xp_cmdshell', but when I tried it, an error informed that this component is turned off by default on the server, and the sysadmin can enable the use 'xp_cmdshell' by using 'sp_configure'.
I spent some time figuring out the correct syntax and I successfully enabled 'xp_cmdshell' and achieved OS command execution through MSSQL. 
I explored the machine and found the user flag in the user desktop.


4. Privilege Escalation

With 'whoami /priv' output I discovered that the user has SeImpersonatePrivilege Enabled, that's a potential escalation vector. After investigating this privilege and attempting to leverage PrintSpoofer, I was unable to successfully reproduce the technique and looked up at the oficial report.
The report revealed that the Console_History.txt contains the administrator credentials, and also used winPEAS to enumerate potential privilege escalation vectors, including the SeImpersonatePrivilege.
Using the administrator credentials, I obtained a SYSTEM shell through Impacket's psexec.py. The flag was located on the Administrator Desktop.

---

5. Lessons Learned

Learned MSSQL usage and the importance of checking local artifacts such a history files. Methodologically I also learned to avoid focusing too long on a single path without validating other possible vectors, is important to enumerate thoroughly before attempting exploitation.

---

6. Final Thoughts

Archetype was a great challenge, it demonstrates how an exposed share containing credentials can lead to a complete compromise of the host. 
