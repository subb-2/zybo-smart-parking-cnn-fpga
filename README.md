# FPGA 기반 스마트 주차장 관리 시스템 — "CARS대라"

📅 프로젝트 정보

* 진행 기간: 2026.07 ~ 2026.08
* 팀 구성: 7명 (팀장: 김수빈)
* 설계 대상: Zybo Z7-20 기반 번호판 4자리 인식 입출차 시스템 (Pcam 5C 영상 입력 / 전처리 / LeNet-5 CNN 가속기 / 초음파 센서 / 차단바) + Jetson Orin Nano 기반 주차장 내부 안내 시스템
* 기술 스택: `SystemVerilog`, `Verilog`, `UVM`, `C/C++ (Vitis)`, `Python`, `Vivado`, `ILA`, `Zybo Z7-20`, `Pcam 5C (OV5640)`, `HC-SR04`, `SG90`, `ESP32`, `Jetson Orin Nano`

---

## 📝 프로젝트 개요

**차량이 차단바 앞에 서면 번호판을 찍어 FPGA의 CNN 가속기로 4자리 숫자를 인식하고, 주차장 안에서는 가장 가까운 빈자리로 안내하는 스마트 주차장 시스템**입니다.

주차장에서 빈자리를 찾느라 돌아다니는 시간과 그 과정에서 생기는 갈등을 줄이는 것을 목표로 했습니다. 입출차 관리는 FPGA(Zybo Z7-20)가, 주차장 내부 안내는 Jetson Orin Nano가 맡도록 나누었습니다.

<div align="center">
  <img src="docs/images/06_overview.jpg" width="80%" height="auto">
</div>

### 시스템 동작 흐름

**주차장 출입구 (Zybo Z7-20)**
1. 초음파 센서(HC-SR04)가 차단바에 다가온 차량을 감지하면 캡처 신호를 보냄
2. Pcam 5C 영상에서 번호판 영역을 잘라 4자리로 나누고, 바이너리 이미지로 바꿔 Pixel Buffer(BRAM)에 저장
3. LeNet-5 기반 Adder Tree CNN 가속기가 4자리를 차례로 추론
4. 인식이 끝나면 서보 모터(SG90)로 차단바를 열고, 차량이 지나가면 다시 닫음
5. 인식한 차량 번호는 UART와 ESP32(Wi-Fi)를 거쳐 Jetson으로 전송

**주차장 내부 (Jetson Orin Nano)**
1. YOLO로 차량을 검출하고 OC-SORT로 추적
2. FPGA에서 받은 차량 번호와 추적 중인 차량을 매칭해 최단거리 빈자리를 배정
3. 실시간 길 안내 후 주차가 끝나면 안내 종료, 출차 시 요금 자동 정산

<div align="center">
  <img src="docs/images/09_parking_layout.jpg" width="70%" height="auto">
</div>

---

## 👥 팀 구성

| 이름 | 담당 |
|------|------|
| **김수빈 (팀장)** | **Pcam 5C 구동 및 HDMI 출력, AXI Weight Loader로 CNN 가중치 BRAM 로드, 초음파 센서 설계, Zybo 전체 통합 및 디버깅** |
| 김지홍 | CNN 통합, Control Unit, Conv3, FC1, FC2 설계 |
| 문태성 | CNN Reference Model 설계, Conv1, Conv2 검증 |
| 송주연 | Pooling Layer 설계, Conv와 Pool 통합 |
| 조준호 | Conv1, Conv2 설계, 차량 Dataset labeling |
| 윤수민 | 카메라 캡처 및 이미지 전처리, 서보 모터, UART, Wi-Fi 송신, PreProcess 통합 |
| 서어진 | Jetson (YOLO, OC-SORT) 관리 시스템, Wi-Fi 수신, 카메라 캡처 UVM 검증 |

---

## 🏗️ 시스템 구조

