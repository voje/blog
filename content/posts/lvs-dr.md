+++
title = "LVS DR"
date = "2021-01-18T23:04:03+01:00"
author = "Kristjan Voje"
authorTwitter = "" #do not include @
cover = ""
tags = ["lvs", "dr", "linux virtual server", "direct return"]
keywords = ["lvs", "dr", "linux virtual server", "direct return"]
description = "Setting up Linux Virtual Server with Direct Return"
showFullContent = false
+++

# LVS DR
Linux Virtual Server Direct Return

We're using a four server scenario: client, load balancer, backend server 1, backend server 2.   

Lost an hour trying to get ansible.openstack to work... not sure why the error feedback is so slow but ended up using bash instead.   
```bash
 2029  for server in $(openstack server list | grep voje-test | awk '{ print $4 }'); do echo $server; openstack server delete $server; done
 2031  openstack server create --image focal-server-cloudimg-2021-01-05.img --flavor m1.tiny --key-name k-cloud-key --network mgmt --network air voje-test-client
 2032  openstack server create --image focal-server-cloudimg-2021-01-05.img --flavor m1.tiny --key-name k-cloud-key --network air voje-test-lb
 2033  openstack server create --image focal-server-cloudimg-2021-01-05.img --flavor m1.tiny --key-name k-cloud-key --network air voje-test-bs-1
 2034  openstack server create --image focal-server-cloudimg-2021-01-05.img --flavor m1.tiny --key-name k-cloud-key --network air voje-test-bs-2
```
Manually add a floating IP to the mgmt interface of `voje-test-client`.   

Subnet: `Air 192.168.18.0/24`

Server list:
```
client
voje-test-client
192.168.18.15
172.29.68.5  # external IP (entrypoint)

LB
voje-test-lb
192.168.18.8

BS1
voje-test-bs-1
192.168.18.13

BS2
voje-test-bs-2
192.168.18.11
```

VMs ready, time to set up some backend webservers.   

bs1
```bash
echo "Hello from backend server 1" > index.html && python3 -m http.server 8080 &
```

bs2
```bash
echo "Hello from backend server 2" > index.html && python3 -m http.server 8080 &
```

client
```bash
ubuntu@voje-test-client:~$ curl voje-test-bs-1:8080
Hello from backend server 1
ubuntu@voje-test-client:~$ curl voje-test-bs-2:8080
Hello from backend server 2
```

Backend servers are (almost) ready.   


## Setting up direct server routing
https://www.server-world.info/en/note?os=Ubuntu_16.04&p=lvs&f=1

### ipvsadm
Install virtual server on the loadbalancer VM.   

lb
```bash
sudo apt-get install ipvsadm
sudo vim /etc/default/ipvsadm
```

### VIP
Set up virtual IP address.   
The guide above is for /etc/network/interfaces, we have netplan.  

Our VIP will be `192.168.18.42`.   

Hmm netplan doesn't define a new interface. Let's see if this causes problems or not.   
```bash
network:
  version: 2
  renderer: networkd
  ethernets:
    enp7s0f0:
      addresses: [aaa.aaa.aaa.aaa/24, bbb.bbb.bbb/24]
      gateway4: aaa.aaa.aaa.1
```

This is my netplan config on loadbalancer:
```
ubuntu@voje-test-lb:~$ cat /etc/netplan/50-cloud-init.yaml
# This file is generated from information provided by the datasource.  Changes
# to it will not persist across an instance reboot.  To disable cloud-init's
# network configuration capabilities, write a file
# /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg with the following:
# network: {config: disabled}
network:
    version: 2
    ethernets:
        ens3:
            addresses: [192.168.18.8/24, 192.168.18.42/24]
            gateway4: 192.168.18.1
        ens7:
            dhcp4: true
```
Make sure to run `netplan try` before anything else so we don't lock ourselves out.   

