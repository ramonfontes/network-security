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

dsniff is a package containing a set of network traffic analysis and password detection tools designed to analyze various application protocols and can be used to perform (in addition to sniffing, filtering, etc.) man-in-the-middle attacks on a LAN. In this exercise, we will focus specifically on the "ARP poisoning" attack, illustrated in the figure below. For a description of the Ettercap tool's parameters and configuration, run the man dsniff commands.