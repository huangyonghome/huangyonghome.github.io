---
title: 03-BGP Lab BGP black hole
date: 2018-06-16 08:59:58
tags: [network,Cisco,route ]
categories: [Network,Route,BGP ]
comments: true
copyright: true
---



## LAB About BGP black hole



### LAB Topology 

 {% qnimg network/bgp3-1.png %}

---

<!--more-->



### Configure

##### 1. configure Ip address on all Routers. Test the connectivity with each neighbors  

##### 2.Configure IGP internal AS123 , here we choose EIGRP.

```
R1 : router eigrp 100
 no auto-summary
 network 12.1.1.0 0.0.0.255
 network 13.1.1.0 0.0.0.255
```

##### 3.configure IBGP  

* on R2 

```
configure bgp and specify R3 as neighbor 

R2(config)#router bgp 123                #enable BGP process and specify itself AS number
R2(config-router)#bgp router-id 2.2.2.2  #specify itself router ID
R2(config-router)#neighbor 13.1.1.3 remote 123      #specify neighbor with neighbor's ID
```



* on R3 

```
R3(config)#router bgp 123
R3(config-router)#bgp router-id 3.3.3.3
R3(config-router)#neighbor 12.1.1.2 remote 123 

*Apr  9 13:26:24.087: %BGP-5-ADJCHANGE: neighbor 12.1.1.2 Up   #prompts that BGP neighbor relationship has been set up 
```

---



##### 4.Check neighbor table 

R3#show ip bgp summary  #this command shows bgp information about neighbors 

```
BGP router identifier 3.3.3.3, local AS number 123      ----------- router itself ID , local AS
BGP table version is 1, main routing table version 1     

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
12.1.1.2        4          123       5       5        1    0    0                       00:01:38                0 
```

>12.1.1.2   #neighbor's ip address
>
>4              #the version of neighbor's BGP protocol 
>
>123        #neighbor's AS number
>
>Msgcvd /MsgSent     #receive or send messages between itself and neighbor
>
>UP/DOWN                 #the time of the relationship
>
>"State"                       #the status of neighbor relationship . "Blank" represents to be normal 
>
>"Ffxrcd"                    #shows how many 前缀 (routes updates) do you receive from neighbor. in this case it is 0



##### 5.configure EBGP 

* On R2 

```
R2(config-router)#router bgp 123
R2(config-router)#bgp route
R2(config-router)#bgp router-id 2.2.2.2
R2(config-router)#neighbor 24.1.1.4 remote 4
```

* On R4

```
R2(config-router)#router bgp 4
R2(config-router)#bgp route
R2(config-router)#bgp router-id 4.4.4.4
R2(config-router)#neighbor 24.1.1.4 remote 123
```



Check neighbor table:

```
R2#show ip bgp summary
BGP router identifier 2.2.2.2, local AS number 123
BGP table version is 1, main routing table version 1

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
13.1.1.3        4          123      22      22        1    0           0 00:16:45        0
24.1.1.4        4            4       4       4        1    0           0 00:00:43        0
```

* On R3

```
R3(config)#router bgp 123
R3(config-router)#bgp router-id 3.3.3.3
R3(config-router)#neighbor 35.1.1.5 remote 5
```

* On R5

```
R5(config)#router bgp 5
R5(config-router)#bgp router-id 5.5.5.5
R5(config-router)#neighbor 35.1.1.3 remote 123
```



Check neighbor table:

```
R5#sho ip bgp summary
BGP router identifier 5.5.5.5, local AS number 5
BGP table version is 1, main routing table version 1

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
35.1.1.3        4          123       4       4        1    0    0        00:00:21        0
```

---

##### 6.Advertise network  

1.R4 advertise its loop interface ip address:

```
R4(config)#router bgp 4
R4(config-router)#network 172.16.1.0 mask 255.255.255.0
```

Check bgp route table : 

```
R4#show ip bgp          #this command is to show the bgp route table ,not ip route table 

BGP table version is 2, local router ID is 4.4.4.4

Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,

              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,

              x best-external, a additional-path, c RIB-compressed,

Origin codes: i - IGP, e - EGP, ? - incomplete

RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop              Metric   LocPrf         Weight   Path

 *>  172.16.1.0/24    0.0.0.0                  0                           32768      i

```

> \*                    #valid route
>
> network        #advertise network infomation
>
> next hop       #the next equipment to reach the network 0.0.0.0 means itself
>
> Metric,Locprf ,weight,path     #some attributes of the route Path



Check it on R2 

```
R2#show ip bgp
BGP table version is 2, local router ID is 2.2.2.2
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.1.0/24    24.1.1.4                 0             0 4 i
```

Check the route table on R2 

```
      12.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        12.1.1.0/24 is directly connected, Serial2/1
L        12.1.1.2/32 is directly connected, Serial2/1
      13.0.0.0/24 is subnetted, 1 subnets
D        13.1.1.0 [90/2681856] via 12.1.1.1, 00:48:40, Serial2/1
      24.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        24.1.1.0/24 is directly connected, Serial4/2
L        24.1.1.2/32 is directly connected, Serial4/2
      172.16.0.0/24 is subnetted, 1 subnets
B        172.16.1.0 [20/0] via 24.1.1.4, 00:11:51
```

We can see a route path to 172.16.1.0 ..it shows to be a BGP route with 20 administrate distance ,the next hop is R4  



> Notice : EBGP admin distance is 20
>
> ​             IBGP admin distance is 200



Check the bgp route table on R3 

