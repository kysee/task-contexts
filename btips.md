---
last_updated: 2026-05-28
---

# BTIPS 작업 컨텍스트

---

## 작업 개요

BEATOZ Linker Protocol V2의 양방향 크로스체인 증명 체계 설계.

- **BPrN → BPuN 방향** (btip-16~26): BTIP19의 단일 경로 신뢰 모델 재작성, 필드명 시맨틱 교정, 제네릭 타입 표기법 도입, Proof Construction/Verification 재구성, 인터페이스 정리 등 완료.
- **BPuN → BPrN 방향** (btip-27~34): BPuN 이벤트 구조, 2단계 머클 트리, Tendermint 합의 연계, 다이제스트 증명 페이로드 설계 완료. 변수명 직관성 리팩터링 완료. EventAttrProof 제거(MerkleProof로 통합), Nullifier eventAttrsRoot 기준 재설계 완료.

---

## 핵심 기술 개념

### 단일 경로 신뢰 모델 (Single-Path Trust Model)

```
Root CA → 인증서 체인 → 블록 커밋 서명 → block_event_root → event_log_root → 개별 이벤트
```

### BPrNTxEventProofPayload (현행 — btip-19 기준)

```text
Structure MerkleProof<Leaf>:
    Property index:    Integer
    Property leaf:     Leaf
    Property siblings: Array of ByteArray

Structure BPrNTxEventProofPayload:
    Property block_number:          Integer
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
- `block_number` 필드 추가 (블록 커밋 서명 입력에 포함, 크로스체인 트랜잭션 식별용)

**Solidity 대응 (btip-21):**
```solidity
struct MerkleProof {
    bytes     leaf;
    uint256   index;
    bytes32[] siblings;
}

