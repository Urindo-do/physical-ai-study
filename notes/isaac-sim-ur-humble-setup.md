# Isaac Sim + ROS 2 Humble로 UR 구동하기 — 내 환경 기준 설치·실행 가이드

> **이 문서의 위치**: 지식축(`notes/`). "그거 어떻게 깔았더라 / 왜 이렇게 했더라"를 찾는 레퍼런스다.
> 실제로 따라 친 날의 출력·시행착오는 그날의 `study-log/`에, 설치가 끝나 **확정된 값**(경로·버전·alias)은
> `docs/environment.md`에 옮긴다. **값이 서로 다르면 `docs/environment.md`가 이긴다.**
>
> - 최초 작성: 2026-10-01
> - 근거: ① 수업 자료 `2026 바이오시스템로봇공학실습 — UR 구동 방법` (HRA Lab, PDF 56쪽) — **절차 참고용**
>   ② Isaac Sim **공식 문서(6.1 기준)** — **명령·버전의 기준** ③ 이 저장소의 `docs/environment.md` — **내 설정의 기준**
> - 표기: ✅ 확인함(실행 또는 공식 문서로 확인) / ⚠️ 미확인(추정이거나 내 노트북에서 아직 안 해봄)

---

## 0. 한눈에 보기

**목표**: 리눅스 노트북 한 대에서 Isaac Sim ↔ ROS 2 Humble ↔ MoveIt 2를 연결해
① Franka MoveIt 예제 → ② UR3 시뮬레이션 → ③ 실제 UR3 순서로 구동한다.

**수업 자료와 내 환경이 다른 점**

| 항목 | 수업 자료 | 내 환경 | 결과 |
|---|---|---|---|
| OS | Windows + VMware 안의 Ubuntu 24.04 | **Ubuntu 22.04.5 단일** | 슬라이드 2~24(VMware·Ubuntu 설치), 46(VM USB/네트워크) **건너뜀** |
| ROS 2 | Jazzy | **Humble** | 패키지 이름의 `jazzy` → `humble` |
| Isaac Sim | 5.1.0 (Windows, App Selector) | **6.1.0 (Linux)** | 실행 방법·일부 노드·에셋 경로가 다름 (§4, §6) |
| ROS 켜는 방식 | `.bashrc`에 `source` 자동 추가 | **자동 소싱 안 함, `humble` alias** | **그대로 유지**. 오히려 Isaac Sim에 필요한 구조 (§2) |
| 통신 설정 | fastdds.xml(UDP 강제), `ROS_DISABLE_SHARED_MEMORY=1` | 한 컴퓨터 안 | **둘 다 안 씀** (§2-2) |
| ROS_DOMAIN_ID | 조 번호 | **13** (`ros_domain` alias) | Isaac Sim 쪽도 13으로 맞춘다 |

---

## 1. 내 노트북 vs Isaac Sim 요구사항

2026-10-01, Isaac Sim **Compatibility Checker 6.1.0** 실측 ✅

| 항목 | 내 값 | 공식 최소 | 판정 |
|---|---|---|---|
| GPU | GeForce RTX 2060 (노트북) | RTX 4080 | 🔴 미달 (RTX 계열이라 RT 코어는 있음) |
| **VRAM** | **6.44 GB** | **16 GB** | 🔴 **가장 큰 위험** |
| Driver | 595.91.7 | 6.x 테스트 버전 595.58.03 | 🟢 |
| CPU | i7-10750H, 12스레드 | i7 7세대, 4코어 | 🟢 |
| CPU governor | Powersave | — | 🟠 성능 모드로 바꿀 것 (`powerprofilesctl set performance`) |
| RAM | 33.5 GB | 32 GB | 🟠 최소는 넘음 (권장 64) |
| 저장공간 | 791 GB 여유 | 50 GB | 🟢 |
| OS | Ubuntu 22.04.5 | 22.04 / 24.04 | 🟢 |

### 1-1. "내 사양에 맞는 버전"이 있나? → 공식적으로는 없다

