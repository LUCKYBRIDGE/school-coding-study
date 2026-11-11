# kimman-ai 재구현 명세

이 문서는 원본 코드를 복사하기 위한 문서가 아니라, 공개적으로 관찰되는 기능과 동작 원리를 토대로 **독립적인 교육용 구현**을 설계하기 위한 명세입니다.

---

# 1. 권장 전체 아키텍처

## 1.1 목표

- 서버 없이 동작
- 학생 계정 없음
- 수업 중 네트워크가 불안정해도 주요 기능 사용 가능
- 사진 원본 외부 전송 없음
- 인쇄 가능
- 모바일/태블릿/PC 공통 지원

## 1.2 권장 스택

가장 단순한 선택:

- Vite
- TypeScript
- Vanilla DOM 또는 Preact
- Canvas API
- IndexedDB/localStorage
- CSS Grid
- Cloudflare Pages

프레임워크가 꼭 필요한 서비스는 아닙니다.

### 추천 폴더 구조

```
src/
├─ app/
│  ├─ routes.ts
│  └─ shared/
│     ├─ StepNav.ts
│     ├─ PrintButton.ts
│     ├─ PrivacyNotice.ts
│     └─ SafetyGuard.ts
├─ pixel/
│  ├─ PixelGrid.ts
│  ├─ binary.ts
│  ├─ rle.ts
│  ├─ rasterize.ts
│  ├─ photo.ts
│  └─ print.ts
├─ monster/
│  ├─ generator.ts
│  ├─ features.ts
│  ├─ hypotheses.ts
│  ├─ trainer.ts
│  └─ print.ts
├─ rps/
│  ├─ camera.ts
│  ├─ preprocess.ts
│  ├─ prototype-model.ts
│  ├─ evaluator.ts
│  ├─ confusion.ts
│  ├─ model-store.ts
│  └─ print.ts
├─ copyright/
│  ├─ licenses.ts
│  ├─ quiz.ts
│  ├─ matching.ts
│  ├─ sticker.ts
│  └─ print.ts
└─ styles/
   ├─ base.css
   └─ print.css
```

---

# 2. 공통 UI 시스템

## 2.1 StepNav

모든 앱의 상단에 동일한 단계 내비게이션을 둡니다.

```ts
type Step = {
  id: string;
  number: number;
  label: string;
};
```

### 요구사항

- 현재 단계 시각적 강조
- 클릭으로 이동 가능
- 모바일에서는 가로 스크롤
- 각 단계에 `aria-current="step"`

---

## 2.2 ActionButton

학생 대상 버튼은 기능명이 아니라 행동 문장으로 씁니다.

좋음:

- 숫자로 바꾸기
- 학습시키기
- 맞춰 보기
- 다시 시험 보기

피해야 함:

- 실행
- submit
- process
- apply

---

## 2.3 상태 피드백

각 행동 뒤에는 바로 다음 중 하나를 보여 줍니다.

- 숫자 변화
- 격자 변화
- 그래프/막대 변화
- 성공/실패 위치
- 점수
- 다음 행동

“처리되었습니다” 같은 추상적 토스트만으로 끝내지 않습니다.

---

# 3. 픽셀 스튜디오 구현

## 3.1 상태 모델

```ts
type BinaryGrid = {
  width: number;
  height: number;
  cells: Uint8Array; // 0 | 1
};

type SavedPattern = {
  id: string;
  name?: string;
  grid: BinaryGrid;
};
```

---

## 3.2 드로잉 인터랙션

Pointer Events 사용을 권장합니다.

```ts
let drawing = false;
let paintValue: 0 | 1 = 1;

grid.addEventListener("pointerdown", e => {
  drawing = true;
  grid.setPointerCapture(e.pointerId);
});

grid.addEventListener("pointermove", e => {
  if (!drawing) return;
  paintCellAt(e.clientX, e.clientY, paintValue);
});

grid.addEventListener("pointerup", () => {
  drawing = false;
});
```

태블릿에서 touch 이벤트를 별도로 처리하지 않아도 됩니다.

---

## 3.3 이진화 표현

UI는 그림과 숫자를 항상 같은 인덱스로 대응시켜야 합니다.

```
row 0 col 0 → cells[0]
row 0 col 1 → cells[1]
...
```

hover/focus 시 양쪽을 함께 강조하면 좋습니다.

---

## 3.4 RLE

```ts
export function encodeBinaryRLE(bits: Uint8Array): number[] {
  const out: number[] = [];
  let expected: 0 | 1 = 0;
  let count = 0;

  for (const bit of bits) {
    if (bit === expected) {
      count++;
    } else {
      out.push(count);
      expected = expected === 0 ? 1 : 0;
      count = 1;
    }
  }

  out.push(count);
  return out;
}
```

