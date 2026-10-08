# Linker for Besu — 설계 작업 컨텍스트

> 마지막 업데이트: 2026-10-07 — **아래 §0000(2026-10-07 BTIP 재편: 공통 인터페이스 50~54)부터 읽기.** 직전은 §000(방법 2 재편), §00(설계 결정·BTIP 초안), §0(착수·신뢰 사슬·gas 산정).  
> 트랙 성격: **설계 단계(구현 아님)**. Linker V2 개념(검증된 Tx/Event 사실을 상대 체인에 전달, 2PC는 앱 책임)을 BPrN(Fabric) 대신 **Hyperledger Besu(QBFT)** 위에서 구현할 수 있는지, 어떤 방법인지 탐색한다.  
> Linker V2 본류 기록은 `linker-v2.md`(별도 트랙). 이 파일은 Besu 트랙 전용 러닝 로그.

---

## 0000. 2026-10-07 BTIP 재편 — 기존 문서 수정 대신 신규 번호(49~55) + deprecated · 공통 인터페이스 50~54 초안

> 이 세션 본류: **(1) 리베이스 2회 대응(BTIP48 머클 사양·BTIP16 hashPair 제거) → (2) 사용자가 feat/besu 작업을 `BTIPS/.tmp/from-old-besu-branch/`에 백업하고 `06154b4`(main)로 되돌림 → (3) 문서 구조 재편 결정: 기존 BTIP(9/21/23/24/37)을 고치지 않고 새 번호로 작성, 기존은 deprecated → (4) BTIP49 + BTIP50~54 공통 인터페이스 작성(50은 본문까지 완성, 51~54는 인터페이스만).**

### 0000.1 저장소 상태 (docs, 브랜치 feat/besu)

- HEAD `52eb2a3`(.gitignore) ← `6ae0d68` BTIP49 ← `08deccd` README 번호 배정 ← `06154b4`(main: "Remove old merkle tree of btip16").
- **미커밋**: `README.md`(목차 50~55), `btip-49.md`(`replaces: btip9` 추가), `btip-50.md`~`btip-54.md`(신규, untracked). 커밋은 사용자 몫.
- `BTIPS/.tmp/from-old-besu-branch/`: 이전 세션(§000)의 산출물 백업 — btip-9/21/23/24/37/b0/b2/b4. **51~55의 재료**(VerifiedEvent/VerifiedResult, onProof/onResult 의사코드, Besu 증명 구조·검증 알고리즘·Validator Set 관리).
- main의 긴급 수정 요지(리베이스로 들어옴): BTIP48 "Beatoz Merkle Tree Specification"(배열 기반 완전 이진 트리, `leaf_hash = sha256(0x00‖data)`, `inner_hash = sha256(0x01‖l‖r)`, `EMPTY_HASH = sha256(null)`, `NULL_HASH = leaf_hash(null)`), BTIP16·17·19·23 등에서 `hashPair`/preHashed 제거, `block_event_root`는 **모든 tx**(실패 tx는 null 리프). 리베이스 충돌 해결로 BTIP23에 생겼던 미정의 변수(`tx_event_root`)·루프 변수 충돌은 고쳤으나, 그 BTIP23은 이후 되돌려져 백업본에만 있음.

### 0000.2 결정 사항