- Isaac Sim 4.5 이후 **모든 버전의 공식 최소 VRAM이 16 GB**다 (5.0·6.x 요구사항 문서 ✅).
- 옛 버전(4.2 이전)도 최소가 RTX 3070 / 8 GB라서 6 GB를 만족하는 버전은 없다.
- **같은 6.44 GB GPU 사례** (Quadro RTX 3000, Windows): 5.x는 시작하자마자 crash, 4.5는 멈춘 뒤 종료.
  NVIDIA 담당자는 원인을 **VRAM 부족**(`rtx.scenedb.plugin`에서 GPU 자원 할당 실패)으로 진단했다.
  버전을 낮추는 해법은 제시하지 않았고, 클라우드(Brev)를 권했다. ✅ (포럼 원문, 출처 §9)
- 반대로 8 GB 노트북에서는 "UI 익히기·간단한 장면·코드 테스트는 되지만 VRAM 한계에 금방 부딪힌다",
  "RTX 렌더러만 최대 ~7 GB"라는 답변이 있다. ✅

**→ 결론: 버전을 바꿔서 해결되는 문제가 아니다.** 그래서 버전은 **내 OS·ROS와의 공식 호환성**으로 고르고,
실제로 뜨는지는 **첫 실행(§4-B)에서 판정**한다. Linux에서는 결과가 다를 수도 있다. ⚠️ (미확인)

### 1-2. 버전 선택: **Isaac Sim 6.1.0**

| 후보 | 장점 | 단점 | 판단 |
|---|---|---|---|
| **6.1.0** (최신) | Ubuntu 22.04 + Humble **공식 지원** ✅, 공식 ROS 워크스페이스(`humble_ws`)가 6.1 기준 ✅, 이미 Compatibility Checker 6.1 실행함 | 수업 자료(5.1)와 노드·에셋 경로가 일부 다름 | **선택** |
| 5.1.0 (수업 버전) | 슬라이드 화면과 똑같음 | VRAM 조건은 동일하게 미달, 문서·워크스페이스가 구버전 | 6.1이 안 될 때 굳이 갈 이유 없음 |
| 4.x 이하 | Python 3.10 (Humble과 같음) | 공식 지원이 끝난 구버전. 6 GB에서 된다는 근거 없음 | 제외 |
| 클라우드 / 연구실 PC | VRAM 문제 없음 | 네트워크 설정 필요, 비용·접근성 | **6.1이 crash하면 이쪽** |

---

## 2. 핵심 구조 — 터미널을 두 종류로 나눈다

### 2-1. 왜 나누나 (Python 버전 충돌)

- Isaac Sim 6.1은 **Python 3.12**, 시스템 ROS 2 Humble은 **Python 3.10**이다.
- 공식 문서: Ubuntu 22.04에서는 **시스템 ROS를 source한 터미널에서 Isaac Sim을 실행할 수 없다.**
  대신 ROS가 하나도 source되지 않았으면(`ROS_DISTRO`가 비어 있으면) Isaac Sim이
  **내장 Humble 라이브러리**를 알아서 불러온다. ✅
- 내 설정은 원래 **자동 소싱을 안 하므로** 새 터미널이 곧 "깨끗한 터미널"이다. 바꿀 게 없다.

```
┌─ 터미널 A: Isaac Sim ──────────────┐        ┌─ 터미널 B, C…: ROS ───────────────┐
│ humble 치지 않는다 ❌                │        │ humble  (ROS 켜기 + DOMAIN_ID=13) │
│ ROS_DOMAIN_ID=13 만 설정            │ ◀────▶ │ isaac_ws (워크스페이스 source)     │
│ ~/isaacsim/isaac-sim.sh            │ 토픽   │ ros2 launch …                     │
│ (내장 Humble · Python 3.12)         │        │ (시스템 Humble · Python 3.10)      │
└────────────────────────────────────┘        └───────────────────────────────────┘
           두 쪽의 ROS_DOMAIN_ID가 같아야 서로 보인다
```

### 2-2. 수업 자료의 fastdds.xml을 쓰지 않는 이유

- 수업 자료는 Isaac Sim(Windows)과 ROS(VM)가 **사실상 다른 컴퓨터**라서, UDP 네트워크로만
  통신하도록 강제했다(`useBuiltinTransports=false`, `ROS_DISABLE_SHARED_MEMORY=1`).
- 공식 문서: 같은 컴퓨터 안이면 **Fast DDS 기본 설정**을 쓰라고 한다. 그래야 공유 메모리 전송을 써서
  시뮬레이션 성능이 가장 좋다. ✅
