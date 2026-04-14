---
last_updated: 2026-04-14
---

# BTIPS 작업 컨텍스트

---

## 작업 개요

BEATOZ Linker Protocol V2의 양방향 크로스체인 증명 체계 설계.

- **BPrN → BPuN 방향** (btip-16~26): BTIP19의 단일 경로 신뢰 모델 재작성, 필드명 시맨틱 교정, 제네릭 타입 표기법 도입, Proof Construction/Verification 재구성, 인터페이스 정리 등 완료.
- **BPuN → BPrN 방향** (btip-27~28): BPuN 이벤트 구조, 2단계 머클 트리, Tendermint 합의 연계, 다이제스트 증명 페이로드 설계 완료. 변수명 직관성 리팩터링 완료.

---

## 핵심 기술 개념

### 단일 경로 신뢰 모델 (Single-Path Trust Model)

```
Root CA → 인증서 체인 → 블록 커밋 서명 → block_event_root → event_log_root → 개별 이벤트
```

### TxEventProofPayload (현행 — btip-19 기준)

```text
Structure MerkleProof<Leaf>:
    Property gindex:   Integer
    Property leaf:     Leaf
    Property siblings: Array of ByteArray

Structure TxEventProofPayload:
    Property mspids:                Array of String
    Property cert_chains: Array of Array of ByteArray
    Property block_commit_sigs:     Array of ByteArray
    Property block_event_root:      ByteArray
    Property event_log_root_proof:  MerkleProof<event_log_root>
    Property event_elem_proofs:     Array of MerkleProof<event_elem>
```

**필드명 변경 이력:**
- `commit_sigs` → `block_commit_sigs` (블록 커밋 서명임을 명시)
- `event_log_proof` → `event_log_root_proof` (event_log_root에 대한 증명임을 명시)
- `mspids` 필드 추가 (조직별 Root CA 조회를 위한 MSP 식별자)
- `endorser_certs_chains` → `cert_chains` (인증서 체인 자체는 endorser 여부를 의미하지 않으므로 중립적 이름으로 변경)

**Solidity 대응 (btip-21):**
```solidity
struct MerkleProof {
    bytes     leaf;
    uint256   gindex;
    bytes32[] siblings;
}

struct TxEventProof {
    string[]      mspids;
    bytes[][]     cert_chains;
    bytes[]       block_commit_sigs;
    bytes32       block_event_root;
    MerkleProof   event_log_root_proof;
    MerkleProof[] event_elem_proofs;
}
```

### Commit Signature (btip-17 기준)

- 각 피어의 블록 `metadata[5]`에는 해당 피어 자신의 커밋 서명만 존재
- n개의 서명을 수집하려면 n개의 보증 피어에게 각각 블록을 조회해야 함

### 머클 트리 null 처리 규칙 (btip-16 기준)

부족한 노드 및 실패 트랜잭션 리프는 `null`로 처리. `null`은 노드가 존재하지 않음을 나타내는 언어 독립적 표기 (Go: `nil`, Python: `None` 등).

| 입력 | 결과 |
|------|------|
| `hashPair(A, B)` | `hash(A \|\| B)` |
| `hashPair(A, null)` | `hash(A)` |
| `hashPair(null, null)` | `null` |

### Precompiled Contracts

- `0xff00` (`beatoz_x509Verify`) — X.509 인증서 체인 검증. 입력 순서: `[subject, issuer]` (subject → issuer). 반환: `[32B pubkey.x][32B pubkey.y][32B serialNumber][OU bytes]` (certs[0] 기준)
- `0x0100` (`beatoz_p256Verify`) — P-256 ECDSA 서명 검증

### 해시 함수

- 전체 문서에서 `keccak256` → `sha256`으로 변경 완료 (btip-20, 21, 23, 24)
- ZKP(Groth16/BN254) 관점에서도 SHA-256이 Keccak-256보다 회로 비용이 약 6배 낮음

### BPuN 단일 경로 신뢰 모델 (btip-27, 28)

```
Validator Set (2/3+ Voting Power)
  → Commit Signatures (CanonicalVote over BlockID)
    → block_hash (Block Header Merkle Root)
      → LastResultsHash (Header 내부 머클 트리의 리프, 인덱스 11)
        → tx_result (ABCIResults 머클 트리의 리프)
          → tx_event_root (tx_result.Data)
            → event_attrs_root (Tx Event Tree의 리프)
              → 개별 이벤트 속성 (Per-Event Tree의 리프)
```

### BPuNTxEventProofPayload (현행 — btip-28 기준)

