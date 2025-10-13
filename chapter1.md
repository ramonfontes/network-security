<a name="environment-setup-and-basic-concepts"></a>
# Environment Preparation and Basic Concepts

<a name="environment-setup"></a>
## Preparing the Environment

### Introduction
The exercises proposed in this section demonstrate that it is possible to carry out security attacks using freely available software and a basic understanding of networking and security. These types of attacks can be very harmful in certain contexts.

### Basic Requirements

- **Ubuntu 22.04+**
- **Containernet** — Follow the installation instructions at: https://github.com/ramonfontes/containernet#installation

Pull the images listed below using the respective commands:

```bash
$ sudo docker pull ramonfontes/seguranca
$ sudo docker pull ramonfontes/openvas
$ sudo docker pull ramonfontes/xss_attack
$ sudo docker pull ramonfontes/vulnerable
$ sudo docker pull ramonfontes/proxy
$ sudo docker pull ramonfontes/rogue-ap
```

The image `ramonfontes/seguranca` contains the following packages:
iptables, nmap, hping3, apache2, dsniff, ettercap-text-only, arpon, curl, sudo, nano, firefox, telnet, openssh-server, ethtool, iproute2, iputils-ping, net-tools, wireshark, tcpdump, aircrack-ng, iperf, gnupg, pciutils, wpasupplicant, snort, metasploit-framework, python3-scapy, netcat, busybox, sshuttle.

For exercises that require authentication, consider both the username and the password to be the same word found in /john/run/passwd.txt. To find the username and password, run the command below from a container of the `ramonfontes/seguranca` image:

````commandline
./john passwd.txt --format=raw-sha1
````

