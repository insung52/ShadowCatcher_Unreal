# UE5 Shadow Catcher — Blueprint Actor v3 (완전 자동화)

> 스크린샷 파일은 `images/` 폴더에 저장. 각 섹션 아래 플레이스홀더에 맞춰 교체.

---

## v2 대비 변경 사항

| 항목 | v2 | v3 |
|---|---|---|
| RT 에셋 | 직접 생성 필요 | 불필요, BeginPlay에서 자동 생성 |
| MI 에셋 | 직접 생성 필요 | 불필요, BeginPlay에서 자동 생성 |
| 컴포넌트 속성 수동 할당 | 배치 후 직접 할당 | 불필요, 자동 처리 |
| CaptureResolution 변수 | 없음 | 부활, Details 슬라이더로 조절 |
| 초기화 시점 | CS (에디터) | BeginPlay (런타임) |
| 에디터 그림자 프리뷰 | 가능 | 없음 (Play 시에만 동작) |
| 여러 캐처 배치 | 에셋 수동 생성 필요 | 자동 분리, 개수 제한 없음 |

---

## 전체 자동화 구조

```
에디터:
  BP_ShadowCatcher 배치 → CS: 크기/위치 설정만
  BP_ShadowCaster 배치  → CS: ShowOnly 자동 등록

Play 시작:
  각 BP_ShadowCatcher의 BeginPlay (인스턴스별 독립 실행)
    → RT 동적 생성 (인스턴스별 고유)
    → MID 동적 생성 → RT 연결
    → ShadowCatcherMesh에 MID 적용
    → ShadowCapture에 RT 연결
    → Capture Every Frame ON
    → RefreshShowOnly
```

> 캐처 여러 개 배치해도 각자 BeginPlay에서 독립 RT/MID 생성 → 충돌 없음.

---

## 사전 준비

`M_ShadowCatcher`, `M_Reference` 에셋만 있으면 됨.
RT 에셋, MI 에셋 생성 불필요.

---

### M_Reference 생성

SceneCapture 전용 기준면 머티리얼. **Default Lit + 흰색** → 씬 조명을 받아 그림자 영역이 어둡게 찍힘.
(Unlit이면 조명 무시 → 그림자가 RT에 찍히지 않음)

1. Content Browser **우클릭 → Material** → 이름: `M_Reference`
2. Details: **Blend Mode `Opaque`** (기본값 유지), Shading Model 건드리지 않음
3. 노드 구성:

```
[Constant3Vector (1.0, 1.0, 1.0)] → Base Color
```

4. **Apply** → **Save**

![M_Reference 노드 그래프](images/m_reference_nodes.png)

---

### M_ShadowCatcher 생성

RT에서 그림자 영역만 추출해 반투명으로 표시하는 머티리얼.

1. Content Browser **우클릭 → Material** → 이름: `M_ShadowCatcher`
2. Details에서 설정:

| 항목 | 값 |
|---|---|
| Blend Mode | `TranslucentGreyTransmittance` |
| Shading Model | `Unlit` |
| Two Sided | **ON** |

3. 노드 구성:

```
[TextureCoordinate] ──UV──→ [TextureSampleParameter2D]
                               Parameter Name: "ShadowCapture"
                                        │
                             [Desaturate]  ← Desaturation: 1.0
                                        │
                                  [OneMinus]
                                        │
                                    Opacity ──→ [Main Material]
```

> - 밝은 픽셀 (그림자 없음) → Desaturate → OneMinus → 0 → 투명
> - 어두운 픽셀 (그림자) → Desaturate → OneMinus → 1 → 불투명 검정

4. **Apply** → **Save**

![M_ShadowCatcher 노드 그래프](images/m_shadowcatcher_nodes.png)

---

## Part 1: BP_ShadowCatcher

### 1-1. Blueprint Actor 생성

