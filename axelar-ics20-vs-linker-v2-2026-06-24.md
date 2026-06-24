---
last_updated: 2026-06-24
type: 보안 분석 (cross-chain source verification)
related: ./linker-v2.md, ./btips-2pc-design.md, ./bpun-origin-payment-design.md, ./audit-report-2026-05-28.md
trigger: Axelar/Secret Network CW20-ICS20 익스플로잇(2026-06, ~$4.67M)
---

# Axelar/Secret ICS20 사고 vs Linker Protocol V2 — 공격 트리 + 약점 영향 분석

> 2026-06 Axelar/Secret Network 브리지 사고(수정된 CW20-ICS20가 소스 채널 검증을 누락 → 무담보 발행 → 정식 채널로 상환, ~467만 달러)를 기준점으로, 동일 부류 공격이 Linker V2에서 어디까지 막히고 어디가 앱 책임으로 남는지 정리한다.
>
> 본 문서는 점검·분석만 수행하며 코드/문서 수정은 하지 않는다. 수정은 별도 컨펌 후.

---

## 0. 한 장 요약

- 이번 사고가 **실제로 깨진 지점은 "체인 편입(admission)"** 이다 — 공격자가 자기 체인을 신뢰 경계에 끌어들였고(IBC permissionless 채널/클라이언트 생성), "이게 내가 신뢰하는 채널인가"의 최종 확인이 앱(CW20-ICS20)에 통째로 위임돼 있었는데 그게 빠졌다.
- **Linker V2는 이 지점을 프로토콜이 막는다.** 소스 체인은 `LinkerPolicy`에 등록된 검증자셋/Root CA를 가진 체인이어야만 검증을 통과하며, 편입은 admin H₀ + 암호학적 체이닝 갱신(BTIP39)으로만 — permissionless 편입이 없다. → **Axelar의 "내 체인 만들어 위조 입금" 수법은 비즈니스 컨트랙트가 출처검사를 0개 해도 재현 불가.**
- 단, 이 장점은 **두 가지 전제** 위에서만 절대적이다:
  1. 신뢰 루트(`LinkerPolicy`의 genesis 정책)가 정당하게 초기화되었을 것 → **`initPolicy` 접근제어 부재로 현재 깨질 수 있음 (HIGH).**
  2. permissioned 모델(알려진 검증자셋/Root CA)을 전제 → IBC permissionless와는 위협 모델이 다른 *선택*이지 공짜 우월성이 아님.
- 앱에 남는 "마지막 한 마일"(어느 소스 컨트랙트/payer가 인가됐나)은 Linker도 위임한다. 여기서 **allowance chainId 스코핑 누락(MEDIUM, 다중체인 조건부)** 과 **cancel→replay 상태 비가역(MEDIUM, admin 조건부)** 이 남는다.

**결론**: 헤드라인 장점(프로토콜 강제 체인 편입)은 실재하나 **현재 `initPolicy` 버그로 조건부**다. `initPolicy`만 게이팅하면 장점은 견고해지고, 나머지는 ICS20보다 *작은* 앱층 숙제로 좁혀진다.

---

## (a) 공격 트리 — Linker가 막는 것 vs 앱 책임

### 공격자 목표

수신 체인에서 **무담보 가치 생성** — BPuN dApp이 백킹 없는 mint, 또는 BPrN ccApp이 인가 없는 지급.

