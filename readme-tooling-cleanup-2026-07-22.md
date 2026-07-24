# Linker V2 — README 전면 정리 + 툴링 후속 (2026-07-22)

> task-contexts 자기완결 기록. `linker-v2.md` §000000000(2026-07-22 핸드오프)에서 참조.
> 전제 문서: `deploy-bootstrap-tooling-2026-07-21.md`(표준순서·counterpart 모델·신원 통일) — 본 문서는 그 후속이며 겹치는 배경은 재서술하지 않는다.
> 대상 저장소: linker-v2 (verifier/on-bpun·on-bprn, prover-ts, test, bprn-organizations), 일부 kysee 메모리.

## 0. 세션 본류

linker-v2 레포 **README 전면 정리(한국어 통일)** 를 진행하다가 정합성 검토에서 파생된 **코드 정리 3건**까지 수행:
① deploy.sh → deploy.ts 전환, ② on-bprn 헬퍼(pay/approve/balance) 신원 통일, ③ `FABRIC_*` → `BPRN_*` env 리네임.
부수 발견: bpn connection-profile.json 소실 → 재생성. 전부 레포 반영 완료, deploy.ts는 **사용자 라이브 배포 테스트 통과**.

---

## 1. 코드 변경

### A. deploy.sh → deploy.ts (verifier/on-bprn/scripts)

- **입력 소스 재편**: peer/orderer 주소·TLS CA 경로·MSP ID는 `connection-profile.json` 파싱으로 획득
  (`clients[0].organization` → `organizations.*.mspid`/`peers[0]`, `channels.<ch>.orderers[0]`; 경로는 프로파일 디렉토리 기준 해석).
  lifecycle(package → install → approve → commit)은 여전히 `peer` CLI spawn — Node SDK(fabric-network 2.2)는 lifecycle 미지원.
- **도구 경로**: `config/config.<bprnChannelName>.json` 에 `deploy` 필드 신설, **3종 모두 필수** —
  `peerBin`(peer 바이너리), `adminMsp`(org Admin MSP), `fabricCfgPath`(`FABRIC_CFG_PATH`로 지정될 디렉토리, core.yaml 필요).
  경로는 config 파일 디렉토리 기준 상대/절대. **flat 이름 폴백 없음** — 필드 없으면 즉시 실패(사용자 지시로 폴백 제거).
- **sanity**: peer bin / core.yaml / adminMsp / TLS CA 2종 존재를 lifecycle 호출 전 일괄 검사(MISSING 전부 출력 후 종료).
- **env 정리**: 좌표 env(`BPRN_PEER_ADDRESS`/`BPRN_ORDERER_ID`/`BPRN_ORDERER_PORT`/`BPRN_MSPID`/`BPRN_CHANNEL`) 폐지 — 프로파일이 정본.
  `BPRN_CC_VERSION`/`BPRN_CC_SEQUENCE`(업그레이드 노브)만 유지.
- **core.yaml은 현상태 유지**(외부 bpn-core `fabric-config` 참조). 클라이언트 lifecycle용 core.yaml은 네트워크 비종속이라
  레포 vendoring으로 외부 의존을 끊는 선택지가 있었으나 사용자 결정으로 보류.
- package.json `deploy` → `tsx deploy.ts`. 구 `deploy.sh` → `scripts/_to_delete/`.
- config 반영: `config.bpn.json`·`config.<bprnChannelName>.json`(템플릿)에 `deploy` 블록 추가(현 값은 `bprn-organizations/bpn/`의 기존 flat 링크 경로 — 선언값이므로 규약상 무방).
  `config.devnet.json`엔 블록 없음 → devnet deploy는 config 단계에서 명확히 실패(devnet deploy 미지원 상태 그대로, 의도).
- **검증: tsc strict 통과 + 사용자 라이브 배포 테스트 통과(2026-07-22).**

### B. pay.ts / approve.ts / balance.ts 신원 통일 (7-21 통일 작업의 누락분)

