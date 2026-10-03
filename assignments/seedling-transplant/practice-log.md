# 수경재배 모종 이식 — 시험 기록 (practice-log)

> 날짜별 시험 기록. **추가만 한다** (과제 `CLAUDE.md` §3, 루트 `CLAUDE.md` §7).
> 판정 기준의 원본은 [`plan.md`](./plan.md), 진도의 원본은 [`checklist.md`](./checklist.md) 체크박스다.
> 그리퍼를 다루는 방법(명령 · 운영 순서 · 레지스터 특성)은 [`notes/ag95-gripper.md`](../../notes/ag95-gripper.md)에 따로 정리했다.

- 작업 PC: `urindodo` 노트북 (Ubuntu 22.04, ROS 2 Humble)
- 연구실 코드: `~/Desktop/TransplantingRobot` (ROS 2 워크스페이스 `ros2_ws/`, 이 저장소 밖)
- 대상: DH-Robotics AG-95 + USB 프로토콜 변환기. 로봇 팔(UR10e) · 카메라는 연결하지 않음

---

## 2026-09-30

### 준비 (P 밖) — 연구실 코드 빌드 · mock 실행 — ✅
- 한 일: 연구실 코드(`TransplantingRobot`)의 의존성을 깔고 빌드한 뒤, 하드웨어 없이(mock) 전체 스택을 띄워봄
- 결정: **conda를 쓰지 않음** — ROS 2 Humble은 시스템 Python 3.10 기준이라 conda 안에서 `colcon build`·`ros2 launch`를 하면
  `rclpy`·`libstdc++`가 섞여 오류가 나기 쉽다. 필요한 것은 거의 다 apt 패키지였고(pip는 선택 사항 `pillow`뿐), 이 PC엔 conda도 없었다
- 명령 · 설정:
  ```bash
  sudo apt update && sudo apt install -y \
    ros-humble-ur-robot-driver ros-humble-ur-description \
    ros-humble-controller-manager ros-humble-joint-state-publisher-gui \
    python3-serial python3-rosdep
  sudo usermod -aG dialout $USER     # 다시 로그인해야 적용됨
  cd ~/Desktop/TransplantingRobot/ros2_ws && source /opt/ros/humble/setup.bash && colcon build
  source scripts/env.sh
  ros2 launch transplant_bringup phase1_offline.launch.py use_rviz:=false run_motion_demo:=false hold_pose:=false
  ```
  - 설치 대상은 모든 `package.xml` 의존성과 대조해 **빠진 것만** 골랐다
  - `apt`·`usermod`는 어느 폴더에서 실행해도 되지만 `colcon build`는 반드시 `ros2_ws`에서
- 결과:
  - `Summary: 10 packages finished [17.7s]`
  - mock 25초 실행 — 노드 29개(로봇 제어 · `/ag95_gripper` · 비전 · 작업 관리 · 웹 백엔드 등), `http://127.0.0.1:8024/` HTTP 200
  - 경고는 `Could not enable FIFO RT scheduling policy` 하나 — 실시간 스케줄링 권한이 없다는 뜻. 실물 로봇 없이 돌릴 땐 무시해도 된다
- 다음: 실물 그리퍼 연결

### P1-1 전원 — ⏸ 보류 (측정값 없음)
- 한 일: 실험용 전원 공급기로 변환기에 24 V 공급, 전류 한계 **2.0 A** (10/01에 1.5 A로도 시험)
- 판정: ✔ 끝 기준("측정 전압이 기록됨") **미충족** — 멀티미터 측정값을 기록하지 않았다.
  쓴 전원 장치가 ToolkitRC P200인지도 기록에 없다 (미확인)
- 다음: 멀티미터로 출력 전압을 재서 이 항목에 이어 적는다

### P1-2 USB 연결 · 장치 인식 — ✅ 통과
- 한 일: 그리퍼 ─(항공 플러그 케이블)─ USB 프로토콜 변환기 ─(USB)─ 노트북. 변환기에 24 V
- 명령 · 설정:
  ```bash
  lsusb                      # STMicroelectronics 장치가 보여야 함
  ls -l /dev/ttyACM0         # crw-rw---- root dialout
  sudo dmesg -w              # 꽂는 순간 "cdc_acm ... ttyACM0: USB ACM device"
  groups                     # dialout 포함
  ```
  - `lsusb`: 연결된 USB 장치 목록 (윈도우의 장치 관리자 → USB 컨트롤러와 비슷)
  - `dmesg -w`: 커널 메시지를 실시간으로 계속 보여준다(`-w` = wait). 꽂는 순간 커널이 무엇으로 인식했는지 보인다
  - `groups`: 내 계정이 속한 그룹. 시리얼 장치는 `dialout` 그룹만 읽고 쓸 수 있다