- **문서 재편 원칙**: 인터페이스가 바뀌는 문서는 새 번호 + 기존 deprecated. "BTIP5x 지원 여부"가 곧 설계 버전 판별. 코드와 문서 싱크 혼란 방지가 목적. 같은 원칙상 BTIP46(dApp 콜백), BTIP29/31/32/33(BPrN 체인코드)도 바뀌면 새 번호 대상.
- **번호**: 49 BPrN Identifier Conversion(9 대체), 50 LinkerRegistry(37 대체), 51 LinkerEndpoint, 52 LinkerNullifier, 53 LinkerVerifier, 54 LinkerPolicy, 55 BESU Tx/Event Proof(README 임시 배정, 파일 없음). 50의 번호 순서는 사용자 지정(Registry → Endpoint → Nullifier → Verifier → Policy).
- **배치 모델 = B**: Endpoint·Nullifier는 체인당 하나(공유), Verifier·Policy는 **소스 네트워크마다 인스턴스**(Besu 네트워크 둘이면 Verifier·Policy 둘). 이유: dApp 인터페이스를 "Endpoint 하나·콜백 하나"로 유지. A(네 개 전부 네트워크별)는 dApp에 체인 선택이 스며들어 기각.
- B에서 `srcChainId`의 역할: **Endpoint 라우팅 전용**. Verifier·Policy는 자기 소스 체인을 상태(`srcChainId()`)로 알므로 `verifyProof(bytes proof)`에 chainId 파라미터 없음. (이전 §000의 "체인 종류당 Verifier 하나, Policy가 chainId별 Validator Set 보관" 모델은 폐기.)
- **Nullifier 키에 `srcChainId` 포함**: `markProcessed(srcChainId, eventKey, handlerDApp)`. eventKey 유일성 책임을 소스 체인 안으로 한정, Verifier 하나의 오동작이 다른 체인 항목을 소비 못 하게.
- **Policy 공통화**: Verifier→Policy 호출을 `verifyCommitment(bytes32 digest, bytes commitment)` 하나로. BPrN: digest=`sha256(block_height‖block_event_root)`, commitment=`abi.encode(sigs, mspids, cert_chains)`; Besu: digest=`seal_hash`, commitment=`abi.encode(block_number, seals)`. Trust Anchor 등록·갱신은 체인별이라 공통 인터페이스에서 제외.
- **dApp 콜백에 `srcChainId` 추가 예정**(B의 귀결): `handleLinkerEvent(srcChainId, selector, indices, values)`, `handleLinkerResult(srcChainId, correlationId, requestHandler, accepted)` — BTIP46 후속 문서에서.
- **LinkerRegistry 키 = `(role, forChainId, atChainId)`**: `role`은 `enum Role { LinkerEndpoint, LinkerNullifier, LinkerVerifier, LinkerPolicy }`(해시 아님, 역할 집합이 닫혀 있음). `forChainId`=담당 체인, `atChainId`=배포 체인, **0 = block.chainid**(저장 시 정규화). 예: 자기 Endpoint `(LinkerEndpoint, 0, 0)`, S 담당 Verifier `(LinkerVerifier, S, 0)`, S 위의 상대 Endpoint `(LinkerEndpoint, 0, S)`. `setContract(..., address(0))` = 등록 해제. CREATE2 배포는 Specification 요구사항. 거쳐 간 안: `getContractAt/For` 분리(기각: 겹침), 단일 `getContract(chainId, role)`+role별 의미(기각: chainId 의미가 role에 따라 달라짐), 상대 Endpoint를 Verifier 상태로(기각: 공식 주소는 레지스트리 한 곳에).
- **BPrN 역할 식별자 통일**: `"LinkerEndpointRole"` 문자열 폐기, `Role` 열거값(0~3). BPrN 레지스트리 반환값 `contractId`는 `string`, 형식(체인코드명 vs BTIP49 주소)은 **구현이 정함**(공통 인터페이스가 강제 안 함).
- **BTIP49**: `channelId = uint256(sha256(channelName))`(접미사 없음), `chaincodeAddress = sha256(chaincodeName + "@" + channelName)[12:32]`. 용도는 **EVM 체인이 BPrN 채널·체인코드를 가리킬 때**(BPrN 체인코드 자체가 쓸 필연은 없으나 자기 채널 atChainId로 같은 값 사용). Abstract에 "채인" 오타 있음(미수정, 사용자 답 대기).
- **문서 스타일(이 세션에서 확정)**: 목차 Abstract/Motivation/Specification(/Changes from BTIPxx); **Rationale 불필요**(정의가 보안·구현상 문제없으면 이유 설명 안 함); 다른 문서 참조 최소화(역할→BTIP 대응표 같은 것 넣지 않음; 작성 시점에 51~54는 "존재하지 않는" 문서로 간주); 컨트랙트 간 호출은 BTIP 번호 대신 **의사 호출**(`registry.getContract(...)`, `policy.verifyCommitment(...)`)로; 주석 지시어 `@notice`(onlyOwner 등 주의)/`@dev`/`@param`; 인터페이스 언어는 Solidity로 통일, BPrN은 문서 말미 `### BPrN-Specific Considerations`에 Go 인터페이스; 대체 문서는 프런트매터 `replaces:` + Abstract 끝 "이 문서는 기존의 BTIPxx을 대체한다" + `## Changes from BTIPxx` 표(달라진 **내용만**, 이유 없음); 예시는 단일 문자 자리표시자 금지·현실적 이름(`stablecoin-channel`, `linker-chaincode`)·코드 블록·변수·`IBTIP50(registry).setContract(...)` 형태; BPrN 쪽은 contract가 아니라 **chaincode**.

### 0000.3 문서 현황

- **btip-50.md 완성**(Abstract·Motivation·Specification(Solidity 인터페이스, 저장 매핑, CREATE2)·BPrN-Specific Considerations(Identifiers for BPrN Chaincodes + LinkerRegistry Chaincode Interface(Go))·Changes from BTIP37 표). 프런트매터 `requires: btip49`, `replaces: btip37`.
- **btip-51~54**: Specification에 인터페이스만. Abstract 한 문장, Motivation 비어 있음, `## Rationale` 제목 남아 있음(50처럼 지울 것).
  - 51 `IBTIP51`: `LinkerResult(correlationId, handlerDApp, status)`, `ErrUntrustedSource(srcChainId, emitter)`, `setRegistry`, `onProof/onResult(srcChainId, bytes proof, handlerDApp)` — @dev에 6단계 의사 호출(eventKeyOf → markProcessed → verifyProof → handleLinkerEvent → emit; onResult는 verifyResultProof → emitter == `registry.getContract(IBTIP50.Role.LinkerEndpoint, 0, srcChainId)` → handleLinkerResult).
  - 52 `IBTIP52`: `ErrDuplicateProof/ErrNotProcessed(srcChainId, eventKey, handlerDApp)`, `setRegistry`, `isProcessed/markProcessed(srcChainId, eventKey, handlerDApp)`, `cancelNullifier(srcChainId, eventKey)`(handlerDApp = msg.sender). markProcessed는 onlyLinkerEndpoint(`registry.getContract(IBTIP50.Role.LinkerEndpoint, 0, 0)`).
  - 53 `IBTIP53`: `VerifiedEvent{eventKey, correlationId, srcBlockNumber, srcTxIndex, selector, indices, values}`, `VerifiedResult{eventKey, emitter, correlationId, requestHandler(bytes), accepted}`, `ErrNotLinkerResult(selector)`, `setRegistry`, `srcChainId()`, `eventKeyOf(proof)`, `verifyProof(proof)`, `verifyResultProof(proof)`.
  - 54 `IBTIP54`: `ErrCommitmentRejected()`, `setRegistry`, `srcChainId()`, `verifyCommitment(bytes32 digest, bytes commitment)`.
- README: 49~55 ***Draft*** 항목. BTIP9/37 등 기존 문서에는 아직 *DEPRECATED*·`superseded-by:` 미표기.

### 0000.4 다음 작업 (내일 이어서)

