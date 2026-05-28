---
last_updated: 2026-05-28
related: ./btips.md
---

# BTIP 결과-주도 2PC 설계 노트 (Cross-Chain Atomic Payment)

> 2026-05-21 세션의 전체 논의·검토 대안·결정·**결정의 의도**·발견한 함정을 자기완결적으로 정리한 문서.
> 구현 결과(파일별 변경)는 `btips.md`의 2026-05-21 세션 로그 참조. 이 문서는 "왜 그렇게 했는가"를 담는다.
>
> §1~7은 결과-주도 2PC 설계. §A는 별개 주제(BTIP-39 Validator Set Update Proof)로 동일 패턴(요약+의도)으로 함께 보관됨.

---

## 1. 풀려는 문제

기존 BTIP 문서는 **BPrN → BPuN 단방향 증명**(결제 발생 사실을 BPuN에 증명)만 다뤘다. 그러나 궁극적으로는 **BPuN dApp이 그 증명을 어떻게 처리했는지(accept/reject)를 BPrN으로 되돌려 증명**하고, 그 결과에 따라 원발신 결제를 완결해야 한다.

**대표 시나리오**: BPrN 스테이블코인 체인코드에서 결제 발생 → BPuN으로 증명 전달 → NFT 컨트랙트(dApp)가 구매자에게 NFT 발행. 그런데 결제 증명은 모두 통과했더라도 dApp이 특정 조건에서 발행을 **거부(reject)** 하면, 앞서 발생한 BPrN 결제는 **취소(환불)** 되어야 한다.

본질: 서로 독립적 합의를 갖는 두 원장에 걸친 분산 트랜잭션의 **원자성(atomicity)** 문제.

---

## 2. 검토한 모델과 선택

### (기각) Saga / 보상 트랜잭션
- BPrN 결제를 **즉시 확정**(판매자 지급) → dApp reject 시 역방향 증명으로 BPrN이 **환불(보상 tx)** 실행.
- **기각 이유**: 결제가 이미 최종 확정되면 수취인이 자금을 옮겨버려 **환불(clawback)이 불가능**할 수 있음. 돈에 대해 위험.

### (채택) 에스크로 + 2PC
- BPrN 결제를 즉시 확정하지 않고 **에스크로(LOCKED)** 로 잠금. ACCEPTED 결과가 와야 판매자 지급, REJECTED면 구매자 환불.
- **의도**: "이미 확정된 결제를 취소"가 아니라 "아직 확정 안 된 잠금을 해제"가 되어 깔끔하고 안전. 자금 손실·이중지급 없음.
- **재사용**: 양방향 증명 시스템이 이미 있으므로(정방향 btip-16~34, 역방향 btip-27~33), **새 증명 원시 타입 불필요**. 정방향 증명 = "요청", LinkerResult 증명 = "응답". 두 증명을 묶는 상위 레이어만 추가.

---

## 3. 핵심 설계 결정과 의도

### 결정 1 — LinkerRegistry 추가 (신규 btip-37/38)
- **무엇**: 공식 컨트랙트 세트(LinkerEndpoint/Verifier/Policy/Nullifier)의 진본성을 역할(role)별로 관리하는 온체인 단일 출처.
- **의도**: 누군가 동일 바이트코드를 재배포(클론)해도, *신뢰하는 쪽(dApp, 에스크로 체인코드)이 클론을 진본으로 오인*하는 것을 막는다.
- **핵심 통찰**: 클론 존재 자체는 무해. 신뢰의 암호학적 뿌리는 **컨트랙트 주소가 아니라 LinkerPolicy의 Root CA**이며 클론은 이를 재현 못 함. 레지스트리는 "진본 식별 + 바인딩"의 편의 계층.
- **단순 주소 공개보다 강한 이유**: ① 역할명으로 조회(하드코딩 회피) ② 코드 해시 공개로 `EXTCODEHASH` 독립 검증 ③ CREATE2 결정적 배포로 레지스트리 주소 자체가 재현·검증 가능. ④ 재지정은 multisig+timelock(키 탈취 시 반응 시간 확보).

