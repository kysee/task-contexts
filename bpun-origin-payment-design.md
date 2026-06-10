---
last_updated: 2026-06-01
status: 설계 결정 (publish 폐기, 미구현)
related: ./btips.md, ./btips-2pc-design.md
---

# BPuN-origin "이벤트 → BPrN 결제" 서비스 설계 연구

> 2026-05-27 세션의 설계 논의·검토 대안·결정·발견한 함정을 자기완결적으로 정리. 인접 주제(2PC)의 확정 사항은 `./btips-2pc-design.md` §9 참조.
>
> **2026-05-28 업데이트**: §11(use-case independent reframing — `Pay` → `publish` 리네임, 두 층 분리, 파라미터 Model A 확정), §12(BPrN-origin STC use case 분석 — IntendedHandler 결정 패턴, timeout 불필요 검증, phishing 분석) 추가.
>
> **2026-06-01 결정**: `LinkerEndpoint.publish` 제안 **폐기**. 새 전제(ccApp이 신뢰 emitter 목록을 관리하지 않음)와 결합하면 publish의 가치 4가지(단일 정규 contractAddress, `requestedBy` 비위조 캡처, 이벤트 스키마 표준화, caller chain 캡처)가 모두 `contractAddress`(evm 이벤트 index 0, EVM 강제 비위조) + ccApp 권한 모델로 대체됨. 정리는 §13 참조. §6 OnOpcode·§7 `Pay`·§11 `publish` 제안은 historical로 보존하되 *폐기됨* 표시. §10 OPEN 항목 다수 해소.

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

## 6. 노드 레벨 msg.sender 캡처 (OnOpcode) — **폐기 (2026-06-01)**

> 본 절은 historical 탐색 기록. 2026-06-01 결정으로 ccApp이 emitter 신뢰 목록을 관리하지 않고 `requestedBy = contractAddress`(evm index 0, EVM 강제 비위조)로 통일하기로 합의 → `msg.sender`를 별도 캡처할 필요가 없어 본 접근은 영구 불요. 아래 분석은 OnOpcode 자체의 기술적 메모로 남김.

- `evmLogsToEvent`는 msg.sender를 모름(`types.Log`에 caller 없음, `Address`=emitter만).
- 실행 중 **LOG opcode 시점**에서만 알 수 있음: `scope.Caller()`=msg.sender, `scope.Address()`=emitter.
- go-ethereum **`tracing.Hooks`(OnEnter/OnExit/OnOpcode)** 로 캡처 가능 → **포크 불필요**(beatoz-go EVM 통합 레이어에 부착).
- **revert 처리**: revert된 프레임 로그는 영수증에 안 남음(사용자 지적). tracer가 `OnExit(reverted)`로 해당 프레임 캡처를 버리면 **남은 캡처 = 영수증 로그**, 순서 일치. (이전에 "포크 필요할 수도"는 과한 우려였음 → 정정)
- **필수 조건**: 디버그 tracer가 아니라 **항상 켜진 결정론적 캡처**(tx_event_root→합의 포함).
- 사용자 코멘트: "OnOpcode 방법 괜찮은 것 같다, 나중에 다시 검토하자." → **재검토 대기.**

## 7. 정규 `LinkerEndpoint.Pay` 제안 (노드 변경 없이 contract-payer 해결) — **폐기 (2026-06-01)**

> **2026-05-28 업데이트**: `Pay`는 결제 편향이라 use-case independent한 `publish`로 변경. 메소드 시그니처·이벤트 구조·두 층 분리(`LinkerApp._lkPublish` 추가)는 §11 참조. 본 절은 *historical 탐색* 기록으로 유지.
>
> **2026-06-01 결정**: §11의 `publish` 후속 제안 전체와 함께 폐기. 폐기 근거는 §13 참조 — `requestedBy = contractAddress`(evm index 0, EVM 강제 비위조)로 통일하면 별도 정규 메소드 없이도 동일 보장이 성립.

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

> **2026-06-01 갱신**: 결론 (iii)·(iv) 추가. 이전 (i)·(ii)는 §13의 결정으로 모두 폐기됨.