```text
Structure TendermintProof:
    Property total:   Integer        // RFC 6962 트리의 전체 리프 수 (분할 지점 결정에 필요)
    Property index:   Integer
    Property leaf:    ByteArray
    Property aunts:   Array of ByteArray

Structure MerkleProof<Leaf>:
    Property index:    Integer
    Property leaf:     Leaf
    Property siblings: Array of ByteArray

Structure CommitSignature:
    Property validator_address:  ByteArray
    Property timestamp:          Timestamp   // CanonicalVote 서명 입력에 포함됨
    Property signature:          ByteArray

Structure BPuNTxEventProofPayload:
    Property height:                   Integer
    Property chain_id:                 String
    Property round:                    Integer
    Property block_hash:               ByteArray
    Property block_parts_header:       PartSetHeader
    Property commit_sigs:              Array of CommitSignature
    Property last_results_proof:       TendermintProof
    Property tx_result_proof:          TendermintProof
    Property event_attrs_root_proof:   MerkleProof<event_attrs_root>
    Property event_attr_proofs:        Array of MerkleProof<attr_leaf>
```

**변수명 변경 이력 (2026-04-14):**
- `event_root` → `tx_event_root` (트랜잭션 내 모든 이벤트의 루트 — tx 레벨 명시)
- `per_event_root` → `event_attrs_root` (하나의 이벤트 내 속성들의 루트 — 커버 범위 명시)
- `results_proof` → `tx_result_proof` (하나의 tx 결과에 대한 증명)
- `header_results_proof` → `last_results_proof` (증명 대상인 LastResultsHash 기준으로 통일)
- `event_root_proof` → `event_attrs_root_proof` (증명 대상인 event_attrs_root 기준)
- `event_attr_proofs` 유지 (개별 attr 증명의 배열이므로 단수형 attr)
- 명명 원칙: 증명 필드는 **증명 대상(what is being proven)** 기준으로 명명

**Leaf Data Summary:**

| 증명 | 머클 트리 루트 | 리프 데이터 | 리프 해시 방식 | 머클 트리 유형 |
|------|--------------|-----------|--------------|-------------|
| `last_results_proof` | `block_hash` | `cdcEncode(LastResultsHash)` | `sha256(0x00 \|\| leaf)` | Tendermint RFC 6962 |
| `tx_result_proof` | `LastResultsHash` | `protobuf(Code, Data, GasWanted, GasUsed)` | `sha256(0x00 \|\| leaf)` | Tendermint RFC 6962 |
| `event_attrs_root_proof` | `tx_event_root` | `event_attrs_root` (32B hash) | preHashed (추가 해시 없음) | BTIP27 단순 연결 |
| `event_attr_proofs[i]` | `event_attrs_root` | `Type \|\| Key \|\| Value` (원본) | `sha256(leaf)` | BTIP27 단순 연결 |

### 두 가지 머클 트리 유형

| 구분 | Tendermint RFC 6962 | BTIP27 단순 연결 |
|------|-------------------|----------------|
| 리프 해시 | `sha256(0x00 \|\| leaf)` | `sha256(leaf)` 또는 preHashed |
| 내부 노드 | `sha256(0x01 \|\| left \|\| right)` | `hashPair(left, right)` with null 처리 |
| 트리 분할 | n보다 작은 최대 2의 거듭제곱으로 분할 | 다음 2의 거듭제곱까지 null 패딩 |
| total 필드 | 필요 (분할 지점 결정) | 불필요 (패딩 방식이므로 구조 결정적) |
| 적용 구간 | Step 3~4 (last_results_proof, tx_result_proof) | Step 5~6 (event_attrs_root_proof, event_attr_proofs) |

### Tendermint 헤더 필드 Amino 인코딩

Tendermint 블록 헤더의 14개 필드(`int64`, `string`, `time.Time`, 구조체, `[]byte` 등 이질적 타입)는 머클 트리에 넣기 전에 `cdcEncode`(`amino.MarshalBinaryBare`)로 정규 직렬화된다. `LastResultsHash`처럼 이미 `[]byte`인 필드에도 Amino가 적용되어 varint 길이 접두사가 추가된다(32B → `[0x20][32B]` = 33B). 이는 타입별 분기 없이 14개 필드를 동일한 직렬화 경로로 통일하기 위한 Tendermint의 설계 선택이다.

### Validator Set 전제

Verifier(BPrN)는 대상 블록 높이의 BPuN Validator Set을 이미 보유하고 있다는 전제. `height`로 Validator 공개키와 Voting Power를 조회하여 서명을 검증한다. Validator Set의 초기 등록 및 변경 동기화는 별도 문서에서 다룬다. 이에 따라 증명 페이로드에 Validator Set 데이터를 포함하지 않는다.

