---
title: 성능 테스트 리포트
type: Docs
created: 2026-06-10T22:17
updated: 2026-06-12T02:57
completed: true
tags:
  - Performance_Test
---
## 문서 개요

본 테스트 리포트는 **CPU 기반 Particle System**과 **GPU-Driven Particle System**의 구조적 차이와 성능 특성을 비교 분석하기 위해 작성되었습니다.

**CPU 기반 Particle System**은 CPU에서 파티클 시뮬레이션 및 빌보드 데이터 생성을 수행한 뒤, 매 프레임 Instance Buffer를 GPU로 업로드하여 렌더링하는 구조입니다.

반면 **GPU-Driven Particle System**은 파티클 데이터를 VRAM에 유지한 채로 Compute Shader를 통해 파티클 시뮬레이션과 생존 파티클 필터링, 간접 드로우 인자 생성 작업을 GPU에서 처리하는 구조입니다.

본 리포트에서는 두 방식의 데이터 흐름, 렌더링 파이프라인, 메모리 구조, 성능 측정 결과를 비교하여 파티클 수 증가에 따른 병목 지점과 GPU-Driven 렌더링 구조의 성능적 이점을 분석합니다.

---
## CPU 기반 Particle System 분석

#### 시스템 구조 및 데이터 흐름

CPU 기반 Particle System은 **파티클의 생성, 갱신 및 소멸 작업을 모두 CPU에서 수행**하는 구조입니다. 각 파티클은 CPU 메모리 상에서 위치, 속도, 크기, 색상, 수명, 생존 여부 등의 정보를 가진 상태 객체(`Particle`)를 통해 관리됩니다.

매 프레임마다 CPU는 **활성 파티클을 순회**하면서 생존 시간을 갱신하고, 수명이 끝난 파티클을 비활성화합니다. 활성 상태를 유지하는 파티클에 대해서는 현재 위치, 회전 궤도, 투명도 등을 계산하여 **최종 렌더링에 적용할 상태를 갱신**합니다.

이렇게 갱신된 파티클 데이터는 CPU에 의해 **빌보드 형태의 데이터로 변환**됩니다.
빌보드 데이터는 위치, 크기, 색상 정보를 포함하며, 이 데이터는 **Instance Buffer 형태로 GPU에 전달**됩니다.

CPU 기반 Particle System의 전체 데이터 흐름은 다음과 같이 정리할 수 있습니다.

```
파티클 상태 갱신
→ 빌보드 데이터 생성
→ Instance Buffer 업로드
→ DrawIndexedInstanced
→ 빌보드 렌더링
```

이 구조에서 **CPU**는 **파티클 상태 갱신**과 **빌보드 렌더링을 위한 인스턴스 데이터 생성**까지 담당합니다.

**GPU**는 CPU가 업로드한 **Instance Buffer**와 **고정된 Quad Vertex Buffer**를 사용하여 빌보드 렌더링 작업을 수행하며, **파티클 시뮬레이션 계산 과정에는 관여하지 않습니다.**

#### 메모리 구조

CPU 기반 Particle System에서는 CPU 메모리에서 **파티클의 원본 상태 데이터**와 **렌더링용으로 갱신된 빌보드 데이터**를 관리합니다.

`Particle` 데이터는 **파티클의 전체 시뮬레이션 상태를 저장하는 데이터**입니다.
파티클의 위치, 속도, 크기, 색상, 수명, 나이, 활성 상태 등이 여기에 포함됩니다.

`Billboard` 데이터는 **빌보드 렌더링에 필요한 최소 정보를 가지는 데이터**입니다.
빌보드 중심 위치, 크기, 색상이 여기에 포함됩니다.

```
-- CPU Memory

Particle
├─ Position
├─ Velocity
├─ Size
├─ Lifetime
├─ Age
├─ Color
└─ Active

Billboard
├─ Position
├─ Size
└─ Color
```

