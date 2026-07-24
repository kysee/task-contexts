# Linker V2 — 설정 모델 재편: sync-conf(push) → autoconf(소비자 pull) + 테스트 자기완결 (2026-07-22~24)

> task-contexts 자기완결 기록. `linker-v2.md` §0000000000(2026-07-24 핸드오프)에서 참조.  
> 전제 문서: `readme-tooling-cleanup-2026-07-22.md`(README 정리·deploy.ts 전환·BPRN_* 리네임) — 본 문서는 그 직후부터의 기록이며 겹치는 배경은 재서술하지 않는다.  
> 대상 저장소: linker-v2 (verifier/on-bpun·on-bprn scripts, prover-ts, test).  
> **전부 커밋 완료** — 커밋 매핑은 §9. 작업 트리 클린(비추적 `.claude/` 제외).

## 0. 세션 본류

1. **sync-conf 모델 구축**(생산자 push): on-bpun `sync-addresses` → `sync-conf` 확장 + on-bprn `sync-conf` 신설, env 파일 이원화(`.env.<bprnChannelName>` / `.env.<bpunChainAlias>`), prover 인자화.
2. **타 시스템 e2e 검증**에서 문제 3건 발견·해소 → **테스트 완전 자기완결화**(bootstrap Mint 폐지, 시나리오 1·2 자체 시드, TEST_PAYER 임의 주소).
3. **설계 논의 후 pull 재편**: push 는 생산자가 소비자의 경로·스키마에 결합(대칭 문제) + genesis validatorSet 은 BPuN RPC 공용 인터페이스라 소비자가 직접 조회 가능 → **소비자 3곳이 각자 `autoconf` 명령으로 필요한 값을 가져오는 모델로 확정**(사용자 지시: "명령도 sync-conf 가 아니라 autoconf 로"). push 형 sync-conf.ts 2종 폐지.
4. `test/autoconf.mjs` 로 만들었다가 **사용자 지적으로 `test/autoconf.ts` 재작성**(.ts 통일).

## 1. 최종 모델: autoconf (소비자 pull)

- **원칙**: 생산자(on-bpun·on-bprn 배포 스크립트)는 자기 산출물만 남긴다. 소비 패키지가 `autoconf` 로 생산자 원시 산출물을 직접 읽어 **자기 소유 파일**을 생성/갱신한다.
- **계약 표면** = 생산자 원시 산출물 3종:  
  `verifier/on-bpun/scripts/config/config.<bpunChainAlias>.json`(providerUrl/chainId/deployerPrvKey),  
  `verifier/on-bpun/scripts/config/deployed.<bpunChainAlias>.<contract>.json`(contract 주소, LinkerEndpoint 기록은 chainId 포함),  
  `verifier/on-bprn/scripts/config/config.<bprnChannelName>.json`(chaincodes 이름) + `bprn-organizations/<bprnChannelName>/connection-profile.json`.
- **표준 실행 순서** (루트 README 반영됨):

  | 순서 | 위치 | 명령 |
  |---|---|---|
  | 1 | `verifier/on-bpun` | `npm run deploy -- <bpunChainAlias>` → `npm run bootstrap -- <bpunChainAlias> <bprnChannelName>` |
  | 2 | `verifier/on-bprn/scripts` | `npm run deploy -- <bprnChannelName>` → `npm run autoconf -- <bpunChainAlias>` → `npm run bootstrap -- <bprnChannelName> <bpunChainAlias>` |
  | 3 | `prover-ts` | `npm run autoconf -- <bprnChannelName> <bpunChainAlias>` → prover 4종 기동 |
  | 4 | `test` | `npm run autoconf -- <bprnChannelName> <bpunChainAlias>` → `npm run test -- <bprnChannelName> <bpunChainAlias>` |

- **재배포 시**: 산출물을 읽는 소비 패키지의 autoconf 만 재실행(BPuN 재배포 → 3곳 전부, BPrN config 변경 → prover-ts·test). 구동 중 prover 는 재시작해야 반영.
- 폐지: `verifier/on-bpun/scripts/sync-conf.ts`(155줄)·`verifier/on-bprn/scripts/sync-conf.ts`(92줄) 삭제, on-bpun package.json 에서 스크립트 제거. counterpart 생성 책임이 on-bpun → on-bprn 으로 이동했으므로 on-bpun 은 이제 배포 산출물 기록 외에 아무 파일도 쓰지 않는다.

## 2. autoconf 명령 3개 상세