### BPrN vs BPuN 신뢰 모델 비교

| 관점 | BPrN (BTIP19) | BPuN (BTIP28) |
|------|--------------|--------------|
| 신뢰 기반 | PKI (Root CA → 인증서 체인) | PoS (Validator Set → 2/3+ 합의) |
| 서명 주체 | 보증 피어 (조직 소속) | Validator (지분 기반) |
| 서명 대상 | `block_event_root` | `BlockID` (블록 헤더 해시) |
| 정책 검증 | Endorsement Policy | 2/3+ Voting Power |

---

## 세션별 완료 작업

### 2026-04-14

#### ✅ btip-27.md — BPuN Event Structure 신규 생성

- BTIP16(BPrN Event Structure)의 BPuN 대응 문서
- Tendermint ABCI `Event` 데이터 모델 정의 (Transaction Event, EVM Contract Event)
- 2단계 머클 트리(Two-Level Merkle Tree) 구조 정의:
  - 1단계: Per-Event Tree — 각 이벤트 속성을 리프로 구성, `event_attrs_root` 산출
  - 2단계: Tx Event Tree — `event_attrs_root` 해시들을 리프로 구성(preHashed), `tx_event_root` 산출
- 리프 구성: `Event.Type || Attribute.Key || Attribute.Value` (구분자 없이 연결, SHA-256 해시)
- `hashPair` 규칙: BTIP16과 동일 (단순 연결 방식, null 처리)
- `tx_event_root` → `ResponseDeliverTx.Data` → `ABCIResults` → `LastResultsHash` → Header → `BlockID` → `CanonicalVote` → Validator Signature 신뢰 경로 기술
- Tendermint RFC 6962 머클 트리와 BTIP27 단순 연결 방식의 차이 명시
- Transfer 트랜잭션, EVM 컨트랙트 트랜잭션 예시 다이어그램 포함

#### ✅ btip-28.md — BPuN Tx/Event Proof 신규 생성

- BTIP19(BPrN Tx/Event Proof)의 BPuN 대응 문서
- BPrN(PKI) vs BPuN(PoS) 신뢰 모델 비교표
- BPuN Single-Path Trust Model 정의
- `BPuNTxEventProofPayload` 다이제스트 설계:
  - 블록 헤더 전체 대신 `block_hash` + `last_results_proof`(인덱스 11 머클 증명)
  - Commit 전체 대신 `commit_sigs`(실제 투표한 Validator 서명만)
  - Validator Set 제외 (Verifier 사이드에 동기화된 상태 전제, `[!NOTE]` 인용문)
  - `CanonicalVote` 재구성에 필요한 최소 필드: `height`, `chain_id`, `round`, `block_hash`, `block_parts_header`
- 4단계 머클 증명 경로:
  - `last_results_proof`: `block_hash` → `LastResultsHash` (Tendermint RFC 6962, 리프: `cdcEncode(LastResultsHash)`)
  - `tx_result_proof`: `LastResultsHash` → `tx_result` (Tendermint RFC 6962, 리프: `protobuf(Code, Data, GasWanted, GasUsed)`)
  - `event_attrs_root_proof`: `tx_event_root` → `event_attrs_root` (BTIP27 단순 연결, preHashed)
  - `event_attr_proofs`: `event_attrs_root` → 개별 속성 (BTIP27 단순 연결, 리프: `Type||Key||Value`)
- Leaf Data Summary 표 포함 (각 증명의 리프 데이터, 해시 방식, 머클 트리 유형)
- Proof Construction Algorithm (Prover), Verification Procedure (Verifier) 슈도코드 및 다이어그램
- Security Considerations: Validator 공모, Validator Set 부트스트래핑

#### ✅ btip-27, btip-28 — 변수명 직관성 리팩터링

변수/필드명이 의미(what it covers)가 아닌 구조적 위치(per event)를 나타내는 문제를 개선. 증명 필드명은 **증명 대상(what is being proven)** 기준으로 통일.

| 구 용어 | 신 용어 | 변경 이유 |
|--------|--------|----------|
| `event_root` | `tx_event_root` | tx 레벨의 루트임을 명시 |
| `per_event_root` | `event_attrs_root` | 이벤트 내 attributes의 루트임을 명시 |
| `results_proof` | `tx_result_proof` | 하나의 tx 결과에 대한 증명 |
| `header_results_proof` | `last_results_proof` | 증명 대상(LastResultsHash) 기준 명명 |
| `event_root_proof` | `event_attrs_root_proof` | 증명 대상(event_attrs_root) 기준 명명 |
| `event_attr_proofs` | `event_attr_proofs` | 유지 — 개별 attr 증명의 배열이므로 단수형 |

