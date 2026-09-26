<a name="network-control-and-protection"></a>
# Network Control and Protection

<a name="firewall"></a>
## Firewall

A firewall is a security device or software designed to monitor, filter, and control network traffic between different segments, such as between an internal network and the Internet. It acts as a protective barrier, allowing or blocking data packets based on pre-defined security rules.

The objective of this section is to experiment with the configuration and use of basic firewall elements, which provide filtering functionality at both the network and application levels.

<a name="packet-filter"></a>
### Packet Filter

IPtables uses the concept of a "chain". Each chain is a list of rules (with an associated default policy) that can match a set of packets, and each rule specifies what to do with a packet that matches it. This is called a "target". In other words, a firewall rule specifies criteria for a packet and a target action. If a packet does not match a rule, the next rule in the chain is evaluated. To filter IP packets, the following three types of chains can be configured:

- INPUT – contains rules that apply to IP packets addressed to the machine where IPtables is installed.
- OUTPUT – contains rules that apply to IP packets originating from the machine where IPtables is installed.
- FORWARD – contains rules that apply to IP packets that are neither addressed to nor originating from the machine where IPtables is installed but need to be routed (forwarded) to a specific interface (if IP forwarding is enabled).

Additional chains can be created based on the operational scenario. IPtables can be configured using the iptables command; for more details, refer to the manual pages using man iptables. The main commands are illustrated below:

- -P chain target sets the default policy (e.g., ACCEPT/DROP) for a chain (INPUT, OUTPUT, FORWARD) to the target value (see explanations below for target values).
- -A chain and -I chain add a new rule at the end or beginning of a chain, respectively.
- -D chain and -R chain delete or replace a rule in a chain, where the rule can be referenced by its index in the rule list or by an IP address.
- -F [chain] flushes all rules in a specific chain (or all chains in the table if no chain is specified).
- -N chain creates a new user-defined chain with the given name (the name must not already exist as a target).
- -X chain deletes a user-defined chain.
- -L or --list [chain] lists all rules in the selected chain; if no chain is specified, all chains are listed.
- -h provides a brief description of the command syntax.

When specifying a filtering rule, information about multiple parts of the IP packet is typically provided, such as source or destination address, or source and destination ports. IPtables provides a series of options to define these criteria (see man iptables for a full list):

- -s IP_source – source IP address
- -d IP_dest – destination IP address
- --sport port – source port
- --dport port – destination port
- -i interface – network interface through which a packet was received (INPUT chain)
- -o interface – network interface through which a packet will be sent (OUTPUT chain)
- -p proto – protocol of the packet to be checked (tcp, udp, icmp, all, a numeric protocol value, or a protocol name from /etc/protocols)
- -j action – action to be executed (target in IPtables notation)
- -y or --syn – if -p tcp is used, matches only packets with the SYN flag set
- --icmp-type type – equivalent to -p icmp, where type specifies the ICMP packet type
- -l – enables logging of rule evaluation results via syslog in /var/log/messages

The action to be executed, which corresponds to the target in an IPtables command, can have one of the following values:

- ACCEPT – allows the packet to pass.
- DROP – discards the packet silently, without sending any error notification.
- REJECT – discards the packet but sends an error packet in response (by default, an "ICMP port unreachable" message is sent, though this can be customized with --reject-with).
- QUEUE – passes the packet to user space (if supported by the kernel).
- RETURN – stops traversing the current chain and resumes evaluation in the previous chain. If the end of a user-defined chain is reached or a RETURN-target rule is matched, the default policy of the chain determines the packet's fate.

Other valid targets include any user-defined chains.

<a name="personal-firewall"></a>
#### Personal Firewall

This exercise helps you become familiar with using the iptables command and then explains how to configure a local packet filter to protect a specific host.

For this exercise, consider running the code provided below and open terminals for the hosts Alice and Bob.

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

    info("*** Creating Links\n")
    net.addLink(s1, alice1)
    net.addLink(s1, bob1)

    info("*** Starting network\n")
    net.build()
    s1.start([])

    alice1.cmd("service ssh start")
    alice1.cmd("echo \'10.200.0.1 alice\' > /etc/hosts && service apache2 start")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