### 검증

항상

```
sum(rle) === width * height
```

이어야 합니다.

---

## 3.5 숫자 보고 그리기 게임

### 상태

```ts
type ReconstructionState = {
  target: BinaryGrid;
  answer: BinaryGrid;
  checked: boolean;
};
```

### 채점

```ts
type CellError =
  | "correct"
  | "false-positive"
  | "false-negative";
```

두 오류를 서로 다른 시각 기호로 표시합니다.

색상만 쓰지 말고

- + 표시: 칠하면 안 되는 칸
- ○ 표시: 칠해야 했던 칸

등의 모양도 함께 사용하면 접근성이 좋아집니다.

---

# 4. 아날로그 선 → 디지털 격자

## 4.1 구현 전략

1. 자유선 canvas
2. 임시 offscreen canvas
3. N×N로 축소
4. 픽셀 평균 계산
5. 임계값 기반 이진화

```ts
function rasterizeToGrid(
  source: HTMLCanvasElement,
  size: number,
  threshold = 200
): Uint8Array {
  const off = new OffscreenCanvas(size, size);
  const ctx = off.getContext("2d")!;
  ctx.drawImage(source, 0, 0, size, size);

  const data = ctx.getImageData(0, 0, size, size).data;
  const out = new Uint8Array(size * size);

  for (let i = 0; i < out.length; i++) {
    const r = data[i * 4];
    const g = data[i * 4 + 1];
    const b = data[i * 4 + 2];
    const gray = 0.299 * r + 0.587 * g + 0.114 * b;
    out[i] = gray < threshold ? 1 : 0;
  }

  return out;
}
```

Safari 구형 지원이 필요하면 OffscreenCanvas 대신 일반 canvas를 숨겨서 사용합니다.

---

# 5. 사진 → 숫자

## 5.1 전처리

```
Image File
→ orientation correction
→ center crop
→ resize N×N
→ grayscale
→ Uint8Array
```

### 주의

휴대전화 사진은 EXIF orientation 때문에 회전될 수 있습니다.

가능하면 브라우저의 `createImageBitmap()` 또는 표준 디코딩 흐름을 사용하고 실제 기기 테스트가 필요합니다.

---

# 6. 몬스터 분류 카드 구현

## 6.1 Feature schema

```ts
type Monster = {
  id: string;
  horn: "none" | "one" | "two";
  eyes: 1 | 2 | 3;
  legs: 0 | 2 | 4;
  color: "red" | "blue" | "yellow";
  wings: boolean;
  tail: boolean;
  mouth: "smile" | "flat" | "fang";
};
```

실제 조합 수는 너무 커지지 않게 제한해야 합니다.

---

## 6.2 카드 생성

### 권장

- seed 기반 난수
- 중복 제거
- 특정 특징이 한쪽 팀에만 몰리는 것을 의도적으로 만들 수 있는 preset 제공

```ts
generateMonsters({
  count: 16,
  activeFeatures: ["horn", "eyes", "wings"],
  seed: 42
});
```

---

# 7. 규칙 후보 엔진

교육용으로는 신경망보다 **설명 가능한 규칙 후보 집합**이 적합합니다.

## 7.1 단일 조건 후보

예:

```
wings === true
horn === "two"
eyes >= 2
color === "red"
```

## 7.2 AND 조건 후보

```
wings === true AND horn === "two"
```

## 7.3 후보 생성

```ts
type Rule = {
  id: string;
  description: string;
  predict(monster: Monster): "A" | "B";
};
```

---

## 7.4 학습

```ts
function fitVersionSpace(
  rules: Rule[],
  examples: LabeledMonster[]
): Rule[] {
  return rules.filter(rule =>
    examples.every(ex =>
      rule.predict(ex.monster) === ex.label
    )
  );
}
```

UI에서는

- 남은 생각
- 새로 버린 생각
- 정답 규칙 포함 여부

를 실시간 표시합니다.

---

# 8. 잘못된 레이블 실험

## 8.1 구조

```ts
type LabeledMonster = {
  monster: Monster;
  displayedLabel: "A" | "B";
  originalLabel: "A" | "B";
  corrupted: boolean;
};
```

교사/학생에게는 “이름표 실수”라는 친숙한 표현을 사용하고, 교사용 설명에서 label noise와 연결합니다.

---

# 9. 가위바위보 모델 구현

## 9.1 데이터 구조

```ts
type Label = "rock" | "paper" | "scissors";

type Sample = {
  label: Label;
  pixels: Uint8Array; // length 576
};

type PrototypeModel = {
  size: 24;
  prototypes: Record<Label, Float32Array>;
  counts: Record<Label, number>;
};
```

