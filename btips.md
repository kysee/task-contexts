---
last_updated: 2026-04-12
---

# BTIPS 작업 컨텍스트

---

## 작업 개요

BTIP19의 **단일 경로 신뢰 모델(Single-Path Trust Model)** 재작성에 따른 관련 BTIP 문서들의 교차 참조 정합성 검증 및 수정 작업.

이전 세션들에서 btip-19 구조 정리(재설명 제거, Abstract 재작성, MerkleProof 구조 재설계), 필드명 시맨틱 교정, 제네릭 타입 표기법 도입, Proof Construction Algorithm 수정, Verification Procedure 재구성, 필드명 일괄 교정, Trust Establishment 통합, 인터페이스 정리 등을 완료.

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

---

## 세션별 완료 작업

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
- **BPuN**: Tendermint/DPoS 기반 퍼블릭 네트워크
- **BEATOZ Linker Protocol V2**: BPrN → BPuN 이벤트 전달 프로토콜

### 관련 문서 의존성
```
btip16 ← btip17 ← btip19 ← btip20
                           ← btip21 (LinkerEndpoint)
                                ← btip26 (dApp 콜백 인터페이스)
                           ← btip23 (LinkerVerifier)
                           ← btip24 (LinkerNullifier)
```
