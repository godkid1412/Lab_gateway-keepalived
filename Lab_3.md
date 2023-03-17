# Lab 3:

Mục đích của


Bài lab gồm:
  - 1 ubuntu server đóng vai trò là gateway. Ubuntu gồm 3 NIC, 2 NIC được kết nối với internet và 1 NIC được kết nối với 1 LAN
  - Các máy vm1, vm2 đóng vai trò làm client muốn truy cập internet
  
## Các bước tiến hành

### Cài NAT trên từng gateway

<details>

<summary>Cài đặt NAT trên gateway 1</summary>

### Thay đổi IPv4 của interface ens37 

Dùng: `sudo vim /etc/netplan/00-installer-config.yaml`.

```
network:
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      addresses:
      - 10.0.0.1/24
    ens38:
      dhcp4: true
  version: 2
```
Dùng `sudo netplan apply` để lưu lại đã thay đổi

Dùng `ip a` để xem cài đặt đã thay đổi
```
gw1@gateway:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:17 brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 172.16.217.130/24 metric 100 brd 172.16.217.255 scope global dynamic ens33
       valid_lft 1719sec preferred_lft 1719sec
    inet6 fe80::20c:29ff:fe28:1317/64 scope link 
       valid_lft forever preferred_lft forever
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:21 brd ff:ff:ff:ff:ff:ff
    altname enp2s5
    inet 10.0.0.1/24 brd 10.0.0.255 scope global ens37
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe28:1321/64 scope link 
       valid_lft forever preferred_lft forever
4: ens38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:2b brd ff:ff:ff:ff:ff:ff
    altname enp2s6
    inet 172.16.217.136/24 metric 100 brd 172.16.217.255 scope global dynamic ens38
       valid_lft 1719sec preferred_lft 1719sec
    inet6 fe80::20c:29ff:fe28:132b/64 scope link 
       valid_lft forever preferred_lft forever
```

### Enable IPv4_forward:

Để nhận các gói tin từ NIC này sang NIC khác của máy

Uncomment dòng `net.ipv4.ip_forward=1` trong file /etc/sysctl.conf.

Mở file /etc/sysctl.conf dùng `sudo vim /etc/sysctl.conf`. Nhấn `i` để vào chế độ insert và sửa. Sửa xong thì nhấn phím `Esc` và gõ `:wq`

Áp dụng thay đổi tức thì mà không cần reboot dùng: `sudo sysctl -p /etc/sysctl.conf`

### Cài đặt NAT:

Cài đặt iptables-persistent: dùng `sudo apt-get install iptables-persistent`

Dùng `sudo iptables --table nat -L -v` để xem các chain và rule trong bảng `nat`

```
gw1@gateway:~$ sudo iptables --table nat -L -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```
Dùng 
```
sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens33 --jump MASQUERADE

sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens38 --jump MASQUERADE

``` 
để thêm rule NAT

Lưu cài đặt đã thay đổi với iptables: `sudo iptables-save > /etc/iptables/rules.v4`

</details>
 fvcf
<summary>Cài đặt NAT trên gateway 2</summary>

### Thay đổi IPv4 của interface ens37 

Dùng: `sudo vim /etc/netplan/00-installer-config.yaml`.

```
network:
  ethernets:
    ens33:
      dhcp4: true
    ens37:
      addresses:
      - 10.0.0.2/24
    ens38:
      dhcp4: true
  version: 2
```
Dùng `sudo netplan apply` để lưu lại đã thay đổi

Dùng `ip a` để xem cài đặt đã thay đổi
```
gw2@gateway:~$ ip a
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens33: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:36:ca:7e brd ff:ff:ff:ff:ff:ff
    altname enp2s1
    inet 172.16.217.137/24 metric 100 brd 172.16.217.255 scope global dynamic ens33
       valid_lft 1548sec preferred_lft 1548sec
    inet6 fe80::20c:29ff:fe36:ca7e/64 scope link 
       valid_lft forever preferred_lft forever
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:36:ca:88 brd ff:ff:ff:ff:ff:ff
    altname enp2s5
    inet 10.0.0.2/24 brd 10.0.0.255 scope global ens37
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe36:ca88/64 scope link 
       valid_lft forever preferred_lft forever
4: ens38: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:36:ca:92 brd ff:ff:ff:ff:ff:ff
    altname enp2s6
    inet 172.16.217.138/24 metric 100 brd 172.16.217.255 scope global dynamic ens38
       valid_lft 1550sec preferred_lft 1550sec
    inet6 fe80::20c:29ff:fe36:ca92/64 scope link 
       valid_lft forever preferred_lft forever
```

### Enable IPv4_forward:

Để nhận các gói tin từ NIC này sang NIC khác của máy

Uncomment dòng `net.ipv4.ip_forward=1` trong file /etc/sysctl.conf.

Mở file /etc/sysctl.conf dùng `sudo vim /etc/sysctl.conf`. Nhấn `i` để vào chế độ insert và sửa. Sửa xong thì nhấn phím `Esc` và gõ `:wq`

Áp dụng thay đổi tức thì mà không cần reboot dùng: `sudo sysctl -p /etc/sysctl.conf`

### Cài đặt NAT:

Cài đặt iptables-persistent: dùng `sudo apt-get install iptables-persistent`

Dùng `sudo iptables --table nat -L -v` để xem các chain và rule trong bảng `nat`

```
gw2@gateway:~$ sudo iptables --table nat -L -v
Chain PREROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 0 packets, 0 bytes)
 pkts bytes target     prot opt in     out     source               destination
```
Dùng 
```
sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens33 --jump MASQUERADE

sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens38 --jump MASQUERADE

``` 
để thêm rule NAT

Lưu cài đặt đã thay đổi với iptables: `sudo iptables-save > /etc/iptables/rules.v4`

</details>