- 그래서 **둘 다 안 쓴다.** 다른 컴퓨터(연구실 PC·클라우드)와 통신하게 되면 그때 공식 `fastdds.xml`을 쓴다.

---

## 3. 진행 체크리스트

| 단계 | 내용 | 슬라이드 | 상태 |
|---|---|---|---|
| A | Isaac Sim 6.1 압축 해제 + `post_install.sh` | 30~31 | ⬜ (설치 위치 미확인) |
| B | **첫 실행 판정** (빈 장면이 뜨는가) | — | ⬜ **관문** |
| C | ROS 패키지 설치 (MoveIt·UR·topic_based_ros2_control) | 26 | ⬜ |
| D | 공식 ROS 워크스페이스 빌드 (`isaac_moveit`) | 26 | ⬜ |
| E | alias 2개 추가 (`isaac_ws`, `isaacsim`) | 26·28 대체 | ⬜ |
| F | ROS 2 Bridge 확장 켜기 + `/clock` 확인 | 32~33 | ⬜ |
| G | **Franka MoveIt 예제** Plan & Execute | 38~39 | ⬜ **1차 목표** |
| H | UR3 시뮬레이션 (Action Graph + mock hardware) | 34·40~45 | ⬜ |
| I | 실제 UR3 (고정 IP · 캘리브레이션 · 드라이버) | 47~57 | ⬜ (수업 시간에) |

**B에서 crash하면 C 이후를 진행하지 않는다.** 이 노트북에서 Isaac Sim을 못 띄우면 워크스페이스를 빌드해도 쓸 데가 없다.
단, C·D는 Isaac Sim과 무관하게 실제 UR3(I)에도 필요하므로 B 결과와 상관없이 해도 손해는 없다.

---

## 4. 단계별 절차

### A. Isaac Sim 6.1 설치 (공식 Workstation Installation 기준)

```bash
mkdir ~/isaacsim
cd ~/Downloads
unzip "isaac-sim-standalone-6.1.0-linux-x86_64.zip" -d ~/isaacsim
cd ~/isaacsim
./post_install.sh
```

- `mkdir ~/isaacsim`: 설치 폴더를 만든다. 공식 문서와 ROS 문서가 모두 이 경로(`$HOME/isaacsim`)를 가정한다.
- `unzip … -d ~/isaacsim`: 압축을 푼다. `-d`는 "여기에 풀어라(destination)".
- `./post_install.sh`: 예제 확장용 링크를 만드는 설치 후 스크립트. `./`는 "지금 폴더에 있는 이 파일을 실행".
- 윈도우와 비교: 슬라이드의 `isaac-sim.selector.bat`(App Selector)는 5.1에서 이미 **Deprecated**라고 표시되어 있다
  (슬라이드 32 창 제목). 리눅스에선 `isaac-sim.sh`를 직접 실행한다.

> ⚠️ 이미 다른 위치에 풀었다면 그 경로를 쓰고, 아래 alias의 `~/isaacsim`을 그 경로로 바꾼다.
> 확인: `ls ~/isaacsim/isaac-sim.sh 2>/dev/null || find ~ -maxdepth 3 -name isaac-sim.sh 2>/dev/null`

### B. 첫 실행 판정 — 관문

**새 터미널**(humble 치지 않은 상태)에서:

```bash
powerprofilesctl set performance        # CPU 절전 모드 해제 (충전기 연결 상태에서)
nvidia-smi                              # 다른 프로그램이 VRAM을 쓰고 있나 확인 (크롬 등은 닫기)
~/isaacsim/isaac-sim.sh --/app/content/emptyStageOnStart=true
```

- `--/app/content/emptyStageOnStart=true`: 시작할 때 기본 장면 대신 **빈 장면**으로 열어 VRAM을 아낀다
  (NVIDIA 포럼 답변의 팁 ✅).
- **첫 실행은 셰이더 캐시를 만드느라 5~10분** 걸린다고 공식 문서에 나와 있다. 멈춘 게 아니다.
- 실행 중 IOMMU 경고 창이 뜰 수 있다. GPU가 1개뿐이라 P2P를 안 쓰므로 OK를 눌러 넘긴다.

