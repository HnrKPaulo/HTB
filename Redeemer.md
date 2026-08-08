
Redeemer

- Platform: Hack The Box
- Difficulty: Easy
- Operating System: Linux

Objective

Redeemer explores the numeration and exploitation of a Redis database server with no protection against unauthorized access.

---

Attack Path

Service Enumeration -> Expanded Service Enumeration -> Redis database server -> Access without authentication protocol -> Recovery of sensitive data.

---

Methodology

1. Initial Enumeration
2. Service Enumeration
3. Exploitation
4. Lessons Learned
5. Final Thoughts

---

1. Initial Enumeration

Port Enumeration

sudo nmap --top-ports 1000 $target -v

	All 1000 scanned ports on $target are in ignored states

Since no port was found open, I expanded the scan to the 10.000 most common ports.

sudo nmap --top-ports 10000 $target -v

	6379/tcp open  redis

---

2. Service Enumeration

Port Scan

sudo nmap -sV -p 6379 $target -v

	6379/tcp Redis key-value store 5.0.7

---

3. Exploitation

REDIS

Accordingly to my research, Redis is an Open Source, RAM stored database system, it is commonly used for cache and quick operations such as webpage cache, login session and so on. It stores key-value data, these keys can store strings, hashes, lists, sets and sorted sets.
I downloaded the 'redis-cli' application and since I wasn't familiar with Redis, I used AI as a learning aid to understand the redis-cli syntax.

The process follows as:

- redis-cli -h $target
The service is insecure and no authentication process is required, it immediately allows to run commands.

- info
The output of this command shows general information and statistics about the server, some are clients connections, memory and cpu consumption, and keyspace.

The keyspace section shows database-related statistics, and reveals only one database 'db0' with 4 keys.

- Select 0
The 'Select' command requires the index of the desired database.

- Keys *
This command returns all of the keys registered on the selected database.

The output shows:

1) flag
2) numb
3) temp
4) stor

- get flag

The 'get' command requires the name of the desired key, not the index.

After running the command, the room's flag is revealed.

---

4. Lessons Learned

Sometimes a higher number of ports must be scanned. In this case, no port were found open in the top 1000 most common scan and it was necessary to increase to the top 10.000 most common. However, in a scenario were the top 1000 ports returned open ports, but no progress is being made, a extended scan could help to enumerate other ports.
AI can be a very helpful tool to accelerate the process to understand and learn about a unfamiliar tool or application, it should be used as a complementary aid instead of doing all the work. 

---

5.  Final Thoughts

Redeemer is a different room, it doesn't features a Linux, Windows or Web application to be explored, instead it demonstrates the importance of explore less common ports and to secure the service from unauthorized access, preventing damage or disclosure of sensitive data.