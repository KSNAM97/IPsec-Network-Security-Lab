# 06. 실습: IPsec VPN + NAT (사설 IP 네트워크)

> 본사(GIT-A)와 지사(GIT-B)의 사설망을 공중망(OSPF)으로 연결하고, 사설망 간 통신을 IPsec으로
> 보호한 뒤 NAT(PAT)와 Reflexive ACL을 추가 구성한다. 마지막으로 Wireshark로 ISP 구간을
> 캡처하여 ESP 암호화를 검증한다.

## 🗺️ 토폴로지

```
 [R7/PC1]                                                          [R8/PC2]
192.168.10.0/24                                              192.168.20.0/24
    |                                                                |
[R1/GIT-A]──[R3/ISP-1]──[R4/ISP-2]──[R5/ISP-3]──[R6/ISP-4]──[R2/GIT-B]
121.160.10.1   <============ 공중망 (OSPF / Frame-Relay) ============>  121.160.20.2
               <---------------- IPsec VPN Tunnel (ESP) ---------------->
```

| 장비 | 역할 | 장비 | 역할 |
|:---:|:---|:---:|:---|
| R7 | 본사 PC | R8 | 지사 PC |
| R1 | GIT 본사 (GIT-A) | R2 | GIT 지사 (GIT-B) |
| R3 | ISP-1 | R4 | ISP-2 |
| R5 | ISP-3 | R6 | ISP-4 |

---

## EX1) 공중망 구성 (OSPF)

> - OSPF Process=1, Area=0, Router-ID = X.X.X.X (X = Router 번호)
> - ISP-1~4의 Serial, Loopback 0 네트워크는 OSPF에 포함
> - ISP-2의 Loopback 112, ISP-3의 Loopback 113 네트워크 포함
> - 업데이트가 필요한 Interface로만 OSPF Packet 송신
> - 모든 네트워크는 Interface에 할당된 SubnetMask로 확인 (point-to-point)

```cisco
! # ISP-1 (R3)
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface serial 1/0.12
 network 100.100.11.0 0.0.0.255 area 0
 network 121.160.10.0 0.0.0.255 area 0
 network 121.160.12.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

```cisco
! # ISP-2 (R4)
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface serial 1/0.12
 no passive-interface serial 1/0.23
 network 100.100.12.0 0.0.0.255 area 0
 network 112.112.2.0 0.0.0.255 area 0
 network 121.160.12.0 0.0.0.255 area 0
 network 121.160.23.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
interface loopback 112
 ip ospf network point-to-point
```

```cisco
! # ISP-3 (R5)
router ospf 1
 router-id 33.33.33.33
 passive-interface default
 no passive-interface serial 1/0.23
 no passive-interface serial 1/0.34
 network 100.100.13.0 0.0.0.255 area 0
 network 113.113.3.0 0.0.0.255 area 0
 network 121.160.23.0 0.0.0.255 area 0
 network 121.160.34.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
interface loopback 113
 ip ospf network point-to-point
```

```cisco
! # ISP-4 (R6)
router ospf 1
 router-id 44.44.44.44
 passive-interface default
 no passive-interface serial 1/0.20
 no passive-interface serial 1/0.34
 network 100.100.14.0 0.0.0.255 area 0
 network 121.160.20.0 0.0.0.255 area 0
 network 121.160.34.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

### 검증

```cisco
ISP-1# show ip ospf neighbor   ! 인접성 1개 확인
ISP-2# show ip ospf neighbor   ! 인접성 2개 확인
ISP-3# show ip ospf neighbor   ! 인접성 2개 확인
ISP-4# show ip ospf neighbor   ! 인접성 1개 확인
```

> ✅ ISP Router의 Routing table에는 사설 네트워크 정보가 확인되지 않아야 한다.

---

## EX2) Default-route 생성

> - GIT-A는 ISP-1을 통해 외부 네트워크로 통신
> - GIT-B는 ISP-4를 통해 외부 네트워크로 통신

```cisco
! # GIT-A (R1)
ip route 0.0.0.0 0.0.0.0 121.160.10.11

! # GIT-B (R2)
ip route 0.0.0.0 0.0.0.0 121.160.20.4
```

### 검증

```cisco
GIT-A# show ip route
      100.0.0.0/24 is subnetted, 1 subnets
C        100.100.1.0 is directly connected, Loopback0
C    192.168.10.0/24 is directly connected, FastEthernet0/0
      121.0.0.0/24 is subnetted, 1 subnets
C        121.160.10.0 is directly connected, Serial1/0.10
S*   0.0.0.0/0 [1/0] via 121.160.10.11

GIT-A# ping 121.160.20.2
GIT-B# ping 121.160.10.1
```

---

## EX3) IPsec VPN 구성

