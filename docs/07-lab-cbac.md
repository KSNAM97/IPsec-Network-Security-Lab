# 07. 실습: CBAC (Context-Based Access Control)

> CBAC는 Stateful Inspection 방화벽 기능으로, 내부에서 외부로 나간 트래픽의 세션 상태를
> 추적하여 응답 트래픽만 동적으로 허용한다. (사설 IP + NAT 환경)

## EX1) 공중망 구성 (OSPF)

> 06장과 동일한 백본 구성. Router-ID는 GIT=X.X.X.X, ISP=XX.XX.XX.XX 형식.

```cisco
! # GIT-A (R1)
router ospf 1
 router-id 1.1.1.1
 passive-interface default
 no passive-interface serial 1/0.10
 network 100.100.1.0 0.0.0.255 area 0
 network 121.160.10.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

```cisco
! # GIT-B (R2)
router ospf 1
 router-id 2.2.2.2
 passive-interface default
 no passive-interface serial 1/0.20
 network 100.100.2.0 0.0.0.255 area 0
 network 121.160.20.0 0.0.0.255 area 0
!
interface loopback 0
 ip ospf network point-to-point
```

### 검증

```cisco
GIT-A# ping 100.100.2.2 source 100.100.1.1
!!!!!
Success rate is 100 percent (5/5)

GIT-B# ping 100.100.1.1 source 100.100.2.2
!!!!!
Success rate is 100 percent (5/5)
```

---

## EX2) NAT (PAT) 구성

> 내부 네트워크가 외부 통신 시 물리 Interface의 IP로 변환.

```cisco
! # GIT-A (R1)
access-list 1 permit 192.168.1.0 0.0.0.255
!
ip nat pool NAT 121.160.10.1 121.160.10.1 netmask 255.255.255.0
ip nat inside source list 1 pool NAT overload
!
interface fastethernet 0/0
 ip nat inside
interface serial 1/0.10
 ip nat outside
```

```cisco
! # GIT-B (R2)
access-list 1 permit 192.168.2.0 0.0.0.255
!
ip nat pool NAT 121.160.20.2 121.160.20.2 netmask 255.255.255.0
ip nat inside source list 1 pool NAT overload
!
interface fastethernet 0/0
 ip nat inside
interface serial 1/0.20
 ip nat outside
```

### 검증

```cisco
PC1# ping 100.100.2.2 source 192.168.1.1
PC1# ping 100.100.2.2 source 192.168.1.2

GIT-A# show ip nat translation
Pro  Inside global       Inside local      Outside local    Outside global
icmp 121.160.10.1:23   192.168.1.1:23   100.100.2.2:23   100.100.2.2:23
icmp 121.160.10.1:24   192.168.1.2:24   100.100.2.2:24   100.100.2.2:24
```

---

## EX3) CBAC 구성

> - 모든 TCP/UDP/ICMP에 대해 내부 → 외부 허용, 외부 → 내부 차단
> - 내부 → 외부 TCP/ICMP 트래픽은 Log-message 출력
> - 외부 → 내부 차단 트래픽은 모두 Log-message 출력

```cisco
! # GIT-A (R1)
ip access-list extended ACL_IN
 permit ospf host 121.160.10.11 host 224.0.0.5
 deny ip any any log-input
!
ip inspect name GIT-A_IPS tcp  audit-trail on
ip inspect name GIT-A_IPS udp
ip inspect name GIT-A_IPS icmp audit-trail on
!
interface serial 1/0.10
 ip access-group ACL_IN in
 ip inspect GIT-A_IPS out
```

### 검증

```cisco
PC1(R7)# ping 100.100.2.2 source 192.168.1.1   ! 통신 O
PC1(R7)# telnet 100.100.2.2                     ! 접속 O

GIT-A# show ip inspect session
Established Sessions
 Session (192.168.1.1:23234)=>(200.20.2.2:23) tcp SIS_OPEN
 Session (192.168.1.2:8)=>(200.20.2.2:0) icmp SIS_OPEN
 Session (192.168.1.1:8)=>(200.20.2.2:0) icmp SIS_OPEN

