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
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:01')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:02')

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


<a name="openvas"></a>
## OpenVAS