1. 51~54: `## Rationale` 제목 제거, Motivation 작성, 50과 같은 스타일로 본문 보강(51은 `Changes from BTIP21`, 52는 `Changes from BTIP24`, 53은 `Changes from BTIP23`, 54는 `Changes from BTIP22` 표), 각 문서 말미 BPrN-Specific Considerations(Go 인터페이스) — 사용자가 "BPrN은 다음 단계"라 했으므로 지시 받고 진행.
2. 51~54 재검토 포인트: 53 `VerifiedEvent.srcBlockNumber/srcTxIndex`가 공통 필드로 필요한지; 51 onProof 5단계의 `handleLinkerEvent(srcChainId, ...)` 시그니처는 BTIP46 후속 문서와 맞출 것; 52 Nullifier 계산식(`sha256(abi.encode(srcChainId, eventKey, handlerDApp))`)은 Specification에 아직 없음.
3. BTIP55 BESU Tx/Event Proof: 백업 `btip-b0.md`를 재료로 신규 작성(b0는 이미 사용자 리뷰 반영본). 이어서 체인별 구현 문서(BPrN-origin Verifier+Policy 구현 — 백업 btip-23/22 재료, Besu-origin — 백업 btip-b2/b4 재료) 번호·구성 결정 필요.
4. 기존 문서 deprecated 처리: 9/21/22/23/24/37에 `superseded-by:` + README *DEPRECATED* 표기(사용자 지시 시).
5. BTIP46 후속(dApp 콜백에 srcChainId, `handleLinkerResult(..., bytes requestHandler, ...)`), BPrN 체인코드 측(29/31/32/33) 후속 — 번호 미정.
6. BTIP49 Abstract "채인" 오타(사용자 답 대기).
7. §000.2의 코드 작업 목록(Jira)은 이번 재편으로 **내용이 바뀜**(레지스트리 키 3차원·enum Role, Verifier/Policy 네트워크별 인스턴스, Nullifier srcChainId, verifyCommitment, dApp 콜백 srcChainId) — 51~55 확정 후 다시 정리.

---

## 000. 2026-09-08 BTIP 리뷰 반영 · 방법 2(코드 층 추상화) 재편 · 코드 작업 목록

> 이 세션 본류: **b0~b4 사용자 리뷰 반영 → "b1은 BTIP21과 동일 인터페이스" 문제 제기 → 방법 2(증명을 `bytes`로, 체인별 LinkerVerifier가 `VerifiedEvent` 반환) 채택 → BTIP21/23/24 일반화, b2 재작성, b1·b3 폐기.** 코드 수정은 미착수(아래 000.2 목록 = Jira 등록용).

### 000.1 한 일

- **리뷰 반영(재편 전)**: b0에서 correlationId·event_id 제거(스코프: dApp 명세·Nullifier 소관), 부재 해명 문장 제거("Besu-origin만 다룬다", "수정 없이 사용", "contract 모드에서는 성립하지 않는다" 등), b0를 Data Structure→Construction→Verification 우선순위로 재작성하고 Besu 내부 인코딩은 Appendix로, TIP는 선택적 사전 검사로 축소("증명이 제출되어야 한다" 수동태·주체 미특정), 검증 단계 설명은 `payload.멤버` 표기, `handlerCcId` 제거, 구조체를 인터페이스 안으로 이동. **용어 '결과증명' = BPuN-origin 증명을 처리한 LinkerEndpoint/Source Chain이 발행한 결과 이벤트의 증명**(Terms로 정의).
- **방법 2 재편(사용자 결정)**:
  - **BTIP21** 일반화: `onProof/onResult(uint256 sourceChainId, bytes proof, address handlerDApp)`. Terms에 Source Chain·결과증명. 순서: `eventKeyOf` → `markProcessed` → `verifyProof` → 스키마 → 콜백 → `LinkerResult(ve.correlationId)`. onResult는 `verifyResultProof` → `vr.emitter == registry(sourceChainId, LINKER_ENDPOINT)` → `handleLinkerResult(vr.correlationId, vr.handler(bytes), vr.accepted)`. 레지스트리 키 `LINKER_VERIFIER_FOR(chainId) = keccak256(abi.encode(LINKER_VERIFIER, chainId))` under self chain(`(sourceChainId, LINKER_VERIFIER)`는 BTIP37상 상대 체인 위의 컨트랙트라 회피).
  - **BTIP23** = `IBTIP23` 공통 인터페이스(`VerifiedEvent{eventKey, correlationId, srcBlockNumber, srcTxIndex, selector, indices, values}`, `VerifiedResult{eventKey, emitter, correlationId, handler(bytes), accepted}`, `eventKeyOf`, `verifyProof`, `verifyResultProof`, `ErrNotLinkerResult`) + **BPrN 구현**(구조체 `BPrNTxEventProof`가 IBTIP21에서 여기로 이동, eventKey=correlationId=tx_event_root, selector=gidx3, emitter=BTIP9 파생 주소, 결과 gidx 4/5/6).
  - **BTIP24**: `eventRootHash`→`eventKey`, Source Chain 무관으로 서술.
  - **b2** 재작성 = IBTIP23의 Besu 구현: `abi.encode(BesuTxEventProof)`, eventKey=keccak(sourceChainId, block_number, tx_index, log_index(receipt 내 순번)), correlationId는 dApp/Besu BTIP(별도) 정의, selector=topic_0, gidx 표(0 address,1 topic_0,2~4 topic_1~3,5 data), 정책 키 `LINKER_POLICY_FOR(chainId)`, verifyResultProof는 `LinkerResult(bytes32,address,uint8)` selector 확인 후 topic_1/topic_2/data 추출.
  - **b1·b3 폐기** → `docs/BTIPS/_to_delete/`로 이동(삭제 권한 거절됨, 사용자가 직접 삭제). b0·b4 참조 정리(`requires` 갱신).
- **용어 정리(재편 후 리뷰)**: `sourceChainId`→`srcChainId`(의사코드 `src_chain_id`); 'Source Chain' Terms 항목 삭제, 산문은 '소스 체인'/'목적 체인'; `VerifiedResult.handler`→**`requestHandler`**(`handledBy`는 "Result를 handle할 앱"으로 오독, `dstHandler`는 `srcChainId`와 관점 충돌로 기각); BTIP21 Rationale에 **관점 고정** 절 신설 — `src*` 식별자는 증명 대상 이벤트가 발생한 체인(결과증명이면 결과 이벤트를 발행한 체인), '목적 체인'은 요청 관점 용어, 결과증명에서는 둘이 같은 체인.
- 문서 규칙 재확인: 문서 간 경계(증명/엔드포인트/nullifier/dApp), 부재 해명 금지, 우선순위(핵심 절 먼저·분석은 부록), 체인 지칭은 문맥 용어(소스/목적).

