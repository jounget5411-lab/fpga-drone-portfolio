<p align="center"><img src="assets/hero.png" alt="FPGA Vision — FPGA 기반 드론 탐지·추적 종합설계" width="100%"></p>

# FPGA 기반 드론 탐지·추적 종합설계

**영상 처리·SNN 분류 구현**

**카메라의 연속 영상을 FPGA에서 처리하고, 시간에 따른 움직임 특징으로 비행 물체를 분류하는 종합설계 프로젝트입니다.**

카메라 입력부터 영상 전처리, 프레임 차분, SNN 추론, 임베디드 소프트웨어의 결과 확인까지 연결한 구현을 정리했습니다. 아래에서는 소스와 블록 설계에서 확인되는 영상 처리·분류 경로를 중심으로 설명합니다.

[전체 팀 코드](https://github.com/jounget5411-lab/capstone_drone) · [설계 상세](docs/architecture.md) · [핵심 코드 안내](docs/code-map.md) · [실행 환경과 검증 범위](docs/reproduction.md)

| 프로젝트 | 핵심 기술 | 다루는 문제 |
|---|---|---|
| 종합설계 · FPGA 영상 처리 | Zynq-7000 · Verilog · C++ · AXI-Stream · VDMA · SNN | 연속 영상의 움직임을 특징으로 압축하고 하드웨어 추론으로 연결 |

## 한눈에 보는 시스템

<p align="center"><img src="assets/architecture.png" alt="카메라 영상에서 프레임 차분과 SNN 분류 결과까지 이어지는 코드 기반 구조도" width="100%"></p>

영상 전체를 그대로 신경망에 전달하는 대신, **이전 프레임과 달라진 위치를 공간별로 집계하고 여러 프레임에 걸쳐 분석**합니다. 프로세서에서는 카메라와 메모리 전송을 설정하고, FPGA에서는 영상 스트림과 추론 연산을 처리합니다.

| 단계 | 구현 내용 |
|---|---|
| 영상 입력 | OV5640 카메라와 MIPI 수신 경로, Bayer → RGB 및 감마 보정 |
| 움직임 추출 | 160 × 90 그레이스케일 영상과 이전 프레임의 픽셀 차이를 비교 |
| 특징 압축 | 10 × 10 픽셀 단위 집계 → 프레임당 144개 특징 → 3비트 양자화 |
| 시간 정보 처리 | 20프레임 순환 버퍼를 이용한 슬라이딩 윈도우 |
| 하드웨어 추론 | 144 → 8 → 2 구조의 SNN과 LIF 뉴런 상태 연산 |
| 결과 확인 | AXI GPIO 결과 읽기, UART 출력 및 원본·차분 영상 덤프 |

*표의 크기와 개수는 코드에 정의된 설정입니다. 처리 속도나 인식 정확도의 실측 결과를 뜻하지 않습니다.*

## 구현에서 집중한 부분

### 영상 처리와 제어 소프트웨어의 경계 설계

FPGA의 영상 경로는 AXI-Stream으로 연결하고, 프로세서의 C++ 애플리케이션은 카메라 초기화·VDMA 설정·가중치 전송을 담당하도록 구성했습니다. 영상 데이터와 제어 명령의 통로를 나눠 각 단계의 상태를 확인할 수 있게 했습니다.

[main.cc](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/new_workspace/app_component/src/main.cc) · [Top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/Top.v)

### 움직임을 공간과 시간의 특징으로 표현

160 × 90 영상에서 움직임 픽셀을 16 × 9개 영역으로 집계합니다. 집계값을 3비트로 양자화하고, 20프레임의 특징을 순환 버퍼에 저장합니다. 프레임 한 장의 모양뿐 아니라 연속된 움직임을 추론 입력으로 구성한 것이 핵심입니다.

[feature_extractor.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/feature_extractor.v) · [snn_top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/snn_top.v)

### 연산 단계를 나누고 중간 결과를 관찰

프레임 차분 출력에 레지스터 단계를 두고, SNN의 MAC 연산과 LIF 갱신도 여러 상태로 나눴습니다. 긴 조합 경로를 줄이려는 구조이며, 실제 타이밍 충족 여부는 합성·구현 보고서에서 별도로 검증해야 합니다.

디버깅 시에는 UART 명령으로 원본 그레이스케일과 차분 영상을 내보내고, 분류 결과와 두 출력 뉴런의 스파이크 수를 확인할 수 있습니다.

[frame_diff.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/new/frame_diff.v) · [snn_top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/snn_top.v)

### CNN 확장 경로도 함께 구성

블록 설계에는 그레이스케일 영상과 SNN 결과 플래그를 입력받는 `dronet_accel_axi` 경로가 있습니다. CNN 연산기, 프레임 버퍼, AXI 제어 인터페이스가 포함되어 있으며, 이 경로의 실제 가중치 적용과 실행 검증은 SNN 경로와 구분해 정리했습니다.

[pcam_base_720p.bd](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/bd/pcam_base_720p/pcam_base_720p.bd) · [확장 경로 설명](docs/architecture.md)

## 코드를 읽는 순서

1. **전체 흐름:** [main.cc](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/new_workspace/app_component/src/main.cc)에서 초기화·가중치 로드·결과 확인을 살펴봅니다.
2. **하드웨어 연결:** [Top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/Top.v)에서 특징 추출기와 SNN의 입출력을 확인합니다.
3. **핵심 연산:** [feature_extractor.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/feature_extractor.v)와 [snn_top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/snn_top.v)에서 특징 버퍼와 추론 상태 머신을 확인합니다.
4. **보드 수준 연결:** [pcam_base_720p.bd](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/bd/pcam_base_720p/pcam_base_720p.bd)에서 카메라·메모리·추론·HDMI 경로를 확인합니다.

## 문서와 코드 출처

이 저장소는 프로젝트 설명과 설계 도해를 정리한 개인 포트폴리오입니다. 전체 팀 코드는 [기존 저장소](https://github.com/jounget5411-lab/capstone_drone)에 보관되어 있습니다. 본 문서는 `206404b` 커밋을 기준으로 작성했습니다.

구조도는 소스에 근거해 새로 그린 설명용 그림입니다. 실제 장비 사진이나 실험 결과를 대신하지 않습니다. 보드 실행 환경, CNN 경로의 확인 사항, 추적 구동부와 성능 자료의 범위는 [실행 환경과 검증 범위](docs/reproduction.md)에 정리했습니다.

---

[국민대 자율주행 프로젝트](https://github.com/jounget5411-lab/kookmin-autonomous-driving-portfolio) · [HL FMA 2025 프로젝트](https://github.com/jounget5411-lab/hlfma-2025-portfolio)

