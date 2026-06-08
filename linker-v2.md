# Linker V2 Solidity 작업 컨텍스트

> 마지막 업데이트: 2026-06-01 — `audit-report-2026-05-28.md` 결과를 기준으로 실제 리포 상태(HEAD `31466e0`)와 재동기화. on-bprn 2026-05-26 구현 세션 산출물은 커밋 `f414756`/`31466e0`로 실제 반영됐음을 확정. 이전 "구현 세션" 항목들의 정합 상태는 §"2026-06-01 — 재동기화" 참조.

---

## 작업 개요

BTIP21, 23, 24에 정의된 Linker Protocol V2 온체인 컴포넌트를 Solidity 컨트랙트로 구현.
BTIP 문서 원본: `/Users/kylekwon/projects/beatoz/docs/BTIPS/`

---

## 프로젝트 구조

- **Solidity 컨트랙트**: `/Users/kylekwon/projects/beatoz/linker-v2/verifier/on-bpun/`
  - Hardhat 3.3.0, Solidity 0.8.28
  - 플러그인: `@nomicfoundation/hardhat-toolbox-viem`
  - 의존성: `@openzeppelin/contracts` (Ownable)
- **Prover API**: `/Users/kylekwon/projects/beatoz/linker-v2/prover-ts/` (2026-04-21 `prover/` → `prover-ts/`)
  - NestJS, TypeScript, `fabric-network` SDK, `protobufjs`

---

## 완료 작업 — Solidity 컨트랙트

### ✅ 공통 타입 및 인터페이스 정의

| 파일 | BTIP | 역할 |
|------|------|------|
| `contracts/interfaces/IBTIP21.sol` | 21 | LinkerEndpoint 인터페이스 + MerkleProof/TxEventProof 구조체 |
| `contracts/interfaces/IBTIP22.sol` | 22 | LinkerPolicy 인터페이스 (getRootCA/getEndorsementPolicy/getCRL) |
| `contracts/interfaces/IBTIP23.sol` | 23 | LinkerVerifier 인터페이스 |
| `contracts/interfaces/IBTIP24.sol` | 24 | LinkerNullifier 인터페이스 |
| `contracts/interfaces/IBTIP26.sol` | 26 | dApp 콜백 인터페이스 (handleLinkerEvent) |

**인터페이스 네이밍 규칙**: `IBTIP{번호}` (예: IBTIP21, IBTIP22, IBTIP23, IBTIP24, IBTIP26)

**구조체 위치**: `LinkerTypes.sol` 삭제 → `IBTIP21` 인터페이스 내부에 정의 (OZ 패턴)

### ✅ 구현체 (4개)

- `contracts/LinkerEndpoint.sol` (BTIP-21)
- `contracts/LinkerNullifier.sol` (BTIP-24)
- `contracts/LinkerVerifier.sol` (BTIP-23)
- `contracts/LinkerPolicy.sol` (BTIP-22)

### ✅ Nullifier 계산 위치 (2026-04-09 리팩토링)

- nullifier(`sha256(abi.encode(eventRootHash, targetDApp))`) 계산은 **`LinkerNullifier` 내부에서만** 수행
- `IBTIP24.markProcessed` 시그니처: `(bytes32 nullifier)` → `(bytes32 eventRootHash, address targetDApp)`
- `LinkerEndpoint`는 nullifier를 직접 계산하지 않음 — `(eventRootHash, targetDApp)` 그대로 전달

### ✅ cancelNullifier / cancelLinkerEvent (2026-04-09 추가)

- `LinkerNullifier.cancelNullifier(bytes32 eventRootHash)` — `msg.sender`를 `targetDApp`으로 사용하여 자신의 nullifier만 취소 가능
- `IBTIP26.cancelLinkerEvent(bytes32 eventRootHash)` — dApp 인터페이스에 대응 함수 추가
- `MockDApp`: `_nullifier` 저장 + `setNullifierContract(address)` + `cancelLinkerEvent` 구현
- `LinkerVerifier.EndorserInfo` → `SignerInfo` 로 개명

### ✅ IBTIP24 `markProcessed` 통합 (2026-04-21)

- 기존 `markProcessed(bytes32, address)` 삭제, `checkAndMark` 을 **`markProcessed returns (bool wasDup)`** 으로 네이밍 통일
- `AlreadyProcessed` 에러 제거 — 중복 시 `wasDup=true`로 반환, `LinkerEndpoint`가 `DuplicateProof`로 revert

### ✅ 배포 순서

1. `LinkerEndpoint()` — no args
2. `LinkerNullifier(endpointAddr)` — endpoint immutable
3. `LinkerVerifier()` — no args
4. `LinkerPolicy()` — no args
5. `MockDApp()` — no args
6. `endpoint.setNullifierContract(nullifierAddr)`
7. `endpoint.setVerifierContract(verifierAddr)`
8. `verifier.setPolicyContract(policyAddr)`
9. `mockDApp.setNullifierContract(nullifierAddr)`

---

## 2026-04-20 — BTIP 문서 변경사항 반영

### ✅ 네이밍/시그니처 통일 (BTIP16/19/21/26)

| 스펙 | Before | After |
|---|---|---|
| `MerkleProof` 구조체 필드 | `gindex` | `index` (0-based leaf position = gidx) |
| `TxEventProof` 필드 | `msp_ids` | `mspids` |
| `TxEventProof` 필드 | (없음) | `uint64 block_number` 추가 |
| `IBTIP26.handleLinkerEvent` 시그니처 | `(uint256 eventType, uint256[] gindices, bytes[] data)` | `(uint64 srcBlockNumber, uint64 srcTxIndex, uint256[] indices, bytes[] values)` |
| `IBTIP26` 추가 함수 | — | `cancelLinkerEvent(bytes32 eventRootHash)` |
| 블록 커밋 서명 입력 (BTIP-17) | `sha256(block_event_root)` | `sha256(block_height_8B_BE \|\| block_event_root)` |
| BTIP16 머클 트리 | 완전 패딩 | **null 기반 패딩** + `hashPair` 규칙 |

### ✅ 소스 식별 (BTIP26)

`handleLinkerEvent`는 `channelId`/`chaincodeId`/`selector`를 **별도 파라미터로 받지 않음**.
소스 식별 정보는 `indices`/`values` 쌍에 BPrN EventLog Header gidx로 포함됨 (channel_id=0, chaincode_id=1, selector=3).
신뢰가 필요한 경우 dApp이 해당 gidx 항목을 직접 확인해야 함.

### ✅ Solidity 측 merkle 검증 (null-aware)

`LinkerVerifier._computeMerkleRoot`:
```solidity
if (sib == bytes32(0)) {
    current = sha256(abi.encodePacked(current));   // hashPair(A, null) = sha256(A)
} else if (nodeIdx % 2 == 0) {
    current = sha256(abi.encodePacked(current, sib));
} else {
    current = sha256(abi.encodePacked(sib, current));
}
```
- `bytes32(0)` = **와이어 포맷의 null 센티넬** (BTIP16)
- 실제 해시가 `0x00…00`일 확률은 무시 가능하므로 sentinel로 재활용

### ✅ LinkerEndpoint — `_toLeafIndex` 헬퍼

```solidity
uint64 srcTxIndex = _toLeafIndex(payload.event_log_root_proof.index);
```
- BTIP16 convention 상 `MerkleProof.index`가 이미 0-based leaf 위치이므로 identity cast (uint64 narrowing)
- `_extractEventType` 등 구형 헬퍼 삭제

### ✅ MockDApp 업데이트

- `LinkerEvent` 구조체 필드: `srcBlockNumber` / `srcTxIndex` / `indices` / `values` / `encoded`
- `LinkerEventReceived(uint64 indexed srcBlockNumber, uint64 indexed srcTxIndex, uint256 count)` 이벤트
- `getEvent(uint256)` 반환값 5-tuple로 확장

---

## 완료 작업 — Prover API

### ✅ NestJS 프로젝트 구조

```
prover-ts/                            # 2026-04-21 `prover/` 에서 리네임
├── package.json
├── tsconfig.json / nest-cli.json
├── .env / .env.example
├── connection-profile.json          # 절대경로 기반 (prover-ts/ 디렉토리에 위치)
├── proto/
│   ├── common.proto                 # Block/Envelope/Payload/Metadata/Timestamp
│   ├── peer.proto                   # Transaction/Endorsement/ChaincodeEvent
│   ├── msp.proto                    # SerializedIdentity
│   └── endorser.proto               # Endorser gRPC service (미사용, 참고용)
└── src/
    ├── main.ts / app.module.ts
    ├── common/
    │   ├── merkle.ts                # SHA256 MerkleTree (null-aware)
    │   └── event-log.ts             # EventLog + ASN.1 DER 파서
    ├── fabric/
    │   ├── fabric.module.ts
    │   ├── fabric.service.ts        # fabric-network SDK (qscc.evaluateTransaction)
    │   └── block-parser.service.ts  # protobufjs 기반 블록 파싱
    └── prover/
        ├── prover.module.ts
        ├── prover.controller.ts     # GET /api/prove
        ├── prover.service.ts        # TxEventProof 생성 + 로컬 검증
        └── dto/
            ├── prove-request.dto.ts
            └── tx-event-proof.dto.ts
```