- 결과:

  | 항목 | 값 |
  |---|---|
  | 변환기 DIP 스위치 | `1000` (USB 모드, 변환기 설명서 4.3절) |
  | USB 장치 | `0483:5740` STMicroelectronics "STM32 Virtual ComPort in FS Mode", 시리얼 번호 `00000000050C` |
  | 장치 파일 | **`/dev/ttyACM0`** — 고정 경로 `/dev/serial/by-id/usb-STMicroelectronics_STM32_Virtual_ComPort_in_FS_Mode_00000000050C-if00` |
  | 드라이버 | `cdc_acm` (리눅스 기본 USB 시리얼 드라이버 — CH340·FTDI 같은 별도 변환 칩이 아님) |
  | 통신 | 115200 bps 8N1, 그리퍼 ID `2` |
  | 권한 | `dialout` — 다시 로그인한 뒤 적용 |

- 처음에 인식이 안 됐던 이유: **변환기에 24 V가 안 들어오고 있었다.** 이때 `lsusb`엔 노트북 내장 장치 3개만 보였고,
  `dmesg -w`를 켠 채 다시 꽂아도 USB 기록이 전혀 없었다 → 변환기는 USB 전원만으로는 켜지지 않는다.
  전원선(power)과 신호선(status)을 연결하자 바로 `ttyACM0`이 생겼다
- 체크리스트 예상과 달랐던 것: 💡`/dev/ttyUSB*`가 아니라 `/dev/ttyACM*`. `cdc_acm` 장치라 `brltty` 문제(CH340 계열)는 해당 없음
- 판정: ✔ 끝("장치 이름이 기록됨") — **통과**
- 다음: P1-3 초기화

### P1-3 초기화 — ✅ 통과 (4번째 시도) · 반복이 필요함
- 한 일: 먼저 그리퍼를 움직이지 않는 **읽기 명령**만 보내 통신을 확인. 연구실 코드
  `ros2_ws/src/transplant_gripper/transplant_gripper/protocol.py`의 명령 프레임을 그대로 썼다
- 결과 — 통신:
  ```
  초기화 여부  tx=ff fe fd fc 02 08 02 00 00 00 00 00 00 fb  rx=... 08 02 00 00 00 00 00 00 fb  -> 0 (미초기화)
  위치        tx=ff fe fd fc 02 06 02 00 00 00 00 00 00 fb  rx=... 06 02 00 00 64 00 00 00 fb  -> 100
  상태        tx=ff fe fd fc 02 0f 01 00 00 00 00 00 00 fb  rx=... 0f 01 00 00 02 00 00 00 fb  -> 2 (정지)
  ```
  - 프레임이 제조사 설명서(`ag95.pdf` 3.3~3.5절)와 바이트 단위로 일치. 예전 스크립트 `command_position1.py`의 프레임과도 같다
  - 펌웨어 버전도 읽힌다 (`13 01` 응답 데이터 `03 13 02 07`)
  - ⚠️ **체크리스트의 매뉴얼 표(Modbus RTU, 주소 1, 0x0100 …)와 다른 프로토콜이다.** USB 변환기는 DH 자체 프레임
    (`ff fe fd fc` 머리 · `fb` 꼬리)을 쓰고 그리퍼 ID는 `2`, 위치는 0~100(%)이다. 정리는 [`notes/ag95-gripper.md`](../../notes/ag95-gripper.md) §2
- 결과 — 초기화 시도:

  | # | 한 것 | 결과 |
  |---|---|---|
  | 1 | 초기화 명령 후 0.6초마다 완료 여부(`08 02`) 질의 | ❌ 닫히고 멈춤 |
  | 2 | 위치 50/100/0 명령 후 초기화 | ❌ 위치 명령은 무시됨, 닫히고 멈춤 |
  | 3 | 초기화 명령 한 번 보내고 12초 동안 아무것도 안 보냄 | ❌ 살짝 움직이고 다시 닫힘 |
  | 4 | 24 V 껐다 켬 (약 66초 꺼둠) → 3과 같은 방식 | ✅ **성공** (init 0→1, 파란불) |

  - 실패 증상: 손가락이 끝까지 닫혀 꽉 조인 채 멈춤. LED 빨간불 그대로, init = 0, 상태 = 3(물체 잡음).
    모터가 닫는 방향으로 계속 힘을 줘서 **따뜻해진다** (24 V에서 약 0.12~0.13 A). 이미 닫힌 상태에서 다시 보내면 "움찔"만 함
  - 성공 모습: 손가락이 열리고 LED가 **파란불**
- 판정: ✔ 끝("초기화 동작 + 초기화 완료 상태가 읽힘") — **통과**. 단 한 번에 되지 않는다 (10/01에 이어 적음)
- 다음: 10/01에 재현