1. Content Browser 빈 공간 **우클릭** → `Blueprint Class` → `Actor`
2. 이름: `BP_ShadowCatcher`
3. **더블클릭** → Blueprint Editor 열림

---

### 1-2. 컴포넌트 추가

**Components** 패널 상단 **`+ Add`** 버튼으로 추가. 이름 변경: F2

| 순서 | 컴포넌트 타입 | 이름 |
|---|---|---|
| 1 | `Static Mesh Component` | `ShadowCatcherMesh` |
| 2 | `Static Mesh Component` | `ReferenceMesh` |
| 3 | `Scene Capture Component 2D` | `ShadowCapture` |

![컴포넌트 패널](images/bp_components.png)

---

### 1-3. 각 컴포넌트 기본 설정

#### ShadowCatcherMesh
| 검색어 | 항목 | 값 |
|---|---|---|
| `static mesh` | Static Mesh | `Plane` |
| `material` | Material | `M_ShadowCatcher` |
| `cast shadow` | Cast Shadow | **OFF** |
| `scene capture` | Hidden in Scene Capture | **ON** |

#### ReferenceMesh
| 검색어 | 항목 | 값 |
|---|---|---|
| `static mesh` | Static Mesh | `Plane` |
| `material` | Material | `M_Reference` |
| `cast shadow` | Cast Shadow | **OFF** |
| `scene capture` | Visible in Scene Capture Only | **ON** |

#### ShadowCapture
| 검색어 | 항목 | 값 |
|---|---|---|
| `capture source` | Capture Source | `Scene Color (HDR) in RGB, Linear` |
| `projection` | Projection Type | `Orthographic` |
| `capture every frame` | Capture Every Frame | **OFF** |
| `capture on movement` | Capture on Movement | **OFF** |

> **Capture Every Frame = OFF 이유**: 에디터에서 슬라이더 조작 시 액터가 재생성되는데,
> ON 상태에서 반복되면 GPU 렌더링 리소스가 누적되어 메모리 초과 발생.
> BeginPlay에서 ON으로 전환하여 런타임에만 활성화.

Show Flags에서 아래 항목 **OFF**:
- `Atmosphere` / `Fog` / `Bloom` / `Ambient Occlusion` / `Motion Blur`
- `Lumen Global Illumination` / `Lumen Reflections` / `Reflections`
- `Screen Space Ambient Occlusion` / `Anti-Aliasing`
- `Contact Shadows` / `Distance Field Shadows`

![ShadowCapture Show Flags](images/show_flags.png)

---

### 1-4. 변수(Variable) 추가

**My Blueprint → Variables → `+`**

| 변수명 | 타입 | 기본값 | Instance Editable | 용도 |
|---|---|---|---|---|
| `CatcherScale` | `Float` | `5.0` | **ON** | Plane 크기 + Ortho Width |
| `CaptureHeight` | `Float` | `600.0` | **ON** | SceneCapture 높이 |
| `CaptureResolution` | `Integer` | `512` | **ON** | RT 해상도 (256/512/1024/2048) |
| `DynamicRT` | `Texture Render Target 2D` (Object Ref) | 없음 | OFF | 내부 초기화용 |
| `DynMat` | `Material Instance Dynamic` (Object Ref) | 없음 | OFF | 내부 초기화용 |

![Variables 패널](images/variables.png)

---

### 1-5. RefreshShowOnly 함수 작성

ShowOnly 목록을 재빌드하는 함수. SceneCapture가 캡처할 오브젝트 목록을 최신 상태로 유지.

**My Blueprint → Functions → `+` → 이름: `RefreshShowOnly`**

```
[RefreshShowOnly 진입]
        │
[Clear Show Only Components]  ← Target: ShadowCapture
        │
[Set Primitive Render Mode]   ← Target: ShadowCapture
        │                        New Render Mode: Use ShowOnly List
[Add Show Only Component]     ← Target: ShadowCapture
        │                        In Component: ReferenceMesh
[Get All Actors Of Class]     ← Actor Class: BP_ShadowCaster
        │ Out Actors
[For Each Loop]
        │ Array Element
[Add Show Only Actor]         ← Target: ShadowCapture
                                 InActor: Array Element
```

