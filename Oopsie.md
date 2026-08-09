
Oopsie

- Platform: Hack The Box
- Difficulty: Easy
- Operating System: Linux

Objective

Oopsie highlights the impact of information disclosure and broken access control in a web application. Enumerating the website reveals a guest login, later manipulating cookies and user IDs allows escalation to an admin role in the website and access to a file upload feature. A uploaded reverse shell is used to gain initial foothold, further enumeration reveals mysqli configuration file with exposed credentials that enable lateral movement to another user. Finally, a misconfigured SUID binary is used to achieve privilege escalation as root.

---

Attack Path

Service Enumeration -> Website Enumeration -> Login as Guest -> User ID and Cookie manipulation -> Admin role -> Upload feature -> Reverse shell uploaded -> /var/www/ file with exposed credentials -> Lateral Movement -> Misconfigured SUID Binary -> Privilege escalation as root.
 
---

Methodology

1. Initial Enumeration
2. Service Enumeration
3. Initial Access
4. Privilege Escalation
5. Lessons Learned
6. Final Thoughts

---

1. Initial Enumeration

Port Enumeration

sudo nmap --top-ports 1000 $target -v 

	22/tcp open  ssh
	80/tcp open  http


---

2. Service Enumeration

Port Scan

sudo nmap -sV -p 22,80 $target -v

	22/tcp OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
	80/tcp Apache httpd 2.4.29 ((Ubuntu))

---


HTTP

firefox $target:80 &

- Website Source Code
- Login as Guest
- User ID and Cookie Manipulation
- Upload feature

The page shows MegaCorp Automotive's website, it presents briefly about MegaCorp's Vehicles.
At the bottom of the page, the 'Service' section says to login to get access to their services, but there's no 'Login' button on the page. 
Analyzing the source code I found a script referencing to '/cdn-cgi/login', revealing the application's login page. When opening this URL, it presents the Login page and interestingly have a 'Login as Guest' option,  I proceed with that option.

---

3. Initial Access

Now I'm at 'admin.php', the page shows options such as Account, Branding, Clients and Uploads, the last requires super admin rights to access.

The 'Account' section sets the URL to '.../admin.php/content=accounts&id=2' which immediately suggests an IDOR vulnerability. Other than that, it shows my Access ID - 2233, Name - Guest, and Email - guest@megacorp.com.
Changing the id to 1 in the URL grants me access to the admin 'Account' info, his Access ID is 34322. I tried to access the 'Uploads' section, but it still blocked. 
I suspected that 'Access ID' would be the cookie session, so I opened the Dev Tools and noticed that in fact, my cookies as 'Guest' are '2233', so I changed it to '34322' and successfully gained access as administrator.

The 'Uploads' section now presents an upload functionality, I was able to select a file and name it to upload. I created a 'test.txt' file with 'testing' written inside and uploaded that to see the output, but it simply returned 'File uploaded successfully'. I manually searched across the URL and discovered that http://$target/uploads/ returns '403 Forbiden', the directory itself isn't exposed, but '/uploads/test.txt' does access the file and shows 'testing'. 
The next thing that I did was to generate a reverse shell in revshells.com and uploaded it. After setting up a Netcat listener and opening '/uploads/shell.php' I successfully gained access to a shell as 'www-data'.

Exploring the machine I found the user's flag in '/home/robert'.

---

4. Privilege Escalation

Since there's a web service, I always explore the /var/www/ directory, and in /var/www/html/cdn-cgi/login/ I found the 'db.php' file, which is robert's configuration file to access mysqli. I noted them and successfully gained SHH access as robert.
I started to explore robert's privileges, I discovered that he cannot use sudo but he's part of the 'bugtracker' group.

Running 'find / -group bugtracker -ls 2>/dev/null' to find any accessible files that have the bugtracker group, I found '/usr/bin/bugtracker' and the permissions '`-rwsr-xr--`'. As robert I can read the file and most important I can execute with the owner (root) permissions since the file has the SUID bit.
Since it's a binary, I used 'strings /usr/bin/bugtracker' to see visible strings and to have a general idea about what this binary does.
Exploring the output I saw an interesting part that follows: 

	------------------
	: EV Bug Tracker :
	------------------
	Provide Bug ID: 
	---------------
	cat /root/reports/

It seems that the binary expects a 'Bug ID' and uses 'cat' in /root/reports/ to show information about this bug.
Running the binary and inserting '1' as ID, the output shows information about a registered bug. The binary seems to not sanitize the user's input, inserting 'a' as ID triggers an error 'cat /root/reports/a no such file or directory'. So I tried to insert '../root.txt' and successfully got the root's flag. I also tried to input '../../etc/shadow' and got the root password hash, but I failed to crack it, I assume the room was intended to stop at retrieving the root flag through the vulnerable binary.

---

5. Lessons Learned

- Hidden functionality can often be discovered by inspecting client-side source code.
- IDOR vulnerabilities should always be tested whenever the object identifiers are exposed.
- Session cookies should never be trusted as the sole authorization mechanism.
- File upload functionality must strictly validate file types and execution permissions.
- SUID binaries should be inspected for insecure input handling and path transversal vulnerabilities.



---

6.  Final Thoughts

Oopsie demonstrates how multiple Broken Access Control vulnerabilities can be chained to fully compromise a system. An IDOR vulnerability exposes privileged account information, while the application improperly trusts a client-controlled session identifier, allowing an attacker to impersonate an administrator simply by modifying a cookie value. This access enables the upload of a malicious PHP file, resulting in remote code execution. Finally, an insecure SUID binary vulnerable to path traversal allows sensitive files owned by root to be accessed, completing the compromise.