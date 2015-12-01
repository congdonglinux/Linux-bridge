#Linux bridge
Linux bridge là 1 công nghệ tạo ra các switch ảo, nhằm tạo ra 1 thiết bị làm việc ở lớp 2, kết nối các interface vật lý trong hệ thống để tạo thành 1 miền layer2 duy nhất. 

Song song cùng công nghệ openvswitch, đây là 2 công nghệ chiếm chủ yếu thị phần hiện nay. Linux bridge có khả năng tương thích tốt với các công nghệ về netfiltering điển hình như ebtables.

Linux bridge được tích hợp sẵn trong nhân Linux, tuy nhiên để sử dụng các tính năng VLAN hay bonding, ta cần nạp các module này vào nhân trước khi sử dụng (sẽ nói ở phần sau)

###Downloading

`#apt-get install bridge-utils`

Khi cài đặt gói này thành công, ta có thể sử dụng công cụ brctl để tạo và sửa các switch ảo:

      #brctl
        #command
        addbr           <bridge>                add bridge
        delbr           <bridge>                delete bridge
        addif           <bridge> <device>       add interface to bridge
        delif           <bridge> <device>       delete interface from bridge
        setageing       <bridge> <time>         set ageing time
        setbridgeprio   <bridge> <prio>         set bridge priority
        setfd           <bridge> <time>         set bridge forward delay
        sethello        <bridge> <time>         set hello time
        setmaxage       <bridge> <time>         set max message age
        setpathcost     <bridge> <port> <cost>  set path cost
        setportprio     <bridge> <port> <prio>  set port priority
        show                                    show a list of bridges
        showmacs        <bridge>                show a list of mac addrs
        showstp         <bridge>                show bridge stp info
        stp             <bridge> <state>        turn stp on/off
		
 
###Tạo switch ảo và gắn các card mạng:

    #brctl addbr {bridgename}
    #brctl delbr {bridgename}
    #brctl addif {bridgename} {device}
    #brctl delif {bridgename} {device}
 
 Show cấu hình :

    #brctl show
    #brctl showmacs bridgename
 
###Spanning Tree Protocol:

    #brctl stp bridgename on/off
    #brctl showstp bridgename
 
 Các cấu hình này sẽ mất sau khi reboot hoặc reset lại mạng. Để sử dụng lâu dài cấu hình, cần khai báo trong `/etc/network/interfaces`

    auto bridgename
    iface bridgename inet static
	    address 192.168.0.2
	    netmask 255.255.255.0
	    gateway 192.168.0.1
	    bridge_ports eth0 eth1
	    bridge_maxwait 5
	    bridge_fd 1
	    bridge_stp on

--------------------------------------------------

##VLAN

Để sử dụng tính năng VLAN, ta cần cài đặt gói **vlan** để sử dụng ở mức người dùng:

`#apt-get install vlan`

Nạp module vlan vào trong nhân:

    #modprobe 8021q
    #echo "8021q" >> /etc/modules

Sau khi nạp thành công module vlan, thư mục cấu hình vlan sẽ xuất hiện ở `/proc/net/vlan/config `

####Tạo các vlan 

`#vconfig add eth1 10`  --->tạo ra VLAN có ID 10 và gán vào card mạng eth1

`#vconfig add eth2 20`  --->tạo ra VLAN có ID 20 và gán vào card mạng eth2

ta sẽ thấy kết quả như sau:

    # Added VLAN with VID == 10 to IF -:eth1:-
    # Added VLAN with VID == 20 to IF -:eth2:-

Xóa VLan:

`#vconfig rem eth1.10`  --->remove VLAN 10 trên interface eth1
`#vconfig rem eth2.20`  --->remove VLAN 20 trên interface eth2

Gắn các vlan interface vào bridge ảo:

    #brctl addbr bridgename
    #brctl addif bridgename eth1.10
    #brctl addif bridgename eth2.20

Để sử dụng lâu dài cấu hình này, cần khai báo trong `/etc/network/interfaces`

    auto eth1.10
    iface eth1.10 inet static
        address 10.0.0.1
        netmask 255.255.255.0
        vlan-raw-device eth1
	
    auto eth2.20
    iface eth2.20 inet static
        address 10.0.0.2
        netmask 255.255.255.0
        vlan-raw-device eth2
	
    auto bridgename
    iface bridgename inet static
	    bridge_ports eth1.10 eth2.20
	    bridge_maxwait 5
	    bridge_fd 1
	    bridge_stp on	
	
-------------------------------------------

##Bonding

Để sử dụng tính năng bonding, cần cài đặt gói **ifenslave** và nạp module bonding vào nhân:

Cài đặt gói ifenslave:

`#apt-get install ifenslave`

Nạp module vào nhân:

    #modprobe bonding
    #echo bonding >> etc/modules

Sau khi nạp thành công module vào nhân, thư mục cấu hình bonding sẽ xuất hiện ở `/proc/net/bonding`

####Các tùy chọn bonding
1. bond-mode: Linux bridge hỗ trợ các mode bonding sau:

   - mode 1: actice-backup
   - mode 2: balance-xor
   - mode 3: broadcast
   - mode 4: 802.3ad - Là chuẩn cho việc triển khai LACP
   - mode 5: balance-tlb
   - mode 6: balance-alb

2. bond-slave: danh sách các interface được bond
3. bond-miimon: thời gian kiểm tra downlink
4. bond-use-carrier: cách xác định trạng thái đường link
5. bond-xmit-hash-policy: thuật toán xác định link sẽ dùng khi truyền thông
6. bond-min-links: số đương link tối thiểu cần trong trạng thái active

Sau khi bond, các inteface slave sẽ có chung MAC của bond interface. Thông thường, interface nào được add vào bond đầu tiên sẽ có MAC được dùng làm MAC cho bond interface.

#### Tạo bond interface:

    #ifenslave bond0 eth1
    #ifenslave bond0 eth2
 --> Tạo ra bond interface có tên bond0 và chứa 2 interface eth1 và eth2

Gán bond interface vào bridge:

`#brctl addif bridgename bond0`


Để lưu lại cấu hình này, cần khai báo trong file `/etc/network/interfaces`

    auto eth0
    iface eth0 inet manual
    bond-master bond0

    auto eth1
    iface eth1 inet manual
    bond-master bond0

    auto bond0
    iface bond0 inet manual
      bond-slaves none
      bond-mode 802.3ad
      bond-miimon 100
      bond-lacp-rate 1

    auto bridgename
    iface bridgename inet static
      bridge_ports bond0
      address 172.22.0.6
      netmask 255.255.240.0
      gateway 172.22.0.1
      bridge_fd 0
      bridge_hello 2
      bridge_maxage 12
      bridge_stp off 
  
  
####Kiểm tra lại cấu hình:

  `#sudo brctl show`

    bridge name     bridge id               STP enabled     interfaces
    bridgename       8000.44383900129b       yes             bond0
                                                         
                                                        