### ✅ API (2026-04-20 개정)

**`GET /api/prove?txId=...&eventIndices=4,5,6,7&channelId=bpn`**

- 쿼리 파라미터: `eventGindices` → **`eventIndices`** 로 rename (BTIP16 unification)
- Response: `TxEventProofDto` — Solidity `IBTIP21.TxEventProof` 구조와 1:1 대응
  - `mspids` (이전: `msp_ids`)
  - `block_number: number` **신규 필드**
  - MerkleProofDto: `index` (이전: `gindex`)

### ✅ gidx / index 정의 (BTIP-16)

- 프로토콜 용어 통일: **gindex → gidx (약칭), MerkleProof 필드 `index`**
- `gidx` = EventLog leaf array에서의 0-based leaf 위치
  - 0=channel_id, 1=chaincode_id, 2=tx_id, 3=selector, 4=elems[0], 5=elems[1], ...
- `leafCount + index` 같은 tree 내부 1-indexed node 위치가 **아님**

### ✅ block_event_root 구성 (BTIP-17, null 기반)

- 블록 내 **모든 tx**를 block 순서대로 leaf로 포함
  - 성공 tx (chaincode event 있음) → `event_log_root` (32B hash)
  - **실패/event 없는 tx → `null` 리프** (BTIP-17) — ZERO_HASH 아님!
    - `hashPair(A, null) = sha256(A)`, `hashPair(null, null) = null`
- leaf가 이미 hash이므로 tree 구성 시 `preHashed=true`
- `event_log_root_proof.index` = 블록 내 tx의 위치 (0-based)
- 파일 변경:
  - `MerkleTree.fromHashedLeaves(leaves: (Buffer | null)[])` — null 허용으로 확장
  - `prover.service.ts`: `blockEvtLeaves: (Buffer|null)[]`, 실패 tx에 `null` push

### ✅ 블록 커밋 서명 입력 (BTIP-17)

- **Sign input = `block_height (8B big-endian) || block_event_root (32B)`**
- `prover.service.ts::generateTxEventProof` 리턴값에 `block_number: number` 추가
- `getBlockNumber()` 반환 타입: `string` → `number` (Long/BigInt 처리 포함)
- `scripts/test-prove.ts::verifyBlockCommitSigs`도 `uint64BE(blockNumber) || blockEventRoot` 기준으로 재계산

### ✅ Siblings 직렬화 컨벤션 (null 센티넬)

- 내부 tree의 null sibling을 와이어로 보낼 때 `0x00…00 (32B)` 로 인코딩
- `MerkleProofDto.siblings: string[]` — `s ?? ZERO_HASH` 적용
- Solidity `LinkerVerifier._computeMerkleRoot`가 `bytes32(0)`을 null로 해석

### ✅ 로컬 proof 검증 (null-aware)

`scripts/test-prove.ts::verifyMerkleProof`:
```ts
if (sibBuf.equals(ZERO32)) {
  current = sha256(current);            // hashPair(A, null)
} else if (idx % 2 === 0) {
  current = hashPair(current, sibBuf);
} else {
  current = hashPair(sibBuf, current);
}
```

### ✅ Fabric 연결

- `fabric-network` v2.2.20 SDK 사용 (`@hyperledger/fabric-gateway` 아님 — BPrN은 구버전 Fabric)
- `qscc.evaluateTransaction('GetBlockByTxID', channelId, txId)` → raw Block proto bytes
- `BlockParserService.decodeBlock()` 으로 protobufjs 디코딩
- 환경설정: `.env` + `connection-profile.json` (절대경로)

### ✅ verifier 스크립트 목록 (`scripts/beatoz/`)

| 스크립트 | 용도 |
|----------|------|
| `deploy.ts <chainAlias>` | 전체 컨트랙트 배포 + wire-up |
| `set-policy.ts <chainAlias>` | LinkerPolicy에 RootCA/EndorsementPolicy 설정 |
| `setup-localnet0.sh` | deploy + set-policy 한 번에 실행 |
| `submit-proof.ts <chainAlias> <dappAlias>` | TxEventProof 제출 + `LinkerProofReceived` 이벤트 파싱/출력 (MockDApp storage 제거 후 대체) |
| `cancel-event.ts <chainAlias> <dappAlias> <eventRootHash>` | nullifier 취소 |

> `query-dapp.ts`는 2026-04-20 MockDApp storage 제거와 함께 삭제됨 (더 이상 조회 대상 없음).

- dApp 주소는 모두 `<dappAlias>` 로 받아 `.net/deployed.<chainAlias>.<dappAlias>.json` 에서 로드
- `utils.ts`: `deliver_tx.log`의 `reason: <hex>` 패턴에서 hex를 추출해 keccak256 selector로 커스텀 에러 디코딩
  - BEATOZ는 revert data를 `deliver_tx.log`에 `reason: <HEX (no 0x)>` 형식으로 기록

---

## 미완료 (2026-06-01 재정리)

> 자세한 현황과 새로 발견된 갭은 §"2026-06-01 — 재동기화" 참조.

- **BPuN→BPrN 이벤트 Prover 미구현** (end-to-end의 결정적 미싱 피스). `prover-ts/src/prover/u2r/` 디렉토리 자체 부재 — 감사 시점의 placeholder도 사라짐.
- **2PC 미구현** — 설계는 `btips`/`btips-2pc-design`에 완결돼 있으나 코드 미착수. 누락 항목: on-bpun(`IBTIP21.onResult`, `LinkerResult` 이벤트, `handleLinkerResult`, `try/catch`, `nonReentrant`, `MIN_CALLBACK_GAS`, `cancelLinkerEvent` IBTIP26 갱신), on-bprn(`OnResult`, `HandleLinkerResult`, `LinkerResultElems`).
- **Prover multi-peer block-commit-sig 수집** — 여전히 단일 블록 조회(피어 1개 sig). BTIP-17 검증 강도를 끌어올리려면 각 endorser 피어에게 개별 블록 조회 필요.
- **on-bpun 정책 엔진 문서화** — `LinkerPolicyVerifier.sol` 외 6개 파일(아래 §"2026-06-01 — 재동기화" 1.B)이 코드에 존재하나 본 컨텍스트 문서엔 기재 부족. BTIP-22 구현 형태(`signatureRuleTree`, `implicitMetaPolicy`, Fabric channel config 기반)로 확장됐음을 인터페이스/스펙 차원에서 정리 필요.
- **`linker-v2` 리포 컴파일 검증** — 샌드박스에서 Go(`go build`)·Hardhat·Foundry 모두 미수행. 로컬에서 `cd verifier/on-bprn && go mod vendor && go build ./...` 및 `cd verifier/on-bpun && npx hardhat compile`(또는 `forge build`) 확인 필요.
- `Counter.sol` 샘플 파일 삭제 (hardhat init 생성물) — 미확인.

---

## 컨트랙트 의존성

```
LinkerEndpoint (BTIP21)
  ├── IBTIP24 → LinkerNullifier (BTIP24)
  ├── IBTIP23 → LinkerVerifier (BTIP23)
  │                └── IBTIP22 → LinkerPolicy (BTIP22)
  └── IBTIP26 → dApp (BTIP26)
```

---

## 기술 참고

### BEATOZ EVM Precompiles
- `0x0100` — P-256 ECDSA 검증
- `0xff00` — X.509 인증서 체인 검증 (입력: [subject, issuer] 순서, 반환: [32B pubkey.x][32B pubkey.y][32B serialNumber][OU bytes])

### Solidity 구현 시 주의사항
- `memory bytes`에 대한 slice 접근 불가 → inline assembly로 디코딩
- 해시 함수: `sha256` 사용 (keccak256 아님)
- CRL: `abi.decode(crl, (bytes32[]))` 형식으로 ABI 인코딩된 serial number 배열
- Endorsement Policy: `abi.decode(policy, (uint256, bytes[]))` → (minEndorsers, requiredOUs)

