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
                           environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:01')
    bob1 = net.addDocker('bob', dimage="ramonfontes/seguranca", cpu_shares=20,
                         volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                         environment={'DISPLAY':":{}".format(DISPLAY_ID)}, mac='00:00:00:00:00:02')

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