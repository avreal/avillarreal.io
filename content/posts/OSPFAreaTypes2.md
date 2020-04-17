---
title: "OSPF Stub Area Types Part 2: Not So Stubby Areas"
urltitle: "ospfareatypes2"
date: 2020-03-31T11:06:53-07:00
draft: false
showdate: true
---

The previous post left off with questions about redistribution into OSPF areas which are also classified as stub areas. Recall that stub areas filter type 4 and 5 LSAs, while totally stubby LSAs filter type 3,4, and 5. So what happens if we we wish to redistribute into a stubby area but keep other external routes out, or redistribute xxxxxxxxxxxxxxxx? In this post I will spin up the [__lab__](/1resources/images/ospfstub1/topology.PNG) (gns3 project download [__here__](/1resources/misc/ospfstub1/OSPFStubArea.gns3project)) from the previous post and demonstrate what are called Not So Stubby Areas. <!--more--> 

### Redistribution into Stub Areas

As we saw in the previous post, configuring an area as a stub  was a matter of a single config line in OSPF config mode. Now lets go to area 3 in our topology and suppose we also want that to be a stub, as we don't want external routes from area 4 and 5 in our routing table. Here is the initial link state database (showing only external routes)

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

By configuring area 3 as a stub we stopped type 4 and 5 LSAs as predicted, but even those LSAs from redistribution directly into our area were stopped. As a result, the routers can still reach all external destinations, except the ones being redistributed from the EIGRP3 router! (the EIGRP3 router can still reach them because they are connected routes there) When we configured the ASBR as a stub earlier, that syslog message was warning us that we have configured our ASBR as a stub where no other areas exist, so there are likely going to be adverse effects as a result of this configuration. 

# INSERT IMAGE OF ROUTING TABLE WITH NO ROUTE TO 33.33.33.1 SOMEWHERE HERE

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

So we have access to the networks from the EIGRP domain now, but if you look at the routing table there is no default route as with a stub area! That is because in NSSA the default route to the ABR is optional. With the base NSSA configuration we are allowing the redistributed networks in as type 7 LSAs but not allowing type 4 or 5 LSAs, as well as not generating a default route to the other external destinations. This is because it is possible that there are multiple paths to external destinations so generating a default route could unintentionally cut off access to a portion of the network.

In terms of our lab, configuring area 3 as NSSA means we can reach the EIGRP3 domain, but none of the other EIGRP domains. The next sections show how this can be rectified.