- `DEFAULT_IDENTITY`(`user1-cert.pem`/`user1-key`/`Org1MSP`/`User1` 하드코딩, `bprn-organizations/<ch>/` 직접 읽기) 제거.
- bootstrap.ts와 동일 패턴: 프로파일 `clients[0]`(signedCertPath/privateKeyPath/mspId/id)에서 획득, 경로는 프로파일 디렉토리 기준 (`profileClient()` 헬퍼).
- 이로써 on-bprn 스크립트의 flat 이름 **코드 의존 완전 제거**(남은 flat 참조는 프로파일·config의 선언값뿐 — 규약상 허용).
- tsc 통과. **라이브 미검증**(balance 등 1회 실행 확인 권장).

### C. `FABRIC_*` → `BPRN_*` env 리네임

- `BPRN_CONNECTION_PROFILE` / `BPRN_CHANNEL_ID` / `BPRN_COMMIT_SIG_METADATA_INDEX`.
  배경: FABRIC_*은 Fabric 표준 변수가 아니라 프로젝트 정의 이름 — BPUN_*와 비대칭이라 체인명 기준으로 통일.
- 코드 소비처 2곳 수정: `prover-ts/src/common/env.ts`, `test/config.ts` (둘 다 transpileModule 검사 통과).
  소스 주석 5곳(btip19 cli/config/server, btip28 config, btip39 config) + 에러 문구 1곳(btip19 tx-event-proof) 동반 수정.
- env 파일: `prover-ts/.env`·`.env.example`, `test/.env.bprn.bpn`, example.
- example 정리: 폐지 변수 4줄(`FABRIC_MSP_ID`/`FABRIC_USER_NAME`/`FABRIC_CERT_PATH`/`FABRIC_KEY_PATH`, 7-21 신원 통일 때 잔존) 삭제,
  파일명 `.env.bprn.channelAlias.example` → **`.env.bprn.channelName.example`**(mv, 코드는 파일명 미참조). test/README 참조 갱신.
- `FABRIC_CFG_PATH`(Fabric 표준)만 유지. btip19/28 README의 변수 표도 갱신.
- 주의: 구동 중 prover는 재시작해야 반영. 레포 밖에서 `FABRIC_*` export하던 것이 있으면 갱신 필요.
  **리네임 후 e2e 재실행은 미실시**(코드 경로는 단순 문자열 치환 + transpile 확인).

### D. bpn connection-profile.json 소실 발견 → 재생성

- 발견: `bprn-organizations/bpn/connection-profile.json` 부재. prover-ts/.env·test/.env.bprn.bpn이 가리키는데 파일이 없어
  bpn 대상 bootstrap/prover/test 전부 불능 상태였음(7-21 핸드오프 시점엔 존재 — 비추적 파일이라 유실 경위 불명).
- 재생성: devnet 프로파일 스키마를 단일 org로 축소(`endorsementPolicyCount: 1`),
  `clients[0]` = {Org1, Org1MSP, User1@org1.bc, `./user1-key`, `./user1-cert.pem`},
  tlsCACerts = `./peer-tls-ca.crt` / `./orderer-tls-ca.pem`,
  `grpcs://peer0.org1.bc:7051` / `grpcs://orderer0.ordererorg.bc:7050`.
- **라이브 테스트 통과로 내용 검증됨.** 단 여전히 비추적 → 유실 위험 상존(추적 전환 여부 미결, §4).

---

## 2. README 전면 정리 (전부 한국어)

반영된 파일 7 + 유지 2:

