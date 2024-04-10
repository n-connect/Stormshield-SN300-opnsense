# Stormshield-SN300
Stormshield SN300 OPNSense 
![IMG_3546](https://user-images.githubusercontent.com/18091782/201496340-d1e2a905-c35f-41c7-b21e-8972f7859640.JPG)

A nice colleague of mine gave me a Stormshield SN300 a few days ago. 
Since I am a big OPNSense fan, I wanted to install the same on the Stormshield. 
Here I summarize briefly what was necessary and whether it was worth it. 
It should also be mentioned that Stormshield is a Subsidiary of Airbus Defense & Space Cyber Programmes.

## Departure
Without further ado, I broke the warranty seal to take a look at the device from the inside. This is what i saw:

![IMG_3543](https://user-images.githubusercontent.com/18091782/201493442-4b952d6a-2f44-49d7-8fc4-1d4736ba7d8b.JPG)

Hidden behind the silver cooling fins on the left, the VIA NANO U3500 @ 1000MHz (1 core, 1 thread) CPU. Behind the black cooling fins in the center of the picture, probably the VIA VX900. Behind the black fins on the right, the switch controller and just above it the Intel(R) Gigabit CT 82574L controller, that connects the board to the switch. Above, in beautiful green, the RAM with 2GB and below left in blue: The gigantic 2GB flash memory. 
2GB are unfortunately too small for the smallest OPNSense image. It needs at least 3GB.
No less important and probably therefore with two green ok stickers, the AMI BIOS from 2014 in version 02.69. Nice. 

## 1st-Boot
The first thing I did when booting was to press delete to get into the BIOS. Quiet and quick boot switched off, date and time set, saved and rebooted. Short POST image, then nothing. Should not bother us further, we want to "install" OPNSense anyway.  
![jm](https://user-images.githubusercontent.com/18091782/201546453-bb3c105d-af71-4145-a629-c045b59d069a.jpg)


## OPNSense Image
Because the 2GB flash is not big enough, I bought a 128GB 2.5 inch SATA3 SSD for a measly 15€.
As soon as the SSD was delivered, I downloaded the nano image from https://opnsense.org/download/ and unpacked it with 

```bzip2 -dk OPNsense-22.7-OpenSSL-nano-amd64.img.bz2```

Why the nano image? Because we don't have to go through an installation routine and can simply write the image with 

```dd if=OPNsense-22.7-OpenSSL-nano-amd64.img of=/dev/sda```

to the new SSD.
What does nano mean: a preinstalled serial image for USB sticks, SD or CF cards as MBR boot. These images are 3GB in size and automatically adapt to the installed media size after first boot.
Because the SATA port on the board is rather inconveniently located and there is no space on either side, you need a SATA power and data combo cable to connect an SSD.
Connecting... We will find the place in the case for the SSD later on.
Because it was still lying around with me: I upgraded the RAM to 4GB.

![IMG_3557](https://user-images.githubusercontent.com/18091782/201497103-8c0ce131-43e9-4d4c-99a5-883a783cfc7e.JPG)

## 2nd-Boot
After the POST messages OPNSense booted like any other hardware. We are welcomed by the FreeBSD and can log in with the known credentials root and opnsense. That was easy.

## Switching
Opnsense does not detect an active network interface after booting, even though I have plugged a network cable into one of the eight switch ports. Unfortunately, there is also no LED that shows a link. Only one interface and no link, for a firewall rather bad. But this should not stop us in our mission. 
As we have seen on the mainboard, there seems to be a switch chipset. You usually connect to a switch with a serial console, at least for the initial configuration.
On https://docs.freebsd.org/en/books/handbook/serialcomms/ you can find out: Call-out ports are named /dev/cuauN on FreeBSD. it also says that here are at least two utilities in the base-system of FreeBSD that can be used to work through a serial connection: cu and tip. We will use cu.

```cu -l /dev/cuau1 -s 19200```

connects us with the switch.
What the switch can do, i have put here for you
https://github.com/bmn-m/Stormshield-SN300/blob/main/SwitchMenu
We skip all the configuration options and display the switchports.

```port state```
shows 
```
Port  State     
----  --------  
1     Disabled   
2     Disabled  
3     Disabled 
4     Disabled  
5     Disabled   
6     Disabled   
7     Disabled   
8     Disabled  
9     Disabled
```
As you probably already guessed correctly, the switchports should be enabled.

```port state 1-9 enable```
does the trick.

With 

```port mode```
we can see
```
Port  Mode         Link  
----  -----------  ----  
1     Auto         1Gfdx
2     Auto         Down
3     Auto         Down
4     Auto         Down
5     Auto         Down
6     Auto         Down
7     Auto         Down
8     Auto         Down
9     1Gfdx        1Gfdx
```
that probably port one is plugged into the switch and port nine is connected to our internal ethernet controller.
Since we want to use all ports with opnsense we have to resort to VLANs. 
With 

```vlan conf```
we can see
```
VLAN Configuration:
===================


Port  PVID  Frame Type  Ingress Filter  Tx Tag      Port Type      
----  ----  ----------  --------------  ----------  -------------  
1     1     All         Disabled        Untag PVID  Unaware        
2     2     All         Disabled        Untag PVID  Unaware        
3     3     All         Disabled        Untag PVID  Unaware        
4     4     All         Disabled        Untag PVID  Unaware        
5     5     All         Disabled        Untag PVID  Unaware        
6     6     All         Disabled        Untag PVID  Unaware        
7     7     All         Disabled        Untag PVID  Unaware        
8     8     All         Disabled        Untag PVID  Unaware        
9     None  Tagged      Disabled        Untag PVID  C-Port         

VID   VLAN Name                         Ports
----  --------------------------------  -----
1     default                           1,9
2                                       2,9
3                                       3,9
4                                       4,9
5                                       5,9
6                                       6,9
7                                       7,9
8                                       8,9

VID   VLAN Name                         Ports
----  --------------------------------  -----
VLAN forbidden table is empty
```
that each port is already assigned an untagged VLAN except for port nine. Our internal port is a VLAN trunk. Perfect, on the switch we are done for now.

In case your are not seeing the above VLAN config, eg. there's no 8 VLANs set for the 8 ports - 1 for each, you need to make it from scatch. You decide:
* You want the [8 VLAN config](Switchconfig_from scratch_all_8VLANs.txt): each port has its own VLAN, VLAN 1 is the WAN, VLAN X is you choosen LAN. That seems to be the original way in NS-BSD from Stormshield's own firmware.
* You want the [2 VLAN config](Switchconfig_from scratch__2VLANs_only.txt): VLAN 1 is the WAN, VLAN 2 for your choosen LAN. That's a more generic router/fw setup. 
* you want something in between? You can make it based on the two samples above.


To exit the switch console you have to enter the following key combination.

```~.```

## Networking
In the next step we have to link the switchports to the OPNSense via the VLANS. This logical link is only an example, but it gives us the greatest possible flexibility.

```
----------------------------------------------
|      Hello, this is OPNsense 22.7          |         @@@@@@@@@@@@@@@
|                                            |        @@@@         @@@@
| Website:	https://opnsense.org/        |         @@@\\\   ///@@@
| Handbook:	https://docs.opnsense.org/   |       ))))))))   ((((((((
| Forums:	https://forum.opnsense.org/  |         @@@///   \\\@@@
| Code:		https://github.com/opnsense  |        @@@@         @@@@
| Twitter:	https://twitter.com/opnsense |         @@@@@@@@@@@@@@@
----------------------------------------------


*** OPNsense.localdomain: OPNsense 22.7.7_1 (amd64/OpenSSL) ***

 LAN (em0) -> v4: 192.168.1.1/24

 HTTPS: SHA256 XX XX XX
 SSH:   SHA256 xxxxxxxx (ECDSA)
 SSH:   SHA256 xxxxxxxx (ED25519)
 SSH:   SHA256 xxxxxxxx (RSA)

  0) Logout                              7) Ping host
  1) Assign interfaces                   8) Shell
  2) Set interface IP address            9) pfTop
  3) Reset the root password            10) Firewall log
  4) Reset to factory defaults          11) Reload all services
  5) Power off system                   12) Update from console
  6) Reboot system                      13) Restore a backup

Enter an option: 1

Do you want to configure LAGGs now? [y/N]: n
Do you want to configure VLANs now? [y/N]: y

WARNING: all existing VLANs will be cleared if you proceed!

Do you want to proceed? [y/N]: y

Valid interfaces are:

em0              00:0d:b4:aa:aa:aa Intel(R) Gigabit CT 82574L

```

Menu-driven, we now assign eight VLAN interfaces to interface em0, the parent interface. The result should look like this: 

```
em0              00:0d:b4:aa:aa:aa Intel(R) Gigabit CT 82574L
em0_vlan1        00:00:00:00:00:00 VLAN tag 1, parent interface em0
em0_vlan2        00:00:00:00:00:00 VLAN tag 2, parent interface em0
em0_vlan3        00:00:00:00:00:00 VLAN tag 3, parent interface em0
em0_vlan4        00:00:00:00:00:00 VLAN tag 4, parent interface em0
em0_vlan5        00:00:00:00:00:00 VLAN tag 5, parent interface em0
em0_vlan6        00:00:00:00:00:00 VLAN tag 6, parent interface em0
em0_vlan7        00:00:00:00:00:00 VLAN tag 7, parent interface em0
em0_vlan8        00:00:00:00:00:00 VLAN tag 8, parent interface em0
```
Now we assign the LAN and the WAN interface, as well as an IP address, so that we can log on to the web interface. For login via web interface a single assignment would be sufficient, but while we are at it...
After the login on the web interface we find the following picture in the menu under Interfaces -> Other Types -> VLAN:

![webgui](https://user-images.githubusercontent.com/18091782/201519306-0bc24536-3d8f-4dcb-b85d-7450a62493a2.png)

## Wireguard
I would like to determine two performance values later. One is the classic routing and packet filter (firewalling) performance and the other is the data throughput when the data is encrypted and sent through a wireguard tunnel. The wireguard management package can be installed via System -> Firmware -> Plugins -> os-wireguard. Since 29.10.2022, the wireguard module is officially available via FreeBSD ports and thus again official part of the FreeBSD kernel, which was not the case for over a year now. https://reviews.freebsd.org/rG4c6c8f51fdb7e2b3870ec5a6fa5dce51ad3b25a5

The VPN configuration will be a classic site to roadwarrior architecture.   
After installing the wireguard package, we can activate wireguard in the menu under VPN -> Wireguard add a local instance and an endpoint.
For the endpoint we need to create a private public key pair. For the local instance this happens automatically.
Assuming you have already installed the wireguard package on a client with
```
sudo apt install wireguard
```
you can generate the key pair with 

```
wg genkey | tee privatekey | wg pubkey > publickey
```
Make the following settings and think of a private network address that you have to enter. The endpoint has to be in the subnet of the local interface and has a 32 subnet mask, i.e. single client.

![lok](https://user-images.githubusercontent.com/18091782/201521480-4cda0b3c-f8d4-42e2-a221-56a324a4d26e.png)
In the endpoint you enter the public key you have just created.
![end](https://user-images.githubusercontent.com/18091782/201521515-e0e9d052-610b-4080-9a58-54cb3f5ede60.png)
Some firewalling.
![wan](https://user-images.githubusercontent.com/18091782/201521946-a62cb09e-d56f-4d12-a17a-ebab9ebd2b50.png)
![wgg](https://user-images.githubusercontent.com/18091782/201521950-55700784-c66a-4a06-b225-dc04199a1170.png)

Create the wireguard config on your roadwarrior. Fill in the private key you just created and the public key from your OPNSense local instance.
```
#wg5.conf
[Interface]
Address = 172.17.17.2/32
PrivateKey = aaaaaaaaaaaaaaaaaaaaaaaaaaaa

[Peer]
PublicKey = yyyyyyyyyyyyyyyyyyyyyyyyyyyyy
AllowedIPs = 172.17.17.0/24
Endpoint = 12.12.12.1:44666
```
With
```
sudo wg-quick@wg5.service
```
we connect our roadwarrior and with

```
sudo wg

interface: wg5
  public key: bbbbbbbbbbbbbbbbbbbbbbbbbbbb
  private key: (hidden)
  listening port: 47569

peer: yyyyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
  endpoint: 12.12.12.1:44666
  allowed ips: 172.17.17.0/24
  latest handshake: 18 seconds ago
  transfer: 348 B received, 436 B sent
``` 
we get the interface status.
Admittedly quite a bit of work for a small test.

## Performance-Test
The first performance test refers to the pure routing and packet filter throughput.
```Server <---WAN---> Stormshield <---LAN---> Client```
Server on the WAN interface, client on the LAN interface.
The second measurement includes the wireguard tunnel.
```Client <---Wireguard---> Stormshield <---LAN---> Server```
Roadwarrior calls a server on the LAN network.
As a test tool I use iPerf3. iPerf3 is a tool to actively measure the maximum achievable bandwidth in IP networks. So we need an iPerf3 server and an iPerf3 client.

Only rouing and paketfiltering:
```
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.1.0.219, port 39284
[  5] local 12.12.12.2 port 5201 connected to 10.1.0.219 port 39292
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec   107 MBytes   895 Mbits/sec    0    304 KBytes       
[  5]   1.00-2.00   sec   109 MBytes   917 Mbits/sec    0    304 KBytes       
[  5]   2.00-3.00   sec   110 MBytes   923 Mbits/sec    0    304 KBytes       
[  5]   3.00-4.00   sec   110 MBytes   923 Mbits/sec    0    304 KBytes       
[  5]   4.00-5.00   sec   109 MBytes   918 Mbits/sec    0    304 KBytes       
[  5]   5.00-6.00   sec   110 MBytes   923 Mbits/sec    0    304 KBytes       
[  5]   6.00-7.00   sec   109 MBytes   919 Mbits/sec    0    317 KBytes       
[  5]   7.00-8.00   sec   110 MBytes   922 Mbits/sec    0    317 KBytes       
[  5]   8.00-9.00   sec   110 MBytes   921 Mbits/sec    0    317 KBytes       
[  5]   9.00-10.00  sec   110 MBytes   924 Mbits/sec    0    317 KBytes       
[  5]  10.00-10.04  sec  3.98 MBytes   878 Mbits/sec    0    317 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.04  sec  1.07 GBytes   918 Mbits/sec    0             sender
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 10.1.0.219, port 57498
[  5] local 12.12.12.2 port 5201 connected to 10.1.0.219 port 57500
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  82.8 MBytes   695 Mbits/sec                  
[  5]   1.00-2.00   sec  87.6 MBytes   735 Mbits/sec                  
[  5]   2.00-3.00   sec  86.8 MBytes   728 Mbits/sec                  
[  5]   3.00-4.00   sec  86.5 MBytes   725 Mbits/sec                  
[  5]   4.00-5.00   sec  86.6 MBytes   727 Mbits/sec                  
[  5]   5.00-6.00   sec  86.3 MBytes   724 Mbits/sec                  
[  5]   6.00-7.00   sec  86.9 MBytes   729 Mbits/sec                  
[  5]   7.00-8.00   sec  86.7 MBytes   727 Mbits/sec                  
[  5]   8.00-9.00   sec  86.8 MBytes   728 Mbits/sec                  
[  5]   9.00-10.00  sec  87.0 MBytes   730 Mbits/sec                  
[  5]  10.00-10.05  sec  4.08 MBytes   728 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-10.05  sec   868 MBytes   725 Mbits/sec                  receiver
```
This looks ok and matches the manufracters values quiet nicely. Their advertising promise: Firewall throughput 800 Mbps.
The values changed significantly for wireguard, routing and packet filtering:
```
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 172.17.17.2, port 40184
[  5] local 10.1.0.219 port 5201 connected to 172.17.17.2 port 40196
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-1.00   sec  3.36 MBytes  28.2 Mbits/sec                  
[  5]   1.00-2.00   sec  3.72 MBytes  31.2 Mbits/sec                  
[  5]   2.00-3.00   sec  4.64 MBytes  38.9 Mbits/sec                  
[  5]   3.00-4.00   sec  2.84 MBytes  23.8 Mbits/sec                  
[  5]   4.00-5.00   sec  3.13 MBytes  26.3 Mbits/sec                  
[  5]   5.00-6.00   sec  3.33 MBytes  27.9 Mbits/sec                  
[  5]   6.00-7.00   sec  3.79 MBytes  31.8 Mbits/sec                  
[  5]   7.00-8.00   sec  5.54 MBytes  46.5 Mbits/sec                  
[  5]   8.00-9.00   sec  2.98 MBytes  25.0 Mbits/sec                  
[  5]   9.00-10.00  sec  3.52 MBytes  29.5 Mbits/sec                  
[  5]  10.00-10.07  sec   285 KBytes  35.7 Mbits/sec                  
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate
[  5]   0.00-10.07  sec  37.1 MBytes  30.9 Mbits/sec                  receiver
-----------------------------------------------------------
Server listening on 5201
-----------------------------------------------------------
Accepted connection from 172.17.17.2, port 37058
[  5] local 10.1.0.219 port 5201 connected to 172.17.17.2 port 37072
[ ID] Interval           Transfer     Bitrate         Retr  Cwnd
[  5]   0.00-1.00   sec  7.77 MBytes  65.2 Mbits/sec   12   61.5 KBytes       
[  5]   1.00-2.00   sec  11.2 MBytes  93.6 Mbits/sec   22   84.2 KBytes       
[  5]   2.00-3.00   sec  10.7 MBytes  89.5 Mbits/sec   10   80.2 KBytes       
[  5]   3.00-4.00   sec  10.1 MBytes  84.4 Mbits/sec   16   81.5 KBytes       
[  5]   4.00-5.00   sec  11.2 MBytes  93.6 Mbits/sec   21   77.5 KBytes       
[  5]   5.00-6.00   sec  11.1 MBytes  93.1 Mbits/sec   17   58.8 KBytes       
[  5]   6.00-7.00   sec  11.1 MBytes  93.1 Mbits/sec   22   60.1 KBytes       
[  5]   7.00-8.00   sec  10.9 MBytes  91.1 Mbits/sec   12   88.2 KBytes       
[  5]   8.00-9.00   sec  5.89 MBytes  49.4 Mbits/sec    0    128 KBytes       
[  5]   9.00-10.00  sec  3.68 MBytes  30.9 Mbits/sec    0    147 KBytes       
[  5]  10.00-10.08  sec   188 KBytes  19.7 Mbits/sec    0    148 KBytes       
- - - - - - - - - - - - - - - - - - - - - - - - -
[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-10.08  sec  93.6 MBytes  77.9 Mbits/sec  132             sender
```

![put](https://user-images.githubusercontent.com/18091782/201536883-941ea6f8-7e31-4b58-9d01-e8ec1a3780f9.png)
These values are rather poor and do not even come close to the IPSec values promised by the manufacturer. Their advertising: IPSec throughput - AES256/SHA2 350 Mbps.
Let's take a quick look at system performance during data transmission with wireguard:
```
root@OPNsense:~ # top -b -n 1
last pid: 36217;  load averages:  1.59,  1.35,  1.27  up 0+02:47:21    13:44:54
37 processes:  1 running, 36 sleeping
CPU:  4.9% user,  0.0% nice,  3.9% system,  0.0% interrupt, 91.2% idle
Mem: 64M Active, 64M Inact, 185M Wired, 89M Buf, 3615M Free

  PID USERNAME    THR PRI NICE   SIZE    RES STATE    TIME    WCPU COMMAND
43379 root         10  52    0   710M    20M uwait    1:33  76.12% wireguard-go

```
The high CPU load could of course be explained by the fact that the CPU has much more to do during encrypting and decrypting wireguard packages, than with IPSec with AES256/SHA2. The VIA NANO U3500 CPU has an built in AES encryption and Secure Hash Algorithm engine. IPSec benefits from both engines and wireguard does not. It looks like I still have to test IPSec on the OPNSense. 

## Preliminary Conclusion
The Stormshield SN-300 is still a good enough piece of hardware for SOHO use. Whether I will use it in my network, I do not know yet. IPSec test follows with certainty. Promised. 😉