### Fabric / BPrN
- BPrN 연결 프로파일: `/Users/kylekwon/go/src/github.com/kysee/zk-chains/provers/bprn/localchannel0/connection-profile.json`
- Block commit signature: `block.metadata[5]` (BTIP-17, BPrN 코어 확장)
- 서명 입력: `block_height (8B BE) || block_event_root (32B)` — BPrN 피어가 이 값에 P-256 서명
- `fabric-network` SDK는 v2.x 사용 (v2.4+ gateway.Gateway API 아님)
- qscc 시스템 체인코드로 블록 조회: `GetBlockByTxID`, `GetBlockByNumber`
- EventLog DER: `onlyElems=true` 모드 → ASN.1 SEQUENCE of OCTET STRING

### BTIP16 Merkle Tree (Appendix)
- null 기반 패딩: leaf 수가 2의 거듭제곱이 아니면 빈 자리를 `null`로 채움
- `hashPair` 규칙:
  - `hashPair(A, B) = sha256(A || B)`
  - `hashPair(A, null) = sha256(A)`
  - `hashPair(null, null) = null`
- 와이어 포맷: Solidity `bytes32`에는 null을 담을 수 없어 `bytes32(0)`을 null 센티넬로 사용
- Solidity 검증기가 `sib == bytes32(0)` 분기로 null을 해석

---

## 최근 작업 이력

### 2026-04-21 — IBTIP24 정리 / 디렉토리 리네임 / BTIP-16 bytes 반영

**IBTIP24 API 정리**:
- 기존 `markProcessed(bytes32, address)` 삭제, `checkAndMark` → **`markProcessed(bytes32, address) returns (bool wasDup)`** 으로 네이밍 통일 (atomic check+mark 의미 유지)
- `AlreadyProcessed` 에러 제거 — 중복 시 `wasDup=true` 리턴으로 대체, `LinkerEndpoint`가 `DuplicateProof`로 revert
- **변경 파일**: `contracts/interfaces/IBTIP24.sol`, `contracts/LinkerNullifier.sol`, `contracts/LinkerEndpoint.sol`, `scripts/beatoz/utils.ts` (selector 테이블에서 `AlreadyProcessed` 엔트리 제거)

**Prover 디렉토리 리네임 `prover/` → `prover-ts/`**:
- `git mv prover prover-ts` — 히스토리 보존
- `.env`의 `FABRIC_CONNECTION_PROFILE` 절대경로도 `prover-ts/connection-profile.json`로 갱신
- NestJS 내부 `ProverService`/`ProverModule`/`./prover/prover.module` 같은 모듈 식별자는 그대로 유지 (디렉토리 리네임과 무관)

**Prover 스크립트 리네임 + 출력 순서 조정**:
- `prover-ts/scripts/test-prove.ts` → `test-proof.ts`
- `TxEventProof` JSON 출력을 **검증 완료 후 마지막**으로 이동 (이전: 요청 직후)
- 사용되지 않던 `derToRS` 함수 삭제

**BTIP-16 — `EventLog.tx_id` / `selector` bytes 반영**:
- `prover-ts/src/common/event-log.ts`:
  - `txId: string` → **`txId: Buffer`** (BTIP-16 spec: `bytes tx_id = 3;`)
  - Constructor에 `selector: Buffer` 파라미터 추가 → **`(channelId, chaincodeId, txId, selector)`** 4-인자 생성
  - `selector`를 `readonly`로 변경 (사후 할당 제거, 생성 시점에 확정)
  - `leaves()`에서 `Buffer.from(this.txId)` → `this.txId` (이미 Buffer)
- `prover-ts/src/prover/prover.service.ts`:
  - Fabric 응답의 `parsed.txId`(hex sha256 문자열) → `Buffer.from(..., 'hex')` 로 디코드
  - Fabric 응답의 `parsed.chaincodeEvent.eventName`(hex selector 문자열) → `Buffer.from(..., 'hex')` 로 디코드 후 `selector`로 전달
  - 근거: BPrN 체인코드는 Fabric의 `event_name` 필드에 sha256 selector(32B)의 **hex 인코딩 문자열**을 실어 보냄

**검증**:
- `npx hardhat compile`: 4 Solidity files 성공 (solc 0.8.28, cancun)
- `npx tsc --noEmit` (prover-ts): EXIT=0

### 2026-04-20 — BTIP 문서 변경사항 반영
- **contracts/interfaces/IBTIP21.sol**: `MerkleProof.gindex → index`, `TxEventProof.msp_ids → mspids`, `block_number: uint64` 추가
- **contracts/interfaces/IBTIP26.sol**: `handleLinkerEvent` 시그니처 변경 `(srcBlockNumber, srcTxIndex, indices, values)`
- **contracts/LinkerEndpoint.sol**: `_extractEventType` 제거, `_toLeafIndex` 헬퍼 추가, `_deliverEvents`에서 `srcBlockNumber`/`srcTxIndex` 전달
- **contracts/LinkerVerifier.sol**: `payload.mspids` 사용, `sha256(block_height || block_event_root)` 서명 입력, `.index` 필드, null-aware merkle sibling 처리 (`bytes32(0)` 분기)
- **contracts/MockDApp.sol**: 새 `handleLinkerEvent` 시그니처 맞춰 `LinkerEvent` 구조체/이벤트/`getEvent` 반환값 갱신
- **scripts/beatoz/submit-proof.ts**: `mspids`/`block_number`/`index` 반영, 이벤트 leaf index를 0-based로 정규화 (`4,5,6,7` → `0,1,2,3`)
- **scripts/beatoz/query-dapp.ts**: `printEvent`를 `srcBlockNumber`/`srcTxIndex`/`indices`/`values`/`encoded` 반환 포맷에 맞게 업데이트
- **prover/src/prover/dto/tx-event-proof.dto.ts**: `mspids`/`block_number`/`index` 필드
- **prover/src/prover/dto/prove-request.dto.ts**: `eventGindices → eventIndices`
- **prover/src/prover/prover.controller.ts**: 파라미터명 반영
- **prover/src/prover/prover.service.ts**: DTO 필드명 반영, `getBlockNumber()` → number 반환, **실패 tx 리프를 `null`로 push (BTIP-17)**, `ZERO_HASH` 주석을 "on-wire null sentinel for siblings"로 수정
- **prover/src/common/merkle.ts**: `MerkleTree.fromHashedLeaves`가 `(Buffer|null)[]` 수용, `proof()` 반환 필드 `gindex → index`, `verifyMerkleProof` 파라미터명 변경
- **prover/src/common/event-log.ts**: `proof()` 반환 필드 `gindex → index`
- **prover/scripts/test-prove.ts**: 쿼리 `eventIndices`, `proof.mspids`/`proof.index` 사용, `verifyBlockCommitSigs`는 `uint64BE(blockNumber) || blockEventRoot`에 대한 서명 검증, `verifyMerkleProof`에 `bytes32(0) → sha256(current)` null-aware 분기 추가
- **검증**: prover `npx tsc --noEmit` EXIT=0 통과

### 2026-04-09 (2차)
- Nullifier 계산을 `LinkerNullifier` 내부로 이동
- `cancelNullifier` / `cancelLinkerEvent` 추가
- `SignerInfo` 개명

### 2026-04-08 (1차)
- 초기 계약 구현 + NestJS Prover API 스캐폴딩

---

## 2026-04-20 (4차) — Gas 최적화 (1.77M → ~420K, -76%)

### 배경

`submit-proof.ts` 실행 시 BEATOZ localnet에서 측정한 gas 사용량이 **1,767,144 gas** — cross-chain proof 제출 치고는 높아 보여서 프로파일링 및 최적화 진행.

최초 추측(precompile이 주범)은 **틀림**. beatoz-go의 실제 precompile 비용:
- `P256Verify` (0x0100): **6,900 gas** (EIP-7212)
- `X509Verify` (0xff00): **50,000 + 50,000 × n(cert)** = 100K/1cert
- 합계: **~107K gas (전체의 6%)**

나머지 **~1.66M은 전적으로 Solidity 실행 비용**이었음 → 컨트랙트 차원 최적화가 필요.

### Gas 프로파일링 환경 구축 (Foundry)

BEATOZ가 Ethereum JSON-RPC(`eth_call`, `eth_getCode`)나 `debug_traceTransaction`을 구현하지 않기 때문에 **Forge의 로컬 revm 환경**에서 mock precompile로 Solidity 실행 gas를 측정.