struct TxEventProof {
    uint64        block_number;
    string[]      mspids;
    bytes[][]     cert_chains;
    bytes[]       block_commit_sigs;
    bytes32       block_event_root;
    MerkleProof   event_log_root_proof;
    MerkleProof[] event_elem_proofs;
}
```

### Commit Signature (btip-17 기준)

- 서명 입력: `block_height || block_event_root` (block_height는 8바이트 big-endian)
- 각 피어의 블록 `metadata[5]`에는 해당 피어 자신의 커밋 서명만 존재
- n개의 서명을 수집하려면 n개의 보증 피어에게 각각 블록을 조회해야 함
- `block_number`가 서명 입력에 포함되므로 암호학적으로 검증 가능

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
Structure RFC6962Proof:
    Property total:   Integer        // RFC 6962 트리의 전체 리프 수 (분할 지점 결정에 필요)
    Property index:   Integer
    Property leaf:    ByteArray
    Property aunts:   Array of ByteArray

Structure MerkleProof<Leaf>:
    Property index:    Integer
    Property leaf:     Leaf
    Property siblings: Array of ByteArray

Structure ValidatorSignature:
    Property validator_address:  ByteArray
    Property timestamp:          Timestamp   // CanonicalVote 서명 입력에 포함됨
    Property signature:          ByteArray

Structure BPuNTxEventProofPayload:
    Property height:                   Integer
    Property chain_id:                 String
    Property round:                    Integer
    Property block_hash:               ByteArray
    Property block_parts_total:        Integer
    Property block_parts_hash:         ByteArray
    Property validator_sigs:                Array of ValidatorSignature
    Property last_results_proof:       RFC6962Proof
    Property tx_result_proof:          RFC6962Proof
    Property event_attrs_root_proof:   MerkleProof<event_attrs_root>
    Property event_attr_proofs:        Array of MerkleProof<attr_value>
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
| `event_attr_proofs[i]` | `event_attrs_root` | `value` (속성값 raw bytes) | `sha256(value)` | BTIP27 단순 연결 |

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

### 2026-06-01 (publish 폐기 결정 + BTIP-37 LINKER_CCS 제거 + BTIP-26 명문화)

> 본 세션은 **설계 마감**을 우선한 정리 세션. `bpun-origin-payment-design.md`의 `LinkerEndpoint.publish` 제안을 폐기하고, 그 후속으로 BTIP-37 `LINKER_CCS` NOTE 제거 + BTIP-26에 수신 측 권한 모델·direct emit 모델 명문화. BPrN-origin 대칭(BTIP-34)은 다음 작업.

#### ✅ `LinkerEndpoint.publish` 제안 폐기 (설계)

`bpun-origin-payment-design.md` §15 신규 작성으로 폐기 결정 정리. 핵심 전제: **"ccApp(BPrN 수신 비즈니스 체인코드)은 신뢰 가능한 emitter dApp 목록을 관리하지 않는다."** 이 전제 하에서 publish가 보장하던 가치 4가지(단일 정규 contractAddress, `requestedBy = msg.sender` 비위조 캡처, 이벤트 스키마 표준화, caller chain 캡처)가 모두 `contractAddress`(evm 이벤트 index 0, EVM 강제 비위조) + 수신측 권한 3분기 모델로 대체됨 → publish 자체 불요.

폐기 결정의 검증 케이스(설계 노트 §15.5 참조):
- EOA payer: Alice가 BPrN에서 `stc.approve(dApp_BPuN_addr, amount)` 사전 인가 → BPuN dApp이 emit X → `contractAddress = dApp`, `allowance[Alice][dApp] >= amount` 통과 → 차감. 악의적 Mallory가 `emit X(payer=Alice, ...)` → `contractAddress=Mallory`, `allowance[Alice][Mallory]=0` → 거부.
- dApp payer (자기 자금): `payer == requestedBy = contractAddress` → ETH 기본 룰로 즉시 통과.

영향 받은 OPEN 항목: bpun-origin-payment-design.md §10의 (1)(2)(6)(7), §14의 (a)(b)(c)(d)(h) 해소. (3)(5)(8)(9)(e)(f)(g) 잔존.

#### ✅ btip-37.md — LINKER_CCS role NOTE 제거

`bpun-origin-payment-design.md` §15.6 항목 4 적용. publish 폐기로 "상대 체인 어플리케이션을 cooperative 발신 주체로 등록"하는 derived role 자체가 무의미. L48-58 NOTE 블록 한 개 삭제. 본문 인터페이스(`getContract`/`setContract`)·이벤트·에러·Appendix Go는 변경 없음 — LinkerRegistry의 본질(`(chainId, role)` 단일 키 모델)은 유지.

#### ⚠️ btip-26.md / btip-34.md 명문화 시도 → 정정 (같은 세션, 2026-06-01)

§15.6 초안의 항목 1·2(BTIP-26/34에 권한 3분기·`requestedBy` 매핑·"신뢰 발신자 목록 금지" 명문화)를 일단 진행했으나 사용자 redirect로 **두 블록 모두 삭제**. 정정 근거:
- **BTIP-26/34는 일반 LinkerApp 인터페이스 스펙**. `payer`/`requestedBy`/`approve`/`allowance`는 *결제 이벤트의 도메인 용어*이므로 이벤트 정의 BTIP 없이 인터페이스 스펙에 박으면 다른 use case(NFT mint·거버넌스 투표 등)의 LinkerApp에 결제 도메인 가정을 강제하게 됨.
- "신뢰 발신자 목록 관리 금지"는 *원래 없던 것을 부정형으로 명시*하는 잘못 — 화이트리스트는 논의 과정에 잠시 도입 검토됐다가 폐기된 것이고, 폐기 결과는 *문서에 없는 상태가 정상*.
- `requestedBy`는 추상화. 이 매핑(`requestedBy = contractAddress` 또는 `chaincode_id`)이 의미를 가지려면 *결제 이벤트의 어떤 필드가 무엇인지* 정의된 BTIP가 선행되어야 함. 결제 이벤트 정의 BTIP 부재 상태에서 매핑만 박은 건 잘못된 작업 순서.

**결과**:
- BTIP-26: 추가했던 IMPORTANT NOTE "수신 측 권한 모델" + "요청 이벤트 발신 — direct emit" 절 *모두 삭제*. 본 세션 시작 전 상태로 복귀.
- BTIP-34: 동일 두 블록 *모두 삭제*. 본 세션 시작 전 상태로 복귀.
- `bpun-origin-payment-design.md` §15.6 정정: BTIP-26/34 변경 작업 *불요*로 결정. 권한 3분기 모델은 §15.4 use case 가이드에 한정 보존. 별도 BTIP로 명문화하려면 *결제 이벤트 정의 BTIP가 선행*되어야 함을 명시.
- §15.9 다음 작업 우선순위에서 "BTIP-26/34 명문화" 항목 제거. 새 항목: 결제 이벤트 정의 BTIP(선결 조건) 또는 2PC 코드 구현.

**교훈** (재발 방지):
- BTIP 본문에 박을 내용은 *해당 BTIP의 책임 범위*에 속하는 것만. LinkerApp 인터페이스 스펙에 결제 도메인 권한 모델을 박으면 안 됨.
- 부정형 의무(*~하지 말 것*)는 *원래 있던 것을 폐기한 경우*에만 강조 목적으로 사용. 처음부터 없던 것은 박지 않음.
- 추상화 용어(`requestedBy` 등)는 *구체 정의를 가진 데이터 모델* 위에서만 도입. 데이터 모델 BTIP가 없으면 그 용어를 도입할 자리가 없음.

#### ✅ btip-25.md / btip-40.md — LinkerTransfer 이벤트 표준 정의 (양 체인)

publish 폐기 후속으로 결제 트리거 이벤트의 표준을 *이벤트 정의 BTIP*에 명문화. §15.6 정정에서 "결제 이벤트 정의 BTIP 선행"이라 했던 BTIP 한 쌍 신설.

**결정 사항**:
- **이벤트 이름**: 양 체인 동일 — `LinkerTransfer`. `Linker` prefix로 ERC-20 `Transfer`와 selector 분리.
- **자료구조명**: BPrN은 `TransferLogElems`(EventLog.elems 스키마), BPuN은 `TransferLogAttrs`(Per-Event Tree attribute 시퀀스). 자료구조명에는 `Linker` prefix를 *두지 않음* — 사용자 결정: 이벤트 정의는 3rd party도 가능하므로 strict한 네이밍 규칙을 강요하지 않는다.
- **Event Log 통합 어휘**: 링커가 다루는 이벤트는 발생 체인 무관하게 통칭 *Event Log*로 부른다. 각 체인의 직렬화 형식 차이는 자료구조명의 접미사(`...LogElems` / `...LogAttrs`)로 드러내고, 이벤트 이름과 필드 시퀀스는 공통. BTIP-25/40 본문에 짧은 NOTE로 명시.
- **필드 시퀀스**: `from, to, amount, correlationId, memo` 양 체인 동일.
- **필드명**: `payer/payee` 대신 `from/to`(더 노멀).
- **correlationId**: 양 체인 모두 명시 박음 — BPrN-origin도 fire-and-forget 없이 항상 2PC 가정. 발신 (cc)App이 정한 매칭 식별자(예: 단조 증가 nonce). *이전 결정* `correlationId = tx_event_root`(btips-2pc-design §5)에서 *명시 id로 전환*. Nullifier 기준(=`tx_event_root`)과 correlationId(=매칭 식별자)가 분리됨 — 의미가 다르므로 분리되어도 무방.
- **handler 필드 부재**: handler(targetDApp/executor)는 *증명 제출 시* 지정되므로 발신 이벤트에 박지 않음. 양 체인 동일.

**파일 변경**:

1. **btip-25.md (rename + 필드 갱신)**:
   - 제목: `TransferEventElems on BPrN` → `LinkerTransfer Event on BPrN`
   - 자료구조명: `TransferEventElems` → `TransferLogElems` (`Linker` prefix 없음)
   - selector: `sha256("TransferEventElems(bytes,bytes,uint256,bytes)")` → `sha256("LinkerTransfer(bytes,bytes,uint256,bytes32,bytes)")`
   - 필드 추가: `CorrelationId: bytes32` (gidx 7, elems index 3). 기존 `Memo`는 gidx 8(elems index 4)로 이동.
   - 머클 트리: 8 leaf 완전 이진 트리 → 9 leaf + null 패딩 16 leaf 트리(BTIP-16 null 패딩 규칙).
   - BTIP-40 cross-reference 추가.
2. **btip-40.md (신규)**:
   - `event LinkerTransfer(address indexed from, address indexed to, uint256 amount, bytes32 indexed correlationId, bytes memo)` Solidity 시그니처.
   - 3 indexed(from, to, correlationId) + 2 non-indexed(amount, memo).
   - BTIP-35 evm 이벤트 attribute 구조 위에 매핑: Per-Event Tree index 0~7(contractAddress/topic.0/topic.1/topic.2/topic.3/data/blockNumber/removed).
   - `TransferLogAttrs` 자료구조명 도입(`Linker` prefix 없음).
   - "Event Log" 통합 어휘 NOTE 본문 포함.
   - BTIP-25와 대칭 표 명시(이벤트 이름·필드 시퀀스 동일, 자료구조명·selector 해시·발신 형식 차이는 각 체인 표현 형식의 본질).
3. **BTIPS/README.md**:
   - BTIP-25 행 제목 갱신.
   - BTIP-40 신규 등재.

**btips-2pc-design.md 영향** (별도 정정 필요 — 본 세션에서 추적만, 다음 세션에 정정):
- §5 결정(correlationId = `tx_event_root`)이 본 결정으로 *명시 id 전환*. §5 본문 갱신 필요. Nullifier 기준(`tx_event_root`)과 correlationId의 의미 분리도 함께 기록.
- §6 OnResult 분리·라우팅 의미는 영향 없음(여전히 발신 (cc)App으로 라우팅, correlationId가 매칭 키).

#### ✅ btip-21.md — setter 정리 + LinkerRegistry 조회 통합 (BTIP-29 대칭)

BPrN 측 [BTIP-29](./btip-29.md)는 2026-05-28에 이미 `SetVerifierChaincodeID`/`SetNullifierChaincodeID` 제거 → `SetRegistryID` 단일화가 적용됐으나, BPuN 측 BTIP-21에는 미적용 상태였음. 본 세션에서 정합.

**변경 내역**:
- `requires`: `btip20` → `btip20, btip37`.
- `setNullifierContract`/`setVerifierContract` 2개 setter *제거*.
- 단일 부트스트랩 setter `setRegistry(address registry)` 신설 — `Ownable.onlyOwner`. 모든 다른 Linker 컴포넌트는 LinkerRegistry를 통해 `(bytes32(block.chainid), role)` 키로 동적 조회.
- `onProof` 슈도코드: "컴포넌트 조회" 단계 추가 (`BTIP37(_registry).getContract(self_chain, LINKER_VERIFIER/LINKER_NULLIFIER)`).
- `onResult` 슈도코드: 동일 + 추가로 `getContract(BPRN_CHAIN_ID, LINKER_ENDPOINT)` 조회로 출처 검증.
- 기존 `> [!NOTE]` LinkerRegistry 진본 NOTE 제거(이제 본 메소드 자체가 LinkerRegistry 사용을 강제).
- **owner 모델 NOTE 추가**: `Ownable` 수준의 단순 owner 검증을 적합성 기준으로 명시. 사용자 결정 — Linker를 모듈별로 구성한 설계 의도(Nullifier/Verifier/Policy/Registry까지 모두 *주소 변경*으로 모듈 업그레이드 가능)에 immutable이 맞지 않음. multisig+timelock 같은 강한 거버넌스는 *별도 BTIP*로 정의될 수 있다고 NOTE로 미래 확장 여지 남김.

**근거**: LinkerRegistry가 `(chainId, role) → address` 단일 출처를 제공하는 이상, LinkerEndpoint가 LinkerVerifier·LinkerNullifier 주소를 자기 상태로 저장할 필요가 없음. LinkerRegistry 한 곳만 알면 거버넌스가 LinkerRegistry에서 컴포넌트 주소를 재지정해도 별도 마이그레이션 없이 즉시 반영됨.

**linker-v2 코드 갭**: `IBTIP21.sol`(L34/37)·`LinkerEndpoint.sol`(L50/56)에 두 setter가 잔존, `setRegistry` 부재. `task-contexts/linker-v2.md` §"2026-06-01 — 재동기화" §1.B 표에 항목 추가.

#### ✅ btip-21.md / btip-26.md — 재진입·가스 항목 정정 + LinkerAppLowGas 권장 패턴

기존 BTIP-21 "재진입·가스" 항목이 *부당 REJECTED 영구 기록*을 완전 방어하지 못함을 토론으로 확인. 해당 항목이 *거짓 안전감*만 주는 상태였음. 정정 방향:

**부당 REJECTED griefing 분석** (BTIP-21 본문에 명시 추가):
- 자금 손실 ❌ (BPrN에서 REFUNDED로 환불, 두 체인 상태 일관)
- *서비스 가용성 공격* ⚠️ — `onProof`가 permissionless이므로 제3자가 낮은 gas limit으로 호출해 dApp을 OOG로 죽이고 영구 REJECTED 강제 가능. 판매자에게 결제 전달 안 됨.

**EVM의 본질적 제약**: 2PC의 *exactly-one-result*와 가스 보호는 본질적으로 충돌. try/catch가 OOG와 비즈니스 revert를 구별 못 함. 완전 방어 불가능.

**정정 결정** (사용자 redirect):
- BTIP-21에서 `MIN_CALLBACK_GAS` + `CATCH_RESERVE` + 사전 require 패턴 *제거*. 부당 REJECTED를 완전히 막지 못하면서 거짓 안전감만 줌.
- 대신 *부당 REJECTED 위험을 솔직히 명시* — 자금 안전·서비스 가용성 분리, EVM 한계, 호출자·dApp 책임 영역 분명히.
- dApp 협조 패턴으로 *명시적 가스 부족*은 완화 가능 — `LinkerAppLowGas` custom error를 BTIP-26 표준으로 도입.

**btip-21.md 변경**:
- "재진입·가스" 단일 항목 → "재진입 가드" + "가스 처리와 한계" 두 항목으로 분리.
- `MIN_CALLBACK_GAS`/`CATCH_RESERVE`/사전 require 모두 제거. EIP-150 63/64 룰이 catch 가스를 자연 보장한다는 사실 명시.
- 부당 REJECTED griefing 위험 솔직히 박음. dApp 협조 패턴(BTIP-26) cross-ref.
- onProof 슈도코드에 `except (BTIP26.LinkerAppLowGas): revert BTIP26.LinkerAppLowGas` 분기 추가 — 표준 가스 부족 신호 인식 시 *같은 에러를 그대로 전파*해 전체 revert(Nullifier 미등록).
- `error LinkerAppLowGas()` *BTIP-26에 단일 정의* — 이름의 의미 주체가 LinkerApp(=dApp)이므로 LinkerApp 인터페이스 BTIP에 정의가 자연. BTIP-21은 catch에서 `BTIP26.LinkerAppLowGas` selector 인식 후 같은 에러로 전파(import).

**btip-26.md 변경**:
- interface에 `error LinkerAppLowGas()` 표준 custom error 정의.
- "권장 패턴 — 명시적 가스 부족 신호" NOTE 추가:
  - dApp이 함수 시작에서 `if (gasleft() < MY_MIN_REQUIRED_GAS) revert LinkerAppLowGas();` 패턴 권장 (강제 아님).
  - LinkerEndpoint가 catch에서 이 신호 인식 → 전체 revert (Nullifier 미등록, 재시도 가능).
  - 그 외 revert(다른 custom error, Error string, Panic, EVM 암묵적 OOG)는 모두 일반 REJECTED.
  - 한계 솔직히: EVM 암묵적 OOG는 보호 못 함. *명시적* gas 부족 grief 차단에 한정.

**판단 근거**: `feedback-btip-scope-discipline` 메모리 적용 — BTIP에 *불완전한 보호 메커니즘*을 박는 것보다 *한계를 솔직히 명시*하는 게 안전. dApp이 자율 보호 옵션을 갖게 BTIP-26에 권장 패턴으로만 둠.

#### ✅ btip-23.md / btip-22.md — 보증 신뢰 검증을 LinkerPolicy로 위임 (BTIP-19 Step 2~4)

`BTIP-23`(LinkerVerifier on BPuN)이 [BTIP-19](./btip-19.md) `verify_event_proof`의 Step 2~6 책임이라는 사실은 변함 없으나, **Step 2~4(보증 신뢰)는 [BTIP-22](./btip-22.md)(LinkerPolicy)의 `verifyChannelEndorsementPolicy`에 단일 호출로 위임**하고 BTIP-23은 **Step 5~6(머클 증명)만 직접 수행**하도록 책임 분리 명문화.

**btip-23.md 변경**:
- `requires`: `btip19, btip20` → `btip19, btip20, btip22`.
- Abstract — 위임/직접 매핑 명시.
- Interface 설명 — verifyProof가 위임 진입점임을 명확화.
- Implementation 절 전면 재작성:
  - Step 2~6 매핑 표 추가 (각 Step의 담당이 BTIP-22 vs BTIP-23).
  - 슈도코드 — Step 2~4는 `LinkerPolicy.verifyChannelEndorsementPolicy(msg_hash, sigs, mspids, cert_chains)` 단일 호출, msg_hash는 BTIP-17 규약(`sha256(block_height_BE_8 || block_event_root)`). Step 5는 `block_event_root` 트리에서 `tx_event_root` 머클 검증, Step 6은 `tx_event_root` 트리에서 개별 이벤트 요소 검증.
  - precompile 설명 제거 — BTIP-22가 모든 precompile(0xff00/0x0100)을 호출하므로 본 BTIP는 직접 호출 안 함. 상세는 BTIP-22 참조.
- Conclusion — 책임 단위 분리 의의 명시(정책 의존 검증 vs 정책 무관 순수 해시 검증).

**btip-22.md 변경**:
- Abstract 직후에 NOTE 추가 — BTIP-19 `verify_event_proof` Step 2/3/4와 본 함수 내부 Step 1/3/2(인증서/정책/서명)의 매핑 표. 본 함수의 *Step 순서가 BTIP-19와 다른 이유*(서명 검증을 인증서 추출 결과 위에서 수행 → 정책 평가 입력) 설명. 외부 시각에서는 *Step 2~4의 원자적 위임 수행*과 동등.

**잔존 미해결 (별도 작업)**:
- ~~BTIP-23의 `setPolicyContract` setter는 BTIP-21의 `setRegistry` 패턴(LinkerRegistry 동적 조회)과 정합 안 됨.~~ — 같은 세션 내 처리(아래 BTIP-23/24 setter 정합 항목 참조).

#### ✅ BTIPS 전체 — 인터페이스/문서 표기 규약 정정

표기 규약 명문화: *문서로서 지칭*은 `BTIPnn`, *인터페이스로서 지칭*은 `IBTIPnn`. 일부 파일이 이미 따르고 있으나 본 세션 결과 전반 정합화.

**규약 적용 기준**:
- `[BTIPnn](./btip-nn.md)` — 문서 링크: 그대로.
- `BTIPnn(LinkerXxx)` (괄호 안에 자료구조/컴포넌트명, 메소드 호출 없음) — 문서 지칭: 그대로.
- 주석의 `// BTIPnn(LinkerXxx)` — 문서 지칭: 그대로.
- `interface BTIPnn {` / `type BTIPnn interface {` — *인터페이스 정의*: `IBTIPnn`으로 정정.
- 코드 블록 안 `BTIPnn(varname).method(...)` — *인터페이스 캐스팅*: `IBTIPnn`으로 정정.
- prose 안 백틱 코드 `msg.sender == BTIPnn(LinkerXxx).method(...)` — *인터페이스 사용*: `IBTIPnn`으로 정정.

**변경 파일**:

| 파일 | 변경 |
|---|---|
| btip-21.md | 슈도코드 캐스팅 BTIP37/23/24/26 → IBTIP37/23/24/26 (10+ 곳) |
| btip-23.md | `interface BTIP23` → `IBTIP23`, 슈도코드 IBTIP22/IBTIP37 캐스팅 정정 |
| btip-24.md | `interface BTIP24` → `IBTIP24`, 슈도코드 IBTIP37 캐스팅 정정 |
| btip-26.md | `interface BTIP26` → `IBTIP26`, prose 코드 인용 BTIP37 → IBTIP37 (2곳) |
| btip-29.md | `type BTIP29 interface` → `IBTIP29`, 슈도코드 IBTIP34 캐스팅 (2곳) |
| btip-31.md | `type BTIP31 interface` → `IBTIP31` |
| btip-32.md | `type BTIP32 interface` → `IBTIP32` |
| btip-33.md | `type BTIP33 interface` → `IBTIP33` |
| btip-34.md | `type BTIP34 interface` → `IBTIP34` |
| btip-37.md | `interface BTIP37` → `IBTIP37` |
| btip-39.md | `type BTIP39 interface` → `IBTIP39` |

**검증**: grep `^interface BTIP[0-9]\|^type BTIP[0-9]` 잔존 0건(DEPRECATED btip-10/11/12/13 제외), grep `BTIPnn(var).method` 캐스팅 잔존 0건.

DEPRECATED 파일(btip-10/11/12/13)은 historical 기록으로 보존 — 변경 없음.

#### ✅ btip-23.md / btip-24.md — setter 정합 (BTIP-21 패턴 적용)

[BTIP-21](./btip-21.md)에서 도입한 *모든 컴포넌트가 LinkerRegistry 한 곳만 알면 됨* 패턴을 BTIP-23/24로 확장. 모듈별 주소 변경을 LinkerRegistry 매핑 갱신으로 처리하는 일관된 설계.

**btip-23.md 변경**:
- `requires`: `btip19, btip20, btip22` → `btip19, btip20, btip22, btip37`.
- Interface: `setPolicyContract(address)` 제거 → `setRegistry(address)` 단일 setter. owner only(`Ownable.onlyOwner`).
- verifyProof 슈도코드: 진입 시점에 `policy = BTIP37(_registry).getContract(bytes32(block.chainid), LINKER_POLICY)` 동적 조회 후 그 주소로 `BTIP22(policy).verifyChannelEndorsementPolicy(...)` 호출.
- setRegistry 설명에 BTIP-21의 동일 메소드 cross-ref.

**btip-24.md 변경**:
- `requires`: `btip20` → `btip20, btip37`.
- Interface에 `setRegistry(address)` 추가. owner only.
- markProcessed의 onlyLinkerEndpoint 접근 제어를 LinkerRegistry 동적 조회로 명시 — `expected = BTIP37(_registry).getContract(bytes32(block.chainid), LINKER_ENDPOINT); assert msg.sender == expected`.
- 본문 설명에 "LinkerEndpoint 호출자 검증은 LinkerRegistry를 통해 동적 조회되므로 개별 setter는 두지 않는다" 명시. BTIP-21 cross-ref.

**근거**: 모든 BPuN-side Linker 컴포넌트(LinkerEndpoint/LinkerVerifier/LinkerNullifier)가 *LinkerRegistry 주소 하나만 자기 상태로 보유*. 다른 컴포넌트 주소·호출자 신원은 모두 LinkerRegistry 동적 조회로 처리. 거버넌스가 LinkerRegistry 매핑을 갱신하면 모든 컴포넌트가 *재배포 없이* 새 주소를 즉시 반영.

**linker-v2 코드 갭** (추가): `LinkerVerifier.sol`에 `setPolicyContract` 잔존, `LinkerNullifier.sol`에 `setLinkerEndpoint`(또는 동등) 잔존, 양쪽 모두 `setRegistry` 부재. `linker-v2.md` 다음 갱신 시 §1.B 표에 추가.

#### 본 세션 최종 BTIP 변경 요약

- ✅ **btip-37.md**: `LINKER_CCS` role NOTE 제거 (L48-58).
- ✅ **btip-25.md**: rename(`TransferEventElems` → `TransferLogElems`) + 제목 변경 + `CorrelationId` 필드 추가 + 머클 트리 갱신.
- ✅ **btip-40.md**: 신규 — `LinkerTransfer` Solidity event 표준 정의 (`TransferLogAttrs`).
- ✅ **btip-21.md**: setter 2개 제거 + `setRegistry` 단일화 + LinkerRegistry 동적 조회 패턴 통합 + 재진입·가스 항목 정정(MIN_CALLBACK_GAS 제거, griefing 위험 명시, LinkerAppLowGas 분기 추가).
- ✅ **btip-26.md**: `LinkerAppLowGas` 표준 custom error 신규 + 권장 보호 패턴 NOTE.
- ✅ **btip-23.md**: BTIP-19 Step 2~4 위임 매핑 명시 + Step 5~6 슈도코드 신설 + precompile 설명 제거 + setter 정합(`setPolicyContract` → `setRegistry`, LinkerRegistry 동적 조회).
- ⚠️ **btip-22.md**: 본 세션 추가 NOTE 시도 후 사용자 롤백 + precompile 정보 추가.
- ✅ **btip-24.md**: setter 정합(`setRegistry` 신설, onlyLinkerEndpoint를 LinkerRegistry 동적 조회로).
- ✅ **BTIPS/README.md**: BTIP-25 제목 갱신 + BTIP-40 등재.
- ❎ **btip-34.md**: 변경 *없음*(시도했던 두 블록 모두 삭제).

#### 다음 작업

- **결제 이벤트 정의 BTIP** 신설 여부 결정 (선결 — 권한 3분기를 BTIP에 박으려는 경우).
- **STC use case 미해결** (`bpun-origin-payment-design.md` §10-8): settle 행선지, PaymentBridge 분리, approve 동기화.
- **2PC 코드 구현** + BPuN→BPrN 이벤트 Prover(`u2r`). 권한 모델 BTIP 명문화와 독립.

### 2026-05-28 (BTIP-37 LINKER_CCS NOTE 추가 + Pay→publish 설계 논의)

> **2026-06-01 갱신**: 본 세션의 BTIP-37 LINKER_CCS NOTE는 2026-06-01에 제거됨. Pay→publish 토론은 publish 폐기 결정으로 historical 기록. 아래 본문 보존.

> 본 세션은 BTIP 문서 변경은 BTIP-37에 NOTE 한 블록 추가만 있고, 대부분은 *향후 설계*를 위한 토론. 전체 설계 토론은 `../task-contexts/bpun-origin-payment-design.md` §11~§14 참조 — 그쪽이 *자기완결적 설계 노트*다. 본 항목은 BTIP doc에 반영된 부분만 정리.

#### ✅ btip-37.md — LINKER_CCS role NOTE 추가 (minimal, BPuN-전용)

Roles 테이블 뒤(L48-58)에 NOTE 블록 한 개만 삽입. 본문 인터페이스(`getContract`/`setContract`)·이벤트·에러·Appendix Go는 **손대지 않음**.

**핵심 결정**:
- 새 메소드/이벤트/에러 0건 — 기존 `getContract`/`setContract` 그대로 재사용
- `LINKER_CCS`는 어플리케이션 주소별 **derived role**: `keccak256(abi.encodePacked("LinkerCCS", appAddr))`
  - 등록: `setContract(chainId, role_for(appAddr), appAddr)` — 거버넌스 전용
  - 조회: `getContract(chainId, role_for(appAddr))` — 등록되면 `appAddr` 반환
  - 한 chainId에 N>1 어플리케이션 등록 자연스럽게 지원 (각 derived role이 다름). 상대 체인이 명확한 운영주체의 프라이빗 네트워크라 N은 작음.
- **BPuN의 LinkerRegistry에만 존재** — BPrN의 LinkerRegistry에는 대칭 `LINKER_DAPPS` 없음. 근거: BPuN-origin 이벤트는 LinkerEndpoint 한 곳에서만 발생하므로 BPrN 수신측은 그 단일 컴포넌트 진본성만 확인하면 충분.
- 어휘 추상화: 본문에 `LinkerEndpointCC`, `ccApp` 같은 미정의 구체 용어 사용 금지. "상대 체인의 비즈니스 어플리케이션"으로 표현.

**경위**: 첫 시도는 `registerCCApp`/`unregisterCCApp`/`isCCAppRegistered` 3메소드 + 2events + 2errors 추가 (Abstract·Motivation·Conclusion·Appendix 전부 수정). 사용자 reject — "왜 이렇게 인터페이스가 많아", "미정의 용어 갑자기 나오면 안 됨", "그냥 LINKER_CCS role 하나만 추가". 5개 edit 전부 rollback 후 minimal NOTE-only로 재작성.

#### 진행 중 / 향후 BTIP 작업 (bpun-origin-payment-design.md §14 OPEN 참조)

> **2026-06-01 갱신**: 아래 5개 항목 중 publish 관련 4개는 `bpun-origin-payment-design.md` §15(2026-06-01)에서 publish 폐기 결정으로 *해소*. ccApp emitter 무신뢰 전제 + `requestedBy = contractAddress` 규약으로 publish 가치 4가지가 모두 대체됨. 새 작업 항목은 §15.9 참조.

- ~~`LinkerEndpoint.publish(...)` 스펙 (BPuN, BTIP-21 확장 또는 신규 BTIP)~~ — *해소.* publish 폐기.
- ~~`LinkerApp._lkPublish(...)` 스펙 (상속 base, BTIP-26 확장 또는 신규 BTIP)~~ — *해소.*
- ~~`LinkerEndpointCC.PreparePublish(...)` 스펙 (BPrN, BTIP-29 확장 또는 신규 BTIP)~~ — *해소.* ccApp이 종전대로 자기 `stub.SetEvent`로 emit.
- ~~`LinkerResult` event를 `LinkerPublish` event의 한 selector로 흡수 검토 (BPuN)~~ — *해소.* LinkerPublish 부재이므로 LinkerResult 그대로 유지.
- ~~BTIP-25 (`TransferEventElems`) deprecation 시점~~ — *불요.* BTIP-25 유지.

**새 작업 항목 (2026-06-01 결정 반영)**:
- BTIP-26/34에 Escrow Lifecycle Reference 절 신설/확장 — `requestedBy = contractAddress`(또는 BPrN-origin은 `Header.chaincode_id`) 규약, 권한 3분기(자기 자금 / `allowance` / EIP-712 서명), "신뢰 emitter 목록 관리 금지" 명문화.
- BTIP-37 §13.5의 LINKER_CCS NOTE 재검토 또는 제거 — publish 가정에 묶여 있어 본 결정으로 무의미해질 가능성.

### 2026-05-28 (네이밍 정합) — commit: `docs: Normalize BTIP naming conventions`

> 본 세션은 BTIP 문서 전반의 네이밍 컨벤션 일관성 정리에 집중. 양방향(BPrN/BPuN) 대칭, BPuN Solidity contract vs BPrN chaincode 구분, BTIP 번호 동반 여부에 따른 접미사 규칙 정착이 핵심.

#### ✅ 자료구조명에 BPrN 접두사 추가 — BPuN 측과 대칭

- `TxEventProofPayload` → `BPrNTxEventProofPayload` (btip-19/20/21/24). BPuN 측 `BPuNTxEventProofPayload`와 대칭.
- Solidity 구조체 `TxEventProof` → `BPrNTxEventProof` (btip-21/23). 위 페이로드의 Solidity 정의.
- **substring 충돌 함정**: `BPuNTxEventProofPayload`도 `TxEventProofPayload` substring을 포함하므로 단순 `replace_all`은 `BPuNBPrNTxEventProofPayload` 오염을 만듬. word-boundary sed(`\bTxEventProof\b`) 또는 사전 BPuN 카운트 확인 필수. 1차 시도에서 task-contexts/btips.md에 오염 9곳 발생 → 즉시 복구.

#### ✅ BTIP-26/34 제목 LinkerApp 도입