```text
                         ┌──────────────────────── Zybo Z7-20 ────────────────────────┐
                         │                                                             │
 Pcam 5C ──MIPI CSI-2──▶ │ [PS] OV5640 I2C 설정 → D-PHY RX → CSI-2 RX → Bayer→RGB      │
 (OV5640)                │      → Gamma → AXI VDMA ⇄ DDR → HDMI 출력 (720p)            │ ──HDMI──▶ 모니터
                         │                    │ (영상 pclk 영역)                        │
                         │                    ▼                                        │
 HC-SR04 ──echo/trig──▶  │ [PL] SR04 FSM ──vga_start──▶ FrameController                │
                         │      (100MHz)   ◀─vga_done──  (Crop → Mono → Register)      │
                         │         │                          │                        │
                         │         │                          ▼                        │
                         │         │                   Pixel Buffer (BRAM, 2-clock)     │
                         │         │                          │                        │
                         │         │   AXI Weight Loader ──▶ CNN 가속기 (LeNet-5)        │
                         │         │   (PS → Weight BRAM)      │ inf_done / inf_out      │
                         │         ◀───────── cnn_done ───────┘                        │
                         │         ▼                                                   │
 SG90  ◀──── PWM ─────── │   차단바 열림/닫힘            UART ──▶ ESP32 ──Wi-Fi──▶ Jetson │
                         └─────────────────────────────────────────────────────────────┘
```

* **클럭 영역**: 영상 경로는 Pcam 영상 pclk, 초음파 센서 FSM과 AXI 로직은 PS의 100MHz(FCLK_CLK0)에서 동작합니다. 두 영역을 오가는 1비트 제어 신호는 모두 Toggle 방식 Pulse Synchronizer로 넘겼습니다.
* **역할 분리**: 카메라 설정, 영상 파이프라인 초기화, 가중치 로드는 PS(C++)가 맡고, 캡처부터 추론, 차단바 제어까지의 실시간 흐름은 PL이 맡습니다.

---

## 🔑 주요 구현 내용 (담당 파트)

### 1. Pcam 5C 구동 및 HDMI 실시간 출력

Digilent Pcam 5C 영상 파이프라인 구조를 바탕으로 카메라 영상을 HDMI로 실시간 출력하고, 같은 영상을 번호판 캡처 경로로 넘기도록 Block Design을 구성했습니다.

| 단계 | 구성 | 역할 |
|------|------|------|
| 센서 설정 | PS I2C (100kHz) + `OV5640.cpp` | OV5640 레지스터 초기화, 해상도와 AWB 설정 |
| 물리 계층 | MIPI D-PHY RX | 카메라 차동 신호 수신, 레인 동기화 |
| 프로토콜 계층 | MIPI CSI-2 RX | 패킷에서 픽셀 데이터 추출 |
| 색 변환 | AXI_BayerToRGB, AXI_GammaCorrection | Bayer 원본을 RGB로 변환, 감마 1/1.8 적용 |
| 프레임 저장 | AXI VDMA ⇄ DDR | 프레임 버퍼 쓰기와 읽기 |
| 출력 | VTC + DVI 클럭 (`DVIClocking`) | 1280x720 HDMI 출력 |

UART 메뉴(`main.cc`)로 해상도, 감마, AWB, 센서 레지스터 읽기와 쓰기를 실행 중에 바꿀 수 있게 해서, 조명 환경이 달라져도 보드를 다시 빌드하지 않고 영상 상태를 조정할 수 있게 했습니다.

### 2. AXI Weight Loader (CNN 가중치 BRAM 로드)

PyTorch로 학습한 CNN 가중치를 PS에서 PL의 Weight BRAM으로 옮기는 AXI4-Lite Slave IP를 설계했습니다. 비트스트림을 다시 만들지 않고 C 코드의 가중치 배열만 바꿔 넣을 수 있다는 점 때문에 `$readmemh` 초기화 대신 AXI 로드 방식을 택했습니다.

**레지스터 맵**

| 주소 | 필드 | 설명 |
|------|------|------|
| 0x00 | `layer[2:0]` | 대상 Weight BRAM 선택 (1~5) |
| 0x04 | `addr` | BRAM 쓰기 주소 |
| 0x08 | `data[7:0]` | 8비트 가중치 |
| 0x0C | `write[0]` | 1을 쓰면 선택한 Layer에 1클럭 Write Enable 발생 |
| 0x10 | `clear[0]` | 추론 완료 상태 클리어 |
| 0x14 | `inf_done` | 추론 완료 상태 (래치) |
| 0x18 | `inf_result[15:0]` | 추론 결과 (4자리) |

**로드 데이터 크기**

| Layer | 크기 | 비고 |
|-------|------|------|
| W1 (Conv1) | 150 B | |
| W2 (Conv2) | 2,400 B | |
| W3 (Conv3) | 48,000 B | |
| W4 (FC1) | 10,080 B | |
| W5 (FC2) | 880 B | 노드당 84 B + 0 패딩 4 B, 88 B 단위로 정렬 |

