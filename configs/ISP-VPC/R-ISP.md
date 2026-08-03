# R-ISP

> ISP 라우터 / WAN 링크 허브

---

## Configuration

```
en
conf t
!
hostname R-ISP
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
 int Fa5/0
  ip add 203.0.113.2 255.255.255.252    ! Branch1(203.0.113.1)과 /30 페어
  no sh
 int Gi0/0
  ip add 203.0.113.5 255.255.255.252    ! HQ-Edge1(203.0.113.6)과 페어
  no sh
 int Gi1/0
  ip add 203.0.113.9 255.255.255.252     ! HQ-Edge2(203.0.113.10)과 페어
  no sh
 int Fa5/1
  ip add 203.0.113.13 255.255.255.252   ! Branch2(203.0.113.14)와 페어
  no sh
 int Gi2/0
  ip add 203.0.113.18 255.255.255.252    ! Branch3(203.0.113.17)과 페어
  no sh
 int Gi3/0
  ip add 203.0.113.21 255.255.255.252    ! Branch4(203.0.113.22)와 페어
  no sh
 int gi4/0
 ip add 203.0.113.25 255.255.255.252          ! WEB-SERVER(203.0.113.26)와 페어
  no sh
!
 ip route 8.8.8.8 255.255.255.255 203.0.113.26   ! 인터넷 목적지(8.8.8.8) → WEB-SERVER로 정적 경로
!
```
