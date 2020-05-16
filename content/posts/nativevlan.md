---
title: "VLAN Hopping Over Trunks: Why Native VLAN Mismatches Matter "
urltitle: "nativevlan"
date: 2020-04-05T11:06:53-07:00
draft: false
showdate: true
---

When learning about VLANs and layer 2 segmentation, we are always told that the purpose of VLANs is to separate broadcast domains, and traffic from one VLAN can not go to the other VLAN unless specifically allowed. However there are situations in which traffic from one VLAN can leak across to another VLAN which is a massive security risk. In this post I will show an example of VLAN hopping in the case of a native VLAN mismatch, and discuss the native VLAN in general. Note that this is geared more towards Cisco IOS, as other vendors may not implement the native VLAN in their switches. <!--more--> 

Before getting into it, lets first look into our lab setup. Click [__here__](/1resources/images/nativevlan/nativevlan.PNG) for full size.

![nativevlanlab](/1resources/images/nativevlan/nativevlan.PNG)

I am using the vios_l2-adventerprisek9-m.03.2017.qcow2 image here, as IOU images have some unexpected behavior when it comes to VLAN tagging. I have created a portable project that can be downloaded [__here__](/1resources/misc/nativevlan/nativevlan.gns3project) (you must have your own IOSv image).


We simply have 3 L3 Switches, each with VLANs 2-3 configured and a PC connected, then a DHCP server. The switches have dot1q trunks configured between them, all with default settings. The PCs are numbered according to the access VLAN they are plugged into. 

Before discussing VLAN Hopping, I'll quickly go over the function of the Native VLAN.

Conventional knowledge tells us that currently, only the PC on VLAN1 will successfully pull an IP address from the DHCP server (which is also on VLAN1). We can verify this as follows:

~~~
PC1> ip dhcp
DDORA IP 192.168.1.2/24 GW 192.168.1.1
PC1>
~~~

~~~
PC2> ip dhcp
DDD
Can't find dhcp server
PC2>
~~~

~~~
PC3> ip dhcp
DDD
Can't find dhcp server
PC3>
~~~

At this point PC1 is able to reach the DHCP server on layer 2 and obtain an IP address but not the other PCs. Now I will set PC2 and PC3 statically on the same layer 3 network. Note that we are not able to ping them as there is still no layer 2 visibility between PC1 and PC2/PC3.

PC2 - 192.168.1.20/24

PC3 - 192.168.1.30/24

~~~
PC1> ping 192.168.1.20 (PC2)
PC1> ping 192.168.1.20) not reachable

PC1> ping 192.168.1.30 (PC3)
host (192.168.1.30) not reachable
PC1>
~~~

Lets take a deeper dive into whats happening on the trunk links when the PCs attempt to send traffic to each other on different VLANs. 

When trying to ping from PC1 to PC2/PC3, first an ARP broadcast is sent out. Lets look at the traffic on the trunk from Gi0/0 on S1.

![pcap1](/1resources/images/nativevlan/pcap1.PNG)

Looks like a normal ARP request, however there is no response and the pings eventually fail. Now lets ping from PC2 to PC1 and look at the ARP request on the trunk from Gi0/0 on the S2 side.

![pcap1](/1resources/images/nativevlan/pcap2.PNG)

It appears we have a new field, an 802.1Q VLAN tag! As before, there is no reply to the ARP request and the pings fail. Let's break down what is happening on the switches.

1. S2 receives broadcast frame from PC2 on Gi1/1 which is an access port on VLAN2.
2. S2 then floods it out of all ports on VLAN2 as well as the trunks allowing VLAN2.
3. On the trunks, S2 adds a VLAN tag to the frame because it originated from a non-native VLAN. This explains why the frames from S1 did not have this tag.
4. Now over to S1 and S3 who have both received this frame. Since the frame has a VLAN tag with an ID of 2, these switches will remove the tag and know to only flood it access ports on VLAN2 or trunks allowing VLAN2.

Or visually,

![ex1](/1resources/images/nativevlan/nativevlan4.PNG)

If we did a pcap on the link between PC1 and S1 we would not see the ARP broadcast when PC2 tries to ping it. Now I have added a dummy PC on vlan2 of S1 (lets call it PCx). When we try to ping from PC2 to PC1, we can see the ARP request that was broadcast by PC2.

![pcap3](/1resources/images/nativevlan/pcap3.PNG)

As we can see, there is no VLAN tag as it has been removed by S1 before flooding it to VLAN2.