**판정**

| 결과 | 의미 | 다음 |
|---|---|---|
| 빈 장면 창이 뜨고 5분 이상 유지 | 통과 | C로 |
| 창이 뜨기 전·직후에 꺼짐, 로그에 `rtx.scenedb` / `out of memory` / `ERROR_DEVICE_LOST` | VRAM 부족 (포럼 사례와 같은 증상) | 연구실 PC·클라우드로 전환 검토 |
| 그 외 에러 | 원인 불명 | 로그 전문 확보 후 분석 |

로그 위치: ⚠️ 미확인 — 터미널 출력을 그대로 저장해두면 된다:
`~/isaacsim/isaac-sim.sh --/app/content/emptyStageOnStart=true 2>&1 | tee ~/isaac_first_run.log`
(`2>&1`: 에러 출력도 일반 출력에 합친다 / `tee`: 화면에도 보여주고 파일에도 저장한다)

### C. ROS 패키지 설치 (ROS 터미널)

```bash
humble
sudo apt update
sudo apt install -y ros-humble-ur ros-humble-moveit ros-humble-topic-based-ros2-control python3-rosdep
```

| 패키지 | 역할 | 슬라이드 Jazzy판과 차이 |
|---|---|---|
| `ros-humble-ur` | UR **메타패키지**: `ur_robot_driver` + `ur_moveit_config` + `ur_calibration` + `ur_controllers` + `ur_dashboard_msgs` | 슬라이드는 26쪽(driver·moveit_config)과 52쪽(`ros-jazzy-ur`)에서 나눠 깔았다. 메타패키지 하나로 둘 다 덮는다 ✅ |
| `ros-humble-moveit` | MoveIt 2 본체 | 같음 |
| `ros-humble-topic-based-ros2-control` | ros2_control ↔ Isaac Sim을 **토픽**으로 잇는 하드웨어 인터페이스 (Franka 예제가 씀) | 같음. Humble 0.3.0 릴리스 ✅ |
| `python3-rosdep` | 워크스페이스 의존성 자동 설치 도구 (D에서 씀) | 슬라이드엔 없음 |

- `gedit`, `git`, `colcon`은 이미 있어서 뺐다 (`docs/environment.md` §6).
- `sudo apt upgrade -y`(슬라이드 26)는 **하지 않는다.** 드라이버·ROS 전체가 한꺼번에 바뀔 수 있어서,
  필요한 것만 설치하는 쪽이 원인 추적이 쉽다.

**확인 ✅ 기준: 4줄이 모두 나와야 한다**
```bash
ros2 pkg list | grep -E "^(ur_robot_driver|ur_moveit_config|ur_calibration|topic_based_ros2_control)$"
```

### D. 공식 ROS 워크스페이스 빌드 (ROS 터미널)

슬라이드는 옛 저장소(`NVIDIA-Omniverse/...`)를 받아 `isaac_moveit` 폴더**만** 복사한다.
현재 공식 저장소는 `isaac-sim/IsaacSim-ros_workspaces`이고, Franka 예제는 **서브모듈**
`moveit_resources`(Isaac Sim용으로 고친 Panda 설정)에 기대므로 폴더만 복사하면 빠진다. ✅
(2026-10-01 저장소 `main` 확인: CHANGELOG에 "Bump versions to 6.1.0 [Humble, Jazzy]")

```bash
humble
cd ~
git clone https://github.com/isaac-sim/IsaacSim-ros_workspaces.git
cd ~/IsaacSim-ros_workspaces
git submodule update --init humble_ws/src/moveit/moveit_resources
```

- `git clone`: 저장소 전체를 받는다. `humble_ws`와 `jazzy_ws`가 같이 들어 있다.
- `git submodule update --init <경로>`: 그 서브모듈 **하나만** 받는다. 전체(`--recursive`)를 받으면
  쓰지 않을 Jazzy용 `ros2_control` 소스까지 받는다.

```bash
cd ~/IsaacSim-ros_workspaces/humble_ws
ls /etc/ros/rosdep/sources.list.d/ 2>/dev/null || sudo rosdep init   # rosdep을 처음 쓰면 1회만
rosdep update
rosdep install -i --from-path src/moveit --rosdistro humble -y
colcon build --symlink-install --packages-up-to isaac_moveit
```