```bash
ubuntu@voje-test-lb:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:ab:32:34 brd ff:ff:ff:ff:ff:ff
    inet 192.168.18.8/24 brd 192.168.18.255 scope global ens3
       valid_lft forever preferred_lft forever
    inet 192.168.18.42/24 brd 192.168.18.255 scope global secondary ens3
       valid_lft forever preferred_lft forever
    inet6 fe80::f816:3eff:feab:3234/64 scope link
       valid_lft forever preferred_lft forever
3: ens7: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether fa:16:3e:61:e8:a6 brd ff:ff:ff:ff:ff:ff
    inet 192.168.254.5/24 brd 192.168.254.255 scope global dynamic ens7
       valid_lft 86023sec preferred_lft 86023sec
    inet6 fe80::f816:3eff:fe61:e8a6/64 scope link
       valid_lft forever preferred_lft forever
```

I can ping the .42 IP from client right now.   
```bash
ubuntu@voje-test-client:~$ arp
Address                  HWtype  HWaddress           Flags Mask            Iface
voje-test-bs-2           ether   fa:16:3e:0c:1e:a0   C                     ens4
voje-test-lb             ether   fa:16:3e:ab:32:34   C                     ens4
voje-test-bs-1           ether   fa:16:3e:37:85:05   C                     ens4
192.168.18.42            ether   fa:16:3e:ab:32:34   C                     ens4
```

Here's the catch: we need to add this virtual IP to all of the nodes (so LB, BS1 and BS2).   
Only LB should be propagated as the owner of the VIP on the subnet though -- the ARP tables on all of the subnet 
machines should map `.42` --> `MAC(LB)` and NOT to `MAC(SB1,2)`.   

On BS1 and BS2:
```
iptables -t nat -A PREROUTING -d 192.168.18.42 -j REDIRECT 
```
Quickly checked... this should add a rule to the `nat` table which says to redirect all packets destined for `192.168.18.42` back to our own machine.   
This will probably mute the ARP broadcasts for our virtual IP.   
TODO: iptables deep dive


On LB
```
sudo ipvsadm -C  # clear rules
sudo ipvsadm -A -t 192.168.18.42:8080 -s rr  # add virtual server
sudo ipvsadm -a -t 192.168.18.42:8080 -r 192.168.18.13 -g  # add backend server
sudo ipvsadm -a -t 192.168.18.42:8080 -r 192.168.18.11 -g  # add backend server
```

```bash
ubuntu@voje-test-lb:~$ ipvsadm -l
Can't initialize ipvs: Permission denied (you must be root)
Are you sure that IP Virtual Server is built in the kernel or as module?
ubuntu@voje-test-lb:~$ sudo ipvsadm -l
IP Virtual Server version 1.2.1 (size=4096)
Prot LocalAddress:Port Scheduler Flags
  -> RemoteAddress:Port           Forward Weight ActiveConn InActConn
TCP  192.168.18.42:http-alt rr
  -> 192.168.18.11:http-alt       Route   1      0          0
  -> 192.168.18.13:http-alt       Route   1      0          0
ubuntu@voje-test-lb:~$
```

Save the rules
```bash
ubuntu@voje-test-lb:~$ sudo ipvsadm -S | sudo tee -a /etc/ipvsadm.rules

-A -t 192.168.18.42:http-alt -s rr
-a -t 192.168.18.42:http-alt -r 192.168.18.11:http-alt -g -w 1
-a -t 192.168.18.42:http-alt -r 192.168.18.13:http-alt -g -w 1
```

After setting the VIM on BS1, half of the replies go through (obviously).   
I'm assuming ARP tables are going to be problematic now.   

### ARP
Clear ARP cache on a node:
```bash
arp -n
ip -s -s neigh flush all
arp -n
```

```bash
root@voje-test-client:~# arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.18.11            ether   fa:16:3e:0c:1e:a0   C                     ens4        # BS2
192.168.18.8             ether   fa:16:3e:ab:32:34   C                     ens4        # LB
192.168.18.13            ether   fa:16:3e:37:85:05   C                     ens4        # BS1
```

Ok now so BS1 should have `iptables` configured so it doesn't broadcast arp while BS2 hasn't been configured properly.   
```bash
192.168.18.42            ether   fa:16:3e:0c:1e:a0   C                     ens4
```

So we're getting answers directly from BS2 which is not OK.   
```bash
root@voje-test-client:~# curl 192.168.18.42:8080
Hello from backend server 2
```

Adding iptables setting to BS2
```bash
iptables -t nat -A PREROUTING -d 192.168.18.42 -j REDIRECT 
```