- 정방향 전달은 **기존 프로토콜로 충분**(전달용 새 프리미티브 불요).
- 화이트리스트 없이 payer 인가 → **allowance/permit 모델**(BPrN에서 self/allowance/EIP-712 인가). **tx.origin 금지.**
- ~~정규 이벤트 + 비위조 requestedBy 확보 방법 2가지: **(i) 정규 `LinkerEndpoint.Pay`**(Solidity msg.sender, 노드 변경 없음) — 현시점 유력 / (ii) 노드 msg.sender 캡처(OnOpcode) — PARKED.~~ **→ 둘 다 폐기.**
- **(iii) `requestedBy = contractAddress`(evm 이벤트 index 0)**: dApp이 자기 코드에서 emit하면 EVM이 자동으로 박는 비위조 값. publish 같은 별도 메소드 없이 *그 자체로* 위조 불가 비위조 requestedBy 확보. §13 §3.
- **(iv) ccApp의 emitter 무신뢰**: BPrN ccApp은 BPuN dApp 주소를 *허용/거부 식별*에 쓰지 않음. 신뢰 앵커는 *오직* payer 측 인가(`payer == requestedBy` 자기 자금 / `allowance[payer][requestedBy]` / EIP-712 서명). §13 §2.
- 자금 소유자가 dApp이면 `payer == requestedBy` 분기로 즉시 통과(ETH 기본 룰 미러). 자금 소유자가 EOA면 allowance 또는 permit 분기. §13 §4.

## 10. 미해결/다음 세션 할 일 (OPEN)

> 2026-05-28 업데이트: 항목 2/4·일부는 §11/§12에서 부분 해결. 신규 항목 6~9 추가.
>
> **2026-06-01 갱신**: §13의 publish 폐기 결정으로 1·2·6·7이 해소(메소드 없으므로 스펙 작성 불요). 3·5·8·9는 잔존. 9는 BTIP 명문화 사항이 더 명확해져 §13 §6로 통합.

1. ~~**OnOpcode 기반 msg.sender 캡처 재검토**~~ — *해소.* `requestedBy = contractAddress` 통일로 노드 캡처 불요(§13 §3).
2. ~~**정규 `LinkerEndpoint.Pay` 스펙 확정**~~ — *해소.* publish 자체 폐기.
3. **크로스체인 신원/주소 통일**: payer(BPrN 자금 계정)·requestedBy(dApp 주소)가 같은 주소 네임스페이스여야 `allowance[payer][requestedBy]`가 양쪽에서 의미. *부분 해결* — BTIP-9 derived 20B address 통일 방향. `approve` 동기화는 미정. **잔존.**
4. ~~**payer 의미 확정**~~: §12에서 use case별 결정 — dApp-orchestrated는 dApp-configured, user-driven payment는 user-input. **사용자 입력 phishing은 protocol 외부**(§12.6).
5. **Nullifier로 재생 방지 → 이중 차감 방지**: existing BTIP-24/33 패턴(`(eventRoot, dApp)` per-pair) 그대로. 별도 작업 불요. **잔존(상태 변경 없음).**
6. ~~**LinkerApp base 컨트랙트 스펙**~~ — *해소.* publish 폐기로 base 헬퍼 무의미.
7. ~~**`publish` 함수 세부**~~ — *해소.* publish 자체 폐기.
8. **STC use case 측 미해결** (§12 후속): (a) settle 시 STC 행선지(burn / payee 계정 / 응답 데이터 중 어느 것), ~~(b) PaymentBridge ccApp 분리 vs STC 통합 결정~~ — *해소(2026-06-10, §16)*: **PaymentBridge를 별도 ccApp으로 두지 않고 STC 체인코드의 기본 기능으로 통합** (`approve` 포함), (c) `approve`의 BPrN/BPuN 동기화 모델 — `approve`가 STC 체인코드 기본 기능으로 확정된 전제 위에서 검토. **(a)·(c) 잔존.**
9. ~~**BTIP-26/34 Escrow Lifecycle 섹션에 IntendedHandler 결정 패턴 권장 명문화**~~ — §13 §6에 BTIP 명문화 사항으로 통합 (publish 무관하게 적용).

---

## 11. (2026-05-28) `Pay` → `publish` 리네임 + 파라미터 확정 — **폐기 (2026-06-01)**

