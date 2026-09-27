# 영상 처리·추론·제어 구조

[프로젝트 소개로 돌아가기](../README.md)

## 전체 시스템 흐름

설계의 출발점은 표적이 드물게 나타나는 감시 환경입니다. 일반 카메라의 연속 프레임 차분을 SNN으로 분류하고, 비행 후보를 확인하면 CNN으로 위치를 구하는 두 단계 구조를 사용했습니다.

| 단계 | 기능 |
|---|---|
| 감시 | 160 × 90 그레이스케일 → 프레임 차분 → 144차원 특징 → LIF SNN |
| 조건부 정밀 탐지 | Fly 판단에 따라 INT8 CNN을 활성화하고 20 × 11 × 6 RAW grid 출력 |
| 프로세서 후처리 | RAW 정수값에 sigmoid·exp를 적용해 박스 복원, confidence 필터와 IoU NMS 수행 |
| 추적 | 박스 중심과 화면 중심의 오차를 팬·틸트 명령으로 전달 |
| 복귀 | 검출이 이어지지 않으면 추적 상태를 해제하고 SNN 감시로 복귀 |

AXI-Lite로 가중치를 런타임에 로드하고, SNN 판단에 따라 CNN을 활성화합니다. 상태가 바뀔 때 다른 클럭 영역으로 전달되는 이벤트가 유지되도록 동기화와 상태 보존을 설계했습니다.

PL은 픽셀 스트림과 반복 MAC을, PS는 초기화·가중치 전송·비선형 후처리·추적 상태와 주변장치 제어를 맡습니다. 이 경계를 나누어 영상·추론·제어를 각각 관찰할 수 있게 했습니다.

## 영상·추론 데이터 경로

`pcam_base_720p.bd`의 연결을 목적별로 정리했습니다.

- 카메라: MIPI D-PHY → CSI-2 → Bayer to RGB → Gamma correction
- 화면 출력: 영상 분기 → VDMA0 / DDR → Video out → HDMI
- 분류 입력: 영상 분기 → 축소·그레이스케일 → 현재 영상과 VDMA1에서 읽은 이전 영상의 차분
- 차분 출력: VDMA2 / DDR로 디버깅용 저장, 동시에 `Top`으로 전달
- SNN: 특징 추출 → 순환 특징 버퍼 → 추론 → 결과·스파이크 수 → AXI GPIO
- CNN: 차분 전 그레이스케일 → `dronet_accel_axi`; SNN의 `result`도 플래그 입력에 연결

<p><img src="../assets/architecture.png" alt="주요 SNN 영상 분류 경로" width="100%"></p>

도해는 주요 SNN 경로에 집중합니다. HDMI 표시 경로와 일부 AXI 제어 연결은 위 목록과 원본 블록 설계를 함께 참고하세요.

## 영상에서 특징까지

`axis_frame_diff`는 현재·이전 픽셀의 절대 차이를 계산하고 임계값과 비교합니다. 기본 파라미터 `THRESHOLD`는 30입니다. 두 스트림의 프레임 시작·줄 끝 신호도 비교해 동기 불일치를 표시합니다.

`feature_extractor`는 이진 움직임 픽셀을 10 × 10 영역별로 집계합니다. 160 × 90 입력을 16 × 9 = 144개 특징으로 줄이며, 집계값은 0·1·2·4·8·16·32·64의 경계를 이용해 0~7로 양자화합니다. 연속 20프레임은 순환 주소로 저장하고, 윈도우가 채워진 뒤 SNN 추론을 시작합니다.

## SNN 연산 구조

| 구성 | 소스에 정의된 형태 |
|---|---|
| 첫 번째 층 | 입력 특징 144개, 뉴런 8개, 8비트 부호 있는 가중치 |
| 두 번째 층 | 입력 스파이크 8개, 출력 뉴런 2개 |
| 시간 단계 | 20 |
| 누산·뉴런 상태 | 32비트 부호 있는 값 |
| LIF 감쇠 | 이전 상태에 7/8을 곱한 뒤 입력 누산값을 더함 |
| 발화 | 임계값 이상일 때 스파이크를 내고 상태를 0으로 재설정 |
| 결과 | 두 출력 뉴런의 누적 스파이크 수를 비교 |

분류 결과는 `FLY / NON-FLY`로 출력합니다.

## PS와 PL의 연결

PS는 Cortex-A9에서 동작하는 standalone C++ 애플리케이션입니다. 카메라와 VDMA를 초기화하고 GPIO에 가중치·임계값을 씁니다. PL은 영상 스트림과 추론을 처리하고, 완료 여부·분류 결과·스파이크 수를 GPIO로 전달합니다.

공개 코드의 클럭 설정은 영상·추론 150 MHz, 제어 100 MHz입니다.

## CNN 통합

블록 설계에는 `dronet_accel_axi_0` 인스턴스와 입력 영상·AXI 제어·SNN 결과 플래그 연결이 있습니다. 관련 RTL에는 ping-pong 프레임 버퍼, convolution 연산기, 출력 버퍼와 제어 상태 머신이 있습니다.

공개 코드 `206404b`에는 영상·SNN 경로와 CNN 블록 연결이 담겨 있습니다. 이후 6월 최종 통합에서 런타임 CNN 가중치 로드, PS 후처리와 팬틸트 제어로 확장했습니다.

RTL·가중치·XSA·bitstream·Vitis 애플리케이션을 연결하는 순서는 [개발 환경과 실행 안내](reproduction.md)에 정리했습니다.

## 관련 코드

[Vivado 블록 설계](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/bd/pcam_base_720p/pcam_base_720p.bd), [Top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/Top.v), [feature_extractor.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/feature_extractor.v), [snn_top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/snn_top.v), [frame_diff.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/new/frame_diff.v), [dronet_cnn_core.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/imports/rtl_0426/dronet_cnn_core.v)
