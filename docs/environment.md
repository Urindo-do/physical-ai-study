# 현재 환경 상태 — 단일 진실 공급원

> **이 문서가 현재값이다.** 학습 로그와 과제 기록에 적힌 값은 **그때 그랬다는 스냅샷**이고,
> 시간이 지나 달라졌을 수 있다. 값이 서로 다르면 **항상 이 문서가 이긴다.**
> (`CLAUDE.md` §3-2 참고)

- 마지막 갱신: **2026-09-22**
- 출처: **2026-09-22에 리눅스 노트북에서 아래 "확인 명령"을 실제로 실행해 대조한 값.**
  예외는 §5 카메라 — 그때 D435i가 연결돼 있지 않아 apt로 확인되는 버전 두 개만 재확인했다.

---

## 1. 기기

| 항목 | 값 | 확인 명령 |
|---|---|---|
| 리눅스 노트북 | Lenovo Legion 5 15IMH05H (랩 지급) | `hostnamectl` |
| hostname | `urindodo-Lenovo-Legion-5-15IMH05H` | `hostname` |
| OS | **Ubuntu 22.04.5 LTS (Jammy Jellyfish)** | `lsb_release -a` |
| 윈도우 PC | **이 저장소 작업에 쓰지 않음** (2026-09-22부터) | — |
| 보유 하드웨어 | Raspberry Pi 5 (사용 경험 있음), Intel RealSense D435i | — |
| 저장소 위치 | `~/physical-ai-study` (리눅스 노트북) | `git -C ~/physical-ai-study remote -v` |
| 작업 방식 | 리눅스 노트북 **VS Code**에서 실습·문서·git 전부 | — |

> 역할 분담은 `CLAUDE.md` §2. **리눅스 노트북 한 대에서 실습·문서·git을 전부 한다.**
> (2026-09-21까지는 "리눅스=실습 / 윈도우=문서·git"이었고 이 저장소를 리눅스에 clone하는 것을
>  금지했다. 바꾼 이유는 `CLAUDE.md` §2의 "이전 규칙" 블록에 남겨두었다.)

## 2. ROS 2

| 항목 | 값 | 확인 명령 |
|---|---|---|
| 배포판 | **Humble Hawksbill** | `printenv ROS_DISTRO` |
| 설치 경로 | `/opt/ros/humble` | `ls /opt/ros` |
| 설치 방식 | `sudo apt install ros-humble-desktop` | — |
| 패키지 버전 | `ros-humble-desktop` **0.10.0-1jammy.20260804.223343** | `dpkg -l \| grep ros-humble-desktop` |
| ROS_DOMAIN_ID | **13** (`humble`을 친 뒤에만 설정됨) | `echo $ROS_DOMAIN_ID` |
| 활성화 방식 | **자동 소싱 안 함.** 터미널마다 `humble`을 직접 쳐야 함 | `alias` |

> 2026-09-22 확인: 새 터미널에서 `printenv ROS_DISTRO`가 **아무것도 출력하지 않고 종료 코드 1**,
> `echo $ROS_DOMAIN_ID`도 빈 값이다. 이게 정상이다 — 자동 소싱을 안 하니까 그렇다.
> `humble`을 친 뒤에 다시 보면 `humble` / `13`이 나온다.

> 자동 소싱을 안 쓰는 건 의도된 설계다. `conda activate` 스타일의 명시적 활성화로,
> 나중에 다른 ROS 버전을 깔아도 충돌하지 않게 하려는 것. 대가는 매 터미널 `humble` 한 번.

## 3. alias (`~/.bashrc` 맨 아래)

실제 `~/.bashrc` 맨 아래는 이렇게 되어 있다 (2026-09-22 `tail -30 ~/.bashrc`로 확인한 원문):