- `rosdep install --from-path src/moveit`: `src/moveit` 아래 패키지들이 필요로 하는 것만 apt로 설치한다.
  범위를 `src` 전체로 잡으면 내비게이션(Nav2) 등 안 쓸 것까지 깔린다.
  `-i`: 이미 워크스페이스 안에 있는 패키지는 건너뜀 / `-y`: 설치 질문에 자동 yes.
- `--packages-up-to isaac_moveit`: `isaac_moveit`과 그것이 의존하는 워크스페이스 패키지만 빌드한다.
- `--symlink-install`: 내 `ros2_ws`와 같은 습관. 파이썬·launch 파일을 고치면 재빌드 없이 반영된다.

**확인 ✅ 기준**
```bash
source install/local_setup.bash
ros2 pkg list | grep -E "^(isaac_moveit|moveit_resources_panda_moveit_config)$"   # 2줄
ls src/moveit/isaac_moveit/launch                                                  # isaac_moveit.launch.xml
```

### E. alias 추가 (`~/.bashrc` 맨 아래)

슬라이드 26의 `echo "source …" >> ~/.bashrc`는 **쓰지 않는다**(자동 소싱이 되면 Isaac Sim 터미널이 오염된다, §2).
내 기존 방식(`humble`, `sws`)과 같은 결로 alias 두 개를 추가한다.

```bash
alias isaac_ws='source ~/IsaacSim-ros_workspaces/humble_ws/install/local_setup.bash; echo "isaac_ws is sourced!"'
alias isaacsim='if [ -n "$ROS_DISTRO" ]; then echo "[isaacsim] ROS가 켜진 터미널이다. 새 터미널에서 실행할 것"; else (export ROS_DOMAIN_ID=13; ~/isaacsim/isaac-sim.sh); fi'
```

- `isaac_ws`: `sws`와 같은 역할. **`humble`을 먼저 친 뒤에** 쓴다(공식 문서도 `/opt/ros/humble/setup.bash` → `local_setup.bash` 순서).
- `isaacsim`: **실수를 막는 구조**다. `ROS_DISTRO`가 있으면(= `humble`을 쳤으면) 실행을 거부한다.
  `( … )`는 **서브셸**: 안에서 `export`한 `ROS_DOMAIN_ID`가 현재 터미널에 남지 않는다.
- 도메인 번호 13은 `ros_domain` alias와 같은 값이다. 수업에서 조 번호로 바꾸면 **두 alias 모두** 고친다.

**확인 ✅ 기준**
```bash
bash -n ~/.bashrc && echo "문법 OK"   # 조용히 끝나야 정상
rebash
alias isaac_ws isaacsim                # 두 개가 출력되면 등록됨
```

**새 터미널 표준 절차 (이 실습용)**

| 터미널 | 순서 |
|---|---|
| A (Isaac Sim) | `isaacsim` **만** |
| B, C… (ROS) | `humble` → `isaac_ws` → `ros2 launch …` |

### F. ROS 2 Bridge 확인

공식 문서 기준, 리눅스에서는 위처럼 실행하면 ROS 2 Bridge가 켜진 채로 시작한다 ✅.
슬라이드 33처럼 `Window → Extensions`에서 `ros2`를 검색해 **ROS 2 BRIDGE**와 **ROS 2 SIMULATION CONTROL**이
ENABLED인지만 눈으로 확인한다. (AUTOLOAD 체크도 같이)

**통신 확인 (공식 문서의 Clock 예제)**
1. Isaac Sim: `Tools → Robotics → ROS 2 OmniGraphs → Clock → OK` → **Play(▶)**
2. ROS 터미널: `humble` → `ros2 topic list` → **`/clock`이 보이면 통과**
3. 안 보이면: 두 쪽 `ROS_DOMAIN_ID`가 같은지부터 본다 (`echo $ROS_DOMAIN_ID`).

### G. Franka MoveIt 예제 — 1차 목표 (슬라이드 38~39)

