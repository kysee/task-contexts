---
last_updated: 2026-05-28
status: 설계 탐색 중 (미구현)
related: ./btips.md, ./btips-2pc-design.md
---

# BPuN-origin "이벤트 → BPrN 결제" 서비스 설계 연구

> 2026-05-27 세션의 설계 논의·검토 대안·결정·발견한 함정을 자기완결적으로 정리. **아직 미구현(설계 탐색 단계)** 이며 다음 세션에서 이어가기 위한 노트. 인접 주제(2PC)의 확정 사항은 `./btips-2pc-design.md` §9 참조.
>
> **2026-05-28 업데이트**: §11(use-case independent reframing — `Pay` → `publish` 리네임, 두 층 분리, 파라미터 Model A 확정), §12(BPrN-origin STC use case 분석 — IntendedHandler 결정 패턴, timeout 불필요 검증, phishing 분석) 추가. §10은 부분 해결 표시 + 신규 OPEN 항목 추가. §7의 `Pay` 어휘는 `publish`로 대체됨.

---

## 1. 풀려는 문제

- BPuN에서 발생한 **이벤트 X**에 대응하여, **BPrN의 토큰 체인코드에서 결제(지불)** 가 일어나게 하는 서비스.
- **결제 자금은 BPrN에 있음.** BPuN 이벤트 X는 트리거.
- X는 임의적이나 사전 정의됨. 현재는 X에 어떤 제약도 두지 않으려 함.
- 목표: **payer = X를 발생시킨 BPuN 계정**(EOA 또는 **컨트랙트**)이 되도록, 그리고 **ccApp/BPrN 측 화이트리스트 관리를 피하고** 싶음.

## 2. 기본 구조 (현재 프로토콜로 가능)

새 프로토콜 프리미티브 불필요. BPuN→BPrN 정방향 전달(btip-27/28/29/31/33/34) + **executor ccApp**:
1. BPuN dApp이 X emit → Prover가 `BPuNTxEventProofPayload` 생성 → btip-29 `OnProof(payload, executor)`.
2. btip-29: 검증 → Nullifier 기록 → `executor.HandleLinkerEvent(srcChainId,…,indices,values)`.
3. executor: 호출자==공식 LinkerEndpoint 확인 → **이벤트 출처/조건 확인** → 토큰 체인코드 `Transfer` 호출.

**프로토콜은 "X가 발생했다"만 증명**한다. BPrN 자금을 움직일 **권한 모델은 앱 계층**의 몫. (링커는 결제를 직접 하지 않음)

## 3. 핵심 원리 — 신뢰는 제거 불가, 재배치만 가능

"출처를 신뢰한다"는 곧 "그 출처(컨트랙트)의 내부 규칙을 신뢰한다". 신원만 믿고 규칙이 허술하면 악용됨. 따라서 BPrN 결제의 권한은 반드시 다음 중 하나에 **앵커**되어야 함:
- (a) **BPuN에 잠긴 값의 보존**(브리지/스왑: BPuN lock/burn ⟺ BPrN release), 또는
- (b) **BPrN 자금 소유자의 명시적 인가**(조건/승인/서명).

→ 본 케이스는 자금이 **BPrN 소유자** 것이므로 **(b)** 가 맞다.

## 4. 화이트리스트 회피 — 검토한 방법

### 방법 A — 조건부 에스크로 (per-instance, 자금주가 조건 지정)
- BPrN 자금 소유자가 lock 시 **방출 조건** `(srcChainId, contractY, topic.0=X[, indexed])` 를 에스크로에 기록.
- executor는 검증된 이벤트가 **그 에스크로의 조건**과 맞는지만 비교 → 방출. **전역 화이트리스트 없음.**
- 단 payer = "조건을 건 BPrN 자금 소유자"이지 "X의 트리거러"는 아님 (의미 차이 주의).

### 방법 B — 단일 정규 중개 컨트랙트 (레지스트리로 신뢰)
- BPuN에 공식 컨트랙트 하나 + LinkerRegistry role 등록 → ccApp은 그 하나만 신뢰.
- **함정(사용자 지적)**: 정규 컨트랙트가 아무 호출에나 자유롭게 emit하면 악의적 dApp이 호출해 결제 유발 가능. → **정규 컨트랙트는 반드시 "값 보존 에스크로"** 여야 함(BPuN lock/burn 강제). 값이 BPuN→BPrN으로 흐를 때만 적합. 본 케이스(자금이 BPrN 소유자)엔 부적합.

