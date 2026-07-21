# Linker V2 — 작업 핸드오프 (2026-07-21)

> task-contexts 자기완결 기록. `linker-v2.md` §00000000(2026-07-21 핸드오프)에서 참조.
> 대상 저장소 브랜치: `fix/bg-613` (verifier/on-bpun·on-bprn, prover-ts, test + bprn-organizations 프로파일).

## 0. 프로젝트 개요

BEATOZ **Linker V2** = BPuN(EVM/Solidity, `verifier/on-bpun`) ↔ BPrN(Hyperledger Fabric/Go 체인코드, `verifier/on-bprn`) 크로스체인 2PC 프로토콜.
- `prover-ts` : TypeScript prover 계층(btip19 r2u-txevt, btip28 u2r-txevt, btip39 u2r-policy 등).
- `test` : Vitest 통합테스트 (BPuN+BPrN e2e, 시나리오 01~07).
- 방향: **u2r**(BPuN 발) / **r2u**(BPrN 발).

**표준 실행 순서**: 배포 → sync → bootstrap → provers → test.

핵심 비대칭:
- BPuN이 아는 BPrN endpoint 주소는 `sha256(channel + "-" + cc)[12:]` 로 **결정적** → 계산으로 획득(sync 불필요).
- BPrN이 아는 BPuN endpoint 주소는 원시 EVM 주소라 **재배포마다 변함** → 배포 아티팩트에서 캡처해야 함(sync가 담당).

---

## 1. 오늘 변경 요약 (테마별)

### A. register-bpun-endpoint 제거 + bootstrap 멱등화
- `verifier/on-bprn/scripts/register-bpun-endpoint.ts` 제거(→ `_to_delete/`), npm 스크립트 제거.
- 표준 순서(배포→sync→bootstrap)를 강제하면 재등록 단계가 불필요.

### B. BPuN 쪽 `config.bpn.json` / `config.devnet.json` 제거 → genesis-policy.hex + 고정 상수
- BPuN bootstrap이 BPrN 정책을 얻던 `on-bpun/scripts/config/config.<bprn>.json` 제거(→ `_to_delete/`).
- `genesisPolicy` : `bprn-organizations/<bprnChannel>/genesis-policy.hex` 에서 읽음(devnet과 동일 포맷, 0x접두사·개행없음). `bprn-organizations/bpn/genesis-policy.hex` 생성.
- `paymentChannel` = CLI의 bprn 채널명. `paymentChaincode` = `BTIP34CCApp`(고정). `endpointCc` = `LinkerEndpointCC`(고정).
- devnet도 동일하게 hex(`bprn-organizations/devnet/genesis-policy.hex`, 커밋됨) + 고정상수 사용.
- 주의: `bprn-organizations/.gitignore`가 devnet 외 전부 무시 → `bprn-organizations/bpn/genesis-policy.hex`는 **로컬(비추적)**. BPrN을 새 크립토로 올리면 이 파일도 새로 생성 필요.

### C. reconcile-or-fail 멱등 가드 (blind skip 금지)
"했으면 SKIP"이 아니라 "**의도한 상태와 일치하면 SKIP, 다르면 크게 실패**".
- **BPuN `bootstrap.ts` `initPolicy`**: 온체인 `channelName()` 읽음. 빈값→seed / ==대상채널→SKIP / 다름→FAIL(정책 재초기화 불가, LinkerPolicy 재배포 필요). initPolicy는 일회성(`_initialized` 가드)이라 이렇게 처리. (인증서 getter가 없어 채널 정체성만 비교 — 한계는 표준흐름상 재배포로 fresh policy가 되므로 실질 문제없음.)
- **BPrN `bootstrap.ts` `SetValidatorSet`**: `GetValidatorSet(height)`로 되읽어 (pub_key, voting_power) 정규화 멀티셋 비교. 없음→run / 일치→SKIP / 다름→FAIL. (기존 `GetLatestHeight>0` blind skip 대체.)
- BPrN Mint는 `BalanceOf(mint.to)>0` skip 유지(저위험).