- BTIP-26: `Application Contract interface for Linker Protocol V2 on BPuN` → `LinkerApp interface on BPuN`
- BTIP-34: `Chaincode Interface for Linker Protocol V2 on BPrN` → `LinkerApp interface on BPrN`
- 근거: LayerZero V2의 `OApp(Omnichain Application)` 패턴을 BEATOZ의 `Linker<X>` 컨벤션으로 흡수. LayerZero V1의 모호한 "User Application(UA)"에서 V2의 짧고 브랜드성 있는 "OApp"으로 진화한 흐름 참고. 후보 `LinkerApplication`(풀워드)·`LApp`(터스, OApp 미러)과의 절충안.
- BTIP-10(V1)은 V2 정리 범위 밖이라 기존 제목 유지.

#### ✅ 제목 (V2) 제거 + BPuN/BPrN 대칭

- btip-21/23: 제목·README에서 `(V2)` 제거 (`LinkerEndpoint(V2) interface on BPuN` → `LinkerEndpoint interface on BPuN`).
- btip-29/31/32/33/34: 제목·README·코드 블록 주석에서 `Chaincode on BPrN` → `interface on BPrN`.
- 결과: BPuN ↔ BPrN 제목이 완전 대칭 — `LinkerXxx interface on {BPuN, BPrN}`.

```
BPuN                                BPrN
─────────────────────────────────   ─────────────────────────────────
LinkerEndpoint interface on BPuN ↔ LinkerEndpoint interface on BPrN
LinkerVerifier interface on BPuN ↔ LinkerVerifier interface on BPrN
LinkerNullifier interface on BPuN ↔ LinkerNullifier interface on BPrN
                  (LinkerPolicy는 BPrN 측에만)
LinkerApp interface on BPuN      ↔ LinkerApp interface on BPrN
```

#### ✅ Prose 컨벤션 정착 — BTIP번호 동반 시 bare, standalone 시 Chaincode 부착

- 규칙: `BTIP번호(LinkerXxx)` 형태(예: `BTIP29(LinkerEndpoint)`)는 BTIP 번호로 식별되므로 bare. 그 외 standalone prose 참조는 `LinkerEndpoint Chaincode` 형태로 Chaincode 부착(BPuN Solidity contract와 disambiguation).
- 적용: btip-29/31/32/33/34/39 prose 12+곳.
- 예외 (변경 없음): `BPuN LinkerEndpoint` 등 BPuN-qualified 참조, pseudocode 메소드 호출(`LinkerNullifier.IsProcessed(...)` 등 call target), 이미 한글 "체인코드" 동반된 곳.
- 트리플 standalone(btip-29 L286 `LinkerVerifier·LinkerNullifier·LinkerRegistry`)도 규칙 일관성 우선해 셋 다 Chaincode 부착. verbose하지만 통일성 유지, 향후 가독성 리팩토링 여지.

#### ✅ README에 누락 BTIP 번호 Reserved 추가

- BTIP15, BTIP18, BTIP22를 Reserved 행으로 신규 추가 (기존 BTIP30/38 Reserved 패턴과 동일).
- 결과: BTIP09~39 범위 내 모든 번호가 README에 등장.

#### 부수 작업

- task-contexts/btips.md의 `TxEventProofPayload` 6곳 동기화.
- 본 컨텍스트 파일의 "파일 상태 요약" 표(btip-29/31/32/33/34) 신규 제목으로 갱신. 표 안의 `ValidatorSetRegistry Chaincode on BPrN`(이전 세션의 stale 잔존)도 현재 `LinkerPolicy interface on BPrN`로 동시 정정.

#### 보류 / 향후

- linker-v2 코드(on-bpun Solidity 구조체명, on-bprn Go 인터페이스)는 본 turn 변경 범위 밖. 추후 구현 동기화 단계에서 새 컨벤션(`BPrNTxEventProof`, `LinkerApp`, 제목 패턴 등) 적용 필요.

---

### 2026-05-28

> 이 세션은 **자잘한 문서 정합** 중심. 전체 BTIP 문서·구현 현황 재분석 후 누락된 문서 작업 2건 완료. 주요 부수 발견: linker-v2.md에 "✅ 완료"로 기록된 on-bprn 레지스트리 기반 리팩토링이 실제 Go 코드에 미반영 상태.

#### ✅ BTIPS/README.md — BTIP37/38(Reserved)/39 등재

2026-05-21 이후 추가된 BTIP들이 README 테이블에 누락돼 있었음. 아래 3행 추가:

| BTIP37 | LinkerRegistry interface (BPuN + BPrN) |
| BTIP38 | (Reserved) |
| BTIP39 | BPuN Validator Set Update Proof |

#### ✅ btip-29 문서 정합 — SetRegistryID 단일화 반영

on-bprn 레지스트리 기반 리팩토링(2026-05-26 구현 세션에서 설계 확정)을 btip-29 BTIP 문서에 반영.

**변경 내역**:
- `requires`: `btip28, btip34` → `btip28, btip34, btip37`
- Chaincode Interface: `SetVerifierChaincodeID` + `SetNullifierChaincodeID` 제거 → `SetRegistryID` 단일 메소드로 교체
- 메소드 설명: 단일 부트스트랩 포인터 역할 명시 (LinkerVerifier·LinkerNullifier ID는 레지스트리를 통해 동적 조회)
- OnProof 구현 설명: "컴포넌트 조회" 항목 추가 (`ResolveContract(VERIFIER/NULLIFIER)`)
- OnResult 구현 설명: "컴포넌트 조회" 항목 추가 + BPuN 공식 엔드포인트 조회는 `GetContract(payload.ChainID, LINKER_ENDPOINT)`임을 명시

#### ⚠️ 부수 발견 — on-bprn 코드-문서 불일치

linker-v2.md 2026-05-26 항목에 "✅ 체인코드 간 호출을 레지스트리 기반으로 통합"으로 기록돼 있으나, **실제 Go 코드는 변경되지 않은 상태**:
- `linker-endpoint/main.go`: `SetVerifierChaincodeID` / `SetNullifierChaincodeID` 아직 존재
- `linker-verifier/main.go`: `SetLinkerPolicyID` 아직 존재
- `linker-nullifier/main.go`: `SetLinkerEndpointID` 아직 존재
- `types/registry.go`: 존재하지 않음 (`SetRegistryID` / `ResolveContract` 미구현)

→ BTIP 문서는 의도된 설계 기준으로 갱신 완료. 코드 구현은 별도 작업으로 진행 필요.

### 2026-05-27

> 이 세션은 **설계 점검 + 2PC 보안 보강(문서)** 중심. BPuN-origin "이벤트→BPrN 결제" 서비스 설계 연구는 분량이 많아 별도 파일 `./bpun-origin-payment-design.md`에 정리(미구현, 다음 세션 이어가기용). 2PC 결정의 의도는 `./btips-2pc-design.md` §9 참조.

#### ✅ BPuN→BPrN 방향 설계 완결성 점검

"BPrN→BPuN은 됐는데 BPuN→BPrN도 다 됐나?" 점검. 결론: **핵심 설계(btip-27/28/29/31/32/33/34/35/39)는 닫혀 있음**(btip-28에 Prover 알고리즘·검증 절차 포함), on-bprn 체인코드도 스텁 없이 구현됨. 남은 건 대부분 **구현**(BPuN→BPrN 이벤트 Prover = `BPuNTxEventProofPayload` 생성기 미구현이 end-to-end의 결정적 미싱 피스)과 문서 정합:
- (문서 갭) BPuN→BPrN 통합 개요 문서 부재(btip-20 "BPrN→BPuN" 대응 없음) — 보류.
- (문서 정합) `BTIPS/README.md`에 BTIP-37/39(및 38 Reserved) 미등재 — 추후.
- secp256k1 잔존 교정·btip-39은 이전 세션에 완료.

#### ✅ btip-28 / btip-32 — stale 참조를 btip-39로 정합 + H₀ admin-trust 확정

btip-28(L101, L430)·btip-32(L76, L148)이 "Validator Set 동기화는 별도 문서에서 다룬다 + Light Client **Sequential/Bisection**"이라고 서술했으나, 그 별도 문서는 이미 **btip-39**이고 btip-39은 Bisection을 기각·Sequential 전용으로 확정함. → 모두 **btip-39 참조로 갱신, "Bisection/Light Client/별도 문서" 표현 제거.** 동시에 **H₀(최초 신뢰 높이) = 관리자 `SetValidatorSet`(admin-trust), 이후 변경 = btip-39 `UpdateValidatorSet`(trustless)** 를 명문화.

#### ✅ 2PC — 결과 콜백에 "핸들러 식별자" 추가 (#5a) — btip-21/26/29/34

LinkerResult가 공식 LinkerEndpoint에서 나왔는지(출처 검증)는 막아도, **공식 엔드포인트를 악용해 정방향 증명을 "항상 거부하는" 핸들러에 흘리면** 의도와 다른 결과로 에스크로가 잘못 settle/refund되는 griefing이 가능. 이를 막기 위해:
- `handleLinkerResult`(btip-26) / `HandleLinkerResult`(btip-34) 시그니처에 **핸들러 식별자**(`handlerCcApp` / `handlerDApp`) 인자 추가.
- `onResult`(btip-21)는 `LinkerResultElems`의 `appChaincodeID`(gidx:8)를, `OnResult`(btip-29)는 `LinkerResult`의 `dApp`을 꺼내 콜백으로 전달. **이벤트 정의는 이미 핸들러 필드를 보유 → 정의 변경 없음, 콜백 포워딩만.**
- 발신 앱은 lock 시점에 **의도한 핸들러**를 기록하고, 콜백의 핸들러와 일치할 때만 완결(앱 책임). 설계 근거는 `./btips-2pc-design.md` §9.

#### ✅ Nullifier 소비 단위 확정 (#4) — btip-24 / btip-33

Nullifier는 `(root, app)`당 **정확히 1회 소비**(요소/속성별 세분화 아님)임을 양방향(btip-24 `(tx_event_root, dApp)`, btip-33 `(event_attrs_root, appChaincodeID)`)에 NOTE로 명문화. 필요한 요소는 한 증명에 모두 포함해야 함.

### 2026-05-26

#### ✅ btip-37 부록 — BPrN 식별자를 HLF 네이티브 string으로 개정 (구현 세션)

BTIP-37 Appendix(BPrN Chaincode Equivalent) 인터페이스를 `bytes32 chainId` / `keccak256 role` / 20B `address` → **HLF 네이티브 string**(`channelID` / 역할 이름 / `chaincodeID`)으로 변경.

- 근거: Fabric은 체인코드를 `(channelID, chaincodeID)` 문자열로 식별하고 `InvokeChaincode`/`SignedProposal`가 모두 string 기반. BTIP-9의 20B 파생 주소는 단방향 해시(`sha256(channelId+chaincodeId)`)라 `InvokeChaincode`에 필요한 이름으로 역산 불가.
- 서술 간결화(사용자 요청): BTIP-9/keccak/"왜 다른지" 구구절절 제거 → "BPuN과 식별자 타입이 다르므로 아래와 같이 정리" + 3줄 매핑(channelID/role/chaincodeID string) + Go 인터페이스 + X.509 admin 한 줄.
- on-bprn 구현도 이에 맞춰 정합화(레지스트리 값=체인코드 이름 string, 체인코드 간 호출을 레지스트리 `ResolveContract`로 통합, 개별 SetXxxID 제거→단일 SetRegistryID). 코드 상세는 `./linker-v2.md` 2026-05-26 구현 세션 참조.

> 같은 날 진행한 코드 구현(BTIP-37/39 구현, secp256k1, BTIP-39 Prover, tendermint 라이브러리 직접 호출 정합 등) 전체는 `./linker-v2.md` 참조. 본 항목은 그 중 btips 문서에 반영된 변경만 기록.

#### ✅ btip-28/31/32 — BPuN Validator 서명 알고리즘 secp256k1 교정

BTIP-39(2026-05-25) 작성 중 발견한 ed25519 잔존 표현을 secp256k1로 교정. BEATOZ는 tendermint-ethaddr 포크로 secp256k1 전용이라는 사실(메모리: `project_beatoz_secp256k1_only`)과 BTIP-32 본문의 일관성 확보.

