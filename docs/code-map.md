# 핵심 코드 안내

[프로젝트 소개로 돌아가기](../README.md)

아래 링크는 공개 코드 `206404b`의 구현을 안내합니다. 6월 최종 통합까지의 경험은 [문제 해결 과정](development.md), 주요 성과는 [결과와 측정 조건](results.md)에서 읽을 수 있습니다.

| 질문 | 파일 | 읽을 부분 |
|---|---|---|
| 프로그램은 어디에서 시작하나요? | [main.cc](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/new_workspace/app_component/src/main.cc) | 카메라·VDMA 초기화, 가중치 로드, UART 메뉴 |
| 영상과 SNN은 어떻게 연결되나요? | [Top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/Top.v) | AXI 입력, 특징 추출기, SNN 및 GPIO 포트 |
| 움직임을 어떻게 숫자로 바꾸나요? | [feature_extractor.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/feature_extractor.v) | 영역별 집계, 양자화, 20프레임 순환 버퍼 |
| 신경망 연산은 어떻게 구현했나요? | [snn_top.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/CORE_IP/src/snn_top.v) | MAC 파이프라인, LIF 상태 갱신, 출력 스파이크 비교 |
| 현재·이전 영상은 어떻게 비교하나요? | [frame_diff.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/new/frame_diff.v) | 절대 차이, 임계값, 동기 신호, 출력 레지스터 |
| CNN 확장은 어디에 있나요? | [dronet_cnn_core.v](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/imports/rtl_0426/dronet_cnn_core.v) | 연산기와 가중치 파일 파라미터 |
| 시스템 전체 연결은 어디에 있나요? | [pcam_base_720p.bd](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/hw_new_0426_cnnintegration/hw_new_backup/hw_new/hw.srcs/sources_1/bd/pcam_base_720p/pcam_base_720p.bd) | 영상 분기, VDMA, SNN, CNN, AXI 제어 |
| 소프트웨어가 어떤 플랫폼을 참조하나요? | [vitis-comp.json](https://github.com/jounget5411-lab/capstone_drone/blob/206404b4667d665620e731083eb8de6d972c248c/new_workspace/app_component/vitis-comp.json) | 플랫폼·프로세서·OS 설정 |

외부 IP와 드라이버를 포함한 전체 소스는 [원본 저장소](https://github.com/jounget5411-lab/capstone_drone)에서 확인할 수 있습니다.