Trying the `arptables` method on all backend servers:
```bash
ubuntu@voje-test-bs-2:~/serve$ sudo arptables -A OUTPUT -s 192.168.18.42 -j mangle --mangle-ip-s 192.168.18.8
ubuntu@voje-test-bs-2:~/serve$ sudo arptables -A INPUT -d 192.168.18.42 -j DROP
ubuntu@voje-test-bs-2:~/serve$ sudo arptables -L
Chain INPUT (policy ACCEPT)
-j DROP -d voje-test-bs-2

Chain OUTPUT (policy ACCEPT)
-j mangle -s voje-test-bs-2 --mangle-ip-s 192.168.18.8
```

Loadbalancer seems to be working now.   
```bash
root@voje-test-client:~# curl 192.168.18.42:8080
Hello from backend server 2
root@voje-test-client:~# curl 192.168.18.42:8080
Hello from backend server 1
root@voje-test-client:~# curl 192.168.18.42:8080
Hello from backend server 2
root@voje-test-client:~# curl 192.168.18.42:8080
Hello from backend server 1
root@voje-test-client:~#
```

Let's make sure the return packets aren't routed via LB.   

Running two http requests on client:
```bash
ubuntu@voje-test-client:~$ curl 192.168.18.42:8080
Hello from backend server 1
ubuntu@voje-test-client:~$ curl 192.168.18.42:8080
Hello from backend server 2
```

