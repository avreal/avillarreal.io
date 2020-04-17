---
title: "OSPF Stub Area Types Part 2: Not So Stubby Areas"
urltitle: "ospfareatypes2"
date: 2020-03-31T11:06:53-07:00
draft: false
showdate: true
---

The previous post left off with questions about redistribution into OSPF areas which are also classified as stub areas. Recall that stub areas filter type 4 and 5 LSAs, while totally stubby areas filter type 3,4, and 5. So what happens if we we wish to redistribute into a stubby area but keep other external routes out, or redistribute and have access to the entire network but with a minimally sized routing table and LSDB? In this post I will spin up the [__lab__](/1resources/images/ospfstub1/topology.PNG) (gns3 project download [__here__](/1resources/misc/ospfstub1/OSPFStubArea.gns3project)) from the previous post and demonstrate what are called Not So Stubby Areas, as well as various subconfigurations of NSSAs. <!--more--> 

### Redistribution into Stub Areas

As we saw in the previous post, configuring an area as a stub was a matter of a single config line in OSPF config mode. Now lets go to area 3 in our topology and suppose we also want that to be a stub, as we don't want external routes from area 4 and 5 in our routing table. Here is the initial link state database (showing only external LSAs)

~~~
EIGRP3#sh ip ospf database

                Summary ASB Link States (Area 3)

Link ID         ADV Router      Age         Seq#       Checksum
6.6.6.5         192.168.103.1   29          0x80000001 0x0053EB
44.44.44.5      192.168.103.1   29          0x80000001 0x00BEF9
55.55.55.5      192.168.103.1   29          0x80000001 0x003166

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
6.6.6.1         6.6.6.5         1944        0x80000001 0x005620 0
6.6.6.2         6.6.6.5         1944        0x80000001 0x004C29 0
6.6.6.3         6.6.6.5         1944        0x80000001 0x004232 0
6.6.6.4         6.6.6.5         1944        0x80000001 0x00383B 0
6.6.6.5         6.6.6.5         1944        0x80000001 0x002E44 0
33.33.33.1      33.33.33.5      18          0x80000001 0x00FBD7 0
33.33.33.2      33.33.33.5      18          0x80000001 0x00F1E0 0
33.33.33.3      33.33.33.5      18          0x80000001 0x00E7E9 0
33.33.33.4      33.33.33.5      18          0x80000001 0x00DDF2 0
33.33.33.5      33.33.33.5      18          0x80000001 0x00D3FB 0
44.44.44.1      44.44.44.5      1882        0x80000001 0x00652C 0
44.44.44.2      44.44.44.5      1882        0x80000001 0x005B35 0
44.44.44.3      44.44.44.5      1882        0x80000001 0x00513E 0
44.44.44.4      44.44.44.5      1882        0x80000001 0x004747 0
44.44.44.5      44.44.44.5      1882        0x80000001 0x003D50 0
55.55.55.1      55.55.55.5      1882        0x80000001 0x00CE80 0
55.55.55.2      55.55.55.5      1882        0x80000001 0x00C489 0
55.55.55.3      55.55.55.5      1882        0x80000001 0x00BA92 0
55.55.55.4      55.55.55.5      1882        0x80000001 0x00B09B 0
55.55.55.5      55.55.55.5      1882        0x80000001 0x00A6A4 0
EIGRP3#
~~~

Now lets configure the routers in area 3 as a stub, using the 'area 3 stub' command on all of the routers. We will start on the ASBR

~~~
EIGRP3(config-router)#area 3 stub
EIGRP3(config-router)#
*Apr 16 22:11:45.831: %OSPF-4-ASBR_WITHOUT_VALID_AREA: Router is currently an 
ASBR while having only one area which is a stub area
EIGRP3(config-router)#
~~~

We immediately see that the ASBR throws a message about our configuration. Why is this the case? Lets configure the other routers as a stub so the adjacencies form again and then take a look.