First, Alice checks the IPtables configuration on her machine using the following command:

```
alice# iptables -L -v -n
```

which is configured with the ACCEPT action for the INPUT, OUTPUT, and FORWARD chains.

Next, Alice starts the SSH and Apache2 services, and Bob checks whether he can reach her via ping and the services she has enabled:

```
bob# ping 10.200.0.1
```

Now he checks the HTTP connection via telnet:

```
bob# telnet 10.200.0.1 80
```

Then he connects to Alice's SSH server (the password for user seg is seg):

```
bob# ssh seg@10.200.0.1
```

At this point, Alice wants to protect herself from external connections, so a modification in the INPUT chain is required. The following command blocks all incoming connections:

``` 
alice# iptables -P INPUT DROP
```

Bob tries the same commands as before and finds that he can no longer reach Alice.

Alice then allows all ICMP traffic by running:

```
alice# iptables -A INPUT -p icmp -j ACCEPT
```

Bob verifies that he can now ping Alice again.

Next, Alice allows incoming TCP traffic to port 80:

```
alice# iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

Bob checks if he can now access http://10.200.0.1.

Alice reviews her IPtables configuration again by running the same command from the beginning of the exercise, observing all the rules she has added.

Then, Alice allows traffic to a port where no service is running, for example, port 81:

```
alice# iptables -A INPUT -p tcp --dport 81 -j ACCEPT
```

Bob starts Wireshark and runs the following Nmap scan:

```
bob# nmap -sT -Pn -n -p 22,80,81 -v 10.200.0.1
```

The results show that port 80 is open, port 81 is closed, and port 22 is filtered. But how does Nmap differentiate between closed and filtered ports? By observing the traffic in Wireshark, it can be seen that:

- For port 80, Bob received a SYN-ACK packet from Alice, so Nmap detected it as open.
- For port 81, Bob received a RST-ACK packet from Alice, indicating the port is closed (no service running).
- For port 22, no response was received due to Alice’s firewall rules, so Nmap marked it as filtered.

Later, Bob starts the Apache2 service, and Alice attempts to visit http://10.200.0.2, but she cannot access it. Observing the captured traffic from Bob, it can be seen that after receiving the TCP SYN packet, Bob responds with a SYN-ACK packet that is not directed to port 80. Due to Alice’s firewall rules, this packet is discarded.

<a name="stateless-packet-filter"></a>
### Stateless Packet Filter

In this exercise, the host Chuck will act as a firewall between Alice and Bob. Specifically, he will be configured to function as a stateless packet filter. Chuck is not the usual “malicious” character but simply a firewall node or gateway between two network points.

A stateless firewall treats each frame or network packet individually. These packet filters operate at the OSI Network Layer (Layer 3) and are more efficient because they only inspect the packet header. They do not track the context of the packet, such as the nature of the traffic. This type of firewall cannot determine whether a packet is part of an existing connection, attempting to establish a new connection, or simply an unauthorized packet.

For this exercise, consider running the code provided below and open terminals for the hosts Alice, Bob, and Chuck.

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

    bob1.cmd("service ssh start")
    bob1.cmd("echo \'10.200.0.2 bob\' > /etc/hosts && service apache2 start")
    alice1.cmd("service ssh start")
    alice1.cmd("echo \'10.200.0.1 alice\' > /etc/hosts && service apache2 start")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Alice and Bob each configure a routing rule so that packets addressed to one another pass through Chuck. To do this, execute:

```
alice# ip route add 10.200.0.2 via 10.200.0.3 dev alice-eth0
bob# ip route add 10.200.0.1 via 10.200.0.3 dev bob-eth0
```

To act as a firewall, Chuck must disable ICMP redirect packet sending so that packets always pass through him, and also enable IP forwarding; otherwise, he will not be able to forward the received packets. This can be done with the following commands:


```
chuck# echo 0 | tee /proc/sys/net/ipv4/conf/chuck-eth0/send_redirects
chuck# echo 1 > /proc/sys/net/ipv4/ip_forward
```

Now, verify that all hosts can communicate with each other, particularly ensuring that the traffic exchanged between Alice and Bob actually passes through Chuck. To do this, start Wireshark on Chuck and have Alice and Bob ping each other.

### Outgoing Traffic

At this stage, Chuck will apply an authorization policy to allow Alice to browse any external web server. Alice will act as a client on the network, and Bob will act as an external web server.

Bob starts the Apache2 and SSH services, while Chuck configures the following authorization policy on the packet filter:

```
chuck# iptables -P FORWARD DROP
chuck# iptables -A FORWARD -p tcp -s 10.200.0.1 --dport 80 -j ACCEPT
chuck# iptables -A FORWARD -p tcp -d 10.200.0.1 --sport 80 -j ACCEPT
```

With these commands, the default policy for the FORWARD chain is set to DROP, but two rules are added to allow forwarding of any packet from Alice's address destined for port 80 on another host, and any packet originating from source port 80 on a host and destined for Alice’s address. This means Alice can connect to the web server running on Bob (or any other web server), but she cannot connect to Bob’s SSH server (remember the password for user seg is seg):

```
alice# ssh seg@10.200.0.2
```

On the other hand, Bob can connect to both a web server and an SSH server running on Alice, as long as he uses port 80 as the source port. To demonstrate this, Bob runs:

```
bob# nmap -sS -Pn -n -p 22,80 10.200.0.1
```

In this case, both ports appear filtered because the firewall (Chuck) drops the packets, except those originating from source port 80 (verify this using Wireshark on Chuck and inspecting the packets).

To exploit this, add an option to Nmap so the scan originates from port 80:


```
bob# nmap -sS -Pn -n -p 22 10.200.0.1 --source-port 80
```

This results in both ports being correctly identified as open.

Now, to allow only Alice’s web connections to any external user (any IP address), Chuck should drop SYN packets directed to her with source port 80:

```
chuck# iptables -I FORWARD -p tcp -d 10.200.0.1 --sport 80 --syn -j DROP
```

This rule is inserted before the rule accepting all packets to Alice with source port 80; otherwise, it would match all SYN packets and accept them. Now, if Bob scans Alice again with Nmap from port 80, the ports will be filtered and detected correctly.

### Remote Service

Now, Chuck applies an authorization policy to allow remote SSH access to Alice. Alice will act as the SSH server on the protected network, and Bob will be an authorized remote client connecting to Alice. Chuck runs:

```
chuck# iptables -A FORWARD -p tcp -d 10.200.0.1 --dport 22 -j ACCEPT
chuck# iptables -A FORWARD -p tcp -s 10.200.0.1 --sport 22 ! --syn -j ACCEPT
```

The first rule accepts packets destined for Alice on port 22, while the second accepts traffic originating from Alice’s port 22, except SYN packets (! --syn).

Verify that Alice cannot connect to Bob via SSH:

```
alice# ssh seg@10.200.0.2
```

Verify that Bob can connect to Alice via SSH:

```
bob# ssh seg@10.200.0.1
```

<a name="statefull-packet-filter"></a>
### Statefull Packet Filter


In this exercise, Chuck will act as a firewall between Alice and Bob. Specifically, he will be configured to function as a stateful packet filter. A stateful firewall monitors the state of network connections (such as TCP flows or UDP communication) and can maintain significant attributes of each connection in memory. These attributes are collectively known as the connection state and may include details such as the IP addresses and ports involved in the connection, as well as the sequence numbers of packets traversing the connection.

Stateful inspection monitors incoming and outgoing packets over time, along with the connection state, and stores this data in dynamic state tables. These cumulative data are evaluated so that filtering decisions are not based solely on administrator-defined rules but also on the context built by previous connections and earlier packets belonging to the same connection.


For this exercise, consider running the code provided below and open terminals for the hosts Alice, Bob, and Chuck, and, immediately after running the script below, follow the same configuration as in the previous exercise from "Stateless Packet Filter" up to the “Outgoing Traffic” section.

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

    alice1.cmd("service ssh start")
    alice1.cmd("echo \'10.200.0.1 alice\' > /etc/hosts && service apache2 start")
    bob1.cmd("echo \'10.200.0.2 bob\' > /etc/hosts")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Bob then runs the following command:

```
bob# nmap -sA -Pn -n -p 22,25 10.200.0.1 --source-port 80
```

The -sA option performs a TCP ACK scan, which only detects whether the ports are filtered by the firewall. The result shows that both ports are not filtered, which is correct considering the filtering rule that allows return traffic for Alice. However, this poses a problem because it goes against the authorization policy (Alice should be able to contact external web servers, but not the other way around) and also because Bob could map the internal network, for example, using a TCP FIN scan.

To fix this issue, first flush all rules in Chuck’s FORWARD chain:

```
chuck# iptables -F FORWARD
```

Then add the following rules:

```
chuck# iptables -A FORWARD -p tcp -s 10.200.0.1 --dport 80 --syn -j ACCEPT
chuck# iptables -A FORWARD -m state --state ESTABLISHED,RELATED -j ACCEPT
```

The first rule allows forwarding of all TCP SYN packets originating from Alice and destined for port 80, while the second rule uses stateful inspection, accepting all packets belonging to an already established or related connection. The first rule allows forwarding of all TCP SYN packets originating from Alice and destined for port 80, while the second rule uses stateful inspection, accepting all packets belonging to an already established or related connection.

The advantages of this solution are that no traffic is statically enabled for Alice, and traffic is dynamically permitted on demand for precise destinations (i.e., specific IP addresses).

Finally, after enabling the Apache2 service on both hosts, verify that with these rules, Alice can connect to any external web server, while Bob cannot connect to Alice’s web server.

Note: A firewall is a device used to monitor incoming and outgoing network traffic based on defined rules. It serves as a barrier between a private internal network and the public Internet. The basic function of a firewall is to allow essential traffic and block threats. However, a firewall does not “magically” detect and block harmful traffic—it must follow rules written by humans, which means errors are possible. One common error is allowing ALL external traffic on essential ports like 21 (FTP), 53 (DNS), 80 (HTTP), 443 (HTTPS), 8080 (alternative HTTP), etc. Initially, this may seem necessary, but the key issue is allowing ALL traffic. A correct firewall configuration would allow only RELATED and ESTABLISHED external traffic, meaning inbound traffic is permitted only if the connection was initiated from inside the network. Misconfigurations can be exploited using techniques such as source port manipulation (--source-port in Nmap).

### Bandwidth Limitation

First of all, you have to delete the Alice and Bob static routing rules:

```
alice# ip route del 10.200.0.2
bob# ip route del 10.200.0.1
```

Now Chuck flushes the current FORWARD rules again:

```
chuck# iptables -F FORWARD
```

and implements the following rules:

```
chuck# iptables -A FORWARD -p icmp -s 10.200.0.1 --icmp-type echo-request -j ACCEPT
chuck# iptables -A FORWARD -p icmp -d 10.200.0.1 --icmp-type echo-reply -j ACCEPT
chuck# iptables -A FORWARD -p icmp -d 10.200.0.1 --icmp-type echo-request -j ACCEPT
chuck# iptables -A FORWARD -p icmp -s 10.200.0.1 --icmp-type echo-reply -j ACCEPT
```

Alice then runs the following command to check the system load average:

```
alice# uptime
```

The uptime command shows, in order, the current time, how long the system has been running, the number of users currently logged in, and the system load averages over the last 1, 5, and 15 minutes.

Next, Alice starts Wireshark, and Bob runs:

```
bob# ping 10.200.0.1 -w 10 -f
```

In Wireshark, it is possible to see that Alice is flooded (the meaning of the -f option) with ping requests, to which she also responds. The -w flag limits the flooding to 10 seconds. This is necessary because indefinite execution in containers could crash the system.

When flooded, Alice runs uptime again and observes whether her load average has increased significantly.

To fix this vulnerability, Chuck first deletes the rule allowing echo-request packets destined for Alice:

```
chuck# iptables -L --line-numbers
chuck# iptables -D FORWARD <rule_to_delete>
```

Then he adds a new rule:

```
chuck# iptables -A FORWARD -p icmp -d 10.200.0.1 --icmp-type echo-request -m limit --limit 20/minute --limit-burst 1 -j ACCEPT
```

This rule allows a maximum of 20 echo requests per minute, distributed evenly (approximately one every 3 seconds). The advantage of this solution is that ICMP traffic is limited, preventing it from consuming excessive bandwidth. For example, it mitigates simple Denial of Service (DoS) attacks. If Bob attempts to flood Alice with pings again, Wireshark will show that the packets she receives are now severely limited.

<a name="proxy"></a>
## Proxy

Proxy servers play a crucial role in facilitating communication between devices and the Internet. They act as intermediaries between a user and the destination server, intercepting requests and forwarding them on behalf of the user. This additional layer of abstraction provides significant benefits, such as enhanced privacy and security. By masking the user’s IP address, proxies make it more difficult to determine location and identity, offering anonymity while browsing. This functionality is particularly valuable for bypassing geographic restrictions and protecting sensitive information.

Beyond privacy, proxy servers are often used to optimize network performance. Through local caching, they temporarily store frequently accessed resources, reducing the load on origin servers and accelerating load times. This approach is especially beneficial in enterprise environments, where bandwidth can be optimized and data transmission costs reduced. Therefore, proxy servers serve a multifaceted role, addressing both security concerns and efficiency in online communication.

However, it is important to note that improper use of proxy servers can raise ethical and legal concerns. In some cases, they are used to circumvent restrictions imposed by corporate networks or for malicious activities, such as distributing malware. Responsible and ethical implementation of proxy servers is therefore crucial to ensure these tools are used in a beneficial and legal manner, providing a safer and more efficient Internet experience.

The network topology for this lab resembles the network topology shown below.

```
 client1
       \
         --- s2 --- proxy1 --- s1 --- WAN          
       /  
 client2
