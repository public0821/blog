

dev@debian:~$ sudo ip addr add 192.168.3.11/24 dev tun1
dev@debian:~$ sudo ip link set tun1 up

dev@debian:~$ ping -c 4 192.168.3.12
PING 192.168.3.12 (192.168.3.12) 56(84) bytes of data.

--- 192.168.3.12 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3023ms

dev@debian:~$ sudo ./tun
Open tun/tap device: tun1 for reading...
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device
Read 84 bytes from tun/tap device


# tcpdump -i tun1
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on tun1, link-type RAW (Raw IP), capture size 262144 bytes
19:57:13.473101 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 1, length 64
19:57:14.480362 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 2, length 64
19:57:15.488246 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 3, length 64
19:57:16.496241 IP 192.168.3.11 > 192.168.3.12: ICMP echo request, id 24028, seq 4, length 64


[Linux Switching – Interconnecting Namespaces](http://www.opencloudblog.com/?p=66)

[Universal TUN/TAP device driver](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)

[Virtual networking in Linux](https://www.ibm.com/developerworks/library/l-virtual-networking/)

[Tun/Tap interface tutorial](http://backreference.org/2010/03/26/tuntap-interface-tutorial/)