~~~
AREA3ABR(config-router)#area 3 stub
AREA3ABR(config-router)#end
AREA3ABR#
~~~
~~~
AREA3(config-router)#area 3 stub
*Apr 16 22:19:14.152: %OSPF-5-ADJCHG: Process 1, Nbr 192.168.103.1 on 
Ethernet0/0 from FULL to DOWN, Neighbor Down: Adjacency forced to reset

*Apr 16 22:19:14.968: %OSPF-5-ADJCHG: Process 1, Nbr 192.168.103.1 on 
Ethernet0/0 from LOADING to FULL, Loading Done
AREA3(config-router)#end
AREA3#
~~~

The other 2 routers didn't seem to mind the configuration. Lets go back to the EIGRP3 router which has redistribution taking place, and is also an OSPF internal router in area3. Since we have configured it as a stub we know that the type 4 and 5 LSAs telling us about external destinations will not be flooded, but what about the external routes coming from the same router? Lets look at the link state database.

~~~
EIGRP3#show ip ospf database external

            OSPF Router with ID (33.33.33.5) (Process ID 1)
EIGRP3#
~~~

The other 2 routers have the same output. So what has happened?

By configuring area 3 as a stub we stopped type 4 and 5 LSAs as predicted, but even those LSAs from redistribution directly into our area were stopped. As a result, the routers can still reach all external destinations (because the ABR injects a default route), except the ones being redistributed from the EIGRP3 router! (the EIGRP3 router can still reach them because they are connected routes there) When we configured the ASBR as a stub earlier, that syslog message was warning us that we have configured our ASBR as a stub where no other OSPF areas exist, so there are likely going to be adverse effects as a result of this configuration!

So how can we configure the internal routers in area 3 to have access to the external destinations that are being redistributed into their area, but also cut down on the size of the routing table/lsdb for other external destinations? The answer is ....

### The Not So Stubby Area