```
[ROOT] 수신 컨트랙트가 정당한 백킹 없이 가치를 발행/지급하게 만든다
│
├─ B1. 소스 체인 위조 (= Axelar의 결정타: 공격자 자기 체인)
│     공격자가 자기 체인/채널을 세워 위조 증명 제출
│     ├─ BPrN→BPuN: LinkerVerifier→LinkerPolicy가 endorsement를 등록된 Root CA로
│     │             체이닝 검증. 공격자 MSP 인증서는 체이닝 실패 → revert
│     │             (handleLinkerEvent 도달 전)
│     └─ BPuN→BPrN: linker-verifier가 commit 서명을 등록된 검증자셋과 대조.
│                   공격자 검증자 서명 위조 불가(개인키 필요), chainId는
│                   CanonicalVote 서명에 바인딩 → 다른 셋으로 리다이렉트 불가
│     ⇒ ★ 프로토콜이 차단 (LinkerPolicy/LinkerVerifier). 비즈니스 컨트랙트
│        출처검사 0개여도 차단됨. ── 단 B4가 성립하면 무력화
│
├─ B2. 정식 증명 재생(replay)
│     LinkerEndpoint가 검증 전 markProcessed(eventRoot, handler), tx 원자성
│     ⇒ ★ 프로토콜이 차단 (LinkerNullifier). ── 단 cancelNullifier 후엔 재개방(b §B2')
│
├─ B3. 신뢰 체인 위 "다른 컨트랙트"가 가짜 결제 이벤트 emit (소스 컨트랙트 위조)
│     이미 신뢰된 BPrN 채널의 임의 체인코드가 TransferLogElems 모양 이벤트 emit
│     ├─ 프로토콜(onProof): chaincode_id 검사 안 함 — 그대로 앱에 전달
│     └─ 앱(BTIP26Token): (channel_id, chaincode_id)==paymentSource 검사 → 거부
│     ⇒ △ 앱 책임. BTIP26Token은 막지만, dApp 개발자가 이 검사를 빠뜨리면
│        무담보 mint 가능. ◀── 사용자가 지적한 "마지막 한 마일" (잔존 위험)
│        ※ 단 Axelar와 달리 발신자는 "아무 체인"이 아니라 "이미 신뢰된 체인 안"
│           으로 한정 → 폭발 반경이 작음(외부인 → 컨소시엄 내부자)
│
├─ B4. 신뢰 루트 자체를 오염 (initPolicy front-run)
│     LinkerPolicy.initPolicy에 접근제어 없음 + deploy/bootstrap 별도 tx
│     → 공격자가 자기 Root CA로 initPolicy 선점
│     ⇒ ✗ 프로토콜 약점. 성립 시 B1의 보호가 통째로 붕괴(공격자 체인이
│        "정식"이 됨) → 완전한 Axelar급 공격 가능. (b §B4 참조)
│
├─ B5. 가짜 결과로 남의 에스크로 settle/refund
│     onResult/OnResult: 소스==공식 LinkerEndpoint(레지스트리 조회)+chainId 바인딩
│     + 앱의 IntendedHandler 대조
│     ⇒ ★ 프로토콜이 차단
│
└─ B6. (u2r) payer 자금 무단 사용
      btip34-ccapp: 소스 컨트랙트 화이트리스트 대신 allowance 모델
      (from==msgSender self-funds | allowance[from][msgSender]≥amount)
      ├─ 악성 dApp이 from=Alice로 emit → allowance[Alice][Mallory]=0 → 거부
      └─ 단 allowance가 [from][spender]만 키잉(chainId 없음)
      ⇒ △ 앱 책임. 단일 신뢰 체인에선 안전, 다중 체인+주소충돌 시 탈취 가능(b §B6)
```

### 분류 요약

| 분기 | 차단 주체 | 상태 |
|---|---|---|
| B1 소스 체인 위조 | **프로토콜** (Policy/Verifier) | ★ 차단 (B4 전제) |
| B2 재생 | **프로토콜** (Nullifier) | ★ 차단 (cancel 예외) |
| B3 소스 컨트랙트 위조 | **앱** (paymentSource 검사) | △ 앱이 하면 차단 / 빠뜨리면 사고 |
| B4 신뢰 루트 오염 | 프로토콜 (현재 미보호) | ✗ **약점** |
| B5 가짜 결과 | **프로토콜** (출처검증) | ★ 차단 |
| B6 payer 무단사용 | **앱** (allowance) | △ 단일체인 안전 / 다중체인 약점 |

### ICS20(이번 사고)과의 핵심 대비

| 칸 | ICS20 (사고 컨트랙트) | Linker V2 |
|---|---|---|
| 체인 편입 | **앱 책임** (permissionless, **여기서 깨짐**) | **프로토콜 강제** (등록된 신뢰 루트만) |
| 위조 불가 출처 정보 | 앱이 직접 채널 확인 | `srcChainId`·`(channel,chaincode)`를 위조 불가로 공급 |
| 기본 동작(무설정 시) | mint (fail-open) | 샘플 BTIP26Token은 전부 거부 (fail-safe) |
| 값 보존 | 별도 불변식 (빠짐) | 2PC 에스크로 (lock↔settle/refund) |
| 앱에 남는 책임 | 큼 (체인+채널+값) | 작음 (신뢰 체인 안에서 "어느 소스/payer") |