### 결정 2 — dApp revert-only (bool 리턴 없음)
- **무엇**: dApp `handleLinkerEvent`는 리턴값 없음. 정상 반환=ACCEPTED, **revert=REJECTED**.
- **의도**: revert는 EVM이 dApp 상태를 **원자적으로 롤백**하므로 "reject ⟺ 부작용 없음"을 *언어 차원에서 강제*. dApp 자기 규율에 의존하지 않음.
- **기각한 대안(bool 리턴)의 함정**: `false`를 리턴하면 그 전 상태 변경은 커밋됨 → "NFT 발행 후 false 리턴"하면 NFT는 발행됐는데 환불되는 **이중 결과**. revert-only는 이 함정을 원천 봉쇄.
- **남는 dApp 의무**(스펙에 명시): 정상 반환 ⟺ 이행 완료(아니면 revert). `true`로 거짓 성공 보고하는 건 평범한 컨트랙트 정확성 신뢰 등급.

### 결정 3 — "정확히 하나의 결과" 보장 (LinkerEndpoint try/catch)
- **무엇**: LinkerEndpoint가 dApp 호출을 `try/catch`로 감싸, 검증 통과해 소비된 모든 증명은 ACCEPTED 또는 REJECTED 결과를 **정확히 하나** 산출.
- **의도/배경**: 사용자가 "Hard revert는 잘못된 증명이니 재제출하면 됨, 배제 가능"이라 했으나 — **전제가 부정확**. Hard revert는 ①증명 무효(재제출로 해결) ②**유효 증명 + dApp revert**(재제출해도 또 revert, 해결 안 됨)의 두 경우가 있음. ②를 위해 LinkerEndpoint가 dApp revert를 캐치해 결정적 REJECTED로 확정해야 함.
- **반드시 LinkerEndpoint(신뢰 컴포넌트)가 강제** — dApp(비신뢰)의 약속에 의존하면 깨짐.
- `markProcessed`(Nullifier)는 try 이전 → dApp revert와 무관하게 커밋(거부도 소비, 재제출 차단).
- **부수 주의**: 일시적 OOG가 영구 REJECTED로 둔갑하지 않도록 `MIN_CALLBACK_GAS` 하한 + nonReentrant 가드.

### 결정 4 — 타임아웃 없음 (결과-주도 완결)
- **무엇**: 에스크로에 시간 제한 없음. ACCEPTED/REJECTED 결과 증명이 도착해야만 settle/refund.
- **의도**: 타임아웃을 두면 **타임락 순서 레이스**가 생김(D_decide < D_refund 보장 실패 시 "NFT는 발행됐는데 환불"되는 이중 결과). 타임아웃 제거로 이 레이스 클래스를 통째로 제거.
- **안전한 이유**: ① 결정 3으로 소비된 증명당 결과 1개 보장 ② 결과는 BPuN 원장에 영구 기록 → 누구나(구매자 본인 포함) 언제든 결과 증명 생성·제출 가능 → BPuN 살아있는 한 동결 없음.
- **유일한 잔여 리스크**: BPuN이 결과 산출 전 영구 정지/포크 이탈 시 동결 → 결정 7(운영 정책)로 통제.
- **추가 이점**: finalize 제출이 자연 인센티브로 정해짐(아래) + "한 번 잠그면 BPuN 결과만이 해제"라는 깔끔한 불변식(대기 중 일방 취소 불가 → front-run 방지).

### 결정 5 — correlationId = 정방향 `tx_event_root`
- **무엇**: 에스크로 키 = LinkerResult.correlationId = 정방향 이벤트의 `tx_event_root`.
- **의도/검증**(현재 btip-16/24 기준 확인 완료):
  - **유일성** ✅: `tx_id`가 BTIP16 머클트리 gidx:2에 커밋됨 → 동일 내용 결제도 tx_id로 구별. btip-24도 "tx_event_root만으로 BPrN tx를 전 세계 고유 식별" 명시.
  - **lock 시점 계산 가능** ✅: tx_event_root는 `(channel_id, chaincode_id, tx_id, selector, elems)`만으로 결정 → 발신 체인코드가 실행 시점에 직접 계산 가능. (반면 `(blockNumber, txIndex)`는 ordering 시점에 정해져 실행 중 모름 → 부적합)
  - **Nullifier와 통일** ✅: btip-24 Nullifier가 이미 tx_event_root 기준. correlationId=Nullifier 기준값 → 식별자 하나로 통일 + content-bound.
- **`tx_id`보다 우월한 이유**: tx_id는 단순 식별자(통일·바인딩 없음). tx_event_root는 위 3가지를 모두 만족 + 내용에 암호학적 바인딩.

