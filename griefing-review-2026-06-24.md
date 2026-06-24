---
last_updated: 2026-06-24
type: 보안 점검 기록 (griefing/DoS — Axelar 트랙과 별개)
related: ./linker-v2.md, ./btips-2pc-design.md, ./axelar-ics20-vs-linker-v2-2026-06-24.md, ../../docs/BTIPS/btip-26.md, ../../docs/BTIPS/btip-34.md
status: BTIP26Token u2r IntendedHandler 갭 — 코드+스펙 수정 완료
---

# griefing / DoS 점검 결과 (2026-06-24)

> 사용자 명시대로 **Axelar 재현 점검과 별개 트랙**. 2PC 결과-라우팅 griefing을 재검토하고, BTIP26Token의 u2r IntendedHandler 갭을 발견·수정했다. griefing 설계 근거의 본류는 `btips-2pc-design.md`(§3 결정 3, §9 #5a)에 있고, 본 문서는 이번 재검토와 수정의 자기완결 기록이다.

## 1. 기존 griefing 결론 재확인 (변경 없음)

`btips-2pc-design.md`에서 벡터별로 이미 닫혀 있음을 재확인:

- **#5a IntendedHandler 바인딩**: 정방향 증명을 "항상 거부" 핸들러로 흘려 강제 REJECTED를 유발하는 griefing → 발신 앱이 lock 시점에 **의도한 핸들러를 기록**하고 결과에서 **대조**(불일치 무시). 프로토콜은 위조 불가한 "누가 판정했나"만 공급, 대조는 앱 책임.
- **per-handler Nullifier** (`onResult`가 `(txEventRoot, handlerDApp)`로 소비): 제3자가 결과를 자기 dApp으로 흘려 증명을 소진·정당 dApp 차단하는 벡터 제거.
- **OOG**: `MIN_CALLBACK_GAS` 하한 폐기(거짓 안전감), `ErrAppLowGas` 명시 신호로 *명시적* grief만 완화. 암묵적 OOG는 EVM 한계로 방어 불가 — 스펙에 명시.
- **freeze**: 타임아웃 없음 + permissionless self-submit → BPuN이 살아있는 한 동결 없음.
- **중복 제출**: Nullifier dedupe.

## 2. 발견 — BTIP26Token의 u2r IntendedHandler 갭 (방향별 비대칭)

- **r2u 발신자 `btip34-ccapp.HandleLinkerResult`**: `handlerDApp == pending.HandlerDApp` 대조 → 불일치 하드 거부. **#5a 구현됨.**
- **u2r 발신자 `BTIP26Token.handleLinkerResult`**: `handlerCcId`를 **대조하지 않음** — `correlationId` 존재만 확인하고 finalize. **#5a 미구현(갭).**
- **위험**: burn 증명을 "항상 수락" ccApp으로 라우팅 → 실제 STC 미전송인데 `LinkerResult(ACCEPTED, handlerCcId=그_ccApp)` → BTIP26Token이 burn을 finalize(NFT 미복원) → 소각자가 NFT도 STC도 잃음.
- **실위험도**: 통제된 BPrN + chaincode 승인 규율 하에선 "항상 수락" ccApp 배제로 즉각 위험은 낮음. 그러나 **#5a 설계 결정(발신자 양쪽 모두 바인딩)에 대한 스펙 비대칭**이라 거버넌스에만 기대지 않도록 메움.

## 3. 수정 (코드 + 스펙) — 완료

- **`BTIP26Token.sol`**: `handleLinkerResult`에 대조 추가 — `handlerCcId == _paymentChaincode`(r2u 결제 출처 STC)가 아니면 **`ErrUntrustedResultHandler` revert**(하드 거부 → `onResult` tx 롤백, pending burn 유지, 정당 결과 재제출 가능). 채널은 `onResult`의 출처 검증(소스=공식 BPrN 엔드포인트)이 이미 핀하므로 **ccApp id만 대조**.
  - 전제: **u2r 핸들러 = r2u 출처(같은 STC)** — 브리지 값 보존 불변식상 정합. 코드 주석에 명시. (둘을 다른 ccApp으로 둘 배포는 별도 설정 필요 — 현 모델 밖.)
- **명칭 통일 `handleCcId` → `handlerCcId`**: btip-26 인터페이스/본문, `IBTIP26.sol`, `BTIP26Token.sol`. 정본은 `handlerCcId`(`btips-2pc-design` 2026-06-09 리네임 `appChaincodeID→handlerCcId`; Go/LinkerEndpoint도 동일). (BTIP26Token 이벤트 정의는 이미 `handlerCcId`였음.)
- **`btip-26.md` §2PC line 126**: "의도한 ccApp 주소가 기록되어 있다면 … 필요에 따라 … 할 수도 있다"(optional) → **"요청 처리 ccApp을 의도한 하나로 한정하려면 발신 시점에 기록하고 `handleLinkerResult`에서 `handlerCcId`를 대조해야 한다(불일치 거부)"** (조건부 의무, 범용 표현 유지, griefing 차단 명시). 결제 도메인 구체("같은 STC")는 스펙에 넣지 않고 BTIP26Token 구현 주석에만.

## 4. 수용된 한계 (결정됨 — 추가 작업 아님)

> griefing 트랙은 **종결**됨. 아래는 미해결 작업이 아니라 검토 후 *수용·문서화된 한계*다.

- **암묵적 OOG → 영구 REJECTED**: EVM 한계로 방어 불가. 재검토했고 수용, 스펙에 명시. (추가 조치 없음)
- **same-STC 전제**: u2r 핸들러 = r2u 출처(같은 STC)를 전제. 다른 ccApp으로 둘 use case는 별도 설정 필요(현 모델 밖).