#### ✅ btip-28 — "Validator Set의 전제" 인용문 변환

- `### heading` 형식 → `> [!NOTE]` 인용문 형식으로 변경

#### ✅ README.md — BTIP27, BTIP28 항목 추가

- BTIP27 (BPuN Event Structure) 항목 추가
- BTIP28 (BPuN Tx/Event Proof) 항목 추가

#### 설계 논의 기록

- **`TendermintProof.total` 필드**: RFC 6962 머클 트리는 "n보다 작은 최대 2의 거듭제곱" 지점에서 좌우 분할하므로, 검증자가 분할 지점을 알기 위해 `total` 필요. BTIP27의 단순 연결 방식은 2의 거듭제곱까지 null 패딩하므로 불필요.
- **`CommitSignature.timestamp` 필드**: `CanonicalVote` 서명 입력 메시지에 `timestamp`가 포함되므로, 서명 검증을 위해 각 Validator의 투표 시각이 필수.
- **Amino 인코딩 이유**: Tendermint 헤더 14개 필드가 이질적 타입(`int64`, `string`, `time.Time`, 구조체, `[]byte`)이므로, 모든 타입을 바이트로 변환하는 정규 직렬화가 필요. `[]byte` 필드에도 일괄 적용하는 이유는 타입별 분기 없는 코드 일관성.

#### ✅ btip-29.md — LinkerEndpoint Chaincode on BPrN 신규 생성

- BTIP21(LinkerEndpoint on BPuN)의 BPrN 대응 문서
- BPuN 이벤트 증명(`BPuNTxEventProofPayload`)의 유일한 관문(Gateway) 체인코드
- Go 구조체로 `BPuNTxEventProofPayload` 전체 정의 (BTIP28 대응)
- `OnProof`: Nullifier 검사(BTIP33) → 검증 위임(BTIP31) → 상태 기록 → 애플리케이션 이벤트 전달
- `SetVerifierChaincodeID`, `SetNullifierChaincodeID` 관리자 함수
- `ProofReceived`, `ProofVerified` 이벤트 정의
- Fabric 체인코드의 원자성 보장 설명 (보증 실패 시 커밋되지 않음)

#### ✅ btip-31.md — LinkerVerifier Chaincode on BPrN 신규 생성

- BTIP23(LinkerVerifier on BPuN)의 BPrN 대응 문서
- BTIP28 검증 절차 Step 2~6 구현 (Step 1은 BTIP29 책임)
- `VerifyProof`: Validator Set 조회(BTIP32) → Ed25519 서명 검증 → RFC 6962 머클 증명 → BTIP27 머클 증명
- 외부 의존성: BTIP32(ValidatorSetRegistry), Ed25519, 두 종류의 머클 증명
- BTIP23(Precompiled Contract 사용)과의 차이 명시

#### ✅ btip-32.md — ValidatorSetRegistry Chaincode on BPrN 신규 생성

- BPuN Validator Set(공개키, Voting Power, 주소)을 블록 높이별로 보관
- BTIP28의 "Verifier가 Validator Set을 보유" 전제를 충족하는 체인코드
- BTIP22(LinkerPolicy on BPuN)와 대칭적 역할 (상대 네트워크의 신뢰 기반 보관)
- `GetValidatorSet(height)`: 범위 조회로 해당 높이에 유효한 ValidatorSet 반환
- `SetValidatorSet`: 현재 수동 등록, 향후 Light Client 자동 동기화로 대체 가능
- 저장소 설계: `VS_{height}` 키, 0-패딩 16자리로 범위 조회 지원
- IBTIP22.sol / LinkerPolicy.sol 참조하여 설계 (소스 경로: `linker-v2/verifier/on-bpun/contracts/`)

#### ✅ btip-33.md — LinkerNullifier Chaincode on BPrN 신규 생성

- BTIP24(LinkerNullifier on BPuN)의 BPrN 대응 문서
- Nullifier: `sha256(txEventRoot || targetApp)` — commit_sigs 제외 (BTIP24와 동일 근거)
- `IsProcessed`, `MarkProcessed`, `CancelNullifier` 인터페이스
- `CancelNullifier`: 호출자 X.509 인증서 기반 접근 제어 (BTIP24의 `msg.sender`에 대응)
- Nullifier 대상 차이: BTIP24는 `event_root_hash`(= `event_log_root`), BTIP33은 `txEventRoot`(= `tx_event_root`)