### 결정 6 — onProof vs OnResult 분리
- **무엇**: 정방향 증명은 BPuN `btip-21.onProof`가 소비(LinkerResult emit). LinkerResult 증명은 BPrN `btip-29.OnResult`가 소비(→ btip-34.HandleLinkerResult).
- **방향 정리**: 두 증명은 **서로 다른 체인**에서 소비됨. 한 onProof가 둘 다 처리하지 않음.
- **OnResult를 분리하는 의도(보안)**: 일반 OnProof는 임의 BPuN 이벤트를 임의 체인코드에 전달. 그러나 LinkerResult는 **반드시 공식 BPuN LinkerEndpoint가 emit**한 것이어야 함(아니면 악성 컨트랙트가 가짜 LinkerResult로 남의 에스크로를 settle/refund). OnResult는 "소스 컨트랙트(index 0) == 공식 엔드포인트(btip-38 레지스트리 조회)" 추가 검증을 강제 → 별도 메소드가 정당.
- 대상은 제출자가 지정하지 않고 검증된 LinkerResult의 `originChaincodeId`에서 추출 → 결과는 발신 체인코드로만 전달.

### 결정 7 — 제출 주체 + 운영 정책
- **제출 인센티브**: ACCEPTED는 지급받을 **판매자**가, REJECTED는 환불받을 **구매자**가 자연스럽게 제출. permissionless.
- **relayer(선택)**: BPrN 운영사가 결과 증명 생성·전달을 대행 가능. **신뢰 앵커 아님** — 위조·검열 불가(증명 자체 검증 + self-submit 항상 개방). 중지되면 당사자가 직접 제출.
- **업그레이드 운영 정책(잔여 리스크 통제)**: BPuN 하드포크/컨트랙트 교체 시 → ① BPrN 신규 결제 일시중단 ② relayer로 in-flight 에스크로 force-drain ③ BPuN 업데이트. 하드포크면 drain은 fork 이전 체인에서 완료.

### 결정 8 — 기타
- **tx당 PaymentLocked 1개**: correlationId 모호성 제거. 배치(1 tx → N 액션)는 추후 eventIndex로 확장.
- **always emit (opt-in 마커 없음)**: 모든 소비 증명에서 LinkerResult emit. *함의*: 기존 fire-and-forget 이벤트도 dApp revert 시 롤백이 아니라 소비됨(REJECTED). 결과 안 쓰면 LinkerResult가 반송 안 될 뿐 무해.
- **문서 구조**: LinkerResult 정의는 emit하는 btip-21에. 에스크로 상태머신은 체인코드 로직이므로 btip-34 "Reference: Escrow Lifecycle" 섹션에(인터페이스 ≠ 애플리케이션 구분). 프로토콜 불변식(exactly-one-result, 라우팅)은 btip-21/29에.

---

## 4. 발견·해소한 함정 (재발 방지용)

| 함정 | 해소 |
|------|------|
| bool `false` 리턴 후 부작용 커밋 → 이중 결과 | revert-only (결정 2) |
| `true` 리턴했으나 미이행 | dApp 의무로 명시(평범한 정확성 신뢰) |
| 유효 증명 + dApp revert를 "잘못된 증명"으로 오인 | try/catch로 결정적 REJECTED (결정 3) |
| 일시적 OOG → 영구 REJECTED 둔갑 | MIN_CALLBACK_GAS 하한 |
| dApp 재진입 | nonReentrant 가드 |
| 타임아웃 시 타임락 순서 레이스(이중 결과) | 타임아웃 제거 (결정 4) |
| 클론 컨트랙트를 진본으로 오인 | LinkerRegistry + msg.sender/caller 검증 (결정 1, 6) |
| 악성 컨트랙트의 가짜 LinkerResult | OnResult의 소스=공식 엔드포인트 검증 (결정 6) |
| relayer vs self-submit 중복 제출 | btip-33 Nullifier dedupe |

---

## 5. 구현 결과 (파일)

- **신규**: `btip-37`(LinkerRegistry BPuN/Solidity), `btip-38`(LinkerRegistry BPrN/Go)
- **개정**: `btip-21`(LinkerResult event + try/catch + always-emit + Result Callback 섹션), `btip-26`(revert-only 의무 + msg.sender 검증), `btip-29`(OnResult), `btip-34`(HandleLinkerResult + Escrow Lifecycle Reference), `btip-33`(dedupe NOTE), `README`
- 상세 diff·라인별 변경은 `btips.md` 2026-05-21 세션 로그 참조.

