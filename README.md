# 🔐 IPsec & Network Security Lab

[![Cisco](https://img.shields.io/badge/Cisco-1BA0D7?style=flat&logo=cisco&logoColor=white)](https://www.cisco.com/)
[![IPsec](https://img.shields.io/badge/IPsec-VPN-blue?style=flat)](https://datatracker.ietf.org/wg/ipsec/about/)
[![GNS3](https://img.shields.io/badge/GNS3-Dynamips-green?style=flat)](https://www.gns3.com/)
[![Wireshark](https://img.shields.io/badge/Wireshark-1679A7?style=flat&logo=wireshark&logoColor=white)](https://www.wireshark.org/)
[![Protocol](https://img.shields.io/badge/Protocol-ESP%20%7C%20AH-orange?style=flat)](https://datatracker.ietf.org/doc/html/rfc4301)
[![License](https://img.shields.io/badge/License-MIT-lightgrey?style=flat)](LICENSE)
[![Docs](https://img.shields.io/badge/Docs-한국어-red?style=flat)](README.md)

> Cisco 라우터 기반의 IPsec VPN, NAT, CBAC, Reflexive ACL 구성 실습 및 통신 보안(암호 시스템) 이론 정리

---

## 📖 소개

오늘날 인터넷은 통신과 정보 전달을 위해 가장 효과적이고 많이 사용되는 정보 전달 수단이다.
Internet망을 통해 많은 사람이 통신함에 따라 **통신 보안**은 매우 중요한 이슈가 되었으며,
데이터를 암호화하는 기법들은 현대 정보 시스템의 필수 요소로서 **안정성·신뢰성·기밀성**을 제공한다.

본 저장소는 통신 보안의 핵심인 **암호 시스템 이론**과 Cisco IOS 라우터(GNS3/Dynamips)를
이용한 **IPsec VPN 사이트 투 사이트 구성 실습**을 정리한 자료다. Wireshark로 ESP 암호화
트래픽을 직접 검증하는 과정까지 포함한다.

---

## 📚 목차

| 챕터 | 제목 | 내용 |
|:---:|:---|:---|
| [01](docs/01-crypto-system.md) | 암호 시스템 (Crypto System) | 대칭키 / 비대칭키 / 해시 알고리즘 |
| [02](docs/02-ipsec-vpn.md) | IPsec VPN 개요 | Transport / Tunnel 모드 |
| [03](docs/03-ah-protocol.md) | AH (Authentication Header) | 무결성 / 인증, 헤더 구조 |
| [04](docs/04-esp-protocol.md) | ESP (Encapsulating Security Payload) | 암호화 + 인증, 헤더 구조 |
| [05](docs/05-isakmp-ike.md) | ISAKMP / IKE | SA, Phase-1(Main/Aggressive), Phase-2(Quick) |
| [06](docs/06-lab-ipsec-nat.md) | 실습: IPsec + NAT (사설 IP) | 공중망 → VPN → PAT → Reflexive ACL + Wireshark |
| [07](docs/07-lab-cbac.md) | 실습: CBAC | Stateful Inspection 방화벽 |
| [08](docs/08-isakmp-profile-ezvpn.md) | ISAKMP Profile / EzVPN | VPN 솔루션 비교 |

---

## 🗺️ 토폴로지

```
 [R7/PC1]                                                          [R8/PC2]
192.168.10.0/24                                              192.168.20.0/24
    |                                                                |
[R1/GIT-A]──[R3/ISP-1]──[R4/ISP-2]──[R5/ISP-3]──[R6/ISP-4]──[R2/GIT-B]
 본사 게이트웨이      <======= 공중망 (OSPF / Frame-Relay) =======>     지사 게이트웨이
                    <------------- IPsec VPN Tunnel (ESP) ------------->
```

| 장비 | 역할 | 장비 | 역할 |
|:---:|:---|:---:|:---|
| R1 | GIT 본사 (GIT-A) | R5 | ISP-3 |
| R2 | GIT 지사 (GIT-B) | R6 | ISP-4 |
| R3 | ISP-1 | R7 | 본사 PC (PC1) |
| R4 | ISP-2 | R8 | 지사 PC (PC2) |

---

## ⚙️ 실습 환경

- **에뮬레이터**: GNS3 / Dynamips
- **장비**: Cisco IOS Router (Frame-Relay 백본)
- **분석 도구**: Wireshark (ESP / ISAKMP 패킷 캡처)

---

## ✅ 핵심 검증 포인트

- ISP 구간 캡처 시 → **ESP로 암호화되어 페이로드 확인 불가** ([캡처 분석](docs/06-lab-ipsec-nat.md#-wireshark-패킷-캡처-검증))
- GIT 라우터 FastEthernet 구간 캡처 시 → **복호화되어 평문 확인 가능**
- `show crypto engine connection active` → 암/복호화 카운터 확인

---

## 📄 라이선스

MIT License
