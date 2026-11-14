# 🚗 AutoLog (현대자동차그룹 차량용 앱 공모전) — Codex 프론트엔드/UI 전면 재구현 핸드오프

> **수신자:** Codex (프론트엔드/UI 엔지니어)  
> **발신자:** Antigravity (코어 로직 & 계약 담당)  
> **프로젝트 위치:** `c:\dev\pleos`  
> **문서 목적:** 백엔드/코어 로직(`F03`) 검증 완료 상태에서, 사용자의 강력한 요구에 따라 **프론트엔드 UI/UX(`F02`)를 차량용 순정 감성으로 전면 재구현**하기 위한 핸드오프 가이드

---

## 1. 프로젝트 배경 및 문제 정의

* **공모전:** 현대자동차그룹 차량용 앱 개발 공모전 (Pleos Playground 기반, 11/15 1차 마감)
* **서비스명:** **AutoLog (오토로그)** — 차량 센서 기반 자동 차계부
* **해결하는 핵심 문제:**
  * 주행, 주유/충전 후 사용자가 수동 입력을 깜빡하여 차계부 기록이 누락되는 문제
  * 차량 센서가 알 수 없는 정보(예: 주유 시 결제 금액, 주유 리터 수)를 시스템이 임의로 왜곡(0원으로 표기)하지 않고, **근거/출처(Provenance)와 함께 안전하게 보존**하는 것
* **핸드오프 사유:**
  * 기존 Antigravity가 작성한 UI가 전형적인 AI 웹 대시보드 느낌으로 조잡하고 못생겨 사용자가 전면 교체를 요구함.
  * **Codex에게 실제 현대차 순정 인포테인먼트(ccNC) 느낌의 세련되고 시인성 높은 차량용 IVI 인터페이스로 재구현을 일임함.**

---

## 2. 현재 완료된 백엔드/코어 로직 (건드릴 필요 없음)

F03에 해당하는 데이터 계약, 순수 계산 엔진, 정정 스토어, 테스트는 모두 구현 및 100% 검증 통과했습니다.

### 2.1 파일 구조 (`c:\dev\pleos`)
```
c:\dev\pleos\
├── src\
│   ├── types\
│   │   └── models.ts             # B01-A 공통 데이터 계약 v0.2 타입 정의
│   ├── mock\
│   │   ├── iceFixture.js         # 11개 원시 ICE 센서 이벤트 Fixture (e01 ~ e11)
│   │   └── MockVehicleDataSource.js # 단계별(next), 일괄(emitAll), 리셋(reset) 어댑터
│   └── core\
│       ├── RecordEngine.js       # Leg 생성, Journey 묶음, 주유 감지 순수 계산 엔진
│       └── RecordStore.js        # 현재 상태 보존, 결제금액 정정, UserCorrection 불변 관리
└── tests\
    └── verify-f03.js             # 계약 및 완료조건 5단계 자동 검증 테스트 (통과 완료)
```

### 2.2 검증된 데이터 스펙 (단위 테스트로 입증됨)
`node tests/verify-f03.js` 실행 시 아래 5개 항목이 보장됩니다:
1. **Fixture**: 11개 이벤트(e01~e11) 시간순 방출
2. **구간(Legs)**: 
   * Leg 1: 08:00~08:15 (15분), 12,340 ➔ 12,345km (**5km**)
   * Leg 2: 08:25~08:45 (20분), 12,345 ➔ 12,352km (**7km**)
3. **여정(Journey)**: 
   * 1개로 묶임 (`groupingBasis: MOCK_SCENARIO`)
   * 총 거리: **12km**, 유효 운행시간: **35분** (중간 10분 시동 OFF 휴식시간은 정확히 제외됨)
4. **주유 감지(EnergyEvent / REFUEL)**:
   * 연료 30% ➔ 75% (**+45%p** 증가, `OBSERVED`)
   * 주유량(`quantityLiters`): **`null`**, `UNKNOWN` (0L가 아님)
   * 결제금액(`amountKrw`): **`null`**, `UNKNOWN` (0원이 아님)
   * 상태: `DETECTED_UNCONFIRMED`
5. **정정(UserCorrection)**:
   * 65,000원 입력 시 `amountKrw` ➔ `{ value: 65000, provenance: 'USER_INPUT' }`
   * 별도 `UserCorrection` 테이블에 `[previous: null(UNKNOWN) ➔ new: 65000(USER_INPUT)]` 기록
   * 원본 센서 관측값(`FUEL_PERCENT`: 30, 75)은 불변 유지

---

## 3. Codex가 구현해야 할 작업 (FrontEnd UI 전면 재구현)

### 3.1 디자인 원칙
* **진짜 현대차 순정 느낌 (Hyundai ccNC Cockpit):**
  * 싸구려 형광 네온이나 웹 대시보드 티를 완전히 제거할 것.
  * OLED 트루 블랙 배경, 차분하고 고급스러운 타이포그래피, 와이드(16:9 / 21:9) 비율 최적화.
  * 운전석 1m 거리에서도 0.5초 만에 읽히는 **빅 넘버(Big Metrics)와 높은 대비**.
