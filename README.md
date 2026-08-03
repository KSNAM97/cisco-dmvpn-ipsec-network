# DMVPN + IPsec 기반 본사-지사 통합 네트워크 구축

**작성자**: 남기석  
**주제**: OSPF · EIGRP 재분배 / NAT / HSRP 이중화 / DMVPN Phase 2 & Phase 3

---

## 프로젝트 개요

외부망(인터넷)으로부터 안전한 통신 환경을 구성하고,  
본사-지사 간 이중화된 터널과 라우팅을 통해 고가용성 WAN 네트워크를 구현한다.

| 구분 | 내용 |
|---|---|
| 본사(HQ) 내부망 | VLAN 10/20/30/40/99 + HSRP 이중화 + OSPF Area 0 |
| 본사-지사 VPN | DMVPN(mGRE+IPsec) Phase 2 (BR1·BR2) / Phase 3 (BR3·BR4) |
| 라우팅 | HQ: OSPF 1 / 지사: EIGRP 100(AS100), EIGRP 200(AS200) + 재분배 |
| 보안 | IPsec AES-256 / SHA / DH Group 5 |
| NAT | 사설망 NAT 제외 + 인터넷 트래픽 PAT |

---

## 토폴로지

![Topology 1](topology/TOPOLOGY-1.png)

---

## 주소 체계

### WAN (ISP 203.0.113.0/24)

| 구간 | 인터페이스 | IP |
|---|---|---|
| R-ISP ↔ BR1 | Fa5/0 / BR1 Fa0/0 | 203.0.113.2 / .1 |
| R-ISP ↔ Edge1 | Gi0/0 / Edge1 Gi0/0 | 203.0.113.5 / .6 |
| R-ISP ↔ Edge2 | Gi1/0 / Edge2 Gi0/0 | 203.0.113.9 / .10 |
| R-ISP ↔ BR2 | Fa5/1 / BR2 Fa0/0 | 203.0.113.13 / .14 |
| R-ISP ↔ BR3 | Gi2/0 / BR3 Gi0/0 | 203.0.113.18 / .17 |
| R-ISP ↔ BR4 | Gi3/0 / BR4 Gi0/0 | 203.0.113.21 / .22 |

### 본사 VLAN (HQ)

| VLAN | 용도 | 네트워크 | VIP(HSRP) | Core1 | Core2 |
|---|---|---|---|---|---|
| 10 | Data | 10.10.10.0/24 | .1 | .2 (Active) | .3 |
| 20 | Voice | 10.10.20.0/24 | .1 | .2 (Active) | .3 |
| 30 | Server | 10.10.30.0/24 | .1 | .2 | .3 (Active) |
| 40 | Guest | 10.10.40.0/24 | .1 | .2 | .3 (Active) |
| 99 | Mgmt | 10.10.99.0/24 | .1 | .2 (Active) | .3 |

### 지사 LAN

| 지사 | 네트워크 | GW |
|---|---|---|
| BR1 | 172.10.20.0/24 | 172.10.20.1 |
| BR2 | 172.100.30.0/24 | 172.100.30.1 |
| BR3 | 172.10.30.0/24 | 172.10.30.1 |
| BR4 | 172.200.40.0/24 | 172.200.40.1 |

### DMVPN 터널

| 터널 | Phase | HUB IP | SPOKE | AS |
|---|---|---|---|---|
| Tunnel0 | Phase 2 | Edge1=172.16.0.1 / Edge2=172.16.0.2 | BR1=.11 / BR2=.22 | EIGRP 100/200 |
| Tunnel1 | Phase 3 | Edge1=172.16.10.1 / Edge2=172.16.10.2 | BR3=.33 / BR4=.44 | EIGRP 100/200 |

### HQ 백본 링크 (OSPF, 172.168.10.0/24)

| 구간 | Edge1 | Edge2 / Core |
|---|---|---|
| Edge1 ↔ Edge2 | Gi1/0 = .1 | Gi1/0 = .2 |
| Edge1 ↔ Core1 | Gi3/0 = .5 | Core1 Gi0/0 = .6 |
| Edge1 ↔ Core2 | Gi2/0 = .9 | Core2 Gi0/1 = .10 |
| Edge2 ↔ Core1 | Gi2/0 = .13 | Core1 Gi0/1 = .14 |
| Edge2 ↔ Core2 | Gi3/0 = .17 | Core2 Gi0/0 = .18 |

---

## 핵심 설계 포인트

### HSRP 이중화
- Core1 Active: VLAN 10, 20, 99 / Core2 Active: VLAN 30, 40
- priority 110 + preempt + track 10 (decrement 20)
- track 10 = boolean or (Gi0/0, Gi0/1) — 업링크 전체 장애 시 반대 Core로 전환

### DMVPN Phase 2 vs Phase 3
| | Phase 2 (Tunnel0) | Phase 3 (Tunnel1) |
|---|---|---|
| Spoke | BR1, BR2 | BR3, BR4 |
| Spoke-to-Spoke | Hub 경유 | 직접 터널 형성 |
| NHRP redirect (HUB) | 미적용 | 적용 |
| NHRP shortcut (SPOKE) | 미적용 | 적용 |
| 터널 키 / network-id | 100 | 200 |

### 재분배 태그 설계 (루프 방지)
| 방향 | Match Tag | Set Tag |
|---|---|---|
| EIGRP100 → OSPF | — | 100 |
| EIGRP200 → OSPF | — | 200 |
| OSPF → EIGRP100 | deny tag 100, permit | 1 |
| OSPF → EIGRP200 | deny tag 200, permit | 1 |
| EIGRP100 ↔ EIGRP200 | deny 상대 tag | 상대 tag |

---

## 파일 구조

```
.
├── README.md
├── topology/
│   └── TOPOLOGY-1.png
├── configs/
│   ├── HQ-Router/
│   │   ├── HQ-EDGE-1.txt
│   │   └── HQ-EDGE-2.txt
│   ├── HQ-Switch/
│   │   ├── HQ-CORE-1.txt
│   │   ├── HQ-CORE-2.txt
│   │   ├── HQ-ACCESS-1.txt
│   │   ├── HQ-ACCESS-2.txt
│   │   ├── HQ-ACCESS-3.txt
│   │   └── HQ-ACCESS-4.txt
│   ├── Branch/
│   │   ├── BRANCH1.txt
│   │   ├── BRANCH2.txt
│   │   ├── BRANCH3.txt
│   │   └── BRANCH4.txt
│   └── ISP-VPC/
│       ├── R-ISP.txt
│       ├── VPC-기본컨피그.txt
│       └── 웹서버-기본컨피그.txt
└── verification/
    ├── verification-guide.md   ← 단계별 검증 명령어
    └── requirements.md         ← 전체 구성 요구사항
```