```text
[Phase 1]                       [Phase 2]
- 인증방식 : 사전 인증방식(PSK)    - IPsec Protocol : ESP
- 암호화   : AES                  - 암호화   : AES
- 인증     : MD5                  - 인증     : SHA-HMAC
- Key교환  : Diffie-Hellman 5
- Key주기  : 10분 (600초)
```

```cisco
! # GIT-A (R1)
access-list 101 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash md5
 group 2
 lifetime 600
!
crypto isakmp key 0 cisco address 121.160.20.2
!
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.20.2
 set transform-set IPSEC
 match address 101
!
interface serial 1/0.10
 crypto map IPSEC_VPN
```

```cisco
! # GIT-B (R2)
access-list 101 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash md5
 group 2
 lifetime 600
!
crypto isakmp key 0 cisco address 121.160.10.1
!
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 121.160.10.1
 set transform-set IPSEC
 match address 101
!
interface serial 1/0.20
 crypto map IPSEC_VPN
```

### 명령어 검증

```cisco
PC1(R7)# ping 192.168.20.2
PC2(R8)# ping 192.168.10.1

GIT-A# show crypto isakmp policy             ! Phase-1 조건 확인
GIT-A# show crypto ipsec transform-set IPSEC ! Phase-2 조건 확인
GIT-A# show crypto isakmp key                ! ISAKMP Key 확인
GIT-A# show crypto isakmp sa                 ! Quick mode 연결 확인
GIT-A# show crypto isakmp peer               ! Phase-1 연결 확인
GIT-A# show crypto engine connection active  ! Phase-2 암/복호화 확인
```

```text
GIT-A# show crypto isakmp key
Keyring    Hostname/Address    Preshared Key
default    121.160.20.2        cisco

GIT-A# show crypto ipsec transform-set IPSEC
Transform set IPSEC: { esp-aes esp-sha-hmac }
   will negotiate = { Tunnel, },

GIT-A# show crypto isakmp peer          ! Phase-1 연결 확인
Peer: 121.160.20.2 Port: 500 Local: 121.160.10.1
 Phase1 id: 121.160.20.2

GIT-A# show crypto engine connection active
ID    Interface  Type  Algorithm  Encrypt Decrypt IP-Address
1     Se1/0.10   IPsec AES+SHA      0       4      121.160.10.1  ← 복호화 확인
2     Se1/0.10   IPsec AES+SHA      4       0      121.160.10.1  ← 암호화 확인
1001  Se1/0.10   IKE   MD5+AES      0       0      121.160.10.1  ← Phase-1 연결 확인
```

---

## 📡 Wireshark 패킷 캡처 검증

IPsec VPN이 정상 동작하면, 공중망(ISP) 구간을 지나는 트래픽은 ESP로 암호화되어
**원본 사설망 데이터를 확인할 수 없다.** ISP-1(R3)의 Serial 인터페이스를 캡처하여 검증한다.

```bash
# Dynagen 콘솔에서 ISP-1의 Serial 1/0 캡처
=> capture R3 s1/0 IPSEC.cap FR
```

![ESP capture](../images/esp-capture.png)

### 패킷 흐름 개요

```text
[Phase-1] ISAKMP Main Mode (6 messages, UDP 500) ── IKE SA 협상
   ↓
[Phase-2] ISAKMP Quick Mode (3 messages, UDP 500) ── IPsec SA 협상
   ↓
[Data]    ESP (IP Protocol 50) ─────────────────── 암호화된 사설망 트래픽
```

### 🔑 ISAKMP Main Mode — 6단계 메시지 1:1 매핑

| 단계 | 패킷 No. | 방향 (Src → Dst) | 길이 | Info | 교환 내용 |
|:---:|:---:|:---|:---:|:---|:---|
| ① | 207 | 121.160.10.1 → .20.2 | 176 | Main Mode | **ISAKMP SA 제안** (암호화·해시·DH·Lifetime 정책) |
| ② | 208 | 121.160.20.2 → .10.1 | 136 | Main Mode | **ISAKMP SA 선택** (수신측 동일 정책 응답) |
| ③ | 209 | 121.160.10.1 → .20.2 | 392 | Main Mode | **DH Key 교환** (Key 재료 + nonce) |
| ④ | 210 | 121.160.20.2 → .10.1 | 392 | Main Mode | **DH Key 교환** (Key 재료 + nonce) |
| ⑤ | 211 | 121.160.10.1 → .20.2 | 124 | Main Mode | **인증** (ID + Hash, 암호화됨) |
| ⑥ | 212 | 121.160.20.2 → .10.1 | 108 | Main Mode | **인증** (ID + Hash, 암호화됨) |

