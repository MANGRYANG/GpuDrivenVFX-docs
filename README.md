# GpuDrivenVFX 개발 문서

> DirectX 11 기반 VFX 시스템인 **GpuDrivenVFX**의 개발 과정, 구조 설계, 구현 기록, 실험 결과를 정리한 문서 저장소입니다.

이 저장소는 GpuDrivenVFX 본 프로젝트의 코드 저장소가 아니라, 개발 과정에서 작성한 문서와 실험 기록을 관리하기 위한 문서 레포지토리입니다.

### GpuDrivenVFX 코드 저장소
[![GpuDrivenVFX](https://img.shields.io/badge/GpuDrivenVFX-Repository-blue?style=for-the-badge&logo=data%3Aimage%2Fsvg%2Bxml%3Bbase64%2CPHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA0OCA0OCI%2BPHBhdGggZmlsbD0iI2ZmNTcyMiIgZD0iTTYgNmgxNnYxNkg2eiIvPjxwYXRoIGZpbGw9IiM0Y2FmNTAiIGQ9Ik0yNiA2aDE2djE2SDI2eiIvPjxwYXRoIGZpbGw9IiNmZmMxMDciIGQ9Ik0yNiAyNmgxNnYxNkgyNnoiLz48cGF0aCBmaWxsPSIjMDNhOWY0IiBkPSJNNiAyNmgxNnYxNkg2eiIvPjwvc3ZnPg%3D%3D&logoWidth=22)](https://github.com/MANGRYANG/GpuDrivenVFX)

### 문서 저장소
[![GpuDrivenVFX Docs](https://img.shields.io/badge/GpuDrivenVFX-Documentation-purple?style=for-the-badge&logo=obsidian)](https://github.com/MANGRYANG/PixelOS-docs)

---

## 프로젝트 개요

GpuDrivenVFX는 C++과 DirectX 11을 사용해 대량의 입자를 실시간으로 시뮬레이션하고 렌더링하는 VFX 시스템 프로젝트입니다.

이 프로젝트의 핵심 목적은 3D 공간에서 입자를 렌더링하는 시스템을 구현하여, CPU 기반 입자 시스템과 GPU Compute Shader 기반 입자 시스템의 성능을 비교분석하는 것입니다.

---

## 문서화 목표

이 문서 저장소는 단순한 개발 일지가 아니라, GpuDrivenVFX를 구현하면서 어떤 문제를 만났고, 어떤 방식으로 해결했는지 기록하기 위한 공간입니다.

---

## 저장소 구조

```text
GpuDrivenVFX/
├── Canvas/
│   └── Obsidian Canvas 관련 파일
│
├── Docs/
|   ├── 프로젝트 개요 문서
│   └── 성능 비교 실험 레포트
│
├── Images/
│   └── 개발 과정 및 실험 결과 스크린샷
│
├── GpuDrivenVFX/
│   ├── 001. GpuDrivenVFX 프로젝트 환경 구성.md
│   ├── 002. Win32 윈도우 생성 및 애플리케이션 루프 추가.md
│   ├── ...
│   ├── 042. 파티클 방출 평면 회전 적용.md
│   └── 043. GPU Timestamp Query 기반 성능 측정.md
│
├── Template/
|   ├── Docs_Template.md
|   └── Review_Template.md
|
└── TestData/
    ├── particle_performance_1.csv
    ├── ...
    └── particle_performance_4.csv