```bash
export PATH="$HOME/.local/bin:$PATH"
alias ros_domain="export ROS_DOMAIN_ID=13; echo \"ROS_DOMAIN_ID=\$ROS_DOMAIN_ID\""
alias humble="source /opt/ros/humble/setup.bash; ros_domain; echo \"ROS2 humble is activated\""
alias rebash="source ~/.bashrc"
alias sws='source ~/ros2_ws/install/setup.bash; echo "ros2_ws is sourced!"'
export PATH="$HOME/.local/bin:$PATH"
export PATH="$HOME/.local/bin:$PATH"
```

확인: `alias` (인자 없이 실행하면 등록된 전체 목록)

> ⚠️ **`export PATH="$HOME/.local/bin:$PATH"` 가 세 번 중복되어 있다.** 동작에는 문제가 없지만
> (같은 경로가 PATH 앞에 세 번 붙을 뿐) 파일을 고칠 때마다 한 줄씩 덧붙인 흔적이다.
> 정리할 때 맨 위 한 줄만 남기면 된다. — **정리 대상, 아직 안 고침**
>
> alias 정의 자체는 이전 기록과 **동작이 같다** (따옴표 스타일과 줄 순서만 다름).
> `ros_domain`/`humble`/`rebash`는 큰따옴표+이스케이프, `sws`만 작은따옴표를 쓴다.

| alias | 하는 일 |
|---|---|
| `humble` | ROS 2 환경 켜기 + `ros_domain` 호출까지 한 번에 |
| `ros_domain` | `ROS_DOMAIN_ID=13` 설정 (보통 `humble`이 알아서 부름) |
| `sws` | 내 워크스페이스 소싱 (`~/ros2_ws`) |
| `rebash` | `.bashrc` 다시 읽기. **ROS를 켜주지는 않는다** |

**새 터미널 표준 절차**: `humble` → (내 패키지 쓸 때만) `sws`

> ⚠️ `.bashrc`를 고친 뒤에는 `rebash` 또는 새 터미널이 필요하다.
> 특히 `.bashrc`가 문법 에러로 한 번 깨졌던 터미널은 alias 자체가 등록 안 된 상태라
> `rebash`도 안 먹는다 — **새 터미널을 열어야 한다.**
> 고치기 전 검사: `bash -n ~/.bashrc` (에러 없으면 조용히 끝남)

## 4. 워크스페이스와 내 패키지

| 항목 | 값 |
|---|---|
| 워크스페이스 | `~/ros2_ws` |
| 패키지 | `vision_pkg` (`ament_python`) |
| 빌드 | `cd ~/ros2_ws && colcon build --symlink-install` |
| 소싱 | `sws` (= `source ~/ros2_ws/install/setup.bash`) |

**등록된 노드** (`setup.py`의 `console_scripts`) — **7개**

| 노드 | 역할 |
|---|---|
| `hello_node` | 패키지 골격 검증용 |
| `image_subscriber` | 카메라 토픽 구독 연습 |
| `fake_tf_marker` | 카메라 없이 TF/Marker 발행 연습 (더미 데이터) |
| `red_detector` | 빨간 물체 2D 검출 (마스킹까지) |
| `red_3d_position` | 빨간 물체의 3D 좌표 계산 (Depth 사용) |
| `red_object_tf` | 빨간 물체 3D 위치 → TF + Marker 발행 |
| `aruco_board_tf` | ArUco 마커 6DoF Pose → TF 발행 |

> 2026-09-22 정정: 이전에는 위 3개(`hello_node`·`red_object_tf`·`aruco_board_tf`)만 적혀 있었다.
> `ros2 pkg executables vision_pkg` 실행 결과 실제로는 **7개가 전부 등록되어 있다.**
> 뒤의 4개는 앞 노드로 가는 중간 단계라 최종본만 적었던 것인데, 지금도 실행 가능한 노드들이다.
> 역할 설명은 노드 이름과 학습 로그 순서에서 유추한 것이다 (소스 재확인은 안 함).

**launch**: `ros2 launch vision_pkg vision.launch.py` — 카메라 + `red_object_tf` + `aruco_board_tf`
세 개를 한 번에 실행한다 (2026-09-22 `vision.launch.py` 확인). 나머지 4개 노드는 launch에 없다.
`marker_length`는 하드코딩이 아니라 ROS 2 Parameter로 뺐다.