**변경 파일**: btip-32(PubKey 주석 33B compressed + BTIP39 참조), btip-31(`verify_ed25519` → `verify_secp256k1`, 암호학적 함수 설명, BTIP23과의 차이 절), btip-28(`ed25519_sign` → `secp256k1_sign`). docs/ 전체에 `[Ee]d25519|EdDSA` 잔존 0.

**기각한 디테일**: 초안에서 "서명 입력 = sha256(CanonicalVote)"를 추가했으나 tendermint-ethaddr의 실제 서명 입력 형식을 확인하지 않은 추측이라 제거. 알고리즘 표기만 유지.

**커밋**: `b66ccde docs: Correct BPuN validator signature scheme to secp256k1`

#### ✅ btip-37 통합 + btip-38 철회 — LinkerRegistry 다중체인 단일 인터페이스

기존 BTIP-37/38은 BPuN/BPrN 측 LinkerRegistry를 별도 BTIP 번호로 정의했고, BPuN 측에는 로컬 컨트랙트용(`getContract(role)`)과 원격 BPrN 엔드포인트용(`getRemoteEndpoint(chainId)`) 메소드가 분리돼 있었음. 이를 단일 키 `(chainId, role) → address` 모델로 통합. 자세한 설계 의도는 `./btips-2pc-design.md` §8 참조.

**핵심 변경 요약**:
- **인터페이스 통합**: `getContract(bytes32 chainId, bytes32 role) → address` / `setContract(...)` 두 메소드만 남김. 자기 체인 조회는 `bytes32(block.chainid)`, 원격 BPrN은 `sha256(channelId)` (BTIP-9 규약).
- **타입 통일**: BPrN 체인코드도 BTIP-9의 `(channelId, chaincodeId) → 20B address` 결정적 파생 규칙으로 `address` 타입 통일. 양 체인이 동일 데이터 모델 사용.
- **제거**: `getCodeHash`/`isAuthentic`(EXTCODEHASH 검증은 로컬 BPuN에만 의미 있는 비대칭 기능 → dead weight), `getRemoteEndpoint`/`setRemoteEndpoint`(통합 키로 흡수), `CodeHashMismatch` 에러.
- **btip-38 철회**: BTIP-37과 데이터 모델·메소드·의미가 모두 동일(차이는 언어뿐) → 별도 번호 분리 정당성 없음. BTIP-37에 Go 인터페이스를 Appendix로 통합. README에서 `(Reserved)` 표시(번호 재사용 안 함).

**호출자 갱신**:
- btip-21 onResult: `getRemoteEndpoint(BPRN_CHAIN_ID)` → `getContract(BPRN_CHAIN_ID, LINKER_ENDPOINT)`
- btip-26 dApp 콜백 호출자 검증: `getContract(LINKER_ENDPOINT)` → `getContract(bytes32(block.chainid), LINKER_ENDPOINT)`
- btip-29 OnResult: `GetRemoteEndpoint(payload.ChainID)` → `GetContract(payload.ChainID, LINKER_ENDPOINT)`. BTIP-38 참조 → BTIP-37
- btip-34: BTIP-38 참조 → BTIP-37 (자기 체인 매핑 조회로 표현 정리)

**커밋**: `50198c7 docs: Unify LinkerRegistry under BTIP37 with (chainId, role) key`

### 2026-05-25

#### ✅ btip-39.md 신규 — BPuN Validator Set Update Proof

BTIP-32(LinkerPolicy)가 "별도 문서에서 다룬다"고 보류했던 Validator Set 자동 동기화 메커니즘 정의. BTIP-28이 전제한 "Verifier가 Validator Set을 보유한다"는 가정을 프로토콜 수준에서 충족. 자세한 설계 의도는 별도 문서 참조(또는 본 파일 §주석).

**핵심 결정**:
- **Sequential 전용** (Skipping/Bisection 제거): trusted_height → trusted_height+1 한 블록씩 전진. Tendermint 합의 항등식 `ValidatorsHash(H) == NextValidatorsHash(H-1)`만으로 결정론적 검증. trustLevel/long-range 가정 회피.
- **secp256k1 전용**: BEATOZ는 tendermint-ethaddr 포크로 secp256k1만 사용. ed25519 분기 코드 없음.
- **변경 탐지 = 헤더 비교**: 단일 헤더 H에서 `ValidatorsHash != NextValidatorsHash` 비교만. Prover는 BPuN 공개 RPC만으로 수행 가능, 권한 키 불요(trustless permissionless).
- **인터페이스 = BTIP-32 확장 단일 메소드**: `UpdateValidatorSet(payload) error`. 저장소(`SetValidatorSet`/`GetValidatorSet`)는 BTIP-32 정의 그대로 사용, 재정의 없음. 검증 통과 시 `BTIP32.SetValidatorSet` self-call(권한 검사 우회).
- **관리자 등록과 공존**: BTIP-32의 `SetValidatorSet`(관리자 전용)은 초기 부트스트래핑(H₀ 등록)/비상 보정에 유지.

**데이터 구조**:
- `SimpleValidatorEntry { pubkey (33B secp256k1 compressed), voting_power int64 }`
- `ValidatorSetProofPayload { trusted_height, target_height, chain_id, round, block_hash, block_parts_total, block_parts_hash, validator_sigs, next_validators_hash_proof(헤더 인덱스 8), next_validator_set }`

**검증 절차 5단계**: 페이로드 무결성 → trusted_set 조회 → Commit 서명 검증(2/3+) → 헤더 머클 증명(인덱스 8) → ValidatorsHash 재계산 일치 → BTIP-32.SetValidatorSet self-call.

**Address Derivation**: secp256k1 → `Keccak256(uncompressedPubkey[1:])[12:]` (Ethereum 스타일). ValidatorsHash 자체는 SimpleValidator(PubKey, VotingPower)만 사용해 주소 파생과 무관, 서명자-Validator 매칭 단계에서만 필요.

**작성 과정의 교정 사항** (재발 방지):
- 초안에서 BTIP-32 저장소를 재정의 시도 → BTIP-32 인터페이스 그대로 사용으로 정정 (사용자 redirect)
- 초안에 ed25519/secp256k1 양쪽 분기 → secp256k1 단일 전제로 정정 (BEATOZ 운영 사실)
- Skipping(Bisection) 모드 포함 → Sequential 전용으로 단순화 (사용자 redirect)
- 변경 탐지 방식 A/B 양쪽 → 방식 A(헤더 비교)만 유지
- "관리자 등록(SetValidatorSet)" 표현 모호 → "관리자 전용 수동 등록 메소드"로 명확화
- "Empty Commit 거부" 절에서 `BlockIDFlagCommit` 무문맥 사용 → BTIP-28 `ValidatorSignature` 정의 참조로 갈음 + 절 제목을 "서명자 임계치 우회 불가"로 변경
- tendermint-ethaddr 포크 명시(URL/버전 link) → "Tendermint v0.34.24 기준" 한 줄로 축약 (BEATOZ 패키지 사정은 비공개 컨텍스트)

**이름 변경 이력 (사용자 요청)**:
- `ValidatorSetUpdateProofPayload` → `ValidatorSetProofPayload`
- `SubmitValidatorSetUpdate` → `UpdateValidatorSet`

**파일 변경**:
- 신규: `btip-39.md` (Abstract / Trust Model / Validator Set Change Detection / Proof Construction / Submission Interface / Verification Procedure / Security Considerations / Conclusion)
- 개정: `BTIPS/README.md` (BTIP39 등재)

**메모리 저장**:
- `project_beatoz_secp256k1_only.md` — BEATOZ Validator는 secp256k1 전용. ed25519 언급은 잘못된 것으로 간주.

