# Lab 2:

Mục đích: 1 server làm nhiệm vụ NAT, trong trường hợp 1 đường dây mạng bị down thì có đường dây Internet dự bị khác để thay 

<img width="574" alt="image" src="https://user-images.githubusercontent.com/54473576/225797096-82864ba9-e576-4705-961b-00fc34c08ceb.png">


Bài lab gồm:
  - 1 ubuntu server đóng vai trò là gateway. Ubuntu gồm 3 NIC, 2 NIC được kết nối với internet và 1 NIC được kết nối với 1 LAN
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

### Cài đặt isc-dhcp-server

Cài đặt isc-dhcp-server

```
sudo apt install isc-dhcp-server
```
Dùng `sudo /etc/dhcp/dhcp.conf` để sửa file config

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
Dùng 
```
sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens33 --jump MASQUERADE

sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens38 --jump MASQUERADE

``` 
để thêm rule NAT

Lưu cài đặt đã thay đổi với iptables: `sudo iptables-save > /etc/iptables/rules.v4`

#### ip route của gateway

```
gw1@gateway:~$ ip route
default dev ens33 scope link 
default via 172.16.217.2 dev ens33 proto dhcp src 172.16.217.130 metric 100 
default via 172.16.217.2 dev ens38 proto dhcp src 172.16.217.136 metric 100 
10.0.0.0/24 dev ens37 proto kernel scope link src 10.0.0.1 
172.16.217.0/24 dev ens33 proto kernel scope link src 172.16.217.130 metric 100 
172.16.217.0/24 dev ens38 proto kernel scope link src 172.16.217.136 metric 100 
172.16.217.2 dev ens33 proto dhcp scope link src 172.16.217.130 metric 100 
172.16.217.2 dev ens38 proto dhcp scope link src 172.16.217.136 metric 100
```

### Đối với máy client:

![image](https://user-images.githubusercontent.com/54473576/225839656-011d4d21-67d6-4eb0-ba37-c60a9086e1ea.png)

![image](https://user-images.githubusercontent.com/54473576/226277064-577eeda9-a41b-40e0-b519-8ec9ab07742e.png)

#### Khi cả 2 interface được kết nối
![image](https://user-images.githubusercontent.com/54473576/226276776-f383c984-813c-406b-ac00-4b272d33b5b2.png)

####Khi 1 trong 2 interface được kết nối

![Screenshot from 2023-03-20 15-16-14](https://user-images.githubusercontent.com/54473576/226283453-9326eed2-cc71-4ef6-8c2c-1e70f6030b0c.png)

![image](https://user-images.githubusercontent.com/54473576/226287062-57643416-54b7-468d-83d9-3e05b7bf339b.png)

#### Khi interface ens33 bị ngắt kết nối:

![image](https://user-images.githubusercontent.com/54473576/226288067-e7a6da40-318a-44e7-8304-a2783fec742e.png)

Khi cả 2 interface đều không được kết nối

![Screenshot from 2023-03-20 15-20-50](https://user-images.githubusercontent.com/54473576/226284055-c1206592-df5a-4fc5-99e6-66f402a00c7a.png)

![Screenshot from 2023-03-20 15-20-55](https://user-images.githubusercontent.com/54473576/226284076-70b0b79f-2b41-4b41-bd8a-da39e4ac72d0.png)