![RefreshShowOnly 함수 노드](images/refresh_show_only.png)

---

### 1-6. Construction Script 작성

**CS는 트랜스폼 연산만 담당.** RT/MID 생성은 BeginPlay에서 처리.
에디터에서 슬라이더를 드래그해도 GPU 리소스 변화 없음.

```
[Construction Script 시작]
        │
[Make Vector (CatcherScale, CatcherScale, 1.0)]
        ├──→ [Set World Scale 3D (ShadowCatcherMesh)]
        └──→ [Set World Scale 3D (ReferenceMesh)]
        │
[CatcherScale × 100] → [Set Ortho Width (ShadowCapture)]
        │
[Set Relative Location (ShadowCapture)] ← (0, 0, CaptureHeight)
        │
[Set Relative Rotation (ShadowCapture)] ← (0, -90, 0)
        │
[RefreshShowOnly (Self)]
```

![Construction Script 노드](images/construction_script.png)

---

### 1-7. BeginPlay 작성

RT/MID 초기화 + Capture Every Frame 활성화.

**Is Valid 가드**: DynamicRT가 이미 있으면 생성 skip.
BeginPlay가 레벨 스트리밍 등으로 여러 번 호출되어도 RT/MID는 최초 1회만 생성.
CS와 달리 BeginPlay는 변수값이 유지되므로 Is Valid 가드가 정상 작동함.

**Event Graph → Event BeginPlay 연결:**

```
[Event BeginPlay]
        │
[Is Valid (DynamicRT)?]
    │
    ├── TRUE ──→ (이미 초기화됨, 생성 skip)
    │
    └── FALSE ─→ [Create Render Target 2D]  ← Width/Height: CaptureResolution
                                               Format: RTF RGBA8
                  │
                  [Set DynamicRT]
                  │
                  [Create Dynamic Material Instance]  ← Target: ShadowCatcherMesh
                                                         Source Material: M_ShadowCatcher
                  │
                  [Set DynMat]
                  │
                  [Set Texture Parameter Value]  ← Target: DynMat
                  ("Set Texture Parameter MID"   Parameter Name: "ShadowCapture"
                   로 표시될 수 있음)             Value: DynamicRT
                  │
                  [Set Material]  ← Target: ShadowCatcherMesh
                                    Element Index: 0
                                    Material: DynMat
                  │
                  [Set Texture Target]  ← Target: ShadowCapture
                                          New Value: DynamicRT
        │ ← TRUE / FALSE 합류
[Set Capture Every Frame]  ← Target: ShadowCapture
                               Value: TRUE
        │
[RefreshShowOnly (Self)]
```

![BeginPlay 노드 전체](images/begin_play_full.png)
![BeginPlay Is Valid 분기 - FALSE 경로](images/begin_play_false_branch.png)
![BeginPlay 합류 이후](images/begin_play_merge.png)

---

### 1-8. 컴파일 및 저장

**Compile** → **Save**. 초록 체크 확인.

---

### 1-9. 월드에 배치

Content Browser에서 `BP_ShadowCatcher` → Viewport 드래그.

Details에서 조절 가능한 항목:

| 변수 | 효과 | 비고 |
|---|---|---|
| `CatcherScale` | Plane 크기 + Ortho Width 자동 동기화 | 에디터에서 즉시 반영 |
| `CaptureHeight` | SceneCapture 높이 | 에디터에서 즉시 반영 |
| `CaptureResolution` | RT 해상도 | Play 전에 설정할 것 |

![레벨에 배치된 BP_ShadowCatcher](images/placed_in_level.png)

---

## Part 2: BP_ShadowCaster

### 2-1. Blueprint Actor 생성

