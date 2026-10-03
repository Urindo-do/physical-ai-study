# DH-Robotics AG-95 그리퍼 — USB 변환기로 다루기

> 지식축 노트. "그 그리퍼 어떻게 켜더라"를 찾을 때 보는 곳이다.
> 날짜별 시행착오는 [`assignments/seedling-transplant/practice-log.md`](../assignments/seedling-transplant/practice-log.md) (2026-09-30 ~ 10-01).
> 표기: ✅ 실물로 확인 / 📄 설명서·코드 근거 / 💡 추정 / ❓ 미확인

---

## 1. 연결 구조

```
AG-95 ─(항공 플러그 케이블)─ USB 프로토콜 변환기 ─(USB)─ 노트북 (/dev/ttyACM0)
                                   │
                                  24 V DC
```

| 항목 | 값 | 확인 |
|---|---|---|
| 변환기 DIP 스위치 | `1000` = USB 모드 | 📄 변환기 설명서 4.3절 |
| USB 장치 | `0483:5740` STM32 Virtual ComPort, `cdc_acm` 드라이버 | ✅ `lsusb`, `sudo dmesg -w` |
| 장치 파일 | `/dev/ttyACM0` (고정 경로는 `/dev/serial/by-id/usb-STMicroelectronics_STM32_Virtual_ComPort_in_FS_Mode_00000000050C-if00`) | ✅ `ls /dev/serial/by-id` |
| 통신 | 115200 bps 8N1, 그리퍼 ID `2` | ✅ 응답 옴 |
| 권한 | `dialout` 그룹 | ✅ `groups` |
| 전원 | 24 V. 실사용 약 0.12~0.13 A | ✅ 전원 공급기 표시 |

- ⚠️ **변환기는 24 V가 들어와야 USB 장치로 잡힌다.** USB만 꽂으면 `lsusb`에도 안 보이고 `dmesg`에도 아무 기록이 없다 ✅
- `/dev/ttyACM0`은 꽂는 순서에 따라 번호가 바뀔 수 있다 💡 — 여러 장치를 꽂을 땐 `by-id` 경로를 쓴다

## 2. 통신 프로토콜 — Modbus RTU가 아니다

USB 변환기를 쓰면 **DH 자체 프레임**으로 말한다 (📄 `ag95.pdf` 3.3~3.5절, 연구실 코드 `transplant_gripper/protocol.py`).
RS-485 직결용 "Short Manual v2.1 (Modbus RTU)"의 주소(0x0100 …)·주소 1·천분율(‰) 표는 **이 연결에 그대로 쓰지 않는다.**

```
ff fe fd fc | ID | 명령 2바이트 | 데이터 … | fb
머리         02   예: 08 02                꼬리   (전체 14바이트)
```

| 명령 | 뜻 | 응답 해석 | 확인 |
|---|---|---|---|
| `08 02` | 초기화 여부 | 0 = 안 됨 / 1 = 됨 | ✅ |
| `06 02` | 위치 (0~100 %) | ⚠️ **목표값으로 보인다** — 아래 §3 | ✅ |
| `0F 01` | 상태 | 0 = 이동 중 / 2 = 도착 / 3 = 물체 잡음 | ✅ (0·2·3 관측) |
| `13 01` | 펌웨어 버전 | 데이터 `03 13 02 07` | ✅ |

- 위치: **0 = 닫힘, 100 = 열림** (`ag95_cli.py` 기준). 50 → 100 이동 약 1.2초 ✅
- ❓ 상태 1(Modbus 표에선 "도착")이 이 프로토콜에서 무엇인지, 속도 설정 명령이 있는지 — 미확인

## 3. 레지스터 특성 — 설명서에 없는 것

- **위치 응답은 실제 손가락 위치가 아니라 목표값으로 보인다.** 이동 중에도 이미 목표값을 보고하고,
  물리적으로 닫혀 있을 때 100을 보고한 적도 있다 → **동작 완료는 상태(`0F 01`)로 판단한다** ✅
- **초기화 전에는 위치 명령을 무시한다** ✅
- **초기화 전에는 힘 설정이 저장되지 않는다** — 30을 쓰면 30으로 응답하지만 다시 읽으면 0.
  전원 직후 0, 초기화 실패 뒤 100으로 읽힘 ✅
- **초기화는 한 번에 안 될 때가 많다 — 같은 명령을 반복하면 된다** ✅
  - 실패 모습: 손가락이 끝까지 닫혀 꽉 조인 채 멈춤, LED 빨간불, init = 0, 상태 = 3(물체 잡음). 모터가 계속 힘을 줘서 따뜻해진다
  - 성공 모습: 손가락이 열리고 LED **파란불**
  - 연구실 원래 코드도 1초 간격 5번 반복한다 (📄 `docs/SYSTEM_AUDIT.md` 4.2절, `ag95_node.py` `_initialize` 주석). **반복을 없애지 않는다**
  - 원인에서 제외한 것: 명령 바이트·순서, 통신, 전류 한계, 전원 껐다 켜기, 꺼 둔 시간, 포트 점유, 부품 걸림 (근거는 practice-log 10/01)
  - 근본 원인 ❓ — 제조사 디버깅 프로그램(Windows)과 비교 전