---

## 6. BPuN-origin 양방향 대칭 확장 (2026-05-22)

BPrN-origin의 거울상(결제·요청이 BPuN에서 발생 → BPrN 앱 체인코드가 처리 → 결과를 BPuN으로 반송)을 구현. 두 엔드포인트가 각각 `onProof`(결과 발행) + `onResult`(결과 수신)를 갖춰 완전 대칭.

### 핵심 결정과 의도

**D-A. correlationId 출처의 방향별 비대칭** (가장 많이 논의)
- BPrN-origin: `tx_event_root`(**내재값** — EP가 증명에서 공짜로 가짐, 발신 체인코드가 BTIP16으로 계산 가능). 유지.
- BPuN-origin: **발신 컨트랙트가 정한 명시 id**(nonce). 이유 — BPuN(EVM) 발신 컨트랙트가 event_attrs_root를 lock 시점에 계산하려면 beatoz-go `evmLogsToEvent` 인코딩(소문자 주소/대문자 topic/10진수 blockNumber/data 생략/순서)에 **강결합**되어 노드 포맷 변경 시 모든 컨트랙트가 깨짐. 그 결합을 피하려 명시 id 채택.
- **검증 완료**: `evmLogsToEvent`([ctrler.go:339](~/go/src/github.com/beatoz/beatoz-go/ctrlers/vm/evm/ctrler.go)) 확인 — contractAddress=`address(this)`, topics/data=컨트랙트 생성, blockNumber=`block.number`(=l.BlockNumber), removed=항상 `"false"`(BFT 즉시완결+emit시점). 즉 계산은 *가능*하나 결합 위험 때문에 명시 id 선택.
- correlationId(매칭)와 Nullifier(재생방지)는 **분리** — Nullifier는 여전히 tx_event_root/event_attrs_root 기준.

**D-B. 거부 표현의 방향별 비대칭** (EVM vs HLF 본질 차이)
- BPuN dApp(btip-26): 거부=`revert` → EVM이 자동 롤백 → "거부=무부작용" 강제.
- BPrN 앱 체인코드(btip-34): **Fabric은 자동 롤백 없음.** InvokeChaincode가 에러 반환해도 그 전 PutState는 커밋됨. 게다가 contractapi는 `err != nil`이면 **리턴값(correlationId)을 버림** → 거부를 err로 표현하면 correlationId 유실. 따라서 거부=`accepted=false` **정상 리턴**(tx 커밋). → **CAUTION: `accepted=false` 반환 전 어떤 PutState도 금지** (안 그러면 "거부했는데 상태 변경은 반영" 불일치). `error`는 correlationId조차 못 주는 하드 실패 전용(tx 전체 실패→재시도).

**D-C. HandleLinkerEvent가 index를 반환** (btip-34)
- `HandleLinkerEvent(...) (LinkerResultRef, error)`, `LinkerResultRef{ CorrelationIndex int64; Accepted bool }`.
- 앱 체인코드는 자기 스킴을 알므로 correlationId가 위치한 **인덱스(gidx)** 를 반환. EP는 **검증·확보한 values에서 그 인덱스 값**을 correlationId로 사용 → 앱 체인코드가 값을 위조 못 함(다른 escrow로 라우팅 불가, 기껏 self-griefing).
- `CorrelationIndex < 0` → fire-and-forget(LinkerResultElems 미발행). 이게 #2의 opt-in 신호.

**D-D. fire-and-forget 처리의 방향별 비대칭**
- #1(btip-21): correlationId가 내재(tx_event_root)+거부가 revert(신호 못 실음) → **always-emit**(fire-and-forget도 안 쓰이는 LinkerResult 발행, 무해).
- #2(btip-29): correlationId가 명시(비내재)+거부가 리턴값 → **opt-in**(CorrelationIndex<0이면 ProofVerifiedEventElems, ≥0이면 LinkerResultElems).

**D-E. 이벤트 정의 네이밍**: BPuN쪽은 Solidity 이벤트 `LinkerResult`, BPrN쪽은 EventLog elems 정의라 `LinkerResultElems`(TransferEventElems/ProofVerifiedEventElems 관례).

**D-F. `chaincodeID` → `appChaincodeID` 리네임** (btip-29/33): 이벤트를 실제 처리하는 앱 체인코드를 linker 체인코드 세트와 구분. (verifier/nullifier setter의 chaincodeID는 endpointID 등으로 별도 처리)

