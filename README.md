# Hands-On Network Security Practice

## Summary

This document provides a comprehensive overview of **network security concepts, tools, and attack methodologies**. It begins with the preparation of the environment and system auditing, followed by techniques to secure communication and control network traffic.  

The material explores essential mechanisms for network protection, including firewalls, packet filtering, proxies, and VPNs. It then delves into network discovery, scanning techniques, and vulnerability assessment tools such as OpenVAS. The practical sections emphasize traffic capture and manipulation, covering methods like ARP spoofing, eavesdropping, and denial-of-service attacks, along with the implementation and analysis of intrusion detection and prevention systems (IDS/IPS).

Further topics address wireless network security, focusing on WEP cracking, authentication protocols, and common Wi-Fi attacks such as deauthentication and evil twin exploits. Finally, the guide examines exploitation and post-exploitation techniques, highlighting tools and methods including Metasploit, SQL injection, brute-force attacks, and cross-site scripting (XSS).

Collectively, these topics provide a comprehensive foundation for understanding, applying, and securing network infrastructures through both offensive and defensive cybersecurity practices.
## Disclaimer

Some of the operations described in this document are illegal and may lead to criminal or civil prosecution. The purpose of this document is to present these actions (and the associated tools) for educational purposes only. Readers are strongly advised to attempt any attacks only on virtual nodes created using the Mininet-WiFi emulator or on systems for which they have explicit authorization. The author of this document disclaims any responsibility for actions taken by participants that violate this policy or applicable law.


## Table of Contents

This repository currently includes partial content from:
https://docs.google.com/document/d/1bHnAAk1UvePMyo3W-KazWQrRUl47yMMHT2mweD-a-Y8/edit?usp=sharing
---

- [Environment Setup and Basic Concepts](chapert1.md#environment-setup-and-basic-concepts)  
  - [Preparing the Environment](chapert1.md#preparing-the-environment)  
  - [System Auditing](#system-auditing)  
  - [Communication Security](#communication-security)  
    - [VPN](#vpn)  
    - [SSH](#ssh)  
    - [IPSEC](#ipsec)  
- [Network Control and Protection](chapert2.md#network-control-and-protection)  
  - [Firewall](#firewall)  
    - [Packet Filter](#packet-filter)  
    - [Personal Firewall](#personal-firewall)  
    - [Stateless Packet Filter](#stateless-packet-filter)  
    - [Stateful Packet Filter](#stateful-packet-filter)  
  - [Proxy](#proxy)  
- [Discovery and Network Mapping](chapert3.md#discovery-and-network-mapping)  
  - [Network Scanning](#network-scanning)  
  - [Port Scanning](#port-scanning)  
    - [Port Scanning with Scapy](#port-scanning-with-scapy)  
  - [OpenVAS](#openvas)  
- [Network Traffic Capture and Manipulation](chapert4.md#network-traffic-capture-and-manipulation)  
  - [ARP Spoofing](#arp-spoofing)  
  - [Passive Eavesdropping Attack](#passive-eavesdropping-attack)  
  - [TCP Session Hijacking](#tcp-session-hijacking)  
  - [Denial of Service](#denial-of-service)  
    - [TCP SYN Flood](#tcp-syn-flood)  
  - [IDS/IPS](#idsips)  
- [Wireless Network Security](chapert5.md#wireless-network-security)  
  - [Cracking WEP Wi-Fi Encryption](#cracking-wep-wi-fi-encryption)  
  - [Dictionary Attack](#dictionary-attack)  
  - [IEEE 802.11x Authentication](#ieee-80211x-authentication)  
  - [Deauthentication Attack](#deauthentication-attack)  
  - [Evil Twin Attack](#evil-twin-attack)  
- [Exploitation and Post-Exploitation](chapert6.md#exploitation-and-post-exploitation)  
  - [Metasploit Framework](#metasploit-framework)  
    - [Backdoors](#backdoors)  
    - [Extracting Database Information](#extracting-database-information)  
    - [Gaining Shell Access](#gaining-shell-access)  
  - [Brute Force](#brute-force)  
    - [SQL Injection](#sql-injection)  
  - [XSS Attack](#xss-attack)