### D. `redeploy-token.ts` 제거
- 멱등 bootstrap 덕에 토큰 교체는 `deploy --only mocks/BTIP26Token` → `bootstrap` 으로 대체. 스크립트 제거(→ `_to_delete/`), npm 스크립트·README 정리.

### E. BPuN scripts 디렉토리 구조 정리
- `verifier/on-bpun/scripts/.net` → `verifier/on-bpun/scripts/config` (rename).
- `verifier/on-bpun/scripts/beatoz/*` → `verifier/on-bpun/scripts/` (한 단계 위로).
- 참조 갱신: `utils.ts`(config 경로), `sync-addresses.ts`(HERE/ONBPUN 깊이·NET), `setup.sh`(PROJECT_ROOT), `package.json`(스크립트 타깃), 각 스크립트 Usage 문자열.
- `.gitignore`: `**/deployed.*`, `**/config.*`(단 `!config.devnet0.json`, `!config.testnet0.json`) — basename 글롭이라 rename 무관.

### F. BPrN counterpart 데이터 모델 (정적 config ↔ 동적 counterpart 분리)
- `verifier/on-bprn/scripts/bootstrap.<ch>.json` → `verifier/on-bprn/scripts/config/config.<ch>.json` (정적 토폴로지: channel/chaincodes/roles/btip34). `config/` 디렉토리 신설.
- 동적 counterpart는 `config/bootstrap.<bpunAlias>.json` (sync 생성, **gitignore**) : `{ bpunChainAlias, chainId, linkerEndpoint, validatorSet }`.
- BPrN `bootstrap.ts` CLI: `npm run bootstrap -- <bprnChannel> <bpunChainAlias>`.
  - `<bprnChannel>` → `config/config.<ch>.json` + `bprn-organizations/<ch>/`.
  - `<bpunChainAlias>` → `config/bootstrap.<bpunAlias>.json`(counterpart).
- `deploy.sh`는 chaincode 이름을 `config/config.<network>.json` 에서 읽음.
- `.gitignore`(`verifier/on-bprn/scripts/.gitignore`): `config/bootstrap.*.json`(동적) 무시, `config/config.*.json` 무시하되 `!config/config.<channel-name>.json`(템플릿)·`!config/config.devnet.json`(레퍼런스) 추적. → 환경별 `config.bpn.json`은 비추적(BPuN 정책과 대칭). `git rm --cached config.bpn.json` 수행.

### G. 신원(identity) 소스 완전 통일 → connection-profile `clients[0]`
이전엔 신원(cert/key/mspId/userName)을 세 소비자가 제각각(bootstrap=flat파일, prover=env, test=env)에서 읽고, 프로파일의 `clients[]`는 아무도 안 읽는 죽은 데이터였음. 이제 셋 다 프로파일 `clients[0]`에서 읽는다.
- `signedCertPath`/`privateKeyPath`는 프로파일 디렉토리 기준 상대해석, `mspId`, `id`(=userName).
- **on-bprn `bootstrap.ts`**: `DEFAULT_IDENTITY`/`Identity`/`cfg.identity` 제거.
- **prover `common/fabric.ts`+`env.ts`**: `FabricConnectionConfig`/`loadFabricEnv`에서 mspId/userName/certPath/keyPath 제거. `FABRIC_MSP_ID`/`FABRIC_USER_NAME`/`FABRIC_CERT_PATH`/`FABRIC_KEY_PATH` env 폐지.
- **test `config.ts`**: 같은 4필드 제거(test는 prover `FabricClient` 재사용).
- **`bprn-organizations/bpn/connection-profile.json`**: `clients[0]` 추가 `{organization, mspId:"Org1MSP", id:"User1@org1.bc", privateKeyPath:"user1-key", signedCertPath:"user1-cert.pem"}`. devnet 프로파일은 이미 `clients[0]` 보유(crypto-config deep 경로).
- env 정리: `prover-ts/.env`, `test/.env.bprn.bpn`, `prover-ts/.env.example`, btip19/btip28 README에서 신원 4줄 제거.

