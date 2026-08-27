# Force/Torque Sensor User Manual — AFT_XXX-C

Rev. 2026.08.27 (초기 버전)

## Foreword

본 매뉴얼은 AIDIN ROBOTICS AFT_XXX-C 센서의 정상적인 사용을 위해 필요한 정보를 담고 있습니다.
로봇 시스템을 사양 범위 밖에서 사용할 경우 제품의 기본 성능이 보장되지 않습니다. 사용 전 본
매뉴얼을 주의 깊게 읽어 주십시오.

**Notice**: 이 매뉴얼은 AIDIN ROBOTICS의 승인 없이 복사, 재생산 또는 공유할 수 없습니다.

**Manufacturer**: AIDIN ROBOTICS

**Safety Precautions**: 센서 설치는 반드시 해당 국가/지역 법규에 따라 자격을 갖춘 사람이
수행해야 합니다.

## 목차

- [1. Product Overview](#1-product-overview)
  - [1.1 AFT_XXX-C](#11-aft_xxx-c)
  - [1.2 Key Features](#12-key-features)
- [2. Installation Guide](#2-installation-guide)
  - [2.1 Axes and Drawings](#21-axes-and-drawings)
  - [2.2 Mounting / Cable](#22-mounting--cable)
- [3. Communication](#3-communication)
  - [3.1 Default CAN Setting](#31-default-can-setting)
  - [3.2 User Commands](#32-user-commands)
  - [3.3 Sensor Data Output](#33-sensor-data-output)
- [4. Additional Information](#4-additional-information)

## 1. Product Overview

### 1.1 AFT_XXX-C

AIDIN ROBOTICS의 6축 힘/토크(Force/Torque) 센서로, 정전용량식(capacitive) 방식을 사용합니다.
로봇 손목/관절 등에 장착해 접촉력을 실시간으로 측정하는 용도입니다.

### 1.2 Key Features

- Smart 6-axis force/torque sensor
- All-in-one sensor (별도 앰프 불필요)
- 디지털 출력 통신 (CAN, CAN-FD)
- 옵션: IMU(가속도/자이로) 부가 데이터 — [3.2](#32-user-commands) 참조
- CAN IAP 부트로더 내장 — 케이블 하나로 펌웨어 현장 업데이트 가능

## 2. Installation Guide

### 2.1 Axes and Drawings

![AFT_XXX-C 좌표계](img/AFT_XXX_coordinate_frame.png)

센서 원점은 상판 중심, **X(적색)·Y(녹색)·Z(청색)**. CAN으로 나오는 Fx/Fy/Fz, Tx/Ty/Tz는
전부 이 좌표계 기준입니다. 우측 그림은 하단 커넥터/체결 구조 참고용입니다.

### 2.2 Mounting / Cable

*(체결 토크, 케이블 길이·핀맵 등은 기구 도면 확인 후 추가 예정)*

## 3. Communication

### 3.1 Default CAN Setting

- 출력 레이트는 100Hz~1000Hz로 변경 가능 — 방법은 [3.2](#32-user-commands) 참조
- **CAN 2.0**
  - Nominal bitrate: 1 Mbps
  - RX ID: `0x220`(명령), `0x230`(측정 기본, 변경 가능)
- **CAN-FD**
  - Nominal bitrate: 1 Mbps / Data bitrate: 최대 4 Mbps(BRS, FD Parameter 설정값에 따름)
  - RX ID: `0x320`(명령), `0x330`(측정 기본, 변경 가능)
- 부트로더 UDS ID: `0x7A0`(요청) / `0x7A8`(응답) — CAN IAP 펌웨어 업데이트용, 별도 문서 참고

### 3.2 User Commands

- **RX CAN IDs**: `0x220`(CAN 2.0), `0x320`(CAN FD)
- **TX CAN IDs**: 기본값 `0x230`(CAN 2.0), `0x330`(CAN FD). 아래 표의 "current TX CAN ID"는
  이 값을 LSB, MSB 순서로 2바이트에 넣은 것입니다 (기본값이면 `[0x30, 0x02]` 또는 `[0x30, 0x03]`).

**Data Transmission Procedure**

AFT_XXX-C는 전원 인가 직후 **별도 명령 없이 기본값(CAN 2.0, INT 온도보상 포함, TX ID `0x230`)으로
자동 출력을 시작**합니다. 다른 모드/데이터타입으로 바꾸려면 **CAN MODE 설정 명령**을 보낸 뒤
**TRANSMIT DATA 요청 명령**을 순서대로 보내면 됩니다 — **CAN MODE 명령은 데이터 출력을 정지(stop)시키는
기능을 함께 가지고 있어**, 이 명령을 보내면 출력이 일단 멈추고 TRANSMIT DATA를 다시 보내야 재개됩니다.

예시(기본 TX ID `0x230` 기준, CAN 2.0 모드로 온도보상 포함 INT 출력 다시 시작):
```
CAN MODE:      ID 0x220,  Data: 0x30 0x02 0x04 0x01
TRANSMIT DATA: ID 0x220,  Data: 0x30 0x02 0x03 0x02
```

**명령 목록**

| COMMAND | RX CAN ID | Data[0] | Data[1] | Data[2] | Data[3]<br>(아래 표 참조) | Data[4] | Data[5] |
|---|---|---|---|---|---|---|---|
| SENSOR TX CAN ID SET | `0x220` `0x230` | current TX CAN ID(LSB) | current TX CAN ID(MSB) | `0x01` | `0x01`(CAN2.0) / `0x02`(CAN FD) | TX CAN ID(LSB) | TX CAN ID(MSB) |
| BIAS | 〃 | 〃 | 〃 | `0x02` | — | — | — |
| TRANSMIT DATA | 〃 | 〃 | 〃 | `0x03` | `0x01`~`0x06` | — | — |
| CAN MODE *(데이터 출력 정지 동반)* | 〃 | 〃 | 〃 | `0x04` | `0x01`~`0x03` | — | — |
| SAMPLE RATE SET | 〃 | 〃 | 〃 | `0x05` | `0x01`~`0x04` | — | — |
| **IMU ADDITIONAL FRAME** *(New to AFT_XXX-C)* | 〃 | 〃 | 〃 | `0x09` | `0x01`~`0x04` | — | — |
| TX CAN ID CONFIRM | `0x220` `0x230` | `0xFF` | `0xFE` | `0xFC` | `0x01`(CAN2.0) / `0x02`(CAN FD) | — | — |
| FACTORY RESET | 〃 | `0xFF` | `0xFE` | `0xFD` | — | — | — |
| ERROR PACKET ON/OFF *(New to AFT_XXX-C)* | 〃 | `0xFF` | `0xFE` | `0xFA` | `0x01`(ON) / `0x02`(OFF) | — | — |

**COMMAND / Data[3] Description**

| COMMAND | Data[3] Description |
|---|---|
| SENSOR TX CAN ID SET | `0x01`: TX CAN 2.0 ID SET ex) ID(LSB)=0x23, ID(MSB)=0x01 → Resulting ID `0x123`<br>`0x02`: TX CAN FD ID SET ex) ID(LSB)=0x56, ID(MSB)=0x04 → Resulting ID `0x456` |
| BIAS | Bias(Zero Setting) |
| TRANSMIT DATA | `0x01`: INT, 온도보상 없음<br>`0x02`: INT, 온도보상 포함<br>`0x03`: INT Combined, 온도보상 없음 (CAN 2.0 미지원, FD 전용)<br>`0x04`: INT Combined, 온도보상 포함 (FD 전용)<br>`0x05`: Float Combined, 온도보상 없음 (FD 전용)<br>`0x06`: Float Combined, 온도보상 포함 (FD 전용) |
| CAN MODE | `0x01`: CAN 2.0 모드<br>`0x02`: CAN FD 모드 BRS OFF<br>`0x03`: CAN FD 모드 BRS ON<br>**이 명령을 보내면 데이터 출력이 즉시 정지합니다.** 새 모드로 출력을 재개하려면 반드시 이어서 TRANSMIT DATA 명령을 보내야 합니다([Data Transmission Procedure](#32-user-commands) 참조) |
| SAMPLE RATE SET | `0x01`: 100Hz (Default)<br>`0x02`: 250Hz<br>`0x03`: 500Hz<br>`0x04`: 1000Hz |
| IMU ADDITIONAL FRAME | `0x01`: OFF<br>`0x02`: 가속도만<br>`0x03`: 자이로만<br>`0x04`: 가속도+자이로 — 켜면 현재 데이터타입과 무관하게 매 전송주기마다 해당 프레임 추가 송신. `IMU_MPUXX50` 빌드에서만 동작(현재 기본 미탑재) |
| TX CAN ID CONFIRM | `0x01`: TX CAN 2.0 ID 확인, 데이터 없음(DLC=0)으로 응답<br>`0x02`: TX CAN FD ID 확인, 데이터 없음(DLC=0)으로 응답 |
| FACTORY RESET | RATE 100Hz(Default), Zero Bias(Default), TX CAN2.0 ID `0x230`(Default), TX CANFD ID `0x330`(Default) |
| ERROR PACKET ON/OFF | `0x01`(또는 0x02 외 값): ON (기본) — 판정용 에러워드 2바이트를 데이터프레임 끝에 추가<br>`0x02`: OFF — **Not yet implemented** |

### 3.3 Sensor Data Output

> 센서 신호 안정화를 위해 **약 30분** 워밍업을 권장합니다. 사용 전 **최소 10분**은 센서를 켜 둔
> 채로 두십시오 — 초반 10분간 출력 데이터가 흔들릴 수 있습니다.
>
> 모든 데이터 조합은 **little-endian**입니다 (아래 그림처럼 낮은 바이트가 낮은 주소/먼저 오는 바이트).

#### Force INT Transmitting mode (Data[2]=0x03, Data[3]=1 or 2)

| INDEX | TX CAN ID | DLC | data[0] | data[1] | data[2] | data[3] | data[4] | data[5] |
|---|---|---|---|---|---|---|---|---|
| FINT | 2.0: `can20ID`+0<br>FD: `canFDID`+0 | 6 | Fx(LSB) | Fx(MSB) | Fy(LSB) | Fy(MSB) | Fz(LSB) | Fz(MSB) |

- `Fx Output = Fx(MSB)*256 + Fx(LSB)` (Fy, Fz 동일)
- `Force[N] = Force Output / 100`
- 최종 계산값은 **16bit 정수**로 캐스팅해서 씁니다.

#### Torque INT Transmitting mode (Data[2]=0x03, Data[3]=1 or 2)

| INDEX | TX CAN ID | DLC | data[0] | data[1] | data[2] | data[3] | data[4] | data[5] |
|---|---|---|---|---|---|---|---|---|
| TINT | 2.0: `can20ID`+1<br>FD: `canFDID`+1 | 6 | Tx(LSB) | Tx(MSB) | Ty(LSB) | Ty(MSB) | Tz(LSB) | Tz(MSB) |

- `Tx Output = Tx(MSB)*256 + Tx(LSB)` (Ty, Tz 동일)
- `Torque[Nm] = Torque Output / 1000`
- 최종 계산값은 **16bit 정수**로 캐스팅해서 씁니다.

#### Combined INT Transmitting mode (Data[2]=0x03, Data[3]=3 or 4, FD only)

| INDEX | TX CAN ID | DLC | data[0] | data[1] | data[2] | data[3] | data[4] | data[5] |
|---|---|---|---|---|---|---|---|---|
| CINT | `canFDID`+2 | 12 | Fx(LSB) | Fx(MSB) | Fy(LSB) | Fy(MSB) | Fz(LSB) | Fz(MSB) |

| INDEX | data[6] | data[7] | data[8] | data[9] | data[10] | data[11] |
|---|---|---|---|---|---|---|
| CINT | Tx(LSB) | Tx(MSB) | Ty(LSB) | Ty(MSB) | Tz(LSB) | Tz(MSB) |

계산식은 FINT/TINT와 동일 (`/100`, `/1000`, 16bit 정수 캐스팅).

#### Combined Float Transmitting mode (Data[2]=0x03, Data[3]=5 or 6, FD only)

| INDEX | TX CAN ID | DLC | data[0~3] | data[4~7] | data[8~11] | data[12~15] | data[16~19] | data[20~23] |
|---|---|---|---|---|---|---|---|---|
| CFLOAT | `canFDID`+2 | 24 | Fx | Fy | Fz | Tx | Ty | Tz |

```c
uint32_t Fx_Raw = (Fx(MSB) << 24) | (Fx(3rd) << 16) | (Fx(2nd) << 8) | (Fx(LSB));
float    Fx_Output = *(float *)&Fx_Raw;
/* Fy, Fz, Tx, Ty, Tz 도 동일하게 반복 */

Force[N]  = Force Output;   /* 이미 물리값이라 나눗셈 없음 */
Torque[Nm] = Torque Output;
```

최종 계산값은 **float**로 캐스팅해서 씁니다.

#### IMU Additional Frame (Data[2]=0x09, New to AFT_XXX-C — [3.2](#32-user-commands) `0x02`~`0x04`일 때만)

| INDEX | TX CAN ID | DLC | data[0] | data[1] | data[2] | data[3] | data[4] | data[5] | 송신 조건(Data[3]) |
|---|---|---|---|---|---|---|---|---|---|
| AIMU (가속도) | 2.0: `can20ID`+3<br>FD: `canFDID`+3 | 6 | AccelX(LSB) | AccelX(MSB) | AccelY(LSB) | AccelY(MSB) | AccelZ(LSB) | AccelZ(MSB) | `0x02`, `0x04` |
| GIMU (자이로) | 2.0: `can20ID`+4<br>FD: `canFDID`+4 | 6 | GyroX(LSB) | GyroX(MSB) | GyroY(LSB) | GyroY(MSB) | GyroZ(LSB) | GyroZ(MSB) | `0x03`, `0x04` |

- 값은 raw int16 그대로(물리단위 환산 없음). 스케일: 가속도 ±16g = 2048 LSB/g, 자이로 ±2000dps = 16.4 LSB/(deg/s)
- 현재 선택된 F/T 데이터타입([3.2](#32-user-commands) Data[2]=0x03)과 무관하게 독립적으로 켜고 끕니다.

## 4. Additional Information

### 4.1 USB-to-CAN interfaces

PCAN-USB FD Device(USB to CAN FD)를 사용합니다. 다른 CAN 보드를 쓰셔도 되지만, 샘플
프로그램을 그대로 쓰려면 peak-system 컨버터를 권장합니다.

- Device Item number: IPEH-004022 — https://www.peak-system.com/PCAN-USB-FD.365.0.html?&L=1
- PCAN-View (Windows 표시용 소프트웨어) — https://www.peak-system.com/PCAN-View.242.0.html?&L=1
- CAN 데이터를 정상적으로 받으려면 CAN H / CAN L 사이에 **120Ω 종단저항**이 필요합니다.

---

Rev. 2026.08.27