> **2026-06-01 결정**: 본 절 전체(메소드 이름 `publish` 채택, 두 층 분리 `LinkerEndpoint.publish` + `LinkerApp._lkPublish`, Model A 파라미터) 폐기. 폐기 근거는 §13 참조. 본문은 *historical 탐색* 기록으로 유지.

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

---

## 13. 양방향 이벤트 발행 통합 분석 + Fabric 제약 (2026-05-28) — **부분 폐기 (2026-06-01)**

> **2026-06-01 갱신**: 본 절의 §11 `publish` 전제 부분(통합 채택안 §13.3 `PreparePublish`, BTIP-37 LINKER_CCS role §13.5 NOTE)은 publish 폐기와 함께 무의미. 다만 §13.2의 Fabric Event Model 제약(nested SetEvent 유실)·§13.6의 사용자 피드백(미정의 용어 금지·인터페이스 proliferation 회피)은 *향후 모든 BPrN ccApp 설계에 그대로 적용되는 기술적 사실*이라 보존. 결과적으로 BPrN-origin도 *ccApp이 자기 stub.SetEvent로 emit* — publish 폐기와 자연 일치.

§11 `publish`, 기존 BTIP-25 `TransferEventElems`, BTIP-21 `LinkerResult` — 세 가지 cross-chain 이벤트 발행 경로가 본질적으로 같은 *증명 가능한 이벤트*인데 emit 방법이 분리돼 있는 것을 통합 가능한지 검토.

### 13.1 현재 세 경로의 본질적 차이

| 경로 | 누가 emit | 소스 식별자 | 검증 방식 |
|---|---|---|---|
| `TransferEventElems` (BPrN-origin) | ccApp이 직접 `stub.SetEvent` | `Header.chaincode_id` = ccApp | 수신측 *per-ccApp* 신뢰 설정 |
| `LinkerResult` (BPuN 2PC 결과) | LinkerEndpoint가 emit | `contractAddress` = LinkerEndpoint | BTIP-37 LinkerRegistry로 canonical 검증 |
| `publish` 신규 (BPuN-origin) | LinkerEndpoint가 emit | 동일 (canonical) | 동일 |

→ BPuN 쪽 둘은 이미 canonical 모델. **비대칭은 BPrN 쪽 `TransferEventElems`만** — ccApp이 직접 emit하므로 source가 per-ccApp.

### 13.2 통합 시도와 Fabric 제약

수준 1 (포맷만 통일) vs 수준 2 (canonical source까지 통일)을 검토. 수준 2 시도 시 핵심 제약 발견:

**Fabric Event Model**: 한 transaction의 최종 event는 **사용자가 호출한 최상위(top-level) chaincode의 `stub.SetEvent`** 만 transaction 응답에 실린다. `InvokeChaincode`로 호출된 nested chaincode의 SetEvent는 자기 stub에만 기록되고 transaction event로 propagate 되지 않는다.

→ `ccApp → InvokeChaincode(LinkerEndpointCC.Publish) → SetEvent` 패턴 불가 (event 유실). 해결하려면 사용자가 LinkerEndpointCC를 직접 호출하고 LinkerEndpointCC가 ccApp을 invoke하는 *방향 역전*이 필요한데, 이는 ccApp을 passive로 만들고 사용자 진입점·ccApp 인터페이스를 전면 재설계해야 함.

### 13.3 채택안 — PreparePublish가 payload만 반환, ccApp이 SetEvent

사용자가 제안한 우회 방식:

```go
// LinkerEndpointCC: payload format 정의·생성만 담당, SetEvent는 안 함
func (le *LinkerEndpointCC) PreparePublish(ctx, selector, actionData []byte) ([]byte, error) {
    invokerCC := getInvokerChaincodeID(ctx)
    payload := encodeLinkerPublishPayload(invokerCC, selector, actionData, ...)
    return payload, nil
}

// ccApp: PreparePublish로 canonical payload 받고 자기가 SetEvent (top-level이라 살아남음)
func (c *STC) PayTo(ctx, ...) error {
    c.lockToEscrow(ctx, ...)
    payload, _ := invokeChaincode(ctx, "LinkerEndpointCC", "PreparePublish", PAY_SELECTOR, ...)
    return ctx.GetStub().SetEvent("LinkerPublish", payload)
}
```

