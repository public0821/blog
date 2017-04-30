sudo ip link add veth0 type veth peer name veth1
sudo ip addr add 192.168.3.11/24 dev veth0
sudo ip link set veth0 up

dev@debian:~$ ping -c 4 192.168.3.1
PING 192.168.3.1 (192.168.3.1) 56(84) bytes of data.
From 192.168.3.11 icmp_seq=1 Destination Host Unreachable
From 192.168.3.11 icmp_seq=2 Destination Host Unreachable
From 192.168.3.11 icmp_seq=3 Destination Host Unreachable
From 192.168.3.11 icmp_seq=4 Destination Host Unreachable

--- 192.168.3.1 ping statistics ---
4 packets transmitted, 0 received, +4 errors, 100% packet loss, time 3015ms
pipe 3

dev@debian:~$ sudo tcpdump -i veth0
[sudo] password for dev:
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes


sudo ip link set veth1 up

dev@debian:~$ sudo tcpdump -n -i veth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:20:18.285230 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:19.282018 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:20.282038 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:21.300320 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:22.298783 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:23.298923 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28

dev@debian:~$ sudo tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
20:20:48.570459 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:49.570012 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:50.570023 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:51.570023 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:52.569988 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28
20:20:53.570833 ARP, Request who-has 192.168.3.1 tell 192.168.3.11, length 28

sudo ip addr add 192.168.3.1/24 dev veth1

dev@debian:~$ ping -c 4 192.168.3.1 -I veth0
PING 192.168.3.1 (192.168.3.1) from 192.168.3.11 veth0: 56(84) bytes of data.
64 bytes from 192.168.3.1: icmp_seq=1 ttl=64 time=0.032 ms
64 bytes from 192.168.3.1: icmp_seq=2 ttl=64 time=0.048 ms
64 bytes from 192.168.3.1: icmp_seq=3 ttl=64 time=0.055 ms
64 bytes from 192.168.3.1: icmp_seq=4 ttl=64 time=0.050 ms

--- 192.168.3.1 ping statistics ---
4 packets transmitted, 4 received, 0% packet loss, time 3002ms
rtt min/avg/max/mdev = 0.032/0.046/0.055/0.009 ms


dev@debian:~$ sudo tcpdump -n -i veth0
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth0, link-type EN10MB (Ethernet), capture size 262144 bytes
20:23:43.113062 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24169, seq 1, length 64
20:23:44.112078 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24169, seq 2, length 64
20:23:45.111091 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24169, seq 3, length 64
20:23:46.110082 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24169, seq 4, length 64


dev@debian:~$ sudo tcpdump -n -i veth1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on veth1, link-type EN10MB (Ethernet), capture size 262144 bytes
20:24:12.221372 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24174, seq 1, length 64
20:24:13.222089 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24174, seq 2, length 64
20:24:14.224836 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24174, seq 3, length 64
20:24:15.223826 IP 192.168.3.11 > 192.168.3.1: ICMP echo request, id 24174, seq 4, length 64


##为什么没有回复的包，也没有看到arp的请求包？

dev@debian:~$ sudo tcpdump -n -i lo
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on lo, link-type EN10MB (Ethernet), capture size 262144 bytes
20:25:49.590273 IP 192.168.3.1 > 192.168.3.11: ICMP echo reply, id 24177, seq 1, length 64
20:25:50.590018 IP 192.168.3.1 > 192.168.3.11: ICMP echo reply, id 24177, seq 2, length 64
20:25:51.590027 IP 192.168.3.1 > 192.168.3.11: ICMP echo reply, id 24177, seq 3, length 64
20:25:52.590030 IP 192.168.3.1 > 192.168.3.11: ICMP echo reply, id 24177, seq 4, length 64

[Linux Switching – Interconnecting Namespaces](http://www.opencloudblog.com/?p=66)

[Virtual networking in Linux](https://www.ibm.com/developerworks/library/l-virtual-networking/)