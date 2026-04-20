# Linker V2 Solidity 작업 컨텍스트

> 마지막 업데이트: 2026-04-20 (3차) — BTIP 문서 변경사항 반영

---

## 작업 개요

BTIP21, 23, 24에 정의된 Linker Protocol V2 온체인 컴포넌트를 Solidity 컨트랙트로 구현.
BTIP 문서 원본: `/Users/kylekwon/projects/beatoz/docs/BTIPS/`

---

## 프로젝트 구조

- **Solidity 컨트랙트**: `/Users/kylekwon/projects/beatoz/linker-v2/verifier/`
  - Hardhat 3.3.0, Solidity 0.8.28
  - 플러그인: `@nomicfoundation/hardhat-toolbox-viem`
  - 의존성: `@openzeppelin/contracts` (Ownable)
- **Prover API**: `/Users/kylekwon/projects/beatoz/linker-v2/prover/`
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
prover/
├── package.json
├── tsconfig.json / nest-cli.json
├── .env / .env.example
├── connection-profile.json          # 절대경로 기반 (prover/ 디렉토리에 위치)
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
| `submit-proof.ts <chainAlias> <dappAlias>` | TxEventProof 제출 (신규 필드 mspids/block_number/index 반영) |
| `query-dapp.ts <chainAlias> <dappAlias> [index]` | MockDApp 이벤트 조회 (srcBlockNumber/srcTxIndex/indices/values 기준) |
| `cancel-event.ts <chainAlias> <dappAlias> <eventRootHash>` | nullifier 취소 |

- dApp 주소는 모두 `<dappAlias>` 로 받아 `.net/deployed.<chainAlias>.<dappAlias>.json` 에서 로드
- `utils.ts`: `deliver_tx.log`의 `reason: <hex>` 패턴에서 hex를 추출해 keccak256 selector로 커스텀 에러 디코딩
  - BEATOZ는 revert data를 `deliver_tx.log`에 `reason: <HEX (no 0x)>` 형식으로 기록

---

## 미완료

- `Counter.sol` 샘플 파일 삭제 (hardhat init 생성물)
- BTIP22 LinkerPolicy 스펙 확정 후 정식 구현
- Prover: 각 endorser 피어에게 **개별 블록 조회**하여 peer별 block commit sig 수집 (현재는 단일 블록 조회 → 하나의 sig만 수집)
- Verifier 계약: 샌드박스 환경 제약으로 `npx hardhat compile` 검증 미완 (로컬에서 `npx hardhat clean && npx hardhat compile` 재확인 필요)

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