핵심: 이번 사고가 깨진 칸(체인 편입)이 Linker에서는 **앱 → 프로토콜로 이동**했고, 앱에 남은 건 더 작은 "마지막 한 마일"이며 그마저 **암호학적으로 신뢰된 토대 위에서** 시작한다.

---

## (b) 약점이 헤드라인 장점을 얼마나 깎는가

헤드라인 장점 = **"B1(소스 체인 위조)을 프로토콜이 막는다."** 아래 약점들이 이를 얼마나 침식하는지 평가.

### B4. `initPolicy` front-run — **HIGH (헤드라인 장점을 조건부로 만듦)**

- **근거(코드)**: `LinkerPolicy.sol`은 `contract LinkerPolicy is IBTIP22`로 선언 — `Ownable`을 import만 하고 **상속하지 않음**. `initPolicy(bytes)`는 `external`이며 가드는 `if (_initialized) revert PolicyAlreadyInitialized();` 뿐 → **호출자 검증 없음.**
- **근거(배포 흐름)**: `setup.sh`가 `[1/2] npm run deploy` → `[2/2] npm run bootstrap` 2단계. `initPolicy`는 bootstrap 단계(`bootstrap.ts` step)에서 호출 → **deploy와 별도 트랜잭션 → 선점 창 실재.**
- **공격**: 공개 BPuN 체인에서 공격자가 mempool 관찰 → 배포 직후 자기 Root CA·endorsement 정책으로 `initPolicy` 선점. 이후 정식 bootstrap의 `initPolicy`는 `PolicyAlreadyInitialized`로 revert. 공격자가 **신뢰 루트를 장악** → B1 붕괴 → 공격자 체인이 "정식"이 되어 무담보 mint(완전한 Axelar급).
- **전제조건**: ① BPuN tx 제출이 permissionless/공개, ② deploy–init 사이 시간 창(현 스크립트 구조상 성립), ③ 공격자의 mempool 관측.
- **폭발 반경**: 해당 BPuN 배포의 r2u 전 경로(전체 dApp). 신뢰 루트 장악이라 사실상 critical.
- **장점 침식도**: **결정적.** 이 버그가 있으면 "Linker는 소스 체인 위조를 막는다"는 **보장이 아니라 레이스 결과**가 된다. 즉 헤드라인 장점이 현재로선 *조건부*.
- **완화/수정(난이도 낮음)**: `initPolicy`를 `onlyOwner`로 게이팅 / 생성자에서 초기화 / deploy+init 원자화 / "deployer만 init" 체크. 하나만 적용해도 B4 소거.
- **순효과**: 수정 전 = 헤드라인 장점 **무효화 가능**. 수정 후 = 장점 견고.

### B6. allowance chainId 스코핑 누락 — **MEDIUM (다중체인 조건부, 마지막 마일 침식)**

- **근거(코드)**: `btip34-ccapp`의 `allowanceKey(owner, spender)` = `ALW_<owner>_<spender>` — **chainId 없음.** `HandleLinkerEvent`는 `srcChainID`를 파라미터로 받지만 인가 판정에 **사용하지 않음**.
- **공격**: 신뢰 소스 체인이 ≥2개 등록된 환경에서, Alice가 chainA의 Mallory를 의도해 `approve(Alice, Mallory)`. 공격자가 chainB에 같은 주소 Mallory를 CREATE2로 배포 → chainB발 이벤트로 `allowance[Alice][Mallory]` 소진. 두 체인 다 정식이라 화이트리스트로는 못 막음(스코핑 문제).
- **전제조건**: ① 다중 신뢰 소스 체인, ② 주소 충돌(CREATE2로 trivial). **현재는 단일 신뢰 체인 가정으로 미성립** → 잠재 위험.
- **폭발 반경**: 미상환 allowance 한도 내. 임의 mint가 아니라 *승인한 사용자의 승인액*에 한정 → 제한적.
- **장점 침식도**: **헤드라인(B1) 불침범.** "마지막 한 마일이 더 작고 안전하다"는 부차 주장만 일부 깎음. 설계 문서 §10-3(주소 네임스페이스 통일) OPEN 항목과 동일.
- **수정**: allowance를 `(from, srcChainId, spender)`로 스코핑. 프로토콜이 이미 위조 불가 `srcChainId`를 공급하므로 **앱층 한 줄** 변경. (LinkerRegistry chainId 화이트리스트로는 해결 불가 — 화이트리스트=membership, 필요한 건 scoping.)