So what is the purpose of the native VLAN? Essentially, the native VLAN just tells the trunk which frames it will tag. If the traffic originates from the native VLAN, it will be sent out of the trunk untagged. If it comes from a non-native VLAN, the frame is tagged and sent out. When the frame is received on the other end of the trunk, the switch will check for a VLAN tag. If there is no vlan tag it assumes that the traffic is meant to go to the native vlan and will forward it to the ports there, otherwise it will forward it to the access VLAN corresponding to the tag (and other trunks if its a broadcast). Here is a simple table showing this:

| Access VLAN | Native VLAN Y           | Native VLAN X           |
|:-----------:|-------------------------|-------------------------|
|      X      | Tagged With X on Egress | Untagged                |
|      Y      | Untagged                | Tagged with Y on Egress |

## Native VLAN Configuration and VLAN Hopping

Now lets look at the native VLAN configuration and ask the question, why should we mess with the native VLAN in the first place? Generally it is considered a security best practice to configure the native VLAN throughout the network as some unused VLAN. This ensures that devices plugged into unconfigured ports wont be able to see interact with other devices and protocols running on VLAN1, as well as mitigating various attacks. However, if care is not taken when changing the native VLAN, unintended consequences can occur. Lets go to S2 and change the native vlan to 2.

~~~
Switch(config-if)#switchport trunk native vlan 2
~~~

NOTE THAT I AM DOING THIS CONFIGURATION IN INTERFACE CONFIG MODE, THE NATIVE VLAN IS CONFIGURED ON A PER TRUNK BASIS, NOT PER SWITCH BASIS.

A couple of things happen once this configuration is done.

~~~
00:030002. Inconsistent local vlan.
00:29.335: %SPANTREE-2-RECV_PVID_ERR: Received BPDU with inconsistent peer vlan id 1 on GigabitEthernet0/0 VLAN2.
00:03:29.368: %SPANTCK_PVID_PEER: Blocking GigabitEthernet0/0 on VLAN0001. Inconsistent peer vlan.
00:03:29.382: %SPANTREE-2-BLOCK_PVID_LOCAL: Blocking GigabitEthernet0/0 on VLAN

00:03:47.979: %CDP-4-NATIVE_VLAN_MISMATCH: Native VLAN mismatch discovered on GigabitEthernet0/0 (2), with Switch GigabitEthernet0/0 (1).
~~~

Spanning tree has noticed that there is a native VLAN mismatch on the trunk and put the port into a PVID inconsistent state. If we check the output of 'show spanning-tree' we see the following on both VLAN 1 and 2:

~~~
Interface           Role Sts Cost      Prio.Nbr Type
------------------- ---- --- --------- -------- --------------
Gi0/0               Desg BKN*4         128.1    P2p *PVID_Inc
~~~

We also see a message from CDP notifying us about the mismatch, but CDP does not take any corrective action.

So after all this talk about the dangers of native VLAN mismatch, spanning tree has jumped in and fixed the problem! However, STP is not always running, suppose the network admin turned it off for whatever reason. Lets see what happens when we turn STP off on ALL of the trunks in our network. For this we will use 'spanning-tree bpdufilter enable' at the interface level.

~~~
*Apr 26 00:13:19.983: %SPANTREE-2-UNBLOCK_CONSIST_PORT: Unblocking GigabitEthernet0/0 on VLAN0001. Port consistency restored.
*Apr 26 00:13:19.986: %SPANTREE-2-UNBLOCK_CONSIST_PORT: Unblocking GigabitEthernet0/0 on VLAN0002. Port consistency restored.
~~~

As we can see VLAN1 and VLAN2 have been unblocked on the trunk, and we have willingly kept our native VLAN mismatch. Now lets ping from PC1 to PC2.

~~~
PC1> ping 192.168.1.20
84 bytes from 192.168.1.20 icmp_seq=1 ttl=64 time=17.194 ms
84 bytes from 192.168.1.20 icmp_seq=2 ttl=64 time=11.901 ms
PC1> 
~~~

Remember, PC1 is on access VLAN1 on S1, and PC2 is on access VLAN2 on PC2. We now have access to a device on a different broadcast domain! Lets take a closer look at the traffic.

1. We enter a ping command on PC1 to the IP address of PC2, I cleared the CAM and ARP tables so an ARP broadcast is sent first. 
2. The frame from PC1 to S1 is untagged as expected.
3. Since PC1 is on VLAN1, and VLAN1 is the native VLAN on the Gi0/0 trunk, the frame is untagged coming out of Gi0/0 of S1.
4. S2 receives this untagged frame on its Gi0/0 interface, since the Native VLAN here is VLAN2, the frame is flooded out of VLAN2.
5. Since the frame was flooded out of VLAN2, PC2 is able to see it and respond.

