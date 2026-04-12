# Linker V2 Solidity 작업 컨텍스트

> 마지막 업데이트: 2026-04-09 (2차)

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
    │   ├── merkle.ts                # SHA256 MerkleTree
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

### ✅ API

**`GET /api/prove?txId=...&eventGindices=4,5,6,7&channelId=bpn`**

Response: `TxEventProofDto` — Solidity `IBTIP21.TxEventProof` 구조와 1:1 대응

### ✅ gindex 정의 (BTIP-16)

- `gindex` = EventLog leaf array 에서의 **0-based leaf index** (단순 위치)
  - 0=channel_id, 1=chaincode_id, 2=tx_id, 3=selector, 4=elems[0], 5=elems[1], ...
- `leafCount + index` 같은 tree 내부 node index가 **아님**

### ✅ block_event_root 구성 (BTIP-17)

- 블록 내 **모든 tx**를 block 순서대로 leaf로 포함
  - 성공 tx (chaincode event 있음) → `event_log_root` (32B hash)
  - 실패/event 없는 tx → `0x00...00`
- leaf가 이미 hash이므로 tree 구성 시 `preHashed=true`
- `event_log_root_proof.gindex` = 블록 내 tx의 위치 (0-based)
- `verifyMerkleProof` 시 `preHashed=true` (leaf를 다시 sha256하면 안 됨)

### ✅ Fabric 연결

- `fabric-network` v2.2.20 SDK 사용 (`@hyperledger/fabric-gateway` 아님 — BPrN은 구버전 Fabric)
- `qscc.evaluateTransaction('GetBlockByTxID', channelId, txId)` → raw Block proto bytes
- `BlockParserService.decodeBlock()` 으로 protobufjs 디코딩
- 환경설정: `.env` + `connection-profile.json` (절대경로)

### ✅ 로컬 proof 검증

생성된 proof를 반환 전에 로컬에서 검증:
1. `event_log_root_proof`: `verifyMerkleProof(gindex, evtLogRoot, siblings, blockEventRoot, true)`
2. `event_elem_proofs[i]`: `verifyMerkleProof(gindex, leaf, siblings, evtLogRoot, false)`

### ✅ 테스트 스크립트

`scripts/test-prove.ts` — `npx tsx scripts/test-prove.ts`

### ✅ verifier 스크립트 목록 (`scripts/beatoz/`)

| 스크립트 | 용도 |
|----------|------|
| `deploy.ts <chainAlias>` | 전체 컨트랙트 배포 + wire-up |
| `set-policy.ts <chainAlias>` | LinkerPolicy에 RootCA/EndorsementPolicy 설정 |
| `setup-localnet0.sh` | deploy + set-policy 한 번에 실행 |
| `submit-proof.ts <chainAlias> <dappAlias>` | TxEventProof 제출 |
| `query-dapp.ts <chainAlias> <dappAlias> [index]` | MockDApp 이벤트 조회 |
| `cancel-event.ts <chainAlias> <dappAlias> <eventRootHash>` | nullifier 취소 |

- dApp 주소는 모두 `<dappAlias>` 로 받아 `.net/deployed.<chainAlias>.<dappAlias>.json` 에서 로드
- `utils.ts`: `deliver_tx.log`의 `reason: <hex>` 패턴에서 hex를 추출해 keccak256 selector로 커스텀 에러 디코딩
  - BEATOZ는 revert data를 `deliver_tx.log`에 `reason: <HEX (no 0x)>` 형식으로 기록

---

## 미완료

- `Counter.sol` 샘플 파일 삭제 (hardhat init 생성물)
- BTIP22 LinkerPolicy 스펙 확정 후 정식 구현
- Prover: 각 endorser 피어에게 **개별 블록 조회**하여 peer별 block commit sig 수집 (현재는 단일 블록 조회 → 하나의 sig만 수집)
- Prover: 최종 컴파일 확인 (재컴파일)

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
- `fabric-network` SDK는 v2.x 사용 (v2.4+ gateway.Gateway API 아님)
- qscc 시스템 체인코드로 블록 조회: `GetBlockByTxID`, `GetBlockByNumber`
- EventLog DER: `onlyElems=true` 모드 → ASN.1 SEQUENCE of OCTET STRING