### 구현 결과 (파일, 2026-05-22)
- **btip-29**: OnProof가 HandleLinkerEvent 반환(LinkerResultRef) 수신 → `LinkerResultElems`(2PC) 또는 `ProofVerifiedEventElems`(fire-and-forget) 발행. `LinkerResultElems` EventLog 정의. `appChaincodeID` 리네임.
- **btip-21**: `onResult` 추가 — BPrN `LinkerResultElems` 증명 소비, 소스=공식 BPrN 엔드포인트(btip-37 getRemoteEndpoint) 검증, btip-24 Nullifier dedup, `OriginContract`로 `handleLinkerResult` 라우팅.
- **btip-26**: `handleLinkerResult(correlationId, accepted)` + BPuN Escrow Lifecycle Reference(명시 correlationId, 컨트랙트당 유일성 의무).
- **btip-34**: `HandleLinkerEvent` 반환 `(LinkerResultRef, error)` + `Accepted=false` 거부 시 PutState 금지 CAUTION + 호출자 검증.
- **btip-33**: `appChaincodeID` 리네임(SetLinkerEndpointID param은 `endpointID`).
- **btip-24**: `LinkerResultElems` 증명 dedup NOTE.

---

## 7. 보류·후속 작업 (다음 세션에서)

1. **멀티 dApp/앱 체인코드 에스크로**: 한 요청이 여러 액션을 트리거하면 correlationId 매칭을 `(correlationId, dApp)`로 확장(현재 1:1).
2. **구현 디테일 명문화**: BPrN-origin에서 발신 체인코드가 계산한 tx_event_root가 네트워크 정규값과 일치함을 스펙에 못박기. (BPuN-origin은 명시 id라 무관)
3. ~~**커밋**~~: 완료(2026-05-22 세션 변경 커밋 + 2026-05-26 후속 커밋).

---

## 8. LinkerRegistry 다중체인 통합 (2026-05-26)

§3 결정 1(LinkerRegistry 추가)과 결정 6(OnResult 출처 검증) 이후, 인터페이스 표면이 분리되어 있던 부분을 단일 키 모델로 재정리. 결정 자체(레지스트리의 필요성·의도)는 변경 없고, **표현 방식만 통합**.

### 변경 전 (btip-37/38 분리 시점)

- **btip-37 (BPuN/Solidity)**: `getContract(role)`(자기 체인 컨트랙트) + `getRemoteEndpoint(chainId)`(상대 BPrN 엔드포인트 문자열) — 두 갈래로 분리. 추가로 `getCodeHash(role)`/`isAuthentic(role, addr)`로 EXTCODEHASH 비교를 통한 독립 검증 지원.
- **btip-38 (BPrN/Go)**: btip-37의 거울상. `GetContract(role)`/`GetRemoteEndpoint(chainId)`/`IsAuthentic(...)`.
- **반환 타입 비대칭**: BPuN 측은 `address`, BPrN 측은 `string`(체인코드 ID 문자열).

### 변경 후

```
getContract(bytes32 chainId, bytes32 role) → address
setContract(bytes32 chainId, bytes32 role, address contractAddr)
```

자기 체인 조회는 `bytes32(block.chainid)`, 원격 BPrN은 `sha256(channelId)` (BTIP-9). 양쪽 모두 20B `address`로 통일.

### 핵심 결정과 의도

**D8-A. (chainId, role) 단일 키로 통합**
- **변경 이유**: 별도 메소드 `getRemoteEndpoint`는 본질적으로 같은 매핑(role → 공식 주소)을 다른 네임스페이스로 분리 보관하는 것일 뿐. `chainId`를 키 차원에 두면 의미 동일하면서 표면 축소.
- **사용자 통찰**: "`LINKER_ENDPOINT_REMOTE`라는 role을 두면 되지 않냐"는 단순화 제안에서 출발. 검토 결과 `(chainId, role)` 합성 키가 멀티-BPrN 지원도 자연스럽게 흡수.

**D8-B. BTIP-9로 BPrN 주소 정규화 → 타입 통일 가능**
- BPrN의 chaincode address = `sha256(channelId + chaincodeId)`의 20B 추출(BTIP-9 정의). 이로써 BPuN 측에서도 BPrN endpoint를 EVM `address` 타입으로 받을 수 있게 됨.
- **검토 단계의 함정**: 초기에 "BPrN endpoint는 string(체인코드 ID)이라 통합 불가"로 판단 → 사용자가 BTIP-9 계획 알림으로 즉시 무력화. **교훈**: 인접 BTIP의 도입 예정 규약을 모르면 잘못된 비대칭을 본질로 오인할 수 있음. 외부 의존 BTIP의 현황을 먼저 확인.