## 5. payer 식별 분석 (payer = 트리거러로 강제하려면)

- **tx.origin / tx.sender**: 항상 EOA → **컨트랙트 payer 배제**. 게다가 tx.origin 인가는 **피싱 안티패턴**. ✗
- **contractAddress(emitter, evm 이벤트 index 0)**: **컨트랙트 payer만** 커버. EOA payer면 contractAddress = (EOA가 호출한) 컨트랙트이지 EOA가 아님 → 엉뚱한 계정 차감. ✗ (EOA 단독 불가)
- 표준 evm 이벤트엔 **"msg.sender(EOA든 컨트랙트든 = 호출자)"를 담는 비위조 필드가 없음.** → 노드가 노출하거나(§6), 정규 Pay로 Solidity에서 캡처(§7) 해야 함.

## 6. 노드 레벨 msg.sender 캡처 (OnOpcode) — **PARKED, 다음에 재검토**

- `evmLogsToEvent`는 msg.sender를 모름(`types.Log`에 caller 없음, `Address`=emitter만).
- 실행 중 **LOG opcode 시점**에서만 알 수 있음: `scope.Caller()`=msg.sender, `scope.Address()`=emitter.
- go-ethereum **`tracing.Hooks`(OnEnter/OnExit/OnOpcode)** 로 캡처 가능 → **포크 불필요**(beatoz-go EVM 통합 레이어에 부착).
- **revert 처리**: revert된 프레임 로그는 영수증에 안 남음(사용자 지적). tracer가 `OnExit(reverted)`로 해당 프레임 캡처를 버리면 **남은 캡처 = 영수증 로그**, 순서 일치. (이전에 "포크 필요할 수도"는 과한 우려였음 → 정정)
- **필수 조건**: 디버그 tracer가 아니라 **항상 켜진 결정론적 캡처**(tx_event_root→합의 포함).
- 사용자 코멘트: "OnOpcode 방법 괜찮은 것 같다, 나중에 다시 검토하자." → **재검토 대기.**

## 7. 정규 `LinkerEndpoint.Pay` 제안 (노드 변경 없이 contract-payer 해결) — 유력

> **2026-05-28 업데이트**: `Pay`는 결제 편향이라 use-case independent한 `publish`로 변경. 메소드 시그니처·이벤트 구조·두 층 분리(`LinkerApp._lkPublish` 추가)는 §11 참조. 본 절은 *historical 탐색* 기록으로 유지.

dApp이 X를 직접 emit하지 않고 **`LinkerEndpoint.Pay(payer, payee, amount, …)`** 를 호출. Pay가 이벤트를 emit:
- `contractAddress == LinkerEndpoint`(정규, 레지스트리 조회), `topic.0 == X sig`, `topic.1 == dApp(=Pay의 msg.sender)`, `topic.2 == payer`, `topic.3 == payee`, `data == amount`.
- **핵심 장점**: dApp(=msg.sender)을 **Solidity에서 캡처** → §6의 노드/OnOpcode 변경 **불필요**.
- **`msg.sender` 바인딩은 안전**: Pay가 `payer == msg.sender` 강제하면, 악의적 dApp도 payer=자신만 가능 → 자기 자금만. 위조 불가(EVM이 msg.sender 설정). "열린 정규 컨트랙트" 함정(§4-B)도 닫힘(자유 emit 아님, caller 바인딩).
- **`tx.origin` 분기는 폐기**: victim을 꾀어 `Pay(victim, attacker)`에 도달하는 tx에 서명시키면 tx.origin=victim 통과 → 탈취(tx.origin 피싱).

## 8. dApp이 "사용자 자금"을 쓰게 — allowance/permit (tx.origin 없이)