tshark on client
```bash
ubuntu@voje-test-client:~$ sudo tshark -i ens4
Running as user "root" and group "root". This could be dangerous.
Capturing on 'ens4'
    1 0.000000000 192.168.18.15 → 192.168.18.42 TCP 74 57748 → 8080 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=3012735747 TSecr=0 WS=128
    2 0.003030878 192.168.18.42 → 192.168.18.15 TCP 74 8080 → 57748 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM=1 TSval=2089864125 TSecr=3012735747 WS=128
    3 0.003105138 192.168.18.15 → 192.168.18.42 TCP 66 57748 → 8080 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3012735750 TSecr=2089864125
    4 0.003279033 192.168.18.15 → 192.168.18.42 HTTP 148 GET / HTTP/1.1
    5 0.003997128 192.168.18.42 → 192.168.18.15 TCP 66 8080 → 57748 [ACK] Seq=1 Ack=83 Win=65152 Len=0 TSval=2089864126 TSecr=3012735750
    6 0.004969762 192.168.18.42 → 192.168.18.15 TCP 250 HTTP/1.0 200 OK  [TCP segment of a reassembled PDU]
    7 0.004987076 192.168.18.15 → 192.168.18.42 TCP 66 57748 → 8080 [ACK] Seq=83 Ack=185 Win=64128 Len=0 TSval=3012735752 TSecr=2089864127
    8 0.005108794 192.168.18.42 → 192.168.18.15 HTTP 94 HTTP/1.0 200 OK  (text/html)
    9 0.005142854 192.168.18.13 → 192.168.18.15 SSH 166 Server: Encrypted packet (len=100)
   10 0.005154296 192.168.18.15 → 192.168.18.13 TCP 66 53974 → 22 [ACK] Seq=1 Ack=101 Win=1283 Len=0 TSval=101857741 TSecr=2089864127
   11 0.005243331 192.168.18.13 → 192.168.18.15 SSH 102 Server: Encrypted packet (len=36)
   12 0.005251227 192.168.18.15 → 192.168.18.13 TCP 66 53974 → 22 [ACK] Seq=1 Ack=137 Win=1283 Len=0 TSval=101857741 TSecr=2089864127
   13 0.005969075 192.168.18.15 → 192.168.18.42 TCP 66 57748 → 8080 [FIN, ACK] Seq=83 Ack=214 Win=64128 Len=0 TSval=3012735753 TSecr=2089864127
   14 0.006670183 192.168.18.42 → 192.168.18.15 TCP 66 8080 → 57748 [ACK] Seq=214 Ack=84 Win=65152 Len=0 TSval=2089864129 TSecr=3012735753
   15 1.198771641 192.168.18.15 → 192.168.18.42 TCP 74 57750 → 8080 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=3012736946 TSecr=0 WS=128
   16 1.200521075 192.168.18.42 → 192.168.18.15 TCP 74 8080 → 57750 [SYN, ACK] Seq=0 Ack=1 Win=65160 Len=0 MSS=1460 SACK_PERM=1 TSval=114231913 TSecr=3012736946 WS=128
   17 1.200583756 192.168.18.15 → 192.168.18.42 TCP 66 57750 → 8080 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3012736948 TSecr=114231913
   18 1.200708799 192.168.18.15 → 192.168.18.42 HTTP 148 GET / HTTP/1.1
   19 1.201302648 192.168.18.42 → 192.168.18.15 TCP 66 8080 → 57750 [ACK] Seq=1 Ack=83 Win=65152 Len=0 TSval=114231914 TSecr=3012736948
   20 1.202135411 192.168.18.42 → 192.168.18.15 TCP 250 HTTP/1.0 200 OK  [TCP segment of a reassembled PDU]
   21 1.202156693 192.168.18.15 → 192.168.18.42 TCP 66 57750 → 8080 [ACK] Seq=83 Ack=185 Win=64128 Len=0 TSval=3012736949 TSecr=114231915
   22 1.202224456 192.168.18.42 → 192.168.18.15 HTTP 94 HTTP/1.0 200 OK  (text/html)
   23 1.202408064 192.168.18.11 → 192.168.18.15 SSH 166 Server: Encrypted packet (len=100)
   24 1.202421700 192.168.18.15 → 192.168.18.11 TCP 66 49296 → 22 [ACK] Seq=1 Ack=101 Win=501 Len=0 TSval=1889704740 TSecr=114231916
   25 1.203001046 192.168.18.15 → 192.168.18.42 TCP 66 57750 → 8080 [FIN, ACK] Seq=83 Ack=214 Win=64128 Len=0 TSval=3012736950 TSecr=114231915
   26 1.203488893 192.168.18.42 → 192.168.18.15 TCP 66 8080 → 57750 [ACK] Seq=214 Ack=84 Win=65152 Len=0 TSval=114231917 TSecr=3012736950
   27 2.182815052 192.168.18.8 → 224.0.0.81   IPVS 194
   28 5.172601317 fa:16:3e:f7:bd:15 → fa:16:3e:37:85:05 ARP 42 Who has 192.168.18.13? Tell 192.168.18.15
   29 5.174174625 fa:16:3e:37:85:05 → fa:16:3e:f7:bd:15 ARP 42 192.168.18.13 is at fa:16:3e:37:85:05
   30 6.377310556 fa:16:3e:0c:1e:a0 → fa:16:3e:f7:bd:15 ARP 42 Who has 192.168.18.15? Tell 192.168.18.11
   31 6.377352752 fa:16:3e:f7:bd:15 → fa:16:3e:0c:1e:a0 ARP 42 192.168.18.15 is at fa:16:3e:f7:bd:15
   32 6.452604770 fa:16:3e:f7:bd:15 → fa:16:3e:0c:1e:a0 ARP 42 Who has 192.168.18.11? Tell 192.168.18.15
   33 6.453102294 fa:16:3e:0c:1e:a0 → fa:16:3e:f7:bd:15 ARP 42 192.168.18.11 is at fa:16:3e:0c:1e:a0
   33 packets captured
```

