# 03. AH (Authentication Header, 인증만)

- AH Protocol은 IP를 기반으로 통신하는 Protocol이며 **Protocol number 51**번을 사용한다.
- 기밀성(암호화)은 보장하지 않으며, IP Header에 포함된 데이터의 **무결성과 데이터 인증 및 보호**를 제공한다.
  (여기서 인증은 세션에 대한 인증이 아닌 Packet의 무결성을 의미) 즉 데이터를 확인할 수는 있지만 위조/변조는 할 수 없다.

> ⚠️ AH 프로토콜은 IP Header의 일부(TTL, TOS)를 제외한 IP Header 전체를 인증하기 때문에
> IP Header 조작이 불가능하므로 **NAT 환경에서 적용할 수 없다.**
> (NAT는 IP Header의 Source Address를 변경하는 기능이므로 사용 불가)

- 인증 알고리즘은 MD5와 SHA 등의 HASH 알고리즘을 사용한다.

## AH Header 위치

```text
[ Transport Mode ]
+-----------+-----------+------------+------+
| IP Header | AH Header | TCP Header | DATA |
+-----------+-----------+------------+------+
|<-------------- Authentication ----------->|

[ Tunnel Mode ]
+---------------+-----------+-----------+------------+------+
| New IP Header | AH Header | IP Header | TCP Header | DATA |
+---------------+-----------+-----------+------------+------+
|<------------------------ Authentication ----------------->|
```

## AH Header 구조

```text
==================================================================
| Next-Header (8bit) | Payload Length (8bit) | Reserved (16bit)   |
------------------------------------------------------------------
|         SPI (Security Parameter Index) (32bit)                  |
------------------------------------------------------------------
|         Sequence Number (32bit)                                 |
------------------------------------------------------------------
|         ICV (Integrity Check Value) (Variable)                  |
==================================================================
```

| 필드 | 설명 |
|:---|:---|
| **Next-Header** (8bit) | AH 뒤의 상위 Protocol 확인 (ICMP=1, TCP=6, UDP=17...) |
| **Payload Length** (8bit) | AH Header의 크기 표기 |
| **Reserved** (16bit) | 아직 사용하지 않는 필드 |
| **SPI** (32bit) | 목적지 IP 주소와 조합하여 생성되는 SA를 구분/식별하기 위한 Index |
| **Sequence Number** (32bit) | 1부터 시작, Packet마다 1씩 증가 (재생 방지). 2³² 도달 시 새 SA 협상 |
| **ICV** (Variable) | 데이터 변조 확인용 Hash code값. IP Header 일부(TTL, TOS) 제외 전체 인증, AH 뒤 모든 데이터의 Hash code 계산 |
