# AFT_RBY2_C CAN 프로토콜 레퍼런스

작성일 2026-08-27 · 변경일 2026-08-27 · 통신: FDCAN2 · 클래식 CAN 2.0(1Mbit) / CAN-FD

AFT_RBY2_C 앱 펌웨어(`app_freertos.c`)가 실제로 처리하는 CAN 명령과, 실제로 송신하는 측정
데이터 프레임을 코드 기준으로 정리한 레퍼런스입니다. 아래 표에 없는 명령/필드는 존재하지 않습니다.

같은 내용을 보기 좋게 꾸민 HTML 버전: [`AFT_RBY2_CAN_Protocol.html`](AFT_RBY2_CAN_Protocol.html) (다운받아 브라우저로 열기)

## 목차

1. [기본 구조 (ID·prefix)](#1-기본-구조)
2. [명령 목록 (D0)](#2-명령-목록-d0-byte-2)
3. [TRANSMITTING 데이터타입 (D1)](#3-transmitting-데이터타입-d00x03-d1)
4. [TX_MODE / UPDATERATE / FDSETTING](#4-tx_mode--updaterate--fdsetting-값표)
5. [IMU 부가 프레임 (신규)](#5-imu-부가-프레임-신규-2026-08-27)
6. [특수 매직 명령](#6-특수-매직-명령)
7. [측정 데이터 프레임 포맷](#7-측정-데이터-프레임-포맷-송신-센서호스트)
8. [기본값](#8-기본값)
9. [주의사항](#9-주의사항)

## 1. 기본 구조

**수신(호스트→센서)**: 표준ID `0x220`(CAN 2.0) 또는 `0x320`(CAN FD)로 8바이트 명령 프레임을 보냅니다.

```
byte 0-1 : 현재 센서 TX ID (little-endian, LSB 먼저) — can20ID 또는 canFDID와 일치해야 함
byte 2   : 명령(D0)
byte 3   : 서브값(D1)
byte 4-7 : 명령별 부가 데이터
```

ID가 일치하지 않으면(`Check_txcanID()`) 명령 자체가 무시됩니다 — 단, [특수 매직 명령](#6-특수-매직-명령)
(`FF FE...`로 시작하는 것들)은 이 ID 검사 없이 별도 분기로 처리됩니다.

## 2. 명령 목록 (D0, byte 2)

| D0 | 이름 | 설명 | 상태 |
|---|---|---|---|
| `0x01` | SETID | D1=1: CAN20 ID 변경(byte4-5, LE) · D1=2: CANFD ID 변경(byte4-5, LE). 0x7FF 초과 시 기본값으로 되돌아감 | 구현됨 |
| `0x02` | BIAS | 현재 출력을 0점으로(`setBias=1`). 서브값 없음 | 구현됨 |
| `0x03` | TRANSMITTING | 전송 시작 + 데이터타입 선택. D1 값은 [3장](#3-transmitting-데이터타입-d00x03-d1) 참조 | 구현됨 |
| `0x04` | TX_MODE | CAN 모드 전환(D1: 1=CAN20 2=CANFD 3=CANFD_BRS). 전환 시 `continuousUpdate=0`(별도 TRANSMITTING 필요) | 구현됨 |
| `0x05` | UPDATERATE | 전송 주기(D1: 1=100Hz 2=250Hz 3=500Hz 4=1000Hz, 그 외=100Hz) | 구현됨 |
| `0x06` | FDSETTING | FD 비트타이밍 프리셋 저장(D1: 1~4). **주의: 재부팅해야 반영** — [9장](#9-주의사항) | 구현됨 |
| `0x07` | CLEARERROR | 이름·주석만 있고 실제 처리 코드 없음(ERRPKT_ENABLE 기능과 묶여 예약된 것으로 추정) | 예약(미구현) |
| `0x08` | GETSTATUS | 이름·주석만 있고 실제 처리 코드 없음 | 예약(미구현) |
| `0x09` | IMU_APPEND | D1: 1=OFF 2=ON. `IMU_MPUXX50` 빌드에서만 컴파일됨. [5장](#5-imu-부가-프레임-신규-2026-08-27) 참조 | 신규 (2026-08-27) |

## 3. TRANSMITTING 데이터타입 (D0=0x03, D1)

| 모드 | D1 | 이름 | 내용 | 프레임(들) |
|---|---|---|---|---|
| CAN 2.0 | `0x01` | INT6_WOT_20 | 온도보상 없이 F/T, int16×6 | `can20ID`+0(force)/+1(torque) |
| CAN 2.0 | `0x02` | INT6_WT_20 | 온도보상 포함 F/T, int16×6 | `can20ID`+0(force)/+1(torque) |
| CAN FD | `0x01` | INT6_WOT_FD | 온도보상 없이 F/T, int16×6 | `canFDID`+0(force)/+1(torque) |
| CAN FD | `0x02` | INT6_WT_FD | 온도보상 포함 F/T, int16×6 | `canFDID`+0(force)/+1(torque) |
| CAN FD | `0x03` | INT12_WOT_FD | 온도보상 없이 F/T 한 프레임(int16×6, 12B) | `canFDID`+2 (1프레임) |
| CAN FD | `0x04` | INT12_WT_FD | 온도보상 포함, 위와 동일 레이아웃 | `canFDID`+2 (1프레임) |
| CAN FD | `0x05` | FLO_WOT_FD | 온도보상 없이 F/T 한 프레임(float32×6, 24B) | `canFDID`+2 (1프레임) |
| CAN FD | `0x06` | FLO_WT_FD | 온도보상 포함, 위와 동일 레이아웃 | `canFDID`+2 (1프레임) |

DLC는 `ERRPKT_ENABLE`(현재 `DISable`) 여부에 따라 6B/12B/24B(꺼짐) ↔ 8B/16B/32B(켜짐, 끝에
에러워드 2B+패딩)로 바뀝니다. [7장](#7-측정-데이터-프레임-포맷-송신-센서호스트) 참조.

## 4. TX_MODE / UPDATERATE / FDSETTING 값표

| 명령 | D1 | 의미 |
|---|---|---|
| TX_MODE (0x04) | `0x01` | CAN_MODE_20 (클래식) |
| TX_MODE (0x04) | `0x02` | CAN_MODE_FD |
| TX_MODE (0x04) | `0x03` | CAN_MODE_FD_BRS |
| UPDATERATE (0x05) | `0x01` | 100Hz |
| UPDATERATE (0x05) | `0x02` | 250Hz |
| UPDATERATE (0x05) | `0x03` | 500Hz |
| UPDATERATE (0x05) | `0x04` | 1000Hz |
| FDSETTING (0x06) | `0x01` | FDSET_SET1 |
| FDSETTING (0x06) | `0x02` | FDSET_SET2 |
| FDSETTING (0x06) | `0x03` | FDSET_SET3 (기본값과 동일 타이밍) |
| FDSETTING (0x06) | `0x04` | FDSET_SET4 |

## 5. IMU 부가 프레임 (신규, 2026-08-27)

`IMU_MPUXX50` 빌드 토글이 켜져 있을 때만 존재하는 기능입니다(현재 기본 `DISable`). MPU-9250(I2C1)에서
매 센서 사이클 읽은 가속도/자이로 raw값을, `0x09` 명령으로 켜고 끕니다.

| D0 | D1 | 동작 |
|---|---|---|
| `0x09` | `0x01` | IMU 부가 프레임 OFF |
| `0x09` | `0x02` | IMU 부가 프레임 ON — 이후 매 전송주기마다 가속도/자이로 프레임 추가 송신 |

선택된 F/T 데이터타입(3장)과 무관하게 독립적으로 켜고 끕니다. 프레임 포맷은 [7장](#7-측정-데이터-프레임-포맷-송신-센서호스트) 참조.

## 6. 특수 매직 명령

아래는 `Check_txcanID()` ID 검사 없이(byte 0-1이 TX ID가 아니라 고정 매직 시퀀스) 별도로 처리됩니다.

| 바이트 시퀀스 | 이름 | 동작 |
|---|---|---|
| `FF FE FD` | FACTORYRESET | 설정 전체 flash 기본값으로 초기화(`setMem=1`) |
| `FF FE FC 01` | CONFIRMID (CAN20) | `can20ID`로 빈 프레임(DLC0) 응답 → 호스트가 실제 TX ID 확인 |
| `FF FE FC 02` | CONFIRMID (FD) | `canFDID`로 빈 프레임(DLC0) 응답 |
| `FF FE FA 01`(또는 02 외 값) | ERRPKT ON | 에러패킷 탑재 켬(flash 저장), `ERRPKT_ENABLE` 빌드에서만 의미있음 |
| `FF FE FA 02` | ERRPKT OFF | 에러패킷 탑재 끔(flash 저장) |
| `FF FE FB 5A` | ENTERBL | CAN 부트로더 진입 핸드셰이크 후 리셋(`USE_BOOTLOADER` 빌드에서만) |
| `F1 F2 F3 F4 FC FD FE FF` | LOGDUMP | 전송 멈추고 저장된 로그 전량 CAN 덤프(`setLogDump=1`) |

## 7. 측정 데이터 프레임 포맷 (송신, 센서→호스트)

전부 리틀엔디안(LSB 먼저). `i16`는 F/T는 force×100 / torque×1000 스케일, IMU는 raw(스케일 없음).

**INT6** (CAN20 `can20ID`+0/+1, FD `canFDID`+0/+1)
```
force 프레임(+0): [0-1]=forceX  [2-3]=forceY  [4-5]=forceZ  [6-7]=에러워드(ERRPKT ON 시만, 없으면 DLC6)
torque프레임(+1): [0-1]=torqueX [2-3]=torqueY [4-5]=torqueZ [6-7]=에러워드(ERRPKT ON 시만, 없으면 DLC6)
```

**INT12 Combined** (FD `canFDID`+2, 1프레임)
```
[0-1]=forceX [2-3]=forceY [4-5]=forceZ [6-7]=torqueX [8-9]=torqueY [10-11]=torqueZ
[12-13]=에러워드 [14-15]=패딩0  (ERRPKT ON 시만, 없으면 DLC12)
```

**FLOAT Combined** (FD `canFDID`+2, 1프레임)
```
[0-3]=forceX(f32) [4-7]=forceY(f32) [8-11]=forceZ(f32)
[12-15]=torqueX(f32) [16-19]=torqueY(f32) [20-23]=torqueZ(f32)
[24-25]=에러워드 [26-31]=패딩0  (ERRPKT ON 시만, 없으면 DLC24)
```

**IMU 부가 프레임** (CAN20 `can20ID`+3/+4, FD `canFDID`+3/+4) — 신규
```
가속도(+3): [0-1]=accelX(raw i16) [2-3]=accelY(raw) [4-5]=accelZ(raw) [6-7]=패딩0
자이로(+4): [0-1]=gyroX(raw i16)  [2-3]=gyroY(raw)  [4-5]=gyroZ(raw)  [6-7]=패딩0
※ MPUXX50_Init() 고정 레인지 기준: accel ±16g = 2048 LSB/g, gyro ±2000dps = 16.4 LSB/(deg/s)
```

## 8. 기본값

| 항목 | 값 |
|---|---|
| 수신(명령) ID — CAN 2.0 | `0x220` |
| 수신(명령) ID — CAN FD | `0x320` |
| 송신(측정) 기본 ID — CAN 2.0 (`can20ID`) | `0x230` |
| 송신(측정) 기본 ID — CAN FD (`canFDID`) | `0x330` |
| 부트로더 UDS ID (요청/응답) | `0x7A0` / `0x7A8` |

## 9. 주의사항

> **FDSETTING(0x06)은 즉시 반영되지 않는다**
> 명령을 받으면 `ADManager.fdcan2.FDSetting` 값만 갱신하고 실제 `FDCAN2_Reconfigure()` 호출은
> 코드에서 주석 처리돼 있습니다. 현재는 flash에 저장된 값이 **다음 부팅 시** 한 번 적용되는
> 구조입니다 — 런타임에 바로 타이밍이 바뀌길 기대하면 안 됩니다.

> **CLEARERROR(0x07) / GETSTATUS(0x08)는 이름만 있고 미구현**
> 상수 정의와 주석만 있고, 명령 디스패치에는 대응하는 처리 코드가 없습니다. 보내도 무시됩니다.
> `ERRPKT_ENABLE` 기능(현재 `DISable`)이 켜지면서 같이 구현될 예정으로 추정 — 실제 사용 전 확인 필수.

이 문서는 2026-08-27 기준 펌웨어 소스 전수 대조로 작성했습니다. 코드가 바뀌면 이 문서도 같이 갱신해야 합니다.