렌더링 단계에서는 CPU에서 생성된 `Billboard` 데이터가 **Instance Buffer 형태로 GPU에 업로드**됩니다.

VRAM에는 빌보드 렌더링을 위한 **Quad Vertex Buffer, Quad Index Buffer, Instance Buffer, Camera Constant Buffer** 등의 데이터가 상주합니다.

```
-- VRAM

Instance Buffer
├─ InstancePosition
├─ InstanceSize
└─ InstanceColor

Quad Vertex Buffer
└─ Corner

Quad Index Buffer
└─ Index

Camera Constant Buffer
├─ View
└─ Projection
```

#### 렌더링 파이프라인

CPU 기반 Particle System은 `DrawIndexedInstanced` 기반의 빌보드 렌더링 방식을 사용합니다.

GPU는 CPU가 업로드한 **Instance Buffer**를 통해 각 파티클의 위치, 크기, 색상 정보를 전달받고, 미리 생성된 **Quad Vertex Buffer**와 **Quad Index Buffer**를 사용하여 **하나의 빌보드 Quad를 구성**합니다. 이후 `DrawIndexedInstanced`를 통해 해당 Quad를 **파티클 개수만큼 반복 렌더링**합니다.

이러한 구조적 특성상, **렌더링할 파티클 수가 늘어날수록 Instance Buffer의 크기와 업로드 비용이 함께 증가**하게 됩니다.

#### 예상 병목 지점

CPU 기반 Particle System의 주요 예상 병목 지점은 크게 두 가지입니다.

1. **CPU Particle Update 비용**
	CPU는 매 프레임마다 **파티클 풀을 순회**하면서 상태 값을 갱신해야 합니다.
	파티클 수가 증가하게 되면 이 반복에 소요되는 비용은 선형적으로 증가합니다.

2. **Instance Buffer 업로드 비용**
	CPU에서 생성한 빌보드 데이터는 GPU가 사용할 수 있도록 **매 프레임 Instance Buffer 형태로 업로드**해야 합니다.
	파티클 수가 증가할수록 업로드해야 하는 데이터 크기도 증가하며, 이는 CPU/GPU 간 데이터 전송 비용의 증가로 이어집니다.

이러한 병목은 파티클 수가 적을 때는 큰 문제가 되지 않지만, **대규모 파티클 렌더링 환경**에서는 프레임 처리 소요 시간 증가의 주요 원인이 될 수 있습니다.

#### CPU 기반 Particle System의 한계

CPU 기반 Particle System은 **구조가 단순하고 디버깅이 용이**하다는 장점이 있습니다.
파티클 상태가 CPU 메모리에 존재하기 때문에 값을 직접 확인하기 쉽고, 시뮬레이션 로직을 일반 C++ 코드로 작성할 수 있습니다.

그러나 대규모 파티클 처리에 있어서는 구조적인 한계가 발생합니다.

파티클 수가 증가할수록 CPU가 처리해야 하는 갱신 연산량이 증가하고, 매 프레임 GPU로 업로드해야 하는 Instance Buffer의 크기도 증가하게 됩니다.

즉, **시뮬레이션 및 데이터 전송 비용이 모두 CPU 부담으로 누적**된다는 것이 가장 큰 한계입니다.

GPU는 많은 수의 파티클을 병렬로 처리할 수 있는 연산 자원을 가지는 하드웨어임에도, CPU 기반 Particle System에서는 **GPU가 주로 렌더링 단계에만 활용**됩니다.
파티클 시뮬레이션 자체는 CPU에서 수행되기 때문에, **GPU의 병렬 처리 능력을 충분히 활용할 수 없습니다.**

따라서 **CPU 기반 Particle System은 구현과 제어가 간단한 소규모 파티클 처리에는 적합**하지만, 대규모 파티클 렌더링 등 **파티클 수가 많아지는 상황에서는 효율이 떨어집니다.**

