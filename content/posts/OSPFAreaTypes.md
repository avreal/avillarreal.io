---
title: "OSPF Stub Area Types Part 1: Stub and Totally Stubby"
urltitle: "ospfareatypes"
date: 2020-03-28T20:15:03-07:00
draft: true
showdate: true
---

For up and coming network engineers and those studying for their CCNA and CCNP, OSPF can be quite a daunting routing protocol. It's metric is much simpler than EIGRP, being just a function of reference bandwitdh vs link bandwidth. However the inner workings of OSPF can be much more complex, with its link state database filled with LSAs of various types across multiple areas that give routers a whole map of the toplogy on which they run Djikstra's shortest path algorithm to build a routing table. To add to this, there are various configurations that allow for certain LSA types to be filtered at different points in the network. The focus of this post will be those area types (called stub areas) that give network engineers more granular control over the size of their router's link state databases, which in turn eases resource consumption. This post was inspired by Narbik Kocharian's [__video__](https://www.youtube.com/watch?v=cM3OI_ZyRuQ) on the subject, which was instrumental for me in terms of really understanding the concepts and use case of these stub areas while studying for my CCNP. I will demonstrate OSPF Stub Areas using a GNS3 lab and observing the changes in the routing tables, Link State Databases, as well as using packet caputures.   <!--more--> 

Before getting into OSPF discussion, we must first examine the lab environment. This lab is built in GNS3 using Cisco IOU L3 images (i86bi-linux-l3-adventerprisek9-15.4.1T.bin). The full topology is as follows:

![topology](/images/topology.png)