---

# 10. 카메라

```ts
const stream = await navigator.mediaDevices.getUserMedia({
  video: {
    facingMode: "user",
    width: { ideal: 640 },
    height: { ideal: 480 }
  },
  audio: false
});
```

### 필수 안전 처리

- 페이지 숨김 시 stop
- route 이동 시 stop
- “카메라 끄기” 제공
- 모든 track에 `stop()`

```ts
function stopCamera(stream: MediaStream) {
  for (const track of stream.getTracks()) {
    track.stop();
  }
}
```

`visibilitychange`, `pagehide`도 처리합니다.

---

# 11. 전처리

24×24 grayscale로 고정합니다.

```ts
function preprocess(
  source: CanvasImageSource,
  size = 24
): Uint8Array
```

### 권장 추가

학생이 실제로 어떤 영역이 모델 입력으로 잘렸는지 알 수 있도록 **입력 ROI 네모**를 카메라 위에 표시합니다.

---

# 12. 평균 프로토타입 학습

```ts
function averageSamples(samples: Uint8Array[]): Float32Array {
  const mean = new Float32Array(samples[0].length);

  for (const s of samples) {
    for (let i = 0; i < s.length; i++) {
      mean[i] += s[i];
    }
  }

  for (let i = 0; i < mean.length; i++) {
    mean[i] /= samples.length;
  }

  return mean;
}
```

이 평균 배열을 24×24 canvas로 다시 그리면 학생에게 “AI가 기억한 평균 모습”을 보여 줄 수 있습니다.

---

# 13. 유사도 계산

## 13.1 가장 단순한 방식: MSE

```ts
function mse(a: ArrayLike<number>, b: ArrayLike<number>) {
  let sum = 0;

  for (let i = 0; i < a.length; i++) {
    const d = a[i] - b[i];
    sum += d * d;
  }

  return sum / a.length;
}
```

낮을수록 비슷합니다.

## 13.2 학생 UI용 score 변환

```ts
score = 1 / (1 + normalizedMse)
```

또는 0~100%로 정규화합니다.

### 주의

“확률”이라고 부르면 안 됩니다.

이 값은 모델의 확률이 아니라 **유사도 점수**입니다.

---

# 14. 시험과 confusion matrix

```ts
type ConfusionMatrix = Record<
  Label,
  Record<Label, number>
>;
```

실제 정답을 먼저 선택한 뒤 모델 판정을 기록합니다.

```ts
function recordPrediction(
  matrix: ConfusionMatrix,
  truth: Label,
  predicted: Label
) {
  matrix[truth][predicted]++;
}
```

---

# 15. 모델 망가뜨리기

이 기능은 반드시 별도 실험 함수로 둡니다.

```ts
reduceOneClassTo(samples, label, 5)
halveAllClasses(samples)
restoreSamples(snapshot)
```

### 더 추가할 수 있는 실험

- 한 클래스만 배경 색 다르게 하기
- 특정 클래스만 한 학생 손으로 수집
- 밝기 변화
- 손 크기 변화
- 일부러 중앙에서 벗어나기

---

# 16. 모델 저장

원본 사진은 저장하지 않습니다.

## 저장 권장 구조

```ts
type StoredModel = {
  id: string;
  name: string;
  createdAt: string;
  size: 24;
  prototypes: {
    rock: number[];
    paper: number[];
    scissors: number[];
  };
  counts: Record<Label, number>;
  lastScore?: number;
};
```

### 저장소

- 모델 1~10개 정도: localStorage도 가능
- 장기적으로 기록이 늘어날 경우: IndexedDB 권장

---

# 17. 저작권 판정단 구현

## 17.1 데이터 중심 설계

```ts
type LicenseCondition =
  | "attribution"
  | "non-commercial"
  | "no-derivatives"
  | "share-alike";
```

학습 카드, 퀴즈, 짝 맞추기, 스티커 생성이 **같은 데이터 원본**을 공유하도록 합니다.

```
licenses.ts
  ↓
learn screen
quiz screen
matching game
sticker maker
print worksheet
```

이렇게 해야 용어가 화면마다 달라지는 오류를 막을 수 있습니다.

---

# 18. 인쇄 시스템

## 18.1 브라우저 인쇄

```css
@media print {
  nav,
  .no-print {
    display: none !important;
  }

  .print-page {
    break-after: page;
  }
}
```

## 18.2 A4

```css
@page {
  size: A4;
  margin: 12mm;
}
```

학생용 활동지는 브라우저 스크린 UI를 그대로 인쇄하지 말고 별도 print layout을 만듭니다.