---
## GPU-Driven Particle System 분석

#### 시스템 구조 및 데이터 흐름

GPU-Driven Particle System은 파티클의 시뮬레이션 데이터를 VRAM에서 관리하고, **GPU의 Compute Shader를 통해 파티클의 생성, 갱신, 소멸 처리를 수행하는 구조**입니다.

CPU는 매 프레임 전체 파티클 데이터를 직접 갱신하거나 GPU로 업로드하지 않습니다.
대신 **파티클 시뮬레이션에 필요한 소량의 상수 데이터만 Constant Buffer 형태로 GPU에 전달**합니다. (`Particle Update Constant Buffer`)

GPU에서는 `GpuParticleUpdateComputeShader`가 실행되어 **`Particle Buffer`에 저장된 데이터를 직접 갱신**합니다. 이 과정에서 **생존 중인 파티클은 `Alive Index Buffer`에 기록**되며, 그 개수는 **`Alive Count Buffer`에 저장**됩니다.

이후 `GpuBillboardDrawArgsComputeShader`가 `Alive Count Buffer`를 기반으로 `Indirect Args Buffer`를 생성하고, 렌더링 단계에서 `DrawIndexedInstancedIndirect` 에 전달하여 생존 상태의 파티클을 빌보드 형태로 렌더링합니다.

GPU-Driven Particle System의 전체 데이터 흐름은 다음과 같이 정리할 수 있습니다.

```
Particle Update Constant Buffer 갱신
→ GpuParticleUpdateComputeShader 실행
→ Particle Buffer 갱신
→ Alive Index Buffer / Alive Count Buffer 생성
→ Indirect Args Buffer 생성
→ DrawIndexedInstancedIndirect
→ 빌보드 렌더링
```

이 구조에서 **CPU**는 파티클 전체 데이터를 직접 처리하지 않고, **GPU 작업을 위한 최소한의 데이터**와 **Dispatch 및 Draw 명령을 전달하는 역할을 수행**합니다.

반면 **GPU**는 **파티클 시뮬레이션, 생존 파티클 필터링, 간접 드로우 인자 생성, 빌보드 렌더링까지 대부분의 작업을 처리**합니다.

#### 메모리 구조

GPU-Driven Particle System에서는 파티클의 원본 상태 데이터가 CPU 메모리가 아닌 **VRAM의 `Particle Buffer`에 저장**됩니다.

CPU 메모리에는 **매 프레임 GPU 시뮬레이션에 필요한 상수 데이터만 존재**합니다. 이 데이터는 `ParticleUpdateBufferData` 구조체 형태로 관리되며, Compute Shader에서 사용할 `Particle Update Constant Buffer`로 업로드됩니다.

```
-- CPU Memory

ParticleUpdateBufferData
├─ EmitterPosition
├─ ParticleSize
├─ EmitterVelocity
├─ ParticleLifetime
├─ EmitterColor
├─ DeltaTime
├─ ParticleCount
├─ SpawnStartIndex
├─ SpawnCount
├─ SpawnSequenceStart
├─ SpiralArmCount
├─ OrbitStartRadius
├─ OrbitAngularVelocity
├─ OrbitRadialVelocity
├─ SpiralDepthScale
├─ SpiralDepthFrequency
├─ FadeOutStartRatio
├─ SpiralRight
├─ SpiralUp
└─ SpiralForward
```

VRAM에는 **파티클 시뮬레이션과 렌더링에 필요한 주요 버퍼들이 상주**합니다.

```
-- VRAM

Particle Update Constant Buffer
└─ ParticleUpdateBufferData

Particle Buffer
├─ Position
├─ Size
├─ Velocity
├─ Lifetime
├─ Color
├─ Age
├─ Active
├─ OrbitRadius
├─ OrbitAngle
├─ AngularVelocity
└─ RadialVelocity

Alive Index Buffer
└─ Alive Particle Index

Alive Count Buffer
└─ Alive Particle Count

Indirect Args Buffer
├─ IndexCountPerInstance
├─ InstanceCount
├─ StartIndexLocation
├─ BaseVertexLocation
└─ StartInstanceLocation

Quad Vertex Buffer
└─ Corner

Quad Index Buffer
└─ Index

Camera Constant Buffer
├─ View
└─ Projection
```