**D8-C. `block.chainid` 활용 — 별도 self 센티넬 불필요**
- EIP-1344의 `CHAINID` opcode(Solidity 0.8+ `block.chainid`)로 자기 체인 식별. dApp 호출 자연: `registry.getContract(bytes32(block.chainid), LINKER_ENDPOINT)`.
- BPuN(uint256) vs BPrN(sha256 32B) 네임스페이스 충돌 — `bytes32(block.chainid)`는 앞 12B가 0인데, sha256 결과의 앞 12B가 0일 확률 2⁻⁹⁶ → 무시 가능.

**D8-D. codeHash/isAuthentic 제거 (옵션 A)**
- EXTCODEHASH 비교를 통한 독립 진본 검증은 **로컬 BPuN에서만 의미**. 다른 BPuN 체인의 EXTCODEHASH는 자기 체인에서 조회 불가, BPrN은 EVM EXTCODEHASH 자체 없음(Fabric 패키지 ID에 해시 포함).
- 통합 인터페이스에 두면 원격 entry는 항상 0이 되어 비대칭 dead weight. dApp이 실제로 EXTCODEHASH 비교를 한다는 근거도 약함.
- **결정**: 통합 게터만 남기고 제거. 필요하면 별도 메소드로 재도입(YAGNI).

**D8-E. btip-38 철회 — 별도 BTIP 번호 분리 정당성 없음**
- 다른 대칭 쌍(btip-22/32 PKI vs PoS, btip-23/31 X.509+P256 vs secp256k1+머클)은 **저장 데이터·검증 알고리즘이 본질적으로 비대칭** → 분리 정당.
- LinkerRegistry는 통합 후 **동일 데이터 모델·동일 메소드·동일 의미** — 차이는 구현 언어뿐. 다른 대칭 쌍과 본질이 다름.
- **결정**: BTIP-37 Appendix로 Go 인터페이스 흡수. 별도 번호 미사용(README `(Reserved)`).
- **사용자 판단**: 초기에 Go 인터페이스를 본문 별도 섹션으로 두려 했으나 "Appendix가 적당하지 않냐"는 redirect — 본문은 인터페이스 표준 그 자체에 집중, 언어별 표현은 부록으로.

### 부수 정리
- **결정 1 본문 표현 갱신 사항**: "공식 컨트랙트 세트의 진본성을 역할별로 관리하는 온체인 단일 출처" → "공식 컴포넌트 세트의 진본성을 `(chainId, role)` 단위로 관리". 정신은 그대로.
- **결정 6 본문 영향**: "btip-38 레지스트리 조회" 같은 표현은 모두 "btip-37 통합 레지스트리(BPrN 측 등록)" 식으로 명세 측에서 갱신 완료(btip-29, btip-34).

### 구현 결과 (파일, 2026-05-26)
- **개정**: btip-37(인터페이스 통합 + Chain ID Convention 절 + Go Appendix), btip-21(onResult 호출), btip-26(dApp 콜백 호출자 검증 2곳), btip-29(OnResult 호출 + BTIP-37 참조), btip-34(BTIP-37 참조).
- **삭제**: btip-38.md (인덱스에서 `(Reserved)`).
- **커밋**: `50198c7 docs: Unify LinkerRegistry under BTIP37 with (chainId, role) key`

---

## 9. 2PC 보안 보강 — 결과 핸들러 바인딩(#5a) + Nullifier 단위(#4) (2026-05-27)

§7의 보류 항목(설계 갭 점검)을 다시 검토하며 나온 결정·정정. §3~6의 의도 위에 누적.

### #5a — 결과 콜백에 "핸들러 식별자" 추가 (구현됨, 문서)