```
R3#show ip bgp
BGP table version is 1, local router ID is 3.3.3.3
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 * i 172.16.1.0/24    24.1.1.4                 0    100      0 4 i
```

Check the ip route table on R3 

```
      12.0.0.0/24 is subnetted, 1 subnets
D        12.1.1.0 [90/2681856] via 13.1.1.1, 00:52:40, Serial1/3
      13.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        13.1.1.0/24 is directly connected, Serial1/3
L        13.1.1.3/32 is directly connected, Serial1/3
      35.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        35.1.1.0/24 is directly connected, Serial5/3
L        35.1.1.3/32 is directly connected, Serial5/3
```



> Warn : We couldn't see BGP routes in route table  



Compare with the BGP route table on R2 and R3 ,we find that:

1.there is no ">" behind "\*" on R3 BGP table ..  ">" means the best route path to the destination 

2.the next hop in the BGP route table on R3 is 24.1.1.4 which the ip address of the interface on R4. But apparently , this is an unreachable address for R3 ,So this BGP route couldn't appear in route table



###### So we get the conclusion : 

while a route from the external AS passes through the internal AS ,the information of "next hop" never changes by default  

##### Then we need to change this situation

the best way is to let R3 know that the next hop is R2 ---------in another word is when the marginal routers receive routes from other AS and pass it through the local AS, the marginal routers take themself as the next hop .Because they know how to get to the distant network  

> note : it is unnecessary to specify next hop self on none-marginal routers 

----



#### 7.implement method :

specify itself as the next hop when configure neighbor relationship :

```
R2(config)#router bgp 123
R2(config-router)#neighbor 13.1.1.3 next-hop-self
```



Check the bgp table on R3"

```
BGP table version is 2, local router ID is 3.3.3.3

Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,

              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,

              x best-external, a additional-path, c RIB-compressed,

Origin codes: i - IGP, e - EGP, ? - incomplete

RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path

 *>i 172.16.1.0/24    12.1.1.2                 0    100      0 4 i

```

the next hop changed to R2 and ">" shows it is the best route path 

check the route table :

```
   12.0.0.0/24 is subnetted, 1 subnets

D        12.1.1.0 [90/2681856] via 13.1.1.1, 01:11:46, Serial1/3

      13.0.0.0/8 is variably subnetted, 2 subnets, 2 masks

C        13.1.1.0/24 is directly connected, Serial1/3

L        13.1.1.3/32 is directly connected, Serial1/3

      35.0.0.0/8 is variably subnetted, 2 subnets, 2 masks

C        35.1.1.0/24 is directly connected, Serial5/3

L        35.1.1.3/32 is directly connected, Serial5/3

      172.16.0.0/24 is subnetted, 1 subnets

B        172.16.1.0 [200/0] via 12.1.1.2, 00:01:34

```

Now ,we can see the BGP route.  

of course . we can see this routes update on R5 now  

```
R5#show ip bgp
BGP table version is 2, local router ID is 5.5.5.5
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  172.16.1.0/24    35.1.1.3                               0 123 4 i
```



It is obviously that when R5 advertise routes updates ,we can see the same situation on R2 and R4, So I am not gonna demonstrate it .  

 We need to specify the next hop on R3 when he configure the neighbor relationship with R2,so that the routes update from R5 could be extended to R4 

##### 8. For now , we are done with configuring BGP .we try to send a packet from R4 to R5 ,through three AS 

failed to ping R5 .check the ICMP debug information on R1 : 

```
R4#ping 192.168.1.1 source 172.16.1.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:
Packet sent with a source address of 172.16.1.1
.....
Success rate is 0 percent (0/5)
```

```
R1#debug ip icmp 
ICMP packet debugging is on
R1#
*Apr  9 17:37:58.362: ICMP: dst (192.168.1.1) host unreachable sent to 172.16.1.1
R1#
*Apr  9 17:38:00.358: ICMP: dst (192.168.1.1) host unreachable sent to 172.16.1.1
```

R1 has not the information of route about 192.168.1.0  



now we run BGP protocol on R1:

```
R3(config-router)#neighbor 13.1.1.1 remote 123
R3(config-router)#neighbor 13.1.1.1 next-hop-self

R2(config-router)#neighbor 12.1.1.1 remote 123
R2(config-router)#neighbor 12.1.1.1 next-hop-se\

R1:
router bgp 123
 bgp router-id 1.1.1.1
 bgp log-neighbor-changes
 neighbor 12.1.1.2 remote-as 123         -------------needn't take himself as the next hop
 neighbor 13.1.1.3 remote-as 123
```

now ,we can see routes in routing table about R1 and R5  

```
Gateway of last resort is not set

      12.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        12.1.1.0/24 is directly connected, Serial2/1
L        12.1.1.1/32 is directly connected, Serial2/1
      13.0.0.0/8 is variably subnetted, 2 subnets, 2 masks
C        13.1.1.0/24 is directly connected, Serial1/3
L        13.1.1.1/32 is directly connected, Serial1/3
      172.16.0.0/24 is subnetted, 1 subnets
B        172.16.1.0 [200/0] via 12.1.1.2, 00:02:26
B     192.168.1.0/24 [200/0] via 13.1.1.3, 00:01:58
```

now R4 and R5 is pingable 

```
R4#ping 192.168.1.1 source 172.16.1.1

Type escape sequence to abort.

Sending 5, 100-byte ICMP Echos to 192.168.1.1, timeout is 2 seconds:

Packet sent with a source address of 172.16.1.1

!!!!!

Success rate is 100 percent (5/5), round-trip min/avg/max = 40/56/76 ms

```













