---
title: 03-BGP-Black Hole
date: 2018-06-16 08:59:58
tags: [network,Cisco,route ]
categories: [Network,Route,BGP ]
comments: true
copyright: true
---



## BGP2. Black Hole

![hsrp8](https://img1.jesse.top/static/images/network/bgp2-1.png)

<!--more-->

Take above as an example ,  There are three AS running BGP protocol .

In the AS 65102 . Router B has neighbor relationship with Router E , But they doesn't connect with physical cable . and Router B has no any neighbor relationship with Router D,C

Now : while AS 65101 ,router A advertise its own routes and transit to Router B ,then Router B transits to E, then to F 

But while a packet with the destination IP address of Router A goes back from Router F to Router A , Router F hand it to Router E ,then Router E checks its route table and hand to Router B .

Since Router E has no physical cable connect to Router B , So he deliver this packet to Router C or D . 

But Router C or D has not any routes about this destination IP in routing table . So , he will just drop it . 

This is an interesting conclusion that Routes might could pass through between different AS ,But it doesn't mean to Data packet . 



**But why routes updates could pass through between different AS, but Data packet couldn't ? **



We still use above topology and give them some IP address :

![hsrp8](https://img1.jesse.top/static/images/network/bgp2-2.png)


**1.Routes updates packet **

Router A could hand its routes to B without any doubts ,because they have physical connections . 

When Router B hand to its neighbor Router E . here is the form of the routes updates packet :

source IP address :2.2.2.2     source port: random              routes updates

Destination IP :  3.3.3.3         Destination Port:179



As we know , BGP routes updates transits through TCP protocol which a point-to-point connection . Since there is OSPF route protocol running internal AS 65102 .Router b

and Router E is reachable . So the route updates packet could be passed successfully 



**2.Data packet **

Router F could hand its routes to E without any doubts ,because they have physical connections .

when Router E hand to its neighbor Router B, E will hand it to IGP neighbor first ,as B and E has no physical connection .

When the Data packet comes to Router C or D ,it should be this :

source ip address: origin         source Port :random

Destination IP : 1.1.1.1            destination port:unknow



it is obvious that the Source ip address and destination ip address never changes while the packet passes through the whole network . On router C or D , there is no any routes about 

1.1.1.1 network ,as they doesn't run BGP protocol . So the packet will be dropped .

---



#### Solution :

1. Full mesh connection ------- link a physical cable between Router B and E
2. all routers in as 65102 run BGP protocol 
3. MPLS/VPN