---

# 19. 보안 헤더

Cloudflare Pages의 `_headers` 파일 예시:

```
/*
  X-Content-Type-Options: nosniff
  Referrer-Policy: no-referrer
  Permissions-Policy: camera=(self), microphone=(), geolocation=(), payment=()
  X-Frame-Options: DENY
  Content-Security-Policy: default-src 'self'; img-src 'self' blob: data:; media-src 'self' blob:; script-src 'self'; style-src 'self' 'unsafe-inline'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'; form-action 'self'
```

가위바위보 외 페이지에서 카메라 자체를 막고 싶다면 경로별 header 정책을 나누는 것도 가능합니다.

---

# 20. 오프라인 지원

원본 사이트는 주요 앱이 외부 서비스에 의존하지 않는다고 설명합니다.

우리 구현에서는 한 단계 더 나아가 PWA 캐시를 고려할 수 있습니다.

### 캐시 대상

- HTML
- JS
- CSS
- 아이콘
- 예시 이미지
- 학습용 정적 데이터

단, 카메라는 HTTPS 환경이 필요합니다.

---

# 21. 테스트 체크리스트

## 픽셀

- [ ] 8×8에서 정확히 64개 값
- [ ] 흑백 반전 후 모든 bit 반전
- [ ] RLE 합계가 셀 수와 동일
- [ ] 모바일 드래그 칠하기
- [ ] 인쇄 시 잘림 없음

## 몬스터

- [ ] 같은 seed는 같은 카드 생성
- [ ] 학습 예시가 늘면 후보 규칙이 늘어나지 않음
- [ ] 잘못된 레이블에서 정답 규칙이 제거될 수 있음
- [ ] 시험 카드는 학습 집합과 분리

## RPS

- [ ] 카메라 종료 시 media track stop
- [ ] 원본 프레임 저장 안 함
- [ ] 24×24 = 576
- [ ] 클래스 평균 정확성
- [ ] confusion matrix 합계 = 시험 횟수
- [ ] 저장 후 새로고침 복원
- [ ] 저장 삭제

## 저작권

- [ ] 모든 용어가 공통 데이터에서 렌더링
- [ ] 뒤집기 게임 상태 초기화
- [ ] 스티커 출력 A4 배치
- [ ] 입력 문자열 HTML escape

---

# 22. 교육적 설명 레이어

학생 UI와 교사용 설명을 분리하는 것이 좋습니다.

### 학생에게

> AI가 여러 사진에서 공통점을 찾아 기억했어요.

### 교사에게

> 이 구현은 24×24 grayscale 이미지의 클래스별 평균 prototype을 만들고, 현재 입력과의 거리로 분류하는 단순화 모델입니다.

이렇게 해야 학생에게는 이해 가능한 설명을 제공하면서도 교사의 기술적 오개념을 줄일 수 있습니다.

---

# 23. 확장 아이디어

## 23.1 음성 데이터

- 1초 소리를 waveform으로 표시
- 일정 구간으로 샘플링
- 평균 음량/주파수 특징 비교

## 23.2 센서 데이터

- micro:bit 가속도 CSV
- 여러 동작의 센서 패턴 분류

## 23.3 분류 경계 시각화

몬스터 특징 두 개를 x/y축으로 배치해 분류 규칙을 시각화할 수 있습니다.

## 23.4 데이터 카드

학생이 모델 저장 시 다음을 함께 기록:

- 누가 모았나: 별명
- 몇 장인가
- 어떤 환경인가
- 어떤 경우에 잘 틀리는가

이는 Model Card의 초등 버전이 될 수 있습니다.

---

# 24. 구현 우선순위

### Phase 1
픽셀 스튜디오 핵심
- grid
- binary
- RLE
- reconstruction

### Phase 2
몬스터 분류
- feature generator
- hypothesis elimination
- train/test split

### Phase 3
가위바위보
- camera
- preprocess
- prototype classifier
- confusion matrix

### Phase 4
기록·인쇄·보안
- model store
- print
- CSP
- privacy page

### Phase 5
저작권·책임
- license data
- quiz
- matching
- sticker

---

# 결론

이 서비스의 핵심을 복제하려면 외형보다 다음 구조를 가져와야 합니다.

```
보이는 데이터
→ 학생의 조작
→ 내부 계산을 시각화
→ 일부러 실패
→ 원인 찾기
→ 다시 개선
→ 교실 기록/인쇄
```

특히 AI 교육 웹앱에서는 “정답을 잘 맞히는 모델”보다 **학생이 왜 맞고 왜 틀렸는지 추론할 수 있는 모델**이 더 교육적입니다.