**얻는 것 vs 잃는 것**:
- ✅ Fabric event 살아남음 (ccApp이 top-level emitter)
- ✅ Payload format은 LinkerEndpointCC 단일 출처에서 canonicalize
- ✅ ccApp UX 보존 (사용자가 ccApp에 직접 호출)
- ✅ LinkerResult도 같은 LinkerPublish format의 한 selector로 흡수 가능 (BPuN 측)
- △ `chaincode_id`는 여전히 ccApp별 → 수신측은 **per-ccApp 신뢰** 필요. ERC-20 standard practice와 동일 수준 → 수용 가능.

### 13.4 아키텍처 결정사항

- **(a) PreparePublish 형태**: chaincode 메소드 (Go library 아님). 형식 변경 시 한 곳에서 관리, state 기반 필드 추가 여지, audit trail 가능.
- **(b) BTIP-25 (`TransferEventElems`)**: 신규 설계 제약 아님. 유지 방법이 있으면 유지, 없으면 deprecate.
- **(c) ccApp의 "LinkerEndpointCC 경유 의무" 강제 수준**: **spec-level** (수준 1). cryptographic enforcement (수준 2)는 향후 BTIP-19 확장으로 추가 가능.
- **(d) 미사용 ccApp**: deprecate 가능.

### 13.5 BTIP-37 확장 — LINKER_CCS role (minimal)

Spec-level enforcement를 위해 BTIP-37에 cooperative ccApp 등록 메커니즘 추가. 첫 시도(`registerCCApp`/`unregisterCCApp`/`isCCAppRegistered` + events + errors)는 *interface proliferation + 미정의 용어(`LinkerEndpointCC`, `ccApp`) 갑작스레 사용*으로 사용자 reject → rollback.

**최종 minimal 접근** (적용 완료, btip-37.md L48-58 NOTE):
- 새 메소드·이벤트·에러 0건 — 기존 `getContract`/`setContract` 그대로 재사용
- LINKER_CCS는 **derived role**: `keccak256(abi.encodePacked("LinkerCCS", appAddr))` — 각 어플리케이션이 자기 주소 기반 고유 role
- 한 chainId에 N>1 어플리케이션 등록 자연스럽게 지원 (각 derived role이 다름). 상대 체인이 명확한 운영주체의 프라이빗 네트워크라 N은 작음.
- **BPuN의 LinkerRegistry에만 존재** — BPrN에는 대칭 `LINKER_DAPPS` 등록부 없음 (BPuN-origin은 LinkerEndpoint 한 곳에서만 emit되므로 BPrN 수신측은 그 단일 컴포넌트 진본성만 확인하면 충분)
- 어휘 추상화: "LinkerEndpointCC", "ccApp" 같은 미정의 구체 용어 대신 "상대 체인의 비즈니스 어플리케이션"으로 표현

### 13.6 사용자 피드백·교훈 (이번 세션)

- **미정의 용어 갑자기 사용 금지** — BTIP 본문에 `LinkerEndpointCC`, `ccApp` 등 미정의 구체 용어를 갑자기 도입하면 안 됨. 추상 표현("신뢰 가능한 상대 체인 어플리케이션") 또는 사전 정의 후 사용.
- **인터페이스 proliferation 회피** — 기존 primitive로 표현 가능하면 새 메소드 추가하지 말 것. 첫 BTIP-37 시도(3메소드+2이벤트+2에러)는 과도했음.
- **호출 방향 ≠ 사용자 진입점** — Fabric event 모델 분석에서 `STCCC → LinkerEndpointCC.Publish` (nested) 패턴 시 LinkerEndpointCC의 SetEvent가 유실됨을 확인.

---

## 14. OPEN 항목 (2026-05-28 후)

> **2026-06-01 갱신**: (a)·(b)·(c)·(d)·(h)는 publish 폐기로 해소. (e)·(f)·(g)는 잔존.