### 000.2 남은 작업 — 코드 (Jira 등록용 요약)

**A. BPuN Linker 코어 리팩터(방법 2)** — `linker-v2/verifier/on-bpun`
1. `IBTIP23` 신설: `VerifiedEvent`/`VerifiedResult` 구조체, `eventKeyOf`/`verifyProof(uint256, bytes)`/`verifyResultProof`. 기존 `LinkerVerifier.verifyProof(BPrNTxEventProof)`를 BPrN 구현 `LinkerVerifierBPrN`으로 개명·이관(`abi.decode`, correlationId/selector/emitter/indices/values 구성, 결과증명 gidx 4/5/6 해석).
2. `IBTIP21`/`LinkerEndpoint`: 시그니처 `onProof/onResult(uint256 sourceChainId, bytes proof, address handlerDApp)`, 구조체 제거, 흐름을 BTIP21 §Implementation대로(eventKeyOf → markProcessed → verifyProof → 스키마 → 콜백 → LinkerResult; onResult 출처 검사는 `vr.emitter`).
3. `IBTIP24`/`LinkerNullifier`: 파라미터명 `eventRootHash`→`eventKey`(ABI 불변, 이름만).
4. `IBTIP37`/`LinkerRegistry`: 역할 파생 규칙 `LINKER_VERIFIER_FOR(chainId)`, `LINKER_POLICY_FOR(chainId)` 상수/헬퍼 추가 + BTIP37 문서 반영. 배포 스크립트(`deploy.ts`·`bootstrap.ts`)에서 BPrN Verifier를 새 키로 등록. **영향 범위(2026-09-08 조사)**: 현 저장 구조 `(chainId=배포 체인, role)`, 외부 접근자는 dApp(BTIP46Token: `LINKER_ENDPOINT`·`LINKER_NULLIFIER`), prover-ts(btip19: `LINKER_ENDPOINT`, btip18: `LINKER_POLICY`), test/helpers·bootstrap. `LINKER_VERIFIER` 키는 외부 미사용 → `_FOR` 전환은 Endpoint·bootstrap만. `LINKER_POLICY`를 `_FOR`로 옮기면 btip18 prover(`util/linker_registry.ts`)도 갱신 필요 — BPrN은 기존 키 유지/전환 여부 결정.
5. `IBTIP46`: `handleLinkerResult(bytes32 correlationId, bytes requestHandler, bool status)`로 변경(현 `string handlerCcId`) + BTIP46 문서 반영. 기존 dApp(BTIP46Mocks, test dApp) 갱신.
6. prover-ts btip19 submitter: `onProof(bprnChainId, abi.encode(payload), handlerDApp)`로 제출 형식 변경(`bprnChainId = uint256(sha256(channel+"/BPrN"))`).
7. 테스트: on-bpun Foundry/Hardhat 테스트 갱신, `test/` e2e(r2u·u2r) 재실행, gas 회귀 확인(bytes 디코드 추가분).

**B. Besu 신규(BPuN 측)** — 신규 패키지 제안 `linker-v2/verifier/on-bpun-besu` 또는 on-bpun 내 `besu/`
8. `LinkerVerifierBesu`(BTIP-b2): RLP 리더, 헤더 파싱, MPT 워크, receipt/log 디코드, VerifiedEvent 구성. 프로토타입(`BesuReceiptVerifier.sol`, 세션 zip)을 기반으로 하되 calldata 직접 참조로 최적화(MPT 노드당 ≈14k → 목표 절반).
9. `LinkerPolicyBesu`(BTIP-b4): 높이별 Validator Set 저장, `verifyCommittedSeals`, `initValidatorSet`, `updateValidatorSet`(sequential·idempotent).
10. Besu prover(prover-ts 신규 모듈, BTIP-b0 §Construction): `eth_getTransactionReceipt`/`debug_getRawHeader`(또는 JSON 재조립+blockHash 자가검증)/`debug_getRawReceipts`(또는 `eth_getBlockReceipts` TRIE_ROOT 인코딩+receiptsRoot 자가검증), Receipts Trie 재구성·증명 추출, seal 정족수 선택, `abi.encode(BesuTxEventProof)` 제출. Validator Set 변경 탐지·`updateValidatorSet` 제출.
11. Besu QBFT localnet(4 validators) 구성 + 실 블록으로 fixture 생성 → LinkerVerifierBesu 통과 확인(round≠0 케이스 포함) + gas 실측.
12. BPuN EVM 버전 확인(`mcopy` 가용 여부) → 미지원 시 복사 루틴 대체.

**C. 문서(코드 착수 전 선행)**
13. dApp/Besu 인터페이스 BTIP(BTIP46/26의 Besu 대응): correlationId 위치(topic_1 등) 규약, 이벤트 정의, 2PC 콜백. → b2 `correlation_id_of` 확정.
14. LinkerEndpoint/Besu 등 Besu 측(BPuN→Besu 방향) BTIP: BTIP28/29/31/32/33/34/39의 Solidity 대응(차이만). `LinkerResult(bytes32,address,uint8)` 시그니처 확정.
15. `docs/BTIPS/README.md` 목차에 b0·b2·b4 추가, BTIP21/23/24 제목 확인. `_to_delete/` 정리.

---

## 00. 2026-09-07 설계 결정 · BTIP-b0~b4 초안 (여기서부터 이어서 읽기)

> 이 세션 본류: **Besu 수정 여부 결정(무수정 채택) → committedSeal 서명 입력·Besu 수정 지점 정밀 분석 → 프라이버시/nullifier 문제 정리 → BTIP-b0~b4 초안 작성(docs/BTIPS/btip-b0.md~b4.md, README 미반영).**