## 4. `ag95_cli.py` — ROS 없이 직접 조작

연구실 저장소의 `scripts/ag95_cli.py` (`pyserial` + `protocol.py`). **반드시 저장소 최상위에서 실행한다.**

```bash
cd ~/Desktop/TransplantingRobot
python3 scripts/ag95_cli.py status                  # 읽기만 함 (안 움직임)
python3 scripts/ag95_cli.py init                    # 성공할 때까지 최대 8번 자동 반복 (1회 약 12초)
python3 scripts/ag95_cli.py open                    # 100% 열기
python3 scripts/ag95_cli.py close                   # 0% 닫기, 물체를 잡았는지 알려줌
python3 scripts/ag95_cli.py 40                      # 원하는 위치 (0~100%)
python3 scripts/ag95_cli.py jog --force 30 --step 1 # 키보드로 조작
```

| 옵션 | 뜻 |
|---|---|
| `--force 30` | 닫는 힘 20~100 %. 모종 · 스펀지는 낮은 값부터 |
| `--tries 8` | `init` 최대 시도 횟수 |
| `--step 1` | `jog`에서 키 한 번에 움직이는 양(%) |
| `--min 0 --max 100` | `jog`에서 넘지 않을 범위. `--min 20`으로 실행하면 20 %에서 멈춘다 |
| `--port`, `--id` | 기본 `/dev/ttyACM0`, `2` |

`jog` 키: `c` 누르고 있으면 닫힘, `o` 누르고 있으면 열림, 떼면 멈춤, `q` 종료.
0·100 %에 닿으면 `limit reached`, 물체를 잡으면 `object caught`가 뜨고 더 닫지 않는다.
키를 꾹 누르면 처음 약 0.5초는 한 번만 움직이고 그 뒤 연속 — 키보드 자동 반복 때문이다.

## 5. 운영 순서 — 매번 이렇게

1. 변환기에 24 V를 켠다 → LED **빨간불** (전원을 켜면 항상 미초기화)
2. `ls /dev/ttyACM0`로 연결 확인
3. ⚠️ 손가락 사이를 비우고 `python3 scripts/ag95_cli.py init` → **파란불**까지 기다린다 (손끝이 칼날이다)
4. `python3 scripts/ag95_cli.py jog --force 30 --step 1`로 작업
5. 끝나면 `python3 scripts/ag95_cli.py open`으로 열어 두고 24 V를 끈다

초기화가 계속 실패하면 닫힌 채 힘을 주는 상태로 오래 두지 않는다(모터 발열) — 24 V를 끄고 식힌 뒤 다시.

## 6. 연구실 웹 운영 콘솔과의 관계

```bash
cd ~/Desktop/TransplantingRobot
./scripts/start_operator_console.sh     # 빌드 → 실행 → http://127.0.0.1:8024/
./scripts/stop_all.sh                   # 끄기
```

- 콘솔은 그리퍼를 **mock으로 고정**해서 띄운다 (`phase1_offline.launch.py`의 `{'transport': 'mock'}`) →
  **실물 그리퍼는 안 움직이고** `/dev/ttyACM0`도 안 쓴다
- 슬라이더 위치: **Transplant Operation** → "Joint Angles & Gripper Waveform" → **🎛️ Joint Override**. RViz 속 그리퍼만 움직인다
- ⚠️ 이 창에서는 **0 % = 완전히 열림, 100 % = 완전히 닫힘** — `ag95_cli.py`와 **반대**
- OPEN/CLOSE 버튼은 서버 연결 상태에서 "데모 전용"으로 잠겨 있다. 초기화 버튼은 없다
- 실물로 띄우려면 ROS 노드에 `transport:=serial` (아래 다음에 할 일)

## 다음에 할 일

- [ ] `init` 자동 반복 버전을 실물로 확인 — 보통 몇 번째에 성공하는지 횟수도 적는다
- [ ] 실제 스펀지로 `object_caught` 판정 확인, 폼에 맞는 `--force` 범위 찾기 (checklist P1-6)
- [ ] ROS 노드를 실물로: `ros2 run transplant_gripper ag95_node --ros-args --params-file ros2_ws/src/transplant_gripper/config/ag95.yaml -p transport:=serial` — `init_attempts`(기본 5)로 충분한지
- [ ] 상태 1의 뜻, 속도 명령 유무 — `ag95.pdf`에서 확인해 §2 표 채우기
- [ ] (선택) DH 디버깅 프로그램(Windows)으로 초기화 비교 → 반복이 하드웨어 특성인지 확정