#### ✅ README.md — BTIP29~33 항목 추가

- BTIP29 (LinkerEndpoint Chaincode on BPrN) 항목 추가
- BTIP30 (Reserved) 항목 추가
- BTIP31 (LinkerVerifier Chaincode on BPrN) 항목 추가
- BTIP32 (ValidatorSetRegistry Chaincode on BPrN) 항목 추가
- BTIP33 (LinkerNullifier Chaincode on BPrN) 항목 추가

#### 설계 원칙 기록

- **자료구조/인터페이스 표기**: BPrN 체인코드는 Go 문법으로 작성 (BPuN 스마트 컨트랙트는 Solidity)
- **BPuN ↔ BPrN 대칭 매핑**: BTIP21↔29, BTIP22↔32, BTIP23↔31, BTIP24↔33

### 2026-04-12

#### ✅ btip-16.md — 머클 트리 null 기반 처리로 변경

- Dummy 바이트(`0x00...00`) 패딩 방식 제거
- 부족한 노드는 `null`로 취급 (메모리 할당 없음)
- `hashPair` 규칙 표 추가: `(A,B)→hash(A‖B)`, `(A,null)→hash(A)`, `(null,null)→null`

#### ✅ btip-17.md — 실패 트랜잭션 리프를 null로 변경

- 실패 트랜잭션 리프 값: `0x00...00` → `null` (전체 6곳)
- "0x00...00으로 채우는" → "null로 자리를 유지하는"으로 표현 변경
- 해싱 최적화 단락: `null` 포함 시 `hashPair` 규칙은 BTIP16 참조로 정리
- 다이어그램 `leaf[1]: 0x00...00` → `leaf[1]: null`

---

### 2026-04-10

#### ✅ btip-24.md — cancelNullifier 추가 및 설명 정비

- `cancelNullifier(bytes32 eventRootHash)` 함수 인터페이스 추가
- 인터페이스 주석, 설명 bullet, Implementation 슈도코드 및 설명 추가
- "dApp에서의 Nullifier 취소" 섹션 추가 (호출 흐름 다이어그램 포함)
- Conclusion에 `cancelNullifier` msg.sender 기반 접근 제어 설명 추가

#### ✅ btip-26.md — cancelLinkerEvent 추가 및 설명 정비

- `cancelLinkerEvent(bytes32 eventRootHash)` 함수 인터페이스 추가
- 파라미터 설명 추가 (`eventRootHash`: 취소할 이벤트의 머클 트리 루트 해시)
- 내부에서 `BTIP24`의 `cancelNullifier` 호출함을 명시

#### ✅ 스펙 참조 표기 규칙 강화 — BTIP21(LinkerEndpoint) 형식 (btip-23, 24, 26)

- **규칙**: 구현체명(LinkerEndpoint 등)은 보조적 표기로만 사용, 메인 이름은 `BTIPxx`
- 표기 형식: `BTIP21(LinkerEndpoint)` (산문), `[BTIP21(LinkerEndpoint)](./btip-21.md)` (링크 포함)
- btip-23: `LinkerVerifier` → `BTIP23(LinkerVerifier)`, `LinkerEndpoint` → `[BTIP21(LinkerEndpoint)](./btip-21.md)`, `LinkerPolicy` → `[BTIP22(LinkerPolicy)](./btip-22-xx.md)`
- btip-24: `LinkerNullifier` → `BTIP24(LinkerNullifier)`, `LinkerEndpoint` → `[BTIP21(LinkerEndpoint)](./btip-21.md)` (산문 6곳 + 섹션 헤딩)
- btip-26: `LinkerEndpoint` → `[BTIP21(LinkerEndpoint)](./btip-21.md)` (산문 2곳)
- 코드 블록 내부 주석, 자기 문서 참조는 변경 제외

#### ✅ README.md — 실제 파일 기준으로 테이블 재정비

- 존재하지 않는 파일 항목 제거: BTIP15, BTIP17-draft, BTIP18, BTIP22
- BTIP25, BTIP26 항목 추가
- 타이틀 실제 frontmatter 기준으로 교정 (BTIP19, BTIP20 등)

---

### 2026-04-06

#### ✅ TxEventProofPayload에 `mspids` 필드 추가 — 조직별 Root CA 조회