### 00.1 한 일

- **설계 결정: 무수정 Besu + receipts MPT 증명을 기본안으로.** 근거: 코어 검증 ≈100k~170k gas(최대 TPS 블록 ≈215k)로 현행 BPrN 증명과 같은 자릿수, 포크 시 운영 부담(업스트림 추종·합의 코드 회귀)과 "어떤 Besu QBFT 네트워크든 연결 가능"이라는 이점 상실. 사용자 결정: "나중에 따져 보기로".
- **보류 옵션(포크안) 기록**: extraData에 `blockEvtLogRoot`(receipt 요소 이진 머클 root의 블록 트리) 필드 추가 → seal 서명에 자동 결속 → verifier가 헤더 없이 seals+root+이진 머클 증명만으로 검증(≈45~65k). 프라이버시(요소 단위 선택 공개)가 프로토콜 레벨로 회복되는 것이 진짜 이점. Besu 수정 범위: `BftExtraData`·`QbftExtraDataCodec`(encode 3브랜치+decode)·`BftBlockCreator`·ProposalValidator 후 root 일치 검사·genesis. **receiptsRoot 자체 수정안은 비권장**(이더리움 블록 형식 표준을 깨면서 branch 노드가 남아 이득 작음).
- **committedSeal 서명 입력 확정**(Besu main 소스): `seal_hash = keccak256(RLP(header with extraData = RLP([vanity, validators, vote, round(유지, int scalar), []])))`, seals 자리는 필드 생략이 아니라 **빈 리스트 0xc0**. vote 없으면 `0xc0`, 있으면 `[recipient, 0xff(ADD) | null(DROP)]`. seal = r||s||recId, **recId 0/1**(ecrecover엔 +27). 프리픽스 없음. 헤더 크기 ≈510+21n B(London), Cancun 포크면 +70.
- **Besu 수정 지점 분석**: 커밋 해시 계산이 두 곳 — (A) `BftBlockHashing.calculateDataHashForCommittedSeal`(오버로드 2, `forCommittedSeal` 헤더함수 경유: seal 생성·Commit 검증·Proposal/RoundChange digest) + (B) `BftBlockInterface.getCommitters`(중복 구현, 블록 임포트 `BftCommitSealsValidationRule`·RPC). (A)만 바꾸면 임포트에서 거부. 공통 choke point = `encodeWithoutCommitSeals`(extraData 내용 변경 시 1곳) / `serializeHeader`(온체인 해시와 공유라 분기 필요). 변경 단계: ①extraData 내용만→codec 1곳 ②새 필드→BftExtraData+codec+creator+검증규칙 4~5파일 ③직렬화/해시 변경→(B) 단일화 후 (A). `recoverCommitterAddresses`는 QBFT 메인에서 미사용.
- **프라이버시**: 무수정 경로는 receipt 원본이 leaf라 tx의 모든 로그·필드 노출(같은 tx의 ERC20 Transfer 포함). **무수정 버전에서는 비고려로 결정.** (대안 기록: 발행 컨트랙트가 요소 BTIP16 머클 root를 emit → 요소 단위 증명. 포크안이면 프로토콜 레벨 해결.)
- **이벤트 대표값 결정**: `event_id = keccak256(abi.encode(source_chain_id, block_number, tx_index, log_index))`, log_index는 **receipt 내부 순번**(RPC 전역 logIndex 아님). nullifier = sha256(abi.encode(event_id, handlerDApp)). 로그 1개 = 이벤트 1개.
- **BTIP-b0~b4 초안 작성·저장**(docs/BTIPS/): b0 Besu Tx/Event Proof(신뢰 사슬·event_id·Event Elements gidx 표·prover/verifier 알고리즘), b1 LinkerEndpoint(BTIP21 diff: `onProof(sourceChainId, payload, handlerDApp)`, correlationId=topic_1, handlerCcId=hex 주소 문자열), b2 LinkerVerifier(Step 2/4/5 직접, Step 3은 b4 위임, `verifyProof` 가 (emitter, topics, data) 반환), b3 LinkerNullifier(BTIP24 diff: eventRootHash→eventId), b4 LinkerPolicy(Validator Set 높이별 보관 `(sinceBlock, validators)`, `verifyCommittedSeals`, `initValidatorSet` owner 1회, `updateValidatorSet` permissionless sequential — 블록 N seal을 N 유효 집합으로 검증 후 헤더 validators를 sinceBlock=N+1로 등록). 인터페이스명 `IBTIPB1`~`IBTIPB4`(임시). md 개행 공백 2개 규칙 적용.
- **초안에서 내가 정한(사용자 미확정) 설계 포인트** — 리뷰 필요:
  1. `correlation_id`를 **topic_1 예약**으로 정의(dApp/Besu가 emit 시 스스로 정함, 동일 dApp 내 유일). 이유: event_id는 위치 기반이라 발행 시점 dApp/Besu가 알 수 없어 2PC 식별자와 nullifier 기준값이 분리됨.
  2. `onProof/onResult`에 **`sourceChainId` 파라미터 추가**(다중 Besu 네트워크; payload에 체인 식별자 없음). chainId = Besu EVM chain id 그대로(BTIP37 키).
  3. Event Elements gidx: 0=address, 1=topic_0(selector), 2=topic_1(correlation_id), 3~4=topic_2~3, 5=data(ABI 전체 한 leaf).
  4. `handleLinkerResult.handlerCcId`(string)에 Besu 주소를 0x hex 문자열로 전달(IBTIP46 시그니처 유지).
  5. b4 Validator Set 모드 = blockheader 전제, contract 모드 비지원 명시.
  6. LinkerResult/Besu 이벤트 시그니처를 BPuN과 동일(`LinkerResult(bytes32,address,uint8)`)로 가정 — Besu 측 문서에서 확정 필요.

