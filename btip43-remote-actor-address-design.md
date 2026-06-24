---
last_updated: 2026-06-24
type: 설계 기록 + 핸드오프 (BTIP-43 Remote Actor Address Derivation)
related: ./linker-v2.md, ./btips-2pc-design.md, ./bpun-origin-payment-design.md, ./axelar-ics20-vs-linker-v2-2026-06-24.md, ../../docs/BTIPS/btip-43.md
status: BTIP-43 Draft 작성 완료 (스펙만, 코드 미구현)
---

# BTIP-43 (Remote Actor Address Derivation on BPrN) — 설계 기록 / 핸드오프

> 이 문서는 2026-06-24 세션에서 BTIP-43이 어떻게·왜 만들어졌는지, 현재 상태, 남은 작업을 **자기완결적으로** 정리한 것이다. 다른 세션/시스템에서 이어받아 작업할 수 있도록 의도·결정·함정을 모두 담는다. "무엇을(스펙)"은 `docs/BTIPS/btip-43.md`에, "왜(보안 분석)"는 `./axelar-ics20-vs-linker-v2-2026-06-24.md`에 있고, 본 문서는 그 둘을 잇는 작업 맥락이다.

## 0. 한 줄 요약

다중 BPuN 소스 체인을 신뢰할 때 발생하는 **"여러 소스 체인상의 동일 주소 문제(주소충돌)"** 를 막기 위해, BPrN이 원격(BPuN) 행위자를 참조할 때 원시 20바이트 주소가 아니라 `btip43Addr(srcChainId, addr) = address(sha256(srcChainId ‖ addr)[12:32])`로 식별하도록 규정하는 BTIP-43을 Draft로 작성했다. **스펙만 완료, 코드 미구현.**

## 1. 발단 (왜 시작했나)

- 트리거: 2026-06 Axelar/Secret Network **CW20-ICS20 익스플로잇**(~$4.67M). 원인은 수정된 CW20-ICS20가 **소스 채널 검증을 누락** → 공격자가 자기 체인/채널에서 위조 입금 패킷으로 무담보 래핑 토큰 발행 → 정식 채널로 상환.
- 이를 Linker Protocol V2와 대조하며 보안 점검 진행(분석 전문은 `axelar-ics20-vs-linker-v2-2026-06-24.md`). 핵심 결론:
  - Linker는 **체인 편입(admission)을 프로토콜이 강제**(등록된 검증자셋/Root CA만) → Axelar식 "내 체인 만들어 위조" 수법은 비즈니스 컨트랙트가 출처검사를 0개 해도 재현 불가.
  - 단 **마지막 한 마일(어느 소스 컨트랙트/payer가 인가됐나)** 은 앱 책임. 여기서 allowance 스코핑 결함 발견 → 일반화하여 BTIP-43이 됨.

## 2. 함께 발견된 다른 보안 항목 (BTIP-43과 별개, 추적 필요)

- **(HIGH, 조치 완료) `LinkerPolicy.sol`의 `initPolicy` 접근제어 부재**: (이전) `contract LinkerPolicy is IBTIP22`로 `Ownable`을 import만 하고 상속 안 함 → `initPolicy`가 `external` + `_initialized` 가드뿐, 호출자 검증 없음. `setup.sh`가 deploy/bootstrap 2단계라 공개 BPuN에서 front-run 시 신뢰 루트(Root CA) 장악 가능했음. **→ `onlyOwner` 적용 + deploy+init 원자화로 해소(사용자 조치 완료).**
- (MEDIUM) `btip34-ccapp.CancelLinkerEvent`가 nullifier만 해제하고 비즈니스 상태(이전된 STC)는 역산 안 함 → admin cancel 후 재제출 시 이중 실행. (코드 주석에 "Model limitation" 명시)

## 3. BTIP-43 설계 진화 (왜 지금 형태가 됐나 — 순서대로)