- `TxEventProofPayload`에 `mspids: Array of String` 필드 추가 (btip-19)
- `mspids[i]`로 `cert_chains[i]`에 대응하는 보증 피어의 조직 MSP ID를 식별
- `verify_event_proof` Step 2: `LinkerPolicy.get_root_ca(payload.mspids[i])`로 조직별 Root CA 조회
- `create_event_proof`: `mspids = [e.mspid for e in target_tx.endorsements]` 추출 추가
- `block_commit_sigs` 설명에 보증 피어 자격 서명만 유효함을 명시

#### ✅ btip-21.md — TxEventProof Solidity struct에 `mspids` 추가

- `string[] mspids` 필드 추가 (보증 피어 조직 MSP 식별자 배열)
- `mspids[i]` 필드 설명 추가

#### ✅ btip-23.md — getRootCA(mspid) 참조 수정

- `getRootCA()` → `getRootCA(mspid)` — 해당 조직의 Root CA 조회로 변경

#### ✅ btip-20.md — 조직별 Root CA 체계 반영

- LinkerVerifier 인증서 체인 검증 설명: `mspids[i]` 기반 조직별 Root CA 조회 반영
- LinkerPolicy Root CA 보관 설명: 조직(MSP)별 보관으로 변경
- ProofBuilder 설명에 `mspids` 추가
- 시퀀스 다이어그램: `getRootCA()` → `getRootCA(mspid)` 변경

---

### 2026-04-05

#### ✅ BTIP26 (btip-26.md) 신규 생성 — ILinkerEventHandler 분리

- btip-21에서 정의하던 `ILinkerEventHandler` 인터페이스를 BTIP26으로 분리
- `interface BTIP26 { function handleLinkerEvent(uint256 eventType, uint256[] calldata index, bytes[] calldata data) external; }`
- btip-10이 Linker Protocol V1의 dApp 인터페이스인 것과 동일한 위치 (V2 dApp 인터페이스)
- btip-21에서는 BTIP26 참조 섹션으로 대체
- btip-21 슈도코드/설명의 `ILinkerEventHandler` → `BTIP26` 교정
- btip-20의 `ILinkerEventHandler` 참조 제거 — 역할 중심 서술로 변경

#### ✅ btip-20.md — 구체적 인터페이스명 제거

- btip-20은 추상 설계 문서이므로 구체적 함수명/인터페이스명을 가급적 피함
- `receiveProof`, `BTIP26`, `ILinkerEventHandler` 등 구체 이름 제거
- "별도로 정의되는 콜백 인터페이스를 구현해야 한다" 수준으로 서술

#### ✅ 스펙명 표기 규칙 일괄 교정 (전체 14개 파일)

- **규칙**: 파일 이름에만 하이픈 사용 (`btip-21.md`), 스펙 이름으로는 하이픈 없이 표기 (`BTIP21`, `btip21`)
- frontmatter `requires:` 필드: `btip-19` → `btip19`
- 마크다운 링크 표시 텍스트: `[BTIP-19](./btip-19.md)` → `[BTIP19](./btip-19.md)` (URL은 유지)
- 본문/코드 주석 내 참조: `BTIP-19` → `BTIP19`
- 대상: btip-14, 15-xx, 17, 18-xx, 19, 20, 21, 22-xx, 23, 24, 25, 26, README

---

### 이전 세션

#### btip-19.md — Proof Construction Algorithm 수정

- 함수 시그니처: `create_event_proof(endorser_blocks, tx_index)` — `target_tx`, `target_block` 제거
- `endorser_blocks[0]`에서 트랜잭션과 블록 데이터 추출
- commit_sigs 수집: 각 보증 피어의 블록 `metadata[5]`에서 개별 수집하는 구조로 변경
- 변수명 교정: `event_root` → `event_log_root`, `event_log` → `event_log_tree`
- `[!TIP]` 인용문 추가: 사전 검증(동일 tx_index, 동일 block_event_root, commit_sigs 정합성)

#### btip-19.md — Verification Procedure 재구성

- Step 순서 변경: 보증 정책 대조(Step 3) → 블록 커밋 서명 검증(Step 4) (비용 최적화)
- `verify_cert_chain` → `(serial_no, pubkey, ou)` 반환, `endorsers` 리스트 수집
- `extract_pub_key` 제거 → `endorser.pubkey` 직접 사용
- 정책 대조: `endorser_count`, `ous`, `serial_nos`로 판단
- CRL 확인 추가: `is_revoked(result.serial_no, crl)` Step 2에 포함
- 다이어그램: 슈도코드 바로 아래, Step 단위 신뢰 확보 중심

#### btip-19.md — Trust Establishment 통합 및 제거