1. Isaac Sim: `Window → Examples → Robotics Examples` → `ROS2 → MoveIt → Franka MoveIt` → Load → **Play**
2. ROS 터미널:
   ```bash
   humble
   isaac_ws
   ros2 launch isaac_moveit isaac_moveit.launch.xml
   ```
   ⚠️ 슬라이드는 `isaac_moveit.launch.py`인데, 현재 공식 저장소(humble·jazzy 둘 다)에는
   **`isaac_moveit.launch.xml`만 있다** ✅ (2026-10-01 저장소 직접 확인).
3. RViz: Planning Group `hand` → Goal State `close` → **Plan & Execute**
   (공식 문서: 일부 컴퓨터에서 `hand`의 `close`가 지연·실패할 수 있음 → 그럴 땐 `panda_arm` + `<random_valid>`로 확인)

**완료 기준**: RViz에서 Execute하면 **Isaac Sim 안의 Franka가 같이 움직인다.**

### H. UR3 시뮬레이션 (슬라이드 34·40~45)

**흐름**
```
[RViz/MoveIt] → [ur_robot_driver, mock hardware] → /joint_states ─┐
                                                                   ▼
                        Isaac Sim Action Graph: ROS2 Subscribe Joint State → Articulation Controller → UR3가 따라 움직임
```
가짜 하드웨어(mock)가 "명령받은 대로 움직였다"고 관절 상태를 내보내면, Isaac Sim이 그걸 받아 화면 속 UR3를 따라 움직이게 하는 구조로 보인다.
⚠️ **추정**: 슬라이드 43에 Subscribe 노드의 `topicName`이 안 나와 있다. `/joint_states`로 맞췄을 가능성이 높지만 미확인.
실행 후 `ros2 topic list`와 `ros2 topic info /joint_states -v`로 실제 구독자를 확인할 것.

**H-1. UR3 에셋** (슬라이드 34)
- `Content` 탭 → `Isaac/Robots/UniversalRobots/ur3/ur3.usd`
- 6.0부터 같은 로봇의 **멀티피직스판**이 `Isaac/Robots_Multiphysics/UniversalRobots/ur3/ur3.usda`에 따로 생겼다.
  기존 경로(`/Isaac/Robots`)도 남아 있다 ✅. **수업과 맞추려면 `Robots`쪽을 쓴다.**
- `Create → Environments → Flat Grid` (슬라이드 40)

**H-2. Action Graph** (슬라이드 41~43) — `Window → Graph Editors → Action Graph → New Action Graph`

| 노드 | 연결·설정 (슬라이드 기준) |
|---|---|
| On Playback Tick | Tick → 아래 세 노드의 Exec In / Time → Clock·Joint State의 Timestamp |
| ROS2 Publish Clock | 시뮬레이션 시간 `/clock` 발행 (MoveIt의 `use_sim_time:=true`가 이걸 씀) |
| ROS2 Publish Joint State | `targetPrim=/ur3/root_joint`, `topicName=isaac_joint_states` |
| ROS2 Subscribe Joint State | Exec Out → Articulation Controller / 각 Command·Joint Names를 연결 |
| Articulation Controller | `targetPrim=/ur3/root_joint` |

⚠️ **6.x에서 바뀐 점** (공식 마이그레이션 가이드 ✅): 6.0부터 **ROS2 Publish Joint State의 `targetPrim` 입력이 Deprecated**다.
아직 동작은 하지만, 새 방식은 **Isaac Read Joint State** 노드가 관절 값을 읽고 그 출력(jointNames·jointPositions 등)을
Publish Joint State에 연결한다. 우선 슬라이드대로 해보고, 경고가 나오거나 값이 안 나오면 새 방식으로 바꾼다.
Subscribe Joint State·Publish Clock·Articulation Controller·On Playback Tick의 변경은 가이드에 언급이 없다.

**H-3. 실행** (ROS 터미널 2개, Isaac Sim은 Play 상태)
```bash
# 터미널 B
humble
ros2 launch ur_robot_driver ur_control.launch.py ur_type:=ur3 use_mock_hardware:=true launch_rviz:=false robot_ip:=127.0.0.1

# 터미널 C
humble
ros2 launch ur_moveit_config ur_moveit.launch.py ur_type:=ur3 use_sim_time:=true
```