### 2.1 `verifier/on-bprn/scripts/autoconf.ts` — `npm run autoconf -- <bpunChainAlias>`

counterpart 파일이 **on-bprn 내부 산출물**이 됐다(구 모델에선 on-bpun sync 가 기록).

| 읽기 | 값 |
|---|---|
| on-bpun `config.<bpunChainAlias>.json` | `providerUrl` |
| on-bpun `deployed.<bpunChainAlias>.LinkerEndpoint.json` | `chainId` · `contract` |
| BPuN RPC `GET /validators?height=1&per_page=100` | genesis validator set (pub_key base64→hex, voting_power) |

출력: `config/bootstrap.<bpunChainAlias>.json` **전체 재작성**(upsert 아님) —  
`{ bpunChainAlias, chainId, linkerEndpoint, validatorSet: { height: 1, validators: [{ pub_key(hex), voting_power }] } }`.  
gitignore 대상(실파일), 꺾쇠 파일명 템플릿만 추적. BPuN 체인이 떠 있어야 한다(RPC 조회). CJS 스타일(`__dirname`) + `async main()`, tsx 로 실행.

### 2.2 `prover-ts/src/autoconf.ts` — `npm run autoconf -- <bprnChannelName> <bpunChainAlias>`

| 기록 파일 | 키 | 값 원본 | 갱신 |
|---|---|---|---|
| `.env.<bprnChannelName>` | `BPRN_CONNECTION_PROFILE` | `bprn-organizations/<ch>/connection-profile.json` 절대경로(존재 검사) | 항상 |
| 〃 | `BPRN_CONFIG_DIR_PATH` | `bprn-organizations/<ch>` 절대경로 | 항상 |
| 〃 | `LINKER_REGISTRY_CC` | on-bprn config `chaincodes.registry` | 항상 |
| 〃 | `BPRN_COMMIT_SIG_METADATA_INDEX` | `5` | 없을 때만 |
| `.env.<bpunChainAlias>` | `BPUN_RPC_URL` / `BPUN_CHAIN_ID` / `BPUN_PRIVATE_KEY` | on-bpun config `providerUrl` / `chainId` / `deployerPrvKey` | 항상 |
| 〃 | `BPUN_LINKER_REGISTRY` / `BPUN_LINKER_POLICY_ADDRESS` | `deployed.<alias>.LinkerRegistry.json` / `deployed.<alias>.LinkerPolicy.json` 의 `contract` | 항상 |
| 〃 | `BPUN_NETWORK_TYPE` | `<bpunChainAlias>` 그대로 | 항상 |
| 〃 | `BPUN_SUBMIT_GAS` | `5000000` | 없을 때만 |
| 〃 | `BPUN_CONFIG_FILE_PATH` | `./src/btip18-r2u-policy/config/beatoz.network.json` | 없을 때만 |
| `src/btip18-r2u-policy/config/beatoz.network.json` | `beatozNetworks.<bpunChainAlias>` | `{ chainId, providerUrl }` upsert(타 alias 보존) | 항상 |
| 〃 | `contractJsonDir` | `verifier/on-bpun/artifacts/contracts` 절대경로 | 항상 |

beatoz.network.json 은 BeatozProvider 가 파일 경로를 요구하는 btip18 전용 설정 — prover-ts 자기 파일이 되면서 on-bpun 과의 결합이 끊겼다(구 모델에선 on-bpun sync-conf 가 기록). **단, git 추적 파일에 머신별 절대경로가 들어가 diff 가 남는 문제 미결(§8)**.

### 2.3 `test/autoconf.ts` — `npm run autoconf -- <bprnChannelName> <bpunChainAlias>`

| 기록 파일 | 키 | 값 원본 | 갱신 |
|---|---|---|---|
| `.env.<bprnChannelName>` | `BPRN_CONNECTION_PROFILE` | connection-profile.json 절대경로 | 항상 |
| 〃 | `LINKER_REGISTRY_CC` / `STC_CC` | on-bprn config `chaincodes.registry` / `chaincodes.btip34` | 항상 |
| 〃 | `BPRN_COMMIT_SIG_METADATA_INDEX` | `5` | 없을 때만 |
| 〃 | `BTIP19_API_URL` | `http://127.0.0.1:3019` | 없을 때만 |
| 〃 | `TEST_PAYER` / `TEST_BENEFICIARY` | `0x000000000000000000000000000000000000a001` (40 hex) | 없을 때만 |
| `.env.<bpunChainAlias>` | `BPUN_RPC_URL` / `BPUN_CHAIN_ID` / `BPUN_PRIVATE_KEY` | on-bpun config | 항상 |
| 〃 | `BPUN_LINKER_REGISTRY` / `BPUN_BTIP26_TOKEN` / `FAKE_EMITTER` | deployed 기록(FakeEmitter 는 없으면 생략) | 항상 |
| 〃 | `BPUN_SUBMIT_GAS` | `5000000` | 없을 때만 |
| 〃 | `BTIP28_API_URL` | `http://127.0.0.1:3028` | 없을 때만 |

