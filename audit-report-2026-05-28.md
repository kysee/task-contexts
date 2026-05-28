---
last_updated: 2026-05-28
type: 점검 리포트 (code ↔ doc 정합성)
related: ./linker-v2.md, ./btips.md, ./btips-2pc-design.md, ./bpun-origin-payment-design.md
---

# Linker Protocol V2 — 전체 점검 리포트

> 컨텍스트 문서(`linker-v2.md`, `btips.md`, `btips-2pc-design.md`, `bpun-origin-payment-design.md`)와 BTIPS 문서, 실제 리포지토리 코드를 대조한 정합성 점검. 코드 상태는 git으로 검증함.

## 점검 근거

- **linker-v2 리포** (`/Users/kysee/projects/beatoz/linker-v2`): 현재 브랜치 `feat/kyle`, HEAD `678d245 feat: Add BTIP32 interface` (2026-05-26 16:13), 워킹트리 clean, stash 없음.
- 검증 범위: `feat/kyle` 워킹트리 + 모든 로컬/원격 브랜치(`main`, `develop`, `fix/block_commit_signer`, `feat/linkerpolicy`)에 대해 핵심 산출물 존재 여부를 `git cat-file`/`git grep`으로 교차 확인.
- **docs 리포** (`/Users/kysee/projects/beatoz/docs`): 별도 git, 브랜치 `feat/2pc`, HEAD `f21c287 docs: Add 'handler address' to handleLinkerResult`.
- 참고: 컨텍스트 문서 내 경로가 `/Users/kylekwon/...`로 적혀 있으나 실제 마운트는 `/Users/kysee/...`. 단순 stale 경로(기능 영향 없음).

## 핵심 결론 (3줄)

1. **BTIPS 설계 문서는 가장 앞서 있고 정합적**이다. 2PC·BTIP-37 통합 레지스트리·BTIP-39·secp256k1까지 명세가 닫혀 있다.
2. **`linker-v2.md`(구현 현황 문서)는 코드와 크게 어긋나 있다.** "✅ 완료"로 기록된 다수 작업이 **어느 브랜치에도 존재하지 않으며**, 반대로 코드에 실재하는 작업이 문서에 누락돼 있다. → 현 시점 `linker-v2.md`를 구현 현황의 근거로 신뢰하기 어렵다.
3. **end-to-end의 결정적 미싱 피스는 그대로**다: BPuN→BPrN 이벤트 Prover(`BPuNTxEventProofPayload` 생성기) 미구현 + on-bprn BTIP-39 검증 로직이 stub.

---

## 컴포넌트별 상태

### on-bprn (Go 체인코드) — `feat/kyle`

| 항목 | 문서(linker-v2.md) 주장 | 실제 코드 | 판정 |
|------|------------------------|-----------|------|
| 서명 검증 알고리즘 | secp256k1 교정(ed25519 제거) 완료 | `linker-verifier/main.go`가 `crypto/ed25519` import + `ed25519.Verify` 사용 | ❌ 불일치 (코드 미반영) |
| ValidatorSet 표현 | 커스텀 타입 제거 → tmproto.ValidatorSet + hex DTO | `types/types.go`의 커스텀 `ValidatorSet`/`Validator` 그대로, `validatorset_dto.go` 없음 | ❌ 불일치 |
| 공유 검증 헬퍼 | `types/tmverify.go` 신설 | 파일 없음 (검증은 verifier에 인라인) | ❌ 없음 |
| 레지스트리 리팩토링 | `types/registry.go` + `SetRegistryID`/`ResolveContract` 통합, 개별 setter 제거 | `registry.go` 없음, 개별 setter(`SetVerifierChaincodeID` 등) 전부 잔존 | ❌ 없음 |
| BTIP-37 Go(linker-registry 체인코드 + `ibtip37.go`) | 신규 구현 | 디렉토리/파일 모두 없음 | ❌ 없음 |
| BTIP-39 `UpdateValidatorSet` | 스텁→5단계 검증 구현 | `linker-policy/main.go`에서 `panic("implement me")` | ❌ 스텁 |
| BTIP-32 저장소(`SetValidatorSet`/`GetValidatorSet`/`GetValidator`) | — | 구현됨 | ✅ |
| 2PC(`OnResult`/`HandleLinkerResult`) | (btips 문서엔 설계됨) | endpoint/dapp 어디에도 없음 | ❌ 없음 |

> **전 브랜치 확인 결과**: `registry.go`, `ibtip37.go`, `linker-registry/`, verifier의 secp256k1은 `main`/`develop`/`fix/block_commit_signer`/`feat/linkerpolicy`/원격 포함 **어느 브랜치에도 존재하지 않음.** 즉 `linker-v2.md`의 "2026-05-26 구현 세션"(L609-698) 산출물은 현 리포에 커밋된 적이 없다.

### on-bpun (Solidity) — `feat/kyle`