CPU 기반 Particle System과 달리, GPU-Driven Particle System에서는 **매 프레임 전체 파티클 데이터를 CPU에서 GPU로 업로드하지 않습니다.**
파티클 데이터는 VRAM에서 관리하며, **Compute Shader를 통해 해당 데이터에 직접 접근하고 갱신**합니다.

따라서 **파티클 수가 증가하더라도 CPU-GPU 간 데이터 전송량은 크게 증가하지 않으며**, 주요 비용은 **GPU 내부의 Compute Shader 처리 및 렌더링 비용으로 이동**합니다.

#### 렌더링 파이프라인

GPU-Driven Particle System은 `DrawIndexedInstancedIndirect` 기반의 빌보드 렌더링 방식을 사용합니다.

GPU-Driven Particle System에서는 CPU가 렌더링할 파티클 개수를 전달하는 대신, **Compute Shader가 생성한 `Indirect Args Buffer`를 기반으로 `DrawIndexedInstancedIndirect`가 실행**됩니다.

렌더링 단계에서 Vertex Shader는 `SV_InstanceID`를 사용하여 `Alive Index Buffer`를 참조하고, 이를 통해 **실제 렌더링 대상이 되는 생존 파티클의 인덱스를 얻습니다.** 이후 해당 인덱스를 이용해 `Particle Buffer`에서 파티클의 위치, 크기, 색상 정보를 읽어 빌보드 정점을 생성합니다.

GPU-Driven Particle System의 렌더링 흐름은 다음과 같이 정리할 수 있습니다.

```
Alive Count Buffer
→ Indirect Args Buffer 생성
→ DrawIndexedInstancedIndirect
→ Vertex Shader에서 Alive Index Buffer 참조
→ Particle Buffer에서 파티클 데이터 읽기
→ Billboard 렌더링
```

이 구조에서는 **생존 파티클 개수 계산과 렌더링 인자 생성 작업이 GPU에서 처리**되므로, CPU가 렌더링할 파티클 개수를 직접 계산하거나 `Instance Buffer`를 구성할 필요가 없어 CPU 부담이 줄어듭니다.

#### 예상 병목 지점

GPU-Driven Particle System의 주요 예상 병목 지점은 크게 두 가지입니다.

1. **GPU Compute Shader 처리 비용**
    GPU는 `Particle Buffer`의 데이터를 매 프레임 Compute Shader를 통해 갱신합니다.
    파티클 수가 증가할수록 **Compute Shader에서 처리해야 하는 스레드 수**와 **GPU 메모리 접근량이 증가**하므로, 해당 비용은 GPU 측 병목으로 작용할 수 있습니다.

2. **렌더링 및 Overdraw 비용**
    GPU-Driven Particle System은 생존 파티클만 렌더링하더라도, 최종적으로는 각 파티클을 빌보드 형태로 화면에 출력해야 합니다.
    **파티클이 많아지거나 화면상에서 많이 겹치는 경우 Vertex Shader 및 Pixel Shader 처리량, Alpha Blending 비용이 증가할 수 있습니다.**
    특히 반투명 파티클은 Overdraw가 발생하기 쉬우므로, 렌더링 단계가 병목 지점이 될 수 있습니다.

GPU-Driven Particle System은 CPU 기반 방식에 비해 CPU 부담이 줄어듭니다.
다만 전체 비용이 사라지는 것은 아니며, **성능 병목의 중심이 GPU 계산, GPU 메모리 접근, 픽셀 처리 비용으로 이동**한 것으로 볼 수 있습니다.