| 파일 | 처리 |
|---|---|
| `README.md`(루트) | 재작성: 개요 + 구성 표 + 전제조건 + 표준 실행 순서 + 배포 현황 통합. 순서는 디렉토리 단위(on-bpun 전부 → on-bprn 전부 → prover 4종 → test), cd 왕복 제거. `DEPLOYMENTS.md` 분리안은 만들었다가 루트에 재통합(파일 없음) |
| `verifier/on-bpun/README.md` | Hardhat 보일러플레이트 → 역할+usage 신규(구성 표, 빌드, npm run 배포 3단계, sync-addresses NOTE) |
| `verifier/on-bprn/README.md` | 신설(구성 표, go build, 배포·부트스트랩) |
| `verifier/on-bpun/scripts/README.md` | 한국어화 + `--only` 재배포 절차를 deploy 단락 TIP으로, burn-nft 이후 헬퍼(nft/submit-proof/cancel-event) 절 생략, burn-nft도 생략 |
| `verifier/on-bprn/scripts/README.md` | 전면 재작성: 전제조건(네트워크+채널 1회 서술, 파일 요구 2갈래 표 — 고정 이름 2개 vs 경로 선언형), 설정 파일 절 신설(정적 config/동적 counterpart/deploy 필드), 명령 절 |
| `test/README.md` | 한국어화 + 코드 기준 정정(아래) |
| `prover-ts/README.md` | 신설: registry 해석 전제(BPUN_LINKER_REGISTRY/LINKER_REGISTRY_CC), 컴포넌트 표, btip39·btip18 env 표(자체 README 없는 둘), e2e 최소 조합(btip39:sync+btip19:api+btip28:api), int64 CAUTION, sync-addresses→재시작 주의 |
| btip19/btip28 README | 내용 유지(이미 한국어·최신), BPRN_* 변수명만 갱신 |

정정 사항(코드 대조):
- test/README: `LinkerResult` → **`LinkerResultAttrs`**, `onResult(proof, dApp)` → **`onResult(payload)`**(handlerDApp 제거 — 7-14 EmitterDApp 라우팅), 전제조건 하드코딩 `bpn` → 플레이스홀더.
- 루트 README 오타: Liner→Linker, 구현현→구현한, Chanincode→Chaincode, **BTIP24→BTIP34**(BTIP34CCApp 구현 대상 — 사실관계 정정).

## 3. 규칙/컨벤션 확정 (이번 세션, 메모리에도 저장됨)

- **`<bprnChannelName>`**: BPrN 선택자 표기 통일. 디렉토리(`bprn-organizations/<ch>/`)·config 파일명(`config.<ch>.json`)에 채널명을 쓰는 것이 규칙이므로 선택자 = 채널명(alias 아님). BPuN은 `<bpunChainAlias>` 유지.
- **md 개행 = 줄 끝 공백 2개 필수**(인용문 `>` 내부 줄 포함).
- **Fabric 아닌 BPrN**: 문서·변수명에서 BPrN/BPuN 기준. Fabric은 기반 기술 설명·고유명칭(peer CLI, fabric-network, FABRIC_CFG_PATH)에만. → C 리네임의 근거.
- README는 역할+usage(기존 규칙 유지), 전제조건은 1회 서술.

## 4. 남은 작업

- **코드 usage 문자열 리네임**: on-bpun `bootstrap.ts`/`setup.sh`(`bprnChannelAlias`), on-bprn `bootstrap.ts`(`<bprnChannel>`), test `run.mjs`(`<bprnChannelAlias>`) → `bprnChannelName` 통일.
- **라이브 재확인**: BPRN_* 리네임 후 e2e 1회, pay/approve/balance(clients[0] 전환) 1회.
- **git 커밋(사용자)**: README 7 + deploy.ts 세트(package.json·config 2종) + 헬퍼 3 + env/리네임 일괄 + `_to_delete/` + bprn-organizations/bpn/connection-profile.json(비추적이면 커밋 불필요, 추적 전환 시 포함).
- **추적 전환 검토**: bpn `connection-profile.json`·`config.bpn.json`(비추적 → 소실 재발 위험).
- devnet deploy 미지원 그대로(7-21 문서 §5의 갭 — 별도 과제).

## 5. 환경 메모

- 클라우드 VM에서 Mac 절대경로 타깃 심볼릭은 항상 BROKEN으로 보임(착시) — 실존 여부는 device_list_dir(Mac측)로 확인할 것.
- 이번 세션 tsc 검증: deploy/pay/approve/balance는 샌드박스 tsc(strict) — fabric-network 설치 후, env.ts/config.ts는 디바이스 node transpileModule.
