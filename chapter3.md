<a name="discovery-and-network-mapping"></a>
# Discovery and Network Mapping

Obtaining information about network nodes involves identifying and mapping connected devices, their services and potential vulnerabilities, a common step in security audits and penetration tests.

Below we explore fundamental techniques and tools for gathering information about network nodes, essential to understanding the infrastructure and locating possible weak points. We start with network scanning, which detects connected devices and maps topology. Then we cover port scanning, used to discover active services and potential entry points. Finally, we introduce OpenVAS, a powerful vulnerability-scanning platform that automates analysis and produces detailed risk reports.

<a name="network-scanning"></a>
## Network Scanning

The goal of network scanning is to collect information about a given network. In practice, the first task is to determine which nodes on the network are active and which are not. Once active hosts are identified, further information can be gathered about them to reveal security weaknesses.

A suite of techniques known as network fingerprinting can be used to discover the remote host’s operating system. This technique relies on the fact that different operating systems (e.g., Windows and Linux) implement the TCP/IP stack in different ways. The program Nmap is a powerful and easy-to-use network scanner that can fingerprint the OS of a remote host. For a general description of the program, consult the manual with man nmap.

Start a network topology with 2 hosts (alice and chuck) and 1 switch using the script below, and open the terminals for hosts alice and chuck.

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

    alice1.cmd("echo \'10.200.0.1 alice\' > /etc/hosts && service apache2 start")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

**Note**: if any Python code does not run correctly, run sudo mn -c to clean up failed Mininet runs. This command will, among other things, automatically remove containers created during execution.

- Alice (the victim) starts the Apache2 web server on port 80 (the default HTTP port).
- Chuck (the attacker) attempts to establish a TCP connection (-sT) to port 80 (-p 80) on Alice in order to fingerprint the operating system (-O):

```
chuck# nmap -sT -p 80 -O -v 10.200.0.1
```

Examine the information collected in the output.


<a name="port-scanning"></a>
## Port Scanning

After discovering a target host, an attacker typically tries to determine which services (applications) are actually running on that host. The technique known as Port Scanning is used for this purpose. A port‑scanning tool can be used to find out which ports on a given host are open (listening for incoming connections) and which are closed.

When using such a tool you can also determine, for each port:

- the standard service name (if any) associated with that port (for example, http or ssh),
- the port number,
- the port state (open, closed, filtered by a firewall or packet filter, or unfiltered), and
- the protocol (typically TCP or UDP).


For this exercise you have to start the same network topology from the previous exercise and open terminals for hosts alice and chuck.

Alice (the victim), who already has the HTTP service running, also starts and enables the SSH service and verifies that only the services she selected are active by running:

```
alice# netstat -ltu
```

Chuck (the attacker) starts Wireshark and then performs a TCP port scan (-sT) of the first 1024 ports on Alice:

```
chuck# nmap -sT -Pn -p 1-1024 -v 10.200.0.1
```

- The -Pn option prevents Nmap from pinging the target host first. By default, Nmap may skip a host if it does not respond to ping. Some hosts are configured not to reply to ICMP echo requests to resist simple discovery techniques — -Pn forces Nmap to scan the host regardless.

In Wireshark you will see all the TCP probes (and their replies) used to scan the selected port range.

**Note**: in Wireshark you should capture on the interface whose name contains the node name and eth0, for example chuck-eth0.

Alice then chooses two services again (HTTP and SSH) but this time changes their default ports before starting them.

Chuck runs the same TCP port scan command again. This time, the basic scan output is not able to identify services correctly when they run on non‑standard ports.

To correctly identify services even when they run on non‑standard ports, run service/version detection:

```
chuck# nmap -sV -Pn -p 1-1024 -v 10.200.0.1
```

- The -sV option instructs Nmap to probe open ports and attempt to determine the actual service and version running on each port (banner grabbing, protocol probes, etc.).


#### Defensive mechanisms

Different countermeasures can help defend against port‑scanning operations:

- Firewalls (packet filters) to block or limit scan traffic.
- Network Address Translation (NAT) to hide internal addressing and topology.
- Intrusion Detection Systems (IDS) to detect scanning patterns and alert or react.
- Authenticated access to services so only authorized clients can use sensitive services.


