
- Platform: Hack The Box
- Difficulty: Introductory
- Operating System: Linux

Objective

Basic SQL Injection attack in a web application.

---

Methodology

1. Initial Enumeration
2. Service Enumeration
3. Vulnerability Research & Access
4. Lessons Learned
5. Final Thoughts
---

1. Initial Enumeration

Port Enumeration

sudo nmap -Pn -Ss --top-ports 1000 $target

- 80/tcp open  http

2. Service Enumeration

Port Scan

sudo nmap -Pn -sV -p 80 $target
- Apache httpd 2.4.38 ((Debian))

HTTP

- Page Source Code analysis
- Interesting directories
- SQL Injection

3. Vulnerability Research & Access

- firefox $target:80 &

It's a login page. 
The first thing that I did was to analise the page's source code, there I found the '/vendor' directory, which is exposed and contains other directories, but they were dead ends. While I manually searched for exposed directories, I also ran Gobuster.

Back at the login form, I tried default combinations of credentials like admin:admin, root:root and so on, but none worked, so I performed a SQL Injection admin:' OR 1=1 -- - and it worked. The page simply show the room's flag.

4. Lessons Learned

Out of curiosity, I read the official report and learned that admin'#:anypassword would work too. In this way, we transform the query by turning in a comment the password check section. The query checks if user = 'admin', returns true and proceeds with the login.

Final Thoughts

This machine despite being introductory to SQL Injection, reinforced the impact and facility of that attack. Injection is the #5 most critical security risk in the OWASP Top10:2025 and to prevent an application from becoming vulnerable to it, its to keep data separate from commands and queries.