**추가된 파일**:
- `verifier/on-bpun/foundry.toml` — src/test 경로, solc 0.8.28, cancun EVM
- `verifier/on-bpun/remappings.txt` — `@openzeppelin`, `forge-std` → `node_modules/`
- `verifier/on-bpun/test-forge/LinkerGasTest.t.sol` — 컨트랙트 배포 + 정책 설정 + onProof 호출
- `verifier/on-bpun/test-forge/mocks/MockP256Verify.sol` — `fallback` 에서 `true32Byte` 반환
- `verifier/on-bpun/test-forge/mocks/MockX509Verify.sol` — `[pubkeyX][pubkeyY][serialNo]["peer"]` 반환
- `.gitignore` 에 `/out-forge`, `/cache-forge`, `/broadcast` 추가

**사용법**:
```bash
# foundry 설치 후
forge test --match-contract LinkerGasTest --gas-report
forge test -vvvv   # 호출 트리 + per-call gas
```

**주의**: revm에는 BEATOZ 커스텀 precompile이 없어 `vm.etch()`로 mock 바이트코드를 해당 주소에 심어서 `staticcall` 이 성공하게 함. Mock precompile의 gas 비용은 실제와 다르지만 Solidity 측 실행 비용은 정확히 측정됨.

### 적용한 최적화 (컨트랙트)

#### 1. `_encodeCert` 리팩토링 (🏆 가장 큰 효과 — 단일 변경으로 -590K gas)

**Before**: [LinkerVerifier.sol:157-171] byte-by-byte 루프로 cert(529B) + rootCA(~560B)를 memory buffer에 복사. calldata→memory 암묵적 복사까지 두 번 일어남.
```solidity
for (uint256 i = 0; i < len; i++) {
    buf[offset + 4 + i] = cert[i];  // MSTORE8 × len
}
```

**After**: 두 개의 helper로 분리 + assembly
```solidity
function _encodeCertCalldata(bytes memory buf, uint256 offset, bytes calldata cert) ...
    // calldatacopy로 한 번에 복사 (O(1))
function _encodeCertMemory(bytes memory buf, uint256 offset, bytes memory cert) ...
    // mcopy로 한 번에 복사 (EIP-5656, Cancun)
```

**OU 복사 루프도 `mcopy` 로 교체** (같은 패턴이 149-151 라인에 있었음).

#### 2. A: `checkAndMark` 결합 (`LinkerNullifier` + `LinkerEndpoint`)

- `IBTIP24.checkAndMark(bytes32, address) returns (bool wasDup)` 추가
- `LinkerEndpoint.onProof`에서 `isProcessed` + `markProcessed` 2회 외부 호출 → 1회로 통합
- verify 실패 시 state change는 tx revert로 자동 롤백되므로 순서를 바꿔도 안전
- 절감: sha256 1회, 외부 호출 1회, 중복 SLOAD 1회

#### 3. B: `_checkEndorsementPolicy` 해시 캐싱

- Nested loop에서 매 반복마다 `keccak256(endorsers[j].ou)` 와 `keccak256(requiredOUs[i])` 재계산 하던 것을 루프 밖에서 1회 계산 후 캐시
- endorser 수 N, required OU 수 M → 재계산 O(N×M) → O(N+M)

#### 4. C: `getEndorsementPolicy` typed return (`IBTIP22`)

- Before: `returns (bytes memory)` 후 caller에서 `abi.decode` — 라운드트립 발생
- After: `returns (uint256 minEndorsers, bytes[] memory requiredOUs)` 직접 반환
- 절감: `abi.encode` + `abi.decode` ~3-5K gas

#### 5. D: `getRootCAAndCRL` 결합 getter (`IBTIP22`)

- `getRootCA` + `getCRL` 2회 external call → 1회로 통합
- `LinkerVerifier._verifyCertChains` 에서 per-endorser external call 1회 감소

#### 6. E: P256 input 버퍼 재사용 (`_verifyCommitSignatures`)

- Before: `bytes memory input = abi.encodePacked(...)` 를 매 endorser마다 재할당 (160B)
- After: loop 밖에서 160B 버퍼 1회 할당, `msgHash` 미리 쓰기, loop 안에서 `calldatacopy` + `mstore`로 `[32:96]`, `[96:128]`, `[128:160]` 만 갱신
- endorser 수에 선형 비례하는 절감

### MockDApp 완전 재설계 (-700K gas 추가 절감)

trace로 발견: `handleLinkerEvent` 하나가 전체 `onProof` gas의 **~82%(766K)** 를 차지. 원인은 `_events.push(LinkerEvent{...})` 에서 대형 struct를 storage에 쓰는 것(특히 `encoded` bytes ~500B → 20K gas × ~16 slots).

**최종 형태 (storage 없이 event emit만)**:
```solidity
event LinkerProofReceived(
    uint64 indexed srcBlockNumber,
    uint64 indexed srcTxIndex,
    uint256[] indices,     // non-indexed
    bytes[] values         // non-indexed
);

function handleLinkerEvent(...) external override {
    emit LinkerProofReceived(srcBlockNumber, srcTxIndex, indices, values);
}
```

**제거된 것**:
- `struct LinkerEvent`, `_events[]` storage array
- `eventCount()`, `getEvent()`, `getEncoded()`, `getAllEncoded()`
- 구 event 이름 `LinkerEventReceived` → `LinkerProofReceived`로 변경

**근거**: Event data 비용은 **8 gas/byte (non-indexed)** 로 storage SSTORE(20K/slot) 대비 **수백 배 싸다**. On-chain 읽기가 필요 없는 데이터는 항상 event가 맞다.

**Indexed 주의사항**: 32바이트 초과 가변 데이터(`bytes`, `string`)를 indexed로 선언하면 topic에 keccak256 해시만 저장되고 **원본은 소실**. `indices`/`values` 같은 페이로드는 반드시 non-indexed여야 함.

### submit-proof.ts 이벤트 파싱 로직 추가

BEATOZ는 각 EVM 로그를 Cosmos 이벤트로 래핑하며 `/tx` RPC 응답에서는 **키/값이 모두 base64**로 인코딩됨:
- `type = "evm"`
- attributes: `contractAddress`, `topic.0`, `topic.1`, ..., `data`, `blockNumber`, `removed`
- 값 디코드 예: `dG9waWMuMA==` → `topic.0`, `0x...` hex는 대문자 + `0x` prefix 없이 저장

**추가된 helper 함수**:
- `b64decode(s)` — base64 패턴이면 디코드, 아니면 pass-through (SDK가 이미 디코드한 경우도 처리)
- `to0xHex(s)` — `0x` prefix 보장, 소문자화
- `eventSignatureHash(abi)` — `keccak256("Name(type1,type2,...)")`
- `decodeLinkerProofReceived(events, abi)` — type="evm" 중 topic[0]이 일치하는 로그 찾아 `web3.beatoz.abi.decodeLog` 로 디코드
- `numericValues(x)` — web3.js의 `{0:..., 1:..., __length__: n}` 형태에서 숫자 키만 추출

**출력 예시**:
```
--- LinkerProofReceived ---
  srcBlockNumber: 3
  srcTxIndex:     0
  indices:        [4, 5, 6, 7]
  values:
    [0] idx=4  0x0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a0a
    [1] idx=5  0x0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b0b
    [2] idx=6  0x3039
    [3] idx=7  0x49742069732061205472616e736665724576656e744c6f67
```

(`[3]`은 ASCII "It is a TransferEventLog"의 hex)

### `query-dapp.ts` 제거

MockDApp에서 storage를 제거했으므로 `getEvent` / `getEncoded` 등 불가능. 대신 `submit-proof.ts` 내부에서 tx.deliver_tx.events 파싱하여 출력.

### 인터페이스 변경 요약

| 파일 | 변경 |
|------|------|
| `IBTIP22.sol` | `getEndorsementPolicy()` 반환 타입 `bytes` → `(uint256, bytes[])`; `getRootCAAndCRL(mspId)` 결합 getter 추가 |
| `IBTIP24.sol` | `checkAndMark(bytes32, address) returns (bool wasDup)` 추가 (기존 `isProcessed`/`markProcessed` 유지) |
| `MockDApp.sol` | 이벤트 이름 `LinkerEventReceived` → `LinkerProofReceived`, `indices`/`values` non-indexed로 추가, `LinkerEvent` struct 및 storage 완전 제거 |
| `LinkerEndpoint.sol` | `checkAndMark` 사용으로 flow 단순화 |
| `LinkerVerifier.sol` | `_encodeCertCalldata` / `_encodeCertMemory` 분리, OU/정책 해시 캐싱, P256 버퍼 재사용 |

### 최종 Gas 측정

| 단계 | BEATOZ 실측 | Forge (revm) | 주요 변경 |
|------|------------|-------------|-----------|
| 원본 | **1,767,144** | 1,360,249 | — |
| #1 (calldatacopy/mcopy) + MockDApp struct 원본 | **1,177,614** | 985,493 | `_encodeCert` assembly |
| A~E 전체 + MockDApp event-only | **~420,000** | 226,323 → 197,760 | 전체 최적화 스택 적용 |