### H. validatorSet 이전 (BPrN config → bpun-alias counterpart, sync가 조회)
- validatorSet은 BPuN 체인의 genesis 검증자셋 → BPrN 채널이 아니라 **BPuN 체인(bpun alias)에 종속**. bpn(1개)과 devnet(3개)이 서로 다른 값인 것도 방증.
- config.bpn/devnet/template.json 에서 `validatorSet` 제거.
- **`sync-addresses.ts`**: BPuN Tendermint RPC `providerUrl + /validators?height=1` 조회 → pub_key(base64→hex)·voting_power 변환(address 생략, 체인코드가 파생) → `config/bootstrap.<bpunAlias>.json`에 `validatorSet` 포함 기록.
- **BPrN `bootstrap.ts`**: `validatorSet`을 counterpart 파일에서 읽어 `SetValidatorSet` 시드.

### I. 잡정리
- BPuN `deploy.ts`: FakeEmitter 배포 로그의 `(test fixture — set FAKE_EMITTER=...)` 안내 제거(sync가 자동 기록하므로 불필요).

---

## 2. 현재 표준 실행 순서 (명령)

```
# BPrN 체인코드 배포 (네트워크+채널 준비 전제)
cd verifier/on-bprn/scripts && npm run deploy -- bpn

# BPuN 컨트랙트 배포 (FakeEmitter·BTIP26Token 포함)
cd verifier/on-bpun && npm run deploy -- localnet0

# BPuN 주소/검증자셋 동기화 (BPuN 체인이 떠 있어야 함 — /validators 조회)
#   test/.env, prover-ts/.env, on-bprn config/bootstrap.localnet0.json 갱신
npm run sync-addresses -- localnet0

# BPuN 부트스트랩
npm run bootstrap -- localnet0 bpn

# BPrN 부트스트랩 (멱등; counterpart의 endpoint+validatorSet 사용)
cd verifier/on-bprn/scripts && npm run bootstrap -- bpn localnet0

# provers 기동: btip19 / btip28 / btip39
# 테스트
cd test && npm run test -- bpn localnet0
```

주의:
- `sync-addresses`는 인자 1개(`<bpunAlias>`)로 축소됨(예전 `localnet0 bpn` → `localnet0`).
- BPrN bootstrap은 인자 2개(`<bprnChannel> <bpunChainAlias>`).
- 이번 리팩터링 후 **sync를 다시 돌려야** `config/bootstrap.<bpunAlias>.json`에 `validatorSet`이 채워짐(기존 파일엔 없음).

---

## 3. 신원 모델 (단일 출처 = connection-profile clients[0])

`bprn-organizations/<network>/connection-profile.json`:
```json
"clients": [
  { "organization": "...", "mspId": "Org1MSP", "id": "User1@org1.bc",
    "privateKeyPath": "<프로파일 디렉토리 기준 상대 or 절대>",
    "signedCertPath": "<...>" }
]
```
- bootstrap.ts / prover common/fabric.ts 둘 다 `profile.clients[0]`에서 cert/key/mspId/userName 획득, 경로는 프로파일 디렉토리 기준.
- test 하네스는 prover `FabricClient`를 재사용 → 자동으로 동일 경로.
- 유지되는 env: `FABRIC_CONNECTION_PROFILE`, `FABRIC_CHANNEL_ID`, `FABRIC_COMMIT_SIG_METADATA_INDEX`.

---

## 4. counterpart / 정적 config 데이터 모델 (BPrN)

`verifier/on-bprn/scripts/config/`:
- `config.<bprnChannel>.json` — 정적 토폴로지(channel, chaincodes, roles, btip34). **counterpartBPuN·validatorSet 없음**. 추적(단, config.bpn.json은 비추적, devnet·템플릿은 추적).
- `bootstrap.<bpunChainAlias>.json` — 동적 counterpart(`chainId`, `linkerEndpoint`, `validatorSet`). sync 생성, gitignore.

