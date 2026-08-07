
CAP

- Platform: Hack The Box
- Difficulty: Easy
- Operating System: Linux

Objective

Cap presents a machine running an HTTP server that performs administrative functions such as network capture. Through an IDOR vulnerability, it is possible to access another user's network capture, recover valid credentials, obtain initial access via SSH, and leverage a Linux capability assigned to Python to escalate privileges to root.

---

Attack Path

Service Enumeration -> http dashboard -> IDOR -> .cap file -> Credential obtained -> SSH -> python3 capability identified -> Privilege Escalation to root.

---

Methodology

1. Initial Enumeration
2. Service Enumeration
3. Initial Access
4. Privilege Escalation
5. Lessons Learned
6. Oficial Report
7. Final Thoughts

---

1. Initial Enumeration

Port Enumeration

sudo nmap --top-ports-1000 $target -v

	21/tcp open
	22/tcp open
	80/tcp open

--- 

2. Service Enumeration

Port Scan
sudo nmap -sV -p 21,22,80 $target -v

	21 ftp vsftpd 3.0.3
	22 ssh OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
	80 http Gunicorn

HTTP 

- firefox $target:80 &
- .cap files
- IDOR

The page presents a dashboard that contains several sections related to network administration, including Security Snapshot, IP Config and Network Status, also I'm already logged as 'Nathan'. Firstly I inspected the page source code searching for possible plain text credentials and directories that could be exposed, but couldn't find any. As I'm logged as Nathan, I used dev tools to look at session cookies, but no cookies were present.
Now exploring the dashboard itself, on the left side there's a menu with Security Snapshots, IP Config and Network Status.

Network Status: http://$target/netstat

The page shows the output of the netstat command, I couldn't find any interesting info here.

IP Config: http://$target/ip

Shows the output of ifconfig command.

Security Snapshot: http://$target/data/1

Shows information about a network capture. It presents Number of Packets, Number of IP Packets, Number of TCP Packets and Number of UDP Packets, all counters were equal to zero, and at the bottom of the page there's a 'Download' button, which downloads the .cap file. I downloaded it and opened the file with wireshark to take a better look and as expected there's no packets. Back to the page, I changed the URL, from data/1 to data/2, and now I have some data to analyze. The file only contains 7 packages, and they're about my connection with the service, nothing that I can work with.
I repeated the process with /data/3, /data/4, and /data/5 but I didn't succeed to get any results, so I decided to explore the FTP service.

FTP

- ftp $target
- Requires user and password

It's necessary to insert User and Password, I tried to authenticate as 'Nathan, 'guest' and 'anonymous' without a password, and also a few default credentials such as admin:admin, root:toor and so on, but all authentication attempts failed.

At this point, I assumed that the intended path was be to obtain credentials from the HTTP application and use them to authenticate in the FTP service. I ran Gobuster to see if I can find a exposed directory or file.

- gobuster dir -u http://$target/ -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -x php,sh,txt,cgi,html,js,css,py

While Gobuster was running, I reviewed mu previous findings and noticed that I missed to change /data/1 url to data/0, so I tried it and successfully had a result, now there's 72 packets, I downloaded the file and analyzed it on wireshark.

WIRESHARK

After opening the file I immediately spotted a request to the ftp service as 'user Nathan' and a successful login response. Following the TCP stream I got Nathan's credentials. nathan:Buck3tH4TF0RM3!

---

3. Initial Access

Back at the FTP, I used the obtained credential to successfully authenticate, exploring the service I was able to get the user flag.
Now that I have a credential and also a SSH service opened, I tried to SSH as Nathan and it worked.

---

4. Privilege Escalation

- Permissions & Group
- app.py
- python3

With this first access I started searching for possible vectors to escalate privileges, unfortunately nathan cannot use sudo neither is member of a group that could be a vector. Since there's a HTTP server, I searched up into /var/www/html and in there I found the app.py file. This file have an interesting part:

	<snipet>
		path = os.path.join(app.root_path, "upload", str(pcapid) + ".pcap")
		ip = request.remote_addr
	# permissions issues with gunicorn and threads. hacky solution for now.
	#os.setuid(0)
	#command = f"timeout 5 tcpdump -w {path} -i any host {ip}"
		command = f"""python3 -c 'import os; os.setuid(0); os.system("timeout 5 tcpdump -w {path} -i any host {ip}")'"""
		os.system(command)
	<snipet>

At this point, I hypothesized that if nathan is allowed to run 'setuid(0)' and he's the owner of the file, I should be able to insert a reverse shell into this file and gain access as a root. (Correction: The Python interpreter have permission to call 'os.setuid(0)', not nathan, not the app.py file.). So I started working on it, I was able to insert the code, but I failed to find where or how to restart the application and trigger the reverse shell. Based on the same hypothesis to run 'setuid(0)' on this machine, I tried to open python3 interpreter and manually execute the required commands to spawn a shell as a root. For this part I had to look up online the correct syntax.

	> python3 
	>>> import os
	>>> os.setuid(0)
	>>> os.system("/bin/bash")

With that I successfully got a shell as root and found the root flag in the /root directory.

---

5. Lessons Learned

The room reinforced the importance of validating assumptions during privilege escalation. Instead of immediately relying on automated enumeration tools, manually reviewing the application source code led me to the same conclusion reached by linPEAS: the Python interpreter had the CAP_SETUID capability. It also highlighted how an application that appears harmless at first can expose valuable information that enables further compromise. 

---

6. Oficial Report

In the oficial report, linPEAS is used to enumerate possible vulnerabilities on the machine, and specially the 'files with capabilities' section is explored, which reveals that python3.8 has the 'cap_setuid' capability. It is interesting that I used another method and was able to infer the same result. Even though I already know linPEAS, I chose to enumerate manually first, to prioritize discretion, avoiding noise and excess activity in the system.


---

7. Final Thoughts 

Cap is a really interesting and challenging room. It demonstrates Broken Access Control, ranked as the #1 category in the OWASP Top 10:2025. Through an IDOR vulnerability, it is possible to access another user's network capture, which contains an FTP authentication exchange. The recovered credential provide initial access to the machine. Although the user initially appears to have no interesting privileges, a Linux capability assigned to Python ultimately allows privilege escalation to root. 