Once this topological limitation became apparent, the IETF introduced a standard for a new OSPF area type in [__RFC 1587__](https://tools.ietf.org/html/rfc1587). With the Not So Stubby Area (referred to as NSSA from now on) came type 7 LSAs, which are very similar to type 5 LSAs but are only propagated within the NSSA. This allows the routers within the NSSA to populate their databases with LSAs for the redistributed routes but also limit the propagation of type 4 and 5 LSAs. Let's go back to the router called AREA3. The full LSDB can be accessed [__here__](/1resources/textfiles/ospfstub2/area3lsdb.txt), but you can refer to the output pasted above to see the initial state.

Configuration is quite simple, as we did with the classic stub area we just do

~~~
area 3 nssa
~~~

on all of the routers in the area. As with classic stub areas, neighboring routers must have the same stub configuration for the adjacency to form. This is facilitated by the OSPF hello packet option field which has an N bit that is flipped when the area is set as a nssa. When this bit is set, the E bit is unset.

Before setting nssa:

![nssa0](/1resources/images/ospfstub2/nssa0.PNG)

After setting nssa:

![nssa1](/1resources/images/ospfstub2/nssa1.PNG)

Now lets check out the LSDB of our internal area 3 router now that area 3 is a nssa.

~~~
AREA3#sh ip ospf database external

            OSPF Router with ID (3.3.3.5) (Process ID 1)
AREA3#
~~~

As expected the type 4 and 5 LSAs are not present but wait..shouldn't we at least have LSAs for the redistributed routes from the EIGRP3 router? YES, but recall from above that a new LSA type was introduced specifically for NSSAs. We can see this when we run the unfiltered sh ip ospf database (or sh ip ospf database nssa-external) command:

~~~
AREA3#sh ip ospf database

            OSPF Router with ID (3.3.3.5) (Process ID 1)

***truncated***

  Type-7 AS External Link States (Area 3)

Link ID         ADV Router      Age         Seq#       Checksum Tag
33.33.33.1      33.33.33.5      1145        0x80000001 0x003B05 0
33.33.33.2      33.33.33.5      1145        0x80000001 0x00310E 0
33.33.33.3      33.33.33.5      1145        0x80000001 0x002717 0
33.33.33.4      33.33.33.5      1145        0x80000001 0x001D20 0
33.33.33.5      33.33.33.5      1145        0x80000001 0x001329 0
~~~

Now lets look at the routing table:

~~~
AREA3#sh ip route | include O E2
AREA3#
~~~

We have no OSPF E2 routes, because type 5 LSAs are blocked. But where are the redistributed routes? (full sh ip route output [__here__](/1resources/textfiles/ospfstub2/area3nssalsdb.txt))

~~~
AREA3#sh ip route

***truncated***

O N2     33.33.33.1 [110/20] via 192.168.30.2, 00:25:35, Ethernet0/1
O N2     33.33.33.2 [110/20] via 192.168.30.2, 00:25:35, Ethernet0/1
O N2     33.33.33.3 [110/20] via 192.168.30.2, 00:25:35, Ethernet0/1
O N2     33.33.33.4 [110/20] via 192.168.30.2, 00:25:35, Ethernet0/1
O N2     33.33.33.5 [110/20] via 192.168.30.2, 00:25:35, Ethernet0/1
~~~

As you can see we have a new route type! OSPF N2 routes are used to specifically denote routes generated from type 7 LSAs. These type 7 LSAs are then translated into type 5 LSAs at the ABR, which are flooded through the rest of the OSPF domain. It is very important to understand that type 7 LSAs *only exist within a NSSA*. 

Note no type 7 LSAs present on the hub router which is connected to an NSSA, they exist as type 5 here.
~~~
HUB#sh ip ospf database nssa-external

            OSPF Router with ID (192.168.10.1) (Process ID 1)
HUB#
~~~

So we have access to the networks from the EIGRP domain now, but if you look at the routing table there is no default route as with a stub area. That is because in NSSA the default route to the ABR is optional. With the base NSSA configuration we are allowing the redistributed networks in as type 7 LSAs but not allowing type 4 or 5 LSAs, as well as not generating a default route to the other external destinations. This is because there could be paths to external destinations both through the ABR, as well as beyond the ASBR.

In terms of our lab, configuring area 3 as NSSA means we can reach the EIGRP3 domain, but none of the other EIGRP domains. The next sections show how this can be rectified.

### Injecting Default Routes Into NSSA

As we saw in the previous section, configuring an area as a NSSA allows for reachability to routes redistributed directly into the domain, but external destinations in other areas become unreachable due to the blocking of type 5 LSAs and no default route. In this section we will see a few ways to rectify that.

First we will go over to area 4 in our topology and look at the LSDB and routing table. Click [__here__](/1resources/textfiles/ospfstub2/area4lsdb.txt) for the full LSDB and [__here__](/1resources/textfiles/ospfstub2/area4route.txt) for the full routing table. We are interested in the type 3,4, and 5 LSAs.

~~~

                Summary Net Link States (Area 4)

Link ID         ADV Router      Age         Seq#       Checksum
1.1.1.1         192.168.104.1   691         0x80000002 0x00A3A2
1.1.1.2         192.168.104.1   691         0x80000002 0x0099AB
1.1.1.3         192.168.104.1   691         0x80000002 0x008FB4
1.1.1.4         192.168.104.1   691         0x80000002 0x0085BD
1.1.1.5         192.168.104.1   691         0x80000002 0x007BC6
2.2.2.1         192.168.104.1   428         0x80000002 0x007FC3
2.2.2.2         192.168.104.1   428         0x80000002 0x0075CC
2.2.2.3         192.168.104.1   428         0x80000002 0x006BD5
2.2.2.4         192.168.104.1   428         0x80000002 0x0061DE
2.2.2.5         192.168.104.1   428         0x80000002 0x0057E7
3.3.3.1         192.168.104.1   941         0x80000004 0x0057E6
3.3.3.2         192.168.104.1   941         0x80000004 0x004DEF
3.3.3.3         192.168.104.1   941         0x80000004 0x0043F8
3.3.3.4         192.168.104.1   941         0x80000004 0x003902
3.3.3.5         192.168.104.1   941         0x80000004 0x002F0B
5.5.5.1         192.168.104.1   941         0x80000002 0x001327
5.5.5.2         192.168.104.1   941         0x80000002 0x000930
5.5.5.3         192.168.104.1   941         0x80000002 0x00FE39
5.5.5.4         192.168.104.1   941         0x80000002 0x00F442
5.5.5.5         192.168.104.1   941         0x80000002 0x00EA4B

***physical interface lsa omitted, there are alot of them!***

                Summary ASB Link States (Area 4)

Link ID         ADV Router      Age         Seq#       Checksum
6.6.6.5         192.168.104.1   941         0x80000002 0x004AF2
55.55.55.5      192.168.104.1   941         0x80000002 0x00286D
192.168.103.1   192.168.104.1   1369        0x80000001 0x002A59

                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
6.6.6.1         6.6.6.5         915         0x80000002 0x005421 0
6.6.6.2         6.6.6.5         915         0x80000002 0x004A2A 0
6.6.6.3         6.6.6.5         915         0x80000002 0x004033 0
6.6.6.4         6.6.6.5         915         0x80000002 0x00363C 0
6.6.6.5         6.6.6.5         915         0x80000002 0x002C45 0
33.33.33.1      192.168.103.1   1345        0x80000001 0x0023BD 0
33.33.33.2      192.168.103.1   1345        0x80000001 0x0019C6 0
33.33.33.3      192.168.103.1   1345        0x80000001 0x000FCF 0
33.33.33.4      192.168.103.1   1345        0x80000001 0x0005D8 0
33.33.33.5      192.168.103.1   1345        0x80000001 0x00FAE1 0
44.44.44.1      44.44.44.5      925         0x80000002 0x00632D 0
44.44.44.2      44.44.44.5      925         0x80000002 0x005936 0
44.44.44.3      44.44.44.5      925         0x80000002 0x004F3F 0
44.44.44.4      44.44.44.5      925         0x80000002 0x004548 0
44.44.44.5      44.44.44.5      925         0x80000002 0x003B51 0
55.55.55.1      55.55.55.5      814         0x80000002 0x00CC81 0
55.55.55.2      55.55.55.5      814         0x80000002 0x00C28A 0
55.55.55.3      55.55.55.5      814         0x80000002 0x00B893 0
55.55.55.4      55.55.55.5      814         0x80000002 0x00AE9C 0
55.55.55.5      55.55.55.5      814         0x80000002 0x00A4A5 0
AREA4#
~~~

~~~
AREA4#show ip route | include O E2
O E2     6.6.6.1 [110/20] via 192.168.104.1, 00:52:14, Ethernet0/0
O E2     6.6.6.2 [110/20] via 192.168.104.1, 00:52:14, Ethernet0/0
O E2     6.6.6.3 [110/20] via 192.168.104.1, 00:52:14, Ethernet0/0
O E2     6.6.6.4 [110/20] via 192.168.104.1, 00:52:14, Ethernet0/0
O E2     6.6.6.5 [110/20] via 192.168.104.1, 00:52:14, Ethernet0/0
O E2     33.33.33.1 [110/20] via 192.168.104.1, 00:26:44, Ethernet0/0
O E2     33.33.33.2 [110/20] via 192.168.104.1, 00:26:44, Ethernet0/0
O E2     33.33.33.3 [110/20] via 192.168.104.1, 00:26:44, Ethernet0/0
O E2     33.33.33.4 [110/20] via 192.168.104.1, 00:26:44, Ethernet0/0
O E2     33.33.33.5 [110/20] via 192.168.104.1, 00:26:44, Ethernet0/0
O E2     44.44.44.1 [110/20] via 192.168.40.2, 00:52:09, Ethernet0/1
O E2     44.44.44.2 [110/20] via 192.168.40.2, 00:52:09, Ethernet0/1
O E2     44.44.44.3 [110/20] via 192.168.40.2, 00:52:09, Ethernet0/1
O E2     44.44.44.4 [110/20] via 192.168.40.2, 00:52:09, Ethernet0/1
O E2     44.44.44.5 [110/20] via 192.168.40.2, 00:52:09, Ethernet0/1
O E2     55.55.55.1 [110/20] via 192.168.104.1, 00:50:55, Ethernet0/0
O E2     55.55.55.2 [110/20] via 192.168.104.1, 00:50:55, Ethernet0/0
O E2     55.55.55.3 [110/20] via 192.168.104.1, 00:50:55, Ethernet0/0
O E2     55.55.55.4 [110/20] via 192.168.104.1, 00:50:55, Ethernet0/0
O E2     55.55.55.5 [110/20] via 192.168.104.1, 00:50:55, Ethernet0/0
~~~

~~~
AREA4#show ip route | include O IA
O IA     1.1.1.1 [110/41] via 192.168.104.1, 00:46:23, Ethernet0/0
O IA     1.1.1.2 [110/41] via 192.168.104.1, 00:46:23, Ethernet0/0
O IA     1.1.1.3 [110/41] via 192.168.104.1, 00:46:23, Ethernet0/0
O IA     1.1.1.4 [110/41] via 192.168.104.1, 00:46:23, Ethernet0/0
O IA     1.1.1.5 [110/41] via 192.168.104.1, 00:46:23, Ethernet0/0
O IA     2.2.2.1 [110/41] via 192.168.104.1, 00:42:50, Ethernet0/0
O IA     2.2.2.2 [110/41] via 192.168.104.1, 00:42:50, Ethernet0/0
O IA     2.2.2.3 [110/41] via 192.168.104.1, 00:42:50, Ethernet0/0
O IA     2.2.2.4 [110/41] via 192.168.104.1, 00:42:50, Ethernet0/0
O IA     2.2.2.5 [110/41] via 192.168.104.1, 00:42:50, Ethernet0/0
O IA     3.3.3.1 [110/41] via 192.168.104.1, 00:27:13, Ethernet0/0
O IA     3.3.3.2 [110/41] via 192.168.104.1, 00:27:13, Ethernet0/0
O IA     3.3.3.3 [110/41] via 192.168.104.1, 00:27:13, Ethernet0/0
O IA     3.3.3.4 [110/41] via 192.168.104.1, 00:27:13, Ethernet0/0
O IA     3.3.3.5 [110/41] via 192.168.104.1, 00:27:13, Ethernet0/0
O IA     5.5.5.1 [110/41] via 192.168.104.1, 00:51:23, Ethernet0/0
O IA     5.5.5.2 [110/41] via 192.168.104.1, 00:51:23, Ethernet0/0
O IA     5.5.5.3 [110/41] via 192.168.104.1, 00:51:23, Ethernet0/0
O IA     5.5.5.4 [110/41] via 192.168.104.1, 00:51:23, Ethernet0/0
O IA     5.5.5.5 [110/41] via 192.168.104.1, 00:51:24, Ethernet0/0

***physical interface routes omitted, there are alot of them!***
AREA4#
~~~

As with the classic stub area, we can configure 

~~~
area 4 nssa no-summary
~~~

on the ABR and then 'area 4 nssa' on the others. Lets see how this affects the LSDB and routing table.

~~~
AREA4#show ip ospf database

            OSPF Router with ID (4.4.4.5) (Process ID 1)

                Router Link States (Area 4)

Link ID         ADV Router      Age         Seq#       Checksum Link count
4.4.4.5         4.4.4.5         34          0x80000008 0x00B4BF 7
44.44.44.5      44.44.44.5      35          0x80000006 0x00E3FE 1
192.168.104.1   192.168.104.1   41          0x80000006 0x00D9F6 1

                Net Link States (Area 4)

Link ID         ADV Router      Age         Seq#       Checksum
192.168.40.2    44.44.44.5      35          0x80000004 0x0098E3
192.168.104.1   192.168.104.1   41          0x80000004 0x001F8B

                Summary Net Link States (Area 4)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         192.168.104.1   57          0x80000001 0x004C17

                Type-7 AS External Link States (Area 4)

Link ID         ADV Router      Age         Seq#       Checksum Tag
44.44.44.1      44.44.44.5      41          0x80000001 0x0027CC 0
44.44.44.2      44.44.44.5      41          0x80000001 0x001DD5 0
44.44.44.3      44.44.44.5      41          0x80000001 0x0013DE 0
44.44.44.4      44.44.44.5      41          0x80000001 0x0009E7 0
44.44.44.5      44.44.44.5      41          0x80000001 0x00FEF0 0
AREA4#
~~~

~~~
AREA4#show ip route
Codes: ***redacted***

Gateway of last resort is 192.168.104.1 to network 0.0.0.0

O*IA  0.0.0.0/0 [110/11] via 192.168.104.1, 00:01:01, Ethernet0/0
      4.0.0.0/32 is subnetted, 5 subnets
C        4.4.4.1 is directly connected, Loopback1
C        4.4.4.2 is directly connected, Loopback2
C        4.4.4.3 is directly connected, Loopback3
C        4.4.4.4 is directly connected, Loopback4
C        4.4.4.5 is directly connected, Loopback5
      44.0.0.0/32 is subnetted, 5 subnets
O N2     44.44.44.1 [110/20] via 192.168.40.2, 00:00:56, Ethernet0/1
O N2     44.44.44.2 [110/20] via 192.168.40.2, 00:00:56, Ethernet0/1
O N2     44.44.44.3 [110/20] via 192.168.40.2, 00:00:56, Ethernet0/1
O N2     44.44.44.4 [110/20] via 192.168.40.2, 00:00:56, Ethernet0/1
O N2     44.44.44.5 [110/20] via 192.168.40.2, 00:00:56, Ethernet0/1
      192.168.40.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.40.0/30 is directly connected, Ethernet0/1
L        192.168.40.1/32 is directly connected, Ethernet0/1
      192.168.104.0/24 is variably subnetted, 2 subnets, 2 masks
C        192.168.104.0/30 is directly connected, Ethernet0/0
L        192.168.104.2/32 is directly connected, Ethernet0/0
AREA4#
~~~

I have not omitted any routes or LSAs here, this is the full output! Now type 3,4, and 5 LSAs are not being flooded throughout the network with the exception of the type 3 LSA specifying a default route as an O IA route. With this configuration the routers in area 4 have full reachability to the OSPF domain, the external routes beyond the OSPF domain, as well as the directly redistributed networks.

However now there is a case that hasn't been satisfied, what if we still want specific routes to the other OSPF areas but not the external destinations, but also want reachability to everything? In this case we will go to area 5 and configure.

~~~
area 5 nssa default-information-originate
~~~

on the ABR and 'area 5 nssa' on the others. The initial LSDB is very similar to the one I pasted above for area 4 so you may reference that for the initial state. Lets look at the result:

~~~
AREA5#show ip ospf database

            OSPF Router with ID (5.5.5.5) (Process ID 1)

                Router Link States (Area 5)

Link ID         ADV Router      Age         Seq#       Checksum Link count
5.5.5.5         5.5.5.5         33          0x80000007 0x006DDC 7
55.55.55.5      55.55.55.5      34          0x80000005 0x002C61 1
192.168.105.1   192.168.105.1   40          0x80000005 0x00DFED 1

                Net Link States (Area 5)

Link ID         ADV Router      Age         Seq#       Checksum
192.168.50.2    55.55.55.5      34          0x80000004 0x004EDE
192.168.105.1   192.168.105.1   40          0x80000004 0x003A6A

                Summary Net Link States (Area 5)

Link ID         ADV Router      Age         Seq#       Checksum
1.1.1.1         192.168.105.1   56          0x80000003 0x0040FD
1.1.1.2         192.168.105.1   56          0x80000003 0x003607
1.1.1.3         192.168.105.1   56          0x80000003 0x002C10
1.1.1.4         192.168.105.1   56          0x80000003 0x002219
1.1.1.5         192.168.105.1   56          0x80000003 0x001822
2.2.2.1         192.168.105.1   56          0x80000003 0x001C1F
2.2.2.2         192.168.105.1   56          0x80000003 0x001228
2.2.2.3         192.168.105.1   56          0x80000003 0x000831
2.2.2.4         192.168.105.1   56          0x80000003 0x00FD3A
2.2.2.5         192.168.105.1   56          0x80000003 0x00F343
3.3.3.1         192.168.105.1   56          0x80000005 0x00F342
3.3.3.2         192.168.105.1   56          0x80000005 0x00E94B
3.3.3.3         192.168.105.1   56          0x80000005 0x00DF54
3.3.3.4         192.168.105.1   56          0x80000005 0x00D55D
3.3.3.5         192.168.105.1   56          0x80000005 0x00CB66
4.4.4.1         192.168.105.1   56          0x80000002 0x00D560
4.4.4.2         192.168.105.1   56          0x80000002 0x00CB69
4.4.4.3         192.168.105.1   56          0x80000002 0x00C172
4.4.4.4         192.168.105.1   56          0x80000002 0x00B77B
4.4.4.5         192.168.105.1   56          0x80000002 0x00AD84
10.10.10.1      192.168.105.1   56          0x80000003 0x009696
10.10.10.2      192.168.105.1   56          0x80000003 0x008C9F
10.10.10.3      192.168.105.1   56          0x80000003 0x0082A8
10.10.10.4      192.168.105.1   56          0x80000003 0x0078B1
10.10.10.5      192.168.105.1   56          0x80000003 0x006EBA
192.168.1.0     192.168.105.1   56          0x80000003 0x0031B4
192.168.2.0     192.168.105.1   56          0x80000003 0x0026BE
192.168.3.0     192.168.105.1   56          0x80000003 0x001BC8
192.168.4.0     192.168.105.1   56          0x80000003 0x0010D2
192.168.5.0     192.168.105.1   56          0x80000003 0x00A04B
192.168.6.0     192.168.105.1   56          0x80000003 0x00F9E6
192.168.10.0    192.168.105.1   56          0x80000003 0x00CD0F
192.168.30.0    192.168.105.1   56          0x80000003 0x00B9FA
192.168.40.0    192.168.105.1   56          0x80000002 0x004D5E
192.168.100.0   192.168.105.1   56          0x80000003 0x005028
192.168.102.0   192.168.105.1   56          0x80000003 0x003A3C
192.168.103.0   192.168.105.1   56          0x80000003 0x002F46
192.168.104.0   192.168.105.1   56          0x80000003 0x002450

                Type-7 AS External Link States (Area 5)

Link ID         ADV Router      Age         Seq#       Checksum Tag
0.0.0.0         192.168.105.1   56          0x80000001 0x0019C4 0
55.55.55.1      55.55.55.5      39          0x80000001 0x001394 0
55.55.55.2      55.55.55.5      39          0x80000001 0x00099D 0
55.55.55.3      55.55.55.5      39          0x80000001 0x00FEA6 0
55.55.55.4      55.55.55.5      39          0x80000001 0x00F4AF 0
55.55.55.5      55.55.55.5      39          0x80000001 0x00EAB8 0
AREA5#
~~~

And the routing table:

~~~
AREA5#sh ip route
Codes: ***redacted***

Gateway of last resort is 192.168.105.1 to network 0.0.0.0

O*N2  0.0.0.0/0 [110/1] via 192.168.105.1, 00:01:19, Ethernet0/0
      1.0.0.0/32 is subnetted, 5 subnets
O IA     1.1.1.1 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     1.1.1.2 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     1.1.1.3 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     1.1.1.4 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     1.1.1.5 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
      2.0.0.0/32 is subnetted, 5 subnets
O IA     2.2.2.1 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     2.2.2.2 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     2.2.2.3 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     2.2.2.4 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     2.2.2.5 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
      3.0.0.0/32 is subnetted, 5 subnets
O IA     3.3.3.1 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     3.3.3.2 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     3.3.3.3 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     3.3.3.4 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     3.3.3.5 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
      4.0.0.0/32 is subnetted, 5 subnets
O IA     4.4.4.1 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     4.4.4.2 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     4.4.4.3 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     4.4.4.4 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     4.4.4.5 [110/41] via 192.168.105.1, 00:01:19, Ethernet0/0
      5.0.0.0/32 is subnetted, 5 subnets
C        5.5.5.1 is directly connected, Loopback1
C        5.5.5.2 is directly connected, Loopback2
C        5.5.5.3 is directly connected, Loopback3
C        5.5.5.4 is directly connected, Loopback4
C        5.5.5.5 is directly connected, Loopback5
      10.0.0.0/32 is subnetted, 5 subnets
O IA     10.10.10.1 [110/31] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     10.10.10.2 [110/31] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     10.10.10.3 [110/31] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     10.10.10.4 [110/31] via 192.168.105.1, 00:01:19, Ethernet0/0
O IA     10.10.10.5 [110/31] via 192.168.105.1, 00:01:19, Ethernet0/0
      55.0.0.0/32 is subnetted, 5 subnets
O N2     55.55.55.1 [110/20] via 192.168.50.2, 00:01:14, Ethernet0/1
O N2     55.55.55.2 [110/20] via 192.168.50.2, 00:01:14, Ethernet0/1
O N2     55.55.55.3 [110/20] via 192.168.50.2, 00:01:14, Ethernet0/1
O N2     55.55.55.4 [110/20] via 192.168.50.2, 00:01:14, Ethernet0/1
O N2     55.55.55.5 [110/20] via 192.168.50.2, 00:01:14, Ethernet0/1

***physical interface routes omitted, there are alot of them!***
AREA5#
~~~

Here we see that there are still type 3 LSAs, but no type 4 and 5. We also still have a default route, but now it is sent through the area as a type 7 LSA, and shows up in the routing table as an O N2 route!

So when do we use 'no-summary' and when do we use 'default-information-originate'? Generally, use 'no-summary' when a default route from the ABR is required to reach anything within the OSPF domain (including external routes). Use 'default-information-originate' when a default route is needed from the ABR to reach external destinations, but access to other areas may go through other routers.

Lets look at a special case of NSSA. Suppose that the ABR in area 5 was also an ASBR redistributing some extra loopbacks into the OSPF domain. We want these to go into area 0 and the rest of the OSPF domain but not into our NSSA. We can simply use 

~~~
area 5 nssa no-redistribution
~~~

on the ABR/ASBR.

This will NOT flood the redistributed routes into the NSSA domain, and they will show up in area 0 as type 5 LSAs. Here is a quick demonstration:

First I configure a loopback address on the AREA5ABR router, configure it to be advertised by EIGRP, then redistribute into OSPF.

~~~
                Type-5 AS External Link States


20.20.20.20     192.168.105.1   100         0x80000001 0x005824 0
~~~

We see it as a type 5 LSA within area 5, it is also present in the other areas. Now lets configure 'area 5 nssa no-redistribution' on the AREA5ABR router and 'area 5 nssa' on the others.

Now we see 

~~~
AREA5#sh ip ospf database external

            OSPF Router with ID (5.5.5.5) (Process ID 1)
AREA5#
~~~

so there are no type 5 LSAs as predicted, then in the type 7 LSAs:

~~~
                Type-7 AS External Link States (Area 5)

Link ID         ADV Router      Age         Seq#       Checksum Tag
55.55.55.1      55.55.55.5      11          0x80000001 0x001394 0
55.55.55.2      55.55.55.5      11          0x80000001 0x00099D 0
55.55.55.3      55.55.55.5      11          0x80000001 0x00FEA6 0
55.55.55.4      55.55.55.5      11          0x80000001 0x00F4AF 0
55.55.55.5      55.55.55.5      11          0x80000001 0x00EAB8 0
~~~

There is no LSA for the 20.20.20.20 interface. However if we go to the hub router and check its LSDB:

~~~
HUB# sh ip ospf database external
                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
***redacted***
20.20.20.20     192.168.105.1   413         0x80000001 0x005824 0
~~~

We see that this network will be seen by everyone else.

As you can see, these OSPF area types are incredibly versatile, and their usage highly depends on the topology you are working with. I highly encourage everyone reading this to lab this up and try different scenarios. For example what if your internal OSPF network is connected to multiple ABRs? Can you leverage different stub configurations on those ABRs for optimal routing and even some sort of load balancing between the ABRs?

If you have any questions about the content of this post, don't hesitate to [__email__](mailto:villarreal.alan.aav@gmail.com) me or leave a comment in the disqus section below!