문서에 없는 **실재 코드(문서가 뒤처짐)**:
- 본격 정책 엔진이 들어왔다: `LinkerPolicyVerifier.sol`, `SignatureVerifier.sol`, `LinkerPolicyLib.sol`, `LinkerPolicyTypes.sol`, `contracts/lib/{MSPRoleLib,P256Verify,X509Verify}.sol`. Fabric 채널 config 기반 endorsement policy 평가(`signatureRuleTree`, `implicitMetaPolicy`)를 수행 — `linker-v2.md`의 "`abi.decode(policy,(uint256,bytes[]))`" 수준보다 훨씬 앞선 재작성.
- `feat/kyle`·`develop`·`feat/linkerpolicy`에 존재, `main`에는 없음.

문서 주장 대비 **없는 것**:
- 2PC: `event LinkerResult` / `onResult` / `handleLinkerResult` / `try-catch` / `nonReentrant` — **전부 없음.** `IBTIP21`은 여전히 구버전(`onProof(TxEventProof,targetDApp)`, 이벤트 `ProofReceived`/`ProofVerified`).
- BTIP-37: `IBTIP37.sol` / `LinkerRegistry.sol` 없음.

존재 확인된 것: 4개 핵심 컨트랙트 + `BTIP26Dapp`(구 MockDApp 리네임), Foundry 가스 환경(`foundry.toml`, `test-forge/LinkerGasTest.t.sol`, mocks), `scripts/beatoz/{deploy,set-policy,submit-proof,cancel-event,utils,setup.sh}` + `send-op-tx.ts`. (`query-dapp.ts`는 삭제됨 — 문서와 일치)

### prover-ts — `feat/kyle`

| 항목 | 문서 주장 | 실제 코드 | 판정 |
|------|----------|-----------|------|
| Fabric SDK | `fabric-network` v2.2.20 전용 (gateway 아님) | `@hyperledger/fabric-gateway`(1.7.0) + `fabric-network`(2.2.20) + `@peculiar/x509` + grpc 병존 | ⚠️ 리팩토링됨(문서와 다름) |
| BTIP-39 Prover (`src/prover/btip39/`) | 신규 작성, `npm run btip39:scan` | 디렉토리 없음, 스크립트 없음 | ❌ 없음 |
| BPuN→BPrN 이벤트 Prover | (미구현으로 기록) | `src/prover/u2r/` 빈 디렉토리(placeholder) | ❌ 미구현 (확인) |
| BPrN→BPuN forward prover | 구현됨 | `prover.service.ts` + `common/{event-log,merkle}.ts` + `scripts/test-proof.ts` 존재 | ✅ (단, 구조 일부 변동) |

### BTIPS 설계 문서 (docs 리포, `feat/2pc`)

`btips.md` 세션 로그대로 정합 작업이 반영돼 있고 가장 최신. README에 BTIP-37/38(Reserved)/39 등재 완료, 2PC 핸들러 바인딩(`handleLinkerResult`에 handler 인자) 반영(docs HEAD 커밋). **문서는 코드보다 앞선 상태**이며, 코드가 따라가야 할 사양의 기준점으로 신뢰 가능.

---

## 가장 중요한 시사점

`linker-v2.md`는 두 방향 모두에서 부정확하다:

- **문서엔 ✅, 코드엔 없음**: on-bprn 2026-05-26 구현 세션 전체(secp256k1, tmproto/DTO 마이그레이션, tmverify.go, registry.go, linker-registry 체인코드, BTIP-39 5단계 검증), BTIP-39 prover, prover의 multi-peer block-commit-sig 수집.
- **코드엔 있음, 문서엔 없음**: on-bpun 정책 엔진(LinkerPolicyVerifier/SignatureVerifier/lib mocks), prover-ts의 fabric-gateway 도입, `u2r` placeholder.

가능한 원인: 해당 작업이 푸시되지 않은 다른 로컬 클론에서 진행됐거나, 문서가 의도(설계)를 "완료"로 기록했을 수 있음. 어느 경우든 **현재 이 리포에는 없다.** → 다음 작업 착수 전 `linker-v2.md`를 실제 코드 기준으로 재동기화하는 것을 권장.

## 권장 다음 단계 (우선순위)

1. **`linker-v2.md` 현황 재동기화** — 본 리포트 기준으로 "완료/미완료"를 실제 코드에 맞춰 정정. (이후 모든 작업의 신뢰 기준)
2. **on-bprn 핵심 갭 구현** (end-to-end 차단 요소):
   - (a) verifier 서명 검증 ed25519 → **secp256k1** 교정 (BEATOZ는 secp256k1 전용 — 현 코드는 실데이터에서 검증 실패함).
   - (b) BTIP-39 `UpdateValidatorSet` stub → 실제 5단계 검증 구현.
   - (c) BTIP-37 레지스트리 리팩토링(registry.go + linker-registry 체인코드 + 개별 setter 통합).
3. **BPuN→BPrN 이벤트 Prover 구현** (`u2r` placeholder 채우기) — end-to-end의 결정적 미싱 피스.
4. (선택) on-bpun/on-bprn 2PC 구현 — 문서엔 설계 완료, 코드 미착수. 위 1~3 이후.

> 본 리포트는 점검만 수행했으며 코드/문서 수정은 하지 않음.