**절감: 1,347,144 gas (-76.2%)**

Forge vs BEATOZ 차이 (~190K) 내역:
- Intrinsic (21K) + Calldata (~45K) + 실제 precompile(~107K, mock 대비) + Cosmos-layer 오버헤드(~20K)

### 비교 (타 cross-chain 프로토콜)

| 프로토콜 | Verification gas |
|---------|-----------------|
| Tendermint Light Client (직접) | >10M |
| ENS DNSSEC-Oracle (구 X.509) | ~2M |
| **Linker V2 (최적화 전)** | **1.77M** |
| Wormhole (20 guardian) | ~1.35M |
| **Linker V2 (최적화 후)** | **~420K** ✅ |
| Axelar GMP | ~700K |
| ZK Groth16 verifier | ~500K |
| Optimism fraud proof | ~40K (오프체인 의존) |

### 파일 변경 목록

**컨트랙트** (`verifier/on-bpun/contracts/`):
- `LinkerEndpoint.sol` — `checkAndMark` 사용
- `LinkerNullifier.sol` — `checkAndMark` 추가
- `LinkerVerifier.sol` — `_encodeCert` 분리/assembly, 해시 캐싱, P256 버퍼 재사용
- `LinkerPolicy.sol` — `getEndorsementPolicy` 반환 타입, `getRootCAAndCRL` 추가
- `MockDApp.sol` — storage/struct 제거, `LinkerProofReceived` 이벤트
- `interfaces/IBTIP22.sol` — typed 반환, combined getter
- `interfaces/IBTIP24.sol` — `checkAndMark`

**스크립트** (`verifier/on-bpun/scripts/beatoz/`):
- `submit-proof.ts` — gas 로그 + 이벤트 파싱/출력 추가
- `query-dapp.ts` — 삭제 (MockDApp 대응 메서드 없음)

**Foundry 환경** (신규):
- `verifier/on-bpun/foundry.toml`
- `verifier/on-bpun/remappings.txt`
- `verifier/on-bpun/test-forge/LinkerGasTest.t.sol`
- `verifier/on-bpun/test-forge/mocks/MockP256Verify.sol`
- `verifier/on-bpun/test-forge/mocks/MockX509Verify.sol`

**기타**:
- `.gitignore` — `/out-forge`, `/cache-forge`, `/broadcast`

### 향후 추가 최적화 여지

1. **Calldata 감소** — proof payload(~3KB) 자체가 calldata 비용 ~45K gas 소비. cert를 on-chain 저장/캐시하거나 ZK proof로 대체 시 큰 감소
2. **X509 precompile 호출 캐싱** — 같은 endorser cert를 반복 검증 중이라면 pubkey/OU를 저장해 skip
3. **Event 추가 슬림화** — `LinkerProofReceived`의 `values` 가 대형이면 외부 indexer에 위임하고 최소 필드만 emit

### 관련 참고 사항 (Key Insights)

- **BEATOZ는 Ethereum JSON-RPC 미구현** → `forge --fork-url` 불가
- **BEATOZ는 `debug_traceTransaction` 미구현** → 실 노드에서 per-call gas breakdown 불가
- **Hardhat v3 + `hardhat-gas-reporter` 비호환** (v3가 Mocha 대신 `node:test` 사용)
- **macOS에서 forge 설치 시 `libusb` 필수** (`brew install libusb`) — USB wallet 미사용이어도 dylib 로드 실패
- **`mcopy` opcode (EIP-5656)** 는 Cancun부터 지원, solc 0.8.24+ 에서 기본 활성화, BEATOZ도 지원 확인(`PrecompiledContractsCancun`에 등록됨)
- **Solidity 반환값은 `calldata` 불가** — 외부 호출의 returndata는 언어/EVM 레벨에서 반드시 memory로 복사됨
- **Event 비용**: 기본 375 gas + topic당 375 gas + 바이트당 8 gas; storage 대비 ~80배 저렴
- **Indexed 대형 데이터 금기**: `bytes`/`string`/struct을 indexed로 하면 keccak 해시만 저장되고 원본 소실

---

## 2026-05-26 — BTIP39 인터페이스 추가 및 프로젝트 정비

### ✅ `.gitignore` 생성

프로젝트 루트에 `.gitignore` 추가. JetBrains IDE(`.idea/`), Node.js(`node_modules/`), Go 바이너리, OS 파일(`.DS_Store`), 환경변수(`.env`), TypeScript(`*.tsbuildinfo`), VS Code(`.vscode/`) 포함.

### ✅ BTIP39 인터페이스 파일 (`verifier/on-bprn/types/ibtip39.go`)

BTIP39(BPuN Validator Set Update Proof) 스펙 기반으로 인터페이스 및 타입 정의 추가.

| 타입/인터페이스 | 역할 |
|----------------|------|
| `SimpleValidatorEntry` | 신규 Validator Set 항목 — `PubKey` (secp256k1 33B compressed) + `VotingPower` (int64). ValidatorsHash 계산 입력 |
| `ValidatorSetProofPayload` | trustless Prover가 제출하는 Validator Set 변경 증명 페이로드. 기존 `RFC6962Proof`, `ValidatorSignature` 재사용 |
| `IBTIP39` | `UpdateValidatorSet(ctx, *ValidatorSetProofPayload) error` — permissionless 증명 제출 진입점 |

- `Round`(`int32`), `BlockPartsTotal`(`uint32`) 필드 타입은 기존 `BPuNTxEventProofPayload`와 일치
- `go build ./types/` 컴파일 확인 완료

### ✅ 코드 주석 영문화

- `verifier/on-bprn/types/ibtip32.go` — 모든 한국어 주석을 영어로 변환
- `verifier/on-bprn/types/ibtip39.go` — 영어 주석으로 작성

**인터페이스 네이밍 규칙 (BPrN Go)**: `IBTIP{번호}` (예: `IBTIP32`, `IBTIP39`) — BPuN Solidity 인터페이스와 동일한 패턴

---

## 2026-05-26 (구현 세션) — BTIP-37/39 구현 + secp256k1 + 레지스트리 리팩토링 + tendermint 라이브러리 정합

> 위 2026-05-26 항목(인터페이스 추가/주석 영문화) 이후 같은 날 진행한 본격 구현 세션. on-bprn(Go)·on-bpun(Solidity)·prover-ts 전반.

### ✅ BTIP-37 LinkerRegistry 구현 (신규)

**on-bpun (Solidity)**: `contracts/interfaces/IBTIP37.sol` + `contracts/LinkerRegistry.sol`
- `(bytes32 chainId, bytes32 role) → address`, `getContract`/`setContract`, `ContractRegistered` 이벤트, `UnknownContract`/`Unauthorized` 에러
- Role 상수 4개(`keccak256`), 접근제어는 OZ `Ownable` + TODO 주석 (CREATE2/multisig/timelock는 운영 이슈로 보류)

**on-bprn (Go)**: `types/ibtip37.go` + `linker-registry/main.go`
- 저장 값 = **체인코드 이름(string)**. Fabric은 체인코드를 `(channelID, chaincodeID)` 문자열로 식별하고 `InvokeChaincode`/`SignedProposal` 모두 이름 기반. BTIP-9의 20B 주소는 단방향 해시라 이름으로 역산 불가 → 이름 저장.
- 키 `(channelID, role)` 모두 string, X.509 admin(`EnsureAdmin`)
- BTIP-37 문서 부록도 이에 맞춰 HLF 네이티브 string으로 개정(btips.md 참조)

### ✅ 체인코드 간 호출을 레지스트리 기반으로 통합

- 개별 setter 전부 제거: `SetLinkerPolicyID`(verifier), `SetLinkerEndpointID`(nullifier), `Set{Verifier,Nullifier}ChaincodeID`(endpoint), dapp의 `Set{Endpoint,Nullifier}ID`
- 단일 부트스트랩 포인터 `SetRegistryID`(레지스트리 체인코드 이름)만 유지
- 공유 헬퍼 `types/registry.go`: `RegistryIDKey`, `SetRegistryID`/`GetRegistryID`, `ResolveContract(role)` — `registry.GetContract(GetChannelID(), role)` invoke 후 이름 반환
- 적용: endpoint→VERIFIER/NULLIFIER, verifier→POLICY, nullifier→ENDPOINT(호출자 인증), dapp→ENDPOINT/NULLIFIER

### ✅ secp256k1 교정 (ed25519 제거)