```

At this point, you should refer to the video available at https://www.youtube.com/watch?v=jbEBBhA--zU. In this video you will learn on how the network topology illustrated in the script below works and how to configure an HTTP proxy.

```
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
    net = Containernet(ipBase='10.201.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    proxy1 = net.addDocker('proxy', dimage="ramonfontes/proxy", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                         mac='00:00:00:00:00:01', ip='10.200.0.100/24')
    client1 = net.addDocker('client1', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            mac='00:00:00:00:00:02', ip='10.200.0.1/24')
    client2 = net.addDocker('client2', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            mac='00:00:00:00:00:03', ip='10.200.0.2/24')
    s1 = net.addSwitch('s1', failMode='standalone')
    s2 = net.addSwitch('s2', failMode='standalone')

    info("*** Creating Links\n")
    net.addLink(client1, s2)
    net.addLink(client2, s2)
    net.addLink(proxy1, s2)
    net.addLink(proxy1, s1)

    info("*** Starting network\n")
    net.build()
    net.addNAT().configDefault()
    s1.start([])
    s2.start([])

    client1.cmd('route add default gw 10.200.0.100')
    client2.cmd('route add default gw 10.200.0.100')
    proxy1.cmd('ifconfig proxy-eth1 10.201.0.100/24')
    proxy1.cmd('route add default gw 10.201.0.4')

    proxy1.cmd('echo 1 > /proc/sys/net/ipv4/ip_forward')
    proxy1.cmd('iptables -t nat -A POSTROUTING -o proxy-eth1 -j SNAT --to-source 10.201.0.100')
    proxy1.cmd('iptables -t nat -A PREROUTING -p tcp --dport 80 -j REDIRECT --to-port 8080')
    proxy1.cmd('iptables -t nat -A PREROUTING -p tcp --dport 443 -j REDIRECT --to-port 8443')
    proxy1.cmd("echo \'10.200.0.100 proxy\' > /etc/hosts && /etc/init.d/squid start")
    proxy1.cmd("/etc/init.d/apache2 start")

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

After reproducing the script above, try configuring Squid to enable the HTTPs proxy. 


## References:
- Gouda, Mohamed G., and Alex X. Liu. "A model of stateful firewalls and its properties." 2005 International Conference on Dependable Systems and Networks (DSN'05). IEEE, 2005.
- Gouda, Mohamed G., and Alex X. Liu. "Structured firewall design." Computer networks 51.4 (2007): 1106-1120.
- Luotonen, Ari. Web proxy servers. Prentice-Hall, Inc., 1998.