"payer==msg.sender만 허용하면 dApp이 자기 자금만 쓴다"는 제약 해소책. **ERC-20 `transferFrom`의 크로스체인판**:
- Pay는 payer를 제한하지 않고 **`requestedBy = msg.sender`(dApp, 위조불가)** 만 이벤트에 기록.
- **BPrN 측 인가**: `account[payer]`는 다음일 때만 차감 —
  1. `payer == requestedBy`(자기 자금), 또는
  2. `allowance[payer][requestedBy] >= amount`(사용자가 사전 승인, 차감), 또는
  3. payer의 **EIP-712 서명**이 이 결제(payee,amount,nonce)를 인가(일회성).
- 악의적 dApp이 `payer=victim`으로 호출해도 `allowance[victim][그 dApp]=0` → 거부. 도난 불가.
- **EVM 기본 원리 확인**: 네이티브 ETH는 "자기 잔액만"(하드 룰, transferFrom 없음). 토큰은 기본 자기 자금, 위임은 approve/transferFrom이 보편. tx.origin 인가는 안티패턴. → 우리 설계는 **표준 EVM 관행 그대로**(제약 아님).

## 9. 현재까지의 결론

- 정방향 전달은 **기존 프로토콜로 충분**(전달용 새 프리미티브 불요).
- 화이트리스트 없이 payer 인가 → **allowance/permit 모델**(requestedBy=msg.sender 기록 + BPrN에서 self/allowance/EIP-712 인가). **tx.origin 금지.**
- 정규 이벤트 + 비위조 requestedBy 확보 방법 2가지: **(i) 정규 `LinkerEndpoint.Pay`**(Solidity msg.sender, 노드 변경 없음) — 현시점 유력 / (ii) 노드 msg.sender 캡처(OnOpcode) — PARKED.

## 10. 미해결/다음 세션 할 일 (OPEN)

> 2026-05-28 업데이트: 항목 2/4·일부는 §11/§12에서 부분 해결. 신규 항목 6~9 추가.

1. **OnOpcode 기반 msg.sender 캡처 재검토**(§6, parked) — `publish` 방식으로 충분한지 vs 노드 노출이 필요한 케이스가 있는지. *변경 없음, 여전히 parked.*
2. ~~**정규 `LinkerEndpoint.Pay` 스펙 확정**~~: §11에서 `publish`로 리네임 + Model A 파라미터 확정. **남는 세부**(아래 7): `actionSelector` first-class 여부, `LinkerPublish` 이벤트 topic 배치, return value, 새 BTIP로 둘지 btip-21 확장.
3. **크로스체인 신원/주소 통일**: payer(BPrN 자금 계정)·requestedBy(dApp 주소)가 같은 주소 네임스페이스여야 `allowance[payer][requestedBy]`가 양쪽에서 의미. *부분 해결* — `block.chainid` + `msg.sender` + BTIP-9 derived 20B address 통일 방향(§11.3). `approve` 동기화는 미정.
4. ~~**payer 의미 확정**~~: §12에서 use case별 결정 — dApp-orchestrated는 dApp-configured, user-driven payment는 user-input. **사용자 입력 phishing은 protocol 외부**(§12.6).
5. **Nullifier로 재생 방지 → 이중 차감 방지**: existing BTIP-24/33 패턴(`(eventRoot, dApp)` per-pair) 그대로. 별도 작업 불요.

**신규 OPEN** (§11/§12에서 파생):

6. **LinkerApp base 컨트랙트 스펙** (§11.2 후속): BTIP-26 범위 확장 vs 신규 BTIP, 상속 optional, endpoint 주소 출처(생성자 vs BTIP-37 동적 조회), BPrN 대칭(BTIP-34), payload 인코딩 규약.
7. **`publish` 함수 세부** (§11.3 후속): `actionSelector` first-class vs `actionData`에 packed, `LinkerPublish` 이벤트 topic 배치, return value 형식(`bytes32 messageId` 등).
8. **STC use case 측 미해결** (§12 후속, 사용자가 "지금 논의하지 말고"한 항목): (a) settle 시 STC 행선지(burn / payee 계정 / 응답 데이터 중 어느 것), (b) PaymentBridge ccApp 분리 vs STC 통합 결정, (c) `approve`의 BPrN/BPuN 동기화 모델.
9. **BTIP-26/34 Escrow Lifecycle 섹션에 IntendedHandler 결정 패턴 권장 명문화**: dApp-orchestrated → dApp-configured, user-driven payment → user-input + payee_account 함께 입력 등 가이드.