- linker-policy: pubkey 길이 33B 검증
- linker-verifier: `crypto/ed25519` 제거 → 포크 `tmsecp256k1.PubKey(pub).VerifySignature(signBytes, sig)` (내부 sha256, **pre-hash 금지**)
- 검증 스킴 근거(beatoz-go `types/crypto/crypto.go`): `ethcrypto.VerifySignature(pub33, sha256(signBytes), sig[:64])`, 주소 `tmsecp256k1.PubKey(pub).Address()`(ethereum-style keccak). `DefaultHash = sha256` 확정.

### ✅ BTIP-39 UpdateValidatorSet 구현 (linker-policy)

- 스텁(panic) → 5단계 검증: 페이로드 무결성(target==trusted+1, proof.index==8) → trusted_set 조회 → commit 서명 2/3+ → RFC6962 헤더 증명(index 8) → ValidatorsHash 재계산 일치 → self-call 저장(permissionless, 권한 우회). 동일 target 동일 해시 멱등 no-op.
- 공유 검증 헬퍼를 `types/tmverify.go`로 이동/신설: `VerifyCommitSignatures`, `VerifyRFC6962`, `ComputeValidatorsHash`, `DecodeAminoBytes`, `ValidatorAddressFromPubKey`

### ✅ tendermint 로직 직접 호출 (직접 구현 금지 원칙)

해시·인코딩·디코딩 등 tendermint 값과 바이트 일치해야 하는 것은 손수 구현하지 않고 tendermint/gogo 라이브러리를 직접 호출:
- ValidatorsHash: `(&tmtypes.ValidatorSet{Validators: vals}).Hash()` — `NewValidatorSet` 회피(주소 정렬·proposer priority 부작용), prover 원본 순서 보존
- RFC6962 증명: `crypto/merkle` `merkle.Proof{...}.Verify` + `HashFromByteSlices`
- cdcEncode된 헤더 bytes 필드 디코드: `gogotypes.BytesValue{}.Unmarshal` (손수 varint/protowire 파싱 폐기)
- tx_result(ResponseDeliverTx) 디코드: `abci.ResponseDeliverTx{}.Unmarshal` (손수 필드 순회 폐기)
- **버그 수정**: 기존 `DecodeAminoBytes`가 `varint(len)||bytes`로 가정 → 실제는 proto `BytesValue`(`0a 20 ...`) → 실증명 제출 시 `amino length prefix does not match payload` 발생. `gogotypes.BytesValue.Unmarshal`로 교정 (BTIP-28 LastResultsHash 경로도 동일)

### ✅ 커스텀 ValidatorSet/Validator 제거 → tmproto.ValidatorSet 사용

- `types.ValidatorSet`/`types.Validator`(커스텀 도메인 타입) 삭제
- 크립토 내부 표현 = `tmproto.ValidatorSet`(`proto/tendermint/types`). 직렬화/저장/외부 경계 = hex DTO
- 이유: tendermint 타입 중복 정의 안 함. tmproto.ValidatorSet엔 `Height` 없음 + `PubKey`가 `crypto.PublicKey` oneof라 `encoding/json` 왕복 불가 → 경계는 DTO, 크립토는 tmproto

### ✅ BTIP-32 외부 인터페이스 hex 문자열화

- `types/validatorset_dto.go`: `ValidatorSetDTO`/`ValidatorDTO`(address/pub_key hex), `HexToBytes`(0x 허용)/`BytesToHex`, `ToProto()`(DTO→tmproto, secp256k1 oneof), `Entries()`(→SimpleValidatorEntry)
- IBTIP32: `GetValidatorSet`→`*ValidatorSetDTO`, `SetValidatorSet(*ValidatorSetDTO)`, `GetValidator(int64, string hex)`→`*ValidatorDTO`
- 이유: 외부 호출 시 base64 인코딩/디코딩 번거로움 제거. 저장은 DTO JSON, 크립토는 tmproto로 변환
- linker-verifier.`fetchValidatorSet`: 정책 응답(hex DTO) → `ToProto()`

### ✅ contractapi 파라미터 스키마 — total_power optional

- `SetValidatorSet` 호출 시 `total_power is required` 에러 → contractapi가 구조체 필드를 전부 required로 검증
- `ValidatorSetDTO.TotalPower`에 `metadata:",optional"` 태그 추가(재계산되는 값이라 입력 불요). 함수 시그니처는 구조체 유지(string으로 바꿨다가 되돌림)

### ✅ BTIP-39 Prover (`prover-ts/src/prover/btip39`, 신규)

BPuN RPC(Tendermint, 26657)로 블록 1~latest 스캔 → Validator Set 변경 탐지 → 각 변경에 `ValidatorSetProofPayload` 생성:
- `rpc.ts`(/status,/block,/commit,/validators), `header-merkle.ts`(v0.34 14필드 cdcEncode + RFC6962 + index8 증명), `validator-set-proof.ts`(payload 빌더 + SimpleValidator 인코딩으로 ValidatorsHash 재계산), `types.ts`(DTO, base64 byte 필드, `bigint` timestamp + `goJsonStringify`), `main.ts`(스캔 엔트리)
- **self-검증**: 재구성 헤더 RFC6962 루트 == 노드 `block_hash`, 재계산 ValidatorsHash == 헤더 `next_validators_hash`. 둘 다 통과해야 출력. (TS는 Go 포크 import 불가 → 재구현하되 노드 값으로 교차검증 — 유일하게 허용되는 재구현)
- 탐지 경계: 1..latest 전체 탐지, target 미커밋이면 "pending tip" 로깅. distinct ValidatorsHash 진단.
- `npm run btip39:scan` (`OUT_DIR`로 파일 저장)
- 검증: 실제 블록 2331 데이터로 헤더 인코딩 오프라인 검증 — 재구성 루트 == 노드 `block_hash`(8019698E…) 일치. `npx tsc --noEmit` 통과

### 배포/부트스트랩 순서 (on-bprn)

1. `linker-registry` 배포 → 각 역할 `SetContract(channelID, "LinkerEndpoint|Verifier|Policy|Nullifier", <cc 이름>)`
2. 각 컴포넌트(+dapp) `SetRegistryID(<registry cc 이름>)` (linker-policy는 불요 — 다른 체인코드를 호출하지 않음)
3. `SetValidatorSet`로 초기 Validator Set 등록 (hex JSON; `total_power` 생략 가능)
- admin은 체인코드별 TOFU(`EnsureAdmin`) — 동일 신원으로 첫 admin 호출
- JSON 인자는 jq로 escape: `jq -cn --arg p "$JSON" '{Args:["SetValidatorSet",$p]}'` (또는 `--rawfile`). **timestamp 정밀도** 위해 jq가 payload를 파싱하지 않게(`--arg`/`--rawfile`) 주의 (1779…E18 같은 큰 정수)

### 검증 상태 / 제약

- 샌드박스에 Go 미설치 + on-bprn vendor에 tendermint 누락 → Go 컴파일은 로컬: `cd verifier/on-bprn && go mod vendor && go build ./...`. `gogo/protobuf/types`, `abci/types`, `crypto/merkle`, `crypto/secp256k1`, `proto/tendermint/{types,crypto}`가 vendor에 포함돼야 함(모두 tendermint가 쓰는 패키지)
- on-bpun Hardhat 컴파일도 샌드박스 제약(HHE21) → 로컬 `npx hardhat compile`
- prover-ts: `tsc --noEmit` 통과

### 주요 결정 요약

- 레지스트리 BPrN 값 = 체인코드 이름(string) (BTIP-9 20B 주소 아님)
- 커스텀 ValidatorSet/Validator 제거 → 크립토 = tmproto, 경계 = hex DTO
- **해시뿐 아니라 인코딩/디코딩도 tendermint/gogo 라이브러리 직접 호출** (DecodeAminoBytes=gogo BytesValue, tx_result=abci.ResponseDeliverTx, ValidatorsHash=tmtypes.Hash(), RFC6962=crypto/merkle)
- BTIP-32 외부 = hex string, `total_power`는 optional 태그

---

## 2026-06-01 — 재동기화 (audit-report-2026-05-28.md 후속)

> `audit-report-2026-05-28.md`는 HEAD `678d245` 기준이었음. 그 뒤 두 커밋(`f414756 feat: Add LinkerRegistry and update chaincodes`, `31466e0 refactor: Use tendermint original logic and data types`)이 푸시되어 감사 보고서가 지적한 on-bprn 갭 대부분이 해소됐고, 동시에 새로 발견·반영된 항목이 있어 본 문서를 코드 사실에 맞춰 정렬한다. 본 절은 *현재 코드(`feat/kyle`@`31466e0`)와 위쪽 본문의 정합 상태*를 절대 기준으로 정리.