⚠️ **Humble판 인자 이름 미확인**: 슬라이드는 Jazzy판(드라이버 3.x), 나는 Humble판(2.15.0)이다.
실행 전에 인자 목록을 먼저 본다.
```bash
ros2 launch ur_robot_driver ur_control.launch.py --show-args | grep -A2 -E "mock|fake"
ros2 launch ur_moveit_config ur_moveit.launch.py --show-args | grep -A2 -E "sim_time|rviz"
```
(`--show-args`: 실제로 띄우지 않고 받을 수 있는 인자만 출력. 예전 버전은 `use_fake_hardware`라는 이름을 썼다.)

**완료 기준**: RViz에서 관절을 움직이고 Plan & Execute → **Isaac Sim의 UR3가 같이 움직인다** (슬라이드 45).

### I. 실제 UR3 (슬라이드 47~57) — 수업 시간에

| 슬라이드 | 수업 자료 | 내 환경 |
|---|---|---|
| 46 | VMware 네트워크 어댑터·USB | **건너뜀** |
| 48~49 | VM 안에서 IP 변경 | 우분투 `설정 → 네트워크 → 유선 → IPv4 → 수동`, 주소 **192.168.68.54**, 넷마스크 255.255.255.0 (UR3 기준) |
| 50 | 로봇 쪽 IP | UR3 = **192.168.68.55** (교내 기기 값) |
| 52 | `sudo apt install ros-jazzy-ur` | C단계 `ros-humble-ur`로 이미 설치됨 |

```bash
# 캘리브레이션 (1회)
humble
ros2 launch ur_calibration calibration_correction.launch.py robot_ip:=192.168.68.55 target_filename:="${HOME}/my_robot_calibration.yaml"

# 터미널 1: 드라이버 → 티치펜던트에서 프로그램 재생(▶)
ros2 launch ur_robot_driver ur_control.launch.py ur_type:=ur3 robot_ip:=192.168.68.55 kinematics_params_file:="${HOME}/my_robot_calibration.yaml" launch_rviz:=false

# 터미널 2: MoveIt → RViz에서 Plan → Execute
ros2 launch ur_moveit_config ur_moveit.launch.py ur_type:=ur3 launch_rviz:=true
```

- 파이썬 제어(슬라이드 56~57): 수업 Google Drive의 `ros_test.py` → `python3 ros_test.py`.
  `position_list`(관절 6개, 라디안)와 `duration_list`(누적 시각, 반드시 증가)를 고쳐 궤적을 바꾼다.
  ⚠️ Jazzy용 코드라 Humble에서 그대로 도는지는 미확인. 시스템 Python 3.10으로 실행하는 ROS 터미널에서 돌린다.
- ⚠️ 실제 로봇은 **비상정지 버튼 위치를 먼저 확인**하고, 첫 Execute는 속도 스케일을 낮춰서 한다.

---

## 5. 슬라이드 → 내 환경 대조표 (전체)

| 슬라이드 | 내용 | 처리 |
|---|---|---|
| 2~24 | VMware·Ubuntu 24.04 설치, Bridged 네트워크 | 건너뜀 (Ubuntu 22.04 단일) |
| 25 | 터미널·sudo 비밀번호 | 동일 |
| 26 | ROS 설치·`.bashrc` 자동 소싱·워크스페이스 | C·D·E로 대체 (자동 소싱 안 함, 공식 저장소 전체 clone) |
| 27~28 | fastdds.xml·환경변수 | 쓰지 않음 (§2-2). `ROS_DOMAIN_ID`만 13으로 |
| 29 | `echo $ROS_DISTRO` 확인 | `humble` 친 뒤에만 `humble`이 나오는 게 정상 |
| 30~32 | Isaac Sim 5.1 Windows·App Selector | A로 대체 (6.1 Linux, `isaac-sim.sh`) |
| 33 | ROS 2 Bridge·Simulation Control 확장 | F (확인만) |
| 34 | UR3 에셋 다운로드 | H-1 (6.x 경로 차이 주의) |
| 35~37 | PowerShell로 fastdds·환경변수 | 쓰지 않음 |
| 38~39 | Franka MoveIt 예제 | G (`.launch.xml`) |
| 40~45 | UR3 시뮬레이션 | H (Humble 인자·6.x 노드 확인) |
| 46~57 | 실제 UR3 | I |

슬라이드 표지에는 `2025-2학기`, 본문 머리글에는 `2026-2`라고 적혀 있다. 작년 자료를 갱신한 것으로 보인다.