**실행 방식**: `node autoconf.ts` — test 패키지에 tsx 가 없어 처음 `autoconf.mjs`(plain ESM)로 만들었으나 사용자 지적으로 `.ts` 재작성.  
Node 22.18+ 는 타입 스트리핑이 기본이라 `node` 가 `.ts` 를 직접 실행한다(추가 도구 불요, 디바이스 Node v22.22.3 확인).  
파일은 CJS 스타일(`require`/`__dirname`) — import 구문을 쓰면 Node 모듈 감지가 ESM 으로 판정해 `__dirname` 이 없고, test/tsconfig(module CommonJS)와도 어긋나므로 **CJS 로 통일**(erasable syntax 만 사용, enum/namespace 금지). 구 autoconf.mjs 는 삭제됨.

## 3. env 파일 체계 + prover 인자화 (sync-conf 단계에서 확립, autoconf 로 계승)

- **파일명**: 접두사 없는 `test/.env.<bprnChannelName>`·`test/.env.<bpunChainAlias>`, `prover-ts/` 동일(사용자 확정). 실파일은 gitignore, **꺾쇠 문자 그대로의 파일명**(`.env.<channelName>`, `.env.<chainAlias>`)이 템플릿으로 추적된다(4종: test·prover-ts × 2). 템플릿 헤더에 "GENERATED/UPDATED by this package's `npm run autoconf`" 명시.
- **`BPRN_CHANNEL_ID` 는 env 파일에 없다**: 채널명은 위치 인자가 정본(디렉토리·파일명 규칙) — prover 로더가 인자에서 주입.
- **prover 인자화**: 모든 prover 엔트리포인트가 선행 위치 인자 `<bprnChannelName> <bpunChainAlias>` 를 받는다. `prover-ts/src/common/env-files.ts` `loadEnvFiles(usageCmd)` — argv[2]·argv[3] 로 env 파일 2개를 dotenv `parse` 로 읽고(**이미 설정된 환경변수가 파일 값보다 우선**), `process.env.BPRN_CHANNEL_ID = channelName` 주입 후 `argv.splice(2,2)` 로 제거(하위 파서는 나머지 인자만 봄).
- **test 도 동일**: `test/run.mjs` 가 인자 2개 필수, `TEST_BPRN_ALIAS`/`TEST_BPUN_ALIAS` env 로 vitest 에 전달. `test/config.ts` 는 `channelId = bprnAlias`(인자), `accounts.payer = req(bprnEnv, 'TEST_PAYER')`.
- **upsertEnv 규칙**(prover-ts·test 공통 구현): "항상" 키는 덮어쓰기, "없을 때만" 키는 부재 시에만 기본값(운영자가 바꾼 값 보존 — 포트 재정의 등), 관리 외 라인 그대로 보존, **끝 개행 보장**(§5 버그 수정분).

## 4. 테스트 완전 자기완결 (bootstrap Mint 폐지)

**발견 경위**: 타 시스템 첫 e2e 에서 시나리오 1·2 만 insufficient balance 실패, 2회차는 통과.  
**원인**: 시나리오 1·2 는 잔액을 자체 시드하지 않고 bootstrap 의 초기 Mint(`btip34.mint` config, `to` = 하드코딩 `0x3FD3…`)에 의존 — 그 머신의 TEST_PAYER(당시 sync-conf 가 BPuN deployer 주소를 기록)와 불일치. 2회차 통과는 시나리오 3·5 가 만든 지급 잔액 덕(우연). BPuN deployer ≠ BPrN payer(사용자 지적 — payer 는 BPrN 도메인).

**수정 3종**:

1. 시나리오 1(`01-r2u-accepted.test.ts`)·2(`02-r2u-rejected.test.ts`)에 pre-state 확인 전 `await H.stcMint(bprn, P, amount);` 자체 시드 삽입(3–6과 대칭). 이제 모든 시나리오가 자기완결.
2. **bootstrap 초기 Mint 단계 폐지**(`verifier/on-bprn/scripts/bootstrap.ts` 는 SetRegistryID/SetSelfCcName 등 와이어링만) + config 3종(`config.bpn.json`·`config.devnet.json`·`config.<bprnChannelName>.json` 템플릿)에서 `btip34.mint` 블록 삭제. **안전 근거 — btip34 admin TOFU**: BTIP34CCApp 의 admin 은 최초 관리자전용함수(SetSelfCcName/Mint/CancelLinkerEvent) 호출 신원으로 등록되는데, bootstrap 의 SetSelfCcName 이 여전히 최초 호출이므로 admin = 프로파일 `clients[0]` 유지. 테스트의 stcMint 도 같은 신원으로 호출되므로 성립.
3. `TEST_PAYER`/`TEST_BENEFICIARY` 는 임의 주소면 충분(BPuN 증명 제출자는 `BPUN_PRIVATE_KEY` 로 별개) — autoconf 가 기본값 `0x…a001` 을 "없을 때만" 기록, `test/config.ts` 의 `0x3FD3…` 하드코딩 제거.

## 5. 트러블슈팅 이력 (재발 시 참조)

| 증상 | 원인 | 해소 |
|---|---|---|
| 전 시나리오 `expected a 20-byte hex address, got 19 bytes` | 구현 중 TEST_PAYER 기본값을 `0x`+38 hex 로 잘못 작성 | 상수·템플릿·실파일(`test/.env.bpn`) 3곳 40 hex 로 수정. "없을 때만" 정책이라 이미 생성된 실파일은 직접 수정해야 했음 |
| btip18 `EADDRINUSE :5000` | macOS AirPlay Receiver(ControlCenter)가 5000 선점 | `prover-ts/.env.localnet0` 에 `BTIP18_R2U_POLICY_PORT=3018` 재정의(“없을 때만” 키가 아니라 운영자 추가 라인 — autoconf 가 보존). 코드 기본값 5000 변경 여부는 미결(§8) |
| env append 가 마지막 라인에 이어붙음 | upsertEnv 가 끝 개행을 보장하지 않았음 | 끝 개행 보장 패치(커밋 6608fa1) + 기존 파일 수정 |
| btip39 `no validator set registered` | on-bprn bootstrap 을 `-- bpn` 만으로 실행(bpunChainAlias 누락) → counterpart 미적용 | `-- bpn localnet0` 재실행으로 해소(사용자 발견). 인자 누락 시 경고 강화는 미결(§8) |
| Cowork 스테이징 파일이 구버전으로 보임 | device_stage_files 스냅샷 캐시 스테일 | **디바이스 파일이 정본** — 검증·편집 전부 device_bash 로 직접 수행(이 레포 작업의 기본 방법론) |

## 6. README / 문서 반영 (커밋 완료)

- 루트 `README.md`: 표준 실행 순서를 §1 표의 4블록으로, 전제조건에 "on-bprn 의 `autoconf` 가 BPuN RPC `/validators` 를 조회하므로 BPuN 가동 필요", 순서 근거("각 패키지의 autoconf 가 on-bpun 배포 산출물을 읽으므로 on-bpun 먼저").
- `verifier/on-bpun/README.md`·`scripts/README.md`: sync-addresses/sync-conf 절 삭제, NOTE → "배포 산출물은 세 패키지의 autoconf 가 읽어 간다", `--only` TIP 의 재배포 절차 → 소비 패키지 autoconf 재실행 안내.
- `verifier/on-bprn/README.md`·`scripts/README.md`: `## autoconf` 절 신설(원본 3종 + 출력 필드 표), bootstrap 절의 counterpart 서술을 autoconf 기준으로.
- `prover-ts/README.md`: `## autoconf` 절(기록 파일|키|값 원본|갱신 표 — §2.2 와 동일 내용), 실행 블록에 autoconf 선행 단계 추가, 인자 2개 규약 서술.
- `test/README.md`: 실행 블록에 autoconf, env 파일 생성 주체 문단.
- `prover-ts/src/btip19-r2u-txevt/README.md`·`btip28-u2r-txevt/README.md`: 변수 표의 생성 주체를 `autoconf` 로.
- 템플릿 4종 헤더 갱신, prover 채널 템플릿에서 `STC_CC` 제거(test 전용 키).
- 문서 규칙(이 세션에서 확정·kysee 메모리에 저장): 부재 사후 해명 금지, "소비처" 같은 경제 용어 금지, 모듈은 `test/`·`prover-ts/` 처럼 정확 표기, "만"·"두 파일" 같은 구어체·모호 표현 금지, 값의 "원본"과 "경로 기록"을 구분해 서술.

