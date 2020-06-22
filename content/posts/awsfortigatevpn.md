---
title: "AWS/On-Prem Fortigate Site to Site VPN"
date: 2020-05-01T10:39:51-07:00
draft: false
urltitle: "awsfortigatevpn"
showdate: true
---


<!---
1. Intro
2. Diagram
3. New VPC
4. Internet Gateway -> Subnets -> Route table
5. AWS Fortigate Install/Setup (security group) (mention charges)
6. Bind elastic IP to the public subnet ENI of the fortigate
7. Create private network interface (ENI) for the fortigate and bind to fortigate
10. Spin up EC2 INstances in the private subnet (security group)
8. Create site to site VPN on AWS forigate (P1, P2, route, policy)
9. Create site to site VPN on local fortigate (DMZ on modem) (P1, P2, route, policy)
11. attempt to ping from local machine to EC2 to bring it up
11.5. SSH into the EC2 instance
12. Show that private subnet can't ping out unless a route is given to it
--->

Part of the curriculum for the AWS Solutions Architect Associate Exam includes a brief look at the VPC and its features, but it is by no means a comprehensive overview of networking in AWS. As someone who works in the networking field and has spent significant personal time labbing, I wanted to get my hands dirty with AWS networking. In this post I'll go over the process of setting up a site to site VPN between my home FortiGate firewall and a FortiGate EC2 instance with a public/private subnet VPC setup, with the intent of simulating a connection between an on prem network and an AWS infrastructure. I was pleasantly surprised at how intuitive this was, given that I only have experience with non cloud FortiGates, and had a ton of fun setting this up. If you have a FortiGate sitting around and are willing to spend a few cents on EC2, follow along!
  <!--more--> 

First, let's look at the whole network setup: (click [__here__](/1resources/images/awsvpn/AWS-Fortigate.png) for full size):

![diagram](/1resources/images/awsvpn/AWS-Fortigate.png)

Quick Summary:

On the AWS side we have a private subnet which will have no access to the outside world. This subnet will host EC2 instances that we can suppose are running some critical backend for an application. The private subnet is connected via an ENI to the FortiGate instance, then another ENI on the instance connects to the public subnet. The public subnet is also connected to an internet gateway for WAN access. My home router is NOT bridged, so I will be using NAT traversal on the VPN, and have set up my fortigate as a DMZ host in my ISP router settings. The purpose of this VPN configuration is to allow a host on my "on prem" network to securely access the private subnet in my VPC. __NOTE: I AM ASSUMING SOME FAMILIARITY WITH AWS AND FORTIGATE SO I WON'T SHOW EVERY SINGLE STEP!!__ Now, onto the step by step!

## Creating A New VPC

For the purpose of this lab, we will create a new VPC. 

![createvpc](/1resources/images/awsvpn/createvpc.PNG)

Nothing fancy here, just using a different private IPv4 address space than our default VPC. From this point on I will use the "filter by VPC" option on the sidebar so I only make changes pertaining to my new VPC.

## Creating Public/Private Subnets, Internet Gateway, and Routing

For this simple architecture, I will use 10.0.1.0/24 as the internal IP for my public subnet and 10.0.100.0/24 for the private subnet. This following screenshot shows what the subnet creation screen looks like (Note: the error displayed is because I had already created this subnet in my VPC before writing this post). Also, for the private subnet, we want to make sure that the "Auto-assign public IPv4" option is not selected.

![createsubnet](/1resources/images/awsvpn/createsubnet.png)

Now that we have our subnets, I created an internet gateway to allow access to the internet from the public subnet. This is super simple - it's just a matter of naming the gateway and attaching it to the VPC so I am not showing it here. Once the internet gateway is attached, we can start building our route tables. Any route tables we create will automatically include a local route which allows access to other devices within the VPC. I took the default route table in the VPC and renamed it "PublicSubnet". I then added a default route pointing to my internet gateway. Here is what that looks like:

![publicroute](/1resources/images/awsvpn/publicroute.PNG)

Then I create a second route table for my private subnet that looks like this:

![privroute](/1resources/images/awsvpn/privroute.PNG)

Note that I did nothing except give it a name, as the only route we want in the private subnet is the local route.

Once the route tables are created, we need to associate them with the subnet(s) in our VPC. This is easily done from the subnet configuration as shown below:

Private:

![privsubassoc](/1resources/images/awsvpn/privsubassoc.PNG)

Public: 

![pubsubassoc](/1resources/images/awsvpn/pubsubassoc.PNG)

Note that I have 2 of each public/private subnets for availability reasons, but this lab will just explore using one subnet in the VPN.