#### GPU-Driven Particle System의 한계

GPU-Driven 방식은 **대규모 파티클 처리에 유리하지만, 구현 복잡도가 비교적 높습니다.**

CPU 기반 방식에서는 파티클 상태를 CPU 메모리에서 직접 확인할 수 있지만, GPU-Driven 방식에서는 파티클 데이터가 VRAM에 존재하기 때문에 중간 상태를 확인하거나 디버깅하기가 상대적으로 어렵습니다.

또한 `Structured Buffer`, `RWStructuredBuffer`, `SRV`, `UAV`, `Indirect Args Buffer` 등의 다양한 GPU 리소스를 관리해야 하며, Compute Shader와 렌더링 파이프라인 사이의 데이터 흐름을 정확히 설계해야 합니다.

성능 측정 측면에서도 주의가 필요합니다. **CPU 타이머로 측정한 GPU Update Time은 실제 GPU 실행 시간이 아니라 Compute Shader Dispatch 제출 비용에 가깝습니다.**
실제 GPU 실행 시간을 정확히 측정하기 위해서는 **GPU Timestamp Query를 활용한 별도 측정이 필요**합니다.

따라서 GPU-Driven Particle System은 대량 파티클 처리와 CPU 부하 감소에는 효과적이지만, **구현 및 디버깅 난이도, 정확한 성능 측정의 복잡성이 증가한다는 한계를 가집니다.**

---
## 성능 테스트를 통한 비교 분석

#### 테스트 환경 및 조건

테스트 신뢰도를 높이기 위하여, **다음 항목을 동일하게 사용하도록 통제하였습니다.**

| 항목                 |                 설정 |
| ------------------ | -----------------: |
| **화면 해상도**         |       `1280 x 720` |
| **카메라 위치**         | `(0.0, 0.0, -2.0)` |
| **파티클 크기**         |            `0.005` |
| **파티클 형태**         |    `중심 방출형 나선 파티클` |
| **Spiral Arm 개수**  |               `8개` |
| **Blend 설정**       |               `ON` |
| **Depth Test 설정**  |               `ON` |
| **Depth Write 설정** |              `OFF` |
| **Emitter 초기 위치**  |  `(0.0, 0.0, 0.0)` |
| **Emitter 초기 속도**  | `(0.0, 0.25, 0.0)` |
| **렌더링 방식**         |          `빌보드 렌더링` |
CPU 기반 방식과 GPU-Driven 방식 모두 **파티클 이동 및 Spiral 계산에 있어 동일한 로직을 사용**하였습니다.

#### 측정 항목

성능 측정 항목은 다음과 같습니다.

| 항목                    |                 의미 |
| --------------------- | -----------------: |
| **Update Time**       |  파티클 갱신 단계에 소요된 시간 |
| **Render Time**       | 파티클 렌더링 단계에 소요된 시간 |
| **Frame Time**        | 한 프레임에 대한 전체 처리 시간 |
| **FPS**               |           초당 프레임 수 |
- **CPU 기반 Particle System**
	`Update Time`은 **CPU에서 파티클의 생성, 갱신, 소멸 처리를 수행**하고, **활성 파티클을 렌더링용 빌보드 데이터로 변환**하는 데 소요된 시간입니다.
	
	`Render Time`은 **CPU에서 생성된 빌보드 데이터를 Instance Buffer로 GPU에 업로드**하고, **렌더링에 필요한 요소들을 바인딩**한 뒤 **`DrawIndexedInstanced` 명령을 제출**하는 데 소요된 시간입니다.

- **GPU-Driven Particle System**
	`Update Time`은 GPU Timestamp Query 기준으로, **Compute Shader가 파티클 데이터를 갱신하고 Alive Index 및 Alive Count Buffer를 생성하는 데 걸린 GPU 실행 시간**입니다.
	
	`Render Time`은 GPU Timestamp Query 기준으로, **Indirect Args Buffer 생성과 `DrawIndexedInstancedIndirect` 기반 빌보드 렌더링에 소요된 GPU 실행 시간**입니다.