추론 완료 신호(`inf_done`)는 짧은 펄스라 PS가 폴링으로 놓칠 수 있어서, IP 내부에서 결과와 함께 래치해 두고 PS가 확인한 뒤 클리어하도록 만들었습니다.

### 3. 초음파 센서 제어 및 입차 시퀀스 FSM (`sr04.sv`)

HC-SR04 거리 측정부(`SR04_Controller`)와 입차 전체 순서를 관리하는 FSM(`sr04_fsm`)을 설계했습니다. 차량 감지부터 캡처, 추론, 차단바 열림, 통과 확인까지 이 FSM이 순서를 잡아 줍니다.

| 상태 | 동작 | 다음 상태로 가는 조건 |
|------|------|------------------------|
| IDLE | 차단바 닫힘, 거리 감시 | 감지 거리 3cm 이하 |
| WAIT | 차량이 멈출 때까지 대기 (약 4초) | 대기 종료 시 `vga_start` 1클럭 출력 |
| WAIT_VGA | 번호판 캡처 및 Pixel Buffer 저장 대기 | `vga_done` |
| WAIT_CNN | CNN 추론 대기 | `cnn_done` |
| OPEN | 차단바 열림 신호 출력 | 바로 WAIT_GONE |
| WAIT_GONE | 차량 통과 확인 | 5~100cm 거리가 8회 연속 측정 |
| WAIT_DELAY | 차단바를 닫기 전 대기 | 대기 종료 후 IDLE |

* `echo` 입력은 2단 플립플롭으로 동기화하고, 1MHz tick의 상승 에지마다 echo 길이를 세어 거리로 환산했습니다.
* 차량이 지나간 것은 거리 값 한 번이 아니라 **8회 연속 측정**으로 판정해서, 센서 값이 튀어도 차단바가 잘못 닫히지 않게 했습니다.

### 4. 전체 시스템 통합 및 CDC 처리

카메라, 전처리, CNN, 센서, 모터, UART를 하나의 Block Design으로 묶고, 모듈마다 다른 클럭 영역 사이의 신호 전달을 정리했습니다.

| 신호 | 방향 | 처리 방법 |
|------|------|-----------|
| `vga_start` | 100MHz (SR04 FSM) → 영상 pclk | `PulseSync_100M_to_25M` (Toggle + 2FF + XOR) |
| `vga_done` | 영상 pclk → 100MHz | `PulseSync_25M_to_100M` (Toggle + 2FF + XOR) |
| 픽셀 데이터 | 영상 pclk 쓰기 → 100MHz 읽기 | Pixel Buffer를 쓰기와 읽기 클럭이 분리된 BRAM으로 구성 |
| 디버그 상태 | 영상 pclk → 100MHz | `vga_debug_cdc`로 동기화 후 AXI GPIO로 읽기 |

1클럭 펄스를 그대로 2FF에 넣으면 빠른 클럭의 펄스를 느린 클럭이 놓칠 수 있어서, 송신 쪽에서 펄스를 토글 신호로 바꾸고 수신 쪽에서 2FF 동기화 후 변화를 검출하는 방식을 썼습니다.

### 5. 보드 디버깅 환경 구축 (ILA + Python)

시뮬레이션만으로는 보이지 않는 보드 위 문제를 찾기 위해 디버깅 경로를 따로 만들었습니다.

* **ILA**: SR04 FSM 상태, 거리 값, `vga_start`, `vga_done`, `cnn_done`을 한 화면에서 보도록 `ila_sr04_debug`를 넣었고, 영상 pclk 확인용과 CNN 제어 확인용 ILA도 따로 두었습니다.
* **AXI GPIO 스냅샷**: PS에서 Pixel Buffer, Conv1~Conv3, FC1, FC2 중간 결과 BRAM을 주소별로 읽어 UART로 덤프하도록 `main.cc`에 메뉴를 추가했습니다.
* **Python 도구** (`python_debug/`)
  * `real_final_live_debug.py`: 115200bps UART로 FSM 상태, CNN 제어 상태, 인식 결과를 주기적으로 받아 화면에 표시하고, 바뀐 값을 표시해 흐름이 어디서 멈췄는지 보여 줌
  * `real_final_full_dump.py`: 프레임 버퍼와 중간 연산 결과 전체를 파일로 저장해 기대값과 비교

---

## 🧩 팀원 담당 파트 요약