### 00.2 남은 작업

- **BTIP-b0~b4 사용자 리뷰** → 위 6개 설계 포인트 확정/수정. `docs/BTIPS/README.md` 목차 반영(미수정, 컨펌 후).
- **Besu 측 컨트랙트 문서**(BPuN→Besu 방향, BPrN 체인코드 btip28/29/31/32/33/34/39의 Solidity 대응): 별도 BTIP 필요하나 "차이만" 규칙으로 축소 가능. Tendermint commit 서명·RFC6962 검증을 EVM에서 수행 — 검증 비용 산정 미실시.
- §0.5 이월: 실 Besu localnet 블록으로 프로토타입 재검증(round≠0 포함), BPuN EVM 버전(mcopy) 확인, prover 구현 시 `debug_getRawHeader/debug_getRawReceipts` 의존 여부 결정.
- Besu 포크안(extraData blockEvtLogRoot)은 "gas가 병목이 되거나 프라이버시가 요구될 때" 재검토.

---

## 0. 2026-09-07 착수 — 블록 구조 신뢰 사슬 · 검증자 집합 모드 · 증명 크기/gas 산정

> 이 세션 본류: **Besu 블록 구조에서 topics → receiptsRoot → 헤더 → committedSeals 신뢰 사슬 확정(소스 확인), QBFT 검증자 집합 모드 비교, Besu→BPuN 증명 페이로드 크기·EVM 검증 gas 실측 기반 산정.**  
> 확인 근거: Besu `main` 브랜치 소스(`TransactionReceiptEncoder`, `TransactionReceiptEncodingConfiguration`, `BodyValidation`, `BftBlockHashing`, `QbftRound`, `QbftExtraDataCodec`, `BftHelpers`, `TransactionValidatorProvider`), besu-docs QBFT 페이지.  
> gas 수치는 **프로토타입 검증 컨트랙트를 Foundry로 실측**한 값(§0.4). 프로토타입은 산정용이며 코드 저장소에 넣지 않았다.

### 0.1 신뢰 사슬: topics → receiptsRoot → 헤더 → committedSeals

실행 계층(receiptsRoot까지)은 이더리움 표준 그대로, 합의 계층(헤더 해시·서명)은 QBFT 고유.

**(1) Log**
```
Log = RLP([ address(20), [topic0..topicN](각 32, 최대 4), data(bytes) ])
```
origin-proof의 "실제 이벤트 데이터" = address / topics / data.

**(2) Receipt — trie root 용 인코딩**  
`BodyValidation.receiptsRoot()`는 `TransactionReceiptEncodingConfiguration.TRIE_ROOT`(= `withOpaqueBytes(false)`만 명시, 나머지 기본값)로 인코딩 → **bloom 포함, revertReason 미포함, compact log 미사용**.
```
legacy : RLP([ status(0/1) | preByzantium stateRoot, cumulativeGasUsed, logsBloom(256), [Log...] ])
typed  : txType(1 byte) || RLP([ status, cumulativeGasUsed, logsBloom, [Log...] ])   -- EIP-2718, 타입 바이트를 리스트 앞에 raw로 씀
```
- `status` vs `stateRoot`: `receipt.getStateRoot() != null`이면 stateRoot, 아니면 status scalar. QBFT 프라이빗넷은 Byzantium 이후 → status.
- **검증자는 bloom 256바이트를 포함한 receipt 전체를 받아야 leaf 해시를 재구성할 수 있다.** bloom 생략 불가.
- 한 tx의 이벤트가 많으면 receipt(=leaf) 크기가 그대로 증명 크기로 간다(§0.3 `v4_tx8_5logs`).

**(3) Receipts MPT**
```
key   = RLP(txIndex)          (블록 내 tx 순번, unsigned scalar; 0 → 0x80, 1..127 → 단일 바이트)
value = 위 Receipt 인코딩
receiptsRoot = MPT(key→value).root   (keccak256)
```
`Util.getRootFromListOfBytes()`가 인덱스를 키로 표준 MPT 구성. `transactionsRoot`도 같은 함수(인코딩만 `EncodingContext.BLOCK_BODY`).  
포함 증명 = root → branch/extension… → leaf 노드 RLP 목록(eth_getProof 형태). 각 노드를 keccak하여 부모의 참조와 대조하며 내려감.

**(4) 헤더와 블록 해시 — QBFT 고유**  
헤더 필드는 이더리움 표준(London 이후 16개: …, receiptsRoot(index 5), …, extraData(index 12), …, baseFeePerGas). `extraData`는 `QbftExtraDataCodec`:
```
extraData = RLP([ vanity(32 bytes), [validatorAddr(20)...], vote(있으면 [addr, type], 없으면 빈 리스트 []), round(int scalar), [committedSeal(65)...] ])
```
`BftBlockHashing`이 두 해시를 구분:
- **커밋 서명 대상 해시** `calculateDataHashForCommittedSeal`: committedSeals만 비우고 **round 유지**한 헤더 RLP의 keccak256. `QbftRound`에서 `replaceRoundForCommitBlock(block, round)` 후 `commitBlock.getHash()`를 `nodeKey.sign()` → committedSeal.
- **온체인 블록 해시** `calculateHashOfBftBlockOnchain`: committedSeals 비우고 **round=0**으로 바꾼 헤더 RLP의 keccak256. `parentHash`·RPC `hash`가 이것. (같은 블록이 라운드만 바뀌어 재제안돼도 해시 동일하게 하기 위함.)

→ **검증자(BPuN)가 받아야 하는 것**: 헤더 전체 + vanity + validators + vote + round + seals. seals 검증에는 round 유지 RLP, 체인 연결(parentHash)에는 round=0 RLP — RLP 재구성이 최대 2회. **Linker 용도(receiptsRoot를 seals에 결속)에는 seals 검증 1회면 충분**하고 블록 해시는 필요할 때만.