- Trust Establishment 섹션의 상세 논증(①~⑥)을 Verification Procedure 불릿에 통합
- Trust Establishment 섹션 자체 제거
- 다이어그램은 Verification Procedure 슈도코드 아래로 이동

#### btip-19.md — Security Considerations 제거

- 섹션 제거, 하위 단락을 `[!NOTE]` 인용문으로 대체 (Collusion Attack, Gas Cost)

#### 필드명 일괄 교정 (btip-19, 20, 21, 23)

- `event_log_proof` → `event_log_root_proof` (21건 + 3건 + 3건 + 9건)
- `commit_sigs` → `block_commit_sigs` (10건 + 2건 + 2건 + 2건)

#### 해시 함수 변경 (btip-20, 21, 23, 24)

- `keccak256` → `sha256` 전체 변경

#### 함수명 교정 (btip-20, 21, 23, 24)

- `verifyAll` → `verifyProof`
- `setPolicy` → `setPolicyContract`
- `setNullifier` → `setNullifierContract`
- `setVerifier` → `setVerifierContract`

#### btip-21.md — 인터페이스 정리

- `onLinkerDeliver` → `handleLinkerEvent`로 이름 변경

#### btip-23.md — 인터페이스 및 구현 정리

- 인터페이스: 내부 함수 제거, `verifyProof`과 `setPolicyContract`만 유지
- Implementation: 슈도코드 제거, 텍스트 기반 설명으로 대체
  - btip-19의 `verify_event_proof` Step 2~6 구현 (Step 1은 LinkerEndpoint 책임)
  - LinkerPolicy(btip-22) 메소드: `getRootCA(mspid)`, `getCRL()`, `getEndorsementPolicy()`
  - Precompile `0xff00`: 입력 순서 `[subject, issuer]`로 교정 (기존 `[issuer, cert]`는 오류)
  - Precompile `0x0100`: P-256 서명 검증

#### btip-24.md — 인터페이스 정리

- `contract LinkerNullifier` → `interface BTIP24`
- `mapping`, `modifier` 등 상태 변수 제거, 함수 시그니처만 유지
- `endorser_sigs` → `block_commit_sigs` (구 모델 용어 교정)

#### btip-20.md — 다이어그램 수정

- Verifier Smart Contract System 다이어그램: `subgraph` 구조 + 색상 구분
- `verifyAll` → `verifyProof` 반영

---

## 파일 상태 요약

| 파일 | 상태 | 현행 주요 구조 |
|------|------|---------------|
| `btip-16.md` | ✅ 수정 완료 | null 기반 머클 트리 패딩, hashPair 규칙 정의 |
| `btip-17.md` | ✅ 수정 완료 | 실패 트랜잭션 리프 null, hashPair 규칙 BTIP16 참조 |
| `btip-19.md` | ✅ 수정 완료 | MerkleProof\<Leaf\>, mspids, cert_chains, block_commit_sigs, event_log_root_proof, event_elem_proofs |
| `btip-20.md` | ✅ 수정 완료 | verifyProof, subgraph 다이어그램, sha256, getRootCA(mspid), 조직별 Root CA 체계 |
| `btip-21.md` | ✅ 수정 완료 | BTIP21 interface, string[] mspids, bytes[][] cert_chains, BTIP26 참조 섹션 |
| `btip-22-xx.md` | ⚠️ 미수정 | 조직별 Root CA 매핑 반영 필요 |
| `btip-23.md` | ✅ 수정 완료 | BTIP23 interface (verifyProof + setPolicyContract), getRootCA(mspid) |
| `btip-24.md` | ✅ 수정 완료 | BTIP24 interface (isProcessed + markProcessed) |
| `btip-25.md` | ✅ 신규 | TransferEventElems 정의 |
| `btip-26.md` | ✅ 신규 | BTIP26 interface (handleLinkerEvent) — dApp 콜백 인터페이스 |
| `btip-27.md` | ✅ 신규 | BPuN Event Structure — 2단계 머클 트리(`event_attrs_root` → `tx_event_root`), Tendermint 합의 통합 |
| `btip-28.md` | ✅ 신규 | BPuN Tx/Event Proof — 다이제스트 페이로드(`last_results_proof`, `tx_result_proof`, `event_attrs_root_proof`, `event_attr_proofs`), Validator 서명 검증 |
| `btip-29.md` | ✅ 신규 | LinkerEndpoint Chaincode on BPrN — Gateway, `OnProof`, Go 구조체 (BTIP21 대응) |
| `btip-31.md` | ✅ 신규 | LinkerVerifier Chaincode on BPrN — BTIP28 Step 2~6 검증 (BTIP23 대응) |
| `btip-32.md` | ✅ 신규 | ValidatorSetRegistry Chaincode on BPrN — Validator Set 블록 높이별 보관 (BTIP22 대응) |
| `btip-33.md` | ✅ 신규 | LinkerNullifier Chaincode on BPrN — 리플레이 방어, `sha256(txEventRoot\|\|targetApp)` (BTIP24 대응) |

