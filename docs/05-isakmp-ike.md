# 05. ISAKMP (키 동기화) / IKE

## 배경 — Key 공유 문제

IPsec은 AH와 ESP를 사용하여 통신한다. 두 프로토콜 모두 송신자와 수신자가 같은 Key를 공유하여
Hash 인증 또는 대칭키 암호화를 수행한다. 여기서 **동일한 Key를 안전하게 나누어 가지는 방법**에
문제가 발생한다. 다음 항목들을 협상하여 결정해야 한다.

```text
- 통신 방식      : AH? ESP?
- 암호화 알고리즘 : DES? 3DES? AES?
- 인증 알고리즘   : MD5? SHA?
- 인증/암호화 Key : 어떤 값을 사용할지
- Key 길이 / 유효기간
```

또한 Key를 오랜 기간 사용하면 **crypto analysis(암호 분석) 공격**에 노출되므로 주기적으로
Session key를 교체해야 한다.

- **ISAKMP** : 위와 같은 일을 하는 **절차**를 명시한 Protocol
- **IKE** : ISAKMP가 명시한 절차에 필요한 **구체적 Protocol 종류와 사용 방법**을 정의

> ISAKMP와 IKE는 규정 내용 중 동일한 기능이 많아 혼용하여 사용하는 경우가 많다.

---

## ISAKMP (Internet Security Association And Key Management Protocol)

- Key 관리를 위한 프레임워크로, SA(Security Association)의 **설정·협상·변경·삭제**에 필요한
  절차와 Packet 구조를 정의하고, Peer들만 식별할 수 있는 기능을 제공한다.
- **Key 교환 메커니즘은 제공하지 않는다.**

## IKE (Internet Key Exchange)

- **UDP Port number 500**번을 사용한다. (방화벽 사용 시 UDP 500 허용해야 IPsec 통신 가능)
- ISAKMP와 Oakley를 조합한 Hybrid Protocol로, Key 교환과 생성에 대한 메커니즘을 정의한다.
- 인증된 Key 정보를 추출하여 ESP, AH Protocol에 사용되는 SA를 협상한다.

---

## SA (Security Association) — 장비 인증

- IPsec 구성 시 AH/ESP 방식으로 통신할 때 인증 또는 인증·암호화에 사용할 **보안 관련 요소들의 집합**.
- 관리자가 수동으로 설정하거나 IKE를 사용하여 자동 생성할 수 있다.
  - **수동 설정** : 생성 후 바로 연결, 관리자가 수동 변경 전까지 유지 (보안에 취약할 수 있음)
  - **IKE 사용** : SA를 자동 생성·교환·협상, 관리자가 지정한 주기로 갱신
- 갱신 조건: 관리자 지정 **시간** 도달 시, 또는 특정 양 이상의 **데이터**(4,608,000 KByte) 처리 시
- ISAKMP의 SA 교환은 단방향이며 **2단계의 SA Session**을 연결해야 한다.

```text
IKE SA = Phase-1 (IKE-1) + Phase-2 (IKE-2)
```

- **Phase-1** : VPN Peer 간 인증 및 안전한 경로를 설정하고 Phase-2 통신 협상 정보를 제공.
  Phase-2의 Negotiation 과정을 보호한다.
- **Phase-2** : 실제 security protocol들이 사용할 SA를 개설하는 과정을 담당.

---

## Phase-1 (IKE-1)

협상 내용:
- ISAKMP의 SA 보호 (암호화 알고리즘 + Hash 알고리즘 인증)
- Session key 생성 (Diffie-Hellman group 사용)
- 인증 방법: Pre-shared key / Public Key encryption / Public Key signature

Phase-1은 **Main mode** 또는 **Aggressive mode**로 협상할 수 있다.

### Main Mode (6개 메시지 교환)

가장 많이 사용되는 방식.

```text
GIT-A                                          GIT-B
  |---- (1) ISAKMP SA 제안 -------------------->|
  |<--- (2) ISAKMP SA 선택 ---------------------|
  |---- (3) Diffie-Hellman Key 교환 ----------->|
  |<--- (4) Diffie-Hellman Key 교환 ------------|
  |---- (5) Authentication (인증) ------------->|
  |<--- (6) Authentication (인증) --------------|
```

| 단계 | 설명 |
|:---:|:---|
| (1) | 통신 시작 측에서 ISAKMP SA 생성·제안 (Authentication 방식, Encryption, HASH, DH-Group, Life-time 포함) |
| (2) | 수신 VPN 장비가 자신의 설정 중 동일한 값 확인 후 동일 SA 전송 (GIT-A 수신 SA와 동일 설정값이어야 함) |
| (3) | DH 알고리즘으로 Encryption/Hashing용 Key 생성. 공개키·비밀키로 만든 nonce 전송 (nonce = 한 번만 사용되는 값) |
| (4) | (3)과 동일하게 반대 방향에서 DH Key 생성·nonce 전송 |
| (5) | 교환한 Key로 암호화·HASH하여 송신 (해독/변경 불가). GIT-A가 ID·인증 정보를 암호화하여 전송. PSK 방식이면 ID와 암호로 Hash Code 생성·전송 |
| (6) | GIT-B가 동일하게 ID·인증 정보를 암호화하여 전송 |

> 💡 (5)(6)부터는 ID가 암호화되므로 Main Mode를 **"Identity Protection"** 이라 부른다.

### Aggressive Mode (3개 메시지 교환)

```text
GIT-A                                          GIT-B
  |---- (1) ISAKMP SA, DH Key, ID ------------>|
  |<--- (2) ISAKMP SA, DH Key, ID -------------|
  |---- (3) Authentication (인증) ------------->|
```

- (1) ISAKMP SA 생성·제안 (Authentication, Encryption, HASH, DH-Group, VPN mode, Encapsulation, 보호대상 포함)
- (2) 수신 장비가 동일 값 확인 후 SA 전송
- (3) GIT-A가 인증과 ID 정보를 전송하면서 연결 완료

> Main mode보다 빠르지만 보안 측면에서 안전하지 못하다.

---

## Phase-2 (IKE-2) — Quick Mode

Phase-1 협상이 종료되면 실제 송/수신 데이터를 보호하기 위해 Phase-2를 협상한다.
협상 내용: Protection suite(AH, ESP), Protection suite algorithm(DES/3DES/AES/MD5/SHA),
보호되어야 할 네트워크 및 IP 주소.

Phase-2는 **Quick mode** 한 가지만 지원한다.

```text
GIT-A                                          GIT-B
  |---- (1) IPsec SA 제안 --------------------->|
  |<--- (2) IPsec SA 제안 ----------------------|
  |---- (3) 수신 확인 ------------------------->|
```

| 단계 | 설명 |
|:---:|:---|
| (1) | 통신 시작 측에서 Data 보호용 IPsec SA 생성·제안 (Encryption, HASH, 보호할 Network 대역, VPN mode, Encapsulation 포함) |
| (2) | 수신 VPN 장비도 Data 보호용 IPsec SA 생성·제안 (동일 항목 포함) |
| (3) | GIT-A가 GIT-B가 동의한 IPsec SA에 대해 수신 확인 Message 전송 |