**(5) committedSeals → 검증자 집합**  
seal마다 ecrecover → 주소가 현재 검증자 집합에 속하고, 중복 없이 **정족수 `ceil(2n/3)`**(`BftHelpers.calculateRequiredValidatorQuorum` = `fastDivCeiling(2n, 3)`) 이상이면 확정.  
QBFT는 **즉시 확정(포크 없음)** → 헤더 1개 + 정족수 seals로 finality. confirmation depth 개념 불필요(단, <1/3 비잔틴 전제).

```
topics/data/address
  → Log RLP
  → Receipt RLP (status, cumGas, bloom, logs)   [+ txType 바이트]
  → Receipts MPT leaf (key = RLP(txIndex))
  → receiptsRoot
  → BlockHeader(round 유지, seals 제거) → keccak → committedSeal 서명 대상
  → committedSeals ecrecover → ValidatorSet ∩ 정족수 ceil(2n/3)
  → (round=0, seals 제거) keccak → 온체인 blockHash / parentHash   [선택]
```

### 0.2 QBFT 검증자 집합 모드 — 무엇을 동기화할 것인가

Besu QBFT `validatorselectionmode`는 두 가지. Besu→BPuN 방향에서 BPuN이 Besu 검증자 집합을 추적해야 하므로(현행 BPuN→BPrN의 ValidatorSet 동기화의 대칭), 모드 선택이 곧 동기화 설계.

| | **blockheader 모드** | **contract 모드** |
|---|---|---|
| 집합의 소재 | 매 블록 `extraData.validators`(초기값 genesis extraData) | `validatorcontractaddress` 컨트랙트의 `getValidators()`; extraData.validators는 **빈 리스트** |
| 블록 N 검증에 쓰는 집합 | 블록 N-1 extraData의 validators (`BlockValidatorProvider.getValidatorsAfterBlock(parent)`) | 블록 N-1 **상태**에서 `getValidators()` 실행 (`TransactionValidatorProvider.getValidatorsAfterBlock(parent)`) |
| 변경 방식 | 검증자 투표(JSON-RPC `qbft_proposeValidatorVote`), 과반 도달 블록의 extraData에 반영, `epochlength`(기본 30000)마다 투표 리셋 | 컨트랙트 로직(임의 거버넌스), 변경은 상태 변화 |
| 외부(BPuN) 라이트클라이언트가 집합 갱신하는 방법 | **자기완결**: 블록 N의 seals를 보유 집합 S(N-1)로 검증 → 같은 헤더의 extraData.validators = S(N)을 새 집합으로 채택. 집합이 바뀐 블록만 순차 제출하면 됨(중간 블록 생략 가능). 추가 증명 없음 | (a) stateRoot 아래 컨트랙트 스토리지 MPT 증명(eth_getProof, 계정 trie + storage trie, 컨트랙트 레이아웃 의존) — 무겁고 레이아웃 결속, 또는 (b) 검증자 컨트랙트가 변경 이벤트를 emit하도록 하고 그 receipt를 §0.1 사슬로 증명 — 일반 이벤트 증명과 동일 메커니즘, "블록 m 이벤트 → 블록 m+1부터 유효" 규약 필요 |
| 모드 전환 | `transitions.qbft[{block, validatorselectionmode, validatorcontractaddress}]`, 전 노드 재시작 | 동일 |

**판단(제안, 미확정)**: **blockheader 모드**를 1안으로 둔다. 근거: 집합이 헤더 자체에 실려 증명이 자기완결이고, "집합 변경 블록을 순차 제출"이라는 규칙이 현행 BTIP39(BPuN ValidatorSet 체이닝 갱신)와 구조가 같다. contract 모드는 온체인 거버넌스가 요구될 때 (b) 이벤트 방식으로 수용 가능하나, 이벤트-유효시점 규약과 컨트랙트 커스터마이징(표준 `ValidatorSmartContractInterface`는 `getValidators()`만)이 추가된다.

주의: blockheader 모드에서 BPuN 검증자는 **"최초 집합 S0"만 admin/genesis로 심고**(현행 LinkerPolicy genesis 정책과 동형), 이후는 헤더 증명으로만 갱신. 집합이 바뀐 블록을 건너뛴 채 나중 블록을 제출하면 seals 검증이 실패하므로 relayer가 변경 블록을 먼저 제출해야 함 — Linker "동기 부여된 자가 제출" 모델과 충돌 없음(제출자가 필요한 헤더를 함께 내면 됨).

### 0.3 증명 페이로드 크기 (합성 블록, London 헤더, typed receipt)

가정: 1 log(3 topics, data 64B), 다른 tx는 log 1개(2 topics, 32B). n=검증자 수, q=ceil(2n/3). 단위 bytes.

| 시나리오 | n/q | 헤더(seals 제거) | seals(65×q) | MPT 노드 | receipt(leaf) | raw 합 | ABI 인코딩 후 |
|---|---|---|---|---|---|---|---|
| v4_tx1 (블록 tx 1개) | 4/3 | 635 | 195 | 468 (1노드=leaf) | 459 | 1,298 | 1,956 |
| v4_tx8 | 4/3 | 636 | 195 | 83+243+467 = 793 | 460 | 1,624 | 2,436 |
| v4_tx50 | 4/3 | 636 | 195 | 179+532+467 = 1,178 | 460 | 2,009 | 2,820 |
| v4_tx200 | 4/3 | 636 | 195 | 5노드 1,569 | 460 | 2,400 | 3,364 |
| v7_tx8 | 7/5 | 699 | 325 | 793 | 460 | 1,817 | 2,820 |
| v10_tx8 | 10/7 | 762 | 455 | 793 | 460 | 2,010 | 3,204 |
| v16_tx8 | 16/11 | 891 | 715 | 793 | 460 | 2,399 | 3,972 |
| v4_tx8_5logs (5 log×4 topics, data 256B) | 4/3 | 636 | 195 | 83+243+2,363 | 2,356 | 3,520 | 4,324 |

