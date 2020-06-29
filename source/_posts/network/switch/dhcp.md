---

title: DHCP process
date: 2018-06-12 22:59:58
tags: [network,DHCP,Cisco ]
categories: [Network,Switch ]
comments: true
copyright: true
---



## How does DHCP work

### Lab Topology



![dhcp](https://img1.jesse.top/static/images/network/dhcp.png)


<!--more-->

R1 simulates a DHCP server ,R2 simulates a client which get IP address from the R1 DHCP server

**R1 configure:**

```
DHCP(config)#ip dhcp pool CCIE                    # define a name of address Pool
DHCP(dhcp-config)#network 10.1.1.0 255.255.255.0  # define a  subnet 
DHCP(dhcp-config)#default-router 10.1.1.1         # define default gataway
DHCP(dhcp-config)#DNS-server 8.8.8.8              # define primary DNS server
DHCP(dhcp-config)#dns-server 114.114.114.114      # define the Backup DNS server
DHCP(dhcp-config)#lease 3                         # define lease time  

DHCP(config)#ip dhcp excluded-address 10.1.1.1 ?  # define a range of address that won't assign to Clients

```



**R2 configure :**

```
R2(config)#int e1/2
R2(config-if)#ip address ?
  A.B.C.D  IP address
  dhcp     IP Address negotiated via DHCP
  pool     IP Address autoconfigured from a local DHCP pool

R2(config-if)#ip address dhcp        # Get IP address by DHCP

```

R2 succeeds in getting an IP address ,it shows 10.1.1.2:


```
*Mar  1 00:08:32.059: %DHCP-6-ADDRESS_ASSIGN: Interface Ethernet1/2 assigned DHCP address 10.1.1.2, mask 255.255.255.0, hostname R2

show DHCP lease , this command will show some information about the ip address and the server of DHCP
```

![dhcp1](https://img1.jesse.top/static/images/network/dhcp1.png)

#### Now. Let's analyse the process of HDCP :

---

![dhcp3](https://img1.jesse.top/static/images/network/dhcp3.png)

**1.Client send a DHCP Discover massage :**

the destination MAC address is broadcast address ,and the destination IP address is the broadcast address too.  
R1 sends a Bootstrap Protocol (discover) with the message type :Boot Request 1

![dhcp4](https://img1.jesse.top/static/images/network/dhcp4.png)

**2.DHCP server send an ARP request message  before assing the ip address .to make sure this address is available.**

**3.DHCP Server send an offer message to reply the client . This offer package contains all information about the ip address **



> Tips: The experiment shows that the DHCP send the offer package to broadcast . But it should bu an unicast pacakage



![dhcp5](https://img1.jesse.top/static/images/network/dhcp5.png)

**4.Clinet send a Request massage to Server to choose an ip address (there could be more than one servers in the network )**

![dhcp6](https://img1.jesse.top/static/images/network/dhcp6.png)



**5.DHCP server send an ACK message to make sure this IP address was accepted by Client. this is a Reply pacakge**

![dhcp7](https://img1.jesse.top/static/images/network/dhcp7.png)




**6.At the end . Client will send a gratuitous ARP (无故ARP) to broadcast address . to inform all others about its own IP address and ARP address :**

![dhcp8](https://img1.jesse.top/static/images/network/dhcp8.png)



--------------------------------------------------------------------------------
#### Summarize :

![dhcp9](https://img1.jesse.top/static/images/network/dhcp9.png)

---

**DHCP mainly contains below five steps :**



1.Client send a Discover message to broadcast address          #apply for an ip address
2.DHCP server send an ARP package to broadcast address     #to see if the ip address which is gonna assign to client exists in network
3.DHCP send an offer package                                                      #assign this ip address to Client
4.Client send a Request package                                                   #to announce that it chooses the ip address (there could be more than one server in network)
5.DHCP Server sends a ACK pacckage                                          #acknowledge this IP address was assigned to client

6.Client send an ARP to tell others about its ip and MAC addree     #this is not about the DHCP process .  



> Tips: DHCP is transmitted by UDP protocol with 67 port

> Caution: The Offer and ACK packages sometimes are sent within broadcast （in our experiment），But sometimes are  unicast  packages

--------------------------------------------------------------------------------