- ~~(a) **`LinkerEndpointCC.PreparePublish` 정확한 스펙**~~ — *해소.* publish 폐기로 메소드 자체 불요.
- ~~(b) **`LinkerEndpoint.publish`(BPuN) 스펙**~~ — *해소.* publish 자체 폐기.
- ~~(c) **`LinkerApp._lkPublish` 스펙**~~ — *해소.* base 헬퍼 무의미.
- ~~(d) **LinkerResult를 LinkerPublish로 흡수**~~ — *해소.* LinkerPublish 부재.
- (e) **STC use case 측 미해결** — settle 시 STC 행선지(burn/payee 계정/응답 데이터), ~~PaymentBridge ccApp 분리 vs STC 통합~~(*해소 2026-06-10, §16 — STC 통합*), approve의 BPrN/BPuN 동기화. **행선지·approve 동기화 잔존.**
- (f) **BTIP-26/34 Escrow Lifecycle 섹션에 IntendedHandler 결정 패턴 명문화** — §15 §6 BTIP 명문화 사항에 통합. **잔존(작성 대기).**
- (g) **수준 2 cryptographic enforcement (향후)** — BTIP-19 확장으로 audit state proof 추가, RWset/state-proof 기반 LinkerEndpointCC 경유 검증. *publish 무관, 별도 미해결.* **잔존.**
- ~~(h) **BTIP-25 deprecation 계획**~~ — *불요.* LinkerPublish가 자리잡지 않으므로 BTIP-25는 그대로 운용. (단, BPrN-origin 이벤트 발행을 BTIP-25에 한정할지/더 일반화할지는 별도 검토 가능.)

---

## 15. (2026-06-01) `publish` 폐기 결정 + ccApp emitter 무신뢰 모델 확정

> 2026-05-27/28의 publish 탐색을 종결하는 결정. 본 절은 *결론*이며, 위쪽 §§7·11·13·14의 publish 관련 본문은 historical로 보존됨. 향후 작업은 본 절을 기준으로 시작.

### 15.1 결정 요지

- **`LinkerEndpoint.publish`(BPuN)·`LinkerApp._lkPublish`(상속 base)·`LinkerEndpointCC.PreparePublish`(BPrN) 전부 폐기.**
- BPuN-origin 이벤트는 **dApp이 자기 코드에서 직접 emit**. 별도 정규 메소드 없음.
- BPrN-origin도 종전 그대로 ccApp이 `stub.SetEvent`로 직접 emit (BTIP-25 `TransferEventElems` 패턴 유지).
- 양방향 대칭: 양쪽 모두 *사용자 호출 컨트랙트가 직접 emit*, 별도 endpoint 경유 없음.

### 15.2 결정의 결정적 전제

**"ccApp(BPrN 수신측 비즈니스 체인코드)은 신뢰 가능한 emitter dApp 목록을 관리하지 않는다."**

이 전제 하에서 publish가 보장하던 가치 4가지가 모두 dead weight가 된다:

| publish의 가치 | 대체 메커니즘 |
|---|---|
| 단일 정규 contractAddress (`= LinkerEndpoint`) | ccApp이 contractAddress로 emitter 필터링하지 않음 → 가치 없음 |
| `requestedBy = msg.sender` 비위조 캡처 | `contractAddress`(evm 이벤트 index 0) 자체가 EVM 강제 비위조 → 동일 보장 |
| 이벤트 스키마 표준화 | BTIP가 X 이벤트 시그니처만 정의하면 충분 (`topic.0 = keccak256(...)`) |
| caller chain 캡처 | STC use case의 권한은 caller chain이 아니라 *payer의 approve 행위*에서 나옴 |

### 15.3 `requestedBy = contractAddress` 규약 (프로토콜 의무)

BPuN dApp이 emit한 이벤트의 `contractAddress`(evm 이벤트 index 0)는 EVM이 자동으로 `address(this)`로 박는다. *위조 불가*. BPrN ccApp은 이 필드를 `requestedBy`로 해석한다.

- dApp이 attribute에 박은 임의 필드(예: `requestedBy=Alice`)는 ccApp이 *requestedBy로 해석하지 않는다*. 위조 가능.
- 이 규약은 BTIP-26/34에 명문화 (§15.6).

### 15.4 ccApp의 권한 모델 (3분기, ERC-20 미러)

`requestedBy = contractAddress` 통일 후, ccApp이 `payer` 인가를 판단하는 3개 분기:

| 분기 | 조건 | 미러 |
|---|---|---|
| 1. 자기 자금 | `payer == requestedBy` | ETH 기본 룰 (transferFrom 없음) |
| 2. 위임 자금 | `allowance[payer][requestedBy] >= amount` | ERC-20 `transferFrom` |
| 3. 서명 인가 | payer의 EIP-712 서명 (`payee, amount, nonce, deadline` 바인딩) | ERC-20 `permit` |

