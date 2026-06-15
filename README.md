# Hands-On Network Security Practice

## Summary

This document provides a comprehensive overview of **network security concepts, tools, and attack methodologies**. It begins with the preparation of the environment and system auditing, followed by techniques to secure communication and control network traffic.  

The material explores essential mechanisms for network protection, including firewalls, packet filtering, proxies, and VPNs. It then delves into network discovery, scanning techniques, and vulnerability assessment tools such as OpenVAS. The practical sections emphasize traffic capture and manipulation, covering methods like ARP spoofing, eavesdropping, and denial-of-service attacks, along with the implementation and analysis of intrusion detection and prevention systems (IDS/IPS).

Further topics address wireless network security, focusing on WEP cracking, authentication protocols, and common Wi-Fi attacks such as deauthentication and evil twin exploits. Finally, the guide examines exploitation and post-exploitation techniques, highlighting tools and methods including Metasploit, SQL injection, brute-force attacks, and cross-site scripting (XSS).

Collectively, these topics provide a comprehensive foundation for understanding, applying, and securing network infrastructures through both offensive and defensive cybersecurity practices.
## Disclaimer

Some of the operations described in this document are illegal and may lead to criminal or civil prosecution. The purpose of this document is to present these actions (and the associated tools) for educational purposes only. Readers are strongly advised to attempt any attacks only on virtual nodes created using the Mininet-WiFi emulator or on systems for which they have explicit authorization. The author of this document disclaims any responsibility for actions taken by participants that violate this policy or applicable law.


## Note

Most of the examples here are fully reproducible, though you may find a few harder to follow - I developed this material while teaching and sometimes filled gaps live during class. I welcome contributions to complete and clarify any missing parts. If you spot an issue or want to improve an example, please contribute!

## Table of Contents

---

- [Environment Setup and Basic Concepts](chapter1.md#environment-setup-and-basic-concepts)  
  - [Preparing the Environment](chapter1.md#preparing-the-environment)  
  - [System Auditing](chapter1.md#system-auditing)  
  - [Communication Security](chapter1.md#communication-security)  
    - [VPN](chapter1.md#vpn)  
    - [SSH](chapter1.md#ssh)  
    - [IPSEC](chapter1.md#ipsec)  
- [Network Control and Protection](chapter2.md#network-control-and-protection)  
  - [Firewall](chapter2.md#firewall)  
    - [Packet Filter](chapter2.md#packet-filter)  
    - [Personal Firewall](chapter2.md#personal-firewall)  
    - [Stateless Packet Filter](chapter2.md#stateless-packet-filter)  
    - [Stateful Packet Filter](chapter2.md#stateful-packet-filter)  
  - [Proxy](chapter2.md#proxy)  
- [Discovery and Network Mapping](chapter3.md#discovery-and-network-mapping)  
  - [Network Scanning](chapter3.md#network-scanning)  
  - [Port Scanning](chapter3.md#port-scanning)  
    - [Port Scanning with Scapy](chapter3.md#port-scanning-with-scapy)  
  - [OpenVAS](chapter3.md#openvas)  
- [Network Traffic Capture and Manipulation](chapter4.md#network-traffic-capture-and-manipulation)  
  - [ARP Spoofing](chapter4.md#arp-spoofing)  
  - [DNS Spoofing](chapter4.md#dns-spoofing)  
  - [Passive Eavesdropping Attack](chapter4.md#passive-eavesdropping-attack)  
  - [TCP Session Hijacking](chapter4.md#tcp-session-hijacking)  
  - [Denial of Service](chapter4.md#denial-of-service)  
    - [TCP SYN Flood](chapter4.md#tcp-syn-flood)  
  - [IDS/IPS](chapter4.md#idsips)  
- [Wireless Network Security](chapter5.md#wireless-network-security)  
  - [Cracking WEP Wi-Fi Encryption](chapter5.md#cracking-wep-wi-fi-encryption)  
  - [Dictionary Attack](chapter5.md#dictionary-attack)  
  - [IEEE 802.11x Authentication](chapter5.md#ieee-80211x-authentication)  
  - [Deauthentication Attack](chapter5.md#deauthentication-attack)  
  - [Evil Twin Attack](chapter5.md#evil-twin-attack)  
- [Exploitation and Post-Exploitation](chapter6.md#exploitation-and-post-exploitation)  
  - [Metasploit Framework](chapter6.md#metasploit-framework)  
    - [Backdoors](chapter6.md#backdoors)  
    - [Extracting Database Information](chapter6.md#extracting-database-information)  
    - [Gaining Shell Access](chapter6.md#gaining-shell-access)  
  - [Brute Force](chapter6.md#brute-force)  
    - [SQL Injection](chapter6.md#sql-injection)  
  - [XSS Attack](chapter6.md#xss-attack)