---

## 표기 규칙

- **스펙 이름**: `BTIP21`, `btip21` (하이픈 없음)
- **파일 이름**: `btip-21.md` (하이픈 있음)
- **마크다운 링크**: `[BTIP21](./btip-21.md)` (표시 텍스트는 스펙명, URL은 파일명)
- **frontmatter requires**: `requires: btip21` (스펙명 규칙 따름)

---

## verify_event_proof 현행 흐름 요약 (btip-19)

```python
def verify_event_proof(payload: TxEventProofPayload):
    # Step 1. Nullifier 중복 검사
    # Step 2. 인증서 체인 검증 + CRL 확인 → mspids[i]로 조직별 Root CA 조회 → (serial_no, pubkey, ou) 수집
    # Step 3. 보증 정책 대조 (endorser_count, ous, serial_nos)
    # Step 4. 블록 커밋 서명 검증 → block_event_root 신뢰 확보
    # Step 5. 블록 이벤트 머클 증명 검증 → event_log_root 신뢰 확보
    # Step 6. 이벤트 머클 증명 검증 → 개별 이벤트 존재 확정
```

---

## 시스템 구성 참고

### BEATOZ EVM Precompiles
- `0x0100` — P-256 ECDSA 검증
- `0xff00` — X.509 인증서 체인 검증 (입력: [subject, issuer] 순서)

### 프로토콜 구조
- **BPrN**: Hyperledger Fabric 기반 프라이빗 네트워크
- **BPuN**: Tendermint/DPoS 기반 퍼블릭 네트워크 (소스: `beatoz-go`, Tendermint v0.34.24 포크)
- **BEATOZ Linker Protocol V2**: 양방향 크로스체인 이벤트 전달 프로토콜
  - BPrN → BPuN: PKI 기반 (btip-16~26)
  - BPuN → BPrN: PoS 합의 기반 (btip-27~28)

### beatoz-go 소스 코드 참조 (BPuN 관련)

| 파일 | 핵심 내용 |
|------|----------|
| `ctrlers/types/trx_ctx.go` | `eventRootEx()` — 2단계 머클 트리 구성 (lines 162-182) |
| `types/merkle/tree.go` | 배열 기반 완전 이진 트리, `hashPair`, `Proof()`, `VerifyProof` |
| `node/app.go` | `asyncExecTrxContext()` — event_root → `ResponseDeliverTx.Data` (lines 485-570) |
| `ctrlers/types/trx.go` | 이벤트 속성 상수 (EVENT_ATTR_TXSTATUS 등), 트랜잭션 유형 |
| `ctrlers/vm/evm/ctrler.go` | `evmLogsToEvent()` — EVM 로그 → ABCI Event 변환 (lines 280-398) |
| `ctrlers/types/trx_ctx_test.go` | `Test_TrxContext_EventRootEx` — 2단계 머클 트리 검증 테스트 (lines 211-301) |
| `go.mod` | Tendermint 포크: `beatoz/tendermint-ethaddr v0.34.24` |

### 관련 문서 의존성
```
BPrN → BPuN 방향:
  btip16 ← btip17 ← btip19 ← btip20
                             ← btip21 (LinkerEndpoint)
                                  ← btip26 (dApp 콜백 인터페이스)
                             ← btip23 (LinkerVerifier)
                             ← btip24 (LinkerNullifier)

BPuN → BPrN 방향:
  btip27 ← btip28 ← btip29 (LinkerEndpoint)
                    ← btip31 (LinkerVerifier)
                         ← btip32 (ValidatorSetRegistry)
                    ← btip33 (LinkerNullifier)

BPuN ↔ BPrN 대칭 매핑:
  BTIP21 (LinkerEndpoint on BPuN)    ↔ BTIP29 (LinkerEndpoint on BPrN)
  BTIP22 (LinkerPolicy on BPuN)     ↔ BTIP32 (ValidatorSetRegistry on BPrN)
  BTIP23 (LinkerVerifier on BPuN)   ↔ BTIP31 (LinkerVerifier on BPrN)
  BTIP24 (LinkerNullifier on BPuN)  ↔ BTIP33 (LinkerNullifier on BPrN)
```