**셋 중 하나도 통과 못 하면 거부.** 세 분기 모두 `requestedBy`의 진실성(`contractAddress`로 EVM 강제) 위에서 작동.

### 15.5 자금 소유자별 동작

**EOA payer (Alice)**:
- BPuN dApp의 `payOrder()` 호출 → dApp이 `emit X(payer=Alice, payee, amount)`.
- `contractAddress = dApp`. `payer != requestedBy` → 분기 1 불가.
- Alice가 사전에 BPrN에서 `stc.approve(dApp_address, amount)` 호출했어야 분기 2 통과. 또는 분기 3 서명.
- 악의적 Mallory가 `emit X(payer=Alice, ...)` 임의 emit 시: `contractAddress=Mallory`, `allowance[Alice][Mallory]=0` → 거부. *도난 불가.*

**dApp payer (Bob_DApp이 자기 BPrN 잔액 사용)**:
- Bob_DApp이 자기 코드에서 `emit X(payer=Bob_DApp, payee, amount)`.
- `contractAddress = Bob_DApp = payer` → 분기 1 즉시 통과. allowance·서명 불요.
- 악의적 Mallory가 `emit X(payer=Bob_DApp, ...)` emit 시: `contractAddress=Mallory ≠ Bob_DApp=payer` → 분기 1 불가, `allowance[Bob_DApp][Mallory]=0` → 거부.

**주소 통일 전제**: payer·requestedBy가 양 체인에서 동일 주소 네임스페이스를 가지려면 BTIP-9 derived 20B address로 통일돼야 함(§10-3 잔존 항목).

### 15.6 BTIP 명문화 사항 (다음 작업)

> **2026-06-01 정정**: 초안의 항목 1·2(BTIP-26/34에 권한 3분기·`requestedBy` 매핑·"신뢰 emitter 목록 관리 금지" 명문화)는 잘못된 방향이었음 — BTIP-26/34는 *일반 LinkerApp 인터페이스 스펙*이고, `payer`/`requestedBy`/`approve`/`allowance`는 *결제 이벤트의 도메인 용어*이므로 이벤트 정의 BTIP 없이 인터페이스 스펙에 박으면 다른 use case(NFT mint·거버넌스 투표 등)의 LinkerApp에 결제 도메인 가정을 강제하게 됨. "신뢰 emitter 목록 관리 금지"는 원래 없던 것을 부정형으로 박는 것이라 무의미. 권한 3분기 모델은 §15.4 *use case 가이드*에 한정하고, 별도 BTIP로 명문화하려면 **결제 이벤트 정의 BTIP가 선행**되어야 함(현재 부재).

본 결정으로 BTIP 문서에 *실제 반영해야* 할 항목:

1. **BTIP-37**: §13.5의 LINKER_CCS role NOTE 제거. publish 가정에 묶여 있어 본 결정으로 무의미. **(완료 2026-06-01)**
2. **BTIP-21·BTIP-29**: publish 관련 메소드/이벤트 추가 *없음*. 기존 onProof/OnProof로 충분. **(작업 자체 불요)**
3. **BTIP-25**: 그대로 유지. BPrN-origin 이벤트는 `TransferLogElems`(구 `TransferEventElems`)로 계속 발행.
4. **BTIP-26·BTIP-34**: 변경 *없음*. 일반 LinkerApp 인터페이스 스펙이므로 결제 도메인 권한 모델을 박지 않는다.

**§15.4 권한 3분기(자기 자금 / allowance / permit)와 `requestedBy = contractAddress` 매핑은**:
- 본 문서(§15.4)에 *애플리케이션 패턴 노트*로 보관됨.
- BTIP로 명문화하려면 **결제 이벤트 정의 BTIP를 먼저** 만들어야 함. 그 BTIP에서:
  - 결제 이벤트 필드(`payer`/`payee`/`amount` 등)를 정의.
  - 어떤 필드가 `requestedBy`로 매핑되는지(BPuN-origin은 `contractAddress`, BPrN-origin은 `Header.chaincode_id`) 정의.
  - 그 이벤트의 *수신 시* 권한 3분기를 명시.
- 결제 이벤트 정의 BTIP가 부재한 현 상태에서는 BTIP 본문에 박을 자리가 없으므로, 권한 모델은 본 use case 가이드에만 보존.