---

## 11. (2026-05-28) `Pay` → `publish` 리네임 + 파라미터 확정

본 문서가 "결제(payment)" 어휘에 과도하게 묶여 일반화 필요성 제기. §7의 `LinkerEndpoint.Pay`는 결제만이 아니라 NFT mint, 거버넌스 투표, 데이터 업데이트 등 **임의 cross-chain 액션 트리거**의 도구로 정의되어야 함. 이름·인터페이스를 use-case independent로 재정의.

### 11.1 메소드 이름 — `publish` 채택

검토 후보 및 결과:

| 후보 | 평가 | 결과 |
|---|---|---|
| `Pay` | 결제 편향, 너무 좁음 | 기각 |
| `Invoke` / `Request` | 너무 일반적, dApp 메소드와 충돌 가능 | 기각 |
| `submitProof` / `submitLinkerProof` | "proof"가 잘못된 단계에 붙음 — publish 시점엔 proof 미존재(Prover가 나중에 구성). 기존 `onProof` 수신측과 의미 충돌. "submit" + `onProof`의 짝(sender↔receiver) 혼동 | 기각 |
| `linkerEmit` / `linkerSend` | 본질은 표현하지만 컨트랙트명(`LinkerEndpoint`)과 메소드 prefix 중복 ("linker" 두 번) | 기각 |
| `lkEmit` / `lnkEmit` (LayerZero `_lz` 미러) | 컨트랙트 풀워드↔메소드 약어 비대칭. `lnk`는 Chainlink LINK 토큰과 혼동 위험. LayerZero `_lz`가 통하는 이유는 inherited base namespace 용 — 외부 endpoint 메소드엔 부적합 | 기각 |
| **`publish`** (prefix 없음) | "공개 원장에 게시" 본질 표현. 기존 `onProof`/`onResult`와 동일 prefix-없는 컨벤션. 다른 cross-chain 프로토콜과 어휘 겹치지 않아 BEATOZ 특색 | ✅ 채택 |

**채택 근거**: BTIP-21 기존 메소드 모두 prefix 없는 동사(`onProof`/`onResult`/`setNullifierContract`/`setVerifierContract`). 컨트랙트명이 namespace 제공하므로 메소드 prefix 불요 — ERC-20 `IERC20.transfer`/`approve`도 같은 관례. LayerZero V1 external endpoint도 `Endpoint.send`로 prefix 없음.

### 11.2 두 층 분리 — `LinkerEndpoint.publish` + `LinkerApp._lkPublish`

LayerZero V1 `LzApp.sol` / V2 `OApp.sol` 패턴 미러:

| 층위 | 위치 | 이름 | 호출 형태 |
|---|---|---|---|
| 외부 endpoint 메소드 | `LinkerEndpoint` external 함수 | `publish(...)` | `ILinkerEndpoint(linker).publish(...)` |
| 상속 base 헬퍼 | `LinkerApp` 추상 base | `_lkPublish(...)` | `_lkPublish(...)` (dApp 내부 호출) |

**`_lkPublish`의 `lk` 접두사가 여기서는 적절한 이유**: dApp 자체 internal 메소드와 같은 namespace(상속)에 들어가므로 LayerZero `_lzSend`와 동일한 충돌 회피 마커 역할. 외부 호출과 달리 컨트랙트 풀워드↔약어 비대칭 발생 없음. Solidity 관례상 internal 함수는 `_` 접두 — `_lkPublish`로 표기.

LinkerApp base 컨트랙트 추가 결정사항은 §10-6으로 이월:
- BTIP-26 범위 확장 vs 신규 BTIP
- 상속은 optional (interface만 구현해도 동작 — LayerZero V2 OApp도 마찬가지)
- LinkerEndpoint 주소 출처 (생성자 hardcode vs BTIP-37 registry 동적 조회 — 후자 추천: 재배포·진본성에 강함)
- BPrN 대칭 (BTIP-34에 어떻게 미러할지 — Go embedding/composition)
- payload 인코딩 규약

### 11.3 `publish` 파라미터 — Model A (open destination)

두 모델 비교:

| 모델 | 시그니처 | 평가 |
|---|---|---|
| **A. Open destination** | `publish(bytes32 actionSelector, bytes calldata actionData)` | ✅ 채택 |
| B. Explicit destination | `publish(bytes32 dstChainId, address dstContract, bytes32 actionSelector, bytes calldata actionData)` | 기각 |

**Model A 채택 근거**:
- 기존 BPrN→BPuN 흐름과 대칭 — `TransferEventElems`는 destination 필드 없음, `onProof(payload, targetDApp)`에서 Prover가 destination 지정. 같은 패턴을 BPuN→BPrN에도 적용.
- `dstContract`를 이벤트에 박아도 LinkerEndpoint는 on-chain routing 안 함 — 정보적 필드일 뿐. 강제하려면 상대 LinkerEndpoint가 추가 검증 필요(복잡성 증가, 기존 흐름과 비대칭).
- Destination 안전성은 **app-level auth(allowance/permit, IntendedHandler 비교)** 가 담당하는 게 정석. ERC-20 `transferFrom`도 token 컨트랙트가 caller 의도를 검증하지 않고 allowance만 확인 — 같은 정신.
- LayerZero V2가 `dstEid`를 first-class로 두는 이유는 endpoint 내부 on-chain routing/fee 계산 때문. BEATOZ는 그런 layer 없음 — Prover가 off-chain routing.

**자동으로 채워지는 src 정보** (파라미터 불요):
- `srcChainId = block.chainid` → EVM 자동 → Tendermint `CanonicalVote.chain_id`에 포함 → Validator 서명으로 암호학적 검증 → 상대 LinkerEndpoint가 proof에서 추출 → BTIP-34 `HandleLinkerEvent.srcChainId` 파라미터로 전달
- `srcContract = msg.sender` → Solidity 자동 캡처 → 이벤트 indexed topic
- `contractAddress` (event index 0) = LinkerEndpoint 자기 주소 → BTIP-37 LinkerRegistry로 진본 검증

**남는 결정** (§10-7로 이월):
- `actionSelector` first-class 유지 vs `actionData`에 packed (LayerZero V2처럼)
- 발행되는 `LinkerPublish` 이벤트의 topic 배치 (indexed 슬롯 3개 어떻게 쓸지)
- return value (`bytes32 messageId` 등 반환할지)

---

## 12. BPrN-origin mirror — STC use case 분석 (2026-05-28)

본 문서의 BPuN-origin 분석과 대칭으로, BPrN-origin 사례를 STC(stablecoin) use case로 검토. **IntendedHandler 결정 패턴**의 일반 원리 도출.

### 12.1 시나리오

- BPrN에 STC(stablecoin) ccApp 1개
- BPuN에 결제를 받는 dApp 다수 (NFT 마켓, 게임, 구독 서비스 등)
- 흐름: 사용자가 STC로 dAppX에 결제 → 증명이 BPuN으로 전달(누구든 제출 가능) → dAppX가 NFT 발행 등 가치 제공

→ **1:N 관계** (one STC, many dApps).

### 12.2 등록 모델은 1:N에 부적합

§3 결정 1 / btips-2pc-design.md §9 #5a(IntendedHandler binding)의 자연스러운 첫 발상: "STC가 신뢰하는 BPuN dApp을 사전 등록". 그러나:
- 운영 부담 — 새 dApp마다 STC admin 거버넌스 절차
- STC가 비즈니스 의사결정자가 됨 — permissionless 생태계와 정반대
- N이 커질수록 운영 불가능

→ **등록 모델 부적합 확정.** 이게 BPuN-origin(§7 본문 가정)과 본질적으로 다른 점.

### 12.3 IntendedHandler 결정 주체 — use case별 차이

btips-2pc-design.md §9 #5a 원칙("자금 risk 지는 source가 lock 시점에 destination 사전 기록")은 유지. 다만 *누가* destination을 결정하느냐는 use case에 따라 다름:

| Use case 유형 | destination 결정자 | 근거 |
|---|---|---|
| **dApp-orchestrated action** (BPuN dApp이 자기 BPrN partner와 협업) | dApp이 사전 설정 (immutable / owner-set) | 사용자는 destination을 모름·검증 능력 없음. dApp이 자기 partner를 알고 신뢰. |
| **User-driven payment** (사용자가 직접 수취인 선택) | **사용자 입력** | 사용자가 *수취인 선택의 주체*. Web2 결제(가게 계좌번호 입력)와 본질 동일. |