1. **출발점**: allowance가 `[owner][spender]`로만 키잉되고 spender(=이벤트 emitter=원격 BPuN 컨트랙트)에 chainId가 없음 → 다중 소스 체인 + 주소충돌 시 타 체인 동일 주소가 위임 소진. 처음엔 "btip-34의 approve/allowance 확장"으로 접근.
2. **자가 점검**: 사용자가 "동일 주소면 동일인 아닌가?" 반문 → **CREATE는 EOA+nonce라 동일인이 맞지만, CREATE2 + 배포후 `initialize` 패턴이면 "같은 주소, 다른 컨트롤러"** 가 성립(실제 사고 부류: Optimism/Wintermute CREATE2 주소 재사용). 즉 좁지만 실재하는 벡터.
3. **범위 확장**: 사용자가 self-funds NOTE를 보고 "self-funds(`from==spender`)도 같은 문제 아닌가?" → 맞음. **잔액 계정(balance)도 20바이트로만 키잉되면 동일 충돌**. 즉 allowance뿐 아니라 잔액 소유 자체의 문제.
4. **"chainId를 모든 balance에 넣을 순 없다"** 우려 → 해소: **필드 추가가 아니라 주소를 파생**. `derive(C, addr)`로 20바이트 정규 주소를 만들면 기존 `balance[20byte]`/`allowance[owner][spender]` 맵 구조 불변, 키 값만 파생 주소.
5. **"raw vs derived를 어떻게 구분?"** → **바이트로 추측 금지, 필드 역할로 결정**. emitter/원격 핸들러는 항상 derive, BPrN-네이티브 보유자는 네이티브. `HandleLinkerEvent`의 `from==emitter`(self-funds) vs `from!=emitter`(위임) 분기가 자산 소유자 도메인을 그대로 가름.
6. **PayToBPuN 결론**: A(네이티브)가 chain C의 dApp M에게 지불 시, 수취/핸들러 = 원격 → `derive(C, M)` 필요 → **`PayToBPuN`이 `toChainId`를 인자로 받아야** 함. 2nd-phase 결과 매칭의 chainId는 검증된 증명(`payload.ChainID`)에서 공짜로 얻음.
7. **구조 재정의(사용자 지시)**: 문서 주제를 "approve/allowance 확장"이 아니라 **"원격 행위자 정규 주소 파생"** 으로 승격. approve/allowance는 적용 케이스 중 하나로 강등. → btip-34 한 인터페이스의 확장이 아니라 btip-26/34 공통 식별 규약(BTIP-9 파생의 거울상).

### 명명/표기 결정 이력 (사용자 피드백)

- `Cross-Chain Principal` → **폐기**(난해) → 명명 없이 풀어쓰다가 최종 함수명 `btip43Addr`.
- `srcChainId` → 한때 `spenderChainId`로 변경 검토 → 일반화하며 다시 `srcChainId`로(emitter/핸들러 등 spender 외 케이스도 포함하므로).
- 파생식 도메인 태그 `"bpun"` **제거**: `srcChainId`가 이미 네임스페이스 역할(BTIP-9 체인코드 주소는 채널·체인코드명 기반이라 입력 공간이 다름).
- 함수명 후보 `derivedAddr` vs `btip43Addr` → **`btip43Addr` 채택**(정의 위치가 이름에 드러남). 사용자가 `derivedAddr` 선호 시 교체 가능.
- 용어: '잔액'/'자금' → **'자산'** 통일(코드 식별자 `balance[...]`는 코드명이라 유지). Abstract "소스 체인 간 주소 충돌" → "여러 소스 체인상의 동일 주소 문제(주소충돌)".

## 4. BTIP-43 현재 내용 (스펙 요지)

```
btip43Addr(srcChainId, addr) = address( sha256( srcChainId ‖ addr )[12:32] )
```

식별 규칙: 역할로 결정. emitter/원격 핸들러 = 항상 `btip43Addr`, BPrN-네이티브 = 네이티브 주소.

적용 4개 지점:

| 지점 | 원격 행위자 | 정규 주소 | srcChainId 출처 |
|---|---|---|---|
| self-funds 자산 | emitter | `balance[btip43Addr(C, emitter)]` | `HandleLinkerEvent`의 `srcChainId` |
| allowance spender | emitter | `allowance[owner][btip43Addr(C, emitter)]` | `HandleLinkerEvent`의 `srcChainId` |
| `PayToBPuN` 수취 dApp | 수취 dApp | `btip43Addr(toChainId, to)` | 호출자 명시 `toChainId` |
| 결과 매칭 핸들러 | `handlerDApp` | `btip43Addr(payload.ChainID, handlerDApp)` | 검증된 결과 증명 |

EOA는 키가 전역 신원이라 제외(컨트랙트 행위자 한정). 단일 소스 체인 배포에선 충돌 없음 → **다중 소스 체인 지원의 선결 요건**.

## 5. 현재 토폴로지 전제 (중요)

