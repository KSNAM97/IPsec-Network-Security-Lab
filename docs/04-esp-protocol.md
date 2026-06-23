# 04. ESP (Encapsulating Security Payload)

- ESP는 **기밀성(암호화)**, 원본 데이터의 **인증·무결성**과 같은 보안 서비스를 지원하기 위해
  설계된 프로토콜로, IP 데이터그램 안에 포함되며 **Protocol number 50**번을 사용한다.
- AH는 데이터의 인증 기능만 제공하지만, ESP는 데이터 암호화 기능과 함께 인증 기능도 포함하기
  때문에 일반적으로 AH보다 많이 사용된다.
- ESP만으로도 암호화 및 인증을 실시할 수 있지만, 보안 강도를 높일 때는 AH와 병행 사용이 가능하다.
  (다만 병행 사용 시 throughput이 증가하여, 현재는 병행하지 않는 것이 일반적)

## ESP Header 위치

```text
[ Transport Mode ]
+-----------+------------+------------+------+-------------+----------+
| IP Header | ESP Header | TCP Header | DATA | ESP Trailer | ESP 인증 |
+-----------+------------+------------+------+-------------+----------+
                         |<-------- 암호화 ------->|
            |<------------------ 인증 -------------------->|

[ Tunnel Mode ]
+-------------+----------+----------+----------+----+-----------+--------+
|New IP Header|ESP Header|IP Header |TCP Header|DATA|ESP Trailer|ESP auth|
+-------------+----------+----------+----------+----+-----------+--------+
                        |<-------------- 암호화 ------------->|
            |<------------------------ 인증 ------------------------->|
```

## ESP Header 구조

```text
==================================================================
|         SPI (Security Parameter Index) (32bit)                  |
------------------------------------------------------------------
|         Sequence Number (32bit)                                 |
------------------------------------------------------------------
|         Payload Data (Variable)                                 |
------------------------------------------------------------------
| Padding (Variable) | Padding Length (8bit) | Next-Header (8bit) |  ← ESP Trailer
------------------------------------------------------------------
|         Authentication Data (Variable)                          |
==================================================================
```

| 필드 | 설명 |
|:---|:---|
| **SPI** (32bit) | SA 식별용 Index. 암호화 알고리즘, 인증 알고리즘, 보호할 네트워크 정보 등 포함 |
| **Sequence Number** (32bit) | 1부터 시작, Packet마다 1씩 증가 (재생 방지) |
| **Payload Data** (Variable) | 원본 IP Header와 데이터 위치 |
| **Padding** (Variable) | 암호화 알고리즘에 맞춰 크기를 맞추기 위한 완충재 (택배 빈공간 채우는 역할) |
| **Padding Length** (8bit) | Padding 길이(Byte) |
| **Next-Header** (8bit) | ESP Header 뒤의 Protocol 종류 표기 |
| **Authentication Data** (ICV) | 변조 방지 Hash code값. 가변값 |

> **ESP Trailer** = Padding + Padding Length + Next-Header

## HMAC (Hash-based Message Authentication Code)

```text
- HMAC-MD5-96 : 128bit Hash Code 생성 → 앞 96bit만 잘라서 사용
- HMAC-SHA-96 : 160bit Hash Code 생성 → 앞 96bit만 잘라서 사용

  수신측에서는 동일한 알고리즘으로 계산한 후 앞의 96bit만 비교한다.
```