**문제(두 공격 벡터로 분해)**: "발신 앱이 받은 LinkerResult가 악의적으로 유도된 refund/settle가 아님을 어떻게 아는가?"
- **벡터 1 — 위조 LinkerResult**(악의적 컨트랙트가 직접 emit): `OnResult`/`onResult`의 **출처 검증**(소스 컨트랙트 == 공식 LinkerEndpoint, LinkerRegistry 조회)이 차단. (원래 설계, #5a 아님)
- **벡터 2 — 공식 엔드포인트 악용**(정방향 증명을 "항상 거부" 핸들러에 흘림): 공식 엔드포인트가 정당하게 `LinkerResult(correlationId, REJECTED, dApp=evil)`를 emit → 출처 검증 통과 → 잘못된 refund. **이게 #5a가 막는 부분.** 단순 오환불이 아니라 **정상 거래에 대한 griefing/DoS**(구매자는 서비스 못 받고 판매자는 미수금).

**해결**: 결과 콜백이 **핸들러 식별자**를 싣게 함.
- `handleLinkerResult(correlationId, handlerCcApp, accepted)`(btip-26) / `HandleLinkerResult(ctx, correlationId, handlerDApp, accepted)`(btip-34).
- `onResult`는 `LinkerResultElems.appChaincodeID`(gidx:8), `OnResult`는 `LinkerResult.dApp`을 꺼내 콜백으로 포워딩. **이벤트 정의는 이미 핸들러 필드를 보유 → 정의 변경 0, 콜백 인자+포워딩만.**
- **신뢰성**: 핸들러 필드는 공식 엔드포인트가 "자기가 호출한 대상"을 정직히 기록 + 증명·출처검증으로 위조 불가. → `handlerDApp == 의도한 dApp`인 결과를 얻으려면 *실제로* 그 dApp에 라우팅해야 하고, 그건 진짜 판정.
- **앱 책임(분담)**: 프로토콜은 "누가 판정했나"를 위조불가하게 공급. "누가 판정*해야* 하나(intended handler)"는 **앱이 lock 시점에 기록**하고 콜백에서 비교(불일치 시 무시). 이는 origin의 "의도한 핸들러" 설정의 거울상이며, 자금 risk를 지는 쪽이 per-instance로 소유 — LinkerRegistry는 비즈니스 dApp을 담지 않으므로 레지스트리로 대체 불가.
- **누가 intended handler를 결정하는가**: 본 #5a는 *기록 의무*만 정의. *누가 그 값을 정하는가*는 use case에 따라 다름 — dApp-orchestrated는 dApp이 사전 설정(immutable/owner-set), user-driven payment는 사용자 입력(Web2 결제 패턴, 1:N 시나리오에 필수). 1:N에서 사전 등록 모델이 깨지는 이유·user-input phishing 재해석은 `./bpun-origin-payment-design.md` §12.3 참조.

### #4 — Nullifier 소비 단위 = (root, app)당 1회로 확정 (구현됨, 문서)

요소/속성별 세분화하지 않음. 양방향 대칭(btip-24 `(tx_event_root, dApp)`, btip-33 `(event_attrs_root, appChaincodeID)`). 필요한 요소는 한 증명에 모두 포함. btip-24/33에 NOTE 명문화.

### #5b — 앱 책임으로 정리 (프로토콜 변경 없음)

"1 tx → N 액션"은 앱이 correlationId로 N개 액션을 기록해두고 단일 accept/reject를 일괄 적용하면 됨(현 핸들러가 이벤트 단위 all-or-nothing이므로). per-action 독립 판정이 필요할 때만 프로토콜(eventIndex) 확장 — 현재 불요.

### #5c — 철회 (비-이슈)

"BPrN-origin tx_event_root가 네트워크 정규값과 일치 명문화"는 불필요. `tx_event_root` 계산은 BTIP16 단일 정의이고, 발신 ccApp·네트워크가 동일 정의·동일 입력을 쓰면 결정론적으로 동일. 별도 보장 문구 불필요.

### 파일 변경 (2026-05-27)
- **btip-26/34**: 결과 콜백에 핸들러 인자 추가 + Escrow 섹션에 의도한-핸들러 바인딩 의무.
- **btip-21/29**: `onResult`/`OnResult`가 핸들러를 콜백으로 포워딩 + 라우팅 설명 갱신.
- **btip-24/33**: #4 NOTE.
- **btip-28/32**: stale 참조 → btip-39 정합 + H₀ admin-trust 명문화(`btips.md` 2026-05-27 참조).

> **관련 미구현 연구**: BPuN-origin "이벤트→BPrN 결제" 서비스(payer 인가, 화이트리스트 회피, payer=msg.sender, allowance 모델 등)는 `./bpun-origin-payment-design.md`에 별도 정리. 2PC와 인접하나 아직 설계 탐색 단계라 분리.
