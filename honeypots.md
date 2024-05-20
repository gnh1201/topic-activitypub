# Honeypots

Operating honeypots on an ActivityPub server can help reduce the generation of unnecessary logs and thereby decrease the load on the system. Additionally, it can make attackers waste their efforts, giving administrators more time to protect the actual system. Here, we introduce honeypot projects that have been confirmed to be applicable to real servers.

## Useful honeypot projects

* (SSH/22) [droberson/ssh-honeypot](https://github.com/droberson/ssh-honeypot) - written in C/C++, [Modified version available](https://github.com/gnh1201/ssh-honeypot)
* (Telnet/23) [robertdavidgraham/telnetlogger](https://github.com/robertdavidgraham/telnetlogger) - written in C/C++, [Modified version available](https://github.com/gnh1201/telnetlogger)
* (FTP/21) [farinap5/FTPHoney](https://github.com/droberson/ssh-honeypot) - written in Go
* (SMTP/25, Encrypted/465,587) [decke/smtprelay](https://github.com/decke/smtprelay) - written in Go
* (MySQLd/3306) [sjinks/mysql-honeypotd](https://github.com/sjinks/mysql-honeypotd) - written in C/C++
* (PostgreSQL/5432) [betheroot/pghoney](https://github.com/betheroot/pghoney)  - written in Go, [Modified version available](https://github.com/gnh1201/pghoney)

## Contact me
* ActivityPub [@gnh1201@catswords.social](https://catswords.social/@gnh1201)
* abuse@catswords.net
