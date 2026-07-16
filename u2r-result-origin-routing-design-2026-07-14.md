---
last_updated: 2026-07-14
type: 설계 + 구현 기록 (u2r 결과 라우팅 origin 결속 — correlationId replay 차단)
related: ./linker-v2.md, ./griefing-review-2026-06-24.md, ./btip43-remote-actor-address-design.md, ../../docs/BTIPS/btip-21.md, ../../docs/BTIPS/btip-29.md, ../../docs/BTIPS/btip-24.md, ../../docs/BTIPS/btip-26.md
status: 설계·BTIP 문서·코드 반영 완료 + 정적 빌드/타입체크 통과 (2026-07-14). 라이브 e2e 미실행.
commits: docs=b927ca7 / linker-v2=de752d1(+c41fc3d,0b649c4). task-contexts=미커밋.
---

# u2r 결과 라우팅 origin 결속 (correlationId replay 차단) — 설계 + 구현 기록

> **다른 세션/노트북 인계용 자기완결 문서.** 이 문서 하나로 무엇을·왜·어떻게 바꿨는지, 지금 어디까지 됐는지, 다음에 뭘 하면 되는지 복원 가능. 관련 취약점 분석은 `griefing-review-2026-06-24.md`(별개 트랙 #5a), BTIP-43은 `btip43-remote-actor-address-design.md`. 정본은 **BTIP 문서(docs/BTIPS)와 코드**이고 이 문서는 그 맥락·이유.

## 0. 한 줄 요약 + 현재 상태

u2r(BPuN-origin) 2PC의 **correlationId replay 이중지불**을 막기 위해, 결과 이벤트에 **발행자(emitter) 주소**를 실어 그 발행자에게만 결과를 전달하도록 바꿨다. BTIP 문서·전 코드·테스트 반영 완료, solc/go/tsc 정적 검증 통과. **남은 것은 라이브 e2e뿐.**

## 1. 취약점 (교차검증 완료)

**시나리오:** nft-dApp/BPuN이 환불(`cancelBuyNFT`≈`burnToBPrN`)로 NFT를 burn·pending(cid1)하고 explicit correlationId를 담은 `TransferLogAttrs`를 발행 → 사용자가 증명을 stablecoin-ccApp/BPrN에 제출해 STC 환불(ACCEPTED). 공격자는 자기 `fake-dApp/BPuN`으로 **동일 cid1**을 담은 `TransferLogAttrs`를 발행하고 같은 stablecoin-ccApp에 제출해 **REJECTED** 결과를 얻은 뒤, 이 REJECTED 결과증명을 nft-dApp에 제출 → nft-dApp이 `handleLinkerResult(status=false)`로 **burn한 NFT를 재발행** → STC 환불 + NFT 복원 = **이중지불**.

**성립 이유(코드 대입):** `handleLinkerResult`는 `_pending[cid].exists` + `handlerCcId==_paymentChaincode`(06-24 #5a fix)만 검사. cid1은 indexed topic이라 공개·복사 가능. BPrN `HandleLinkerEvent`는 emitter 화이트리스트 없음(SetTrustedDApp은 06-15 제거). 두 결과는 다른 BPrN tx라 nullifier 충돌 없음. cid 소비는 1회지만 "정확히 하나 적용"일 뿐 "올바른 하나"를 보장 못 함. 제출 순서는 permissionless라 공격자 통제. 재현: 시나리오 03(ACCEPTED payout)+04(REJECTED restore) 통과 경로 조합, 유일 델타는 fake emitter(하니스 `findTransferLogAttrs`가 btip26Token 하드코딩이라 못 냈던 것 — 프로토콜 방어가 아니라 하니스 편의).

## 2. 근본 원인

BPuN `onResult(payload, handlerDApp)`의 `handlerDApp`이 **제출자 지정**이라, 결과가 "누구의 요청에 답하는가"가 프로토콜에서 결속되지 않는다. u2r correlationId는 dApp이 정한 explicit 값(공개·복사가능)이고 엔드포인트가 echo만 한다. r2u는 correlationId=tx_event_root(원 요청 tx에 결속, 복사불가)라 면역.

## 3. 확정 해소 설계 — 방안 A(b): 발행자 주소를 결과에 실어 그쪽으로만 라우팅

**대원칙(명문화):** **"요청을 발행한 컨트랙트가 그 결과를 수신·정산한다."** gateway/vault 분리는 origin(gateway)이 결과를 받아 내부에서 vault로 위임(프로토콜은 앱 내부 구성 불관여). 발행 dApp은 IBTIP26 구현체.

- **u2r (LinkerResultElems, BPrN이 emit → BPuN onResult 소비):** 결과에 **EmitterDApp**(요청 이벤트의 contractAddress) 추가. BPuN `onResult`가 `handlerDApp` 파라미터 제거하고 **EmitterDApp로만 라우팅**. → 위조 결과는 EmitterDApp=fake-dApp이라 fake-dApp으로 라우팅, 피해 dApp 미도달. **이게 실질 보안 수정.**
- **r2u (LinkerResultAttrs, BPuN이 emit → BPrN OnResult 소비):** 대칭으로 **emitterCcAddr** 추가. r2u는 면역이라 라우팅은 caller-supplied `handlerCcId` **유지**하되, **`emitterCcAddr == BTIP9(self채널, handlerCcId)` 검증** 추가(불일치 revert). BTIP9이 channelName을 품어 handlerCcId·self채널 동시 검증. (BPrN은 이름으로 invoke하고 BTIP9은 단방향이라 string 전환 대신 이 forward-hash 대조로 재인덱싱 없이 해결.)
- **dApp(IBTIP26/BTIP26Token)·btip34-ccapp 무변경.**

## 4. 최종 데이터/흐름 (정본)

### LinkerResultElems (BPrN 결과 이벤트, BTIP-29) — DER positional elems
| gidx | elems | 필드 | 타입 | 비고 |
|---|---|---|---|---|
| 4 | 0 | CorrelationId | []byte(32 raw) | |
| 5 | 1 | **EmitterDApp** | []byte(20 raw) | 요청 발행 dApp/BPuN 주소(=요청 contractAddress) |
| 6 | 2 | HandlerCcId | string | 결과 산출 ccApp/BPrN |
| 7 | 3 | Status | byte | 0x01/0x00 |

selector = `sha256("LinkerResultElems([]byte,[]byte,string,byte)")`.

### LinkerResultAttrs (BPuN 결과 이벤트, BTIP-21 — 구 `LinkerResult` 리네임) — EVM event
`event LinkerResultAttrs(bytes32 indexed correlationId, address indexed emitterCcAddr, address indexed handlerDApp, uint8 status)`.
sig = `LinkerResultAttrs(bytes32,address,address,uint8)`. `emitterCcAddr`는 발행 ccApp/BPrN의 **BTIP9 파생 주소**(이름 문자열이 아니라 주소라 `CcId`가 아닌 `CcAddr`).
BTIP-40 attr 인덱스(BPrN OnResult가 읽는): 0=contractAddress, 1=topic0, 2=correlationId, **3=emitterCcAddr**, 4=handlerDApp, 5=data(status).

### 네이밍
- 발행자 필드 = `역할어 emitter + 체인접미사`(handler 관례와 대칭): `EmitterDApp`(BPuN dApp, 주소) / `emitterCcAddr`(BPrN ccApp, BTIP9 주소). 항상 CorrelationId **다음**.
- Go 상수(linker-endpoint): `bprnLinkerResultElemsSig` / `bpunLinkerResultAttrsSig`(둘 다 네트워크 접두사 + 정확한 이벤트명).

## 5. 코드 반영 (전부 커밋 완료 — 아래 커밋 참조)

**docs (BTIP, repo=beatoz/docs, commit `b927ca7`):** btip-21(LinkerResult→LinkerResultAttrs·emitterCcAddr·onResult 시그니처), btip-29(LinkerResultElems+EmitterDApp·selector·OnProof emit·OnResult index shift+emitterCcAddr 검증), btip-24(onResult nullifier `(root,emitterDApp)`), btip-26(LinkerResult→LinkerResultAttrs 리네임).

**linker-v2 (repo=beatoz/linker-v2, commits `de752d1`+`c41fc3d`+`0b649c4`):**
- on-bpun `contracts/interfaces/IBTIP21.sol` — 이벤트·onResult(payload) 시그니처.
- on-bpun `contracts/LinkerEndpoint.sol` — GIDX_RESULT_EMITTER_DAPP=5, selector, onProof emit LinkerResultAttrs(emitterCcAddr=`_btip9Address`), onResult EmitterDApp 라우팅·nullifier(root,emitterDApp), `_btip9Address` 헬퍼. **onProof·BTIP26Token 무변경.**
- on-bpun `contracts/mocks/FakeEmitter.sol` (신규) — arbitrary TransferLogAttrs 발행 fixture(06 테스트용).
- on-bprn `linker-endpoint/main.go` — 상수 리네임, evmAttr 인덱스 shift(+emitterCcAddr=3), OnProof `emitLinkerResult`에 emitterDApp=`attrAddr20At(contractAddress)`, OnResult 라우팅 무결성 검증(`emitterCcAddr==BTIP9(GetChannelID(),handlerCcId)` else `ErrUntrustedResultHandler`)+`attrAddr20At` 헬퍼.
- prover `prover-ts/src/btip19-r2u-txevt/{submitter,cli,server}.ts` — onResult에서 handlerDApp/to_dapp 제거, ABI onResult 인자 축소. **빌더 무변경**(전체 leaf 자동 증명). btip28 무변경.
- test `test/helpers.ts`(ENDPOINT_ABI onResult 축소, submitBpunEndpoint 가변인자, `u2rOnResult(bprnTxId, attrs?)`, `findTransferLogAttrs(txHash, contractAddr?)` 일반화, `forgeTransferLogAttrs`), `03/04/05` u2rOnResult 호출 축소, 신규 `06-u2r-correlation-replay.test.ts`(FAKE_EMITTER env 있을 때만).

## 6. 빌드/검증 상태

- on-bpun (solc/forge): **통과**(사용자). `mocks/FakeEmitter.sol` 포함 확인 권장.
- on-bprn (`go build ./...`): **통과**(§8 resultChainId 정합 후).
- prover-ts (`tsc --noEmit`, clean 전체): **통과**(§8 툴체인 이슈 해소 후).
- test (`tsc --noEmit -p tsconfig.json`): **통과**.
- 라이브 e2e: **미실행**.

## 7. 남은 작업

1. **task-contexts 커밋** — `.git/index.lock` 잔존(마운트 권한). `rm -f task-contexts/.git/index.lock` 후 이 문서 + linker-v2.md 커밋.
2. **라이브 e2e** — selector·이벤트 구조가 바뀌었으므로 **LinkerEndpoint(BPuN)·linker-endpoint(BPrN) 재배포** → 03/04/05 재실행(u2rOnResult 시그니처). **06 replay PoC**는 FakeEmitter 배포 후 `FAKE_EMITTER=<addr>` env 세팅 후 실행. 두 체인 + btip39:sync + btip19/28:api 필요. (06 확인점: FakeEmitter self-funds 분기가 STC 잔액부족 REJECTED를 내는지 라이브에서 확정.)
3. **prover-ts package-lock 커밋** — beatoz-sdk-wrapper develop 재설치로 lock 커밋 해시 갱신됨(§8).
4. **BTIP-43 로드맵 시점** — resultChainId/`btip43Addr(resultChainId, handlerDApp)` 다중 소스 체인은 **문서+코드 함께** 재도입(§8 참조).

## 8. 이번 세션에 함께 처리한 별개 이슈

- **resultChainId 문서 정합**: 커밋 `5735c70 "Pass result chain id to BTIP34 callbacks"`가 `types/ibtip34.go`+endpoint `deliverResult`에 `resultChainId`를 넣었으나 **BTIP-34 문서엔 반영 안 됨** → `btip34-ccapp` 구현(문서 시그니처)과 불일치 → `var _ types.IBTIP34 = (*Contract)(nil)` 컴파일 실패. 사용자 지시로 **문서를 정본** 삼아 `types/ibtip34.go`·`linker-endpoint/deliverResult`에서 `resultChainId` 제거(btip34-ccapp은 원복). BTIP-43(btip43Addr) 구현 시 문서+코드 함께 재도입.
- **btip18/툴체인(u2r 무관, wrapper 재설치가 드러냄):** (a) `btip18-r2u-policy`가 쓰는 `LinkerPolicyClient`가 설치본에 없어 `beatoz-sdk-wrapper@github:beatoz/beatoz-sdk-wrapper#develop` 재설치. (b) 재설치가 typescript를 5.9.3으로 올려 `baseUrl` deprecation(TS5101) → `prover-ts/tsconfig.json`에서 무의미한 `baseUrl` 제거. (c) `bprn-block-listener.ts`의 `qsccClient`가 async-init 패턴(`await Qscc.create`)이라 `strictPropertyInitialization`에 걸림 → definite assignment `private qsccClient!: Qscc;`.

## 9. 산출물 / 커밋

- 정본 스펙: `docs/BTIPS/btip-21,29,24,26.md` (commit `b927ca7`).
- 코드: `linker-v2` (commits `de752d1`, `c41fc3d`, `0b649c4`) — working tree 클린.
- 본 설계·구현 기록: `task-contexts/u2r-result-origin-routing-design-2026-07-14.md` (**미커밋**).
- 인접: `griefing-review-2026-06-24.md`(#5a handlerCcId — 본 건은 그 신뢰 핸들러를 정상 사용하는 별개 벡터), `btip43-remote-actor-address-design.md`.
