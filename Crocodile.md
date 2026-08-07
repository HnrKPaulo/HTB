
CROCODILE

- Platform: Hack The Box
- Difficulty: Easy
- Operating System: Linux

Objective

An exposed FTP service reveals sensitive data, which can be used to access an administrative panel of a website.

---

Attack Path

Service Enumeration -> Anonymous FTP authentication -> Access to plaintext credentials -> Website's administrative panel access.
 
---

Methodology

1. Initial Enumeration
2. Service Enumeration
3. Initial Access
4. Final Thoughts

---

1. Initial Enumeration

Port Enumeration

sudo nmap --top-ports $target -v

	21/tcp open  ftp
	80/tcp open  http


---

2. Service Enumeration

Port Scan

sudo nmap -sV -p 21,80 $target -v

	21/tcp  vsftpd 3.0.3
	80/tcp Apache httpd 2.4.41 ((Ubuntu))

---


HTTP

firefox $target:80 &

- Potential users for authentication
- Form
- Exposed Directories

It's a web page for an unnamed company who offers digital solutions such as Graphics and Website Design, and Digital Marketing.

Exploring the page I found the 'team' section, that presents:

- Jeffery Riley - Art Director
- Riley Beata - Web Developer
- Mark A. Parker - UX Designer

I noted their names to try as possible usernames in an authentication process.

At the bottom of the page I also found the 'contact' section with a form. I tried a basic XSS payload, but the form appears to be non-functional.

Analyzing the source code of the page, I found the /assets directory, which is exposed. Alongside the expected css, fonts, images and js directories, I also found the 'contact.php' file. When accessed, it displays 'There was a problem with your submission, please try again'. Just to confirm that the form isn't a vector.

At this point I know that there are exposed directories, so I ran Gobuster, and meanwhile it is working I looked at the ftp service.

	 gobuster dir -u http://$target/ -w /usr/share/dirbuster/wordlists/directory-list-lowercase-2.3-medium.txt -x php,sh,txt,cgi,html,js,css,py 

FTP

ftp $target

I successfully logged-in as 'anonymous' and found 2 archives in there: 'allowed.userlist' and 'allowed.userlist.passwd' those files contains 4 users and their respectives passwords, I combined them on a single file 'creds.txt'.
The users are aaron,pwnmeow,egotisticalsw and admin.

Back at gobuster, the output shows some results, including the 'login.php' and 'config.php'

---

3. Initial Access

At the login page I also analyzed the source code in case there's a hardcoded login logic, but it was not the case. Then tried the 'admin' credential and successfully gained access to the dashboard, the room's flag is at the top of the site. 

---

4.  Final Thoughts

Crocodile is a very easy machine that demonstrates how an exposed FTP service storing sensitive information can quickly lead to unauthorized access. Although the compromise ends at a web administration panel in this room, in a real-world environment such access could allow an attacker to modify website content, steal additional information, or leverage the compromised account to expand their foothold within the organization. The room also reinforces the importance of disabling anonymous FTP access and avoiding the storage of plaintext credentials on publicly accessible services.