통합을 맡으면서 각 모듈의 입출력과 동작 타이밍을 맞춰야 했기 때문에, 팀원 담당 모듈도 함께 정리합니다.

### PreProcess (카메라 캡처 및 전처리)

번호판 영역을 잘라(Crop) 바이너리로 바꾸고(Mono), 4자리로 나눠 좌우 반전 후 Pixel Buffer에 저장합니다. 28x28 숫자 데이터는 패딩을 붙여 CNN 입력 크기인 32x32로 맞춥니다. 화면에는 `plate_guide`로 번호판을 맞출 가이드 박스를 HDMI 영상 위에 겹쳐 표시합니다.

<div align="center">
  <img src="docs/images/19_preprocess.jpg" width="80%" height="auto">
</div>

### CNN 가속기 (LeNet-5, Adder Tree)

Conv1/Pool1 → Conv2/Pool2 → Conv3 → FC1 → FC2 순서로 4자리를 차례로 추론합니다. 5x5 커널의 곱셈 25개를 직렬로 더하던 구조를 Adder Tree로 바꿔 **누적 연산을 25클럭에서 5클럭으로** 줄였습니다.

<div align="center">
  <img src="docs/images/20_cnn_flow.jpg" width="48%" height="auto">
  <img src="docs/images/21_adder_tree.jpg" width="48%" height="auto">
</div>

### Jetson 관리 시스템

YOLO로 차량을 검출하고 OC-SORT로 추적합니다. FPGA가 보낸 차량 번호와 추적 ID를 매칭해 빈자리를 배정하고, 안내 UI에 경로를 표시합니다.

---

## 🚀 문제 해결 (Troubleshooting)

### 1. Pcam 5C 영상이 나오지 않는 문제

* **문제**: 비트스트림과 펌웨어를 올려도 HDMI에 카메라 영상이 나오지 않거나 불안정하게 나옴.
* **원인**: 한 가지가 아니라 여러 원인이 겹쳐 있었음.
  * FFC 케이블 접촉 불량
  * 보드 전원 점퍼(JP6) 설정
  * Vivado 2020.1과 2020.2 사이의 프로젝트 버전 불일치
  * MIPI D-PHY 동기화 불안정
* **해결**: 물리 연결 → 전원 → 툴 버전 → D-PHY 순서로 원인을 하나씩 분리해 확인하고 각각 수정해 HDMI 실시간 출력을 안정화함.

### 2. 초음파 센서가 차량을 감지해도 캡처가 시작되지 않는 문제 (CDC)

* **문제**: 초음파 센서는 차량을 정상 감지하는데 번호판 캡처가 시작되지 않음.
* **원인**: 처음에는 SR04 FSM이 VGA 모듈과 같은 영상 클럭에서 동작했음. 이후 SR04를 시스템 클럭(100MHz)으로 옮기면서 두 모듈 사이에 클럭 영역이 갈라졌고, 100MHz에서 만든 1클럭 `vga_start` 펄스가 영상 클럭 쪽에 제대로 전달되지 않았음. ILA로 SR04 쪽 송신 신호와 FrameController 쪽 수신 신호를 나란히 비교해 확인함.
* **해결**: VGA 담당 팀원과 원인을 공유하고, 양방향(`vga_start`, `vga_done`)에 Toggle 방식 Pulse Synchronizer를 추가해 해결함.

```systemverilog
// 송신(100MHz): 펄스를 토글 상태로 변환
always_ff @(posedge clk_100m or posedge reset)
    if (reset)         toggle_100m <= 1'b0;
    else if (pulse_in) toggle_100m <= ~toggle_100m;

// 수신(pclk): 2FF 동기화 후 변화 검출
always_ff @(posedge clk_25m or posedge reset)
    if (reset) {q1, q2, q3} <= '0;
    else       {q1, q2, q3} <= {toggle_100m, q1, q2};

assign pulse_out = q2 ^ q3;
```

### 3. Weight Loader 경로 Negative Slack 및 Pixel Buffer 저장 불량