### P1-4 위치 이동 — ⚠️ 일부만 (초기화 후)
- 결과: 위치 50% / 100% 명령에 약 **1.2초** 만에 도착
- 알게 된 것 (레지스터):
  - **위치 레지스터(`06 02`)는 실제 손가락 위치가 아니라 목표값으로 보인다** — 이동 중에도 이미 목표값을 보고했고,
    손가락이 물리적으로 닫혀 있을 때 100을 보고한 적도 있다. 동작 완료는 상태(`0F 01`)로 판단해야 한다
  - **힘은 초기화 전에 설정해도 저장되지 않는다** — 30을 쓰면 30으로 응답하지만 다시 읽으면 0.
    전원 직후 0, 초기화 실패 뒤엔 100으로 읽혔다
  - **초기화 전에는 위치 명령을 무시한다** (시도 #2)
- 판정: ✔ 끝("명령값과 읽은 값이 표로") **미충족** — 손끝 간격 실측 · 힘별 비교 · 상태 네 가지 만들기가 남음
- 다음: 자로 위치별 간격 재기, 스펀지로 힘 비교

---

## 2026-10-01

### P1-3 초기화 — 재현 ✅ (여러 번 반복 후)

| # | 한 것 | 결과 |
|---|---|---|
| 1 | 전원 켠 직후 `ag95_cli.py init` | ❌ 꽉 닫히고 멈춤 |
| 2 | 24 V 껐다 켬 → `init` | ❌ |
| 3 | 손가락을 손으로 억지로 벌림 → 껐다 켬 → 어제 성공한 순서와 똑같이 고친 `init` | ❌ |
| 4 | 힘을 30%로 설정한 뒤 초기화 | ❌ 초기화 전이라 힘 설정이 저장되지 않음 |
| 5 | 전원 전류 한계를 2.0 A로 올림 → `init` | ❌ |
| 6 | 전류 한계 1.5 A → `init` → 실패 → **그냥 반복** | ✅ **성공** (24 V를 다시 끄지 않았음. 정확한 반복 횟수는 기록 안 함) |

- **원인에서 제외한 것** (근거 있음):

  | 후보 | 근거 |
  |---|---|
  | 명령 바이트 · 순서 | 설명서와 같고, 성공한 실행과 실패한 실행의 순서가 똑같다 |
  | 통신 · 신호선 | 모든 명령에 올바른 응답. 초기화 명령에 실제로 움직인다 |
  | 전원 전류 한계 | 실제 0.13 A인데 한계는 2.0 / 1.5 A |
  | 전원 껐다 켜기 | 켠 직후에도 실패했고, 성공은 껐다 켜지 않고도 났다 |
  | 꺼 둔 시간 | 성공 66초, 실패 20~97초 — 차이 없음 |
  | 다른 프로그램의 포트 점유 | `fuser`로 확인 — 없음. ModemManager는 연결될 때만 확인하고 "지원 안 함"으로 손을 뗀다 |
  | 손가락 끝 부품 걸림 | 육안 확인 — 걸리지 않음 |

- 결론: **같은 초기화 명령을 여러 번 보내야 성공하는 하드웨어 특성**으로 본다
  - 연구실 원래 개발자도 같은 문제를 겪었다 — 예전 코드는 `initialize.py`를 1초 간격으로 5번 실행했고,
    검증 없이 이 반복을 없애지 말라고 경고해 뒀다 (`docs/SYSTEM_AUDIT.md` 4.2절, `ag95_node.py`의 `_initialize` 주석)
  - 근본 원인은 **미확인** — 제조사 디버깅 프로그램(Windows)으로는 아직 비교하지 않았다
  - 연구실 코드 `docs/KNOWN_ISSUES.md` 26번에 같은 내용 기록
- **정정**: 작업 도중 `ag95_node`의 반복 초기화를 "설명서와 맞지 않아 고쳐야 할 부분"이라고 판단한 적이 있다.
  틀린 판단이었다. 반복은 이 하드웨어에 필요하므로 그대로 둔다
- ⚠️ 초기화가 계속 실패하면 닫힌 채 힘을 주는 상태로 오래 두지 않는다 (모터 발열) — 24 V를 끄고 식힌 뒤 다시

### P1-4 위치 · 키보드 조작 — ⚠️ 일부만
- 한 일: `ag95_cli.py jog`로 키보드로 열고 닫기 — 사용자가 직접 사용
- 결과: 0%나 100%에 닿으면 `limit reached`. 처음에 `--min 20`으로 실행해서 20%에서 멈춘 적이 있다(옵션 때문, 고장 아님)
- 0 = 닫힘, 100 = 열림 (`ag95_cli.py`의 `open`=100 · `close`=0 정의, `jog`로 열고 닫으며 사용)
- 미확인: 물체 잡기 판정(`object_caught`)을 **실제 물체로는 아직 확인하지 않음**
- 판정: ✔ 끝 **미충족** (9/30과 같음)

### P1-5 파이썬 스크립트 — ⚠️ 일부만 (`ag95_cli.py`)
- 한 일: ROS 없이 실제 그리퍼를 직접 조작하는 `scripts/ag95_cli.py` 작성 (연구실 코드 저장소 안, 아직 커밋 안 됨)
- 라이브러리: **`pyserial` 직접** + 연구실 코드의 `protocol.py` 프레임 재사용.
  고른 이유 — 이 변환기는 Modbus가 아니라 DH 자체 프레임이라 `minimalmodbus`·`pymodbus`가 맞지 않고, 프레임 코드가 이미 있었다
- 동작: `status` / `init`(성공할 때까지 최대 8번 자동 반복, 1회 약 12초) / `open` / `close` / 숫자 위치 / `jog`.
  포트 · ID는 `--port`(기본 `/dev/ttyACM0`) · `--id`(기본 `2`) 옵션. 사용법은 [`notes/ag95-gripper.md`](../../notes/ag95-gripper.md) §4
- 판정: ✔ 끝("스크립트 **한 번 실행**으로 열기 → 닫기 → 상태 출력") **미충족** — 명령이 하나씩 따로다.
  또 체크리스트가 정한 위치(`code/gripper/`)가 아니라 연구실 저장소에 있다
- 미확인: `init` **자동 반복 버전은 아직 실물로 안 돌렸다** (코드만 고침)
- 다음: 자동 반복 `init`을 실물로 확인

### 겪은 실수와 해결

| 증상 | 원인 | 해결 |
|---|---|---|
| `can't open file '.../scripts/scripts/ag95_cli.py'` | `scripts` 폴더 안에서 `scripts/ag95_cli.py`를 실행 | 저장소 최상위(`~/Desktop/TransplantingRobot`)에서 실행 |
| `^[[200~sg: command not found` | 붙여넣을 때 보이지 않는 문자(브래킷 붙여넣기 표시)가 섞임 | 다시 입력 |
| `Command 'python' not found` | Ubuntu에는 `python` 명령이 없음 (윈도우와 다름) | `python3` |
| `Permission denied: /dev/ttyACM0` (가능성) | `dialout` 권한이 아직 적용 안 됨 | 다시 로그인. 그전엔 `sg dialout -c "python3 ..."` |
| USB가 12초 동안 7번 다시 연결됨 (18:02) | 케이블을 다시 꽂는 중 접촉 불안정 | 단단히 꽂음. 이후 정상 |

### 회고
- 계획 대비: P1-2 · P1-3 통과. P1-4 · P1-5는 일부, P1-1 · P1-6은 아직. 체크리스트는 Modbus RTU를 가정했는데
  실물은 USB 변환기의 자체 프레임 프로토콜이었다 — 매뉴얼 표가 이 연결엔 맞지 않는다
- 시간이 든 곳: 초기화. 이틀 동안 원인을 하나씩 지웠고, 결국 "반복하면 된다"는 이미 알려진 특성이었다.
  **연구실 코드의 주석과 `SYSTEM_AUDIT.md`를 먼저 읽었으면** 더 빨리 끝났다
- 같은 방법이 2번 실패하면 멈추고 가설부터 — 이번엔 가설을 하나씩 지운 기록(제외 표)이 남아서 다음에 쓸 수 있다

### 다음에 할 일
- [ ] P1-1 멀티미터로 전원 출력 전압 측정 · 기록
- [ ] `init` 자동 반복 버전을 실물로 확인
- [ ] P1-4 위치 0/50/100 손끝 간격 실측, 힘 20/60/100 비교, 상태 네 가지 만들기
- [ ] P1-6 실제 스펀지 조각으로 `close`/`jog`의 `object_caught` 판정 확인, 적당한 `--force` 찾기
- [ ] ROS 노드를 실물로: `ros2 run transplant_gripper ag95_node --ros-args --params-file ros2_ws/src/transplant_gripper/config/ag95.yaml -p transport:=serial` — `init_attempts`(기본 5, 1초 간격)로 충분한지
- [ ] (선택) DH 디버깅 프로그램(Windows)으로 첫 시도 초기화가 되는지 비교 — 거기서도 반복이 필요하면 하드웨어 특성으로 확정
- [ ] 연구실 저장소: `docs/AG95_LIVE_CHECK.md` · `scripts/ag95_cli.py` · `docs/KNOWN_ISSUES.md` 커밋, 바탕화면 런처의 옛 경로와 README Layout 섹션 정리
