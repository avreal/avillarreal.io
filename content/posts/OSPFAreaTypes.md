---
title: "OSPF Stub Area Types Part 1: Stub and Totally Stubby"
urltitle: "ospfareatypes"
date: 2020-03-20T20:15:03-07:00
draft: false
showdate: true
---

For up and coming network engineers and those studying for their CCNA and CCNP, OSPF can be quite a daunting routing protocol. It's metric is much simpler than EIGRP, but the inner workings can be much more complex. Think LSAs, LSDBs, Dijkstra's algorithm, etc. Adding to this, there are various configurations that allow for certain LSA types to be filtered at different points in the network and numerous other features. The focus of this post will be those area types (called stub areas) that give network engineers more granular control of the size of their router's link state databases, which in turn eases resource consumption. I am assuming some familiarity with LSA types and basic OSPF concepts so I will not go over that in this discussion, however the effects of stub areas should be clear even without knowing LSAs. This post was inspired by Narbik Kocharians' [__video__](https://www.youtube.com/watch?v=cM3OI_ZyRuQ) on the subject, which was instrumental for me in really understanding the concepts and use cases of these stub areas while studying for my CCNP ROUTE. I will demonstrate OSPF Stub Areas using a GNS3 lab and observing the changes in the routing tables/link state databases. I highly encourage everyone to follow along and lab this as well!   <!--more--> 

Before getting into OSPF discussion, we must first examine the lab environment. This lab is built in GNS3 2.2.3 using IOU L3 images (i86bi-linux-l3-adventerprisek9-15.4.1T.bin). The full topology is as follows (click [__here__](/1resources/images/ospfstub1/topology.PNG) for full size):

![topology](/1resources/images/ospfstub1/topology.PNG)

A couple of notes about the topology:

1. For the purposes of this lab, the individual physical interface IP addresses do not matter, we are mostly concerned with the loopbacks configured on the routers named AreaX as well as the ASBRs.
2. x = {1,5} = {1,2,3,4,5}, so each of these routers with this notation has 5 loopbacks configured
3. The routers called AreaX are used to simulate an internal network in the specified area consisting of several routers participating in OSPF.
4. No static routing is configured.
5. I have created a gns3 portable project, that can be downloaded [__here__](/1resources/misc/ospfstub1/OSPFStubArea.gns3project). 

## Initial State

Let us first look at the initial state of the router called AREA1. As we can see the LSDB is rather large (click [__here__](/1resources/textfiles/ospfstub1/area1lsdb.txt) to see the full output). For this post in particular we are interested in the Type 3,4, and 5 LSAs. Note that the output that I am pasting here is omitting LSAs from the physical links on the network, as we are looking only at the loopbacks configured as described above. That means the LSDB is even larger than what I am showing here (14 extra type 3 LSAs)! The LSDB for Areas 1 and 2 are going to be almost exactly the same, except area 1 is going to have type 3 LSAs for area 2 and vice versa. (In fact the initial LSDB for all of the AREAx routers will be similar)

### Area 1 Type 3 LSA
~~~
                 Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
2.2.2.1         192.168.100.1   17          0x80000004 0x0097AD
2.2.2.2         192.168.100.1   17          0x80000004 0x008DB6
2.2.2.3         192.168.100.1   17          0x80000004 0x0083BF
2.2.2.4         192.168.100.1   17          0x80000004 0x0079C8
2.2.2.5         192.168.100.1   17          0x80000004 0x006FD1
3.3.3.1         192.168.100.1   17          0x80000004 0x0073CE
3.3.3.2         192.168.100.1   17          0x80000004 0x0069D7
3.3.3.3         192.168.100.1   17          0x80000004 0x005FE0
3.3.3.4         192.168.100.1   17          0x80000004 0x0055E9
3.3.3.5         192.168.100.1   17          0x80000004 0x004BF2
4.4.4.1         192.168.100.1   17          0x80000004 0x004FEF
4.4.4.2         192.168.100.1   17          0x80000004 0x0045F8
4.4.4.3         192.168.100.1   17          0x80000004 0x003B02
4.4.4.4         192.168.100.1   17          0x80000004 0x00310B
4.4.4.5         192.168.100.1   17          0x80000004 0x002714
5.5.5.1         192.168.100.1   17          0x80000004 0x002B11
5.5.5.2         192.168.100.1   17          0x80000004 0x00211A
5.5.5.3         192.168.100.1   17          0x80000004 0x001723
5.5.5.4         192.168.100.1   17          0x80000004 0x000D2C
5.5.5.5         192.168.100.1   17          0x80000004 0x000335
10.10.10.1      192.168.100.1   18          0x80000009 0x00082A
10.10.10.2      192.168.100.1   18          0x80000009 0x00FD33
10.10.10.3      192.168.100.1   18          0x80000009 0x00F33C
10.10.10.4      192.168.100.1   18          0x80000009 0x00E945
10.10.10.5      192.168.100.1   18          0x80000009 0x00DF4E

~~~

### Area 1 Type 4 LSA
~~~
                Summary ASB Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
6.6.6.5         192.168.100.1   17          0x80000001 0x0068D9
33.33.33.5      192.168.100.1   17          0x80000001 0x00617B
44.44.44.5      192.168.100.1   17          0x80000001 0x00D3E7
55.55.55.5      192.168.100.1   17          0x80000001 0x004654

~~~

### Area 1 Type 5 LSA
~~~
                Type-5 AS External Link States

Link ID         ADV Router      Age         Seq#       Checksum Tag
6.6.6.1         6.6.6.5         1446        0x80000002 0x005421 0
6.6.6.2         6.6.6.5         1446        0x80000002 0x004A2A 0
6.6.6.3         6.6.6.5         1446        0x80000002 0x004033 0
6.6.6.4         6.6.6.5         1446        0x80000002 0x00363C 0
6.6.6.5         6.6.6.5         1446        0x80000002 0x002C45 0
33.33.33.1      33.33.33.5      1408        0x80000002 0x00F9D8 0
33.33.33.2      33.33.33.5      1408        0x80000002 0x00EFE1 0
33.33.33.3      33.33.33.5      1408        0x80000002 0x00E5EA 0
33.33.33.4      33.33.33.5      1408        0x80000002 0x00DBF3 0
33.33.33.5      33.33.33.5      1408        0x80000002 0x00D1FC 0
44.44.44.1      44.44.44.5      1395        0x80000002 0x00632D 0
44.44.44.2      44.44.44.5      1395        0x80000002 0x005936 0
44.44.44.3      44.44.44.5      1394        0x80000002 0x004F3F 0
44.44.44.4      44.44.44.5      1394        0x80000002 0x004548 0
44.44.44.5      44.44.44.5      1394        0x80000002 0x003B51 0
55.55.55.1      55.55.55.5      1411        0x80000002 0x00CC81 0
55.55.55.2      55.55.55.5      1411        0x80000002 0x00C28A 0
55.55.55.3      55.55.55.5      1411        0x80000002 0x00B893 0
55.55.55.4      55.55.55.5      1411        0x80000002 0x00AE9C 0
55.55.55.5      55.55.55.5      1411        0x80000002 0x00A4A5 0

~~~

### And now we can look at the routing table, Recall NO static routing is configured:

### Inter Area Routes

~~~
AREA1#sh ip route ospf | include O IA
O IA     2.2.2.1 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     2.2.2.2 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     2.2.2.3 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     2.2.2.4 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     2.2.2.5 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     3.3.3.1 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     3.3.3.2 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     3.3.3.3 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     3.3.3.4 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     3.3.3.5 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     4.4.4.1 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     4.4.4.2 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     4.4.4.3 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     4.4.4.4 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     4.4.4.5 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     5.5.5.1 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     5.5.5.2 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     5.5.5.3 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     5.5.5.4 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     5.5.5.5 [110/41] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     10.10.10.1 [110/31] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     10.10.10.2 [110/31] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     10.10.10.3 [110/31] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     10.10.10.4 [110/31] via 192.168.100.1, 00:04:19, Ethernet0/0
O IA     10.10.10.5 [110/31] via 192.168.100.1, 00:04:19, Ethernet0/0
~~~

### External Routes

~~~
AREA1#sh ip route ospf | include O E2
O E2     6.6.6.1 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     6.6.6.2 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     6.6.6.3 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     6.6.6.4 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     6.6.6.5 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     33.33.33.1 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     33.33.33.2 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     33.33.33.3 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     33.33.33.4 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     33.33.33.5 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     44.44.44.1 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     44.44.44.2 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     44.44.44.3 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     44.44.44.4 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     44.44.44.5 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     55.55.55.1 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     55.55.55.2 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     55.55.55.3 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     55.55.55.4 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0
O E2     55.55.55.5 [110/20] via 192.168.100.1, 00:05:35, Ethernet0/0

~~~

## Stub Configuration

 Suppose we want to decrease the size of the routing table by eliminating specific routes to external destinations, as the devices in this area do not care about how they reach them and only have 1 exit point anyway. We still want specific routes to the destinations within other areas in OSPF, but we do not care about specific routes to the EIGRP domain. This can be accomplished by using a stub area.

Configuration is quite simple, we just do

~~~
(config-router)#area 1 stub
~~~

on all routers in area 1. When we do this on the AREA1ABR router first but not AREA1 the OSPF adjacency goes down. This is because the routers must agree on whether they will allow external routing to form the adjacency. Lets look at the OSPF hello packet once this is done.

Here is a piece of the hello packet from the router that does NOT have 'area 1 stub' configured:

![nostub](/1resources/images/ospfstub1/nostubpcap.PNG)

Now the packet from the router that does have 'area 1 stub' configured:

![stub1](/1resources/images/ospfstub1/stubpcap.PNG)

Note the difference in the E bit. Once I set 'area 1 stub' on the AREA1 router the adjacency forms again and we can observe what has changed.

~~~
AREA1#sh ip ospf database external

            OSPF Router with ID (1.1.1.5) (Process ID 1)

AREA1#
~~~

~~~
AREA1#sh ip route ospf | include O E2
AREA1#
~~~

What happened to the external routes and type 4/5 LSAs? Can we still reach those external networks?

The answer is YES, but how is this accomplished?

When we configure area 1 as a stub, we are essentially putting the effort of routing to external destinations from this area solely on the ABR. This is accomplished with a special type 3 LSA propagated throughout the area. Let us take a look at the type 3 LSAs in the database:

~~~
                Summary Net Link States (Area 1)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         192.168.100.1   27          0x80000001 0x00E08E
2.2.2.1         192.168.100.1   27          0x80000007 0x00AF94
2.2.2.2         192.168.100.1   27          0x80000007 0x00A59D
2.2.2.3         192.168.100.1   27          0x80000007 0x009BA6
2.2.2.4         192.168.100.1   27          0x80000007 0x0091AF
2.2.2.5         192.168.100.1   27          0x80000007 0x0087B8
3.3.3.1         192.168.100.1   27          0x80000007 0x008BB5
3.3.3.2         192.168.100.1   27          0x80000007 0x0081BE
3.3.3.3         192.168.100.1   28          0x80000007 0x0077C7
3.3.3.4         192.168.100.1   28          0x80000007 0x006DD0
3.3.3.5         192.168.100.1   28          0x80000007 0x0063D9
4.4.4.1         192.168.100.1   28          0x80000007 0x0067D6
4.4.4.2         192.168.100.1   28          0x80000007 0x005DDF
4.4.4.3         192.168.100.1   28          0x80000007 0x0053E8
4.4.4.4         192.168.100.1   28          0x80000007 0x0049F1
4.4.4.5         192.168.100.1   28          0x80000007 0x003FFA
5.5.5.1         192.168.100.1   28          0x80000007 0x0043F7
5.5.5.2         192.168.100.1   28          0x80000007 0x003901
5.5.5.3         192.168.100.1   28          0x80000007 0x002F0A
5.5.5.4         192.168.100.1   28          0x80000007 0x002513
5.5.5.5         192.168.100.1   28          0x80000007 0x001B1C
~~~

Looks identical to the initial state except for the first entry, what is this 0.0.0.0 LSA telling us? We can take a closer look:

~~~
AREA1#sh ip ospf database summary

            OSPF Router with ID (1.1.1.5) (Process ID 1)

                Summary Net Link States (Area 1)

  Routing Bit Set on this LSA in topology Base with MTID 0
  LS age: 108
  Options: (No TOS-capability, DC, Upward)
  LS Type: Summary Links(Network)
  Link State ID: 0.0.0.0 (summary Network Number)
  Advertising Router: 192.168.100.1
  LS Seq Number: 80000001
  Checksum: 0xE08E
  Length: 28
  Network Mask: /0
        MTID: 0         Metric: 1
~~~

Network number of 0.0.0.0 and a network mask of 0. This looks like a default route! We can verify by looking at the routing table:

~~~
AREA1#show ip route

Gateway of last resort is 192.168.100.1 to network 0.0.0.0

O*IA  0.0.0.0/0 [110/11] via 192.168.100.1, 00:03:09, Ethernet0/0
~~~

The AREA1ABR router has created and flooded a type 3 LSA which results in the creation a default route to all external destinations through itself. We were able to cut the routing table and LSDB of non ABR area 1 routers in half with 2 configuration lines!

To summarize, in a stub area type 4 and 5 LSAs will not be propagated by the ABR. Instead the ABR will generate a type 3 LSA specifying a default route and propagate that. Now all of the non ABR routers in area 1 will have a single default route to external networks that travels through the ABR.

## Totally Stubby Configuration

Now suppose that we wish to decrease the size of the database and the routing table even further, as our poor old router is starting to struggle with the overhead of running OSPF, but we don't want to manage static routes. OSPF gives us a method of not only stopping type 4/5 LSAS, but also limiting the propagation of type 3 LSAs. This is called a Totally Stubby Area.

Lets hop over to area 2 for this demonstration. The LSDB is going to be nearly identical to the initial state I showed above, except IA routes to 2.2.2.2 instead are to 1.1.1.1 (in area 1).

The configuration is quite similar, except on the ABR we add 'no-summary' (as in no summary LSA, you can see where this is going) to the command.

~~~
AREA2ABR(config-router)#area 2 stub no-summary
~~~

and on the regular AREA2 router (and all other routers if they were there) we configure

~~~
AREA2(config-router)#area 2 stub
~~~

as before.

Now lets look at the OSPF database in area 2.

~~~
AREA2#sh ip ospf database

            OSPF Router with ID (2.2.2.5) (Process ID 1)

                Router Link States (Area 2)

Link ID         ADV Router      Age         Seq#       Checksum Link count
2.2.2.5         2.2.2.5         104         0x80000008 0x006584 6
192.168.102.1   192.168.102.1   105         0x80000007 0x00429F 1

                Net Link States (Area 2)

Link ID         ADV Router      Age         Seq#       Checksum
192.168.102.1   192.168.102.1   100         0x80000006 0x005D5F

                Summary Net Link States (Area 2)

Link ID         ADV Router      Age         Seq#       Checksum
0.0.0.0         192.168.102.1   168         0x80000001 0x00D29A
~~~

Is there a bug, did all the other routers die? NO, by configuring area 2 as a totally stubby area the ABR is NOT flooding type 3,4, or 5 LSAs within the area. Instead, as with the stub area we have a special 0.0.0.0 Type 3 LSA,

~~~
AREA2#show ip ospf  database summary

            OSPF Router with ID (2.2.2.5) (Process ID 1)

                Summary Net Link States (Area 2)

  Routing Bit Set on this LSA in topology Base with MTID 0
  LS age: 359
  Options: (No TOS-capability, DC, Upward)
  LS Type: Summary Links(Network)
  Link State ID: 0.0.0.0 (summary Network Number)
  Advertising Router: 192.168.102.1
  LS Seq Number: 80000001
  Checksum: 0xD29A
  Length: 28
  Network Mask: /0
        MTID: 0         Metric: 1
AREA2#

~~~

in fact this is the only type 3 LSA being flooded into the area, which results in a route that directs all area 2 routers to send all traffic not otherwise specified towards the ABR. Now compare the LSDB after configuring area 2 as totally stubby to the [__initial LSDB of area 2__](/1resources/textfiles/ospfstub1/area2lsdb.txt) and we have drastically reduced the OSPF overhead on the routers within area 2.

At this point I hope this has cleared up stub areas and totally stubby areas, however the configuration above should raise some questions. For example, what if we wanted to configure an area with an ASBR as a stub? What if we wish to redistribute directly into a stub but not allow external routes from other areas? In my next post I will discuss the extension to the classic stubby areas, known as Not So Stubby Areas (seriously).

If you have any questions about the content of this post, don't hesitate to [__email__](mailto:villarreal.alan.aav@gmail.com) me or leave a comment in the disqus section below!