* **문제**: 번호판을 찍어도 Pixel Buffer에 저장된 이미지가 기대한 숫자 모양과 다르게 깨져 있었고, 입출차 시스템 전체 Timing Report에서 **WNS -7.434ns** (TNS -4,349ns, Failing Endpoint 3,432개)가 나옴.
* **원인**: AXI Weight Loader에서 Weight BRAM으로 가는 경로에서 복잡한 주소 연산을 한 클럭 안에 모두 처리하고 있었음. 이와 별개로 이미지를 바이너리화해 저장하는 방식이라 조명 변화에 예민한 점도 저장 불량의 원인이었음.
* **해결**: 한 클럭에 처리하던 주소 연산을 여러 클럭에 나눠 처리하도록 경로 중간에 레지스터를 추가해 **WNS를 -7.434ns에서 -3.447ns로, Failing Endpoint를 3,432개에서 504개로** 줄임. 저장 경로는 VGA 담당 팀원과 함께 디버깅해, 최종적으로 Pixel Buffer 이미지가 기대 이미지와 일치하는 것을 확인함.

<div align="center">
  <img src="docs/images/33_timing_weight_loader.jpg" width="48%" height="auto">
  <img src="docs/images/34_pixelbuffer_fixed.jpg" width="48%" height="auto">
</div>

> 전체 시스템 기준으로는 Negative Slack이 남아 있습니다. 남은 경로를 모두 정리하는 것은 이후 과제로 남겼습니다.

### 4. 추론 완료 신호가 두 번 세어지는 문제

* **문제**: 4자리를 차례로 추론하는 중에 자릿수 카운트가 한 번에 두 칸씩 넘어가는 경우가 생김.
* **원인**: CNN 완료 신호(`inf_done`)가 1클럭이 아니라 7클럭 동안 유지되어, 받는 쪽에서 같은 완료를 두 번 셈.
* **해결**: 팀원과 함께 받는 쪽에 상승 에지 검출을 추가해 완료 1회당 정확히 1번만 세도록 수정함.

---

## ✅ 결과

<table>
<tr>
<td width="48%">

**시스템 통합 결과**

| 항목 | 결과 |
|------|------|
| 번호판 인식 | 4자리 중 1자리 → **4자리 모두 인식** (25% → 100%) |
| 입차 시퀀스 | 감지 → 캡처 → 추론 → 차단바 열림/닫힘 **보드 동작 확인** |
| 최종 발표 | 전체 시스템 시연 완료 |

</td>
<td width="4%"></td>
<td width="48%">

**CNN 가속기 (팀 결과)**

| 항목 | 결과 |
|------|------|
| 추론 속도 | 가속 전 대비 **6.6배** (숫자 1개 기준) |
| 자원 사용량 | LUT 34%, FF 10%, BRAM 22%, DSP 1% |
| 타이밍 (CNN 단독, 125MHz) | WNS +0.214ns, 제약 충족 |
| 카메라 캡처 UVM | Reference와 비교해 정상 동작 확인 |

</td>
</tr>
</table>

<div align="center">
  <img src="docs/images/27_uvm_capture_result.jpg" width="48%" height="auto">
  <img src="docs/images/29_cnn_speedup.jpg" width="48%" height="auto">
</div>

---

## 🎥 Demo Video

<!-- 시연 영상을 GitHub 이슈나 README 편집 창에 끌어다 놓으면 생기는 user-attachments 주소로 교체 -->
<video src="여기에_시연_영상_주소" width="400" controls></video>

---

## 📚 배운 점

* **통합 단계의 오류는 경계에서 생긴다**: 각 모듈은 혼자 돌릴 때 모두 정상이었지만, 클럭 하나를 바꾼 것만으로 모듈 사이 신호가 끊겼습니다. 모듈을 붙일 때는 기능보다 먼저 클럭 영역과 신호 길이(펄스인지 레벨인지)를 확인해야 한다는 것을 배웠습니다.
* **보이지 않으면 볼 수 있는 길부터 만든다**: 보드 위 문제는 시뮬레이션에서 재현되지 않아서, ILA와 AXI GPIO 스냅샷, Python 덤프 도구를 먼저 만들고 나서야 원인을 좁힐 수 있었습니다. 송신 쪽과 수신 쪽 신호를 나란히 놓고 비교하는 방식이 가장 빨랐습니다.
* **원인은 하나가 아닐 수 있다**: Pcam이 안 켜진 이유는 케이블, 전원, 툴 버전, D-PHY가 겹쳐 있었습니다. 물리 연결부터 차례로 하나씩 분리해 확인하는 순서를 지킨 덕분에 같은 곳을 반복해서 의심하지 않을 수 있었습니다.
* **팀장으로서의 일정 관리**: 매일 아침 회의로 진행 상황을 공유하고, 통합을 기다리는 동안 다른 팀원들이 주차장 모형 제작과 Jetson 데이터셋 작업을 먼저 진행하도록 순서를 조정했습니다. 통합이 막힌 동안에도 팀 전체 일정이 멈추지 않게 하는 것이 중요하다는 것을 느꼈습니다.