tshark on LB1 (one-directional traffic)
```bash
ubuntu@voje-test-lb:~$ sudo tshark -i ens3
Running as user "root" and group "root". This could be dangerous.
Capturing on 'ens3'
    1 0.000000000 192.168.18.15 → 192.168.18.42 TCP 74 57748 → 8080 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=3012735747 TSecr=0 WS=128
    2 0.000027288 192.168.18.15 → 192.168.18.42 TCP 74 [TCP Out-Of-Order] 57748 → 8080 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=3012735747 TSecr=0 WS=128
    3 0.002421007 192.168.18.15 → 192.168.18.42 TCP 66 57748 → 8080 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3012735750 TSecr=2089864125
    4 0.002443634 192.168.18.15 → 192.168.18.42 TCP 66 [TCP Dup ACK 3#1] 57748 → 8080 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3012735750 TSecr=2089864125
    5 0.002572930 192.168.18.15 → 192.168.18.42 HTTP 148 GET / HTTP/1.1
    6 0.002580217 192.168.18.15 → 192.168.18.42 TCP 148 [TCP Retransmission] 57748 → 8080 [PSH, ACK] Seq=1 Ack=1 Win=64256 Len=82 TSval=3012735750 TSecr=2089864125
    7 0.004282543 192.168.18.15 → 192.168.18.42 TCP 66 57748 → 8080 [ACK] Seq=83 Ack=185 Win=64128 Len=0 TSval=3012735752 TSecr=2089864127
    8 0.004289545 192.168.18.15 → 192.168.18.42 TCP 66 [TCP Dup ACK 7#1] 57748 → 8080 [ACK] Seq=83 Ack=185 Win=64128 Len=0 TSval=3012735752 TSecr=2089864127
    9 0.005268132 192.168.18.15 → 192.168.18.42 TCP 66 57748 → 8080 [FIN, ACK] Seq=83 Ack=214 Win=64128 Len=0 TSval=3012735753 TSecr=2089864127
   10 0.005275033 192.168.18.15 → 192.168.18.42 TCP 66 [TCP Out-Of-Order] 57748 → 8080 [FIN, ACK] Seq=83 Ack=214 Win=64128 Len=0 TSval=3012735753 TSecr=2089864127
   11 1.198180488 192.168.18.15 → 192.168.18.42 TCP 74 57750 → 8080 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=3012736946 TSecr=0 WS=128
   12 1.198220933 192.168.18.15 → 192.168.18.42 TCP 74 [TCP Out-Of-Order] 57750 → 8080 [SYN] Seq=0 Win=64240 Len=0 MSS=1460 SACK_PERM=1 TSval=3012736946 TSecr=0 WS=128
   13 1.199833857 192.168.18.15 → 192.168.18.42 TCP 66 57750 → 8080 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3012736948 TSecr=114231913
   14 1.199845239 192.168.18.15 → 192.168.18.42 TCP 66 [TCP Dup ACK 13#1] 57750 → 8080 [ACK] Seq=1 Ack=1 Win=64256 Len=0 TSval=3012736948 TSecr=114231913
   15 1.199953771 192.168.18.15 → 192.168.18.42 HTTP 148 GET / HTTP/1.1
   16 1.199958876 192.168.18.15 → 192.168.18.42 TCP 148 [TCP Retransmission] 57750 → 8080 [PSH, ACK] Seq=1 Ack=1 Win=64256 Len=82 TSval=3012736948 TSecr=114231913
   17 1.201486687 192.168.18.15 → 192.168.18.42 TCP 66 57750 → 8080 [ACK] Seq=83 Ack=185 Win=64128 Len=0 TSval=3012736949 TSecr=114231915
   18 1.201492492 192.168.18.15 → 192.168.18.42 TCP 66 [TCP Dup ACK 17#1] 57750 → 8080 [ACK] Seq=83 Ack=185 Win=64128 Len=0 TSval=3012736949 TSecr=114231915
   19 1.202248090 192.168.18.15 → 192.168.18.42 TCP 66 57750 → 8080 [FIN, ACK] Seq=83 Ack=214 Win=64128 Len=0 TSval=3012736950 TSecr=114231915
   20 1.202253023 192.168.18.15 → 192.168.18.42 TCP 66 [TCP Out-Of-Order] 57750 → 8080 [FIN, ACK] Seq=83 Ack=214 Win=64128 Len=0 TSval=3012736950 TSecr=114231915
   21 2.180436156 192.168.18.8 → 224.0.0.81   IPVS 194

21 packets capture
```

ARP tables on client look healthy after a few flushes (.42 should route to LB's MAC).   
```bash
ubuntu@voje-test-client:~$ arp -n
Address                  HWtype  HWaddress           Flags Mask            Iface
192.168.18.8             ether   fa:16:3e:ab:32:34   C                     ens4
192.168.18.11            ether   fa:16:3e:0c:1e:a0   C                     ens4
192.168.18.13            ether   fa:16:3e:37:85:05   C                     ens4
192.168.18.42            ether   fa:16:3e:ab:32:34   C                     ens4
```

## TODO
some more testing, install tshark on BS1,2.   