Or visually, 

![ex2](/1resources/images/nativevlan/nativevlan3.PNG)

This is evident in the packet capture, where both the request (from S1 to S2) and the reply (from S2 to S1) travel across the trunk untagged, even though the devices generating these frames are on different access VLANS.

ARP Request

![pcap4](/1resources/images/nativevlan/pcap4.PNG)

ARP Response

![pcap5](/1resources/images/nativevlan/pcap5.PNG)

So we can now go from VLAN1 to VLAN2 and back, but apply the same logic above to traffic from VLAN1 on S1 to VLAN1 on S2. Such traffic will no longer work! Now think of the ramifications of this. There are a number of combinations of native vlan configurations that allow for hopping over VLANs. Suppose the network was configured as follows:

* S1 Gi0/0 - native vlan 1
* S2 Gi0/0 and 0/1 - native vlan 2
* S3 Gi0/0 and 0/1- native vlan3
* DHCP Server - plugged into vlan 3
* PC1 - Vlan1
* PC2 - Vlan2
* PC3 - Vlan3

Can you think of how traffic will be able to flow throughout the network?

![ex3](/1resources/images/nativevlan/nativevlan2.PNG)

1. All 3 PCs will be able to reach the DHCP server and pull an IP
2. All 3 PCs have full access to each other
3. None of the frames from the PCs will be tagged on the trunks.

This could be disastrous in a production network, where an attacker will check for every possible opening and exploit it. So in the event of such a configuration, what is in place to mitigate this? As we have seen before spanning tree will take care of this but this kills all traffic on the mismatched VLANs, and what if the network admin doesn't want to use spanning-tree for some reason? In this case we can also use the 'vlan dot1q tag native' command. (Of course we can just NOT have native vlan mismatches but mistakes happen :) )

## vlan dot1q tag native

So we know that traffic originating from a VLAN other than the native VLAN will be sent through a trunk with an 802.1q tag holding a VLAN ID. Traffic belonging to the native VLAN will be sent out untagged. However by using the following command in global config mode

~~~
(config)#vlan dot1q tag native
~~~

All traffic sent out from the trunks on the switch will be tagged, even the native vlan. Another effect of this is that ALL UNTAGGED TRAFFIC RECEIVED ON A TRUNK ON THAT SWITCH WILL BE DROPPED (besides control traffic such as CDP etc). Note that this command is configured in global config mode, whereas the native VLAN is configured per trunk interface. Can you think of what effects this will have if we configure this on S1, following the same setup as the previous section?

The first effect is that traffic from access ports on VLAN1 will have their frames tagged on the trunk leaving Gi0/0 of S1! We can see this here:

![pcap5](/1resources/images/nativevlan/pcap5.PNG)


When this is done, devices on VLAN1 across all 3 switches can see each other, but you can no longer VLAN hop! Lets go through step by step from PC2 to PC1.

1. PC2 sends an ARP request broadcast which is received on access VLAN2
2. Since the native VLAN is 2, the frame is sent out of Gi0/0 on S2 untagged
3. Since we configured 'vlan dot1q tag native' on S1, the frame is dropped on the trunk as it comes in!

Now lets follow from PC1 to PC2

1. PC1 sends an ARP request broadcast which is received on access VLAN1
2. The native VLAN is 1, so the frame would normally be sent untagged, however since 'vlan dot1q tag native' is configured the frame is sent with a tag specifying a vlan ID of 1.
3. The frame is received on Gi0/0 of S2, where the native vlan is 2. Since it has a tag of 1, the tag is stripped and the frame is flooded out of VLAN1.
4. There aren't any devices on VLAN1 to receive the frame so nothing happens.

Now we can update our table for the case of the native VLAN being tagged:

| Access VLAN | Native VLAN Y and Native Tagged | Native VLAN X and Native Tagged |
|:-----------:|---------------------------------|---------------------------------|
|      X      |     Tagged With X on Egress     |       Tagged with X on Egress   |
|      Y      |     Tagged With Y on Egress     |       Tagged with Y on Egress   |

## In Conclusion 

As we have seen, even a slight mismanagement of the layer 2 network can have disastrous results and security ramifications. This is just one form of VLAN hopping, which is simply one type of attack in the vast landscape of network exploits.  I highly encourage everyone to lab these scenarios, as well as test out various combinations of these configurations in your own lab. Examining packet captures is an excellent way to gain a greater understanding of whats happening at every step of a traffic flow and how your configurations affect them!

If you have any questions about the content of this post, don't hesitate to [__email__](mailto:villarreal.alan.aav@gmail.com) me or leave a comment in the disqus section below!