* **Car UX 규정 준수:**
  * 주행 중 터치 방지 가이드라인 반영.
  * 정차 상태에서 3회 이내 터치로 끝나는 간편 키패드(퀵 프리셋 버튼: 5만, 6.5만, 7만 등).

### 3.2 필수 4개 화면 (F01/F02 규격)
Codex는 아래 4개 화면의 뷰와 전환 인터랙션을 구현해야 합니다.

1. **화면 1: 최초 실행 (Onboarding & Mode Notice)**
   * AutoLog 서비스 소개: "스스로 기록하고 안전하게 보존하는 차량용 차계부"
   * `MOCK DATA` 환경 안내 배지
   * [모의 주행 시연 시작 ▶] 원터치 진입 버튼
2. **화면 2: 주행 요약 (Journey & Legs)**
   * 집 ➔ 주유소 ➔ 회사 경로 시각화
   * 대형 수치: **총 12.0 km**, **총 35분 운행**
   * 2개 Leg 세부 내역 및 중간 10분(08:15~08:25) 시동 OFF 휴식 분리 표시
3. **화면 3: 주유 감지 카드 (Refuel Event)**
   * 상태: `DETECTED_UNCONFIRMED`
   * 연료 30% ➔ 75% (+45%p `OBSERVED`)
   * **주유량 및 결제금액이 0이 아니라 `미확인(null, UNKNOWN)`으로 표기되는 점을 시각적으로 부각**
   * [금액 정정하러 가기 ➔] 버튼
4. **화면 4: 조회 및 정정 (Correction & Audit Log)**
   * 결제금액 65,000원 입력 필드 + 정정 적용 버튼
   * 적용 후 `amountKrw`가 `65,000원 (USER_INPUT)`으로 변경
   * 하단 `UserCorrection` 분리 테이블에 정정 이력이 깨끗하게 추가됨을 표시
   * 원본 Mock 센서 데이터 불변 상태 명시

### 3.3 데모 제어 바 (Demo Controller)
* 화면 하단 또는 플로팅 도크 형태로 항상 접근 가능해야 함:
  * `[다음 이벤트 1개 실행 ▶]` (`dataSource.next()`)
  * `[전체 시연 즉시 실행 ▶▶]` (`dataSource.emitAll()`)
  * `[↺ 데모 초기화 (Reset)]` (`dataSource.reset()`, `store.reset()`) ➔ **DoD 필수 요건**
  * 현재 이벤트 상태 티커 (예: `Step 11/11 [e11] IGNITION = OFF`)

---

## 4. 프론트엔드 연동 가이드 (JS API)

Codex는 `c:\dev\pleos\src`의 기존 모듈을 그대로 임포트하여 사용하면 됩니다:

```javascript
import { MockVehicleDataSource } from './src/mock/MockVehicleDataSource.js';
import { RecordEngine } from './src/core/RecordEngine.js';
import { RecordStore } from './src/core/RecordStore.js';

// 1. 초기화
const dataSource = new MockVehicleDataSource();
let engineOutput = RecordEngine.process([]);
const store = new RecordStore(engineOutput);

// 2. 이벤트 구독 및 UI 갱신
dataSource.subscribe((latestEvent, allEvents) => {
  engineOutput = RecordEngine.process(allEvents);
  store.updateEngineOutput(engineOutput);
  updateUI(store.getData());
});

store.subscribe(() => {
  updateUI(store.getData());
});

// 3. 데모 조작
dataSource.next();     // 다음 1개 이벤트 방출
dataSource.emitAll();  // 11개 전체 이벤트 방출
dataSource.reset();    // 초기화

// 4. 금액 정정 (65,000원 입력 시)
store.updateEnergyField('fuel-1', 'amountKrw', 65000);
// store.getData().energyEvents[0].fields.amountKrw -> { value: 65000, provenance: 'USER_INPUT' }
// store.getCorrections() -> [{ correctionId: 'correction-1', ... }]
```

---

## 5. 완료 기준 (DoD) 체크리스트

Codex는 작업을 마친 후 다음 항목을 점검해야 합니다:
- [ ] 브라우저에서 단독으로 바로 열리는 파일(`index.html` 또는 번들 파일) 제공
- [ ] 디자인이 현대차 ccNC 순정 콕핏 느낌으로 깔끔하고 시인성이 뛰어난가?
- [ ] 화면 1 ➔ 2 ➔ 3 ➔ 4 전환 및 데모 조작(Next/All/Reset)이 정상 동작하는가?
- [ ] 금액 정정 시 65,000원이 `USER_INPUT`으로 반영되고 `UserCorrection` 테이블에 쌓이는가?
- [ ] `node tests/verify-f03.js`를 깨뜨리지 않고 데이터 정합성이 유지되는가?
