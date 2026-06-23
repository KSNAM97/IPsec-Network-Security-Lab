# 08. ISAKMP Profile / CISCO EzVPN

## ISAKMP Profile

- IKE-1 협상을 위해 **모듈 방식 설정**을 제공하여 ISAKMP parameter 값들을 IPsec Tunnel에 Mapping한다.
- 일반적으로 두 대 이상의 장비에 IPsec Tunnel을 가질 때 사용된다. (서로 다른 사이트마다 다른
  IKE-1 parameter 값을 요구하는 경우)
- EzVPN 원격 설정과 VRF 인식 IPsec 설정을 함께 사용할 수 있다.
- `crypto-map` 명령어 없이 Tunnel interface에 IKE-2의 SA parameter를 결합시키기 위해 개발되었다.
- Encryption은 GRE Tunnel을 경유하는 데이터에 New IP Header가 추가된 후 이루어진다.
- Point-to-point와 MGRE(Multipoint GRE)로 구성 가능하다.
- `crypto-map` 형식과 유사하지만 **Tunnel 생성 전 Peer 주소를 알 필요가 없다.**
  `set peer`, `match address` command를 사용하지 않고, IOS software가 Tunnel parameter와
  NHRP cache로부터 동적으로 수신한다.
- 확장성이 뛰어나며 다양한 환경에서 효과적인 사용이 가능하다.

---

## CISCO EzVPN (Easy VPN)

### VPN 유형 구분

| 유형 | 설명 |
|:---|:---|
| **LAN-to-LAN / Site-to-Site / Gateway-to-Gateway** | 본사-지사 또는 네트워크 간 사설망을 연결하기 위해 사용하는 VPN |
| **EzVPN (Access VPN / Client VPN)** | 개인이 외부에서 회사 내부로 통신 시, 일반 회선 사용으로 데이터가 노출되는 문제를 해결하기 위해 개발된 기능 |

### 특징

- 원격 사용자·원격 사무실·원격 근무자를 위해 간편한 원격접속 **Point-to-point VPN 솔루션**을 제공.
- 중앙 집중적 VPN 관리와 쉬운 provisioning이 가능하여 구성 복잡성을 단순화 → 확장성·유연성 우수.
- EZVPN Client 프로그램으로 접속 시 송/수신 데이터가 모두 암호화·인증되어 안전한 통신 가능.
- **2중 보안을 위해 Phase-1.5를 사용**한다.
  - Phase-1 SA 연결 후 개인 계정과 Password를 사용하여 Phase-1.5 연결
  - Phase-1.5 연결 후 Phase-2를 사용하여 데이터 보호

---

## VPN 솔루션 비교

```text
==========================================================================
            | Crypto-map  | GRE-over-IPsec | DMVPN        | EzVPN
--------------------------------------------------------------------------
 표준       | RFC 준수    | 가능(일반적X)  | 가능(일반적X)| 가능(일반적X)
 사용환경   | 비 IOS Peer | Site-to-site   | Hub & Spoke  | Remote Access
 Multicast  | 지원 X      | 지원 O         | 지원 O       | 지원 O
==========================================================================
```