---

## 6. 수업 자료에서 정정·보완한 것

- 슬라이드 26 명령의 `₩` 기호는 한글 폰트에서 **역슬래시 `\`**(줄 이어쓰기)가 그렇게 보이는 것이다.
- 슬라이드 26의 `cp -r … 2>/dev/null || true`는 **복사가 실패해도 에러를 숨기고 성공한 척**한다.
  그래서 폴더 이름이 바뀌어 복사가 안 돼도 아무 경고 없이 넘어간다. D에서 저장소 전체를 받는 이유 중 하나다.
- 슬라이드 39의 `isaac_moveit.launch.py` → 현재는 `isaac_moveit.launch.xml`.
- `ROS_DISABLE_SHARED_MEMORY=1`은 한 컴퓨터 안에서 쓰면 **성능만 손해**다 (§2-2).

---

## 7. 미확인 목록 (해보면서 채운다)

- [ ] 이 노트북에서 Isaac Sim 6.1이 뜨는가 (B) — **가장 큰 위험**
- [ ] Isaac Sim 설치 경로 (`~/isaacsim`인가)
- [ ] Humble판 `ur_control.launch.py`가 `use_mock_hardware`를 받는가
- [ ] 슬라이드 43의 Subscribe Joint State `topicName` 값
- [ ] 6.1에서 Publish Joint State `targetPrim` 방식이 그대로 동작하는가
- [ ] `ros_test.py`가 Humble에서 그대로 도는가
- [ ] 수업에서 쓸 `ROS_DOMAIN_ID` (조 번호로 바꿔야 하나)

---

## 8. 다음에 할 일

1. A·B 실행 → 결과를 `study-log/`에 기록 (첫 실행 로그 포함)
2. 통과하면 C~G 진행. 설치·alias가 확정되면 **`docs/environment.md` §3(alias)·§6(패키지)에 Isaac Sim 항목 추가** (CLAUDE.md §3-2 트리거)
3. crash하면 대안(연구실 PC·클라우드)을 조사해 §1-2 표에 결과를 적는다
4. 이 문서의 ⚠️ 항목을 확인할 때마다 ✅로 바꾸고 근거를 적는다

---

## 9. 출처

- 수업 자료: `2026 바이오시스템로봇공학실습 — UR 구동 방법`, HRA Laboratory, Department of Convergence Biosystems Engineering, Chonnam National University (PDF 56쪽)
- [Isaac Sim Requirements (6.x)](https://docs.isaacsim.omniverse.nvidia.com/latest/installation/requirements.html)
- [Isaac Sim Workstation Installation](https://docs.isaacsim.omniverse.nvidia.com/latest/installation/install_workstation.html)
- [Isaac Sim ROS 2 Installation](https://docs.isaacsim.omniverse.nvidia.com/latest/installation/install_ros.html)
- [Isaac Sim MoveIt 2 Tutorial](https://docs.isaacsim.omniverse.nvidia.com/latest/ros2_tutorials/robot_control/tutorial_ros2_moveit.html)
- [Migration: ROS 2 OmniGraph Nodes (6.0)](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_6_0/ros2_omnigraph_migration.html)
- [Migration: Robot Asset Path (6.0)](https://docs.isaacsim.omniverse.nvidia.com/latest/migration_guides/isaac_sim_6_0/robot_asset_path_migration.html)
- [isaac-sim/IsaacSim-ros_workspaces](https://github.com/isaac-sim/IsaacSim-ros_workspaces)
- [ROS Index: topic_based_ros2_control](https://index.ros.org/p/topic_based_ros2_control/) · [ROS Index: ur](https://index.ros.org/p/ur/)
- [NVIDIA 포럼: Isaac Sim 5.x and 4.5 won't run on Quadro RTX 3000 (6 GB VRAM)](https://forums.developer.nvidia.com/t/isaac-sim-5-x-and-4-5-wont-run-on-quadro-rtx-3000-6-gb-vram-recommended-path-for-internship/372141)
- [NVIDIA 포럼: Is 8GB VRAM Enough for Basic Isaac Sim 5.1 Usage?](https://forums.developer.nvidia.com/t/is-8gb-vram-enough-for-basic-isaac-sim-5-1-usage/373600)