- 헤더 고정분 ≈ 505B(그중 블록 logsBloom 256B — 대부분 0이라 calldata 4 gas/byte) + extraData(32 vanity + 21×n + 약 5).
- receipt 460B 중 256B가 bloom. 이벤트 증명 1건의 최소 비용 하한이 여기서 결정됨.
- MPT 깊이: 블록 tx ≤ 16이면 2단(root branch → leaf), 수백이면 3~4단. 프라이빗넷에서는 2~3노드가 일반.
- 프로토타입은 헤더를 **seals 제거·round 유지 상태로 prover가 미리 만들어 제출**(seals는 별도 배열) → 검증자가 헤더 RLP를 재구성하지 않고 `keccak(headerStripped)`가 곧 서명 대상. seals가 그 바이트열에 대한 서명이므로 prover가 헤더를 조작하면 seals가 맞지 않아 안전. 검증자는 extraData 꼬리의 seals 리스트가 빈 리스트인지만 확인.

### 0.4 EVM 검증 gas 산정 (Foundry 실측, solc 0.8.28, optimizer 200, via-IR, cancun)

프로토타입 `BesuReceiptVerifier.verify(Proof)`: 헤더 파싱(receiptsRoot·validators 추출, `keccak(validators concat)`을 저장된 `validatorsHash`와 비교 = SLOAD 1회) → seals ecrecover·멤버십·중복·정족수 → MPT 워크 → receipt에서 log(address, topics, data) 추출 → 이벤트 emit. **Python(py-trie/rlp/eth-keys)으로 독립 생성한 fixture를 통과**했으므로 인코딩 해석은 교차검증됨.

| 시나리오 | exec gas | calldata gas | 합계(+21,000 base) |
|---|---|---|---|
| v4_tx1 | 62,876 | 16,764 | **≈100.6k** |
| v4_tx8 | 91,852 | 22,608 | **≈135.5k** |
| v4_tx50 | 92,344 | 28,836 | ≈142.2k |
| v4_tx200 | 125,516 | 35,752 | ≈182.3k |
| v7_tx8 | 104,615 | 26,544 | **≈152.2k** |
| v10_tx8 | 117,745 | 30,468 | ≈169.2k |
| v16_tx8 | 146,111 | 38,280 | ≈205.4k |
| v4_tx8_5logs | 100,686 | 53,392 | ≈175.1k |

단계별(exec 내역):
- 헤더 파싱: ≈21k(n=4) + ≈1.3k/검증자 (validators 복사를 바이트 루프로 하면 50k+8.6k/검증자였음 — `mcopy`로 교체해 절감. **BPuN EVM이 Cancun 미만이면 identity precompile/루프로 대체, 수 k 차이**)
- seals: ≈4.4k/seal (ecrecover 3,000 + 호출·멤버십 스캔)
- MPT: ≈12.5k(leaf 1노드) / ≈41k(3노드) / ≈75k(5노드) — branch 노드 파싱(17항목)이 지배
- log 추출: ≈12.5k(460B receipt)

**결론(산정)**: Besu→BPuN 이벤트 증명 1건의 Linker 코어 검증은 **검증자 4~10개 기준 약 100k~170k gas**, 큰 블록/많은 이벤트에서도 250k 이내. 여기에 현행 LinkerEndpoint 수준의 부대비용(nullifier SSTORE ≈22k, registry/policy 조회, dApp 콜백)이 얹히면 **대략 150k~300k** — 사용자가 언급한 현행 BPrN 증명(수십만 gas)과 같은 자릿수이거나 낮다. ECDSA(secp256k1) ecrecover가 프리컴파일이라 현행 P-256/X.509 검증 대비 서명 비용 자체는 크게 유리.

한계: 합성 블록(실 Besu 노드 RLP 아님), 프로토타입 검증기(검증 로직 완전성보다 비용 추정 목적; 예: 헤더 필드 개수/포크별 필드 검증, extraData 경계 검증 최소화), calldata gas는 EIP-2028 단가(16/4) 가정.

### 0.5 남은 작업 / 미결 결정

- **실 Besu 블록으로 재검증**: QBFT localnet 띄워 `eth_getBlockByNumber` 헤더 RLP·`eth_getTransactionReceipt`·receipts trie 증명(Besu는 `eth_getProof`가 계정/스토리지만 지원 — receipts 증명은 relayer가 `eth_getBlockReceipts`로 전체 receipt 받아 로컬 trie 구성해야 함, 이것이 prover 설계의 핵심 입력) 생성 → 프로토타입 통과 확인. round≠0 블록(라운드 변경 발생)도 케이스에 포함.
- **검증자 집합 모드 결정**(§0.2 제안 = blockheader). 결정 후 "집합 변경 블록 순차 제출" 규칙과 초기 집합 S0 부트스트랩을 BTIP 초안으로.
- **BPuN EVM 버전 확인**(mcopy 등 Cancun 옵코드 가용 여부) — 산정치 보정.
- **증명 페이로드 스키마**: `headerStripped` + `seals[]` + `txIndex` + `logIndex` + `proofNodes[]` 를 BTIP19(BPrNTxEventProof)의 대응물로 정의. 현행 `MerkleProof{leaf,index,siblings}`(이진 Merkle)와 달리 MPT 노드 리스트 형태라 IBTIP21 구조 신설 필요. correlationId/nullifier 고정점을 무엇으로 잡을지(blockHash+txIndex+logIndex 후보) — BTIP-25 이슈와 연계.
- **BPuN→Besu 방향**: 현행 BPrN 체인코드(btip19/28/34/39 계열)를 Solidity로 옮기는 범위 산정. 이 방향은 검증 로직이 BPuN Tendermint 서명 검증이라 본 세션 산정과 무관(별도).
- 설계 OPEN 정리 원칙(design-before-code) 유지: 위 결정 전에는 코드 착수하지 않음.

---
