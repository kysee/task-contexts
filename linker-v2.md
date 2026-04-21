# Linker V2 Solidity 작업 컨텍스트

> 마지막 업데이트: 2026-04-21 — IBTIP24 `markProcessed` 통합, `prover` → `prover-ts` 리네임, BTIP-16 `tx_id`/`selector` 바이트 반영

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

