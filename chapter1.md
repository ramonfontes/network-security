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