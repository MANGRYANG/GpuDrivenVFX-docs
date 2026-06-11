---
title: GpuDrivenVFX 프로젝트 개요
type: Docs
created: 2026-05-14T21:28
updated: 2026-05-19T12:25
completed: true
tags:
  - Overview
---
**GpuDrivenVFX 프로젝트**는 `C++`과 `DirectX 11`을 사용하여 **CPU 기반의 입자 시뮬레이션과 GPU Compute Shader 기반의 입자 시뮬레이션**을 모두 구현하고, 두 방식의 구조적 차이와 성능 병목을 비교분석하는 것을 목표로 기획되었습니다.

프로젝트의 **주요 목표**는 다음과 같습니다.

- `DirectX 11` 렌더링 파이프라인을 직접 구성해보는 경험
- **CPU Particle System**과 **GPU Particle System**의 구조 차이에 대한 이해
- **HLSL Compute Shader**를 통해 입자 시뮬레이션을 GPU에서 처리
- 성능 측정 데이터를 기반으로 병목 관측
- **게임 클라이언트 VFX 시스템**의 최적화 관점을 이해

예상되는 프로젝트 산출물의 **주요 기능**은 다음과 같습니다.

- **DirectX 11 기반 렌더링 프레임워크**
    - Win32 Window 생성
    - D3D11 Device / DeviceContext / SwapChain 초기화
    - RenderTargetView, DepthStencilView, Viewport 구성
    - 기본 Render Loop 구현

- **기본 3D 렌더링 파이프라인**
    - Vertex Shader / Pixel Shader 작성
    - Vertex Buffer / Index Buffer 사용
    - Constant Buffer를 통한 행렬 전달
    - World / View / Projection Matrix 적용
    - 간단한 카메라 시스템 구성

- **CPU 기반 Particle System**
    - CPU에서 입자 생성, 갱신, 소멸 처리
    - 위치, 속도, 수명, 색상, 크기 등 입자 상태 관리
    - 매 프레임 갱신된 입자 데이터를 GPU로 업로드
    - Billboard 방식의 입자 렌더링

- **GPU Compute Shader 기반 Particle System**
    - HLSL Compute Shader를 이용한 입자 시뮬레이션
    - Structured Buffer / RWStructuredBuffer 사용
    - SRV / UAV 기반 GPU 데이터 접근
    - CPU 개입을 최소화한 GPU 병렬 입자 갱신

- **CPU / GPU 시뮬레이션 모드 전환**
    - 런타임에서 CPU Simulation과 GPU Simulation 전환
    - 동일한 입자 수와 조건에서 두 방식 비교
    - 입자 수 변경 및 Reset 기능 제공

- **성능 측정 시스템**
    - FPS 및 Frame Time 측정
    - CPU Update Time 측정
    - GPU Update Time 측정
    - Render Time 측정
    - Buffer Upload Time 측정
    - 입자 수에 따른 성능 변화 관측

- **CSV 로그 출력**
    - 프레임별 성능 데이터 저장
    - Particle Count, Simulation Mode, Update Time, Render Time 기록
    - 추후 그래프 분석 및 README 문서화에 활용
