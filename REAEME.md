# Shadow Catcher Plugin — 사용 가이드

씬에 존재하는 오브젝트의 그림자만 투명한 평면 위에 실시간으로 렌더링하는 플러그인.
그림자가 없는 영역은 완전히 투명하게 처리되어 다른 배경과 자연스럽게 합성됩니다.

---

## 설치 방법

### 1. Plugins 폴더에 복사

ZIP 파일 압축 해제 후 프로젝트 루트 폴더의 `Plugins/` 안에 넣습니다.
`Plugins/` 폴더가 없으면 직접 생성하세요.

```
MyProject/
├── Content/
└── Plugins/
    └── ShadowCatcher/        ← 여기에
        ├── ShadowCatcher.uplugin
        └── Content/
```

### 2. 프로젝트 열기

UE5에서 프로젝트를 열면 플러그인 감지 팝업이 뜹니다.
**Yes** → 에디터 재시작.

팝업이 뜨지 않으면: **Edit → Plugins → "ShadowCatcher" 검색 → 체크박스 ON → 재시작**

![alt text](image.png)

### 3. Content Browser에서 확인

Content Browser 우측 하단 **Settings → Show Plugin Content** ON.
`ShadowCatcher Content` 폴더가 보이면 준비 완료.

---

## 씬에 배치하기

### Step 1. BP_ShadowCatcher 배치

`ShadowCatcher Content` 폴더에서 `BP_ShadowCatcher`를 Viewport로 드래그합니다.

배치 후 **Details 패널**에서 조절 가능한 항목:

| 항목 | 설명 | 기본값 |
|---|---|---|
| **Catcher Scale** | 그림자를 받을 평면의 크기 | 5.0 |
| **Capture Height** | 위에서 내려다보는 카메라 높이 | 600.0 |
| **Capture Resolution** | 캡처 해상도 (높을수록 선명, 성능 소모 증가) | 512 |

> Catcher Scale을 바꾸면 평면 크기와 캡처 범위가 자동으로 동기화됩니다.

### Step 2. BP_ShadowCaster 배치

그림자를 드리울 오브젝트를 `BP_ShadowCaster`로 대체하거나,
기존 메시를 BP_ShadowCaster의 ShadowMesh 슬롯에 넣습니다.

배치하는 순간 자동으로 BP_ShadowCatcher의 캡처 목록에 등록됩니다.

### Step 3. Play

**Play 버튼**을 누르면 자동으로 초기화됩니다.
별도 설정 없이 그림자가 평면 위에 표시됩니다.

---

## 에디터에서 그림자가 보이지 않는 이유

에디터에서는 평면이 투명하게 보입니다. **이것은 정상 동작입니다.**

### 왜 그런가

이 플러그인은 에디터가 아닌 **런타임(Play)**에서 초기화됩니다.

```
에디터 상태:
  BP_ShadowCatcher 배치됨
  → 렌더 타겟(RT) 아직 없음
  → 머티리얼에 연결된 텍스처 없음
  → 평면이 투명하게 보임  ← 정상

Play 버튼 누름:
  → RT 자동 생성
  → 머티리얼에 RT 연결
  → SceneCapture 캡처 시작
  → 그림자 표시됨  ← 정상
```

### 왜 에디터에서 바로 작동하지 않게 설계했는가

에디터에서 SceneCapture를 항상 켜두면, Details 패널의 슬라이더를 조절할 때마다
액터가 재생성되면서 GPU 렌더링 리소스가 매번 새로 할당됩니다.
이를 빠르게 반복하면 GPU 메모리가 순식간에 고갈됩니다.

런타임에만 활성화함으로써 이 문제를 근본적으로 차단합니다.

---

## 여러 개 배치

BP_ShadowCatcher를 여러 개 배치해도 각자 독립적으로 작동합니다.

```
BP_ShadowCatcher_01  →  RT_A (자동 생성)  →  독립된 그림자 캡처
BP_ShadowCatcher_02  →  RT_B (자동 생성)  →  독립된 그림자 캡처
```

각 인스턴스가 Play 시 자신만의 렌더 타겟을 생성하기 때문에
서로 간섭하지 않습니다.

---

## 그림자 경계 품질 조절

그림자 경계의 반짝임이 보일 경우:

**Project Settings → Engine → Rendering → Shadows**
- `Shadow Map Method`: `Virtual Shadow Maps` → `Shadow Maps` 로 변경

Virtual Shadow Maps(VSM)는 UE5 기본값으로, 경계에 temporal 노이즈를 적용합니다.
전통적인 Shadow Maps로 전환하면 경계가 안정적으로 표시됩니다.

---

## 주의사항

- **Play 전에 설정할 것**: Catcher Scale, Capture Height, Capture Resolution은
  Play 중에 변경해도 즉시 반영되지 않습니다. Play 전 에디터에서 설정하세요.

- **BP_ShadowCaster 삭제 후**: BP_ShadowCatcher를 살짝 이동했다가 되돌리면
  Construction Script가 재실행되어 ShowOnly 목록이 갱신됩니다.

- **Directional Light 필요**: 그림자가 생기려면 씬에 Directional Light가 있어야 하며
  `Cast Shadows`가 켜져 있어야 합니다.
