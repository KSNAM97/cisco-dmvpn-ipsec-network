# SW-Access3

> HQ 액세스 스위치 / VLAN30(Server) · VTP Client

---

## Configuration

```
en
conf t
!
hostname SW-Access3
!
no ip domain-lookup
!
line console 0
 logging synchronous
 exec-timeout 0 0
 length 0
 exit
!
line vty 0 4
 logging synchronous
 exec-timeout 0 0
 length 0
 exit
!
end
!
wr
!
conf t
!
vtp password Cisco123
vtp domain HQ
vtp version 3
vtp mode client
!
int range gi0/0 - 1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,40,99
 no sh
!
enable secret cisco
!
line vty 0 15
  password cisco
  login
!
int vlan 99
 ip add 10.10.99.33 255.255.255.0
 no sh
!
 ip default-gateway 10.10.99.1
!
spanning-tree mode rapid-pvst

!
int gi0/0
 spanning-tree vlan 30 cost 100         ! VLAN30은 gi0/0(Core1) 고비용
int gi0/1
 spanning-tree vlan 30 cost 10           ! gi0/1(Core2=vlan30 active/루트) 우선 ← 방향 반대!
!
int range gi0/2-3
 switchport mode access
 switchport access vlan 30
 spanning-tree bpduguard enable
 spanning-tree portfast
 no sh
!
```