PC2(R8)# ping 100.100.1.1     ! 통신 X
PC2(R8)# telnet 100.100.1.1   ! 접속 X
```

```text
! 외부 → 내부 차단 로그
%SEC-6-IPACCESSLOGDP: list ACL_IN denied icmp 121.160.20.2 (Serial1/0.10) -> 100.100.1.1 (0/0), 1 packet
%SEC-6-IPACCESSLOGP:  list ACL_IN denied tcp 121.160.20.2(40529) (Serial1/0.10) -> 100.100.1.1(23), 1 packet
```

---

## EX4) Static NAT + 특정 트래픽 허용

> - GIT-B 사설망에서만 GIT-A 내부 서버(192.168.1.100)로 통신 가능
> - 통신 시 GIT-A의 Loopback 0 IP(100.100.1.1) 사용
> - 나머지 모든 네트워크는 192.168.1.100으로 통신 불가

```cisco
! # PC1 (R7)
interface fastethernet 0/1
 ip address 192.168.1.100 255.255.255.0 secondary
```

```cisco
! # GIT-A (R1)
ip nat inside source static 192.168.1.100 100.100.1.1
!
ip access-list extended ACL_IN
 permit ospf host 121.160.10.11 host 224.0.0.5
 15 permit ip host 121.160.20.2 host 100.100.1.1   ! GIT-B IP 허용 추가
 deny ip any any log-input
```

### 검증

```cisco
PC2(R8)# ping 100.100.1.1     ! 통신 O
PC2(R8)# telnet 100.100.1.1   ! 접속 O

ISP-1# ping 100.100.1.1       ! 통신 X
ISP-4# ping 100.100.1.1       ! 통신 X
```

---

## EX5) IPsec 구성 (Loopback 사용)

> - GIT-A IPsec 구성 시 Loopback 1 (100.100.100.1/24) 사용
> - GIT-B IPsec 구성 시 Loopback 0 사용

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
interface loopback1
 ip address 100.100.100.1 255.255.255.0
router ospf 1
 network 100.100.100.0 0.0.0.255 area 0
!
access-list 101 permit ip 192.168.1.0 0.0.0.255 192.168.2.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash md5
 group 5
 lifetime 600
!
crypto isakmp key 0 cisco address 100.100.2.2
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 100.100.2.2
 set transform-set IPSEC
 match address 101
crypto map IPSEC_VPN local-address loopback 1
!
interface serial 1/0.10
 crypto map IPSEC_VPN
```

```cisco
! # GIT-B (R2)
access-list 101 permit ip 192.168.2.0 0.0.0.255 192.168.1.0 0.0.0.255
!
crypto isakmp enable
crypto isakmp policy 10
 authentication pre-share
 encryption aes
 hash md5
 group 2
 lifetime 600
!
crypto isakmp key 0 cisco address 100.100.100.1
crypto ipsec transform-set IPSEC esp-aes esp-sha-hmac
!
crypto map IPSEC_VPN 10 ipsec-isakmp
 set peer 100.100.100.1
 set transform-set IPSEC
 match address 101
crypto map IPSEC_VPN local-address loopback 0
!
interface serial 1/0.20
 crypto map IPSEC_VPN
```

### 검증

```cisco
PC1# ping 192.168.2.2 source 192.168.1.2
.!!!!
Success rate is 80 percent (4/5), round-trip min/avg/max = 84/88/92 ms

GIT-A# show crypto isakmp peer
Peer: 100.100.2.2 Port: 500 Local: 100.100.100.1
 Phase1 id: 100.100.2.2

GIT-A# show crypto engine connections active
ID    Interface  Type  Algorithm  Encrypt Decrypt IP-Address
1     Se1/0.10   IPsec AES+SHA      0       9      100.100.100.1
2     Se1/0.10   IPsec AES+SHA      9       0      100.100.100.1
1001  Se1/0.10   IKE   MD5+AES      0       0      100.100.100.1
```