1. Content Browser 우클릭 → `Blueprint Class` → `Actor`
2. 이름: `BP_ShadowCaster`
3. 더블클릭 → Blueprint Editor

---

### 2-2. 컴포넌트 추가

`+ Add` → `Static Mesh Component` → 이름: `ShadowMesh`

---

### 2-3. ShadowMesh 설정

| 검색어 | 항목 | 값 |
|---|---|---|
| `static mesh` | Static Mesh | `Cube` (교체 가능) |
| `cast shadow` | Cast Shadow | **ON** |
| `scene capture` | Hidden in Scene Capture | **ON** |
| `hidden shadow` | Cast Hidden Shadow | **ON** |

> **세 플래그의 역할**
> - ShowOnly 목록에 있어야 SceneCapture 캡처에 참여 가능
> - `Hidden in Scene Capture = ON` → 메시 색상은 캡처에서 제외
> - `Cast Hidden Shadow = ON` → 그림자는 캡처에 유지

![ShadowMesh 플래그 설정](images/shadow_mesh_flags.png)

---

### 2-4. Construction Script

```
[Construction Script 시작]
        │
[Get All Actors Of Class] ← Actor Class: BP_ShadowCatcher
        │ Out Actors
[For Each Loop]
        │ Array Element
[Cast To BP_ShadowCatcher]
        │ As BP_ShadowCatcher
[RefreshShowOnly] ← Target: As BP_ShadowCatcher
```

> BP_ShadowCaster를 배치하거나 이동하면 CS가 실행되어
> 씬의 모든 BP_ShadowCatcher ShowOnly 목록이 자동 갱신됨.

![BP_ShadowCaster Construction Script](images/caster_cs.png)

---

### 2-5. 컴파일 및 저장

**Compile** → **Save**

---

## 최종 워크플로우

```
1. BP_ShadowCatcher 월드에 드래그
   → Details에서 CatcherScale, CaptureHeight, CaptureResolution 설정
   → 에디터에서 크기/위치 미리보기 가능 (그림자는 Play 후 확인)

2. BP_ShadowCaster 월드에 드래그
   → ShowOnly 자동 등록

3. Play 버튼
   → 각 BP_ShadowCatcher가 독립 RT/MID 자동 생성
   → 그림자 캡처 시작

4. 여러 개 배치
   → 각자 고유한 RT/MID 자동 생성, 충돌 없음
```

---

## 트러블슈팅

| 증상 | 원인 | 해결 방법 |
|---|---|---|
| 에디터에서 평면이 투명 | 정상 동작. BeginPlay 전 RT 미연결 | Play 눌러서 확인 |
| Play 해도 평면이 투명 | Capture Every Frame이 OFF 유지 | BeginPlay의 Set Capture Every Frame 노드 확인 |
| Play 해도 평면이 완전히 검음 | DynamicRT 미생성 또는 Set Texture Target 누락 | BeginPlay FALSE 경로 노드 전체 확인 |
| 여러 캐처가 같은 그림자 공유 | DynMat 파라미터 미연결 | Set Texture Parameter Value 노드 확인 |
| Scale 바꿔도 크기 안 바뀜 | CS 실행 핀 연결 누락 | 흰색 삼각형 핀 순서 확인 |
| Cube 색상이 RT에 찍힘 | Cast Hidden Shadow 미설정 | ShadowMesh 두 플래그 모두 ON 확인 |
| BP_ShadowCaster 배치해도 ShowOnly 미등록 | CS에 RefreshShowOnly 누락 | BP_ShadowCaster CS 노드 확인 |
| ShadowCapture가 아래를 안 봄 | Rotation 설정 누락 | CS의 Set Relative Rotation Pitch = -90 확인 |
| GPU 메모리 계속 증가 | Capture Every Frame이 에디터에서 ON | ShadowCapture 컴포넌트 기본값 OFF 확인 |
| Compile 옆 느낌표 | 노드 연결 오류 | 빨간 오류 노드, 핀 타입 확인 |
