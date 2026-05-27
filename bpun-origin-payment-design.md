---
last_updated: 2026-05-27
status: 설계 탐색 중 (미구현)
related: ./btips.md, ./btips-2pc-design.md
---

# BPuN-origin "이벤트 → BPrN 결제" 서비스 설계 연구

> 2026-05-27 세션의 설계 논의·검토 대안·결정·발견한 함정을 자기완결적으로 정리. **아직 미구현(설계 탐색 단계)** 이며 다음 세션에서 이어가기 위한 노트. 인접 주제(2PC)의 확정 사항은 `./btips-2pc-design.md` §9 참조.

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

1. **OnOpcode 기반 msg.sender 캡처 재검토**(§6, parked) — (i) 정규 Pay 방식으로 충분한지 vs 노드 노출이 필요한 케이스가 있는지.
2. **정규 `LinkerEndpoint.Pay` 스펙 확정**: 시그니처, 이벤트 topic 배치, BPrN 인가 로직(`payer==requestedBy || allowance || EIP-712`). 새 BTIP로 둘지 btip-21/26/34 확장으로 둘지.
3. **크로스체인 신원/주소 통일**: payer(BPrN 자금 계정)·requestedBy(dApp 주소)가 같은 주소 네임스페이스(ethereum-style/BTIP-9)여야 `allowance[payer][requestedBy]`가 양쪽에서 의미. `approve`는 자금이 있는 BPrN에서(크로스체인 승인은 별도 확장?).
4. **payer 의미 확정**: "X의 트리거러" 강제(→ msg.sender 노출 필요) vs "사전 인가한 자금 소유자"(→ allowance/조건부 에스크로). 현재 무게추는 **allowance 모델(=자금 소유자 인가)**.
5. Nullifier로 동일 이벤트 재생 방지 → allowance 이중 차감 방지(설계에 반영 필요).