### 1. 컴포넌트별 실제 상태 (코드 기준)

#### 1.A on-bprn (`verifier/on-bprn/`) — 감사 시점 ❌ → 현재 ✅

감사 리포트가 "어느 브랜치에도 존재하지 않음"으로 기록한 산출물이 `f414756`/`31466e0`에 실재함을 직접 확인. 위 §"2026-05-26 (구현 세션)" 항목들의 *코드 반영 여부*는 다음과 같다.

| 항목 | 파일 | 상태 |
|------|------|------|
| `types/registry.go` (`RegistryIDKey`, `SetRegistryID`/`GetRegistryID`, `ResolveContract`) | `types/registry.go` | ✅ 실재 |
| `types/ibtip37.go` (Role* 상수 + `IBTIP37`) | `types/ibtip37.go` | ✅ 실재 |
| `linker-registry/` 체인코드 (`GetContract`/`SetContract`, `ContractRegistered` 이벤트) | `linker-registry/main.go` | ✅ 실재 |
| `types/tmverify.go` (`VerifyCommitSignatures`/`VerifyRFC6962`/`ComputeValidatorsHash`/`DecodeAminoBytes`) | `types/tmverify.go` | ✅ 실재 |
| `types/validatorset_dto.go` (`ValidatorSetDTO`/`ValidatorDTO`/`ToProto`/`Entries`/`HexToBytes`) | `types/validatorset_dto.go` | ✅ 실재 |
| 커스텀 `ValidatorSet`/`Validator` 제거 → tmproto + hex DTO | `types/types.go` | ✅ (도메인 타입 없음, 주석으로 명시) |
| BTIP-39 5단계 `UpdateValidatorSet` 구현 (panic 제거) | `linker-policy/main.go:223+` | ✅ 실재 (idempotent 멱등 처리 포함) |
| linker-verifier secp256k1 교정(ed25519 제거) → `tmverify.VerifyCommitSignatures` 위임 | `linker-verifier/main.go` | ✅ (ed25519 import 0) |
| 개별 setter 제거 → `SetRegistryID` + `ResolveContract` | `linker-endpoint`, `linker-verifier`, `linker-nullifier`, `dapp-example` 전부 | ✅ 실재 |
| `IBTIP32`(`GetValidatorSet`/`SetValidatorSet`/`GetValidator`/`GetLatestHeight`) hex DTO 인터페이스 | `types/ibtip32.go` | ✅ 실재 |
| `IBTIP39`(`UpdateValidatorSet`) 인터페이스 | `types/ibtip39.go` | ✅ 실재 |

→ **on-bprn은 감사 리포트가 지적한 갭이 전부 해소됨.** 위쪽 본문의 "2026-05-26 (구현 세션)" 항목들은 모두 위 두 커밋으로 푸시·머지됐다.

#### 1.B on-bpun (`verifier/on-bpun/`) — 구현 ↔ 문서 갭

