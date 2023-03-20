# Lab 1:

![image](https://user-images.githubusercontent.com/54473576/225796696-78ef217c-4c50-4ec4-a1f6-66d4b96a97cf.png)


Bài lab gồm:
  - 1 ubuntu server đóng vai trò là gateway. Ubuntu gồm 2 NIC, 1 NIC được kết nối với internet và 1 NIC được kết nối với 1 LAN
  - Các máy vm1, vm2 đóng vai trò làm client muốn truy cập internet
  
## Các bước tiến hành

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
       valid_lft 1238sec preferred_lft 1238sec
    inet6 fe80::20c:29ff:fe28:1317/64 scope link 
       valid_lft forever preferred_lft forever
3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:21 brd ff:ff:ff:ff:ff:ff
    altname enp2s5
    inet 10.0.0.1/24 brd 10.0.0.255 scope global ens37
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe28:1321/64 scope link 
       valid_lft forever preferred_lft forever
```

### Cài đặt isc-dhcp-server

Cài đặt isc-dhcp-server

```
sudo apt install isc-dhcp-server
```
Dùng `sudo vim /etc/dhcp/dhcpd.conf` để sửa file config

```
# To specify the default and maximum lease time 
default-lease-time 600;

max-lease-time 7200;

ddns-update-style none;

authoritative;

subnet 10.0.0.0 netmask 255.255.255.0
{
        range 10.0.0.100 10.0.0.200;
        option routers 10.0.0.1;
        option subnet-mask 255.255.255.0;
}
```

Dùng `sudo systemctl restart isc-dhcp-server` để khởi động dịch vụ `isc-dhcp-server`

Dùng `sudo systemctl enable isc-dhcp-server` để bật dịch vụ `isc-dhcp-server`


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
Dùng `sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens33 --jump MASQUERADE` để thêm rule NAT

Lưu cài đặt đã thay đổi với iptables: `sudo iptables-save > /etc/iptables/rules.v4`

### Đối với máy client:

![image](https://user-images.githubusercontent.com/54473576/225839656-011d4d21-67d6-4eb0-ba37-c60a9086e1ea.png)

### Kiểm tra trên máy client
![Screenshot from 2023-03-20 13-44-04](https://user-images.githubusercontent.com/54473576/226266198-252cf5b2-4c9a-41c6-b204-23ac13081903.png)

![Screenshot from 2023-03-20 13-46-32](https://user-images.githubusercontent.com/54473576/226266564-c93c0152-8603-46db-a239-05d773e5f468.png)