즉, CPU 기반 방식의 측정값은 **CPU가 파티클 데이터와 렌더링 명령을 준비하는 비용**에 가깝고, GPU-Driven 방식의 측정값은 **GPU에서 Compute Shader 기반 파티클 갱신, Indirect Args 생성, Indirect Draw 기반 렌더링 작업이 실행되는 비용**에 가깝습니다.

#### 측정 결과 및 결과 해석석

| No. | 목표 활성 파티클 수 | Spawn Rate | Lifetime |                    측정 결과 CSV 파일 |
| :-- | ----------: | ---------: | -------: | ------------------------------: |
| 1   |     10,000개 |     500개/초 |      20초 | ![[particle_performance_1.csv]] |
| 2   |    100,000개 |   5,000개/초 |      20초 | ![[particle_performance_2.csv]] |
| 3   |    300,000개 |  15,000개/초 |      20초 | ![[particle_performance_3.csv]] |
| 4   |    600,000개 |  30,000개/초 |      20초 | ![[particle_performance_4.csv]] |
**Lifetime을 20초로 고정하고 Spawn Rate를 조정**하여 목표 활성 파티클 수를 조정하였습니다.

테스트 완료 후 각 테스트에 대한 **유효 데이터(안정화 이후 중간 구간 30개 데이터)를 선정**하고, **평균 데이터를 산출**하였습니다.

- **테스트 No.1 (목표 활성 파티클 수 10,000개) 유효 데이터 평균**
   
|                         |    FPS     | Update (ms) | Render (ms) | Frame (ms) | Valid Rows |
| ----------------------- | :--------: | :---------: | :---------: | :--------: | :--------: |
| **CPU Particle System** | 239.957394 |  3.0270855  |  0.160955   | 4.1674065  |   26-55    |
| **GPU Particle System** | 239.898457 |  0.379791   |  0.0178325  | 4.1684305  |   86-115   |
   
- **테스트 No.2 (목표 활성 파티클 수 100,000개) 유효 데이터 평균**

|                         |    FPS     | Update (ms) | Render (ms) | Frame (ms) | Valid Rows |
| ----------------------- | :--------: | :---------: | :---------: | :--------: | :--------: |
| **CPU Particle System** | 70.3603805 | 12.6928315  |  1.392532   | 14.2125555 |   26-55    |
| **GPU Particle System** | 239.852458 |  0.364943   |   0.10048   |  4.16923   |   86-115   |
   
- **테스트 No.3 (목표 활성 파티클 수 300,000개) 유효 데이터 평균**

|                         |    FPS     | Update (ms) | Render (ms) | Frame (ms) | Valid Rows |
| ----------------------- | :--------: | :---------: | :---------: | :--------: | :--------: |
| **CPU Particle System** | 24.5041025 | 36.3716455  |  4.2803435  | 40.809752  |   26-55    |
| **GPU Particle System** |  239.9305  |   0.38128   |  0.279223   | 4.1678735  |   86-115   |

- **테스트 No.4 (목표 활성 파티클 수 600,000개) 유효 데이터 평균**

|                         |    FPS     | Update (ms) | Render (ms) | Frame (ms) | Valid Rows |
| ----------------------- | :--------: | :---------: | :---------: | :--------: | :--------: |
| **CPU Particle System** | 13.525203  | 65.5107585  |  8.2513555  | 73.9360555 |   26-55    |
| **GPU Particle System** | 239.900083 |   0.39296   |  0.549824   | 4.1684025  |   86-115   |

산출한 평균값 데이터를 기반으로, **파티클 부하 단계에 따른 측정 지표별 성능 변화 그래프를 작성**하였습니다.