확인: `ros2 pkg list | grep vision_pkg`, `ros2 pkg executables vision_pkg`

## 5. 카메라 (Intel RealSense D435i)

> ⚠️ **2026-09-22 실측 시 카메라가 연결돼 있지 않았다** (`lsusb`에 D435i 없음. Intel 항목은
> 내장 `AX201 Bluetooth`뿐). 아래 Serial No.·FW·USB·스트림 값은 **마지막 launch 로그 기준이고
> 이번에 재확인하지 못했다.** 드라이버 버전 두 개만 apt로 재확인했다(아래 ✅).
> 다음에 카메라를 꽂으면 launch 로그로 나머지를 대조할 것.

| 항목 | 값 | 확인 방법 |
|---|---|---|
| 모델 | **D435i** | launch 로그 |
| Serial No. | `033422070476` | launch 로그 |
| FW version | 5.17.3.10 | launch 로그 |
| RealSense ROS | **v4.58.3** ✅ | `dpkg -l \| grep ros-humble-realsense2-camera` |
| LibRealSense | **v2.58.3** ✅ | `dpkg -l \| grep ros-humble-librealsense2` |
| USB | Bus 002 / port 2-3 / **USB 3.2** (5000M) | `lsusb`, `lsusb -t` |
| Depth 스트림 | Z16, 848x480, 30fps (기본값) | launch 로그 |
| Color 스트림 | RGB8, 1280x720, 30fps (기본값) | launch 로그 |

**실행**
```bash
ros2 launch realsense2_camera rs_launch.py align_depth.enable:=true pointcloud.enable:=true
```

**토픽** — ⚠️ 네임스페이스가 `/camera/camera/...` 로 **camera가 두 번** 들어간다.
오래된 튜토리얼의 `/camera/color/...`를 그대로 치면 "그런 토픽 없음"이 뜬다.

| 용도 | 토픽 | 타입 |
|---|---|---|
| RGB | `/camera/camera/color/image_raw` | `sensor_msgs/msg/Image` |
| Depth | `/camera/camera/depth/image_rect_raw` | `sensor_msgs/msg/Image` |
| **Aligned Depth** | `/camera/camera/aligned_depth_to_color/image_raw` | `sensor_msgs/msg/Image` |
| CameraInfo | `/camera/camera/color/camera_info` | `sensor_msgs/msg/CameraInfo` |
| PointCloud | `/camera/camera/depth/color/points` | `sensor_msgs/msg/PointCloud2` |

- 실측 주파수: RGB / Aligned Depth 둘 다 **≈ 30.0 Hz** (`ros2 topic hz <토픽>`)
- RViz Fixed Frame: `camera_link`
- 3D 좌표 계산 결과의 기준 프레임: **`camera_color_optical_frame`** (`camera_link`와 축 방향이 다름)
- ArUco 마커 실측 한 변: **4 inch ≈ 0.1016 m** (`marker_length` 파라미터 기본값)

확인: `ros2 topic list -t`, `ros2 node list`, `lsusb | grep -i intel`

## 6. 설치된 주요 패키지

버전은 2026-09-22에 `dpkg -l` / `snap list` / `pip3 list`로 실측한 값이다.