## 7. 검증 상태

- **정적**: on-bprn scripts·prover-ts 전체 `node node_modules/typescript/lib/tsc.js --noEmit`, test `tsc -p . --noEmit` — 전부 통과(디바이스 실제 node_modules 로).
- **test/autoconf.ts 실행 검증**: 인자 누락 usage(exit 1) + `bpn localnet0` 실제 실행으로 `.env.bpn`·`.env.localnet0` 갱신 동작 확인. 단 Cowork VM 마운트에서 실행해 절대경로가 VM 기준으로 기록됐으므로 **실행 전 상태로 복원해 둠** — Mac 에서 재실행하면 `/Users/kylekwon/…` 로 정상 기록.
- **미실시**: autoconf 3곳의 Mac 라이브 실행, autoconf 재편 이후 e2e. sync-conf 시절 e2e 는 타 시스템에서 수정(§4·§5) 반영 후 btip39 포함 prover 기동까지 확인됐으나, **클린 체인에서 mint 폐지 상태의 전체 e2e 1회가 남아 있다**.
- 라이브 검증 절차: on-bprn `autoconf` → prover-ts `autoconf` → test `autoconf` → prover 4종 재기동 → `npm run test -- <bprnChannelName> <bpunChainAlias>`.

## 8. 남은 작업 / 미결 결정

- **라이브**: §7 의 클린 체인 e2e 1회.
- **미결 결정(사용자 응답 대기)**:
  - `beatoz.network.json` 이 추적 파일인데 autoconf 가 머신별 절대경로(`contractJsonDir`)를 기록해 diff 발생 — 비추적화 + 템플릿 추적 전환 권장안 제시 상태.
  - btip18 코드 기본 포트 5000 → 3018 변경 여부(현재는 `.env.localnet0` 재정의로만 해소).
  - on-bprn scripts 에 수동 `mint` 명령 추가 여부(bootstrap Mint 폐지 후 운영 편의).
  - on-bprn bootstrap `<bpunChainAlias>` 인자 누락 시 경고 강화 여부(§5 btip39 건).
  - bpn `connection-profile.json`·`config.bpn.json` 추적 전환 여부(7-22 이월 — 소실 재발 위험).
- **소소한 통일**: usage 문자열 `bprnChannelAlias`/`<bprnChannel>` → `bprnChannelName`(on-bpun `bootstrap.ts`·`setup.sh`, test `run.mjs`) — 7-22 이월.
- 정리 완료 확인: push 형 sync-conf.ts 2종·구 `prover-ts/.env`·`_to_delete/` 폴더들 전부 삭제됨, git 커밋 완료.

## 9. 커밋 매핑 (7-22 핸드오프 이후, 오래된 것부터)

| 커밋 | 내용 |
|---|---|
| `9bb3829` | deploy.ts 전환(7-22 문서 §1.A) |
| `7d796d9` | FABRIC_* → BPRN_* 리네임(7-22 문서 §1.C) |
| `0342bfb` | .gitignore 재편(on-bprn scripts/.gitignore 제거 → on-bprn/.gitignore 통합, env 템플릿 예외) |
| `224bfbb` | README 정리(7-22 문서 §2) |
| `6165850` | sync-conf 모델 구현(env 이원화·prover 인자화) |
| `ca55a16` | sync-conf 개선 + 테스트 자기완결(§4)·TEST_PAYER(§5) |
| `da3c184` | README 상세화(파일|키|값 표 등) |
| `6608fa1` | upsertEnv 끝 개행 버그 수정(§5) |
| `809c1ea` | **sync-conf → autoconf 재편 전체**(신규 autoconf 3종 + sync-conf 2종 삭제 + README 일괄 + test/autoconf.ts) |

## 10. 환경 메모

- 디바이스 파일이 정본 — Cowork 스테이징 캐시는 구버전일 수 있어 검증·편집은 device_bash 직접 수행.
- device_bash 는 rm 불가(mv 로 `_to_delete/` 이동 후 사용자가 삭제).
- Node 22.18+ 타입 스트리핑: `node <file>.ts` 직접 실행 가능(erasable syntax 한정). test 패키지가 tsx 없이 .ts 를 실행하는 근거.