- **FPS 변화 그래프**
  ![[파티클 부하 단계에 따른 FPS 변화 그래프.png]]

	측정 결과, **목표 활성 파티클 수가 10,000개일 땐 두 방식 모두 240에 근사한 수치를 나타내 유사한 성능을 보였습니다.**
	해당 구간에서는 파티클 수가 적어 CPU 기반 방식에서도 병목이 크게 발생하지 않았기 때문에 두 시스템 간 성능 차이가 거의 나타나지 않은 것으로 보입니다.
	
	그러나 **목표 활성 파티클 수가 100,000개 이상으로 증가하면서, CPU 기반 Particle System의 FPS가 급격하게 감소하였습니다.**
	
	반면 **GPU-Driven 방식은 모든 테스트 구간에서 240 FPS 수준을 유지하였습니다.**
	이는 해당 방식이 **파티클 상태 업데이트와 렌더링 준비 작업을 GPU에서 처리**하여, 파티클 수 증가에 따른 CPU 부담을 크게 줄였기 때문으로 해석할 수 있습니다.

- **Update Time 변화 그래프**
  ![[파티클 부하 단계에 따른 Update Time 변화 그래프.png]]
  
	**CPU 기반 방식의 Update Time은 10,000개에서 약 3.03ms였으나, 600,000개에서는 약 65.51ms까지 증가**하였습니다.
	이는 파티클 수가 증가할수록 CPU가 처리해야 하는 생성, 갱신, 소멸, Billboard 데이터 생성 비용이 함께 증가하기 때문으로 해석할 수 있습니다.
	
	반면 **GPU-Driven 방식의 Update Time은 전 구간에서 0.36-0.39ms 수준으로 유지**되는 것이 관측되었습니다.
	이를 통해 **Compute Shader 기반 병렬 갱신 구조가 CPU 기반 순차 처리 방식보다 대규모 파티클 처리에 유리**하다는 것을 확인할 수 있었습니다.
   
- **Render Time 변화 그래프**
  ![[파티클 부하 단계에 따른 Render Time 변화 그래프.png]]

	**CPU 방식의 Render Time은 10,000개에서 약 0.16ms였지만, 600,000개에서는 약 8.25ms까지 증가**하였습니다.
	이는 활성 파티클 수가 증가할수록 CPU에서 생성한 Billboard 데이터를 Instance Buffer로 업로드하는 비용과 렌더링 명령 준비 비용이 함께 증가하기 때문입니다.
	
	**GPU-Driven 방식의 Render Time도 파티클 수 증가에 따라 0.02ms에서 약 0.55ms까지 증가했지만, CPU 방식에 비해 증가 폭은 훨씬 적은 수준으로 확인**되었습니다.

- **Frame Time 변화 그래프**
  ![[파티클 부하 단계에 따른 Frame Time 변화 그래프.png]]

	**CPU 기반 방식에서의 Frame Time은 10,000개에서 약 4.17ms였으나, 600,000개에서는 약 73.94ms까지 증가**하였습니다.
	
	반면 **GPU-Driven 방식에서는 모든 구간에서 약 4.17ms 수준을 유지**하였습니다.
	이 결과는 GPU-Driven 방식이 현재 테스트 범위 내에서는 프레임 시간 증가를 효과적으로 억제하고 있음을 반증합니다.

*※ GPU-Driven 방식의 FPS와 Frame Time이 모든 테스트 구간에서 240FPS, 4.17ms 수준으로 거의 일정하게 나타난 것은 실제 최대 성능이 240FPS라는 의미라기보다, **테스트 환경에서 프레임 제한 또는 VSync와 같은 외부 요인에 의해 FPS가 제한되었을 가능성**이 있습니다.*

*따라서 **GPU-Driven 방식은 현재 테스트 조건에서 성능 한계에 도달하지 않았고, 프레임 제한에 가까운 상태로 동작한 것으로 해석하는 것이 적절**하다고 보여집니다.*