> 💡 **패킷 크기로 단계 추론**
> - ①② : 정책 협상(평문 헤더)이라 비교적 작음
> - ③④ : DH 공개값(group 2 = 1024bit)이 실려 가장 큼 (392 byte)
> - ⑤⑥ : Session key로 ID·인증 정보가 암호화되어 다시 작아짐 → "Identity Protection"

### Quick Mode (Phase-2) & ESP 데이터

```text
No.   Source         Destination    Protocol  Info
213   121.160.10.1   121.160.20.2   ISAKMP    Quick Mode
215   121.160.20.2   121.160.10.1   ISAKMP    Quick Mode
216   121.160.10.1   121.160.20.2   ESP       ESP (SPI=0xa73a5544)  ← 암호화 페이로드
219   121.160.20.2   121.160.10.1   ESP       ESP (SPI=0x8fea22c7)
220   121.160.10.1   121.160.20.2   ESP       ESP (SPI=0xa73a5544)
```

### 🔬 ESP 패킷 Hex 영역 해설

캡처 화면에서 ESP 패킷(Frame 258, 172 bytes)을 선택하면 다음과 같이 해석된다.

```text
Frame 258: 172 bytes on wire (1376 bits), 172 bytes captured
Cisco HDLC
Internet Protocol Version 4, Src: 121.160.20.2, Dst: 121.160.10.1
Encapsulating Security Payload          ← 상위 계층이 ESP로만 표시 (해독 불가)
```

```text
Offset  필드                            해석
------  ------------------------------  ----------------------------------------
0x0000  Cisco HDLC(1871 0800) + IP Hdr  0x45 = IPv4, IHL 20byte
        IP Protocol 필드 = 0x32 (50)    → ESP 임을 의미
0x0014  SPI = a73a5544                  SA 식별 Index
0x0018  Sequence Number                 재생 방지 카운터
0x001C  Payload Data (AES 암호화)        원본 192.168.x.x ICMP가 암호화됨
 ...    [무의미한 랜덤 바이트 연속]       복호화 Key 없이는 평문 복원 불가
0x00A0  ESP Trailer + ICV               패딩 + 무결성 검증값
```

**핵심 관찰 포인트**
1. **IP Header Protocol 필드 = 50** → 상위 프로토콜이 ESP, Wireshark가 TCP/UDP/ICMP를 파싱 불가
2. **SPI** → GIT-A·GIT-B가 협상한 SA 식별값 (`show crypto engine` 출력과 동일)
3. **Payload Data** → 원본 사설 IP·ICMP 시그니처가 Hex/ASCII 어디에도 보이지 않음
4. **ESP Trailer + ICV** → 변조 시 수신측에서 폐기

### ✅ 결론

- ISP 구간(Serial)에서는 패킷이 **ESP로 캡슐화·암호화**되어 원본 사설망 통신을 읽을 수 없다.
- 반대로 GIT 라우터의 **FastEthernet(내부) 구간**을 캡처하면 복호화된 평문 트래픽이 확인되어,
  IPsec이 공중망 구간만 선택적으로 보호함을 알 수 있다.

> 💡 **Wireshark 유용 필터**
> | 목적 | 필터 |
> |:---|:---|
> | ISAKMP(IKE)만 | `isakmp` 또는 `udp.port == 500` |
> | ESP 데이터만 | `esp` 또는 `ip.proto == 50` |
> | 특정 SPI 추적 | `esp.spi == 0xa73a5544` |

---

## EX4) NAT (PAT / Overload) 구성

> - GIT-A의 192.168.10.0/24는 외부 통신 시 Serial1/0.10 공인 IP로 PAT
> - GIT-B의 192.168.20.0/24는 외부 통신 시 Serial1/0.20 공인 IP로 PAT
> - 다수의 내부 사용자가 하나의 공인 IP를 공유하도록 Overload 사용
> - VPN 트래픽(사설↔사설)은 NAT 제외(deny)

```cisco
! # GIT-A (R1)
access-list 110 deny   ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255
access-list 110 permit ip 192.168.10.0 0.0.0.255 any
!
ip nat pool GIT_A 121.160.10.1 121.160.10.1 netmask 255.255.255.0
ip nat inside source list 110 pool GIT_A overload
!
interface fastethernet 0/0
 ip nat inside
interface serial 1/0.10
 ip nat outside
```

```cisco
! # GIT-B (R2)
access-list 110 deny   ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255
access-list 110 permit ip 192.168.20.0 0.0.0.255 any
!
ip nat pool GIT_B 121.160.20.2 121.160.20.2 netmask 255.255.255.0
ip nat inside source list 110 pool GIT_B overload
!
interface fastethernet 0/0
 ip nat inside
interface serial 1/0.20
 ip nat outside
```

### 검증

