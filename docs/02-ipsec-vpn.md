# 02. IPsec VPN 개요

IPsec은 IETF가 개발한 공개 표준으로, IP 환경에서 트래픽 보호를 위해 암호화 및 인증
Protocol을 사용한다. 데이터 전송 시 크게 **Transport mode**와 **Tunnel mode** 두 가지를 지원한다.

## Transport Mode

- Host와 Host 종단 장비 간 사이에서 통신 시 데이터를 보호한다.
- Peer-to-Peer로 구성되며 IP Header를 제외한 나머지 데이터에 대한 암호화 및 인증 기능을 지원한다.
- 원본 IP Header를 유지하면서 IP Header와 Data 사이에 IPsec Header를 삽입한다.

```text
[원본]
+-----------+------------+------+
| IP Header | TCP Header | DATA |
+-----------+------------+------+

[Transport Mode 적용]
+-----------+-------------+------------+------+
| IP Header | IPsec Hdr   | TCP Header | DATA |
+-----------+-------------+------------+------+
```

## Tunnel Mode (LAN to LAN)

- 네트워크와 네트워크 구간 사이에서 통신 시 데이터를 보호한다.
- **LAN-to-LAN, Site-to-Site, Gateway-to-Gateway** 라고 부른다.
- VPN 장비를 통과하는 모든 데이터를 보호하며, 원본 IP Header 앞에 New IP Header를
  encapsulation하고 New IP Header 뒤의 데이터에 대한 암호화 및 인증 기능을 지원한다.

```text
[원본]
+-----------+------------+------+
| IP Header | TCP Header | DATA |
+-----------+------------+------+

[Tunnel Mode 적용]
+---------------+-----------+-----------+------------+------+
| New IP Header | IPsec Hdr | IP Header | TCP Header | DATA |
+---------------+-----------+-----------+------------+------+
```

## 전송 프로토콜

IPsec VPN은 통신 시 **AH**, **ESP** Protocol을 사용하여 전송이 가능하다.

| 프로토콜 | Protocol number | 기능 |
|:---:|:---:|:---|
| AH | 51 | 무결성 + 인증 (암호화 ✕) |
| ESP | 50 | 암호화 + 무결성 + 인증 |