<a name="port-scanning-with-scapy"></a>
### Port Scanning with Scapy

Scapy is a powerful Python library for crafting, sending and sniffing network packets. It’s widely used for security testing, protocol analysis, fuzzing, and custom scanners.

Features:

- Create & send packets for many protocols (IP, TCP, UDP, ICMP, etc.).
- Capture (sniff) and analyze traffic in real time.
- Perform custom port scans, banner grabs, and protocol probes.
- Simulate attacks (e.g., DoS) for resilience testing — only on authorized targets.

**Important**: Scapy requires root privileges to send/receive raw packets (e.g., sudo python3 script.py). Only scan systems you own or are authorized to test.

#### Scapy Project Example: Network Scanning

Here's a simple example of a port scanning project:

```
from scapy.all import *

def port_scan(target_ip, port_range, iface="eth0"):
    for port in range(1, port_range + 1):
        pkt = IP(dst=target_ip) / TCP(dport=port, flags='S')
        response = sr1(pkt, timeout=1, verbose=0, iface=iface)
        if response and response.haslayer(TCP) and response.getlayer(TCP).flags == 0x12:
            print(f"Port {port} is open on {target_ip}")
        elif response and response.getlayer(TCP).flags == 0x14:
            print(f"Port {port} is closed on {target_ip}")
# Usage: Scan ports 1-30 on '192.168.1.1'
port_scan('192.168.1.1', 30, iface="eth0")
```

Below is a simple example of network traffic capture and analysis, filtering packets based on specific protocols. Packet sniffing is essential for monitoring and diagnosing network traffic.

```
from scapy.all import *

def packet_callback(packet):
    if packet.haslayer(IP):
        ip_layer = packet.getlayer(IP)
        print(f"Source: {ip_layer.src}, Destination: {ip_layer.dst}")
    if packet.haslayer(TCP):
        tcp_layer = packet.getlayer(TCP)
        print(f"TCP Port: {tcp_layer.sport} -> {tcp_layer.dport}")

# Captura pacotes TCP na interface especificada
sniff(filter="tcp", prn=packet_callback, count=10, iface="eth0")

```



<a name="openvas"></a>
## OpenVAS

OpenVAS (Open Vulnerability Assessment System) is an open‑source vulnerability scanning tool used to identify security flaws in systems, networks, and applications. It is part of Greenbone Vulnerability Management (GVM), which also includes components for report management, authentication, and integrations.

OpenVAS performs active network scans that simulate attacker behavior to find vulnerabilities. It begins by discovering which hosts and services are active, identifies running software versions, and checks for known flaws using an extensive vulnerability test feed (NVT feed). During a scan you can set the target (an IP, domain, or network range) and choose a scanning policy (quick, full, or custom) depending on your needs. After the scan finishes, OpenVAS produces a detailed report that lists discovered vulnerabilities by severity and suggests remediation and mitigation steps for each finding.

For this exercise you have to start a network topology with 2 hosts (alice and chuck) and 1 switch using the script below. After running the script, a window for host chuck will open and execute an OpenVAS startup script. Allow approximately 1 minute for OpenVAS to finish initializing before starting scans.

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
    alice1 = net.addDocker('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/openvas", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw', 'openvas:/data'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID), 'PASSWORD': "seg"}, 
                           mac='00:00:00:00:00:02')

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, chuck1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    makeTerm(chuck1, cmd="bash -c '/scripts/start.sh'")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Open a terminal on chuck and start Firefox:

```
chuck# firefox
```

In Firefox, open the OpenVAS web interface at:

```
http://<chuck-ip>:9392
```

If OpenVAS has finished starting, a login/dashboard page should load. Allow roughly 1 minute after the OpenVAS startup script finishes for the service to become fully available.

After opening the OpenVAS web interface, log in with (the credentials are set in the topology script for this exercise).:

- Username: admin
- Password: seg


## References

- https://nmap.org/book/
- https://www.openvas.org/
- De Vivo, Marco, et al. "A review of port scanning techniques." ACM SIGCOMM Computer Communication Review 29.2 (1999): 41-48.
