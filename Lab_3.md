# Lab 3:
<img width="738" alt="image" src="https://user-images.githubusercontent.com/54473576/225810956-0764dd06-c5d1-40a8-a252-af80cb512678.png">

Mục đích: 
```
2 con gateway đều làm nhiệm vụ nat ra ngoài.
Mỗi gateway đều có 2 đường để NAT ra ngoài cho trường hợp 1 trong 2 đường đây có vấn đề

Cài đặt keepalive cho trường hợp gateway chính bị down thì có con gateway khác thay thế làm nhiệm bụ NAT
```

Bài lab gồm:
  - 1 ubuntu server đóng vai trò là gateway. Ubuntu gồm 3 NIC, 2 NIC được kết nối với internet và 1 NIC được kết nối với 1 LAN
  - Các máy vm1, vm2 đóng vai trò làm client muốn truy cập internet
  
## Các bước tiến hành

### Cài NAT trên từng gateway

<details>

<summary>Cài đặt NAT trên gateway 1</summary>

#### Thay đổi IPv4 của interface ens37 

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
    inet 172.16.217.133/24 metric 100 brd 172.16.217.255 scope global dynamic ens33
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
    inet 172.16.217.139/24 metric 100 brd 172.16.217.255 scope global dynamic ens38
       valid_lft 1719sec preferred_lft 1719sec
    inet6 fe80::20c:29ff:fe28:132b/64 scope link 
       valid_lft forever preferred_lft forever
```

#### Enable IPv4_forward:

Để nhận các gói tin từ NIC này sang NIC khác của máy

Uncomment dòng `net.ipv4.ip_forward=1` trong file /etc/sysctl.conf.

Mở file /etc/sysctl.conf dùng `sudo vim /etc/sysctl.conf`. Nhấn `i` để vào chế độ insert và sửa. Sửa xong thì nhấn phím `Esc` và gõ `:wq`

Áp dụng thay đổi tức thì mà không cần reboot dùng: `sudo sysctl -p /etc/sysctl.conf`

#### Cài đặt NAT:

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
Dùng các lệnh dưới để thêm rule NAT
```
sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens33 --jump MASQUERADE

sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens38 --jump MASQUERADE