So now we have our 2 subnets - one that only has local access and one that has access to the outside world. Time to spin up a fortigate in our VPC and get it connected to our subnets!

## FortiGate EC2 Instance

The process for setting up the Fortigate EC2 instance is not much different than setting up a regular free tier t2.micro instance. We must go to the AWS Marketplace and subscribe for access to the FortiGate AMI (click [__here__](https://aws.amazon.com/marketplace/pp/Fortinet-Inc-Fortinet-FortiGate-Next-Generation-Fi/B00PCZSWDA) to access the marketplace page). At the time this post was written (May 2020), Fortinet was offering a 30 day free trial for their FortiGate platform. Note that "free" here means that the AMI/software itself is free, but you will still be charged the standard price by AWS for the type of instance you use. In my case I am using the smallest possible instance for FortiGate which is a t2.small that runs $0.023 an hour. There are a lot of options in the EC2 creation process, but for this lab we only need to configure the following:

1. Select our lab VPC
2. Set PublicSubnet as the subnet 
3. The internal IP can be automatically chosen but I have used a static IP of 10.0.1.254 as it is personal preference. (see image below showing 1,2, and 3)
4. Use a security group that is wide open for inbound and outbound access (this is not considered best practice but we can worry about hardening the network later). This security group will also be used later for our private instances.

![instancenetwork](/1resources/images/awsvpn/instancenetwork.PNG)

Now that this is done, we have a t2.small FortiGate EC2 instance spinning up. As you know, our EC2 instance will have a public IP that will allow us to reach it over the internet. Since I have all of my elastic IPs available, I will allocate and then associate an elastic IP with my instance so the IP doesn't change and bring my VPN down (click [__here__](/1resources/images/awsvpn/elasticip.PNG) for full size).

![elasticip](/1resources/images/awsvpn/elasticip.PNG)

Now when I go to https://3.133.25.16 I am greeted with a login screen for my EC2 FortiGate firewall. Following the instructions in the "Usage Instructions" tab, you can log in and then change the password to your liking. Looking at network -> interfaces we see that port 1 has the IP I assigned during the instance creation.

![fortigateint1](/1resources/images/awsvpn/fortigateint1.PNG)

However, at this point the private subnet is not connected to the FortiGate. To do this we will create a new Elastic Network Interface (ENI) and attach it to our instance. This is done in the EC2 sidebar by clicking on Network Interfaces, then on 'Create Network Interface'. I will use the following settings:

![newinterface](/1resources/images/awsvpn/newinterface.PNG)

Once this is done, we can go back to our instance and right click and select Networking -> Attach Network Interface. Then, we select the interface that was just created. Now we can go back to our FortiGate and see that port2 is now there, but does not have an IP address. We must go into the port settings and set it for DHCP (so that it pulls the IP we set up in the ENI). After doing that we see the following:

![fortigateint1](/1resources/images/awsvpn/fortigateinterfaces.PNG)

Now port1 is connected to our public subnet, and port2 is connected to our private.

## EC2 Instances On Private Subnet

Now that the fortigate is staged for our VPN setup, I will spin up two t2.micro Amazon Linux EC2 instances (free tier!) in the private subnet at 10.0.100.100 and 10.0.100.101, these will be the devices I will try to hit over the VPN. This step should be quite simple so I will just say that we must configure them to be in the private network and specify the IP as shown here (click [__here__](/1resources/images/awsvpn/ec2network.png) for full size):

![ec2network](/1resources/images/awsvpn/ec2network.png)

I will configure them with the same wide open security group the FortiGate is using, and all other settings will be default. Because these are private instances with no internet access, I actually have no way of getting into them to take a look unless I add routes to give them outbound access. This is ok as once the VPN is up, we can SSH into them from the on prem network.

## Site to Site VPN Configuration (AWS)

Now that the EC2 FortiGate is ready to go, we can start the site to site VPN configuration. This lab makes use of a route based VPN, so we will need the following:

1. Phase 1 Config
2. Phase 2 Config
3. Firewall Policy
4. Static Routes

The VPN is built on the public subnet as it must be allowed to go out to the public internet - we can then use routing to control the traffic flow from the private subnet out over the VPN.

### AWS Fortigate VPN Config

The Phase 1 and Phase 2 looks like this (click [__here__](/1resources/images/awsvpn/awsfortigateVPNconfig.png) for GUI config):

~~~
config vpn ipsec phase1-interface
    edit "AlanVPN"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal aes256-sha256
        set dhgrp 5
        set remote-gw *REDACTED*
        set psksecret ENC ujOXAy6XC/6N0wHTwt5gIMozjYS4SuTASEVAB2N6hyqK58pd9k83fQzE/e/rbG6rKwWEWRAHtE1DQ9uV0yOQ2+YxuIGZFT/Oc/z+GXZUqL9Scmlo8ajADWxpjYjm82lyUjLd/nZtZXwi9lUORVMyS0oEk2/d0wJMu4QUlq5b5w/k4D3ZR0ck55yXt8jJt+kE8+vdBg==
    next
end
config vpn ipsec phase2-interface
    edit "AlanVPN"
        set phase1name "AlanVPN"
        set proposal aes256-sha256
        set dhgrp 5
        set src-subnet 10.0.100.0 255.255.255.0
        set dst-subnet 192.168.100.0 255.255.255.0
    next
end
~~~

Next, we configure policies for the VPN in/out (click [__here__](/1resources/images/awsvpn/fortigateawspolicy.png) for full size):

![fortigateawspolicy](/1resources/images/awsvpn/fortigateawspolicy.PNG)

Then, we configure a static route for access to the network on my home FortiGate:

![fortigateawsroute](/1resources/images/awsvpn/fortigateawsroute.PNG)

### Local Fortigate VPN Config

Now we must configure the VPN on our home firewall in a similar fashion. The Phase 1 and Phase 2 looks like this (click [__here__](/1resources/images/awsvpn/localfortigateVPNconfig.png) for GUI config):

~~~
config vpn ipsec phase1-interface
    edit "AlanVPN"
        set interface "wan1"
        set peertype any
        set net-device disable
        set proposal aes256-sha256
        set dhgrp 5
        set remote-gw 3.133.25.16
        set psksecret ENC Dz+bkPOt5KNoqYE5TfcqvGaDkkLwY/L1aK42yWDnb6G04NLS5COHYfFeSpmItT8hQB1dk19TiBDEZbyT59MmlSclyxjin9pL/Eq29TzVRwuykvw3X5AYws2KZG2gDI9qRnF1rytuByKzdNwGA1M+cZBAklYEpoXemoSC+U9kvuv6vub05H2pwu2t0bkTqUuQF8H3vg==
    next
end
config vpn ipsec phase2-interface
    edit "AlanVPN"
        set phase1name "AlanVPN"
        set proposal aes256-sha256
        set dhgrp 5
        set src-subnet 192.168.100.0 255.255.255.0
        set dst-subnet 10.0.100.0 255.255.255.0
    next
end
~~~

Next, we configure policies for the VPN in/out (click [__here__](/1resources/images/awsvpn/fortigatelocalpolicy.png) for full size):

![fortigatelocalpolicy](/1resources/images/awsvpn/fortigatelocalpolicy.PNG)

Then, we configure a static route for access to the private network on the EC2 FortiGate:

![fortigatelocalroute](/1resources/images/awsvpn/fortigatelocalroute.PNG)

Once this is complete on both sides, we can bring the VPN up by sending traffic through it. Let's do a ping from my laptop (192.168.100.2s) to the instance with IP 10.0.100.100.


### Ping From PC to EC2 Instance On Private Subnet
~~~
C:\Users\Alan>ping 10.0.100.100

Pinging 10.0.100.100 with 32 bytes of data:
Request timed out.
Reply from 10.0.100.100: bytes=32 time=24ms TTL=253
Reply from 10.0.100.100: bytes=32 time=23ms TTL=253
Reply from 10.0.100.100: bytes=32 time=24ms TTL=253

Ping statistics for 10.0.100.100:
    Packets: Sent = 4, Received = 3, Lost = 1 (25% loss),
Approximate round trip times in milli-seconds:
    Minimum = 23ms, Maximum = 24ms, Average = 23ms

C:\Users\Alan>
~~~

The first ping drops because the VPN was not up yet, but success! I am now able to reach my private subnet instances from my home network. Now let's SSH into our private instance from my home network using putty:

![putty](/1resources/images/awsvpn/putty.PNG)

Note that I am using a PRIVATE IP instead of a public to access my EC2 instance - This is due to the EC2 instance being on a private subnet that does not have a public facing IP.

~~~
Using username "ec2-user".
Authenticating with public key "imported-openssh-key"
Last login: Fri May 15 23:27:55 2020 from 10.0.100.254

       __|  __|_  )
       _|  (     /   Amazon Linux 2 AMI
      ___|\___|___|

https://aws.amazon.com/amazon-linux-2/
[ec2-user@ip-10-0-100-100 ~]$

~~~

And we're in! Let's run a few commands to demonstrate the environment in which this instance resides:

~~~
[ec2-user@ip-10-0-100-100 ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 0 received, 100% packet loss, time 2041ms

[ec2-user@ip-10-0-100-100 ~]$
~~~

As discussed before, the private subnet has no outbound internet access, so pings to 8.8.8.8 fail as expected.

~~~
[ec2-user@ip-10-0-100-100 ~]$ ping 10.0.1.254
PING 10.0.1.254 (10.0.1.254) 56(84) bytes of data.
64 bytes from 10.0.1.254: icmp_seq=1 ttl=255 time=0.361 ms
64 bytes from 10.0.1.254: icmp_seq=2 ttl=255 time=0.500 ms
64 bytes from 10.0.1.254: icmp_seq=3 ttl=255 time=0.416 ms
^C
--- 10.0.1.254 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2048ms
rtt min/avg/max/mdev = 0.361/0.425/0.500/0.061 ms
[ec2-user@ip-10-0-100-100 ~]$
~~~

We are still able to ping across to the public subnet within the VPC.

~~~
[ec2-user@ip-10-0-100-100 ~]$ ping 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
^C
--- 192.168.100.2 ping statistics ---
21 packets transmitted, 0 received, 100% packet loss, time 20480ms

[ec2-user@ip-10-0-100-100 ~]$
~~~

Pings to 192.168.100.2 (the laptop I am using to access the EC2 instance in the first place) fail! Even though my laptop can reach the instance and receive return traffic (because we are using a stateful firewall), the instance can not initiate a session to my laptop. This is because the routing table in the Private Subnet does not allow this traffic to leave the VPC! Recall from earlier we only created a local route to 10.0.0.0/16 within the private subnet. If for some reason we wanted the instances on the private subnet to reach my laptop, we just need to add a route!

![newroute](/1resources/images/awsvpn/newroute.PNG)

The target of the route is the ENI corresponding to port 2 on the FortiGate, which acts like the gateway for the private subnet when it needs outbound access. Traffic sent to this ENI will then be passed through the port2 -> AlanVPN firewall policy as traffic will go from instance -> ENI -> VPN.

~~~
[ec2-user@ip-10-0-100-100 ~]$ ping 192.168.100.2
PING 192.168.100.2 (192.168.100.2) 56(84) bytes of data.
64 bytes from 192.168.100.2: icmp_seq=1 ttl=126 time=23.6 ms
64 bytes from 192.168.100.2: icmp_seq=2 ttl=126 time=23.7 ms
64 bytes from 192.168.100.2: icmp_seq=3 ttl=126 time=24.1 ms
64 bytes from 192.168.100.2: icmp_seq=4 ttl=126 time=23.4 ms
^C
--- 192.168.100.2 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 23.475/23.737/24.128/0.308 ms
[ec2-user@ip-10-0-100-100 ~]$
~~~

We now have a bidirectional VPN. One cool feature of the EC2 FortiGate firewall is that you can store logs on its EBS disk, so there is very little delay in viewing logs (as opposed to fortianalyzer). We can see my traffic coming into the network over the VPN by viewing logs on the AlanVPN -> port2 policy:

![logging](/1resources/images/awsvpn/logging.PNG)

We can also look at the traffic from the instance to my laptop in the port2->AlanVPN policy after adding the route:

![logging](/1resources/images/awsvpn/loggingout.PNG)

Now if I added a default route to this ENI the EC2 instance would get internet access but have to pass through the firewall, this traffic can be viewed in the port2 -> port1 policy, where I could then use various firewall features to secure the traffic.

~~~
[ec2-user@ip-10-0-100-100 ~]$ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=39 time=11.6 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=39 time=11.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=39 time=11.7 ms
^C
--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2003ms
rtt min/avg/max/mdev = 11.688/11.772/11.850/0.066 ms
[ec2-user@ip-10-0-100-100 ~]$
~~~

Here we see traffic leaving the private subnet and going out to the internet in the port2 -> port1 policy.

![logginginternet](/1resources/images/awsvpn/logginginternet.PNG)

## Wrapping Up

As shown above, setting up a VPN directly with a FortiGate EC2 instance is a simple way of establishing a secure channel between an on prem and a cloud network. This is just one of many ways to accomplish this task, and a skilled cloud architect/network engineer should be able to take all of the parameters and come up with a solution that best meets an organization's network and security needs. In the future I will experiment with other methods of connecting my own "on prem" network with my AWS cloud, and perhaps explore those options in future posts. As usual, if you have any questions, don't hesitate to [__email__](mailto:villarreal.alan.aav@gmail.com) me or leave a comment in the disqus section below!