```cisco
GIT-A# show ip nat translation
PC1(R7)# ping 112.112.2.2    ! 인터넷 망 통신 가능
PC1(R7)# ping 113.113.3.3    ! 인터넷 망 통신 가능
PC1(R7)# ping 192.168.20.2   ! 본사 <--> 지사 통신 가능 (IPsec, NAT 제외)

GIT-B# show ip nat translation
PC1(R8)# ping 112.112.2.2    ! 인터넷 망 통신 가능
PC1(R8)# ping 192.168.10.1   ! 본사 <--> 지사 통신 가능
```

---

## EX5-1) Reflexive ACL 구성 (기본)

> - 내부(192.168.10.0/24)는 외부로 통신 가능, 외부 → 내부는 차단
> - 내부에서 외부로 통신한 트래픽의 응답은 허용
> - IPsec VPN, Static NAT 구간 통신은 가능

```cisco
! # GIT-A (R1)
ip access-list extended IN------------>OUT
 permit ip host 121.160.10.1 any reflect GIT-A
!
ip access-list extended IN<------------OUT
 evaluate GIT-A
!
interface serial 1/0.10
 ip access-group IN------------>OUT out
 ip access-group IN<------------OUT in
```

```cisco
! # GIT-B (R2)
ip access-list extended IN------------>OUT
 permit ip host 121.160.20.2 any reflect GIT-B
!
ip access-list extended IN<------------OUT
 evaluate GIT-B
!
interface serial 1/0.20
 ip access-group IN------------>OUT out
 ip access-group IN<------------OUT in
```

### 검증

```cisco
PC(R7)# ping 112.112.2.2    ! 내부 사설 → 외부 공인 통신 O (Dynamic NAT)
PC(R7)# ping 113.113.3.3    ! 내부 사설 → 외부 공인 통신 O (Dynamic NAT)
ISP-2# ping 121.160.10.1    ! 외부 → 특정 내부 사설 통신 X (Static NAT)
PC(R7)# ping 192.168.20.2   ! 내부 사설 → 지사 사설 통신 O (IPsec VPN)
```

```cisco
GIT-A# show access-list
Reflexive IP access list GIT-A
 permit icmp host 113.113.3.3 host 121.160.10.1 (20 matches) (time left 217)
 permit icmp host 112.112.2.2 host 121.160.10.1 (10 matches) (time left 189)
Extended IP access list IN------------>OUT
 10 permit ip host 121.160.10.1 any reflect GIT-A (18 matches)
Extended IP access list IN<------------OUT
 10 evaluate GIT-A
```

---

## EX5-2) Reflexive ACL + IPsec 허용 추가

> Reflexive ACL이 IPsec 협상 트래픽(UDP 500, ESP)을 차단하지 않도록 inbound ACL에 명시 허용.

```cisco
! # GIT-A (R1)
ip access-list extended IN------------>OUT
 permit ip host 121.160.10.1 any reflect GIT-A
!
no ip access-list extended IN<------------OUT
ip access-list extended IN<------------OUT
 permit udp host 121.160.20.2 eq 500 host 121.160.10.1 eq 500
 permit esp host 121.160.20.2 host 121.160.10.1
 evaluate GIT-A
!
interface serial 1/0.10
 ip access-group IN------------>OUT out
 ip access-group IN<------------OUT in
!
interface serial 1/0.10 point-to-point
 no crypto map IPSEC_VPN
 crypto map IPSEC_VPN
```

```cisco
! # GIT-B (R2)
ip access-list extended IN------------>OUT
 permit ip host 121.160.20.2 any reflect GIT-B
!
no ip access-list extended IN<------------OUT
ip access-list extended IN<------------OUT
 permit udp host 121.160.10.1 eq 500 host 121.160.20.2 eq 500
 permit esp host 121.160.20.2 host 121.160.10.1
 evaluate GIT-B
!
interface serial 1/0.20
 ip access-group IN------------>OUT out
 ip access-group IN<------------OUT in
!
interface serial 1/0.20 point-to-point
 no crypto map IPSEC_VPN
 crypto map IPSEC_VPN
```

### 검증

```cisco
PC1# ping 192.168.20.2
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 100/112/124 ms

GIT-A# show crypto isakmp peer
Peer: 121.160.20.2 Port: 500 Local: 121.160.10.1
 Phase1 id: 121.160.20.2

GIT-A# show crypto engine connections active
ID    Interface  Type  Algorithm  Encrypt Decrypt IP-Address
15    Se1/0.10   IPsec AES+SHA      0       4      121.160.10.1
16    Se1/0.10   IPsec AES+SHA      4       0      121.160.10.1
1007  Se1/0.10   IKE   MD5+AES      0       0      121.160.10.1
```