The version of Containernet available in the repository above is compatible with Mininet-WiFi (https://github.com/intrig-unicamp/mininet-wifi), the WiFi network emulator developed by your professor. If you want to learn more about Mininet-WiFi, you can access the virtual book at https://github.com/ramonfontes/mn-wifi-ebook or obtain a printed copy at mininet-wifi.github.io/book. Alternatively, for a quick overview of the Mininet-WiFi emulator, you may want to watch the video available at: https://www.youtube.com/watch?v=2SdRbGOYnlU.

### Troubleshooting

This section is intended to point out common issues and how to fix them. It will be populated as problems are encountered.

- *Problem*: The xterm command issued from the Containernet console produces no output.
- *Possible fix*: You may need to set the DISPLAY environment variable to 0 ou 1 in the codes available in this document.


- *Problem*: When accessing via SSH, you get the following error: X11 connection rejected because of wrong authentication
-  *Possible fix*: Running the following command may resolve this issue: `sudo xauth merge ~/.Xauthority`

<a name="system-auditing"></a>
### System Auditing

One of the keys to protecting a system is knowing what’s happening inside it — which files are being modified, who is accessing what and when, and which applications are being executed. Incrond used to be one of the most popular auditing tools for Linux systems a few years ago. Nowadays, however, your best option for monitoring all system activities is likely auditd.

One major advantage of auditd is that it performs its checks at the kernel level, below user space, which makes it much harder to tamper with or bypass. This provides a significant edge over shell-based auditing systems, which can’t be fully trusted if the system has already been compromised before they start running.

Auditd is actively developed by Red Hat and is available for most, if not all, major Linux distributions. If it’s not yet installed on your system, you can easily find it in your distribution’s repositories.

The auditd suite consists of several components, but for our purposes, the main ones are:

- auditd, the daemon responsible for monitoring system activity; and
- aureport, a tool that generates detailed reports from auditd logs.

#### Installation

Install the auditd package using your distribution’s package manager, then verify that it’s running. Most modern Linux distributions run auditd as a systemd service, so you can check its status with:

```commandline
systemctl status auditd.service
```

If it’s installed but not running, start it with:

```commandline
systemctl start auditd.service
```

To enable it to run automatically at system startup, use:
```commandline
systemctl enable auditd.service
```

*Note*: Before checking any reports, let auditd run for a while so it can populate its logs with meaningful events.

```
sudo aureport

Summary Report
======================
Range of time in logs: 20/11/2023 21:39:28.208 - 21/11/2023 09:32:18.241
Selected time for report: 20/11/2023 21:39:28 - 21/11/2023 09:32:18.241
Number of changes in configuration: 98
Number of changes to accounts, groups, or roles: 0
Number of logins: 0
Number of failed logins: 0
Number of authentications: 8
Number of failed authentications: 0
Number of users: 5
Number of terminals: 10
Number of host names: 2
Number of executables: 23
Number of commands: 22
Number of files: 102
Number of AVC's: 0
Number of MAC events: 0
Number of failed syscalls: 32
Number of anomaly events: 9
Number of responses to anomaly events: 0
Number of crypto events: 0
Number of integrity events: 0
Number of virt events: 0
Number of keys: 2
Number of process IDs: 180
Number of events: 1648
```

That’s already interesting! Take a look at the line that says “Number of failed authentications: 0.”
If you ever see a high number here, it could indicate that someone is trying to break into your system by brute-forcing a user’s password.

Let’s dig a little deeper:

```
sudo aureport -au

Authentication Report
============================================
# date time acct host term exe success event
============================================
1. 21/11/2023 07:51:04 gdm alpha-Inspiron-5480 /dev/tty1 /usr/libexec/gdm-session-worker yes 150
2. 21/11/2023 07:51:20 alpha alpha-Inspiron-5480 /dev/tty1 /usr/libexec/gdm-session-worker yes 206
3. 21/11/2023 07:51:36 mysql ? ? /usr/bin/su yes 321
4. 21/11/2023 07:56:03 nobody ? ? /usr/bin/su yes 366
5. 21/11/2023 07:56:04 nobody ? ? /usr/bin/su yes 378
6. 21/11/2023 07:56:04 nobody ? ? /usr/bin/su yes 384
7. 21/11/2023 09:01:35 alpha ? /dev/pts/0 /usr/bin/sudo yes 498
8. 21/11/2023 09:29:05 alpha ? /dev/pts/0 /usr/bin/sudo yes 587
```

The -au option lets you view detailed information about authentication attempts.
When you run aureport -au, it displays timestamps, the user account being accessed, and whether each authentication attempt was successful.

#### Customizing Monitoring Rules

To send your rules to auditd immediately, you use the auditctl command.
But before adding any custom rules of your own, it’s a good idea to check whether any default rules are already in place.


```
auditctl -l
No rules
```

The -l option lists all currently active rules.

A common use case for auditd is to monitor specific files or directories for changes or access.
For example, as a regular user, create a new directory inside your /home directory as shown below:

```
mkdir test_dir
```

Now, set up a watch on the directory you just created.

```commandline
auditctl -w /home/[user]/test_dir/ -k test_watch
```

The -w option tells auditd to watch the directory test_dir/ for any changes.
The -k option attaches the string test_watch (known as a key) to any log entries generated by that rule.
You can choose any key name you like, but it’s best to make it descriptive and easy to remember, since you’ll use it later to filter out unrelated log entries when reviewing auditd logs.

Now, perform some actions inside test_dir/ — for example:

- Create a few subdirectories,
- Create or copy some files,
- Delete a few files, or
- List the directory contents. 
 
When you’re done, check what auditd recorded with:

```commandline
sudo ausearch -k test_watch
```

Notice the use of -k test_watch — even if you have a dozen different rules logging various activities, the key string lets you tell ausearch to display only the events you’re interested in.

Even with this filter, the amount of information ausearch outputs can be quite overwhelming. However, you’ll also notice that the data is highly structured.
Each event consists of multiple records, and each record contains keyword/value pairs separated by an equals sign (=).
Some values are plain strings, while others are lists enclosed in parentheses.

You can look up the meaning of each field in the official manual, but the key takeaway is that this structured format makes the output easy to parse with scripts or tools.
In fact, aureport does an excellent job of turning this data into clear, organized summaries.

To demonstrate, let’s pipe the ausearch output into aureport:

```
sudo ausearch -k test_watch | aureport -f -i
```

It’s starting to make sense now! You can clearly see who is doing what, with which file, and when.

When you’re done and no longer need that watch, you can remove it using the command below:

```commandline
auditctl -W /home/[user]/test_dir/ -k test_watch
```

#### One File at a Time

Monitoring entire directories can generate a huge amount of log data. Sometimes it’s more practical to monitor only a few strategic files to ensure no one is tampering with them.
A classic example is:

```commandline
sudo auditctl -w /etc/passwd -p wa -k passwd_watch
```

This rule ensures that no unauthorized changes are made to your /etc/passwd file.

The -p parameter tells auditd which permissions or actions to watch for. The available options are:

- r — monitor read access to a file or directory
- w — monitor write access (content changes)
- x — monitor execution
- a — monitor attribute changes (permissions, ownership, etc.)

Because there are legitimate reasons for applications to read /etc/passwd, you typically don’t monitor read operations to avoid false positives. Likewise, it doesn’t make sense to monitor execution on a non-executable file.
That’s why the example above watches only for writes and attribute changes.

If you don’t specify which permissions to monitor, auditd assumes you want to track all of them.
That’s why, when monitoring the earlier test_dir/ directory, even a simple ls command triggered audit events.


#### Persistent Rules

To make your audit rules persistent (so they survive reboots), you can either:

- Add them directly to /etc/audit/audit.rules, or
- Create a new rules file inside /etc/audit/rules.d/.

If you’ve been experimenting with rules using auditctl and are satisfied with your setup, you can dump your current configuration into a file with:

```
echo "-D" > /etc/audit/rules.d/my.rules
auditctl -l >> /etc/audit/rules.d/my.rules
```

This will save your active rules into a file called my.rules, saving you from retyping everything manually.
If you followed this tutorial and used the sample rules above, your my.rules file would look like this:

```
-D
-w /home/[your_user]/test_dir/ -k test_watch
-w /etc/passwd -p wa -k passwd_watch
```

To avoid conflicts with pre-existing rule files, back them up first:

```commandline
mv /etc/audit/audit.rules /etc/audit/audit.rules.bak
mv /etc/audit/rules.d/audit.rules /etc/audit/rules.d/audit.rules.bak
```

Then restart the daemon so auditd immediately loads your new configuration:

```
sudo systemctl restart auditd.service
```

Now, every time your system boots, auditd will automatically start monitoring everything you’ve configured.

#### References

- Configure Linux system auditing with auditd - https://www.redhat.com/pt-br/blog/configure-linux-auditing-auditd 

<a name="communication-security"></a>
## Communication Security

Communication security is essential for protecting data in transit against interception, tampering, and unauthorized access. In both corporate and personal networks, even seemingly simple information can be targeted if it is not transmitted securely.

The purpose of this section is to explore and practice the configuration and use of essential tools and protocols for communication security, including:

- VPN (Virtual Private Network): Creates a secure tunnel over public networks, ensuring data confidentiality and integrity.
- SSH (Secure Shell): Provides secure remote access to systems using end-to-end encryption.
- IPSec (Internet Protocol Security): Protects IP packets through authentication and encryption, widely used in virtual private networks and secure communications between devices.

By understanding and applying these elements, it is possible to ensure that information travels securely, even in potentially vulnerable environments.

<a name="vpn"></a>
### VPN

A VPN, or Virtual Private Network, is a technology that enables the establishment of a secure, encrypted connection between a device and a private network, typically over the Internet. This is achieved by routing Internet traffic through VPN servers, thereby hiding the device’s real IP address and protecting data from interception by third parties. VPNs are often used to enhance online security and privacy, allowing users to browse anonymously and access geographically restricted content, such as streaming services, by simulating a connection from a different location. Additionally, VPNs are valuable for businesses that wish to securely extend their internal networks to remote employees, protecting sensitive data during transmission over the Internet.

The network topology for this exercise resembles the diagram below. In this scenario, consider that the client is located at home, while R1 is the company’s router that bridges the Internet and the company’s local network.


```

                                                                   srv1
                                                             (10.200.0.2)
                                                                     /
                                                                   /
client (10.201.0.1) - s1 - (10.201.0.100) r1 (10.200.0.100) - s2 - vpn (10.200.0.1)
                                                                   \
                                                                     \
                                                              (10.200.0.3)
                                                                    srv2
```

Read and analyze the code below to understand how the network topology illustrated above was constructed.

```commandline
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br
'''

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
    vpn1 = net.addDocker('vpn', dimage="ramonfontes/vpn", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:01')
    srv1 = net.addDocker('srv1', dimage="ramonfontes/seguranca", cpu_shares=20,
                          volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                          environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:04')
    srv2 = net.addDocker('srv2', dimage="ramonfontes/seguranca", cpu_shares=20,
                          volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                          environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:05')
    client1 = net.addDocker('client', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:02',
                            ip='10.201.0.1/24')
    s1 = net.addSwitch('s1', failMode='standalone')
    s2 = net.addSwitch('s2', failMode='standalone')
    r1 = net.addHost('r1', ip='10.201.0.100/24')

    info("*** Creating Links\n")
    net.addLink(client1, s1)
    net.addLink(r1, s1)
    net.addLink(r1, s2)
    net.addLink(vpn1, s2)
    net.addLink(srv1, s2)
    net.addLink(srv2, s2)

    info("*** Starting network\n")
    net.build()
    s1.start([])
    s2.start([])

    client1.cmd('route add default gw 10.201.0.100')
    srv1.cmd('ip route del 0/0')
    srv2.cmd('ip route del 0/0')
    vpn1.cmd('ip route del 0/0')
    srv1.cmd('route add default gw 10.200.0.100')
    srv2.cmd('route add default gw 10.200.0.100')
    vpn1.cmd('route add default gw 10.200.0.100')
    r1.cmd('ifconfig r1-eth1 10.200.0.100/24')

    r1.cmd('iptables -F')
    r1.cmd('iptables -A FORWARD -p tcp --dport 22 -j ACCEPT')
    r1.cmd('iptables -A FORWARD -p tcp --sport 22 -m state --state ESTABLISHED,RELATED -j ACCEPT')
    r1.cmd('iptables -t nat -A PREROUTING -p tcp --dport 22 -j DNAT --to 10.200.0.1:22')
    r1.cmd('iptables -P INPUT DROP')
    r1.cmd('iptables -P OUTPUT DROP')
    r1.cmd('iptables -P FORWARD DROP')
    vpn1.cmd("echo \'10.200.0.1 vpn1\' > /etc/hosts && /etc/init.d/ssh start")
    srv1.cmd("echo \'10.200.0.2 srv1\' > /etc/hosts && service apache2 start")
    srv2.cmd("echo \'10.200.0.3 srv2\' > /etc/hosts && service apache2 start")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Now, save the code snippet above into a Python file and execute it.
Then, open a terminal for the client and try accessing the web pages hosted on srv1 and srv2, available at the addresses 10.200.0.2 and 10.200.0.3.

Next, in the same client terminal, connect to the VPN using the following command:
```
client# sshuttle -r seg@10.201.0.100 -x 10.201.0.100 10.0.0.0/8
```
Then, try accessing the web pages mentioned above once again.

<a name="ssh"></a>
### SSH

SSH, or Secure Shell, is a widely used network protocol for securely accessing and managing remote systems. It provides robust security by encrypting data exchanged between the client and server, preventing third parties from intercepting sensitive information such as passwords and confidential data. SSH is an essential tool for system administrators and developers, enabling remote server access, secure file transfers, and command execution on Unix-like systems, including Linux. It is a fundamental technology for ensuring the integrity and confidentiality of communications in networked environments.

In addition, SSH supports public-key authentication, which is more secure than password-based methods. Users can generate SSH key pairs, where the public key is stored on the server and the private key remains with the user. This eliminates the need to enter passwords during authentication, making connections more resilient to brute-force attacks. In summary, SSH plays a crucial role in safeguarding the integrity and privacy of network connections in an increasingly connected world.

To practice an SSH connection, run the same script from the previous exercise and connect to the SSH server at 10.201.0.100 from the client.


<a name="ipsec"></a>
### IPsec

The IP Security Protocol (IPsec) is a widely used suite of protocols designed to ensure secure communications over the Internet and private networks. Two of its main components are the Authentication Header (AH) and the Encapsulating Security Payload (ESP). The AH is responsible for providing authentication and integrity for IP packets, protecting them against tampering and forgery. In contrast, the ESP adds encryption, confidentiality, and authentication features, enhancing the overall security of communications.

IPsec also supports two modes of operation: tunnel mode and transport mode. Tunnel mode is commonly used in Virtual Private Network (VPN) scenarios, where the entire original IP packet is encapsulated within a new IP packet containing additional security information. This creates a secure tunnel between two endpoints, protecting all traffic transmitted between them. Transport mode, on the other hand, protects only the payload of the IP packets while leaving the original IP header intact, which is useful when intermediate networks do not need to be aware of the applied security measures.

In this section, we will experiment with both AH and ESP, as well as transport and tunnel modes. The network topology used in the exercises is illustrated below.

```
 h1 - \                              / - h3 
       s2 --- r1 --- s1 --- r2 --- s3
 h2 - /                              \ - h4
```

For this exercise, run the script provided below.

```commandline
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br
'''

import sys

from mininet.net import Mininet
from mininet.cli import CLI
from mininet.log import setLogLevel


def topology():
    "Create a network."
    net = Mininet()

    print("*** Creating nodes")
    h1 = net.addHost('h1', mac='00:00:00:00:00:01', ip='192.168.0.1/24')
    h2 = net.addHost('h2', mac='00:00:00:00:00:02', ip='192.168.0.2/24')
    h3 = net.addHost('h3', mac='00:00:00:00:00:03', ip='192.168.1.1/24')
    h4 = net.addHost('h4', mac='00:00:00:00:00:04', ip='192.168.1.2/24')
    r1 = net.addHost('r1', mac='00:00:00:00:00:05', ip='10.0.0.1/8')
    r2 = net.addHost('r2', mac='00:00:00:00:00:06', ip='10.0.0.2/8')
    s1 = net.addSwitch('s1', failMode='standalone')
    s2 = net.addSwitch('s2', failMode='standalone')
    s3 = net.addSwitch('s3', failMode='standalone')

    print("*** Creating links")
    net.addLink(r1, s1)
    net.addLink(r2, s1)
    net.addLink(r1, s2)
    net.addLink(r2, s3)
    net.addLink(h1, s2)
    net.addLink(h2, s2)
    net.addLink(h3, s3)
    net.addLink(h4, s3)

    print("*** Building network")
    net.start()

    r1.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')
    r2.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')
    r1.cmd('ifconfig r1-eth1 192.168.0.10')
    r2.cmd('ifconfig r2-eth1 192.168.1.10')
    r1.cmd('ip route add 192.168.1.0/24 via 10.0.0.2')
    r2.cmd('ip route add 192.168.0.0/24 via 10.0.0.1')
    h1.cmd('route add default gw 192.168.0.10')
    h2.cmd('route add default gw 192.168.0.10')
    h3.cmd('route add default gw 192.168.1.10')
    h4.cmd('route add default gw 192.168.1.10')

    print("*** Adding some commands")
    if 'ESPTR' in sys.argv[1]:
        h1.cmd('ip xfrm policy add dir in src 192.168.1.1/32 dst 192.168.0.1/32 tmpl proto esp mode transport')
        h1.cmd('ip xfrm policy add dir out src 192.168.0.1/32 dst 192.168.1.1/32 tmpl proto esp mode transport')
        h3.cmd('ip xfrm policy add dir in src 192.168.0.1/32 dst 192.168.1.1/32 tmpl proto esp mode transport')
        h3.cmd('ip xfrm policy add dir out src 192.168.1.1/32 dst 192.168.0.1/32 tmpl proto esp mode transport')

        h1.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
        h1.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
        h3.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
        h3.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
    elif 'AHTR' in sys.argv[1]:
        h1.cmd('ip xfrm policy add dir in src 192.168.1.1/32 dst 192.168.0.1/32 tmpl proto ah mode transport')
        h1.cmd('ip xfrm policy add dir out src 192.168.0.1/32 dst 192.168.1.1/32 tmpl proto ah mode transport')
        h3.cmd('ip xfrm policy add dir in src 192.168.0.1/32 dst 192.168.1.1/32 tmpl proto ah mode transport')
        h3.cmd('ip xfrm policy add dir out src 192.168.1.1/32 dst 192.168.0.1/32 tmpl proto ah mode transport')

        h1.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        h1.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        h3.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        h3.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
    elif 'AHESTR' in sys.argv[1]:
        h1.cmd('ip xfrm policy add dir in src 192.168.1.1/32 dst 192.168.0.1/32 tmpl proto esp tmpl proto ah mode transport')
        h1.cmd('ip xfrm policy add dir out src 192.168.0.1/32 dst 192.168.1.1/32 tmpl proto esp tmpl proto ah mode transport')
        h3.cmd('ip xfrm policy add dir in src 192.168.0.1/32 dst 192.168.1.1/32 tmpl proto esp tmpl proto ah mode transport')
        h3.cmd('ip xfrm policy add dir out src 192.168.1.1/32 dst 192.168.0.1/32 tmpl proto esp tmpl proto ah mode transport')

        h1.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
        h1.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
        h3.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
        h3.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto esp spi 1 enc \'cbc(aes)\' 0x3ed0af408cf5dcbf5d5d9a5fa806b224 mode transport')
        h1.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        h1.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        h3.cmd('ip xfrm state add src 192.168.0.1 dst 192.168.1.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        h3.cmd('ip xfrm state add src 192.168.1.1 dst 192.168.0.1 proto ah spi 0x401 mode transport auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
    elif 'ESPTU' in sys.argv[1]:
        r1.cmd('ip xfrm policy add dir in src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto esp mode tunnel')
        r1.cmd('ip xfrm policy add dir fwd src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto esp mode tunnel')
        r1.cmd('ip xfrm policy add dir out src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto esp mode tunnel')
        r2.cmd('ip xfrm policy add dir in src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto esp mode tunnel')
        r2.cmd('ip xfrm policy add dir fwd src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto esp mode tunnel')
        r2.cmd('ip xfrm policy add dir out src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto esp mode tunnel')

        r1.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
        r1.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
        r2.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
        r2.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
    elif 'AHTU' in sys.argv[1]:
        r1.cmd('ip xfrm policy add dir in src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto ah mode tunnel')
        r1.cmd('ip xfrm policy add dir fwd src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto ah mode tunnel')
        r1.cmd('ip xfrm policy add dir out src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto ah mode tunnel')
        r2.cmd('ip xfrm policy add dir in src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto ah mode tunnel')
        r2.cmd('ip xfrm policy add dir fwd src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto ah mode tunnel')
        r2.cmd('ip xfrm policy add dir out src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto ah mode tunnel')

        r1.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        r1.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        r2.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        r2.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
    elif 'AHESTU' in sys.argv[1]: # not working yet
        r1.cmd('ip xfrm policy add dir in src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto ah tmpl src 10.0.0.2 dst 10.0.0.1 proto esp mode tunnel')
        r1.cmd('ip xfrm policy add dir fwd src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto ah tmpl src 10.0.0.2 dst 10.0.0.1 proto esp mode tunnel')
        r1.cmd('ip xfrm policy add dir out src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto ah tmpl src 10.0.0.1 dst 10.0.0.2 proto esp mode tunnel')
        r2.cmd('ip xfrm policy add dir in src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto ah tmpl src 10.0.0.1 dst 10.0.0.2 proto esp mode tunnel')
        r2.cmd('ip xfrm policy add dir fwd src 192.168.0.1/32 dst 192.168.1.1/32 tmpl src 10.0.0.1 dst 10.0.0.2 proto ah tmpl src 10.0.0.1 dst 10.0.0.2 proto esp mode tunnel')
        r2.cmd('ip xfrm policy add dir out src 192.168.1.1/32 dst 192.168.0.1/32 tmpl src 10.0.0.2 dst 10.0.0.1 proto ah tmpl src 10.0.0.2 dst 10.0.0.1 proto esp mode tunnel')

        r1.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
        r1.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
        r2.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
        r2.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto esp spi 0x201 mode tunnel enc \"cbc(aes)\" 0x303631383332323363323361323165386233366335363662')
        r1.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        r1.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        r2.cmd('ip xfrm state add src 10.0.0.1 dst 10.0.0.2 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')
        r2.cmd('ip xfrm state add src 10.0.0.2 dst 10.0.0.1 proto ah spi 0x201 mode tunnel auth \"hmac(sha1)\" 0x12345678123456781234567812345678')

    print("*** Running CLI")
    CLI(net)

    print("*** Stopping network")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()

```

Since the code above includes all possible experimentation scenarios, it is necessary to run it by specifying the desired mode of operation as an argument. For example, the following command runs the script with ESP in Transport mode:

```
$ sudo python ipsec.py -ESPTR
```

```
Supported arguments:
  ESPTR   — ESP (Encapsulating Security Payload) in Transport mode
  AHTR    — AH (Authentication Header) in Transport mode
  AHESTR  — AH + ESP in Transport mode
  ESPTU   — ESP in Tunnel mode
  AHTU    — AH in Tunnel mode
  AHESTU  — AH + ESP in Tunnel mode (not supported yet)
```