BPuN 쪽 `verifier/on-bpun/scripts/config/`:
- `config.<bpunAlias>.json` — providerUrl/chainId/deployerPrvKey (비추적; devnet0/testnet0만 레퍼런스 추적).
- `deployed.<bpunAlias>.<Contract>.json` — 배포 산출물(비추적). `LinkerEndpoint`엔 chainId+contract 둘 다 있음(counterpart 소스).

---

## 5. 남은 작업 / 향후 과제

### BPrN devnet 지원 (요청: **bootstrap부터, deploy는 나중에**)
- **bootstrap**: 신원(profile clients[0])·validatorSet(counterpart) 측면에서 devnet 준비 완료. devnet Fabric 네트워크가 떠 있고, devnet의 BPuN 상대에 대해 sync가 counterpart를 만들었다면 동작 가능.
- **deploy (미완, 갭)**: devnet은 커밋된 `crypto-config/` deep 레이아웃이라 `deploy.sh`가 기대하는 flat 항목이 없음:
  - `peer`(바이너리) 없음, `fabric-config/`(core.yaml) 없음, `admin-msp/` 없음(devnet crypto엔 User1만, Admin 없음) ← lifecycle approve/commit 블로커.
  - `peer-tls-ca.crt`/`orderer-tls-ca.pem`은 deep 경로엔 있으나 flat 이름 없음.
  - 토폴로지: devnet은 3-org(`endorsementPolicyCount:2`), deploy.sh는 단일-org 가정 → 다중-org approve/commit 필요.
  - 참고: bootstrap.ts 헤더에 "deploy is delegated to bpn-core-2.2/scripts/run" — 원래 배포는 외부 툴링.

### 수동 정리 (device_bash는 rm/rmdir 불가 → Mac에서 직접)
- `verifier/on-bpun/scripts/config/_to_delete/` (구 config.bpn.json, config.devnet.json)
- `verifier/on-bpun/scripts/beatoz/` (빈 디렉토리)
- `verifier/on-bprn/scripts/_to_delete/` (구 bootstrap.devnet.json, bootstrap.<channel-name>.json, register-bpun-endpoint.ts 등)
- (git) `.gitignore` 스테이징 후 커밋: config.bpn.json 추적 해제 + 신설 규칙들.

---

## 6. 환경/도구 메모 (작업 시 유의)

- 저장소는 **사용자 Mac에 마운트**(device_bash로 접근). `filesystem__*` MCP는 스테일 캐시 → device_bash가 ground truth.
- device_bash(클라우드 Linux VM, FUSE 마운트): `node`/`python3` 실행 가능. **`tsx`/`esbuild`/`peer`는 실행 불가**(macOS 네이티브 바이너리). **`rm`/`rmdir`/`unlink` 불가**(mv는 가능). `git status`류가 `.git/index.lock`을 못 지워 스테일 락 남길 수 있음 → 발생 시 락을 mv로 옆으로 치우거나 Mac에서 `rm -f .git/index.lock`.
- 트랜스파일 체크: `node -e "require('typescript').transpileModule(...)"` (per-file 구문/기본 타입만; 크로스파일 타입은 grep로 보완).
- macOS 기본 `/bin/bash`는 3.2(연관배열 `declare -A` 없음) — deploy.sh는 병렬 인덱스 배열 사용.

## 7. 커스텀 에러 셀렉터 (참고)
- `OrgPolicyNotFound(string)` = `0x87efec19`
- `ErrUnauthorizedCaller(address)` = `0x783d6f44`
- `ErrUntrustedSource` (BPrN) = `0x2a534f1f`
- `Error(string)` = `0x08c379a0`

## 8. 사용자 규칙
- 한국어 존댓말(반말 금지).
- **모든 파일 편집은 명시적 확인 후**.