### B2′. cancelNullifier → replay 상태 비가역 — **MEDIUM (admin 조건부 footgun)**

- **근거(코드)**: `btip34-ccapp.CancelLinkerEvent`는 nullifier만 해제하고 **HandleLinkerEvent가 옮긴 STC는 역산하지 않음**(코드 주석 "Model limitation" 명시). `BTIP26Token.cancelLinkerEvent`도 동일(취소 후 재제출 시 NFT 재mint 가능).
- **공격**: admin/owner가 cancel → 같은 증명 재제출 → 비즈니스 동작 2회 실행(STC 이중 전송 / NFT 이중 mint).
- **전제조건**: admin/owner 권한(또는 키 탈취). 외부 무권한 공격 아님.
- **폭발 반경**: 취소된 이벤트 1건의 금액/토큰 중복.
- **장점 침식도**: B2(재생 방지)를 admin 행위 하에 침식. 헤드라인(B1) 불침범.
- **수정**: 취소 시 비즈니스 상태도 역산(event_attrs_root별 settlement 기록 후 reverse). cancel 자체를 신중 사용(탈출구 성격).

### 종합 평가표

| 약점 | 심각도 | 전제조건 | 폭발 반경 | 헤드라인(B1) 영향 | 수정 난이도 |
|---|---|---|---|---|---|
| B4 initPolicy front-run | **HIGH** | 공개 체인+배포창 | BPuN 전체 r2u | **결정적(무효화 가능)** | 낮음 |
| B6 allowance 스코핑 | MEDIUM | 다중 신뢰체인+주소충돌 | 미상환 allowance 한도 | 불침범(마지막 마일) | 낮음(앱) |
| B2′ cancel→replay | MEDIUM | admin 권한 | 취소 1건 중복 | 불침범 | 중 |

### 최종 판정

- **사용자 회의("표준 안 지키면 똑같다")는 마지막 한 마일(B3/B6)에 한해 맞다.** 그 구간은 Linker도 앱에 위임한다.
- **그러나 이번 사고가 실제로 깨진 칸(B1 체인 편입)은 Linker가 프로토콜에서 막는다** — 이건 실재하는 구조적 장점이다.
- 단 그 장점은 현재 **B4(initPolicy)로 조건부**다. `initPolicy` 게이팅이 **최우선 권고**. 이것만 닫으면:
  - 헤드라인 장점(B1 차단)이 보장으로 승격,
  - 잔존 위험은 B3/B6/B2′ — 모두 *작은 앱층 숙제*로, ICS20이 앱에 떠넘긴 체인 편입 책임보다 본질적으로 작다.
- 공정 단서: 이 장점은 **permissioned 설계의 결과**다. IBC permissionless는 개방형 생태계를 위한 의도된 선택이며, Linker는 그 유연성을 포기한 대가로 안전을 얻고 동시에 B4 같은 *자기만의 신뢰-루트 부트스트랩 위험*을 떠안는다.

---

## 권고 (우선순위)

1. ~~**(HIGH) `LinkerPolicy.initPolicy` 접근제어**~~ — **완료**: `onlyOwner` 적용 + deploy+init 원자화. 헤드라인 장점(B1 차단)이 보장으로 승격됨.
2. **(MEDIUM) allowance 스코핑** — `btip34-ccapp` allowance 키에 `srcChainId` 포함(프로토콜이 이미 공급). 다중체인 전 선반영.
3. **(MEDIUM) cancel 상태 역산** — production STC/dApp은 cancel 시 비즈니스 상태도 되돌리도록 명문화/구현.
4. **(설계 명문화) 출처검증 강제** — 화이트리스트가 아니라 `LinkerApp` base 헬퍼/스펙으로 "수신 앱은 `(srcChainId, source)`로 인가를 스코핑한다"를 권고·표준화(B3/B6 재발 방지의 설계 정합적 방어).

> 본 분석은 점검만 수행. 위 1~4의 코드/문서 반영은 항목별 컨펌 후 진행.

---

## 점검 결과 — "Axelar 사고의 Linker 재현 가능성" (2026-06-24 리뷰 결론)