``` 

Dùng `sudo iptables -t nat -L -v` để xem lại các cài đặt

```
gw1@gateway:~$ sudo iptables --table nat -L -v
Chain PREROUTING (policy ACCEPT 25 packets, 3119 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 6 packets, 1119 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 34 packets, 2818 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 34 packets, 2818 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  any    ens33   10.0.0.0/24          anywhere            
    0     0 MASQUERADE  all  --  any    ens38   10.0.0.0/24          anywhere            
gw1@gateway:~$ 

```
Lưu cài đặt đã thay đổi với iptables: `sudo iptables-save > /etc/iptables/rules.v4`
</details>
  
---------
  
<details>
<summary>Cài đặt NAT trên gateway 2</summary>

#### Thay đổi IPv4 của interface ens37 

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

#### Enable IPv4_forward:

Để nhận các gói tin từ NIC này sang NIC khác của máy

Uncomment dòng `net.ipv4.ip_forward=1` trong file /etc/sysctl.conf.

Mở file /etc/sysctl.conf dùng `sudo vim /etc/sysctl.conf`. Nhấn `i` để vào chế độ insert và sửa. Sửa xong thì nhấn phím `Esc` và gõ `:wq`

Áp dụng thay đổi tức thì mà không cần reboot dùng: `sudo sysctl -p /etc/sysctl.conf`

#### Cài đặt NAT:

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
Dùng các lệnh dưới để để thêm rule NAT
```
sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens33 --jump MASQUERADE

sudo iptables --table nat --append POSTROUTING -s 10.0.0.0/24 --out-interface ens38 --jump MASQUERADE
``` 


Dùng `sudo iptables -t nat -L -v` để xem lại các cài đặt

```
gw1@gateway:~$ sudo iptables --table nat -L -v
Chain PREROUTING (policy ACCEPT 25 packets, 3119 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain INPUT (policy ACCEPT 6 packets, 1119 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain OUTPUT (policy ACCEPT 34 packets, 2818 bytes)
 pkts bytes target     prot opt in     out     source               destination         

Chain POSTROUTING (policy ACCEPT 34 packets, 2818 bytes)
 pkts bytes target     prot opt in     out     source               destination         
    0     0 MASQUERADE  all  --  any    ens33   10.0.0.0/24          anywhere            
    0     0 MASQUERADE  all  --  any    ens38   10.0.0.0/24          anywhere            
gw1@gateway:~$ 

```

Lưu cài đặt đã thay đổi với iptables: `sudo iptables-save > /etc/iptables/rules.v4`

</details>

 ### Cài đặt và cấu hình keepalived trên từng server
 
  #### **Cài đặt keepalived trên cả 2 gateway**
   
  ```
    sudo apt-get install linux-headers-$(uname -r)

    sudo apt-get install keepalived
  ```
  
  #### tạo hoặc chỉnh sửa file config của keepalive ta dùng:
   ```
   sudo vim /etc/keepalived/keepalived.conf
   ```
  
 <details>
   <summary>File cấu hình keepalive của gateway 1</summary>

   ```
   global_defs {
        smtp_server localhost
        smtp_connect_timeout 30
   }

   vrrp_instance VI_1 {
        state MASTER
        interface ens37
        virtual_router_id 101
        priority 101
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass 1111
        }
        virtual_ipaddress {
                10.0.0.4
        }
   }
   ```
   
</details>
  
----------------------
  
<details>
<summary>File cấu hình của gateway 2</summary>
  
```
  bal_defs {
        smtp_server localhost
        smtp_connect_timeout 30
  }

  vrrp_instance VI_1 {
        state BACKUP
        interface ens37
        virtual_router_id 101
        priority 100
        advert_int 1
        authentication {
                auth_type PASS
                auth_pass 1111
        }
        virtual_ipaddress {
                10.0.0.4
        }
  }
```
</details>
  
  Giá trị ưu (prioty) tiên cao hơn thì đó là master server. Không quan trọng sử dụng trạng thái nào
  
  `virtual_router_id` phải giống nhau ở cả 2 server
  
  Mặc định `vrrp_instance` hỗ trợ tối đa 20 `virtual_ipaddress`
  
  #### Khởi động lại KeepAlived trên cả 2 server
  
  ```
  sudo service keepalived start
  ```
  
  Kiểm tra trên máy có prioty cao hơn. Trên interface được cấu hình có thêm virtual_ipaddress đã cấu hình  trước đó
  
  ```
  3: ens37: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    link/ether 00:0c:29:28:13:21 brd ff:ff:ff:ff:ff:ff
    altname enp2s5
    inet 10.0.0.1/24 brd 10.0.0.255 scope global ens37
       valid_lft forever preferred_lft forever
    inet 10.0.0.4/32 scope global ens37
       valid_lft forever preferred_lft forever
    inet6 fe80::20c:29ff:fe28:1321/64 scope link 
       valid_lft forever preferred_lft forever
  ```

### Cài đặt isc-dhcp-server trên cả 2 server

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
        #range 10.0.0.50 10.0.0.70; với gw2
        option routers 10.0.0.4;
        option subnet-mask 255.255.255.0;
}
```

Dùng `sudo systemctl restart isc-dhcp-server` để khởi động dịch vụ `isc-dhcp-server`

Dùng `sudo systemctl enable isc-dhcp-server` để bật dịch vụ `isc-dhcp-server`

  
### Đối với máy client:

![image](https://user-images.githubusercontent.com/54473576/225839656-011d4d21-67d6-4eb0-ba37-c60a9086e1ea.png)

### Kiểm tra failover

#### Khi cả 2 gw cùng hoạt động

![image](https://user-images.githubusercontent.com/54473576/226300480-4dc572f3-32b7-4156-9848-c38696a70af0.png)

Địa chỉ IP được nhận từ gateway1 có dải từ 10.0.0.100 đến 10.0.0.200

#### Khi gateway1 bị down
Máy mới kết nối sẽ nhận được địa chỉ từ gateway 2 có dải từ 10.0.0.50 đến 10.0.0.70 

![image](https://user-images.githubusercontent.com/54473576/226301467-1a328ef2-a6df-4517-931c-64556b2cc587.png)

  
  