→ STC use case는 후자. **사용자 입력**으로 IntendedHandler 설정이 정합.

**"User-input phishing" 재해석**: 이전 §8/§7 우려는 dApp-orchestrated 시나리오 가정. User-driven payment에선 사용자가 *원래 수취인 직접 선택*이므로 잘못 지정해도 Web2 phishing과 동일 수준 — cross-chain 고유 공격 벡터 아님.

### 12.4 아키텍처 — 2개 비교, Architecture 2 채택

| 항목 | 아키텍처 1 (Fire-and-forget) | 아키텍처 2 (STC 2PC pay) |
|---|---|---|
| STC 측 함수 | transfer만 (현 BTIP-25 그대로) | `PayTo(targetDApp, amount, ...)` + escrow + `HandleLinkerResult` |
| dApp 등록 | 불요 | 불요 |
| 1:N 지원 | ◎ | ◎ |
| dApp 거부 시 보호 | 없음 (사용자 STC 손실) | refund 보장 (2PC) |
| 적합 환경 | 평판·계약 등 off-chain 보호 충분 | permissionless·dApp 미신뢰 |

→ **아키텍처 2 채택** (사용자 자금 보호 우선).

### 12.5 Timeout 우려 검증 — 불필요

이전 분석에서 "사용자가 잘못된 dApp 주소를 입력하면 escrow 영구 LOCKED" 우려 제기. 재검토:
- 누구든(특히 사용자 본인) proof를 self-submit 가능 (permissionless)
- LinkerEndpoint try/catch:
  - 존재하지 않는 주소 → call revert → catch → **REJECTED** 발행
  - 정직한 잘못된 dApp → revert → **REJECTED** 발행
- LinkerResult: handler = 사용자가 지정한 잘못된 주소 == IntendedHandler → 일치 → REJECTED → **refund**

→ Self-submit으로 escrow 자동 환불. **Timeout/cancel 도입 불필요.** btips-2pc-design.md §3 결정 4("타임아웃 없음") 유지.

### 12.6 남는 위험 — Phishing dApp이 ACCEPT

User-input 시나리오에서 protocol-level 해결 불가한 유일한 경로:
1. 사용자가 phishing dApp 주소를 IntendedHandler로 지정 (가짜 사이트에 속아서)
2. Phisher가 proof를 자기 자신에 제출 → ACCEPTED 반환 (자금 탈취 목적)
3. LinkerResult: handler = phisher, ACCEPTED
4. STC: handler == IntendedHandler → 일치 → settle → 자금 phisher 측으로

이 경로는 protocol 검증 모두 통과. 사용자가 *의도적으로* phisher 지정했기 때문. Web2/Web3 표준 결제 phishing과 본질 동일 (`approve(phisher, MAX)`, `transfer(phisher, amount)`):
> 사용자 의도 결정 → 프로토콜이 의도대로 실행 → 의도가 잘못된 거면 사용자 손해

**완화는 protocol 밖 계층** — Wallet UI(주소 자동완성, phishing 경고, dApp 메타데이터 표시), dApp 식별(ENS-style naming, code hash 비교), 사용자 교육, 평판 시스템. **BEATOZ Linker Protocol 스펙엔 추가 안 함.** 단 BTIP-26/34 Escrow Lifecycle 섹션에 IntendedHandler 결정 패턴 권장 명문화 가능 (§10-9).

### 12.7 양방향 통합 원칙 (재확인)

방향(BPuN-origin / BPrN-origin)·use case(orchestration / payment)와 무관하게 동일 원칙:

> **자금/리소스 risk를 지는 source 측 컨트랙트가, 자기가 신뢰하는 destination을 lock 시점에 사전 기록한다. 사용자는 source 측 컨트랙트를 신뢰하는 것으로 충분 — destination 자체를 직접 검증할 책임은 없다.**

btips-2pc-design.md §9 #5a 책임 분담 재확인. 차이는 *destination 결정 주체*뿐 (dApp인지 사용자인지).