> 이 리뷰의 범위는 *전체 보안 점검*이 아니라 **Axelar/Secret ICS20 사고(출처 검증 누락 → 무담보 발행)가 Linker에서 재현되는가**로 한정. 아래는 사용자와의 검토 끝에 도달한 결론이다.

### 결론

> **Linker의 의도된 배포(통제된 단일-BPrN + 규율 있는 chaincode 승인 + 단일 STC + `initPolicy` onlyOwner 조치)에서는 Axelar 부류 사고가 재현되지 않는다.**

- **프로토콜 계층(체인 편입)**: 공격자의 자체 네트워크 X는 LinkerVerifier/LinkerPolicy(endorsement·Root CA)에서 거부 → Axelar의 핵심 수법(자기 체인을 신뢰 집합에 편입)이 불가. *전제: `initPolicy` 게이팅(아래).*
- **값 보존**: 2PC 에스크로 + Nullifier(증명당 결과 1회)로, 무담보 발행은 위 편입을 뚫어야만 가능.
- **출처(어느 체인코드?) 검증**: 프로토콜이 이미 **신뢰 BPrN 채널 자체를 Root CA로 핀**하므로 dApp이 채널을 다시 볼 필요 없음. 남는 "체인코드 핀"은 **Fabric chaincode lifecycle 승인(참여 org 동의)** 으로 발행측에서 강제됨 — "STC 락을 동반하지 않고 `TransferLogElems`를 발행하는 체인코드"는 컨소시엄 검토에서 배포 거부됨. 즉 **출처 검증은 소멸한 게 아니라 "수신 dApp 코드 검사" → "배포 승인 규율"로 이동**하며, 통제 환경에서 이는 정당·충분하다.

### 핵심 통찰 (검토 과정에서 정리된 것)

1. **이벤트 발행 ≠ 자산 이동.** `TransferLogElems`는 라벨일 뿐, STC 잔액은 STC 체인코드 state에 있고 그 락↔emit을 묶는 건 STC 체인코드 코드뿐. 따라서 "그 이벤트=실제 STC 락"은 자동이 아니라 *발행 체인코드가 그걸 보장할 때만* 참.
2. 그 보장은 **(a) 수신측 출처 핀** 또는 **(b) 발행측 배포 거버넌스 불변식** 중 **적어도 하나**로 강제돼야 함. 통제된 BPrN은 (b)로 충족.
3. **`To`(목적지) 검증은 '출처'와 다른 별개 축** — Axelar 부류와 무관. (permissionless 라우팅 관련 이슈로, 본 리뷰 범위 밖이나 dApp 정확성 차원에서 별도 고려.)

### 결론이 의존하는 전제 (명시·영속 유지 필요)

1. **`initPolicy` onlyOwner 조치** — **적용 완료**(+ deploy와 동시 init으로 front-run 갭 제거). 신뢰 루트 탈취 경로 차단됨.
2. **거버넌스 불변식**: "2PC 의미(STC 락+정산)를 갖는 이벤트 시그니처는 그 의미를 올바로 구현한 체인코드만 발행한다"를 — 모든 신규 체인코드·버전·멤버십 변경에 걸쳐 — 승인 검토에서 강제. (자동이 아닌 *유지해야 하는 약속*.)
3. **honest-majority** lifecycle 승인 (통제 환경 전제).

### 잔여 가치 / 권고

- **수신측 출처 핀의 유일한 추가 가치**: BPuN dApp을 **BPrN 거버넌스를 신뢰하지 않는/볼 수 없는 제3자**가 만들 때의 자기방어. 컨소시엄이 dApp까지 신뢰 경계에 두면 이 가치도 없음.
- **권고**: 위 전제 2(거버넌스 불변식)를 **BTIP/운영 문서에 명시적 불변식**으로 박아둘 것. "통제된 BPrN"이라는 말만으로 자동 보장되지 않으므로, 규약으로 고정해야 영속적으로 성립.

### 범위에서 제외된 항목 (별도 추적)

- `initPolicy` front-run(§B4) — onlyOwner+원자배포로 제외 처리.
- cancelNullifier 상태 비가역(§B2′), 다중체인 allowance 스코핑(§B6, → BTIP-43) — 본 리뷰(Axelar 재현)와 별개 트랙. BTIP-43 및 설계 기록 참조.