**현재 실재하는 contracts/** (실측):
```
contracts/
├── BTIP26Dapp.sol          (구 MockDApp 리네임)
├── LinkerEndpoint.sol
├── LinkerNullifier.sol
├── LinkerVerifier.sol
├── LinkerPolicy.sol
├── LinkerPolicyVerifier.sol      ← 정책 엔진 (signatureRuleTree, implicitMetaPolicy)
├── LinkerPolicyLib.sol           ← 정책 평가 라이브러리
├── LinkerPolicyTypes.sol         ← 정책 타입 (ConfigBlockInfo, SignatureRuleTree 등)
├── SignatureVerifier.sol         ← 인증서/서명 검증 분리
├── LinkerRegistry.sol            ← BTIP-37 구현 (Ownable, (chainId, role) 키)
├── interfaces/
│   ├── IBTIP21.sol  IBTIP22.sol  IBTIP23.sol  IBTIP24.sol  IBTIP26.sol
│   └── IBTIP37.sol               ← BTIP-37 인터페이스
└── lib/
    ├── MSPRoleLib.sol            ← MSP role 평가
    ├── P256Verify.sol            ← 0x0100 precompile 래퍼
    └── X509Verify.sol            ← 0xff00 precompile 래퍼
```

**위 본문 대비 정합 상태**:

| 항목 | 본문 기재 | 실제 코드 | 판정 |
|------|----------|-----------|------|
| BTIP-37 `IBTIP37`/`LinkerRegistry` | 2026-05-26 (구현 세션) "신규" | `interfaces/IBTIP37.sol`/`LinkerRegistry.sol` 실재 (Ownable + `(chainId, role) → address`, `LINKER_*` 상수 4개) | ✅ |
| 정책 엔진 4종(`LinkerPolicyVerifier`, `LinkerPolicyLib`, `LinkerPolicyTypes`, `SignatureVerifier`) + `lib/{MSPRoleLib,P256Verify,X509Verify}` | **본문 미기재** | 실재(Fabric channel config 기반 endorsement policy 평가) | ⚠️ 문서 갭 — 본 절 §3 신규 기록 |
| `BTIP26Dapp.sol`(구 MockDApp) | 본문에 "MockDApp → BTIP26Dapp 리네임" 언급 부족 | 실재 | ⚠️ 본문 표 갱신 필요 (아래 본 절 §3) |
| Foundry 환경 (`foundry.toml`, `test-forge/LinkerGasTest.t.sol`, `mocks/`) | 본문 2026-04-20 (4차) 기재 | 실재 | ✅ |
| 2PC 산출물 (`onResult`, `LinkerResult`, `handleLinkerResult`, `try/catch`, `nonReentrant`, `MIN_CALLBACK_GAS`, `cancelLinkerEvent` IBTIP26 갱신) | 본문 미기재(설계만 `btips`/`btips-2pc-design`에 존재) | **모두 부재** (`grep` 0건) | ❌ 미구현 — 다음 우선순위 |
| **IBTIP21 setter 정리**(`setNullifierContract`/`setVerifierContract` 제거 → `setRegistry` 단일화, LinkerRegistry 동적 조회) — BTIP-29(BPrN)에 이미 적용된 정합 | 본문(linker-v2.md) 미기재 — 2026-06-01 BTIP-21 명문화로 합의됨 | `setNullifierContract`/`setVerifierContract` 두 setter가 `IBTIP21.sol`(L34/37) + `LinkerEndpoint.sol`(L50/56) 양쪽에 잔존, `setRegistry` 부재 | ❌ 미구현 — 다음 우선순위 (2PC와 함께 또는 선행) |
| **IBTIP23 setter 정리**(`setPolicyContract` 제거 → `setRegistry` 단일화, LinkerRegistry로 LinkerPolicy 동적 조회) | 본문 미기재 — 2026-06-01 BTIP-23 명문화 | `setPolicyContract` 잔존(`LinkerVerifier.sol`/`IBTIP23.sol`), `setRegistry` 부재 | ❌ 미구현 |
| **IBTIP24 setter 정리**(`setRegistry` 신설, onlyLinkerEndpoint를 LinkerRegistry 동적 조회로) | 본문 미기재 — 2026-06-01 BTIP-24 명문화 | LinkerEndpoint 주소 보관 setter가 잔존(있다면), `setRegistry` 부재 | ❌ 미구현 |
| **BTIP-23 Step 2~4 위임**(verifyProof에서 `LinkerPolicy.verifyChannelEndorsementPolicy` 단일 호출, precompile 직접 호출 제거) | 본문 미기재 — 2026-06-01 BTIP-23 명문화 | `LinkerVerifier.sol`이 정책 엔진(LinkerPolicyVerifier 등)과 어떻게 결합돼 있는지 별도 확인 필요. *위임 패턴이 일치하면 ✅, 아니면 갭* | ⚠️ 코드 재확인 필요 |
| **BTIP-40 `LinkerTransfer` Solidity event 표준**(`event LinkerTransfer(address indexed from, address indexed to, uint256 amount, bytes32 indexed correlationId, bytes memo)`) | 본문 미기재 — 2026-06-01 BTIP-40 신규 | 표준 event 정의 부재(현 BTIP26Dapp은 자체 `LinkerProofReceived` event 사용) | ❌ 미구현 — *결제 use case dApp이 채택 필요* |
| **BTIP-26 `ErrAppLowGas` 표준 custom error**(권장 보호 패턴 — dApp이 `revert ErrAppLowGas()` 시 LinkerEndpoint가 catch에서 인식해 전체 revert) | 본문 미기재 — 2026-06-01 BTIP-26 명문화 | `ErrAppLowGas` 정의/사용 부재. BTIP26Dapp.sol·LinkerEndpoint.sol 모두 미반영 | ❌ 미구현 (권장 옵션이므로 우선순위 낮음) |
| **BTIP-37 LINKER_CCS NOTE 제거** | 본문 미기재 — 2026-06-01 BTIP-37 명문화 | `LinkerRegistry.sol`이 LINKER_* 상수 4개만 보유 — LINKER_CCS 미구현 상태로 일치 | ✅ |
| **markProcessed 내부 revert**(`returns (bool wasDup)` 제거, 중복 시 `DuplicateProof`로 직접 revert) + `DuplicateProof` 정의 BTIP-21 → BTIP-24 이동 | 본문 미기재 — 2026-06-01 BTIP-24/21 명문화 | `IBTIP24.sol`이 `markProcessed returns (bool wasDup)` 패턴이고 `DuplicateProof`가 `IBTIP21.sol`에 정의돼 있을 것 | ❌ 미구현 (시그니처/에러 위치 둘 다 변경 필요) |
| `IBTIP21` 시그니처 | 본문 2026-04-20 (1차) 기준 (`TxEventProof.mspids`/`block_number`/`event_log_root_proof` 등) | 일치 | ✅ |
| `IBTIP21.onResult` 추가 (2PC) | 본문 미기재 | 부재 | ❌ 미구현 |
| `scripts/beatoz/` (`deploy.ts`/`set-policy.ts`/`init-policy.ts`/`submit-proof.ts`/`cancel-event.ts`/`setup.sh`/`utils.ts`) + `send-op-tx.ts` | 본문 표는 `set-policy.ts`/`setup-localnet0.sh`로 기재 | `set-policy.ts`+`init-policy.ts` 둘 다 존재, `setup.sh`만 있고 `setup-localnet0.sh` 없음 | ⚠️ 스크립트명 갱신 (아래 본 절 §3) |

#### 1.C prover-ts (`prover-ts/`) — 부분 정합

| 항목 | 본문 기재 | 실제 코드 | 판정 |
|------|----------|-----------|------|
| BTIP-39 Prover `src/prover/btip39/` (`rpc.ts`, `header-merkle.ts`, `validator-set-proof.ts`, `types.ts`, `main.ts`) + `npm run btip39:scan` | 2026-05-26 (구현 세션) "신규" | 실재 (5개 파일, `package.json` `scripts.btip39:scan` 등록) | ✅ |
| BPuN→BPrN 이벤트 Prover `src/prover/u2r/` (감사: "빈 디렉토리 placeholder") | 본문 표에 "미구현으로 기록" | **디렉토리 자체 부재** (감사 시점의 placeholder도 삭제) | ❌ 여전히 미구현, 시작점 없음 |
| Fabric SDK | 본문: `fabric-network` v2.2.20 전용 (감사: `@hyperledger/fabric-gateway` 1.7.0 추가됨) | `package.json`에 `fabric-network` 2.2.20 + `@peculiar/x509` + `protobufjs`만 — fabric-gateway 부재 | ✅ 본문(v2.2.20 전용)이 현 상태 정확. 감사의 fabric-gateway 관찰은 stale 또는 별도 클론 추정 |
| `scripts/test-proof.ts` (구 `test-prove.ts`) | 2026-04-21 항목 | 실재 | ✅ |
| BPrN forward prover (`prover.service.ts`, `common/{event-log,merkle}.ts`) | 본문 다수 항목 | 실재 | ✅ |
| Multi-peer block-commit-sig 수집 | 본문 미완료 | 미구현 | ❌ 그대로 미완료 |

### 2. 다음 우선순위 (코드 기준)

감사 리포트의 권장 1·2번 항목은 (이 재동기화로) 해소됐고, 남은 결정적 갭은 다음 두 가지다.

1. **BPuN→BPrN 이벤트 Prover (`u2r`) 구현** — `prover-ts/src/prover/u2r/` 디렉토리부터 신설. 출력은 BTIP-28 `BPuNTxEventProofPayload`(types/types.go 기 정의). BPuN Tendermint RPC(26657)에서 블록·last_results·tx_result·event_attrs·event_attr Proof를 모두 구성해야 함. BTIP-39 Prover(`btip39/header-merkle.ts`)의 헤더 Merkle 인코딩과 v0.34 cdcEncode 로직을 재사용 가능.
2. **2PC 구현** — 설계는 `btips-2pc-design.md` §1~9 + `btips.md` 2026-05-21/22/27 세션에 완결. 코드 차원에서는:
   - on-bpun: `IBTIP21.onResult` 추가, `LinkerResult` 이벤트, `LinkerEndpoint.onProof`에 `try/catch`+`nonReentrant`+`MIN_CALLBACK_GAS`+always-emit, `IBTIP26`에 `handleLinkerResult(correlationId, handlerCcApp, accepted)` 추가, `BTIP26Dapp` 콜백 호출자 검증(`LinkerRegistry.getContract(block.chainid, LINKER_ENDPOINT)`).
   - on-bprn: `linker-endpoint`에 `OnResult` 메소드, `LinkerResultElems` EventLog, `dapp-example`에 `HandleLinkerResult` 콜백, `LinkerEndpointCC.HandleLinkerEvent` 반환을 `(LinkerResultRef, error)` 형태로 변경(BTIP-34).
3. **on-bpun Prover multi-peer block-commit-sig 수집** — BTIP-17 검증 강도 보강.

### 3. 본문 기재 보강 (본문 표/스크립트 표 등 정정)

다음 본문 항목은 사실 정정이 필요하나 분량이 많아 본 절에 통합 기록하고 본문 표는 *과거 기록 그대로 보존*한다. 이후 작업에서 본문 표 갱신을 분리 진행한다.

- **MockDApp → `BTIP26Dapp.sol`**: 본문 곳곳의 `MockDApp.sol` 참조는 현 파일명 `BTIP26Dapp.sol`로 매핑됨. 인터페이스/이벤트(`LinkerProofReceived`)/구조는 본문 기재대로.
- **on-bpun 정책 엔진 추가 (본문 미기재)**: BTIP-22 `IBTIP22`의 단순 ABI 인코딩 정책(`abi.decode(policy,(uint256,bytes[]))`)에서 진화하여, Fabric channel config의 endorsement policy를 그대로 평가하는 엔진이 들어옴.
  - `LinkerPolicyTypes.sol`: `ConfigBlockInfo`, `SignerInfo`, `CertInfo`, `OrgCertificates`, `SignatureRuleTree`, `implicitMetaPolicy` 등 정의.
  - `LinkerPolicyVerifier.sol`(`IBTIP22` 구현): `verifyBlockValidationPolicy`/`verifyChannelEndorsement(Policy)`/`verifyPolicy` 등. SignatureRuleTree 평가(`_evaluateSignatureRuleTree`)와 implicitMetaPolicy(M-out-of-N at sub-policy) 계산을 수행.
  - `LinkerPolicyLib.sol`: 정책 트리/조합 평가 헬퍼.
  - `SignatureVerifier.sol`: 인증서(`X509Verify`)/서명(`P256Verify`) 검증 분리 모듈. `LinkerPolicy`의 `OrgCertificates`(rootCert + intermediateCerts) 조회.
  - `lib/{MSPRoleLib,P256Verify,X509Verify}.sol`: precompile 래퍼 + MSP role 평가.
  - 본문 §"BEATOZ EVM Precompiles"의 `0x0100`/`0xff00` 사용처가 직접 `LinkerVerifier`가 아니라 본 정책 엔진을 경유함.
- **`set-policy.ts` + `init-policy.ts` 공존**: 본문 표는 `set-policy.ts` 단독으로 기재하고 있으나 실제 디렉토리엔 두 스크립트가 모두 존재. 정확한 역할 분리는 스크립트 헤더 코멘트 참조(이번 재동기화에선 미확인).
- **`setup-localnet0.sh` → `setup.sh`**: 본문 표의 `setup-localnet0.sh`는 실제 파일명 `setup.sh`(`Usage: ./scripts/beatoz/setup.sh <chain>`). 인자 받아 일반화됨.
- **BTIP26Dapp의 `cancelLinkerEvent` 이벤트 추가**: `event LinkerEventCancelled(bytes32 indexed eventRootHash)` 신설 — 본문 미기재.

### 4. 신뢰 기준 갱신

- 본문(이 문서)은 §"2026-06-01" 시점에 코드 사실에 맞춰 정렬됨. `audit-report-2026-05-28.md`는 *역사 기록*으로 보존하되, "어느 브랜치에도 없음" 판정 항목들은 위 §1.A로 해소됐음을 유의.
- 향후 본문의 "✅ 완료" 표기는 *코드에 푸시된 시점의 기록*임을 명확히 하기 위해 가능하면 커밋 해시 동반 기록 권장(예: `…(반영: f414756)`).

