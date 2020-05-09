---
title: "AWS/On-Prem Fortigate Site to Site VPN"
date: 2020-05-09T10:39:51-07:00
draft: false
urltitle: "awsvpn"
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

HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO HELLO  <!--more--> 

### Local Fortigate VPN config
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

### AWS Fortigate VPN Config

~~~
config vpn ipsec phase1-interface
    edit "AlanVPN"
        set interface "port1"
        set peertype any
        set net-device disable
        set proposal aes256-sha256
        set dhgrp 5
        set remote-gw 98.111.206.31
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