**부수 관찰**:
- [btip-32.md:23](../docs/BTIPS/btip-32.md#L23) `Validator.PubKey` 주석에 "Ed25519 공개키 (32 bytes)" 잔존. 별도 작업으로 secp256k1 33B compressed로 교정 필요.

### 2026-05-22

#### ✅ BPuN-origin 양방향 대칭 확장 (결과-주도 2PC)

BPrN-origin의 거울상(BPuN 발신 → BPrN 앱 체인코드 처리 → 결과 BPuN 반송) 구현. 두 엔드포인트가 `onProof`(결과 발행) + `onResult`(결과 수신) 모두 갖춤. **전체 논의·결정 의도·검토 대안은 `./btips-2pc-design.md` 6절** 참조. 아래는 요약.

- **correlationId 방향별 비대칭**: BPrN-origin = `tx_event_root`(내재), BPuN-origin = **발신 컨트랙트 명시 id**(nonce). 이유: BPuN(EVM) 컨트랙트가 event_attrs_root를 계산하면 beatoz-go `evmLogsToEvent` 인코딩에 강결합(노드 포맷 변경 시 깨짐). 명시 id로 회피. correlationId(매칭)와 Nullifier(재생방지, root 기준)는 분리.
- **거부 표현 비대칭**: BPuN dApp = `revert`(EVM 자동 롤백), BPrN 앱 체인코드 = `accepted=false` 정상 리턴(Fabric 무롤백 → **CAUTION: 거부 전 PutState 금지**). contractapi가 err 시 리턴값을 버려 correlationId 유실되므로 거부를 err로 못 함. `error`는 하드 실패(tx 전체 실패→재시도) 전용.
- **HandleLinkerEvent 반환 `(LinkerResultRef{CorrelationIndex int64, Accepted bool}, error)`**: 앱 체인코드가 correlationId의 **인덱스**를 반환 → EP가 검증된 values에서 값을 꺼냄(위조 방지). `CorrelationIndex<0` = fire-and-forget(opt-in 신호).
- **fire-and-forget 비대칭**: #1 always-emit(내재 correlationId+revert는 신호 못 실음), #2 opt-in(CorrelationIndex 센티넬).
- **네이밍**: BPuN쪽 Solidity 이벤트 `LinkerResult`, BPrN쪽 EventLog `LinkerResultElems`. `chaincodeID`→`appChaincodeID`(btip-29/33).
- **검증**: `evmLogsToEvent`(ctrler.go:339) 확인 — event_attrs_root 계산은 가능하나 결합 위험으로 명시 id 선택. `removed`는 BFT 즉시완결로 항상 false.

**파일 변경**: btip-29(OnProof 결과 발행+LinkerResultElems 정의+appChaincodeID), btip-21(onResult 추가), btip-26(handleLinkerResult+BPuN Escrow Lifecycle), btip-34(HandleLinkerEvent→LinkerResultRef+CAUTION), btip-33(appChaincodeID), btip-24(dedup NOTE).

### 2026-05-21

> 📎 이 세션의 **전체 논의 흐름·검토 대안·결정의 의도·발견한 함정**은 별도 설계 노트 `./btips-2pc-design.md`에 자기완결적으로 정리됨. 아래는 작업 요약.

#### ✅ 결과-주도 2PC (Cross-Chain Atomic Payment) 설계 및 스펙 작성

**문제**: 기존 문서는 BPrN→BPuN 단방향 증명만 다룸. 그러나 BPuN dApp의 처리 결과(accept/reject)를 BPrN으로 되돌려, 원발신 결제를 settle/refund해야 함. (예: BPrN 결제 → BPuN NFT 발행, dApp이 발행 거부 시 BPrN 결제 환불)

**해결 모델 — 에스크로 + 타임아웃 없는 결과-주도 2PC**:
- BPrN 결제를 즉시 확정하지 않고 **에스크로(LOCKED)** 로 잠금. ACCEPTED/REJECTED 결과 증명 도착으로만 SETTLED/REFUNDED.
- **타임아웃 없음** — 타임락 순서 레이스(이중 결과) 원천 제거. 안전 근거: 소비된 증명당 결과 정확히 1개 보장 + 결과는 BPuN 원장에 영구 기록 → 누구나(구매자 본인 포함) 언제든 결과 증명 제출 가능 (BPuN 살아있는 한 동결 없음).
- 잔여 리스크: BPuN 영구 정지 시 동결 → #6 운영 정책(결제 일시중단 + relayer force-drain 후 업그레이드)으로 통제.

**핵심 설계 결정**:
- **correlationId = 정방향 `tx_event_root`** (tx_id가 BTIP16 머클트리 gidx:2에 커밋되어 유일·content-bound, btip-24 Nullifier 기준값과 동일, 발신 체인코드가 실행 시점에 직접 계산 가능). tx_id보다 우월.
- **dApp revert-only** (bool 리턴 없음): 정상 반환=ACCEPTED, revert=REJECTED. LinkerEndpoint가 try/catch로 감싸 "소비된 증명당 결과 정확히 1개" 보장. revert가 부작용을 원자적 롤백.
- **always emit**: opt-in 마커 없이 모든 소비 증명에서 LinkerResult emit.
- **tx당 PaymentLocked 1개** (correlationId 모호성 제거).
- **onProof vs onResult 분리**: 정방향 증명(BPuN btip-21.onProof)과 LinkerResult 증명(BPrN btip-29.OnResult)은 서로 다른 체인에서 소비. OnResult는 "소스 컨트랙트 == 공식 BPuN LinkerEndpoint" 추가 검증(레지스트리 조회) 후 HandleLinkerResult로 라우팅.

#### ✅ btip-37.md 신규 — LinkerRegistry interface on BPuN (Solidity)

- 공식 컨트랙트 세트(LinkerEndpoint/Verifier/Policy/Nullifier) 진본성 단일 출처
- `getContract(role)`, `getCodeHash(role)`, `isAuthentic(role, addr)`, `getRemoteEndpoint(chainId)`, `setContract`(거버넌스), `setRemoteEndpoint`
- CREATE2 결정적 배포(salt+bytecode 해시 공개) + multisig·timelock 거버넌스
- "신뢰의 뿌리는 주소가 아니라 LinkerPolicy의 Root CA" 명시 — 레지스트리는 진본 식별의 편의·바인딩 계층

#### ✅ btip-38.md 신규 — LinkerRegistry Chaincode on BPrN (Go)

- btip-37의 BPrN 대응 (대칭). `GetContract`, `IsAuthentic`, `GetRemoteEndpoint`(공식 BPuN 엔드포인트 주소 보관), `SetContract`/`SetRemoteEndpoint`(관리자)
- Fabric 패키지 ID에 해시 포함되어 사칭 자연 방지

#### ✅ btip-21 개정 — LinkerResult 이벤트 + always-emit + try/catch

- `LinkerResult(correlationId=tx_event_root, status, originChannelId, originChaincodeId, dApp)` 이벤트 추가
- onProof: dApp 호출을 try/catch로 감싸 ACCEPTED/REJECTED 판정, 항상 LinkerResult emit. markProcessed는 try 이전(거부도 소비). nonReentrant + MIN_CALLBACK_GAS
- "Result Callback (2PC)" 섹션 추가. Linker 컴포넌트 주소는 btip-37 레지스트리로 진본 확인

#### ✅ btip-26 개정 — revert-only 의무 + 호출자 검증

- handleLinkerEvent(이미 리턴값 없음): "정상 반환 ⟺ 이행 완료, 거부는 revert로만, 부작용 남긴 채 정상반환 금지"
- `msg.sender == BTIP37(LinkerRegistry).getContract(LINKER_ENDPOINT)` 검증 의무

#### ✅ btip-29 개정 — OnResult 메소드 추가

- `OnResult(ctx, payload)`: LinkerResult 증명 소비. OnProof와 검증 동일하나 소스 컨트랙트(index 0)==공식 BPuN 엔드포인트(btip-38 GetRemoteEndpoint) + topic.0==LinkerResult 시그니처 추가 검증
- 대상은 검증된 LinkerResult의 originChaincodeId에서 추출 → BTIP34.HandleLinkerResult(correlationId, status) 호출
- btip-33 Nullifier로 relayer vs self-submit 중복 차단

#### ✅ btip-34 개정 — HandleLinkerResult 콜백 + Escrow Lifecycle Reference

- `HandleLinkerResult(ctx, correlationId []byte, accepted bool)` 전용 finalize 콜백 추가
- HandleLinkerEvent/HandleLinkerResult: "정상 반환 ⟺ 이행 완료(아니면 에러), 호출자==공식 LinkerEndpoint(btip-38) 검증" 의무
- **Reference: Escrow Lifecycle 섹션** 추가: 원발신 체인코드 두 역할, 상태머신(LOCKED→SETTLED/REFUNDED), 타임아웃 없음 근거, 제출 인센티브(판매자=ACCEPTED/구매자=REJECTED/relayer), 운영 정책(pause+force-drain)

#### ✅ btip-33 점검 — LinkerResult 중복 제출 방지 NOTE 추가

- OnResult가 소비하는 LinkerResult 증명도 event_attrs_root 기준 Nullifier로 dedupe (relayer vs 당사자 중복 제출 차단)

#### ✅ README.md — BTIP37, BTIP38 등재

#### 양방향 대칭 확장 보류 사항

- 현재 BPrN-origin 흐름만 구현. BPuN-origin(btip-29가 forward+emit, btip-21에 onResult 추가)은 추후 대칭 확장.

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
  - Commit 전체 대신 `validator_sigs`(실제 투표한 Validator 서명만)
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

- **`RFC6962Proof.total` 필드**: RFC 6962 머클 트리는 "n보다 작은 최대 2의 거듭제곱" 지점에서 좌우 분할하므로, 검증자가 분할 지점을 알기 위해 `total` 필요. BTIP27의 단순 연결 방식은 2의 거듭제곱까지 null 패딩하므로 불필요.
- **`ValidatorSignature.timestamp` 필드**: `CanonicalVote` 서명 입력 메시지에 `timestamp`가 포함되므로, 서명 검증을 위해 각 Validator의 투표 시각이 필수.
- **Amino 인코딩 이유**: Tendermint 헤더 14개 필드가 이질적 타입(`int64`, `string`, `time.Time`, 구조체, `[]byte`)이므로, 모든 타입을 바이트로 변환하는 정규 직렬화가 필요. `[]byte` 필드에도 일괄 적용하는 이유는 타입별 분기 없는 코드 일관성.

#### ✅ btip-29.md — LinkerEndpoint Chaincode on BPrN 신규 생성

- BTIP21(LinkerEndpoint on BPuN)의 BPrN 대응 문서
- BPuN 이벤트 증명(`BPuNTxEventProofPayload`)의 유일한 관문(Gateway) 체인코드
- Go 구조체로 `BPuNTxEventProofPayload` 전체 정의 (BTIP28 대응)
- `OnProof`: Nullifier 검사(BTIP33) → 검증 위임(BTIP31) → 상태 기록 → 애플리케이션 이벤트 전달
- `SetVerifierChaincodeID`, `SetNullifierChaincodeID` 관리자 함수
- `ProofVerifiedEventElems` 이벤트 정의 (BTIP16 EventLog 형식, HLF 단일 이벤트 제약 반영)
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
- Nullifier: `sha256(eventAttrsRoot || chaincodeID)` — validator_sigs 제외 (BTIP24와 동일 근거)
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

### 2026-04-21

#### ✅ btip-16 — EventLog 구조 변경: evtlog_id 제거, selector를 Header 필드로 추가

- Header에서 `evtlog_id` (string) 필드 제거
- Header에 `selector` (bytes) 필드 추가 — 이벤트 종류를 식별하는 32바이트 해시값
- EventLog는 `header` + `elems` 2개 필드 구조 유지 (selector는 Header 내부)
- protobuf: `Header { channel_id, chaincode_id, tx_id, selector }`, `EventLog { header, elems }`
- Selector 섹션: `evtlog_id` 참조 제거, `EventName(type1,type2,...)` 형식으로 설명
- gidx 테이블: gidx:3 소스를 `Header.selector`로 변경
- Event Architecture, Merkle Tree 구조 설명 갱신

#### ✅ btip-16 — Header.tx_id 타입 변경: string → bytes

- `string tx_id` → `bytes tx_id`로 변경

#### ✅ btip-25, 29 — btip-16 EventLog 구조 변경 반영

- btip-25: `Header.evtlog_id` 참조 문장 제거, gidx 테이블 tx_id 타입 `string` → `bytes`
- btip-29: `EventLog.Header`의 `evtlog_id` → `EventLog.Header`의 `selector`로 변경, gidx 테이블 tx_id 타입 `string` → `bytes`
- btip-29 OnProof 슈도코드: `Header(channel_id, chaincode_id, tx_id, selector=...)` 형식으로 변경
- btip-26: 이벤트 출처 정보 인용문 "EventLog Header에는"으로 복원

#### ✅ btip-24 — IBTIP24 소스 코드 반영: markProcessed 원자적 check+mark

- `markProcessed` 반환 타입 추가: `returns (bool wasDup)`
- 이미 처리된 쌍이면 `true` 반환 (상태 변경 없음), 새로 처리하면 `false` 반환
- `isProcessed` + `markProcessed` 2단계 패턴 → `markProcessed` 단일 호출로 간소화
- 인터페이스 설명, 구현 슈도코드, 호출 순서 섹션 갱신
- 소스 코드 위치: `linker-v2/verifier/on-bpun/contracts/interfaces/IBTIP24.sol`

#### ✅ btip-21 — onProof 흐름 변경: markProcessed 원자적 호출 반영

- `isProcessed` 호출 제거, `markProcessed` 단일 호출로 중복 검사+등록 수행
- 순서: 검증 위임 → `markProcessed(event_root_hash, targetDApp)` → `wasDup`이면 revert
- 설명 텍스트: "Nullifier 계산" + "중복 처리 검사" 항목 → "이벤트 루트 해시 추출" + "중복 검사 + 처리 완료 기록" 항목으로 통합

#### ✅ btip-23 — IBTIP22 소스 코드 반영: 조직별 CRL, 복합 getter

- `getCRL()` → `getCRL(mspid)` — 전역 CRL에서 조직별 CRL로 변경
- `getRootCAAndCRL(mspid)` 복합 getter 추가 (Root CA + CRL 단일 호출)
- `getEndorsementPolicy()` 반환 타입 명시: `(uint256 minEndorsers, bytes[] requiredOUs)`
- 소스 코드 위치: `linker-v2/verifier/on-bpun/contracts/interfaces/IBTIP22.sol`

#### ✅ btip-19 — verify_event_proof CRL 조직별 조회 반영

- `get_crl()` 전역 호출 제거
- `get_root_ca_and_crl(mspids[i])` 루프 내 조직별 Root CA + CRL 동시 조회로 변경

#### ✅ btip-20 — 시퀀스 다이어그램 갱신

- `isProcessed` 호출 제거, `markProcessed(event_root_hash, targetDApp)` → `wasDup = false` 반영
- `getRootCA(mspid)` → `getRootCAAndCRL(mspid)` 변경, 반환값 `rootCA, crl` 표시
- `getEndorsementPolicy()` 반환값 `(minEndorsers, requiredOUs)` 표시

#### ✅ btip-35.md — BPuN Transaction Event Definition 신규 생성

- `asyncExecTrxContextEx` 구현 기반 BPuN 트랜잭션 이벤트 정의 문서
- `"tx"` 이벤트: 4개 attribute (type, sender, receiver, amount) — Per-Event Tree 인덱스 0~3
- `"evm"` 이벤트: 가변 attribute (contractAddress, topic.N, data, blockNumber, removed)
- Event Ordering: `"tx"` 항상 Events[0], `"evm"` 이벤트는 EVM 로그 순서대로 추가
- 실패 트랜잭션: `tx_event_root` 미기록, 증명 대상 아님
- 트랜잭션 유형 문자열: transfer, staking, unstaking, proposal, voting, contract, setdoc, withdraw
- Per-Event Tree 구조 다이어그램 포함

#### ✅ btip-27 — btip-35 관련 변경 롤백

- btip-27은 작성 시점의 beatoz-go 이벤트 구성(asyncExecTrxContextOld 기준)이 원본이므로 롤백
- btip-35에서 btip-27의 이벤트 구성이 초기 구현 기준이며 본 문서로 대체됨을 Motivation에 명시
- 원칙: 앞선 문서(btip_N)가 후행 문서(btip_N+M)를 참조하지 않음

#### ✅ README.md — BTIP35 항목 추가

### 2026-04-20

#### ✅ btip-27 — Leaf 해시 단순화: `sha256(event_type || key || value)` → `sha256(value)`

- 리프 데이터를 `Attribute.Value`만으로 변경, `Event.Type`과 `Attribute.Key` 제거
- 속성의 의미는 머클 트리 내 인덱스(위치)에 의해 결정 (BTIP16 gindex 패턴과 통일)
- 이벤트 타입 식별: EVM의 `topic.0` (keccak256 시그니처)가 별도 leaf로 존재하여 머클 증명 가능
- 소스 컨트랙트 식별: `contractAddress`가 별도 attribute(index 0)로 존재하여 머클 증명 가능
- 보안상 문제 없음: 의미 결정 정보가 모두 트리 내 별도 leaf로 존재

#### ✅ btip-28 — EventAttrProof 단순화 및 event_type 필드 제거

- `EventAttrProof`: `key` 필드 제거, `value`를 `ByteArray`(raw bytes)로 변경
- `BPuNTxEventProofPayload`: `event_type` 필드 제거 (이벤트 타입은 `topic.0`으로 식별)
- Proof Construction: `attr_leaves = [attr.Value for attr in ...]`로 변경
- Verification Step 6: `sha256(value)`로 리프 해시 재구성
- Leaf Data Summary 테이블 갱신

#### ✅ btip-31 — LinkerVerifier leaf 해시 변경

- Step 6: `sha256(payload.EventType + proof.Key + proof.Value)` → `sha256(proof.Value)`

#### ✅ btip-29 — Go 구조체 및 OnProof 변경

- `EventAttrProof`: `Key` 필드 제거, `Value`를 `[]byte`로 변경
- `BPuNTxEventProofPayload`: `EventType` 필드 제거
- `OnProof`: HandleLinkerEvent 호출을 `(chainId, blockNumber, txIndex, indices, values)` 형태로 변경

#### ✅ btip-26, btip-34 — 통합 HandleLinkerEvent 인터페이스 재정의

- **양방향 통합 인터페이스**: `HandleLinkerEvent(blockNumber, txIndex, indices[], values[])` — chainId는 별도 파라미터가 아닌 indices/values 내 필수 인덱스로 전달
- btip-26 (Solidity): `function handleLinkerEvent(uint64 blockNumber, uint64 txIndex, uint256[] indices, bytes[] values)`
- btip-34 (Go): `HandleLinkerEvent(ctx, blockNumber uint64, txIndex uint64, indices []uint64, values [][]byte)`
- **이벤트 출처 정보** 인용문 추가 (강제가 아닌 가이드 톤):
  - BPuN→BPrN: `index 0` (contractAddress), `index 1` (topic.0). chain_id는 CanonicalVote 서명으로 검증되므로 별도 인덱스 불필요.
  - BPrN→BPuN: EventLog Header `channel_id`, `chaincode_id`, `selector` (각 gindex로 머클 증명 가능)
  - 논조: "출처 신뢰가 필요한 경우 Prover에게 해당 증명을 포함시키고 dApp/체인코드에서 확인해야 한다"

#### ✅ btip-21 — onProof handleLinkerEvent 호출 변경

- `handleLinkerEvent(event_type, indices, data)` → `handleLinkerEvent(block_number, tx_index, indices, data)`
- `blockNumber`: payload에서, `txIndex`: gindex→leaf index 변환, chainId는 별도 파라미터 아님

#### ✅ btip-27 — EVM 이벤트 순서 NOTE 제거

- EVM 이벤트(`"evm"`)가 기본 이벤트(`"tx"`)보다 먼저 추가된다는 `[!NOTE]` 블록 삭제 (불필요)

#### ✅ btip-28, 29, 31 — EventAttrProof 제거, MerkleProof로 통합

- `EventAttrProof` 구조체 정의 제거 (btip-28의 Proof Data Structures에서 삭제)
- `EventAttrProof` Go struct 제거 (btip-29의 Data Structures에서 삭제)
- 페이로드 필드: `Array of EventAttrProof` → `Array of MerkleProof<attr_value>`
- 모든 필드 참조 변경: `proof.value`/`proof.Value` → `proof.leaf`/`proof.Leaf`
- 설명 텍스트, 주석, Leaf Data Summary 테이블 갱신
- 근거: key 제거 후 EventAttrProof(index, value, siblings)는 MerkleProof(index, leaf, siblings)와 구조적으로 동일

#### ✅ btip-26, 34 — 보안 필수 인덱스 → 이벤트 출처 정보 톤 변경

- "보안 필수 인덱스" 제목을 "이벤트 출처 정보"로 변경
- 강제적 톤("반드시 포함/거부") → 가이드 톤("출처 신뢰가 필요한 경우 Prover에게 포함시키고 확인해야 한다")

#### ✅ btip-34, 29 — HandleLinkerEvent에 chainId 파라미터 추가

- BPuN→BPrN 방향: `chainId`를 HandleLinkerEvent 첫 번째 파라미터로 추가
- 근거: BPuN ABCI 이벤트에는 BPrN EventLog Header의 `channel_id`(gidx:0)에 대응하는 네트워크 식별 필드가 Per-Event Tree에 없음. 모든 이벤트에 중복 추가하는 것도 비효율적. CanonicalVote 서명으로 블록 레벨에서 검증된 `payload.ChainID`를 LinkerEndpoint가 별도 파라미터로 전달
- btip-34: `HandleLinkerEvent(ctx, srcChainId, srcBlockNumber, srcTxIndex, indices, values)`
- btip-29 OnProof: `chain_id = payload.ChainID` 추출 후 전달
- btip-26(BPrN→BPuN)은 변경 없음 — `channel_id`가 EventLog Header gidx:0에 있어 indices/values로 전달 가능
- 양방향 HandleLinkerEvent 시그니처 비대칭은 두 방향의 이벤트 아키텍처 차이에서 비롯되는 필연적 결과

#### ✅ 전체 — gindex → gidx 통일, MerkleProof 필드 index 통일

- `gindex` 약칭을 btip-16에서 정의한 `gidx`로 전체 통일 (btip-16, 19, 20, 21, 25, 26, 27, 29)
- MerkleProof 구조체 필드: `gindex` → `index` (btip-19, 21). btip-28은 이미 `index` 사용 중이었으므로 양방향 일관성 확보
- 코드 내 `.gindex` 참조 → `.index`로 변경 (btip-19 코드, btip-21 코드, btip-20 설명)
- `gindex_to_leaf_index()` → `to_leaf_index()` (btip-21)

#### 설계 논의 기록

- **event_type 암호학적 신뢰 불필요**: `targetDApp`과 동일한 신뢰 수준(Prover 신뢰)으로 충분. BPrN→BPuN은 프라이빗 환경이므로 더욱 문제 없음. BPuN→BPrN은 `topic.0`이 이벤트 시그니처로서 별도 leaf에 존재.
- **소스 컨트랙트 식별**: 양방향 모두 머클 트리 내에 소스 식별 정보 존재. BPrN: `chaincode_id`(EventLog header gidx), BPuN: `contractAddress`(ABCI event attribute index 0). beatoz-go `evmLogsToEvent()`에서 확인 완료.
- **네트워크 식별**: BPrN→BPuN: `channel_id`(EventLog header gidx), BPuN→BPrN: `chain_id`(CanonicalVote 서명으로 블록 레벨 검증, 별도 인덱스 불필요). `chainId`는 별도 파라미터가 아닌 `indices/values` 내 출처 정보 또는 블록 레벨 검증으로 처리.
- **BPuN EVM 이벤트 attribute 구조 (beatoz-go 확인)**: `contractAddress`(index 0), `topic.0`(index 1), `topic.N`(index 1+N), `data`, `blockNumber`, `removed`

### 2026-04-16

#### ✅ btip-34.md — Chaincode Interface for Linker Protocol V2 on BPrN 신규 생성

- BTIP26(BPuN dApp 콜백 인터페이스)의 BPrN 대응 문서
- `HandleLinkerEvent(ctx, sourceAddress, eventType, keys, values)`: BTIP29가 검증 완료 후 호출하는 콜백
- `CancelLinkerEvent(ctx, eventAttrsRoot)`: 처리된 이벤트의 Nullifier 취소 (BTIP33 호출)
- BPuN(Solidity)은 `uint256[] index, bytes[] data` 패턴, BPrN(Go)은 `string keys[], string values[]` 패턴

#### ✅ btip-28, 29, 31, 34 — EventAttrProof 구조화 필드 도입

- **기존**: `MerkleProof<attr_leaf>` — 리프가 `Type||Key||Value` 바이트 연결로 파싱 모호
- **변경**: `EventAttrProof` 구조체 도입 — `Key`, `Value`를 별도 문자열 필드로 분리
- `event_type` 필드를 `BPuNTxEventProofPayload`에 추가
- Verifier가 리프 해시를 `sha256(event_type || key || value)`로 재구성하여 검증
- 연결 모호성 해결: `"ab"+"cd"+"ef"` vs `"abc"+"d"+"ef"` 문제 원천 방지

#### ✅ btip-29, 33, 34 — Nullifier 기준 eventAttrsRoot로 재설계

- **기존**: `sha256(txEventRoot || chaincodeID)` — 트랜잭션 레벨, 같은 tx 내 다른 이벤트 제출 차단
- **변경**: `sha256(eventAttrsRoot || chaincodeID)` — 이벤트 레벨, 독립적 처리 가능
- btip-33: `CalculateNullifier`, `IsProcessed`, `MarkProcessed`, `CancelNullifier` 시그니처 변경
- btip-29: Nullifier 호출 및 이벤트 구조 `TxEventRoot` → `EventAttrsRoot` 변경
- btip-34: `CancelLinkerEvent`의 파라미터 `eventAttrsRoot` 반영

#### ✅ btip-29 — sourceAddress 파라미터 추가

- `contractAddress` 속성(ABCI Event에서 `evmLogsToEvent()` 생성)을 검증된 속성에서 추출
- BTIP34의 `HandleLinkerEvent`에 `sourceAddress` 파라미터로 전달
- 수신 체인코드가 신뢰할 수 있는 소스 컨트랙트인지 판별 가능

#### ✅ btip-31 — verify_bpun_event_proof 참조 수정

- `BTIP28의 verify_bpun_event_proof` → `BTIP28의 Verification Procedure` (실존하는 섹션명으로 교정)

#### ✅ btip-28 — event_attrs_root 중간값 설명 추가 및 btip-27 참조 정리

- 중간값 계산 설명에 `event_attrs_root` 언급 추가
- btip-27과 중복된 2단계 머클 트리 상세 → btip-27 참조로 갈음

#### ✅ README.md — BTIP34 항목 추가

#### 설계 논의 기록

- **Nullifier 속성 레벨 세분화**: `eventAttrsRoot` 기준으로도 동일 이벤트 내 다른 attribute 증명 제출이 차단됨. 단, `event_attr_proofs`가 배열이므로 필요한 속성을 한 번에 제출하면 실용적으로 문제 없음. 추후 재검토 예정.
- **양방향 통합 데이터 전달 방식**: `(indices[], values[])` 패턴으로 양방향 통일. 속성의 의미는 인덱스가 결정 (BPrN: gindex, BPuN: attr index). 이전의 `keys[]` + `values[]` 패턴에서 변경됨.

#### ✅ btip-28, 29, 31, 32, 33 — 용어 변경

- `TendermintProof` → `RFC6962Proof` (Tendermint 버전 의존 제거)
- `CommitSignature`/`commit_sigs` → `ValidatorSignature`/`validator_sigs` (BPrN의 Block Commit Signature와 혼동 방지)
- `Tendermint v0.34.x` → `Tendermint` (버전 표기 제거)

#### ✅ btip-29 — ProofVerifiedEventElems 이벤트 통합

- `ProofReceived` + `ProofVerified` 두 이벤트 → `ProofVerifiedEventElems` 하나로 통합
- HLF 트랜잭션당 단일 이벤트 제약(SetEvent) 반영
- BTIP16 EventLog 형식 준수: `evtlog_id = "ProofVerifiedEventElems"`, `selector = sha256("ProofVerifiedEventElems(bytes,string)")`
- elems: `[event_attrs_root (bytes), chaincodeID (string)]`

#### ✅ btip-17, 19, 20, 21, 23 — Block Commit Signature에 block_height 추가

- btip-17: 서명 입력 변경 `Sign(PrivKey, block_event_root)` → `Sign(PrivKey, block_height || block_event_root)`
- btip-19: `BPrNTxEventProofPayload`에 `block_number` 필드 추가, Step 4 서명 검증 로직 수정
- btip-21: `TxEventProof` Solidity 구조체에 `uint64 block_number` 추가
- btip-23: `beatoz_p256Verify` 설명에 `block_height || block_event_root` 서명 원문 반영
- btip-20: 블록 커밋 서명 검증 설명 업데이트
- 양방향 `(block_number, tx_index)` 기반 크로스체인 트랜잭션 식별 체계 확보

### 2026-04-15

#### ✅ btip-27~33 — 가독성 전면 개선 (Technical Writer 관점)

- 문장·단락 간 연결 표현 추가, 끊기는 2-문장 패턴 통합
- 불릿 설명 앞 안내 문장 추가, 코드 블록 전후 맥락 연결
- Abstract·Motivation·Conclusion 문체 개선

#### ✅ btip-27 — 구조 개선

- "Transaction Event", "EVM Contract Event" 단락 → 각각 `Example: Transfer Transaction`, `Example: EVM Contract Transaction` 섹션의 `[!NOTE]` 인용문으로 이동
- `## Event Root in Tendermint Consensus` 섹션 → btip-28 `### Single-Path Trust Model (BPuN)`로 통합 이동 (BTIP27 범위는 `tx_event_root` 산출까지)
- `Merkle Tree Construction`, `Merkle Proof` 섹션 제거 → BTIP16 Merkle Tree 참조 한 줄로 대체

#### ✅ btip-16 — BTIP16 Merkle Tree Appendix 추가

- `## Appendix: BTIP16 Merkle Tree` 섹션 추가 (문서 끝, `---` 구분선 이후)
- 트리 구성, hashPair 규칙, MerkleProof 구조, 검증 알고리즘 독립 정의
- 기존 `## Merkle Tree 구조` 섹션의 hashPair 표 제거 → Appendix 참조로 대체
- `## Event Architecture` 내용 보강 (EventLog 설계 의도, gidx 할당 체계, EventLog Root 신뢰 결합)

#### ✅ btip-28 — 다수 개선

- **페이로드 필드 설명**: `**리프 데이터**: X` → `리프 데이터는 X` (자연스러운 문장 형식)
- **불투명 함수 명확화**:
  - `canonical(block_id)` → `CanonicalBlockID(Hash=..., PartSetHeader=...)` 인라인 전개
  - `marshal(CanonicalVote(...))` → `proto.marshal(CanonicalVote(...))`
  - `decode_last_results_hash(leaf)` → `amino.unmarshal(leaf)` + 주석
  - `unmarshal_deliver_tx(leaf)` → `proto.unmarshal(leaf, ResponseDeliverTx)` + 주석
  - `marshal(deterministic_response_deliver_tx(r))` → `proto.marshal(ResponseDeliverTx(Code=..., Data=..., ...))` + 주석
- **PartSetHeader 인라인화**: `block_parts_header: PartSetHeader` → `block_parts_total: Integer` + `block_parts_hash: ByteArray` (타입 정의 제거)
- **Single-Path Trust Model 확장**: 기존 다이어그램 + [!NOTE] 뒤에 3개 `####` 섹션 추가
  - `#### tx_event_root → LastResultsHash`: ABCIResults 계산 수식
  - `#### LastResultsHash → block_hash`: 헤더 필드 Amino 인코딩 + RFC6962 루트
  - `#### block_hash → Validator Signature`: CanonicalVote 직렬화 + ed25519 서명
- **MerkleProof 링크 수정**: `[BTIP27]의 BTIP16 머클 트리` 오류 → `[BTIP16 Merkle Tree]` 직접 링크

#### ✅ btip-29, 31, 32, 33 — Chaincode Interface Go 문법 교정

- `func (c *Type) Method(...)` 구체 타입 메서드 문법 → `type BTIPxx interface { Method(...) }` Go 인터페이스 문법으로 전면 교체

#### ✅ btip-29, 33 — targetApp → chaincodeID 용어 교정

- BPrN 체인코드 컨텍스트에서는 대상 식별자가 `address` (dApp)가 아닌 `chaincodeID`
- `targetApp` → `chaincodeID` 전체 치환 (파라미터명, 설명 텍스트, 슈도코드)
- "애플리케이션" → "체인코드" 관련 표현 교정

#### ✅ 용어 통일 — BTIP16 머클 트리

- `단순 연결 방식 머클 트리` → `BTIP16 머클 트리` (btip-27, 28, 29, 31)
- 관련 링크 참조도 `[BTIP16 Merkle Tree](./btip-16.md#appendix-btip16-merkle-tree)`로 수정

#### ✅ 변수명 리팩터링 (btip-27, 28, 29, 31)

| 구 용어 | 신 용어 | 변경 이유 |
|--------|--------|----------|
| `event_root` | `tx_event_root` | tx 레벨의 루트임을 명시 |
| `per_event_root` | `event_attrs_root` | 이벤트 내 attributes의 루트임을 명시 |
| `results_proof` | `tx_result_proof` | 하나의 tx 결과에 대한 증명 |
| `header_results_proof` | `last_results_proof` | 증명 대상(LastResultsHash) 기준 명명 |
| `event_root_proof` | `event_attrs_root_proof` | 증명 대상(event_attrs_root) 기준 명명 |
| `event_attr_proofs` | `event_attr_proofs` | 유지 (단수형 유지) |

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

#### ✅ BPrNTxEventProofPayload에 `mspids` 필드 추가 — 조직별 Root CA 조회

- `BPrNTxEventProofPayload`에 `mspids: Array of String` 필드 추가 (btip-19)
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
| `btip-16.md` | ✅ 수정 완료 | Header: `channel_id`, `chaincode_id`, `tx_id`(bytes), `selector`(bytes). `evtlog_id` 제거. BTIP16 Merkle Tree Appendix |
| `btip-17.md` | ✅ 수정 완료 | 실패 트랜잭션 리프 null, hashPair 규칙 BTIP16 참조, `Sign(PrivKey, block_height \|\| block_event_root)` |
| `btip-19.md` | ✅ 수정 완료 | MerkleProof\<Leaf\> (`index` 필드), block_number, mspids, cert_chains, block_commit_sigs, `get_root_ca_and_crl(mspid)` 조직별 조회 |
| `btip-20.md` | ✅ 수정 완료 | verifyProof, subgraph 다이어그램, sha256, `getRootCAAndCRL(mspid)`, `markProcessed` → `wasDup` 반영 |
| `btip-21.md` | ✅ 수정 완료 | BTIP21 interface, MerkleProof `index` 필드, `markProcessed` 원자적 호출, `handleLinkerEvent(srcBlockNumber, srcTxIndex, indices, data)` |
| `btip-22-xx.md` | ⚠️ 미수정 | 조직별 Root CA 매핑 반영 필요 |
| `btip-23.md` | ✅ 수정 완료 | BTIP23 interface (verifyProof + setPolicyContract), `getCRL(mspid)`, `getRootCAAndCRL(mspid)`, `getEndorsementPolicy()` 구조화 반환 |
| `btip-24.md` | ✅ 수정 완료 | BTIP24 interface — `markProcessed` returns `bool wasDup` (원자적 check+mark), `cancelNullifier` |
| `btip-25.md` | ✅ 신규 | TransferEventElems 정의 |
| `btip-26.md` | ✅ 수정 완료 | BTIP26 interface — `handleLinkerEvent(srcBlockNumber, srcTxIndex, indices, values)`, 이벤트 출처 정보 인용문(gidx 통일) |
| `btip-27.md` | ✅ 수정 완료 (btip-35 관련 롤백) | BPuN Event Structure — 2단계 머클 트리, leaf 해시 `sha256(value)`, 인덱스 기반 의미 결정. 이벤트 구성은 작성 시점(asyncExecTrxContextOld) 기준, btip-35에서 대체 |
| `btip-28.md` | ✅ 수정 완료 | BPuN Tx/Event Proof — `EventAttrProof` 제거(MerkleProof로 통합), `sha256(leaf)` 리프 해시 |
| `btip-29.md` | ✅ 수정 완료 | LinkerEndpoint interface on BPrN — MerkleProof 사용, `HandleLinkerEvent(srcChainId, srcBlockNumber, srcTxIndex, indices, values)` 호출 |
| `btip-31.md` | ✅ 수정 완료 | LinkerVerifier interface on BPrN — `sha256(proof.Leaf)` 리프 재구성 (BTIP23 대응) |
| `btip-32.md` | ✅ 신규 | LinkerPolicy interface on BPrN — `type BTIP32 interface` (BTIP22 대응) |
| `btip-33.md` | ✅ 수정 완료 | LinkerNullifier interface on BPrN — `type BTIP33 interface`, `sha256(eventAttrsRoot\|\|chaincodeID)` (BTIP24 대응) |
| `btip-34.md` | ✅ 수정 완료 | LinkerApp interface on BPrN — `HandleLinkerEvent(srcChainId, srcBlockNumber, srcTxIndex, indices, values)`, 이벤트 출처 정보 인용문 (BTIP26 대응) |
| `btip-35.md` | ✅ 신규 | BPuN Transaction Event Definition — `"tx"` 4 attrs (type, sender, receiver, amount), `"evm"` 가변 attrs, 이벤트 순서, 실패 tx 처리 |

---

## 표기 규칙

- **스펙 이름**: `BTIP21`, `btip21` (하이픈 없음)
- **파일 이름**: `btip-21.md` (하이픈 있음)
- **마크다운 링크**: `[BTIP21](./btip-21.md)` (표시 텍스트는 스펙명, URL은 파일명)
- **frontmatter requires**: `requires: btip21` (스펙명 규칙 따름)
- **참조 방향**: btip_N은 btip_N+M을 참조하지 않음 (후행 문서에서 선행 문서를 참조)

---

## verify_event_proof 현행 흐름 요약 (btip-19)

```python
def verify_event_proof(payload: BPrNTxEventProofPayload):
    # Step 1. Nullifier 중복 검사
    # Step 2. 인증서 체인 검증 + CRL 확인 → get_root_ca_and_crl(mspids[i])로 조직별 Root CA + CRL 조회 → (serial_no, pubkey, ou) 수집
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
- **BPuN**: Tendermint/DPoS 기반 퍼블릭 네트워크 (소스: `beatoz-go`, Tendermint 포크)
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
                         ← btip34 (dApp 콜백 인터페이스)
                    ← btip31 (LinkerVerifier)
                         ← btip32 (ValidatorSetRegistry)
                    ← btip33 (LinkerNullifier)

BPuN ↔ BPrN 대칭 매핑:
  BTIP21 (LinkerEndpoint on BPuN)    ↔ BTIP29 (LinkerEndpoint on BPrN)
  BTIP22 (LinkerPolicy on BPuN)     ↔ BTIP32 (ValidatorSetRegistry on BPrN)
  BTIP23 (LinkerVerifier on BPuN)   ↔ BTIP31 (LinkerVerifier on BPrN)
  BTIP24 (LinkerNullifier on BPuN)  ↔ BTIP33 (LinkerNullifier on BPrN)
  BTIP26 (dApp Interface on BPuN)   ↔ BTIP34 (Chaincode Interface on BPrN)
```
