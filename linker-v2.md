# Linker V2 Solidity 작업 컨텍스트

> 마지막 업데이트: 2026-06-24 (세션 종료 시점) — **아래 §000000(2026-06-24 핸드오프)부터 읽기.** 직전 핸드오프는 §00000(2026-06-18), §0000(2026-06-15), §000(2026-06-14), §00(2026-06-12), 직전 기준선은 §0(2026-06-10).

---

## 000000. 2026-06-24 핸드오프 (여기서부터 이어서 읽기)

> 이 세션 본류: **보안 점검(Axelar/Secret ICS20 사고 대조) + BTIP-43 신설 + griefing(#5a) u2r 갭 수정**. 다음 본류는 여전히 §00000.5 allowance 테스트 실행(BTIP-43 코드 구현은 다중 소스 체인 로드맵 시점).

### 000000.1 한 일

- **보안 분석 문서 신설**: `axelar-ics20-vs-linker-v2-2026-06-24.md` — Axelar/Secret CW20-ICS20 익스플로잇(소스 채널 검증 누락 → 무담보 발행)을 기준으로 Linker V2 공격 트리(프로토콜이 막는 것 vs 앱 책임) + 약점 영향 분석. **말미 "점검 결과" 절 = 최종 결론**: 통제된 단일-BPrN + chaincode 승인 규율 + initPolicy onlyOwner 전제 하에 **Axelar 사고는 재현되지 않음**(출처 검증이 "수신 dApp 코드" → "배포 승인 거버넌스"로 이동, 통제 환경에서 충분). 핵심 통찰: 이벤트 발행 ≠ 자산 이동. 전제(거버넌스 불변식)는 BTIP/운영문서에 명시 필요.
- **BTIP-43 신설(Draft)**: `docs/BTIPS/btip-43.md` "Remote Actor Address Derivation on BPrN". README 인덱스 등재 완료. 다중 소스 체인 시 "동일 주소·다른 컨트롤러"(주소충돌)를 막기 위해 원격 BPuN 행위자를 `btip43Addr(srcChainId, addr)=address(sha256(srcChainId‖addr)[12:32])`로 식별.
- **설계 기록(자기완결 인계)**: `btip43-remote-actor-address-design.md` — BTIP-43 발단·진화·결정·현재상태·구현영향·남은작업 전부. **다음 세션은 이 문서부터 읽으면 BTIP-43 맥락 복원 가능.**
- **griefing(#5a) u2r 갭 수정 (코드+스펙)**: BTIP26Token의 `handleLinkerResult`가 결과 핸들러를 대조 안 하던 비대칭 갭(r2u btip34-ccapp은 하는데 u2r은 안 함) 해소. 수정: `handlerCcId == _paymentChaincode`(= r2u 결제 출처 STC) 대조 + 불일치 시 `ErrUntrustedResultHandler` revert. `handleCcId`→`handlerCcId` 명칭 통일(btip-26 인터페이스/본문, IBTIP26.sol, BTIP26Token.sol). btip-26 §2PC line 126을 "출처 핸들러 한정하려면 기록·대조해야 한다"(조건부 의무)로 강화. 전제: u2r 핸들러=r2u 출처(같은 STC) — 코드 주석에 명시. **자기완결 기록: `griefing-review-2026-06-24.md`.**

### 000000.2 핵심 발견 (추적 필요)

- **(HIGH, 조치 완료) `LinkerPolicy.sol` `initPolicy` 접근제어 부재** → front-run으로 신뢰 루트(Root CA) 탈취 가능했음. `onlyOwner` 적용 + deploy+init 원자화로 해소.
- 현 구현은 **1 BPuN : 1 BPrN**(policy가 단일 소스 체인). BTIP-43 결함은 다중 소스 체인 전제라 **현재 악용 불가**, 다중 체인 지원 시 필요.

### 000000.3 남은 작업

- **btip-34 → btip-43 상호참조 포인터 추가** — `btip-34.md`에 BTIP-43 참조 링크 한 줄. (진행 예정; 설계 기록 §7)
- **btip-34 Approve 인터페이스 확장** — spender 소스 체인 파라미터 추가: `Approve(owner, spenderChainId, spender, amount)`, 키 = `btip43Addr(spenderChainId, spender)`. `Allowance`도 대칭. (설계 기록 §7; "toChainId"는 PayToBPuN용이고 Approve는 `spenderChainId`)
- BTIP-43 코드 구현 전반(다중 소스 체인 로드맵 시점) — 구현영향은 설계 기록 §6.

---

## 00000. 2026-06-18 핸드오프 (여기서부터 이어서 읽기)

> 이 세션 본류: **Vitest 통합 테스트 하니스 정비** — (A) test/를 prover **API(증명 생성)만 호출 + 제출은 직접** 하는 구조로 리팩터링, (B) test env 파일 네이밍을 `bprn`/`bpun` 접두사화, (C) ID(시스템 코드값) vs Alias(설정 선택자) 용어 혼용 정리. **다음 본류는 여전히 §0000.5 allowance 테스트 실행**(BTIP26Token 재배포 선행).

### 00000.1 test/ Vitest 하니스 (linker-v2/test) — 위치·구성

- **`linker-v2/test/`** (verifier·prover-ts 둘 다 호출하므로 최상위). 파일: `run.mjs`, `config.ts`, `helpers.ts`, `01-r2u-accepted.test.ts`(시나리오1만 구현), `vitest.config.ts`(순차: fileParallelism=false, singleFork, 180s), `README.md`(4 시나리오 명세).
- 실행: **`npm run test -- <bprnChannelAlias> <bpunChainAlias>`** (예: `npm run test -- bpn localnet0`). run.mjs가 `TEST_BPRN_ALIAS`/`TEST_BPUN_ALIAS` 세팅 → `npx vitest run`.
- config.ts: **prover-ts/.env, deployed.*.json 의존 끊음.** 컨트랙트 주소·키·cc명을 env 파일에 인라인, ABI는 최소 fragment(`BTIP26_ABI`)로 빌드 아티팩트 의존도 제거.

### 00000.2 prover API 호출 + 직접 제출 리팩터링 (이 세션 핵심)

- **변경 전**: helpers가 prover-ts `buildTxEventProof` + `EndpointSubmitter`를 in-process로 사용(증명 생성·제출 둘 다 prover-ts).
- **변경 후**: **증명 생성 = prover HTTP `GET /proof`, 제출 = test가 직접**(이미 양 체인에 접속해 dApp/ccApp tx 제출 중이므로).
  - **btip19:api(:3019, BPrN 증명)** `GET /proof?tx_id=&channel=&attrs=` → `payload`(BPrNTxEventProof). int64 안전 → `JSON.parse(text).payload`. 제출: **web3로 BPuN `LinkerEndpoint.onProof/onResult(toSolidityProof(payload), handlerDApp)`**. 엔드포인트 주소 = env `BPUN_LINKER_ENDPOINT`. `ENDPOINT_ABI`·`toSolidityProof`는 helpers에 **인라인 복제**(prover-ts submitter 동일).
  - **btip28:api(:3028, BPuN 증명)** `GET /proof?tx_hash=&event_index=` → `payload`(BPuNTxEventProof). ⚠️ **int64 정밀도 함정**: payload의 validator vote timestamp(Unix ns)가 `Number.MAX_SAFE_INTEGER` 초과 → `JSON.parse`하면 손상. 응답 텍스트에서 `payload` 객체를 **파싱 없이 raw 추출**(`extractRawObjectField` = 중괄호 밸런싱, 문자열 이스케이프 처리)해 그대로 fabric에 전달. 제출: **fabric `LinkerEndpointCC.OnProof/OnResult(rawPayloadJson, handlerCcId)`**. 엔드포인트 cc는 `resolveContractByRole(registryCc, 'LinkerEndpointRole')`.
- helpers 시그니처: `r2uOnProof(bprnTxId,toDapp,attrs)→{txHash,height,endpoint}`; `r2uOnResult(fabric,bpunTxHash,eventIndex,toCcapp)`; `u2rOnProof(fabric,...)`; `u2rOnResult(bprnTxId,toDapp,attrs)`. **BPuN 제출=fabric 불필요, BPrN 제출=fabric 필요**.
- config.ts: `CFG.prover.btip19Url`(BPrN env), `CFG.prover.btip28Url`(BPuN env) 추가. 불필요해진 `bpunSubmitConfig`/`btip28Config` 팩토리 제거(`fabricConnectionConfig`만 유지).
- **새 전제조건**: e2e에 btip19:api/btip28:api **서버 실행 필요**(README 전제조건 반영). `BTIP19_API_URL`/`BTIP28_API_URL` env 추가. `tsc --noEmit` 통과(Node 18+ global fetch).

### 00000.3 test env 파일 네이밍: `bprn`/`bpun` 접두사

- `.env.<alias>` → **`.env.bprn.<bprnChannelAlias>` / `.env.bpun.<bpunChainAlias>`**(접두사로 체인 쪽 즉시 구분).
- config.ts `loadEnvFile(side, alias)` → `.env.${side}.${alias}`. run.mjs는 불변(위치 인자 2개).
- 실파일(gitignore): `.env.bprn.bpn`, `.env.bpun.localnet0`. 예시(추적): `.env.bprn.channelAlias.example`, `.env.bpun.chainAlias.example`. `.gitignore`(`.env.*`, `!.env.*.example`) 그대로 커버.

### 00000.4 ID(코드값) vs Alias(선택자) 용어 정리

- **원칙**: ID/Name = 블록체인 시스템이 인지하는 코드값(chainId `0x1234`, Fabric 채널명 `bpn`). Alias = 설정 파일을 고르는 운영자용 명칭(`localnet0`). 값이 우연히 같아도(채널 alias `bpn` == 채널명 `bpn`) 개념은 다름.
- **발견된 혼용**: on-bpun bootstrap 2nd 인자 `bprnChannelId`는 실은 `config.<X>.json` **선택자(=alias)**, 실제 채널명은 그 config의 `counterpartBPrN.channel`/`paymentChannel` 필드. 1st 인자 `<chain>`("chain name")도 실은 alias(실 chain id는 `config.chainId`).
- **수정**: `bprnChannelId`→**`bprnChannelAlias`**, 사용자 확정 (b)로 **on-bpun 스크립트 9개 전체 `chainAlias`→`bpunChainAlias`**(setup.sh `CHAIN_ALIAS`→`BPUN_CHAIN_ALIAS` 포함). test 명명과 대칭(`bpunChainAlias`/`bprnChannelAlias`). **인자 위치·값 불변 → 실행 방식 동일.**
- **on-bprn은 alias로 감싸지 않음**: `--channel <name>`=실제 채널명(`getNetwork`에 직접 전달), `--network <name>`=org 디렉토리 선택자. BPrN쪽엔 channelAlias 개념 없음(그대로 둠).
- **올바른 용법이라 유지**: `BPUN_CHAIN_ID`, `config.chainId`, `counterpartBPuN.chainId`, 최상위 README의 chainId/channelName 값.
- 수정 파일: `setup.sh`, on-bpun `bootstrap.ts`/`deploy.ts`/`nft.ts`/`burn-nft.ts`/`submit-proof.ts`/`cancel-event.ts`/`utils.ts`, on-bpun `scripts/beatoz/README.md`, `test/README.md`.

### 00000.5 다음 작업 / 미해결

- **본류 변함없음 = §0000.5 allowance 테스트**: BTIP26Token(`burnToBPrNFrom`) 재컴파일+재배포 선행, 재배포 시 counterpartBPuN/BPrN 갱신.
- test 시나리오: **01(r2u accepted)만 구현**. 2(r2u REJECTED/refund), 3(u2r self-funds accepted), 4(u2r delegated 미승인 REJECTED) 미구현(README 명세). helpers에 `u2rOnProof`/`u2rOnResult`·`stcApprove`/`stcMint`·`findEventIndexByContract` 준비됨.
- **go/solc 미검증**(샌드박스). test `tsc`만 통과. e2e 실행엔 두 체인 + `btip39:sync` + **btip19:api/btip28:api 서버** 필요.
- git 미커밋(리포별 사용자 커밋).

---

## 0000. 2026-06-15 핸드오프 (여기서부터 이어서 읽기)

> 이 세션의 본류: **(A) btip34-ccapp을 allowance 결제 모형으로 재작성, (B) BPuN deploy/bootstrap 분리 + 양쪽 cross-chain endpoint 등록, (C) localnet 양방향(r2u+u2r) 2PC e2e 전부 성공, (D) allowance 테스트 준비(burnToBPrNFrom 추가).** 다음 세션 본류 = **allowance 테스트 실행**(아래 §0000.5).

### 0000.1 네이밍/디렉토리 통일 (전부 "bpn" 채널 기준)

- **BPrN 크립토 디렉토리**: `bprn-organizations/localnet/` → **`bprn-organizations/bpn/`** (`--network bpn`이 크립토+config 둘 다 선택). prover-ts `.env`/`.env.example`, on-bprn README, link-test-env.sh 주석 모두 갱신.
- **BPrN 부트스트랩 config**: `bootstrap.localnet.json` → **`bootstrap.bpn.json`**(+`.example`). on-bprn bootstrap은 `--network bpn` → `bprn-organizations/bpn/` + `bootstrap.bpn.json` 자동 선택.
- **BPuN policy/payment config**: `config.bprn.json` → **`config.bpn.json`**(genesisPolicy + paymentChannel + paymentChaincode + counterpartBPrN 보유). `.gitignore`에 `!**/config.bpn.json` 예외(비밀 아님). 구 `config.bprn.json`은 권한으로 못 지움 → 사용자 `rm` 필요.
- **BPuN 체인 config**: `config.localnet0.json` 신규(chainId `0x1234`, providerUrl `http://127.0.0.1:26657`, deployerPrvKey). gitignore 대상.
- ⚠️ **잔재 삭제(사용자)**: `verifier/on-bpun/scripts/.net/config.bprn.json`, `verifier/on-bpun/scripts/beatoz/init-policy.ts`(deprecated 스텁), `verifier/on-bprn/scripts/mint-once.ts`(임시) — 마운트 권한으로 자동삭제 실패, `rm`은 사용자가.

### 0000.2 btip34-ccapp 재작성 (allowance 결제 모형) — 핵심

trusted-source/mint → **ERC20 allowance 결제**로 재작성(설계: bpun-origin-payment-design 결정 (iv)·§15.4). `SetTrustedDApp`·화이트리스트·mint-credit 제거.

- **U2R `HandleLinkerEvent`**: STC를 **`from` → `beneficiary`** 전송. 인가 = `from == msgSender`(self-funds, **approve 불필요**) 또는 `allowance[from][msgSender] >= amount`(차감). `msgSender` = TransferLogAttrs의 contractAddress(=이벤트 emit한 컨트랙트 = 크로스체인 caller). 잔액·allowance·amount=0 부족 = business rejection(Status=false→REJECTED), 형식·topic0 불일치 = hard error. **변수명 `requestedBy`→`msgSender` 확정**(2026-06-15 사용자 합의: emit=상대체인 HandleLinkerEvent 호출로 보면 emitter가 곧 msg.sender. spender는 self-funds 분기 못 담음).
- **R2U `PayToBPuN(from, to, amount, beneficiary, memo)`**: `from` 명시. from→to pending 잠금, TransferLogElems emit. `HandleLinkerResult`: ACCEPTED→`to`에 settle, REJECTED→`from` 환불.
- **신설**: `Approve(owner, spender, amount)`, `Allowance(owner, spender)`, `Transfer(from,to,amount)` from 명시. `Mint`(테스트 시드) 유지.
- 메모리 참조: [[project-btip34-ccapp-allowance-redesign]].

### 0000.3 BPuN deploy/bootstrap 분리 + 양쪽 cross-chain endpoint 등록

- **deploy.ts = 배포 전용**(6개 컨트랙트 배포·주소 저장만). **bootstrap.ts 신규** = 와이어링(setSelfContract×4, setRegistry×4) + setPaymentSource + initPolicy(config.bpn.json genesisPolicy) + **counterpartBPrN 등록** + registry 검증. 인자 `<chainAlias> <bprnChannelId> [--dry-run]`. setup.sh = deploy→bootstrap, `<chain> <bprnChannelId>`. init-policy.ts deprecated.
- **cross-chain endpoint 등록(양방향 대칭) — e2e에서 발견한 필수 단계**:
  - **BPrN bootstrap → `counterpartBPuN`**: OnResult가 `GetContract(BPuN_chainId, "LinkerEndpointRole")`로 BPuN endpoint resolve해 LinkerResult 출처 검증. `bootstrap.bpn.json`에 `counterpartBPuN{chainId, linkerEndpoint(=BPuN endpoint addr)}` → `SetContract(chainId, addr, LinkerEndpointRole)`.
  - **BPuN bootstrap → `counterpartBPrN`**: onResult가 `getContract(uint256(sha256(channel+"/BPrN")), keccak256("LinkerEndpoint"))`로 BPrN endpoint resolve. `config.bpn.json`에 `counterpartBPrN{channel, endpointCc}` → 파생값 계산해 `setContract(chainId, role, BTIP9addr=sha256(channel+"-"+endpointCc)[12:])`.

### 0000.4 e2e 양방향 성공 (localnet0 ↔ bpn) + 고친 함정들

**r2u(BPrN-origin)**: PayToBPuN → btip19:cli `--attrs 0,1,3,5,6,7 --to-dapp <BTIP26Token>`(BPuN onProof, NFT 민트) → btip28:cli `--event-index 4 --to-ccapp BTIP34CCApp --on-result`(BPrN settle). **u2r(BPuN-origin)**: burn-nft → btip28:cli `--event-index 2 --to-ccapp BTIP34CCApp`(BPrN OnProof, self-funds 전송) → btip19:cli `<BPrN txId> --to-dapp <BTIP26Token> --on-result`(BPuN BurnFinalized). 값 보존 확인.

함정(전부 코드 반영): ① cross-chain endpoint 등록 누락(§0000.3). ② **btip28 null-sibling 와이어**: JSON `null`이 contractapi `[]string` 스키마에서 거부 → `""`로 보내야(Go HexBytes가 ""·null 둘 다 nil; tx-event-proof.ts toWire). ③ **btip28 `--event-index`는 ABCI 이벤트 전체(비-evm `tx` 포함) 0-based 카운트**. ④ **BTIP26Token.burnToBPrN의 from=address(this)**(msg.sender 아님): self-funds 분기 → approve 불필요, to/beneficiary=msg.sender, to/beneficiary 파라미터 제거. ⑤ **burn-nft.ts memo(bytes)는 hex 문자열**(`'0x'+hex`)로(Node Buffer 직접 전달 시 web3 ABI 거부). ⑥ btip28 burn tx: TransferLogAttrs=data 있는 evm(attrs=8), ERC721 소각 Transfer=전부 indexed(attrs=7), 보통 index 2.

prover-ts `.env`(gitignore): `FABRIC_*`→bprn-organizations/bpn, `LINKER_REGISTRY_CC=LinkerRegistryCC`(CC 접미사!), `BPUN_CHAIN_ID=0x1234`, `BPUN_LINKER_REGISTRY=<BPuN registry>`, `BPUN_PRIVATE_KEY=<deployer>`. btip39:sync 데몬 상시 실행. 메모리: [[project-r2u-e2e-localnet-2026-06-15]].

### 0000.5 다음 작업 — allowance 테스트 (본류, 미실행)

준비 완료: **`BTIP26Token.burnToBPrNFrom(tokenIds, from, memo)`** 추가(burnToBPrN 동일 + from 파라미터로 emit → `from != contractAddress` → BPrN delegated 분기). `burn-nft.ts --from <addr>` 옵션(주면 burnToBPrNFrom). `approve.ts`(on-bprn) 신규.

**미실행 — 다음 세션 본류**: burnToBPrNFrom은 **BTIP26Token 재컴파일+재배포 필요**. 절차: ① BTIP26Token 재배포(+setRegistry/setPaymentSource; setup.sh 전체면 LinkerEndpoint 주소 바뀌어 counterpartBPuN/BPrN 갱신+재부트스트랩) ② r2u로 새 BTIP26Token에 NFT 민트 ③ payer A에 STC + `approve --owner A --spender <BTIP26Token> --amount`(approve.ts) ④ `burn-nft.ts ... --from A` ⑤ btip28 `--to-ccapp BTIP34CCApp`(delegated 분기, allowance 차감) ⑥ btip19 `--on-result`. 검증: `balance --addr A --spender <BTIP26Token>`로 allowance 소진 확인. **REJECT 경로**: approve 안 한 주소를 `--from`으로 → allowance=0 → business rejection → NFT 복원.

### 0000.6 현재 배포 주소 / 계정 (localnet0, chainId 0x1234)

- BPuN: LinkerRegistry `0xfc487a5db9977aafebbe786587f5258a5de8e75f`, LinkerEndpoint `0x2a534f1f32267923f448174a9d8c79a20c765b2e`, BTIP26Token `0x618cbe55828113d9cc654c5e6a13195a9cf9fd22`(**burnToBPrNFrom 재배포 시 변경**).
- BPrN(channel `bpn`) cc: LinkerRegistryCC / LinkerEndpointCC / LinkerVerifierCC / LinkerNullifierCC / LinkerPolicyCC / BTIP34CCApp.
- 테스트 계정 `0x3FD3E78544B064B0B3FC1F11DC7FB065D80CD52F` = BPuN deployer = NFT 소유자 = STC 홀더(부트스트랩 Mint 대상, mint.amount 10e18).

### 0000.7 보조 스크립트 / git

- on-bprn/scripts: `pay.ts`·`balance.ts`·`approve.ts`(정식, npm run) + `mint-once.ts`(임시) + `README.md`(스크립트별 usage+파라미터). on-bpun/scripts/beatoz: `nft.ts`·`bootstrap.ts` 신규 + `README.md`. btip28/btip19 제출 시 txId/txHash 출력 + 다음명령 힌트(pay/burn-nft/btip19/btip28).
- ⚠️ **go build 미검증**(샌드박스 Go 없음): btip34-ccapp(msgSender 리네임)·BTIP26Token(burnToBPrNFrom, burnToBPrN from 변경) 재컴파일/빌드는 사용자 머신에서. 단순 리네임/추가라 영향 적음.
- **git 전부 미커밋**(본 세션 변경 다수). 리포별로 사용자가 직접 커밋.

---

## 000. 2026-06-14 핸드오프

> 이 세션은 (A) stale 메모 정정, (B) `bprn-organizations` 디렉토리 재배치 + connection-profile 상대경로화, (C) on-bprn 부트스트랩 스크립트 작성을 수행. 다음 세션은 **localnet e2e**(§00.4 2번)가 본류.

### 000.1 이 세션에서 한 일 (요약)

**(A) stale 메모 정정 — 코드가 노트보다 앞서 있었음 (사용자 직접 병합)**

- 발견: `LinkerPolicyVerifier`(별도 컨트랙트)는 **폐지**되고 `LinkerPolicy.sol`이 `contract LinkerPolicy is IBTIP22`로 **Trust Anchor 데이터 + IBTIP22 검증을 단일 컨트랙트**에 보유(검증 로직은 `lib/LinkerPolicyVerifierLib.sol` library). 이 병합이 모든 노트(2026-06-12 핸드오프 포함)보다 최신이라, 노트·주석이 옛 2-컨트랙트 구조를 설명하고 있었음.
- 따라서 §00.4 보류 항목 중 **IBTIP22 메소드명 불일치·LINKER_POLICY 바인딩 재설계는 실제로 해소됨**: registry의 LINKER_POLICY role이 deploy.ts에서 `LinkerPolicy`를 가리키고(=데이터 컨트랙트 기준 자연 충족), 메소드명도 LinkerVerifier·LinkerPolicy 양쪽이 스펙명 `verifyChannelEndorsementPolicy`로 일치.
- **정정한 파일**: `verifier/on-bpun/scripts/beatoz/deploy.ts`(LINKER_POLICY 주석을 실제 동작으로 교체), 본 문서 §00.4·§0.4(정정 마커 추가). **잔재**: `verifier/on-bpun/scripts/.net/deployed.localnet0.LinkerPolicyVerifier.json`은 소스 없는 옛 배포 기록 → 삭제 후보.

**(B) `bprn-organizations` 재배치 + connection-profile 상대경로화**

- **이동**: `prover-ts/bprn-organizations/` → **`linker-v2/bprn-organizations/`**(최상위 공유 루트). 이유: prover-ts + verifier/on-bprn/scripts 둘 다 소비 → 교차 패키지 상대경로/심볼릭 회피. 심볼릭 링크 4개(절대경로 타깃) 생존.
- **connection-profile.json 상대경로화**: `tlsCACerts.path`를 `peerOrganizations/...`/`ordererOrganizations/...` 상대경로로 변경(stale `/Users/kylekwon/...` 제거). 멀티네트워크 이식성 확보.
- **로더 수정(결합 필수)**: `prover-ts/src/common/fabric.ts` `FabricClient.connect`에 `resolveProfileTlsPaths()` 추가 — 상대 CA 경로를 **프로파일 파일 디렉토리 기준**으로 절대화(절대경로는 그대로, 하위호환). 이걸 안 하면 fabric-network가 CWD 기준으로 풀어 **prover가 깨짐**. tsc 클린(유일 에러는 기존 `app.module.ts`).
- **link 스크립트 재배치**: `bprn-organizations/link-test-env.sh`(신규) — `<network>` 파라미터화, DST=`<script-dir>/<network>`, 기본 src `~/go/src/github.com/beatoz/bpn-core-2.2/bprn-test-env`(kysee). 구 `prover-ts/scripts/link-bprn-test-env.sh`는 마운트 권한으로 삭제 불가 → **이전 안내 스텁으로 덮어씀**(완전 삭제는 사용자 `rm` 필요).
- **`.env.example`**: 새 경로 반영(`../bprn-organizations/localnet/...`, 경로는 prover-ts/ cwd 기준 상대).

**(C) on-bprn 부트스트랩 스크립트 (`verifier/on-bprn/scripts/`)**

- **자체완결 node 프로젝트**(사용자 선택). 배포는 bpn-core에 위임, **부트스트랩(invoke)만** 담당.
- 파일: `bootstrap.ts`(Gateway connect 자체구현, fabric-network 2.2.20), `package.json`, `tsconfig.json`, `bootstrap.localnet.example.json`(validator set 포함 예시), `README.md`(역할+usage), `.gitignore`.
- **invoke 시퀀스**(bpn-core `3_linker_init.sh` + 코드 시그니처에서 도출): registry.SetContract×4(endpoint/verifier/policy/nullifier role) → endpoint·verifier.SetRegistryID / nullifier.SetRegistry → policy.SetValidatorSet(JSON) → btip34.SetRegistryID/SetSelfCcName/SetTrustedDApp/Mint.
- **파라미터**: `--network <name>`(→`bprn-organizations/<name>/`의 profile+User1), `--config`(기본 `bootstrap.<network>.json`), `--dry-run`, `--continue-on-error`. config로 cc 이름·role·trustedDApp·mint·validatorSet 지정(기본값은 var.sh 이름: LinkerRegistry/LinkerEndpoint/LinkerVerifier/LinkerPolicy/LinkerNullifier/BTIP34CC, role 평문 `*Role`).
- 검증: 샌드박스 임시 디렉토리에서 prover-ts node_modules 빌려 **tsc --noEmit 통과(에러 0)**. 라이브 BPrN invoke는 미검증(사용자 머신 첫 실행).

### 000.2 주의·제약 (반드시 인지)

1. **라이브 BPrN 접속 테스트 미실시**: 샌드박스는 `grpcs://peer0.org1.bc` 접속 불가 + 크립토 심볼릭이 호스트 절대경로 → 실제 invoke 검증은 사용자 머신 첫 실행. 코드는 tsc 통과 + 기존 FabricClient 패턴 그대로.
2. **`--init-required` 없이 배포 필요**: fabric-network 2.2는 init(`--isInit`) 호출 불가 → 부트스트랩 스크립트는 plain submitTransaction만. init-required로 배포하면 첫 invoke가 init여야 해서 안 됨. bpn-core 배포 시 init-required 빼거나 init은 peer CLI로 별도 처리.
3. **심볼릭 stale**: 현 심볼릭 4개는 `/Users/kylekwon/...`(stale) 타깃 → 사용자 머신에서도 깨져 있을 것. 이동 후 `link-test-env.sh <network> [src]` 재실행으로 교정.
4. **마운트 권한으로 `rm` 불가**: 구 link 스크립트 스텁화로 우회. 완전 삭제는 사용자.

### 000.3 실행 런북 (사용자 머신)

```
# 1) 크립토 심볼릭 재생성
cd linker-v2/bprn-organizations && ./link-test-env.sh localnet [bprn-test-env 경로]
# 2) 체인코드 배포 — bpn-core-2.2/scripts/run (⚠️ --init-required 없이)
# 3) 부트스트랩
cd linker-v2/verifier/on-bprn/scripts
npm install
cp bootstrap.localnet.example.json bootstrap.localnet.json   # trustedDApp(=BPuN BTIP26Token 주소)/mint 채우기
npm run bootstrap -- --network localnet --dry-run            # 계획 확인
npm run bootstrap -- --network localnet                      # 제출
```

### 000.4 git 상태 / 다음 작업

- **git 미커밋**: 본 세션 변경 전부 미커밋(디렉토리 이동, connection-profile, fabric.ts, .env.example, on-bprn/scripts 신규, deploy.ts 주석, task-contexts). ⚠️ `linker-v2/.git/index.lock` 잔존(이전 git status가 남긴 것, 마운트 권한으로 자동삭제 실패) — **커밋 전 `rm -f linker-v2/.git/index.lock`**. 이 리포 git 작업은 사용자가 직접(샌드박스가 lock 정리 못 함).
- **다음 작업**: §00.4 우선순위 유지 — (1) on-bprn `go build`(사용자 확인 완료), **(2) localnet e2e**(부트스트랩→u2r/r2u→`--on-result` 교차 2PC 완결)가 본류, (3) multi-peer commit-sig 등 보류.

---

## 00. 종합 핸드오프 (2026-06-12 세션 종료)

### 00.1 상시 작업 규칙 (사용자 지시, 세션 불문 적용)

1. **모든 파일 수정은 사용자의 명시적 컨펌 후에만 시작한다.** 의견 질문("~해야 해?")은 수정 지시가 아니다. 분석/의견 제시 → "수정할까요?" → 명확한 승인 → 편집.
2. README 등 **사용 문서는 역할+usage만** — 내부 설계·신뢰 경로·타 모듈 대응 관계 서술 금지. 문장은 짧고 군더더기 없이("~만", "자동", 노트식 표기 금지). 예외: 모르면 실사용이 깨지는 주의사항(int64 정밀도 등).
3. BTIP 스펙·설계 문서는 사용자가 직접 관리 — 모순 발견 시 보고와 해소안 제시까지만.

### 00.2 현재 시스템 전체 그림 (구현 완료 상태)

**컴포넌트** (모두 구현됨, localnet e2e만 미실시):

| 위치 | 컴포넌트 | 상태 |
|---|---|---|
| BPuN (on-bpun) | LinkerRegistry·Endpoint·Nullifier·Verifier·Policy(+PolicyVerifier)·**BTIP26Token**(NFT 브리지 dApp 샘플) | solc 전체 컴파일 OK |
| BPrN (on-bprn) | linker-registry·endpoint·nullifier·verifier·policy·**btip34-ccapp**(STC 모형 ccApp, 구 stc-example)·dapp-example(최소 예제) | ⏳ `go build ./...` 로컬 미검증 (샌드박스 Go 없음) |
| prover-ts | **btip39-u2r-policy**(validator set sync 데몬), **btip28-u2r-txevt**(BPuN→BPrN tx/event), **btip19-r2u-txevt**(BPrN→BPuN tx/event), **common/** | tsc 0 에러 + 단위 자가시험 통과. btip39는 localnet e2e도 통과(06-11) |

**prover 명명 규칙**: 디렉토리 = `btipXX-{r2u|u2r}-{policy|txevt}`, XX = 생성하는 증명 페이로드의 BTIP 번호 (btip19=BPrNTxEventProof→btip-21 onProof/onResult, btip28=BPuNTxEventProof→btip-29 OnProof/OnResult).

**prover 실행** (모두 `prover-ts/`에서, env는 `.env`):

```
npm run btip39:sync                 # validator set 동기화 데몬
npm run btip28:cli -- <txHash> [--to-ccapp <cc명>] [--on-result] [--event-index N] [--attrs ...] [--out f]
npm run btip28:api                  # :3028, GET /proof(생성) POST /proof(생성+제출, to_ccapp)
npm run btip19:cli -- <txId>   [--to-dapp <0xaddr>] [--on-result] [--channel ch] [--attrs gidx...] [--out f]
npm run btip19:api                  # :3019, GET /proof(생성) POST /proof(생성+제출, to_dapp)
```

2PC 결과증명 방향 주의: BPrN LinkerResultElems→BPuN은 **btip19 --on-result**, BPuN LinkerResult→BPrN은 **btip28 --on-result**.

**src/ 구조** (2026-06-12 재편 — "자체 완결" 원칙은 사용자 지시로 폐기, 공용화):

```
src/common/   btip16-merkle.ts(BTIP-16 트리) tm-merkle.ts(Tendermint RFC6962+cdcEncode, idx8/idx11 빌더)
              event-log.ts(EventLog+DER) rpc.ts(Tendermint RPC 전체) fabric.ts(FabricClient: Gateway·
              resolveContractByRole·qscc 블록조회·EndorserTx 파싱·커밋서명) env.ts go-json.ts
src/btip{19,28,39}-*/   cli.ts/server.ts(btip39는 main.ts)/config.ts/submitter.ts/tx-event-proof.ts/types.ts/README.md
잔재: src/app.module.ts·src/main.ts (구 NestJS 골격, app.module은 깨진 import — 정리 미지시)
```

**crypto 자료**: `prover-ts/bprn-organizations/localnet/`(네트워크별 디렉토리) — bpn-core-2.2/bprn-test-env로의 파일 단위 심볼릭 링크 + connection-profile.json. 재생성: `scripts/link-bprn-test-env.sh`. `.env`의 FABRIC_* 경로도 localnet/ 기준.

### 00.3 오늘 확정된 설계 결정

1. **BTIP-25 CorrelationId 필드 제거 확정** (해시 고정점 불가 발견 → 사용자 컨펌): TransferLogElems = 9-leaf(From=4, To=5, Amount=6, Beneficiary=7, Memo=8), selector = `sha256("TransferLogElems([]byte,[]byte,[]byte,[]byte,[]byte)")` (5-인자 — 구 6-인자 해시 무효). BPrN-origin correlationId = 요청 이벤트의 **tx_event_root 내재값**(필드 없음). D안(명시 id) 배제 근거 = dApp revert 시 correlationIndex 유실로 REJECTED 결과 발행 불가(D-B 충돌). btip-25/40·2pc-design·BTIP26Token.sol·btip34-ccapp 모두 반영됨.
2. **tendermint-ethaddr = 바닐라 v0.34 + 주소 유도(keccak)만 변경** (사용자 확인) → LastResultsHash 의미(블록 H 결과→헤더 H+1), deterministic DeliverTx(Code/Data/GasWanted/GasUsed, Events 미포함) 모두 바닐라 기준으로 확정 추론 가능.
3. **BPrN 이벤트 와이어 관례** (bpn-core 실물 확인): `SetEvent(hex(selector), DER(elems))`, 피어가 Header 재구성+root 계산. bprn-sdk-go `chaincodes/event` v0.10.2가 정본(on-bprn에 vendor됨). protobuf emit은 증명 불가 — linker-endpoint 수정 완료.

### 00.4 다음 작업 (우선순위)

1. **on-bprn `go build ./...` 로컬 검증** (vendor 정합·btip34-ccapp 포함) — 유일한 빌드 미검증.
2. **localnet e2e** (§"2026-06-12 — btip28" e2e 절차 참조): ① 체인코드 배포+부트스트랩(btip34-ccapp: SetRegistryID/SetSelfCcName/SetTrustedDApp/Mint; BPuN: deploy.ts `<chain> bpn <STC cc명>`로 setPaymentSource) ② u2r: burn-nft.ts → btip28:cli --to-ccapp ③ r2u: btip34-ccapp.PayToBPuN → btip19:cli --to-dapp ④ 양방향 2PC 완결(--on-result 교차).
3. e2e 관전 포인트: BTIP-35 attr 실데이터 형식(topic 대문자/0x 유무 — endpoint attrHex32At·btip34-ccapp hexEqual 가정), IsBTIP27 활성 높이, btip19 ABI·@beatoz/web3 제출 경로 첫 실행.
4. 보류: multi-peer commit-sig, 3_linker_update_vs.sh stale. (~~IBTIP22 메소드명·LINKER_POLICY 바인딩 재설계~~ — **2026-06-14 코드 확인 결과 해소**: `LinkerPolicy`가 `is IBTIP22`로 데이터+검증 단일 컨트랙트가 되어 role이 LinkerPolicy를 직접 가리키고 메소드명도 스펙명과 일치. §0.4 정정 참조.)

### 00.5 git 상태 주의 (이 세션이 남긴 워킹 트리)

`git mv`(리네임=staged)와 sed(내용 수정=unstaged)가 섞여 같은 파일이 R+M으로 양쪽에 보임. **커밋 전 `git add -A`로 전부 스테이징해서 한 커밋으로 묶을 것** — 지금 그대로 commit하면 staged 절반만 들어감. 마지막 커밋: 4d19cff(btip28). 미커밋: btip19 모듈, 디렉토리 재편(공용화·리네임), bprn-organizations/localnet, on-bprn(endpoint 수정·btip34-ccapp·vendor·9-leaf), on-bpun(BTIP26Token gidx·deploy.ts·burn-nft.ts), BTIPs(btip-25/40), task-contexts.

---

---

## 2026-06-12 — prover-ts 디렉토리 재편 (사용자 지시)

- **bprn-organizations 재배치 (후속 사용자 지시)**: `prover-ts/bprn-organizations/*` 전부를 **`bprn-organizations/localnet/`** 하위로 이동(connection-profile.json + peer/orderer crypto 심볼릭 링크). 경로 참조 일괄 수정: `.env`(FABRIC_CONNECTION_PROFILE/CERT_PATH/KEY_PATH), `.env.example`, `scripts/link-bprn-test-env.sh`(DST), connection-profile.json 내부 tlsCACerts 절대경로. 심볼릭 링크 대상은 bpn-core-2.2 절대경로라 이동 무영향(링크 생존 확인). 네트워크별 디렉토리(localnet/...) 구조의 시작.


- **삭제**: `src/fabric/`, `src/prover/`(잔재) — 구 NestJS r2u prover 골격. 어느 btip 모듈도 참조하지 않음을 확인 후 제거. package.json `btip39:scan`(삭제된 src/prover/btip39 대상) 스크립트도 제거.
- **리네임** (`btipxx-{r2u|u2r}-{policy|txevt}` 형식): `src/btip19` → **`btip19-r2u-txevt`**, `src/btip28` → **`btip28-u2r-txevt`**, `src/btip39` → **`btip39-u2r-policy`**. npm 스크립트 경로 갱신(스크립트 이름은 btip19:cli 등 유지), 내부 주석·Usage 문자열의 구 경로 갱신. 3개 모듈 일괄 tsc 0 에러.
- **공용 모듈화 (후속 사용자 지시)**: "자체 완결(모듈 간 의존 0)" 원칙을 **사용자 지시로 폐기**하고 공통 로직을 `src/common/`으로 추출. 구성:
  - `common/btip16-merkle.ts`(구 merkle.ts — 2026-06-12 사용자 지시 리네임)·`event-log.ts`, `common/rpc.ts`(btip39+btip28 병합 — status/block/commit/validators/tx/blockResults), `common/tm-merkle.ts`(구 header-merkle.ts — Tendermint 규격: RFC6962 트리+cdcEncode, NextValidatorsHash(8)·LastResultsHash(11) 빌더 모두), `common/fabric.ts`(FabricClient: Gateway 접속·getContract·**resolveContractByRole**(registry 단일 출처)·qscc 블록 조회·EndorserTx 파싱·커밋서명 추출 + blockNumberOf), `common/env.ts`(required·loadFabricEnv·loadBpunRpcUrl), `common/go-json.ts`(goJsonStringify).
  - 모듈별 변화: btip39(rpc/header-merkle 삭제, submitter·config를 FabricClient/공용 env 기반으로 재작성), btip28(rpc/header-merkle/btip16-merkle 삭제 — 트리는 common/merkle.ts 사용으로 재작성, submitter·config 동일 처리), btip19(merkle/event-log/fabric 삭제 — 전부 common 사용). 각 모듈 config는 `FabricConnectionConfig`를 extends.
  - 검증: common+3모듈 일괄 tsc 0 에러, 기능 자가시험 재통과(2단계 이벤트 트리·proto 벡터·EventLog 증명 — 공용 모듈 경유).
- ⚠️ 잔재: `src/app.module.ts`(삭제된 ./prover/prover.module import — 기존부터 깨져 있던 NestJS 골격)·`src/main.ts` — 지시 범위 밖이라 미정리. 전체 tsc는 app.module.ts 1건 에러(모듈·common tsc는 전부 0).

---

## 2026-06-12 — btip19 r2u tx/event prover (`prover-ts/src/btip19/`)

> 사용자 명시 지시: "btip19 구현해. btip28과 동일한 기능과 범위로." — r2u 페이로드는 BTIP-19 `BPrNTxEventProof`(사용자 확인), 제출 인터페이스는 BTIP-21(LinkerEndpoint/BPuN `onProof`/`onResult`). `src/btip19/`는 타 개발자 리팩토링이 비워둔 자리(구 src/prover 삭제됨)에 신규 구현.

### 구성 (btip28과 동일 패턴, 자체 완결)

- **`btip19:cli`** (`cli.ts`): `<txId> [--channel] [--attrs i,j,k(gidx)] [--to-dapp 0x...] [--on-result] [--out]` — 항상 stdout에 증명 출력, `--to-dapp` 시 BPuN LinkerEndpoint로 제출(onProof/onResult). 제출 메시지는 stderr.
- **`btip19:api`** (`server.ts`, 기본 포트 3019): GET /proof(생성) / POST /proof(생성+제출, `to_dapp` 필수, `on_result?`) / GET /health. Fabric gateway는 lazy 연결 후 재사용.
- **빌더** (`tx-event-proof.ts`): 구 prover.service.ts 로직 이식 — qscc GetBlockByTxID → 블록 1회 순회로 BTIP-17 leaves(실패/무이벤트 tx = null leaf) + 대상 EventLog(DER elems, hex tx_id/event_name 디코딩) → block_event_root·증명 생성. **모든 증명을 로컬 verifyMerkleProof로 자가검증 후 발행**(구 코드엔 없던 추가). cert PEM→DER, commit sig DER→r||s 64B(메타데이터 index env, 기본 5). null sibling = **ZERO_HASH 센티넬**(Solidity 검증자 와이어 — btip28의 JSON null과 다름, 방향별 검증자 관례 차이).
- **제출기** (`submitter.ts`): **@beatoz/web3 의존 추가**(prover-ts package.json). LinkerEndpoint 주소는 BPUN_LINKER_REGISTRY에서 `getContract(chainId, keccak256("LinkerEndpoint"))`로 해석(레지스트리 단일 출처; `.call()` raw returnData에서 주소 추출 — deploy.ts와 동일 패턴). **ABI 프래그먼트를 artifacts의 LinkerEndpoint.json과 대조 검증 완료**(필드명·타입·순서 일치).
- **자체 완결 복사**: merkle.ts·event-log.ts(src/common에서), fabric.ts(구 fabric.service.ts의 NestJS 제거 포트 — qscc 조회·EndorserTx 파싱·커밋서명 추출).
- env: 생성 = FABRIC_*(6종)+FABRIC_COMMIT_SIG_METADATA_INDEX(5) / 제출 = BPUN_RPC_URL·BPUN_CHAIN_ID·BPUN_PRIVATE_KEY·BPUN_LINKER_REGISTRY·BPUN_SUBMIT_GAS(5M).
- 검증: btip19 단독 tsc 0 에러. 단위 자가시험 통과 — DER 인코더/디코더 왕복, 9-leaf TransferLogElems 형상(leavesLen=9), 전체 gidx 증명 검증, BTIP-17 null leaf 블록 트리 증명. README는 역할+usage 형식.
- e2e 미실시(localnet 필요): r2u 본류(btip34-ccapp.PayToBPuN → btip19:cli --to-dapp BTIP26Token) + 2PC 완결(LinkerResultElems → --on-result는 btip28이 담당... 주의: **r2u 결과증명(BPuN LinkerResult)을 BPrN으로 보내는 건 btip28 --on-result**, **u2r 결과증명(BPrN LinkerResultElems)을 BPuN으로 보내는 건 btip19 --on-result**).

---

## 2026-06-12 — btip28 u2r tx/event prover (`prover-ts/src/btip28/`) + btip34-ccapp 리네임

> 사용자 명시 지시 2건: ① `stc-example` → **`btip34-ccapp`** 디렉토리·내부 명칭 일괄 변경(완료). ② **btip28 prover를 `prover-ts/src/btip28/`에 자체 완결로 구현**(btip39와 동일 원칙 — 타 모듈 의존성 0, rpc/header-merkle은 btip39에서 복사 격리).
>
> 용어 정리(사용자 확인): btip39 데몬 = **u2r policy prover**(validator set 동기화 채널), 본 모듈 = **u2r tx/event prover** — 증명 페이로드는 BTIP-28(`BPuNTxEventProof`), 제출 인터페이스는 BTIP-29(OnProof/OnResult).

### 신뢰 경로와 구현 (BTIP-28 검증 파이프라인 = linker-verifier가 정본)

- 경로: commit sigs(GetValidatorSet(height)로 대조) → block_hash → **헤더 트리 idx 11** cdcEncode(LastResultsHash) → ABCIResults 트리(deterministic DeliverTx) → **tx_event_root = DeliverTx.Data**(beatoz-go node/app.go가 `txctx.EventRoot()` 결과를 Data에 기록 — 확인함) → Tx Event Tree(preHashed) → Per-Event Tree(sha256(attr.Value)).
- **anchor 높이 적응 해석**: Tendermint v0.34는 블록 H의 결과를 **헤더 H+1**의 LastResultsHash에 커밋 → prover는 deterministic results root를 직접 재구성해 헤더 H+1·H 양쪽과 대조, 일치하는 쪽을 `payload.height`(anchor)로 채택(H+1 미생성 시 1초 간격 재시도). 포크가 의미를 바꿨어도 자동 적응; 어느 쪽도 불일치면 "deterministic 인코딩 발산" 에러로 정지.
- **deterministic DeliverTx 인코딩**: protobuf {Code=1, Data=2, GasWanted=5, GasUsed=6}(영값 생략, 비결정 필드 제외 — btip-28 §tx_result_proof). 알려진 벡터로 단위검증 통과. ~~⚠️ 포크가 results 해싱에 Events를 포함했을 가능성~~ → **해소(2026-06-12 사용자 확인): tendermint-ethaddr는 주소 유도 공식만 변경, 그 외 바닐라 v0.34와 동일.** 따라서 LastResultsHash 의미(블록 H 결과 → 헤더 H+1)와 deterministic 인코딩(Events 미포함)은 바닐라 그대로이고, anchor는 항상 H+1로 귀결(양쪽 대조 로직은 방어적 자가검증으로 유지).
- **BTIP-27 이벤트 트리**: beatoz-go `eventRootBTIP27`(per-event: sha256(attr.Value) leaves / tx-level: preHashed roots, beatoz-go types/merkle == bprn-sdk-go merkle 동일 구현 diff 확인) 미러 `btip16-merkle.ts` — 1-indexed 배열 트리, null 패딩, hashPair(nil,nil)=nil. **siblings의 null은 JSON null로 전송**(Go HexBytes null→nil, VerifyMerkleProof hashPair(current,nil) 처리).
- **3중 자가 검증**(불일치 시 페이로드 미발행): ① results root == anchor 헤더 LastResultsHash ② 헤더 14-leaf 재구성 root == block_id.hash ③ 재계산 tx_event_root == DeliverTx.Data.
- **단위 자가시험 통과**: BTIP-16 증명을 Go 검증자 워크 미러로 검증(n=1~13, raw/preHashed, null sibling 포함), 2단계 합성, proto 벡터, RFC6962(n=1~12). `tsc --noEmit` 전체 통과.

### 파일 (`prover-ts/src/btip28/` — 자체 완결)

- `main.ts`: CLI — `ts-node src/btip28/main.ts <txHash> <handlerCcId> [--on-result] [--event-index N] [--attrs i,j,k] [--dry-run] [--out path]`. 이벤트 기본 선택 = 유일한 "evm" 이벤트(복수면 목록 출력 후 인덱스 요구), attrs 기본 = 전체. npm script: **`btip28:prove`**.
- `server.ts` (2026-06-12 사용자 지시 추가): **증명 생성 HTTP API** — Node 내장 http(의존성 0), npm script **`btip28:api`**, 포트 `BTIP28_API_PORT`(기본 3028), Fabric 설정 불필요(생성 전용). `POST /proof` {tx_hash, event_index?, attr_indices?} → {tx_height/tx_index/event_index/event_type/attr_indices/tx_event_root/event_attrs_root, **payload**(=OnProof/OnResult용 payloadJSON)}. 응답 전체를 goJsonStringify로 직렬화 — **timestamp(int64) bare 리터럴이라 일반 JSON.parse→stringify 재인코딩 금지**(README CAUTION 명시). 형식 오류=400 / 노드·자가검증 실패=502. `GET /health`.
- `README.md` (사용자 지시 추가): 역할+usage만 — 2회 질책 후 축소(신뢰 경로·자가검증·btip39 대응·파일 표·콜백 메소드명 전부 제거, --on-result 판단 기준은 옵션 행 한 줄). int64 CAUTION만 유지(모르면 실사용이 깨지므로).
- **진입점 재편(2026-06-12 사용자 지시, 최종)**: `main.ts`(btip28:prove) **삭제** — 진입점은 2개로 정리.
  - **`btip28:cli`** (`cli.ts`): `<txHash> [--to-ccapp <name>] [--on-result] [--event-index N] [--attrs i,j,k] [--out path]` — 항상 stdout에 증명 출력(GET 응답과 동일 형식, goJsonStringify), **`--to-ccapp` 지정 시 출력 + 해당 ccApp으로 자동 제출**(OnProof, `--on-result`면 OnResult; 제출 메시지는 stderr). 미지정 시 BPUN_RPC_URL만 필요.
  - **`btip28:api`** (`server.ts`): **GET /proof**(query: tx_hash, event_index?, attr_indices=csv) = 생성만 / **POST /proof**(body: tx_hash, **to_ccapp 필수**, on_result?, event_index?, attr_indices?) = 생성 + 자동 제출(응답에 `submitted:{method, to_ccapp, endpoint}` 추가). Fabric 설정은 제출 경로에서만 요구(POST 시 lazy load).
- ⚠️ **타 개발자 리팩토링 감지(2026-06-12)**: `src/prover/*` 전체 삭제(git D) + `src/btip19/` 신설 — r2u prover가 btip19로 이동 중인 듯. `app.module.ts`가 삭제된 `./prover/prover.module`을 참조해 **전체 tsc는 그쪽 에러 1건**(btip28 단독 tsc는 0 에러). package.json `btip39:scan` 스크립트도 dangling. 불가침 영역이라 미수정 — ryan 영역 정리 후 해소될 것.
- `tx-event-proof.ts`: 빌더(`buildTxEventProof`) — anchor 해석·deterministic 인코딩·BTIP-27 트리·자가검증·페이로드 조립.
- `btip16-merkle.ts`: BTIP-16 배열 트리(증명 생성 포함). `header-merkle.ts`: btip39 복사 + **idx 11 `buildLastResultsHashProof`**로 개작. `rpc.ts`: btip39 복사 + `/tx`·`/block_results` 추가(base64 디코딩 포함). `types.ts`: Go BPuNTxEventProof 미러(hex 와이어, timestamp bigint, goJsonStringify). `submitter.ts`: registry `GetContract(channel, "LinkerEndpointRole")` 해석 → OnProof/OnResult 제출. `config.ts`: btip39와 동일 env 키(데몬 옵션 제외).

### e2e 절차 (다음 세션)

1. (선행) on-bprn `go build` + 체인코드 배포: linker-endpoint(이번 DER emit 수정 반영)·btip34-ccapp 포함, registry 등록 + btip34-ccapp Setup(SetRegistryID/SetSelfCcName/SetTrustedDApp/Mint). BPuN deploy.ts 재배포(+setPaymentSource).
2. u2r 본류: `burn-nft.ts`로 TransferLogAttrs 발생 → `npm run btip28:prove -- <txHash> <btip34-ccapp 배포명>` → STC 잔액(beneficiary) 확인 → endpoint가 emit한 LinkerResultElems(DER) 확인.
3. 2PC 완결(r2u 결과): LinkerResultElems를 r2u prover(BPrNTxEventProof)로 증명해 BPuN `onResult` 제출 → BTIP26Token pending 확정(ACCEPTED=소각 확정). ※ r2u prover(src/prover, 타 개발자 영역)가 9-leaf 신형식과 무관하게 동작하는지 확인.
4. 역방향(r2u 본류): btip34-ccapp.PayToBPuN → r2u prover → BPuN onProof → NFT 민트 → LinkerResult → **btip28 `--on-result`**로 BPrN OnResult 제출 → btip34-ccapp 에스크로 settle/refund.

---

## 2026-06-11 — ccApp/BPrN STC 모형 (`verifier/on-bprn/btip34-ccapp/`, 구 stc-example) + BPrN 이벤트 와이어 관례 정합

> 사용자 지시: "setPaymentSource 추가하고, ccApp/BPrN은 TransferLogElems 발신 + TransferLogAttrs 처리까지 구현". 구현 전 조사에서 **BPrN 이벤트 와이어 관례를 bpn-core 실물에서 확정**했고, 그 결과 linker-endpoint의 기존 emit이 증명 불가능한 형식이었음을 발견·수정함.

### 와이어 관례 확정 (bpn-core 실물 — `core/committer/txvalidator/v20/validator_ex.go`)

- **체인코드 이벤트 발신 규칙**: `stub.SetEvent(hex(selector), DER(elems))`.
  - Fabric event name = **selector의 hex 문자열** (피어가 `hex.DecodeString(ccEvent.EventName)`으로 복원, 실패 시 리터럴 폴백).
  - payload = **elems만 ASN.1 DER** (`event.EventLog.MarshalDER(true)`). Header(channel/chaincode/tx_id/selector)는 **피어가 tx 컨텍스트에서 재구성**.
  - 피어는 `github.com/beatoz/bprn-sdk-go/chaincodes/event`(v0.10.2)로 EventLog를 조립하고 `Root()`로 tx_event_root 산출 → btip-17 블록 트리·커밋 서명.
  - BTIP-16의 protobuf 스키마는 **논리 구조 기술**이고 실제 SetEvent payload 형식이 아님. (linker-v2 `types/eventlog.go`의 protobuf Marshal은 이 오해의 산물이었음 — **삭제**.)
- **발견된 버그(수정 완료)**: linker-endpoint `emitLinkerResult`가 `SetEvent("LinkerResultElems", protobuf(EventLog 전체))`로 emit → 피어 `UnmarshalDER` 실패 → `evtLogRoot=nil` → **LinkerResultElems가 블록 트리에 안 들어가 영원히 증명 불가**(u2r 2PC 결과가 BPuN으로 못 돌아감). → SDK 관례(`hex(selector)` + `MarshalDER(true)`)로 수정.
- **추가 수정**: OnProof의 2PC correlation 추출 — BTIP-35 leaf는 hex 텍스트(topic.1 = 64자 대문자)인데 raw로 LinkerResultElems gidx 4에 싣고 있었음 → BPuN onResult가 `corr.length != 32`로 revert했을 것. `attrHex32At`로 **디코딩 후 32B raw** 탑재로 수정.
- **bprn-sdk-go v0.10.2 의존 추가** (재사용 원칙): `go.mod` require + `vendor/github.com/beatoz/bprn-sdk-go/chaincodes/event{,/merkle,/types}` 수동 vendor + modules.txt 등재. go.sum 미보충(vendor 모드 빌드는 무관, `go mod tidy` 시 자동). 로컬 bprn-sdk-go 체크아웃 == tag v0.10.2 정확히 일치 확인.

### `btip34-ccapp` — ccApp/BPrN 레퍼런스 (IBTIP34 완전 구현 + 토큰 모형)

- **r2u 1단계 — `PayToBPuN(toHex, amountDec, beneficiaryHex, memoHex) → correlationId(hex)`**:
  - 잔액 차감(에스크로 잠금) → **신형 TransferLogElems(gidx 4~8)** emit → **tx_event_root를 SDK 동일 코드로 emit 시점에 직접 계산**해 `PENDING_<root>`에 {from, handler_dapp(=to), amount} 기록 → root hex 반환(= r2u prover 입력).
  - `To`(gidx 5) = 대상 dApp/BPuN(BTIP26Token) 주소 — dApp의 `To==address(this)` 검증과 잠금 시점 intended-handler 기록을 겸함.
  - **클라이언트 직접 호출 전용** (c2c로 부르면 피어가 outer cc를 Header에 박아 root가 어긋남 — 주석 명시).
- **BTIP-25 CorrelationId 고정점 모순 → 필드 제거(A안)로 ✅ 확정 (2026-06-12 사용자 컨펌)**: gidx 4는 트리의 리프인데 값이 그 트리의 root여야 함 → 해시 고정점이라 성립 불가. 프로토콜은 gidx 4를 읽지 않음(btip-21 onProof는 증명된 root 직접 사용). 해소안 A(필드 삭제)/B(빈 값 유지+문구 수정)/D(명시 id 회귀) 논의 — D안은 IBTIP26 시그니처 변경(correlationIndex 반환)·prover 페이로드 요건 추가·**revert 시 correlationId 유실(D-B 충돌, REJECTED 결과 발행 불가)** 연쇄가 확인되어 배제. 내재값 방식의 숨은 장점 = dApp이 revert해도 endpoint가 협조 없이 항상 REJECTED 발행 가능("always emit" 결정 8과 정합).
  - 적용된 내용: **CorrelationId 필드 삭제, 9-leaf 재배치**(From=4, To=5, Amount=6, Beneficiary=7, Memo=8), **selector 변경** `sha256("TransferLogElems([]byte,[]byte,[]byte,[]byte,[]byte)")` (5-인자, 구 6-인자 해시 무효). 매칭 식별자 = 요청 이벤트의 `tx_event_root` 내재값(필드 없음).
  - 반영 파일: btip-25(struct·매칭 식별자 NOTE 신설·selector·트리 9-leaf), btip-40(Abstract·Symmetry — 자산 전송 필드만 동일, 매칭 식별자는 방향별 상이 명시), btips-2pc-design(§5·D-A 2026-06-11 재정정 — D-A 최초 형태 확정), BTIP26Token.sol(GIDX_TO=5/AMOUNT=6/BENEFICIARY=7 + SIG, solc 컴파일 OK), btip34-ccapp(elems 5개·SIG), btips.md 상태 테이블(btip-25/40 행).
- **u2r — `HandleLinkerEvent` = BTIP-40 TransferLogAttrs 처리** (BTIP26Token.handleLinkerEvent의 거울상):
  - 검증 3종: ① topic.0(gidx 1) == keccak256(TransferLogAttrs sig) ② contractAddress(gidx 0) == **admin이 `SetTrustedDApp`으로 고정한 dApp** (setPaymentSource의 BPrN 대칭 — 공짜 STC 민트 차단) ③ to(topic.3, gidx 4) == **자기 BTIP-9 주소**(sha256(channel+"-"+ccName)[12:]).
  - 형식·출처 위반 = **error(하드 실패, nullifier 롤백 → 올바른 핸들러로 재제출 가능)** / amount==0 = **Status=false 정상 반환**(비즈니스 거부 → REJECTED 결과 → dApp이 소각 복원).
  - data(gidx 5) ABI 수동 디코딩 (amount, beneficiary; memo 미사용) → **beneficiary에게 지급**(모형은 민트 — 프로덕션은 settled 풀에서 인출, 주석 명시).
  - 2PC: `CorrelationIndex=2`(topic.1의 gidx) 반환 — endpoint가 그 leaf를 hex 디코딩해 LinkerResultElems에 탑재(위 수정).
- **r2u 2단계 — `HandleLinkerResult(correlationId, handlerDApp, status)`**: PENDING 매칭(미지 → error 롤백), handlerDApp == 잠금 시점 기록과 대조(불일치 → error), ACCEPTED=에스크로 확정 소멸 / REJECTED=from 환불.
- **계정 모형**: 주소 = sha256(creator)[12:] 20B hex (`MyAddress` 조회 제공), 잔액 = 10진수 문자열. Mint(admin)/Transfer/BalanceOf/GetPending.
- **부트스트랩 setter**: `SetRegistryID`, `SetSelfCcName`(피어가 Header에 박을 자기 cc명 — root 계산·BTIP-9 자기 주소에 필요, Fabric은 c2c 수신 시 자기 이름을 알 수 없음), `SetTrustedDApp(BTIP26Token addr hex)`.
- `CancelLinkerEvent`: nullifier 해제만 (모형 한계 — 프로덕션은 지급 기록 역산 필요, 주석).
- dapp-example은 BTIP-34 최소 인터페이스 예제로 그대로 유지(불변경).

### 부트스트랩/스크립트

- **`deploy.ts <chainAlias> [paymentChannel] [paymentChaincode]`**: 인자 제공 시 `btip26Token.setPaymentSource(channel, ccName)` 호출, 미제공 시 SKIP 경고(미설정 = 전부 거부가 안전 기본값).
- **`burn-nft.ts`**: `to` 인자가 `channel/chaincode` 형식이면 **BTIP-9 주소로 변환**(기본값 `bpn/STC`), 0x 주소도 그대로 허용.
- **양방향 부트스트랩 절차(신규 환경)**: ① BPrN에 btip34-ccapp 배포(예: 이름 "STC") → SetRegistryID/SetSelfCcName("STC")/SetTrustedDApp(BTIP26Token addr) + admin Mint ② BPuN deploy.ts에 `bpn STC` 인자로 setPaymentSource ③ r2u: STC.PayToBPuN(to=BTIP26Token addr, beneficiary, amount) → 반환된 correlationId(=tx_event_root)+txId로 prover → OnProof... ④ u2r: burn-nft.ts → 신규 u2r prover(다음 작업).

### 세션 논의 기록 (코드 외)

- **BTIP-35 leaf의 이중 인코딩 구조 정리** (사용자 질의로 명확화): `EventAttrProofs.Leaf`(HexBytes)는 *운반* 인코딩일 뿐이고, 머클 leaf의 **원본 데이터 자체**가 BPuN 노드(`evmLogsToEvent`, beatoz-go ctrler.go)가 만든 hex 텍스트다 — topic.N = 64자 대문자 hex ASCII(topic은 `Index:true`), contractAddress = 소문자 hex, data = 소문자 hex, blockNumber = 10진수 문자열, removed = "false". 증명이 통과하려면 Leaf는 이 텍스트 바이트 그대로여야 하고(`sha256(Value)`가 리프), **의미값(raw bytes)이 필요한 소비 지점에서 정확히 1회 hex 디코딩** — endpoint의 `attrHex32At`가 그 지점. topic 값이 hex 텍스트인 이유는 Tendermint 이벤트 인덱싱·쿼리(`tx_search`/`subscribe`)가 문자열 기반이기 때문. **현행 인코딩 유지로 정리**(2026-06-11).
- **수정 작업 규칙 (2026-06-11 사용자 지시, 상시 적용)**: 의견 질문("~해야 해?")은 수정 지시가 아님. **모든 파일 수정은 사용자의 명시적 컨펌을 받은 뒤에만 시작** — 분석/의견 제시 → "수정할까요?" 질문 → 명확한 승인 → 편집 순서 엄수.

### 검증 상태 / 다음

- ⏳ `go build ./...` **로컬 미검증** (샌드박스 Go 없음) — on-bprn 전체 빌드 필요(특히 vendor 정합·btip34-ccapp). deploy.ts/burn-nft.ts도 재실행 검증 필요. Solidity는 solc 전체 컴파일 OK(gidx 재배치 반영 후 재확인).
- ✅ CorrelationId 필드 제거(A안) — 2026-06-12 사용자 컨펌으로 확정.
- 다음: **u2r prover 본체**(BPuNTxEventProof 구성 → OnProof/OnResult 제출) — btip39의 header-merkle 재사용, BTIP-27 2단계 이벤트 트리.

---

## 2026-06-11 — u2r Prover 2단계 준비: BTIP26Token (NFT 브리지 샘플, 사용자 재설계)

> 1차 구현(BTIP26Dapp + 임의 emit `transferToBPrN`)을 사용자 redirect로 **의미 있는 use case로 재설계**: BPrN 결제 → NFT 민트(r2u), NFT 소각 → BPrN 환급 요청(u2r). bpun-origin-payment-design §12 예시 시나리오의 실물 샘플.

- **`BTIP26Dapp.sol` → `BTIP26Token.sol`** (파일·컨트랙트명, ERC721 `"BTIP26Token"/"B26T"`). 전 스크립트·gas test 참조 일괄 rename.
- **r2u (BPrN-origin) — `handleLinkerEvent` = TransferLogElems 해석 → NFT 민트**:
  - **(사용자 지적으로 보강) 이벤트 타입·출처 검증 선행**: ① selector(gidx 3) == `sha256("TransferLogElems([]byte,...)")` 아니면 `ErrUnexpectedEvent` revert — 타 이벤트 타입 오인 차단. ② 발행자(gidx 0 channel_id + gidx 1 chaincode_id)가 **owner가 지정한 결제 체인코드(`setPaymentSource(channel, chaincode)`)와 일치**해야 — TransferLogElems 형식을 흉내 낸 임의 체인코드의 "공짜 민트" 차단(화폐 진본성; ERC-20 결제의 토큰 주소 고정과 동일 — §15.2 "신뢰 emitter 목록 금지"는 ccApp→dApp 방향 얘기라 무충돌). 미설정 시 `ErrPaymentSourceNotSet`으로 전부 거부(안전 기본값). **부트스트랩에 `btip26Token.setPaymentSource("bpn", "<STC cc명>")` 단계 추가 필요.**
  - BTIP-25 gidx에서 추출: `To`(6)·`Amount`(7)·`Beneficiary`(8). **`To != address(this)`면 `ErrWrongRecipient` revert(=REJECTED → BPrN 환불 경로)** — 수취인이 이 컨트랙트인 결제만 유효.
  - **`Amount / NFT_PRICE`(1e18)개 민트, 수령자는 `Beneficiary`** (나머지 버림, 0개면 `ErrAmountTooSmall` revert). `From`(5)은 증명에 없어도 동작 — **Stealth Address 은닉 모델이 실제로 작동하는 지점**.
  - 요소 디코딩: address=20B raw, amount=big-endian ≤32B. 누락 gidx는 `ErrMissingEventElem` revert. `LinkerProofReceived`(관찰용)·`Minted` 이벤트.
- **u2r (BPuN-origin) — `burnToBPrN(tokenIds[], to, beneficiary, memo)`**:
  - msg.sender 소유 검증 후 일괄 `_burn` → `amount = count × NFT_PRICE`로 **BTIP-40 `TransferLogAttrs`** emit (correlationId = 전역 유일 권장식) + `_pending[correlationId]`에 **소각 tokenIds 보관**.
  - `handleLinkerResult`: pending 매칭(미지 correlationId → `ErrUnknownCorrelationId` revert — nullifier 롤백 경로), **ACCEPTED=소각 확정, REJECTED=동일 tokenIds를 원소유자에게 re-mint(복원)** → `BurnFinalized`. 2PC 보류-완결 모델의 완전한 샘플.
- **`scripts/beatoz/burn-nft.ts`** (emit-transfer.ts 대체) — `burnToBPrN` 호출 + TransferLogAttrs 디코드 + u2r prover 입력값(height/txHash/contractAddress/topic.0 = `0xc4959202…f2278`) 출력.
- `utils.ts` 셀렉터 추가: ErrUnknownCorrelationId(3efa4d9d)·ErrMissingEventElem(3ab032a4)·ErrMalformedEventElem(6701fd82)·ErrWrongRecipient(21706e47)·ErrAmountTooSmall(8e96b283)·ErrNoTokens(0d11fe7f)·ErrNotTokenOwner(be2c5ef6). solc 전체(10파일) 컴파일 OK.
- **재배포 필요**: `deploy.ts` 재실행. **테스트 순서 의존성**: burn하려면 NFT가 먼저 있어야 → r2u 민트가 선행 — BPrN 측 발신 체인코드가 **신 TransferLogElems 형식으로 emit해야** prover API+submit-proof.ts 경로로 민트 가능 (구 4월 fixture는 구 형식). BPrN 발신 체인코드(STC 모형) 정합이 다음 선결 과제. *(주: 같은 날 후반 세션에서 CorrelationId 필드가 제거되어 최종 형식은 gidx 4~8 — 위 최신 절 참조.)*
- 다음: u2r prover 본체 — `BPuNTxEventProof` 구성 → `OnProof(payload, handlerCcId)`. OnResult 테스트 시 BPrN registry에 BPuN LinkerEndpoint 등록(btip-37 NOTE) 필요.

---

## 2026-06-11 — u2r Prover 1단계: BTIP-39 sync 데몬 (`prover-ts/src/btip39/`)

> u2r Prover를 두 기능으로 분리(사용자 설계): **① Validator Set 자동 동기화(본 작업)**, ② BPuN Tx/Event 증명 제출(다음). 위치는 **`src/btip39/`** — `src/prover/`(타 개발자 영역, 추후 `src/prover/btip39` 제거 예정)에 **의존성 0** (사용자 지시).

### 동작 (BTIP-39)

1. `BPUN_RPC_URL`을 `BTIP39_POLL_INTERVAL_SEC`(기본 5초)마다 폴링.
2. 커서 이후의 **모든 커밋된 헤더를 순회**하며 `ValidatorsHash(H) != NextValidatorsHash(H)`로 변경 감지. (모든 블록 검사 필수 — A→B→A 중간 변경을 건너뛰면 Sequential 신뢰 사슬이 끊겨 이후 증명이 영구 실패. 스캔 상한은 tip-1 — 증명 구성에 commit(H)·validators(H+1) 필요.)
3. 변경 시 `buildValidatorSetProof(trusted=H)`(노드 값과 self-check) → `goJsonStringify`(bigint 보존) → fabric-network `submitTransaction('UpdateValidatorSet', payloadJSON)`.
4. 상태 파일에 `lastScannedHeight`/`lastValidatorsHash`/`lastSubmittedTarget` 저장(tmp+rename 원자 쓰기, 변경 등록마다 + 틱 종료마다).

### 신뢰/복구 설계

- **상태 파일은 커서 캐시일 뿐** — 기동 시 `LinkerPolicy.GetLatestHeight()`(권위 상태)와 대조: 파일 부재/유실/체인 리셋(BPuN tip보다 큼)/trustedBase 미만이면 trustedBase로 재설정. `GetLatestHeight()==0`이면 H₀ 미부트스트랩으로 기동 거부.
- 제출 실패 시 커서 미전진 → 다음 틱 재시도. 체인코드 멱등(동일 해시 no-op)이라 crash-retry 안전.
- 외부 등록(다른 prover/관리자) 감지 시 해시만 채택하고 로그.

### 파일 (`prover-ts/src/btip39/` — 자체 완결)

- `main.ts` 데몬 루프(SIGINT/SIGTERM 정리), `config.ts` env 로더(dotenv), `state.ts` 커서 영속화, `submitter.ts` fabric-network Gateway — **LinkerPolicy 체인코드명은 항상 BTIP37 registry에서 `GetContract(channel, "LinkerPolicyRole")`로 해석** (사용자 확정: 컴포넌트명 직접 설정 옵션 제거 — registry가 단일 출처, 데몬도 registry 하나만 안다).
- `rpc.ts`/`header-merkle.ts`/`types.ts`/`validator-set-proof.ts` — `src/prover/btip39`에서 **복사 격리**(import 아님). 복사본은 TS 타입명 `BPuNValidatorSetProof`로 개명(와이어 동일). 원본 `src/prover/`는 무변경(git 확인).
- `package.json`: **`npm run btip39:sync`**. `.env.example`에 데몬 절 추가.

### 환경설정 (확정)

필수: `BPUN_RPC_URL`, `FABRIC_CONNECTION_PROFILE`/`FABRIC_CHANNEL_ID`/`FABRIC_MSP_ID`/`FABRIC_USER_NAME`/`FABRIC_CERT_PATH`/`FABRIC_KEY_PATH`(기존 prover API와 동일 키 재사용), **`LINKER_REGISTRY_CC`**(LinkerPolicy는 registry에서 해석 — 직접 지정 없음).
선택: `BTIP39_POLL_INTERVAL_SEC`(5), `BTIP39_STATE_FILE`(./.btip39-state.json), `BTIP39_START_HEIGHT`(기본 GetLatestHeight), `BTIP39_MAX_BLOCKS_PER_TICK`(600).

### 검증 상태

- `npx tsc --noEmit` 통과. `src/prover/` 의존 grep 0.
- ✅ **localnet end-to-end 통과 (2026-06-11)**: H₀(height 1) 부트스트랩 상태에서 데몬 기동 → 캐치업 → `change detected at 2331` → **`UpdateValidatorSet submitted: target_height=2332`**. BTIP-39 trustless 경로 전체(hex 페이로드 → HexBytes 디코드 → commit 서명 2/3+ → RFC6962 index 8 → ValidatorsHash 재계산 → self-call 저장)와 registry 평문 role 해석·상태 파일 커서 재개가 실데이터로 첫 검증됨.

### 트러블슈팅 기록 (e2e 과정에서 해소)

1. **"Unable to find any target committers"** — fabric-network의 committer = **orderer**. `evaluateTransaction`(조회)은 피어만 사용하므로 잘 되다가 `submitTransaction`에서만 실패. 원인: `connection-profile.json`에 `orderers` 절 부재(조회 전용이던 기존 prover API에선 드러나지 않던 공백). **수정**: `channels.bpn.orderers` + top-level `orderers`(`grpcs://orderer0.ordererorg.bc:7050`, ssl-target-name-override, TLS CA) 추가. 데몬의 실패-시-커서-유지 재시도 동작은 의도대로 작동함을 함께 확인.
2. **crypto-config 의존성 정리 (사용자 지시, 2차 개선)**: `kysee/zk-chains` 경로 의존 금지 → crypto 출처는 **`bpn-core-2.2/bprn-test-env`만**. 1차(디렉토리 통 링크)에서 사용자 redirect로 **파일 단위 심볼릭 링크**로 재구성 — `prover-ts/organizations/`(HLF fabric-samples 관용 레이아웃, `msp`는 한 신원 내부 구조명이라 부적합) 아래 표준 경로를 미러링해 **필요 파일 4개만** 링크(peer TLS CA, orderer TLS CA, User1 signcert/priv_sk). 깨진 링크가 곧 필요 파일 목록이 되는 효과. **`scripts/link-bprn-test-env.sh`**(파일 목록의 단일 정의, 누락 검사 포함)로 재생성 — 타 개발자는 이 스크립트 한 번 실행. connection-profile·.env 경로 모두 `organizations/` 경유, `.gitignore`에 `/organizations`. bprn-test-env에는 connection profile이 없음(네트워크 구성 yaml뿐) — SDK 클라이언트 산출물이므로 prover-ts 자체 보유가 맞다고 확인.
   - **상태 파일 위치 관례 (사용자 질문으로 확정)**: 기본값을 cwd에서 **XDG state**(`$XDG_STATE_HOME/linker-prover/btip39-state.json`, 기본 `~/.local/state/...`)로 변경. `BTIP39_STATE_FILE` 오버라이드 유지. 커서는 체인 대조로 자가 복구되는 캐시라 리포 밖이 적합.
4. **(사용자 redirect) `organizations/` → `bprn-organizations/`** — connection-profile.json도 `bprn-organizations/` 안으로 이동(프로파일은 추적, 링크 트리 `peerOrganizations`/`ordererOrganizations`만 gitignore). `.env`의 `FABRIC_CONNECTION_PROFILE` 경로 갱신. bprn-test-env에는 connection profile이 없음을 확인(네트워크 구성 yaml뿐 — SDK 클라이언트 산출물이라 prover 측 보유가 정상).
5. **캐치업 성능 개선 (사용자 지적: "최초 부트업이 너무 느리지 않나")**: ① 헤더 fetch를 **병렬화**(`BTIP39_FETCH_CONCURRENCY`, 기본 25 — 검사·제출은 높이 순 순차 유지, Sequential 신뢰 사슬 보존), ② **캐치업 중에는 틱 간 휴식 생략**(틱이 provable tip(latest-1)에 도달했을 때만 5초 휴식 — 폴 주기는 정상 상태의 페이스만 결정). 제출 실패 시 커서 유지·즉시 중단 후 재시도 동작은 유지. 배치는 "남은 블록 수"로 잘리므로 tip 정상 상태(1초 틱)에선 틱당 0~2개 요청으로 자연 퇴화 — 병렬성은 캐치업에서만 발현(사용자 우려 확인). 제출은 항상 순차 1회라 fetch 동시성으로 인한 중복 제출 없음.
6. **재스캔/멱등 시맨틱 정리 (사용자 질문)**: state 파일만 삭제 시 커서는 `GetLatestHeight()`(trustedBase)로 복구되어 **과거 변경을 재감지하지 않음**(권위=체인 설계 의도). 재감지·재제출은 `BTIP39_START_HEIGHT`를 과거로 명시할 때만 — 클램프를 "명시 지정은 존중(경고)"으로 완화. 재제출 증명은 체인코드가 **전체 검증 수행 후** 동일 해시 no-op 수락(PutState·이벤트 없음 — write-set 빈 tx), 해시 불일치 시 conflicting 에러(방어 검증). 데몬 로그도 구분: `target ≤ trustedBase`면 "re-submitting … (expect idempotent no-op)" → "accepted as idempotent no-op"으로 표시. **localnet에서 재스캔(블록 1부터)→재감지→no-op 수락 전 과정 실증 완료 (2026-06-11).**
7. **conflicting(신뢰 계보 분기) 처리 (사용자 질문)**: target 높이에 *다른 해시*의 셋이 저장돼 있으면 — 한 체인에서 불가능, BPuN 리셋/포크 신호 — 체인코드는 검증 통과 여부와 무관하게 **거부**(permissionless 호출자의 이력 재작성 금지; 복구는 admin `SetValidatorSet` 비상 보정). 데몬은 이 영구 에러를 일반 실패와 구분: `TrustDivergenceError`로 **즉시 정지(exit 2) + admin 개입 안내 로그** (기존엔 1초 무한 재시도였음 — 수정).
3. (잔존 주의) `bpn-core-2.2/scripts/run/3_linker_update_vs.sh`는 hex 전환 이전의 **base64 페이로드 fixture가 하드코딩된 stale 스크립트** — 현행 체인코드에서 실패. 데몬이 그 역할을 대체하므로 제거/갱신 후보.

---

## 0. 작업 핸드오프 (2026-06-10 작성 — 과거 기준선, 최신은 §00)

> 노트북 이동 대비 요약. 상세는 각 날짜 절 참조.

### 0.0 LinkerEndpoint 전면 개정 (2026-06-10, 2일차 — 최신)

**on-bpun (Solidity)**
- `IBTIP21.sol` 재작성: `setRegistry` / `onProof(payload, handlerDApp)` / `onResult(payload, handlerDApp)` / `event LinkerResult(bytes32 indexed correlationId, address indexed handlerDApp, uint8 status)` / `error ErrUntrustedSource(address)`. `ProofReceived`/`ProofVerified` 이벤트, `setNullifierContract`/`setVerifierContract` 제거. 구조체는 불변.
- `IBTIP26.sol`: `handleLinkerResult(bytes32, string, bool)` + `error ErrAppLowGas()` 추가.
- `LinkerEndpoint.sol` 재작성 (`Ownable` + `ReentrancyGuard`):
  - `onProof`: registry 조회(`getContract(block.chainid, LINKER_VERIFIER|NULLIFIER)`) → `markProcessed(txEventRoot, handlerDApp)` **선행** → `verifyProof` → try/catch `handleLinkerEvent`(revert=REJECTED, `ErrAppLowGas` 4-byte selector 매칭 시 전체 revert) → `LinkerResult` 정확히 1회 발행(correlationId=`tx_event_root`).
  - `onResult`: markProcessed → 출처검증(channel_id(gidx:0)→`uint256(sha256(ch||"/BPrN"))` chainId, chaincode_id(gidx:1)→BTIP9 주소 파생, registry의 BPrN LINKER_ENDPOINT와 대조, 불일치 시 `ErrUntrustedSource`) → selector(gidx:3)==`sha256("LinkerResultElems([]byte,string,byte)")` → `verifyProof` → gidx 4/5/6 추출 → `handleLinkerResult(bytes32, string, bool)` (try/catch 없음 — 미스라우팅은 전체 revert로 nullifier 롤백).
  - ⚠️ **스펙 이탈 1 (보안 정정)**: btip-21 의사코드는 onResult에서 `markProcessed(root, address(this))`(전역 소비)인데, 이러면 permissionless onResult를 "아무거나 수락하는 공격자 dApp"으로 흘려 정당한 dApp의 결과 수신을 영구 차단 가능(griefing). **`(root, handlerDApp)` 단위로 구현** — BTIP-29 BPrN 측과 동일 입도. → btip-21 의사코드 정정 필요.
- `BTIP26Dapp.sol`: `handleLinkerResult` mock(`LinkerResultReceived` 이벤트) 추가.
- `deploy.ts`: VERIFIER/NULLIFIER role 등록 + `endpoint.setRegistry` 와이어링으로 교체. `submit-proof.ts`: targetDApp→handlerDApp rename. `LinkerGasTest.t.sol`: 신 구조체명/registry 와이어링으로 수선(forge-std 부재로 미실행).

**on-bprn (Go)**
- `types/ibtip34.go` 신설: `LinkerResultRef{CorrelationIndex int64, Status bool}` + `IBTIP34` 인터페이스.
- `types/registry.go`: `ResolveContractOn(ctx, chainName, role)` 추가(크로스체인 키 조회 — OnResult가 `payload.ChainID`로 BPuN endpoint 조회). `ResolveContract`는 위임 thin wrapper화.
- `linker-endpoint/main.go` 재작성:
  - `OnProof(ctx, payloadJSON, handlerCcId)`: MarkProcessed **선행**(중복=DuplicateProof, Fabric tx 원자성으로 후속 실패 시 롤백) → VerifyProof → `HandleLinkerEvent` 호출 후 `LinkerResultRef` JSON 수신 → `CorrelationIndex>=0`이면 검증된 attr에서 correlationId 추출 후 `LinkerResultElems` EventLog 발행(elems 3개: CorrelationId/HandlerCcId/Status). `ProofVerifiedEventElems` 발행 **제거**(Fabric 1-event/tx 슬롯을 결과 이벤트에 양보).
  - `OnResult(ctx, payloadJSON, handlerCcId)`: MarkProcessed 선행(`(event_attrs_root, handlerCcId)` 단위) → 출처검증(`ResolveContractOn(payload.ChainID, LINKER_ENDPOINT)` vs attr0 contractAddress; attr1 topic.0 vs `keccak256("LinkerResult(bytes32,address,uint8)")` — hex 비교는 0x/대소문자 관용) → VerifyProof → attr2(correlationId 32B)/attr3(topic.2 → 하위 20B 주소)/attr4(data status word) 추출 → `HandleLinkerResult` 호출.
  - ⚠️ **스펙 이탈 2**: btip-29 OnResult 의사코드는 IsProcessed 선검사+verify 후 Mark인데, Fabric 원자성 하에서 동치이므로 OnProof와 동일하게 Mark 선행으로 단순화(InvokeChaincode 1회 절약).
- `dapp-example/main.go`: `HandleLinkerEvent` → `(LinkerResultRef, error)` 반환(예제는 fire-and-forget `CorrelationIndex=-1`), `HandleLinkerResult` 신설(`RES_`+correlationId hex 키로 `ResultRecord` 저장, 32B/20B 길이 검증, caller=endpoint 검증), `var _ types.IBTIP34` 컴파일 단언.

**스펙 문서 정정 — 완료 (2026-06-10, 사용자 승인 후 docs 리포 반영)**
1. ✅ btip-21 onResult 의사코드: gidx 4/8/5 → **4/5/6** 정정, `markProcessed(root, address(this))` → **`(root, handlerDApp)`** 보안 정정(per-handler 소비 rationale 주석 포함), 출처검증을 BTIP37/BTIP9 파생 규칙(channel_id→chainId, chaincode_id→주소)으로 구체화, onProof/onResult 단계 번호 중복(2/2, 5/5) 정리.
2. ✅ LinkerResultElems selector **`LinkerResultElems([]byte,string,byte)`** 통일(btip-21 1곳, btip-29 2곳) + btip-29 표 타입 오타 정정(CorrelationId byte→bytes, Status bytes→byte).
3. ✅ btip-29 OnResult 의사코드 Mark 선행으로 정정(IsProcessed+후행 Mark 제거, `(event_attrs_root, handlerCcId)` 소비 단위 명시, ChainID 검증 근거 주석, 필드 추출을 value_at_index(2/3/4)로 구체화).
→ 코드와 스펙 문서 정합 상태. docs 리포 커밋은 미수행(사용자 직접 커밋).

**검증 상태 (2일차)**
- Solidity: solc 0.8.28(cancun)+OZ 5.6.1 standalone 컴파일 — 컨트랙트 전체(36 파일) **통과**(BTIP26Dapp 호출자 검증 반영 후 재확인). `npx hardhat compile` 정식 실행은 여전히 미수행(환경 제약) — 확인 필요.
- Go: 툴체인 부재로 컴파일 미수행 — **`cd verifier/on-bprn && go build ./...` 1순위**. 정적 점검(시그니처·import·brace balance)만 완료. role 평문화로 `types.RoleID`/`RoleIDHex` 삭제됨 — 잔존 참조 없음 확인.
- forge 가스 테스트: 구조체명/registry 와이어링/`dapp.setRegistry` 수선 완료, forge-std 부재로 미실행.
- localnet end-to-end: 미실행. OnResult의 BTIP35 attr 값 인코딩(0x prefix 유무, topic 대문자) 가정은 관용 비교로 처리했으나 **u2r prover 구현 시 localnet 실데이터로 검증 필요**.

### 0.0.1 세션 후반 작업 (2026-06-10, 운영·스크립트·역할 식별자)

1. **deploy.ts 배포 검증 단계 추가**: 와이어링 후 4개 role을 `registry.getContract(BigInt(config.chainId), role)`로 조회해 배포 주소와 대조, 불일치 시 throw. **핵심 발견: `@beatoz/web3`의 `contract.methods.X().call()`은 반환값을 ABI 디코딩하지 않고 `vm_call` RPC 결과 `{value:{returnData:'0x<raw>'}}`를 그대로 반환** — address는 `returnData` 마지막 20바이트로 직접 디코딩해야 함(`decodeAddressResult`). `config.chainId`(hex, 예: 0xbea700)가 EVM `block.chainid`와 일치해야 검증이 통과(불일치 시 ErrUnknownContract로 표면화).
2. **배포 순서 변경**: LinkerRegistry가 항상 1순위(사용자 지시). Registry → Endpoint → Nullifier → Verifier → Policy → PolicyVerifier → Dapp.
3. **LINKER_POLICY 바인딩**: policy(데이터 컨트랙트) 기준 제안이 있었으나 LinkerVerifier가 resolved 주소에 IBTIP22를 직접 호출하는 의존성 때문에 **policyVerifier 유지 결정**("policyVerifier 로 일단 가자") — §0.4 보류 항목 + deploy.ts NOTE 참조.
4. **bpn-core-2.2 localnet 스크립트 수정** (`scripts/run/3_linker_init.sh`, `4_query_linker_registry.sh`): (a) `SetContract` 인자 순서 `(channel, chaincodeName, role)` 교정, (b) 체인코드별 부트스트랩 메소드명 교정(`SetRegistryID`=endpoint/verifier/BTIP34CC, `SetRegistry`=nullifier), (c) BTIP34CC 부트스트랩 추가, (d) 사용자에 의해 `IS_INIT=""` 처리됨(재실행 시 already-initialized 회피).
5. **BPrN role 식별자 평문화** (§0.5 컨벤션 참조): btip-37은 **사용자가 직접 개정** — BPrN Roles 표의 식별값은 `"LinkerEndpointRole"` 등 "Role" 접미사 포함 평문. `types.RoleLinker*` 상수 값 = 와이어 값. 코드·스크립트 정합 완료.
6. **BTIP26Dapp 호출자 검증**(BTIP-26 IMPORTANT 의무): `onlyLinkerEndpoint` modifier(registry 조회로 msg.sender 검증, `ErrUnauthorizedCaller`) + dApp도 registry 단일 보유 전환(`setNullifierContract` 폐기→`setRegistry`, cancel 경로 nullifier도 registry 해석).
7. **재배포 필요**: linker-registry·linker-endpoint(체인코드), BPuN 컨트랙트 전체(IBTIP21/26 인터페이스 변경). localnet은 네트워크 초기화 후 재실행 권장(이전 실행의 nullifier init 및 hex-role 잔존 데이터).

### 0.1 한 일 (2026-06-10 1일차, 시간순)

1. **문서 후속 정리** (사용자 지시): btips-2pc-design.md §5 재정정(correlationId — BPrN-origin=`tx_event_root` 강제 / BPuN-origin=명시 id 비대칭 회귀), bpun-origin-payment-design.md **§16 신설(PaymentBridge 폐기 — STC 체인코드 기본 기능으로 통합, approve 포함)**, btip-39 버전 v0.34.24 통일, 오탈자 일괄, btip-40 필드 재정합(correlationId 선두 + beneficiary 추가 — btip-25와 시퀀스 대칭 복원), btip-25 Beneficiary NOTE(EIP-5564 **Stealth Address** — "Secure Address" 아님).
2. **LinkerRegistry 구현** — §"2026-06-10 — LinkerRegistry 구현" 참조. uint256 chainId, Err* 에러, BPrN keccak role(hex 와이어), `setSelfContract` 편의 함수.
3. **LinkerPolicy/BPrN 구현** — §"2026-06-10 — LinkerPolicy/BPrN 구현" 참조. hex DTO 폐기 → btip-32 스펙 구조체, `BPuNValidatorSetProof` 개명, Address-PubKey 파생 일치 강제. (LinkerPolicy/BPuN 정책 엔진은 **불가침** — 사용자 지시.)
4. **hex 와이어 전환** (사용자 redirect: "base64 말고 hex") — §"2026-06-10 — on-bprn 와이어 인코딩 hex 전환" 참조. `types.HexBytes` 도입. **핵심 발견: contractapi는 최상위 byte-slice 파라미터에 raw 문자열을 그대로 주입**(JSON 미경유) → 최상위 byte 파라미터는 hex string으로 선언하는 원칙 확정.
5. **LinkerNullifier 구현** — §"2026-06-10 — LinkerNullifier 구현" 참조. markProcessed 내부 revert + `DuplicateProof` IBTIP24로 이동, setRegistry 패턴, BPrN `SetRegistry`/`ccAppId`.
6. **LinkerVerifier 구현** — §"2026-06-10 — LinkerVerifier 구현" 참조. setRegistry 패턴, 페이로드 rename(`BPrNTxEventProof`/`tx_event_root_proof` — Solidity·스크립트·prover-ts 와이어까지), **스펙 정정**(btip-19/23 Step 5 리프는 preHashed — 재해싱 아님), endpoint→nullifier base64 인자 잠재 버그 2건 해소.
7. **`BPuNTxEventProof` 개명 확정** (사용자: "BPuNTxEventProof이 맞는거야") — btip-28 정의 문서 포함 전면 개명 완료.

### 0.2 컴파일/검증 상태

- **Solidity**: solc 0.8.28(cancun) 단독 컴파일로 5개 컨트랙트(Registry/Nullifier/Endpoint/Verifier/BTIP26Dapp) 통과. ⚠️ 환경 제약으로 `npx hardhat compile` 정식 실행은 미수행 — **집에서 1순위 확인**.
- **Go (on-bprn)**: 환경에 Go 부재로 **컴파일 미수행** — **집에서 1순위로 `cd verifier/on-bprn && go mod vendor && go build ./...`**. 정적 점검(시그니처·grep)만 완료. `golang.org/x/crypto/sha3` 직접 import 추가됨(vendor 기존재, go.mod indirect → tidy 시 정리됨).
- **prover-ts**: `npx tsc --noEmit` 통과.
- **미실행 테스트**: BPrN-origin end-to-end(submit-proof) — 페이로드 필드 rename(`tx_event_root_proof`)이 prover API 응답↔스크립트↔컨트랙트에 일관 적용됐는지 localnet 재검증 필요.

### 0.3 다음 작업 (우선순위)

1. **(집 도착 직후) 빌드 검증**: on-bprn `go build ./...`, on-bpun `npx hardhat compile`(+`forge build` 가능 시). 깨지면 §0.0 변경 파일 목록으로 추적.
2. ~~**LinkerEndpoint 전면 개정 (BTIP-21/29)**~~ — **완료 (2026-06-10 2일차, §0.0)**. 스펙 문서 정정 3건도 docs 리포 반영 완료(§0.0 — 커밋만 잔여).
3. **BTIP-26/34 콜백 + BTIP-40 표준 이벤트** — `handleLinkerResult`/`HandleLinkerResult` 기본형은 §0.0에서 구현됨. BTIP26Dapp registry 기반 호출자 검증(`onlyLinkerEndpoint` modifier) 완료(2026-06-10 — dApp도 registry 단일 보유로 전환: `setNullifierContract` 폐기 → `setRegistry`, deploy.ts/gas test 정합). 잔여: 2PC pending 상태 모델, dapp-example의 `TransferLogAttrs` 처리(`CorrelationIndex=2` 반환 예시).
4. **u2r Prover** (BPuN→BPrN, end-to-end 미싱 피스) — `prover-ts/src/prover/u2r/` 신설. **hex 와이어 기준**. btip39 prover의 header-merkle 재사용. OnResult가 가정한 BTIP35 attr 인코딩(0x/대소문자)도 여기서 실데이터 검증.
5. multi-peer block-commit-sig 수집.

### 0.4 보류/관찰 (사용자 결정 대기 또는 타인 영역)

- ~~**LINKER_POLICY 바인딩 재설계**~~ / ~~**IBTIP22 메소드명 불일치**~~ — **해소 (2026-06-14 코드 확인, 사용자 직접 병합 반영)**: 별도 `LinkerPolicyVerifier` 컨트랙트는 폐지되고 `LinkerPolicy.sol`이 `contract LinkerPolicy is IBTIP22`로 **Trust Anchor 데이터(채널·정책·org 인증서/CRL)와 검증(`verifyChannelEndorsementPolicy`)을 단일 컨트랙트에 보유**. 검증 로직은 `lib/LinkerPolicyVerifierLib.sol`(library)로 분리. registry의 LINKER_POLICY role도 deploy.ts에서 `LinkerPolicy` 주소를 가리킴 → 사용자가 원하던 "데이터 컨트랙트 기준"이 자연 충족되어 getter·2-hop 조회 불요. 메소드명도 LinkerVerifier·LinkerPolicy 양쪽이 스펙명 `verifyChannelEndorsementPolicy`로 일치. 잔재: `scripts/.net/deployed.localnet0.LinkerPolicyVerifier.json`은 소스 없는 옛 배포 기록(삭제 후보), deploy.ts 옛 NOTE는 2026-06-14 정정 완료.
- **결제 이벤트 정의 BTIP** 미작성 (권한 3분기 명문화 선결 조건 — bpun-origin-payment-design §15.9).
- STC use case 잔존: settle 행선지, approve BPrN/BPuN 동기화 (§10-8 (a)(c) — PaymentBridge 분리 문제는 §16에서 해소).

### 0.5 오늘 확정된 컨벤션 (코드 작성 시 적용)

- **와이어 byte 값 = hex** (base64 금지): 구조체 필드는 `types.HexBytes`, 최상위 파라미터는 hex string 선언(contractapi raw passthrough 때문). c2c 전용 []byte는 raw 허용.
- **에러 `Err` prefix** (스펙 정의 에러는 스펙 이름 그대로: `DuplicateProof`, `BlockEventMerkleProofFailed` 등).
- **모든 컴포넌트는 LinkerRegistry 주소 하나만 보유** — `getContract(block.chainid, LINKER_*)`(BPuN) / `ResolveContract(role)`(BPrN). 부트스트랩 메소드명: BPuN `setRegistry`, BPrN은 BTIP별로 `SetRegistryID`(endpoint/verifier) vs `SetRegistry`(nullifier).
- **role 식별자 체인별 분리 (2026-06-10 사용자 결정, btip-37은 사용자 직접 개정)**: BPuN = `keccak256(roleName)` bytes32(현행 유지), BPrN = **평문 식별 문자열 `"LinkerEndpointRole"` / `"LinkerVerifierRole"` / `"LinkerPolicyRole"` / `"LinkerNullifierRole"`** (btip-37 "BPrN-Specific Considerations" Roles 표가 기준). 두 체인이 동일 식별자 인코딩을 공유할 필요 없음. `types.RoleID`/`RoleIDHex` 삭제, `types.RoleLinker*` 상수 값이 곧 와이어 값, linker-registry 평문 role(+`_` 금지 검증), bpn-core-2.2 스크립트 정합 완료. btip-37 NOTE: onResult/OnResult 출처검증을 위해 **상대 체인 LinkerEndpoint를 자기 체인 LinkerRegistry에 등록**해야 함.
- 합의-크리티컬 연산은 tendermint-ethaddr 포크 직접 호출, BTIP-32 구조체는 순수 직렬화 경계.

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

## 미완료 (2026-06-10 재정리)

> 자세한 현황과 새로 발견된 갭은 §"2026-06-01 — 재동기화" + §"2026-06-10 — BTIP 리뷰 사이클 코드 갭" 참조.

- **BPuN→BPrN 이벤트 Prover 미구현** (end-to-end의 결정적 미싱 피스). `prover-ts/src/prover/u2r/` 디렉토리 자체 부재 — 감사 시점의 placeholder도 사라짐.
- **2PC 미구현** — 설계는 `btips`/`btips-2pc-design`에 완결돼 있으나 코드 미착수. 누락 항목(2026-06-10 개정 스펙 기준): on-bpun(`IBTIP21.onResult(payload, handlerDApp)`, `LinkerResult(correlationId, handlerDApp, status)` 이벤트, `handleLinkerResult(correlationId, handleCcId, status)`, `try/catch` + `IBTIP26.ErrAppLowGas` 분기, `nonReentrant`), on-bprn(`OnResult(ctx, payload, handlerCcId)`, `HandleLinkerResult`, `LinkerResultElems` 3 elems). ~~`MIN_CALLBACK_GAS`~~ — 스펙에서 제거됨(2026-06-01).
- **Prover multi-peer block-commit-sig 수집** — 여전히 단일 블록 조회(피어 1개 sig). BTIP-17 검증 강도를 끌어올리려면 각 endorser 피어에게 개별 블록 조회 필요.
- ~~**on-bpun 정책 엔진 문서화**~~ — **해소(2026-06-10)**: ryan의 LinkerPolicy 스위트(btip-15/18/22/38)가 해당 스펙 문서. 코드 ↔ 스펙 정합 확인은 §"2026-06-10 — BTIP 리뷰 사이클 코드 갭" 참조.
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

---

## 2026-06-10 — BTIP 리뷰 사이클(2026-06-05~10) 코드 갭

> 사용자 직접 리뷰로 BTIP 스펙이 개정됨 (상세: `task-contexts/btips.md` §"2026-06-05 ~ 2026-06-10"). 코드는 미변경이므로 §"2026-06-01 — 재동기화" §1.B의 기존 갭에 **아래 항목이 추가/변경**된다. 구현 착수 시 §1.B와 본 절을 함께 소화할 것.

### on-bpun (Solidity) 추가 갭

| 스펙 변경 | 코드 영향 |
|---|---|
| 파라미터명 `targetDApp` → `handlerDApp` (BTIP-21/24/26) | `IBTIP21/24/26.sol` + 구현체 전반 rename |
| `onProof(payload, handlerDApp)` + **`onResult(payload, handlerDApp)`** — onResult도 handler 명시 수신 (BTIP-21) | `onResult` 신설 시 새 시그니처 기준. OriginContract 기반 자동 라우팅 설계 폐기됨 |
| `LinkerResult(bytes32 correlationId, address handlerDApp, uint8 status)` 3필드 (BTIP-21) | 2PC 구현 시 이 시그니처로. 구 5필드(originChannelId/originChaincodeId 포함) 설계 폐기 |
| `markProcessed`를 **암호학적 검증보다 선행** 호출 (BTIP-21 onProof/onResult) | `LinkerEndpoint.onProof` 호출 순서 변경 (현 코드는 verify → checkAndMark 순) |
| 에러 `Err` prefix — `ErrAppLowGas`(BTIP-26), `ErrUntrustedSource(address)`(BTIP-21), `ErrUnknownContract`/`ErrUnauthorized`(BTIP-37) | ~~`LinkerRegistry.sol` rename~~ **✅ 반영(2026-06-10 구현)**. `ErrAppLowGas`/`ErrUntrustedSource`는 미구현 항목의 명칭 확정 |
| `handleLinkerResult(bytes32 correlationId, string handleCcId, bool status)` (BTIP-26) | 2PC 구현 시 이 시그니처로 |
| **IBTIP37 `chainId` 타입 `bytes32` → `uint256`** + BPrN chainId = `uint256(sha256(channelName + "/BPrN"))` (BTIP-37) | ~~`IBTIP37.sol`/`LinkerRegistry.sol` 키 타입 변경~~ **✅ 반영(2026-06-10 구현, §아래 LinkerRegistry 구현 절)**. 컴포넌트들의 `bytes32(block.chainid)` 캐스팅 제거는 setRegistry 구현 시 적용 |
| `TransferLogAttrs`(BTIP-40) 개정 (2026-06-10) — `(bytes32 indexed correlationId, address indexed from, address indexed to, uint256 amount, address beneficiary, bytes memo)`, BTIP-25와 필드 시퀀스 동일 | 표준 event 자체가 코드 미구현 (기존 갭) — 구현 시 개정 시그니처 기준 |

### on-bprn (Go) 추가 갭

| 스펙 변경 | 코드 영향 |
|---|---|
| 타입명 `BPuNTxEventProofPayload` → `BPuNTxEventProof` (BTIP-29) | `types/types.go` rename |
| `OnProof(ctx, payload, handlerCcId)` / **`OnResult(ctx, payload, handlerCcId)`** — 파라미터명 `appChaincodeID` → `handlerCcId`, OnResult도 handler 명시 수신 (BTIP-29) | `linker-endpoint/main.go` 시그니처 정합. originChaincodeId 기반 자동 라우팅 폐기 |
| `MarkProcessed`를 암호학적 검증보다 선행 (BTIP-29 OnProof) | OnProof 단계 순서 변경 |
| **`ProofVerifiedEventElems` 제거** — fire-and-forget은 이벤트 미발행 (BTIP-29) | `linker-endpoint`가 해당 이벤트를 발행 중이면 제거 |
| `LinkerResultElems` 3 elems: CorrelationId(g4)·HandlerCcId(g5)·Status(g6), selector `sha256("LinkerResultElems([]bytes,string,byte)")` (BTIP-29) | 2PC 구현 시 이 구조로. OriginChainId/OriginContract 폐기 |
| `LinkerResultRef{CorrelationIndex int64, Status bool}` — `Accepted` → `Status` (BTIP-34) | 2PC 구현 시 json 태그 `status` |
| `HandleLinkerResult(ctx, correlationId, handlerDApp, status)` / `CancelLinkerEvent`는 **Admin 전용** (BTIP-34) | dapp-example 콜백 시그니처 + CancelLinkerEvent 접근 제어 |
| Nullifier 파라미터명 `appChaincodeID` → `ccAppId`, `SetLinkerEndpointID` → `SetRegistry` (BTIP-33) | `linker-nullifier/main.go` — SetRegistryID 패턴은 기 구현(2026-05-26), 메소드명 `SetRegistry` 정합 여부 확인 |
| `SetLinkerPolicyID` → `SetRegistryID` (BTIP-31) | 기 구현(2026-05-26 레지스트리 리팩토링)과 일치 — 확인만 |

### LinkerPolicy 스위트 (btip-15/18/22/38, ryan) ↔ on-bpun 정책 엔진

- §"2026-06-01 — 재동기화" §3에서 "본문 미기재"로 관찰했던 on-bpun 정책 엔진(`LinkerPolicyVerifier`/`LinkerPolicyLib`/`LinkerPolicyTypes`/`SignatureVerifier` + `lib/*`)의 **스펙 문서가 생겼다**: btip-15(데이터셋) → btip-38(Orderer 추출·블록 메타데이터 index 6 기록) → btip-18(`initPolicy`/`syncPolicy` 주입·동기화) → btip-22(`verifyChannelEndorsementPolicy` 검증).
- **확인 필요**: 코드의 `verifyBlockValidationPolicy`/`verifyChannelEndorsement(Policy)`/`verifyPolicy`와 btip-18/22 정의(`initPolicy`/`syncPolicy`/`verifyChannelEndorsementPolicy` 시그니처·에러)의 정합 여부. btip-38은 BPrN 코어(Orderer) 측 작업이라 linker-v2 리포 범위 밖(bpn-core 쪽).

### 우선순위 영향

§"2026-06-01" §2의 우선순위(1. u2r Prover, 2. 2PC, 3. multi-peer sig)는 유지하되, 2PC 구현은 **본 절의 개정 시그니처 기준**으로 진행한다.

---

## 2026-06-10 — LinkerRegistry 구현 (BTIP-37 개정 정합)

> 본격 구현 사이클의 첫 작업. BTIP-37 개정 스펙(uint256 chainId, Err* 에러, BPrN string 인터페이스 + keccak role)을 양 체인 코드에 반영. 기존 구현을 대체.

### on-bpun (Solidity)

- **`interfaces/IBTIP37.sol` 재작성**:
  - `chainId` 타입 `bytes32` → **`uint256`** (event/error/getContract/setContract 전부).
  - 에러 rename: `UnknownContract` → **`ErrUnknownContract(uint256, bytes32)`**, `Unauthorized` → **`ErrUnauthorized()`**.
  - 주석의 chainId 규약 갱신 — self = `block.chainid`(캐스팅 없음), BPrN = `uint256(sha256(channelName + "/BPrN"))`, BPrN 체인코드 주소 = BTIP9 `sha256(channelName + "-" + chaincodeName)` 하위 20B.
- **`LinkerRegistry.sol` 재작성**:
  - 매핑 `mapping(uint256 => mapping(bytes32 => address))`.
  - `setContract`를 `onlyOwner` modifier 대신 **`onlyGovernance`**(내부 `if (msg.sender != owner()) revert ErrUnauthorized();`)로 게이트 — revert가 BTIP-37 표준 에러로 나가도록 (OZ `OwnableUnauthorizedAccount` 대신).
  - **BPrN 변환 헬퍼 2개 신설** (public pure, 인터페이스 외 구현 편의): `bprnChainId(channelName)` = `uint256(sha256(channelName ‖ "/BPrN"))`, `bprnChaincodeAddress(channelName, chaincodeName)` = `address(uint160(uint256(sha256(channelName ‖ "-" ‖ chaincodeName))))` (하위 20B = index 12~31).
  - Role 상수 4개(keccak256) 유지. CREATE2/거버넌스 TODO 주석 유지(강한 거버넌스는 별도 BTIP 여지로 문구 조정).
- **`scripts/beatoz/utils.ts`**: CUSTOM_ERRORS 테이블 — `82b42900 Unauthorized()` 제거, **`cc12cef6 ErrUnauthorized()`** + **`4a696776 ErrUnknownContract(uint256,bytes32)`** 추가.

### on-bprn (Go)

- **role 식별자 = keccak256(role name) 32B로 통일** (설계 해석): BTIP-37 Roles 절("역할 이름의 keccak256 해시를 bytes32 식별자로 사용")이 인터페이스 공통 규약이고, BPrN 인터페이스의 `role []byte`는 그 32B 식별자로 해석. 양 체인이 동일 role 키를 공유하게 됨. (이전 구현은 role-name string)
- **`types/ibtip37.go` 재작성**: `IBTIP37` 시그니처를 스펙 그대로 — `GetContract(ctx, channelName string, role []byte) (string, error)`, `SetContract(ctx, channelName, chaincodeName string, role []byte) error` (파라미터명 channelID/chaincodeID → channelName/chaincodeName, SetContract 인자 순서 = 스펙). **`RoleID(name) []byte`** 헬퍼 신설(`golang.org/x/crypto/sha3` NewLegacyKeccak256 — vendor에 기존재). Role* name 상수 유지. 기대 해시값 4종을 주석으로 cross-check 기록.
- **`linker-registry/main.go` 재작성**: World-State 키 = `REG_<channelName>_<hex(role)>`. `role` 길이 32B 검증. 이벤트 `ContractRegistered` payload `{channel_name, chaincode_name, role(hex)}`.
- **`types/registry.go` `ResolveContract`**: 호출자 시그니처 유지(`(ctx, roleName string)` — endpoint/verifier/nullifier/dapp 4개 호출처 무변경). 내부에서 `RoleID(role)`을 계산해 `json.Marshal`(→ `"<base64>"`)로 인자 구성 — contractapi `[]byte` 파라미터의 JSON 경계 규약.

### 부트스트랩 순서 갱신 (on-bprn)

> **2026-06-10 (hex 전환) 갱신**: 최초 구현의 "quoted base64 role 인자"는 같은 날 hex 전환 작업(§"2026-06-10 — on-bprn 와이어 인코딩 hex 전환")에서 **plain hex string**으로 교체됨. 아래가 현행.

`SetContract` 인자 순서·형식: `(channelName, chaincodeName, role)`, role은 **hex(keccak256(roleName))** 64자 문자열(0x 허용).

| role | keccak256 hex (CLI 인자 그대로) |
|---|---|
| LinkerEndpoint | `4ed35b94634c3f9fd25a8993e4528daad7eb89221b566ea0af412ad4a03405d3` |
| LinkerVerifier | `822055b50a5a52035b99bbc44511444e8cc9f3d81703d9468857e6931672e0bb` |
| LinkerPolicy | `e7465b2d910809841cf34ad7fef5d296fa00ac87b94b155e839049deb8b58bf6` |
| LinkerNullifier | `2d8fbdab736f2df6d8a770a3ad0847ec0ccdc713e0b14586f0c1845074c0f253` |

예: `{"Args":["SetContract","localchannel0","linker-verifier","822055b50a5a52035b99bbc44511444e8cc9f3d81703d9468857e6931672e0bb"]}`. 이후 단계(각 컴포넌트 `SetRegistryID`, `SetValidatorSet`)는 §"2026-05-26 (구현 세션)" 부트스트랩 순서와 동일.

### 검증 상태

- **Solidity**: 샌드박스 Hardhat은 HHE21(기존 제약) → **solc 0.8.28(cancun) 단독 컴파일로 검증 통과** (LinkerRegistry.sol + IBTIP37.sol + OZ Ownable, 에러 0). 로컬에서 `npx hardhat compile` 재확인 권장.
- **Go**: 샌드박스에 Go 미설치(기존 제약) → 로컬 `cd verifier/on-bprn && go build ./...` 필요. `golang.org/x/crypto`는 go.mod에 indirect로 기존재 + vendor에 sha3 포함 — 직접 import로 전환되므로 `go mod tidy` 시 indirect 마크만 제거됨.
- 에러 셀렉터 검증: `keccak256("ErrUnauthorized()")[:4] = cc12cef6`, `keccak256("ErrUnknownContract(uint256,bytes32)")[:4] = 4a696776` (viem으로 계산).

### 남은 연결 작업 (LinkerRegistry 후속)

- BTIP-21/23/24 `setRegistry` 단일화 시 본 개정 키 타입(`uint256 chainId`) 기준으로 조회 코드 작성 — `getContract(block.chainid, LINKER_*)`.
- deploy 스크립트(`deploy.ts`)에 LinkerRegistry 배포·등록 단계 추가 (현재 스크립트는 레지스트리 미배포).

---

## 2026-06-10 — LinkerPolicy/BPrN 구현 (BTIP-32/39 개정 정합)

> 구현 사이클 2번째 작업. 개정 btip-32/39 기준으로 on-bprn `linker-policy` 정합. **LinkerPolicy/BPuN(정책 엔진)은 범위 밖 — 미변경** (사용자 지시).

### 데이터 모델 교체 — hex DTO 폐기, 스펙 구조체 채택

- **`types/validatorset_dto.go` 삭제** — `ValidatorSetDTO`/`ValidatorDTO`/`HexToBytes`/`BytesToHex` 제거. 2026-05-26에 도입했던 "외부 = hex 문자열" 경계 모델을 폐기하고, btip-32 Data Structures의 **스펙 구조체를 직렬화 경계로 직접 사용**:
  - `types/ibtip32.go` — `Validator{Address(20B), PubKey(33B), VotingPower int64}`, `ValidatorSet{Height, Validators, TotalPower(optional)}`. byte 필드는 contractapi JSON 경계에서 ~~base64~~ → **hex** (같은 날 hex 전환, §"2026-06-10 — on-bprn 와이어 인코딩 hex 전환").
  - `ToProto()`(tmproto 변환, hex 파싱이 없어져 **무오류 시그니처로 단순화**)·`Entries()`(infallible) 메소드를 스펙 구조체로 이동. 크립토 내부 = tmproto 유지(직접 구현 금지 원칙 그대로).
  - `Validator.Address`는 `metadata:",optional"` — SetValidatorSet 입력에서 생략 가능.
- **근거**: 사용자 리뷰를 거친 btip-32가 []byte 스펙 구조체를 유지·확정했고, BTIP-37 구현에서 role []byte로 이미 "JSON 경계 = base64" 방향이 섰음. hex 편의 계층은 스펙에 없는 비표준 표면.

### linker-policy/main.go 재작성

- **BTIP-32**:
  - `SetValidatorSet(ctx, *ValidatorSet)` — admin 전용. `normalizeValidatorSet`: pubkey 33B·voting power>0 검증 + **Address-PubKey 파생 일치 강제**(tendermint-ethaddr `ValidatorAddressFromPubKey`; Address 생략 시 자동 파생, 불일치 시 거부 — 이전 구현은 hex 형식만 검사) + TotalPower 재계산.
  - `GetValidatorSet(ctx, height) (*ValidatorSet, error)` — 범위 조회(VS_ 16자리 0-패딩 키) 유지.
  - `GetValidator(ctx, height, address []byte)` — **스펙 시그니처로 변경** (구: hex string 주소). 20B 길이 검증.
  - `GetLatestHeight` 유지.
- **BTIP-39**: `UpdateValidatorSet(ctx, *BPuNValidatorSetProof)` — 페이로드 타입명 **`ValidatorSetProofPayload` → `BPuNValidatorSetProof`** (개정 btip-39 명명). 검증 5단계(Step 0 무결성 → Step 1 trusted set 조회 → Step 2 Commit 서명 2/3+ → Step 3 RFC6962 헤더 증명(index 8) + amino 디코드 → Step 4 ValidatorsHash 재계산 일치 → Step 5 self-call 저장)·멱등 처리 로직은 기존 구현 유지(스펙 무변). json 태그 동일 — **prover-ts btip39 출력과 와이어 호환 유지** (`trusted_height`/`pubkey`(base64)/`voting_power`/`next_validator_set` 확인).
- `types/ibtip39.go` — 타입·인터페이스 주석 정합. `types/types.go` — DTO 참조 주석 갱신.

### 파급 수정

- `linker-verifier/main.go` `fetchValidatorSet` — DTO 언마샬 → `types.ValidatorSet` 언마샬 + `ToProto()` (무오류).
- **부트스트랩 입력 형식**: `SetValidatorSet` JSON byte 필드 = ~~base64~~ → **hex 문자열**(같은 날 hex 전환) — `{"height":H₀,"validators":[{"pub_key":"<hex 66자>","voting_power":N}]}` (`address` 생략 가능 — 자동 파생, `total_power` 생략 가능). jq escape 주의사항(`--arg`/`--rawfile`)은 종전과 동일.

### 검증 상태

- 샌드박스 Go 미설치 → 로컬 `cd verifier/on-bprn && go build ./...` 필요. 정적 점검 완료: tmverify/auth 헬퍼 시그니처 일치, DTO 잔존 참조 0(grep), prover-ts 와이어 필드명 일치.
- ~~기존 World-State와 저장 형식 비호환(hex JSON → base64 JSON)~~ → 같은 날 hex 전환(아래 절)으로 와이어/저장 모두 hex 유지. 단 구조 변화(필드 구성)는 있으므로 재부트스트랩은 여전히 권장.

---

## 2026-06-10 — on-bprn 와이어 인코딩 hex 전환 (사용자 redirect)

> 사용자 지적: "왜 base64? hex가 가독성에 좋고 BPuN도 hex를 쓴다." — encoding/json의 `[]byte` 기본(base64)을 따랐던 것을 **hex 와이어로 전면 전환**. 아울러 contractapi 직렬화 동작을 vendor 소스로 실측하여 설계 원칙을 확정.

### contractapi 직렬화 실측 (vendor `fabric-contract-api-go/v2` 소스 확인)

- **최상위 파라미터가 byte-slice kind이면**(named type 포함, `types.IsBytes`) JSON 처리 없이 **arg 문자열의 raw bytes를 그대로** 파라미터에 넣는다 (`convertArg`: `reflect.ValueOf([]byte(paramValue))`). 반환도 `%s` raw 출력.
  - → 직전 LinkerRegistry 구현의 `role []byte` + quoted-base64 인자는 **버그였음** (chaincode가 quoted-base64 문자열의 raw bytes를 role로 받게 됨). 본 전환에서 해소.
- **구조체(포인터) 파라미터/반환**은 `json.Unmarshal`/`json.Marshal` 경유 → 커스텀 `MarshalJSON`/`UnmarshalJSON`이 동작.

### 설계 원칙 (확정)

1. **구조체 byte 필드** → `types.HexBytes`(신규, `types/hexbytes.go`): `[]byte` underlying(API에 무변환 전달 가능), JSON은 소문자 hex(입력 0x 허용). **JSON null·빈 문자열 → nil** (BTIP16 머클 null 센티넬 보존).
2. **최상위 byte-값 파라미터** → **hex string으로 선언** + 내부 `HexToBytes` 디코드 (contractapi raw-passthrough 때문에 []byte 선언으로는 hex/base64 어느 쪽도 안전하게 못 받음).

### 변경 파일

- `types/hexbytes.go` 신규 — `HexBytes` + `HexToBytes`(0x 허용; DTO 폐기 때 지웠던 헬퍼 부활).
- `types/types.go` — `RFC6962Proof{Leaf, Aunts}`, `MerkleProof{Leaf, Siblings}`, `ValidatorSignature{ValidatorAddress, Signature}`, `BPuNTxEventProofPayload{BlockHash, BlockPartsHash}` → HexBytes. (**BPuN-origin 증명 페이로드 와이어 전체가 hex로** — 향후 u2r Prover도 hex 기준.)
- `types/ibtip32.go` — `Validator{Address, PubKey}` → HexBytes. `GetValidator(ctx, height, address string)` — hex string 파라미터(스펙 []byte의 와이어 표현임을 주석 명시).
- `types/ibtip39.go` — `BPuNValidatorSetProof{BlockHash, BlockPartsHash}`, `SimpleValidatorEntry{PubKey}` → HexBytes.
- `types/ibtip37.go` — `role` 파라미터 `[]byte` → **hex string** + `RoleIDHex(name)` 헬퍼. `linker-registry/main.go` — `decodeRole`(hex 디코드 + 32B 검증), 키는 종전과 동일 `REG_<channel>_<hex>`.
- `types/registry.go` — `ResolveContract`가 `RoleIDHex(role)`을 plain 인자로 전달 (base64 marshal 제거).
- `types/merkle.go` — `VerifyMerkleProof(siblings []HexBytes)` 시그니처 변경. `types/tmverify.go` — `VerifyRFC6962`에서 aunts `[][]byte` 변환 후 `merkle.Proof` 구성.
- `linker-policy/main.go` — GetValidator hex 파라미터 반영.
- **prover-ts btip39** — `validator-set-proof.ts` 출력 base64 → hex(7곳: validator_address/signature/block_hash/block_parts_hash/leaf/aunts/pubkey), `types.ts` 주석 정정. `npx tsc --noEmit` 통과.

### 운영 영향

- `SetValidatorSet` H₀ JSON: byte 필드 = hex 문자열(0x 허용) — `{"height":H₀,"validators":[{"pub_key":"02ab…(66자)","voting_power":N}]}` (address·total_power 생략 가능).
- `SetContract` role 인자 = 64자 hex (위 부트스트랩 표).
- nullifier 등 chaincode-간 전용 []byte 파라미터(`eventAttrsRoot` 등)는 c2c raw 전달이라 현행 유지 — CLI 직접 호출 대상 아님.

---

## 2026-06-10 — LinkerNullifier 구현 (BTIP-24/33 정합)

> 구현 사이클 3번째 작업. 양 체인 LinkerNullifier를 개정 스펙에 정합. 부수로 btip-24 슈도코드의 rename 잔재(`targetDApp` 4곳)와 stale 캐스트(`bytes32(block.chainid)` → `block.chainid`, BTIP-37 uint256 개정 미반영)를 스펙 측에서 먼저 정정.

### on-bpun (Solidity) — BTIP-24

- **`IBTIP24.sol` 재작성**:
  - `markProcessed(bytes32 eventRootHash, address handlerDApp)` — **`returns (bool wasDup)` 제거, 중복 시 내부 `DuplicateProof` revert** (2026-04-21 wasDup 패턴 폐기).
  - **`error DuplicateProof(bytes32, address handlerDApp)` 정의가 IBTIP21 → IBTIP24로 이동** (발신 주체 원칙). IBTIP21에는 cross-ref 주석만.
  - `setRegistry(address)` 신설 (owner only). 파라미터 `targetDApp` → `handlerDApp`.
- **`LinkerNullifier.sol` 재작성**:
  - `constructor(endpointAddr)` immutable 패턴 폐기 → **무인자 생성자 + `setRegistry`**. `markProcessed`의 onlyLinkerEndpoint를 `IBTIP37(_registry).getContract(block.chainid, LINKER_ENDPOINT)` 동적 조회로.
  - Nullifier 계산(`sha256(abi.encode(eventRootHash, handlerDApp))`)은 내부 `_nullifier()` 단일화.
  - 에러 Err* 컨벤션: `ErrUnauthorized`/`ErrNotProcessed(bytes32)`/`ErrRegistryNotSet`/`ErrZeroAddress` (스펙 정의 에러는 `DuplicateProof`뿐 — 나머지는 구현 레벨).
  - `cancelNullifier`는 msg.sender = handlerDApp 의미 유지.
- **`LinkerEndpoint.sol` 최소 정합** (BTIP-21 전면 개정은 다음 작업): `markProcessed` 단순 호출(자동 전파)로 변경, wasDup 분기·`DuplicateProof` revert 제거.
- **`deploy.ts` 갱신**: LinkerRegistry 배포 + `registry.setSelfContract(LINKER_ENDPOINT, endpoint)` + `nullifier.setRegistry(registry)` 단계 추가, `LinkerNullifier()` 무인자. (§"LinkerRegistry 구현" 절의 "deploy.ts 미배포" 잔여 항목 해소.)
- **`LinkerRegistry.sol`에 `setSelfContract(role, addr)` 편의 함수 추가** — 스크립트가 EVM chainid를 외부에서 알 필요 없이 자기 체인 등록 (BEATOZ는 eth_call류 RPC가 제한적).
- **`utils.ts`**: `ErrNotProcessed(bytes32)`=`2906a727`, `ErrRegistryNotSet()`=`6bf41600`, `ErrZeroAddress()`=`ecc6fdf0` 추가, DuplicateProof 라벨 handlerDApp 갱신.
- **검증: solc 0.8.28 단독 컴파일 통과** (LinkerRegistry + LinkerNullifier + LinkerEndpoint 동시).

### on-bprn (Go) — BTIP-33

- **`types/ibtip33.go` 신설** — 스펙 인터페이스 정의 + `var _ types.IBTIP33` 컴파일 타임 검증 (기존엔 인터페이스 부재).
- `linker-nullifier/main.go`: **`SetRegistryID` → `SetRegistry`**(btip-33 스펙 — endpoint/verifier의 `SetRegistryID`(btip-29/31)와 이름이 다름에 주의, 공유 헬퍼 `types.SetRegistryID`는 유지), 파라미터 `chaincodeID` → `ccAppId`, `NullifierRecord` byte 필드 HexBytes(저장 JSON hex)·`chaincode_id` → `ccapp_id`.
- `types/nullifier.go`: `CalculateNullifier(eventAttrsRoot, ccAppId)` rename.
- `eventAttrsRoot []byte` 파라미터는 c2c raw 전달 유지 (hex 전환 원칙의 c2c 예외).
- MarkProcessed의 caller 검증(SignedProposal 헤더 기반 InvokerChaincodeID == registry의 LinkerEndpoint)·CancelNullifier(호출 체인코드 자신만) 로직은 기존 구현이 스펙과 일치 — 유지.

### 운영 주의

- 부트스트랩에서 **linker-nullifier만 `SetRegistry`**, linker-endpoint/verifier/dapp은 `SetRegistryID` (BTIP별 메소드명 차이).
- on-bpun 배포 순서 변경: endpoint → registry → nullifier(무인자) → … + registry 등록/연결 단계 (deploy.ts 헤더 주석 참조).
- 검증: on-bprn은 로컬 `go build ./...` 필요.

---

## 2026-06-10 — LinkerVerifier 구현 (BTIP-23/31 정합)

> 구현 사이클 4번째 작업. **LinkerPolicy/BPuN(정책 엔진 4종 + lib)은 미변경** (사용자 지시 유지). 부수로 스펙 정정 2건 + 페이로드 타입/필드명 rename 일괄 + 잠재 버그 2건 해소.

### 스펙 정정 (구현 전 선행)

- **btip-19/23 Step 5 리프 해시 정정**: 슈도코드가 `leaf_hash = sha256(tx_event_root_proof.leaf)`로 리프를 재해싱했으나, 실제 시스템(Solidity verifier `abi.decode(leaf,(bytes32))` 직접 사용 + prover `fromHashedLeaves` preHashed + 2026-04 localnet end-to-end 통과)은 **preHashed 리프 직접 사용**. 검증된 구현 동작 기준으로 양 스펙 슈도코드 정정 ("이미 32바이트 해시이므로 추가 해시 없이 직접 사용").
- btip-23 슈도코드 `bytes32(block.chainid)` → `block.chainid` (BTIP-37 uint256 정합, btip-24와 동일 잔재).
- btip-31/33/34의 `BPuNTxEventProofPayload` 표기 → **`BPuNTxEventProof`** (btip-29 개명 기준). btip-28(정의 문서)도 **사용자 확정으로 `BPuNTxEventProof`로 개명 완료** (2026-06-10, 7곳).

### on-bpun (Solidity) — BTIP-23

- **`IBTIP21.sol` 페이로드 rename**: struct `TxEventProof` → **`BPrNTxEventProof`**, 필드 `event_log_root_proof` → **`tx_event_root_proof`** (btip-19/21 개정 정합).
- **`IBTIP23.sol` 재작성**: `verifyProof(IBTIP21.BPrNTxEventProof)` + `setRegistry(address)` (`setPolicyContract` 폐기).
- **`LinkerVerifier.sol` 재작성**: 위임 구조는 기존과 동일(이미 Step 2~4를 정책 컨트랙트에 단일 호출 위임 중이었음) — 변경점은 ① `_policy` 저장 폐기 → `IBTIP37(_registry).getContract(block.chainid, LINKER_POLICY)` 동적 조회, ② Step 번호를 btip-19 기준(2~4/5/6)으로 재표기, ③ `eventLogRoot` 변수명 → `txEventRoot`, ④ 에러 `PolicyNotSet`/`ZeroAddress` → `ErrRegistryNotSet`/`ErrZeroAddress`/`ErrUnauthorized`(Err* 컨벤션, `BlockEventMerkleProofFailed`/`EventMerkleProofFailed`는 스펙 이름 유지).
- ⚠️ **IBTIP22 메소드명 불일치 관찰**: btip-22 스펙은 `verifyChannelEndorsementPolicy`, 코드 `IBTIP22.sol`/`LinkerPolicyVerifier.sol`은 `verifyChannelEndorsement`(+ 별도 `verifyChannelEndorsementPolicy` 오버로드 2개 존재). 정책 엔진 불가침이므로 verifier는 **현행 `verifyChannelEndorsement` 호출 유지** — 정책 스위트 정합 작업(ryan 영역)에서 해소 필요.
- **`LinkerEndpoint.sol`**: 페이로드 타입/필드 rename 반영 (BTIP-21 전면 개정은 여전히 다음 작업).
- **`deploy.ts`**: `verifier.setPolicyContract(policyVerifier)` → `registry.setSelfContract(LINKER_POLICY, policyVerifier)` + `verifier.setRegistry(registry)`. (LINKER_POLICY role에 **LinkerPolicyVerifier**(IBTIP22 구현체)를 등록 — 종전 wiring과 동일 대상.)
- **`submit-proof.ts`/`utils.ts`**: 필드 rename 반영, `PolicyNotSet` 셀렉터 제거.
- **prover-ts**: DTO `TxEventProofDto` → `BPrNTxEventProofDto`, 와이어 필드 `event_log_root_proof` → `tx_event_root_proof` (controller/service/test-proof 일괄).
- **검증**: solc 0.8.28 5개 컨트랙트 동시 컴파일 OK, prover-ts `tsc --noEmit` OK.

### on-bprn (Go) — BTIP-31

- 구조는 기 정합(2026-05-26 레지스트리 리팩토링) — 추가 작업:
  - **`types/ibtip31.go` 신설** + `var _ types.IBTIP31` assert (기존엔 인터페이스 부재).
  - `VerifyProof(ctx, payloadJSON string)` → **`VerifyProof(ctx, *types.BPuNTxEventProof)`** — contractapi가 JSON 인자를 구조체로 unmarshal하므로 수동 unmarshal 제거. linker-endpoint의 invokeVerifier(payload JSON 인자 전달)와 와이어 동일.
  - `SetRegistryID(ctx, ccId)` 파라미터명 정합.
  - Go 전역 타입 rename: `BPuNTxEventProofPayload` → **`BPuNTxEventProof`** (types.go/linker-verifier/linker-endpoint).

### 잠재 버그 해소 (contractapi raw-passthrough 후속)

- **linker-endpoint → linker-nullifier 인자 버그**: `IsProcessed`/`MarkProcessed` 호출 시 `base64.StdEncoding.EncodeToString(eventAttrsRoot)`를 인자로 전달 — raw passthrough로 nullifier가 base64 문자열 44바이트를 받아 32바이트 검증에 걸리는 잠재 버그(u2r 미구현으로 미발현). **raw bytes 직접 전달로 수정.**
- **dapp-example → linker-nullifier `CancelNullifier`**: 동일 패턴 수정 (base64 import 제거).

### 검증 상태

- on-bprn: 로컬 `go build ./...` 필요 (샌드박스 Go 부재).