### 15.7 양방향 대칭 정리

| 방향 | emitter | source 식별 | ccApp/dApp 권한 |
|---|---|---|---|
| BPrN-origin | ccApp `stub.SetEvent` | `Header.chaincode_id` | BPuN dApp 측 권한 모델 (대칭 분기) |
| BPuN-origin | dApp `emit X(...)` | `contractAddress`(evm index 0) | BPrN ccApp 측 권한 모델 (15.4 3분기) |

publish 폐기로 양방향이 *완전 대칭* — endpoint 경유 없이 origin 측 컨트랙트가 직접 emit, 수신 측 컨트랙트가 권한 결정.

### 15.8 잃는 것 / 잔존 위험

- BPrN/BPuN 주소 네임스페이스 통일이 약하면 `allowance[payer][requestedBy]` 매핑이 깨짐 → §10-3 항목으로 잔존.
- dApp이 payer 필드를 *정직하게 박는다*는 신뢰는 dApp 코드 정확성 수준에서만 보장 (위조해도 권한 3분기로 차단되므로 도난은 없음 — *정직성 위반의 결과는 dApp 자기 자신의 트랜잭션 거부*).
- 1:1 high-trust 시나리오에서 "LinkerEndpoint emit만 받음" 정책으로 noise 차단하던 옵션 상실. 다만 ccApp이 신뢰 emitter 목록을 관리 안 하는 전제와 이미 양립 불가능했으므로 잃을 게 없음.

### 15.9 다음 작업 우선순위 (이번 결정 후)

> **2026-06-01 정정**: §15.6 정정으로 BTIP-26/34 명문화 작업은 *불요*로 결정. BTIP-37 LINKER_CCS 제거는 완료.

1. **결제 이벤트 정의 BTIP** (선결 — 권한 모델을 BTIP 차원에서 명문화하려는 경우): 결제 이벤트의 필드(`payer`/`payee`/`amount`/correlationId 등)를 정의하는 새 BTIP가 있어야 §15.4의 권한 3분기·`requestedBy` 매핑을 그 BTIP 안에 박을 수 있음. 현재는 use case 가이드(§15.4)로 보관.
2. **STC use case 미해결 항목** (§10-8 잔존). settle 행선지, approve 동기화 (PaymentBridge 분리 여부는 2026-06-10 해소 — §16). 위 항목 1을 진행한다면 함께 정의.
3. **2PC 코드 구현** — on-bpun/on-bprn 양쪽에 LinkerResult/onResult/handleLinkerResult/try-catch/nonReentrant 추가, BPuN→BPrN 이벤트 Prover(u2r) 구현. 권한 모델 BTIP 명문화와 독립적으로 진행 가능.

---

## 16. (2026-06-10) PaymentBridge 폐기 — STC 체인코드 기본 기능으로 통합

> 사용자 결정. §12.4 Architecture 2 채택 이후 보류돼 있던 "PaymentBridge ccApp 분리 vs STC 통합"(§10-8(b), §14(e))을 종결.

- **결정**: **PaymentBridge를 별도 ccApp으로 두지 않는다.** 크로스체인 결제 처리(보류/잠금 상태 관리, `HandleLinkerEvent`/`HandleLinkerResult` 구현, 권한 3분기 평가)는 **STC 체인코드의 기본 기능**으로 구현한다.
- **`approve`의 위치**: STC 체인코드가 `approve`(allowance 관리)를 기본 기능으로 보유한다. §15.4 권한 3분기의 분기 2(`allowance[payer][requestedBy]`)는 STC 체인코드 자체 상태로 평가된다 — 별도 브리지 컨트랙트에 allowance를 위임하지 않음.
- **함의**:
  - STC 체인코드가 [BTIP34](../docs/BTIPS/btip-34.md) 인터페이스(LinkerApp 콜백)를 직접 구현하는 ccApp이 된다. §12.4 Architecture 2("STCCC가 직접 escrow")와 정합.
  - 잔존 항목 (a) settle 행선지, (c) approve 동기화 모델은 *STC 체인코드 내부 설계* 문제로 좁혀짐 — 별도 컴포넌트 경계 설계는 불요.
- **§10-8(b)·§14(e) 해당 부분 해소.**