---

## 🖥️ 개발 환경

| 항목 | 내용 |
|------|------|
| HDL | SystemVerilog, Verilog (일부 IP는 VHDL) |
| 검증 | UVM (카메라 캡처), SystemVerilog Testbench (CNN) |
| EDA Tool | Xilinx Vivado, Vitis |
| 타겟 보드 | Digilent Zybo Z7-20 (Zynq-7000 XC7Z020) |
| 카메라 | Digilent Pcam 5C (OV5640, MIPI CSI-2) |
| 센서 및 구동부 | HC-SR04 초음파 센서, SG90 서보 모터 |
| 통신 | AXI4-Lite, UART (115200bps), ESP32 Wi-Fi |
| 보드 디버깅 | ILA, AXI GPIO, Python (pyserial) |
| 관리 시스템 | Jetson Orin Nano, YOLO, OC-SORT |

---

## 📁 파일 구성

```text
├── RTL/                                   # 모듈별 RTL (단독 테스트용)
│   ├── SR04/
│   │   ├── SR04_Controller.sv             # 초음파 거리 측정 (trigger / echo / 1MHz tick)
│   │   ├── sr04.sv                        # 입차 시퀀스 FSM + CDC Pulse Sync + ILA
│   │   └── SR04_testTOP.sv
│   ├── SYNC_2FF.sv                        # 2FF 동기화기, Toggle 방식 Pulse Synchronizer
│   ├── VGA/
│   │   ├── FrameController.sv             # 번호판 Crop / Mono / 4자리 Register
│   │   ├── PixelBuffer.sv                 # 쓰기 pclk / 읽기 100MHz 분리 BRAM, 32x32 패딩
│   │   └── VGA_top.sv
│   ├── PreProcess.sv                      # 센서 + 캡처 + UART 묶음 Top
│   ├── CNN/CNN_TOP/                       # LeNet-5 CNN (Conv / Pool / FC / Control)
│   │   └── mem/                           # 가중치 및 테스트 이미지 .mem
│   ├── SG90/SG90_Controller.sv            # 차단바 서보 PWM
│   ├── UART/                              # 추론 결과 UART 송신 (FIFO 포함)
│   └── Wi-Fi/                             # ESP32 Wi-Fi 송신 코드
│
├── System_Integration/                    # Zybo 통합 Vivado / Vitis 프로젝트
│   ├── zybo-z7_real_final.srcs/
│   │   └── sources_1/
│   │       ├── bd/system/                 # Block Design (PS, MIPI, VDMA, HDMI, AXI)
│   │       ├── new/
│   │       │   ├── cnn_top_v.v            # CNN Block Design 래퍼 + 디버그 read-back
│   │       │   ├── preprocess_v.v         # PreProcess Block Design 래퍼
│   │       │   ├── plate_guide.v          # HDMI 화면 번호판 가이드 박스
│   │       │   ├── vga_debug_cdc.v        # 영상 pclk 디버그 상태 → 100MHz 동기화
│   │       │   └── cnn_uart_debug.v
│   │       └── ip/ila_*                   # ILA 디버그 코어
│   ├── ip/ip_repo/axi_weight_loader_v_1.0/ # AXI4-Lite Weight Loader IP
│   ├── vitis_workspace/0812_ParkingLot/src/src/
│   │   ├── main.cc                        # 영상 파이프라인 초기화, 가중치 로드, 디버그 메뉴
│   │   ├── cnn_weights.cc / .h            # CNN 가중치 배열
│   │   ├── ov5640/                        # OV5640 I2C 설정, VDMA, GPIO 드라이버
│   │   └── hdmi/VideoOutput.h
│   └── python_debug/
│       ├── real_final_live_debug.py       # UART 실시간 상태 모니터링
│       └── real_final_full_dump.py        # 프레임 버퍼, 중간 연산 BRAM 전체 덤프
│
├── cctv_gui/                              # 출입구 CCTV UI
├── python_code/                           # Jetson 관리 시스템 (검출, 추적, 안내)
├── structure.md                           # Jetson 시스템 설계 문서
└── docs/images/                           # README 이미지
```