- **현 구현은 1 BPuN : 1 BPrN 1:1**. 근거:
  - BPrN측 `linker-policy`: ValidatorSet 저장 키가 `VS_<height>`로 **height만**(`types/ibtip32.go`의 ValidatorSet에 chainId 없음). `linker-verifier`도 `GetValidatorSet(height)`로 height만 넘김 → 단일 BPuN.
  - BPuN측 `LinkerPolicy.sol`: 단일 채널 genesis 정책으로 init → 단일 BPrN. `LinkerEndpoint.onProof`는 `getContract(block.chainid, LINKER_VERIFIER)`로 자기 체인 verifier 하나만 사용.
  - 레지스트리(`LinkerRegistry`, `(chainId, role)` 키)·`OnResult`의 `ResolveContractOn(payload.ChainID, ...)`는 다중체인 형태이나, **검증/신뢰-루트 계층(policy)이 단일 소스**라 실효 1:1.
- **따라서 BTIP-43 결함은 현재 악용 불가**(다중 소스 체인 전제). 다중 BPuN 지원(policy 다중화 또는 ValidatorSet에 chainId 도입) 시점에 BTIP-43이 실제로 필요.

## 6. 구현 영향 (아직 미구현 — 다음 작업)

BTIP-43을 코드에 반영할 때 손볼 곳(주로 `verifier/on-bprn/btip34-ccapp/main.go`):

1. **파생 헬퍼 추가**: `btip43Addr(srcChainId, addr) = sha256(chainId ‖ addr)[12:]`. (`types/`에 두면 재사용 용이)
2. **`HandleLinkerEvent`**: emitter(contractAddress)를 `btip43Addr(srcChainID, emitter)`로 식별. self-funds 자산 소유자 = 그 파생 주소. 위임 인가 = `allowance[from][btip43Addr(srcChainID, emitter)]`.
3. **`Approve`/`Allowance`**: 시그니처에 spender의 소스 체인 포함 → `allowance[owner][btip43Addr(spenderChainId, spender)]`. (owner는 네이티브)
4. **`PayToBPuN`**: 인자에 `toChainId` 추가 → 수취/핸들러 = `btip43Addr(toChainId, to)`. pending에 그 파생 주소 기록.
5. **`HandleLinkerResult`**: 결과의 handlerDApp을 `btip43Addr(payload.ChainID, handlerDApp)`로 만들어 pending의 기록과 비교.
6. **잔액·allowance 맵 구조는 불변**(키 값만 파생 주소).
- ⚠️ **미해결 의존**: BPrN-**네이티브 계정**(EOA/직접 보유자)을 20바이트 주소 공간에서 어떻게 키잉하는지는 여전히 미정(`bpun-origin-payment-design §10-3` "주소 네임스페이스 통일" OPEN). 파생 주소와 네이티브 주소가 도메인 분리돼야 충돌 없음(BTIP-43은 파생 측만 정의).

## 7. 남은 작업 / 인계 사항

1. **btip-34 → btip-43 상호참조 포인터 추가** — `btip-34.md` 본문(호출자/출처 식별 또는 escrow 핸들러 절)에 "원격 행위자 식별·위임 권한 스코핑은 BTIP-43 참조" 링크 한 줄. (canonical 스펙 수정, 진행 예정)
2. **btip-34 Approve 인터페이스 확장 (chainId 스코프)** — btip-34 구현의 `Approve`가 spender의 **소스 체인**을 받도록 확장:
   - `Approve(owner, spenderChainId, spender, amount)` → 키 = `allowance[owner][btip43Addr(spenderChainId, spender)]`.
   - `Allowance(owner, spenderChainId, spender)` 조회도 대칭 확장.
   - ※ 사용자 메모상 "toChainId"라 했으나, 그건 `PayToBPuN`의 *수취* 체인 파라미터(§6-4)이고, `Approve`에서 받는 것은 *spender*의 체인이므로 명칭은 `spenderChainId`가 정확.
3. **BTIP-43 코드 구현 전반** (§6) — 다중 소스 체인 로드맵 시점. Approve(1번)·PayToBPuN(`toChainId`)·HandleLinkerEvent(emitter 파생)·결과 매칭 일괄.
4. **네이티브 계정 네임스페이스 정의** (§10-3) — BTIP-43 파생과 도메인 분리되는 네이티브 측 키잉 규약 별도 정리 필요.

## 8. 산출물 위치

- 스펙: `docs/BTIPS/btip-43.md` (Draft), README 인덱스 등재 완료.
- 보안 분석: `task-contexts/axelar-ics20-vs-linker-v2-2026-06-24.md`.
- 본 설계 기록: `task-contexts/btip43-remote-actor-address-design.md`.