| 패키지 | 설치 방식 | 버전 | 용도 |
|---|---|---|---|
| `ros-humble-desktop` | apt | 0.10.0-1jammy.20260804 | ROS 2 본체 |
| `ros-humble-realsense2-camera` | apt | **4.58.3** | RealSense ROS 2 드라이버 |
| `ros-humble-librealsense2` | apt | **2.58.3** | RealSense SDK 본체 (위 드라이버가 씀) |
| `ros-humble-cv-bridge` | apt | 3.2.1 | ROS Image ↔ OpenCV 변환 |
| `python3-colcon-common-extensions` | apt | 0.3.0-100 | `colcon build` |
| `terminator` | apt | 2.1.1-1 | 터미널 (창 분할용) |
| `language-pack-ko` / `ibus-hangul` | apt | 22.04+20240902 / 1.5.4 | 한글 입력 |
| VS Code | **snap** (`--classic`, rev 259) | **1.135.0** | 에디터 (이제 문서·git도 여기서) |
| Chrome | `.deb` 직접 설치 | 152.0.7977.64-1 | 브라우저 |
| `python3-pip` | apt | 22.0.2 | Python 패키지 설치 |
| `jupyter` / `jupyterlab` / `notebook` | **pip3 (사용자 설치)** | 1.1.1 / 4.6.3 / 7.6.2 | Python 노트북 |

> `jupyter`는 apt가 아니라 pip3로 **사용자 홈에** 깔려 있다 (`~/.local/bin/jupyter`).
> `.bashrc`의 `export PATH="$HOME/.local/bin:$PATH"` 가 있어야 명령이 잡히는 이유가 이것이다.

| 런타임 | 버전 | 확인 명령 |
|---|---|---|
| Python | **3.10.12** | `python3 --version` |
| **OpenCV (`cv2`)** | **4.5.4** | `python3 -c "import cv2; print(cv2.__version__)"` |

`cv2`의 출처는 apt 패키지 `python3-opencv 4.5.4+dfsg-9ubuntu4`다
(`python3 -c "import cv2; print(cv2.__file__)"` → `/usr/lib/python3/dist-packages/...`).
즉 **우분투 22.04가 주는 버전을 그대로 쓰고 있어서** 아래 4.7 문제가 생긴다.

> ⚠️ **OpenCV 4.5.4는 4.7 미만**이라 ArUco에서 **구 API**를 써야 한다
> (`cv2.aruco.detectMarkers()`, `cv2.aruco.estimatePoseSingleMarkers()`).
> 인터넷 예제가 안 돌아가는 원인의 대부분이 이 버전 차이다. 예제를 고르기 전에 버전부터 확인할 것.
> `cv2`(OpenCV)와 `cv_bridge`는 **별개의 소프트웨어**다 — `cv_bridge`는 ROS Image ↔ OpenCV 변환 다리.

## 7. 자주 쓰는 확인 명령 모음

```bash
# 환경이 켜졌나
alias                    # 등록된 alias 전체
echo $ROS_DOMAIN_ID      # 13이어야 함
printenv ROS_DISTRO      # humble

# 지금 뭐가 떠 있나
ros2 node list
ros2 topic list -t
ros2 topic hz <토픽>

# 내 패키지가 보이나
ros2 pkg list | grep vision_pkg
ros2 pkg executables vision_pkg

# 카메라가 OS에 잡히나 (ROS 이전 단계)
lsusb | grep -i intel
lsusb -t                 # USB 3.x인지 (5000M) 확인

# .bashrc 고치기 전후
bash -n ~/.bashrc        # 문법 검사 (조용하면 정상)
```

---

## 갱신 이력

| 날짜 | 내용 |
|---|---|
| 2026-09-22 | §2~§6 전체를 리눅스 노트북에서 **실측 대조**. §4 등록 노드 3개→**7개**로 정정, §3 alias를 `.bashrc` 원문으로 교체(+`export PATH` 3중 중복 발견), §6에 실측 버전·`librealsense2`·`cv-bridge` 추가, §5는 카메라 미연결로 재확인 못 했음을 명시 |
| 2026-09-22 | §1 기기 — 작업 환경을 리눅스 노트북 단일 기기로 전환(실습·문서·git 전부). 저장소 위치 `~/physical-ai-study`와 작업 방식(VS Code) 행 추가 |
| 2026-09-03 | 최초 작성. 저장소에 흩어져 있던 환경 정보를 모음. `alias humble` 정의가 문서마다 4가지 버전으로 존재해 어느 게 현재값인지 알 수 없던 문제를 해결하려고 만듦 |
