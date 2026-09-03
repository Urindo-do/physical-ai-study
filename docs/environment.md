# 현재 환경 상태 — 단일 진실 공급원

> **이 문서가 현재값이다.** 학습 로그와 과제 기록에 적힌 값은 **그때 그랬다는 스냅샷**이고,
> 시간이 지나 달라졌을 수 있다. 값이 서로 다르면 **항상 이 문서가 이긴다.**
> (`CLAUDE.md` §3-2 참고)

- 마지막 갱신: **2026-09-03**
- 출처: 저장소에 기록된 가장 최근 값들을 모은 것. 실제 노트북에서 재확인한 것은 아니므로,
  리눅스를 켜면 아래 "확인 명령"으로 한 번 대조해보고 어긋나면 이 문서를 고친다.

---

## 1. 기기

| 항목 | 값 | 확인 명령 |
|---|---|---|
| 리눅스 노트북 | Lenovo Legion 5 15IMH05H (랩 지급) | `hostnamectl` |
| hostname | `urindodo-Lenovo-Legion-5-15IMH05H` | `hostname` |
| OS | **Ubuntu 22.04.5 LTS (Jammy Jellyfish)** | `lsb_release -a` |
| 윈도우 PC | 문서·git 작업 전용. 저장소 clone 위치 `C:\Users\rlaeo\physical-ai-study` | — |
| 보유 하드웨어 | Raspberry Pi 5 (사용 경험 있음), Intel RealSense D435i | — |

> 역할 분담은 `CLAUDE.md` §2. **리눅스=실습 / 윈도우=문서·git**, 이 저장소를 리눅스에 clone하지 않는다.

## 2. ROS 2

| 항목 | 값 | 확인 명령 |
|---|---|---|
| 배포판 | **Humble Hawksbill** | `printenv ROS_DISTRO` |
| 설치 경로 | `/opt/ros/humble` | `ls /opt/ros` |
| 설치 방식 | `sudo apt install ros-humble-desktop` | — |
| ROS_DOMAIN_ID | **13** | `echo $ROS_DOMAIN_ID` |
| 활성화 방식 | **자동 소싱 안 함.** 터미널마다 `humble`을 직접 쳐야 함 | `alias` |

> 자동 소싱을 안 쓰는 건 의도된 설계다. `conda activate` 스타일의 명시적 활성화로,
> 나중에 다른 ROS 버전을 깔아도 충돌하지 않게 하려는 것. 대가는 매 터미널 `humble` 한 번.

## 3. alias (`~/.bashrc` 맨 아래)

```bash
export PATH="$HOME/.local/bin:$PATH"
alias rebash='source ~/.bashrc'
alias ros_domain='export ROS_DOMAIN_ID=13; echo "ROS_DOMAIN_ID=$ROS_DOMAIN_ID"'
alias humble='source /opt/ros/humble/setup.bash; ros_domain; echo "ROS2 humble is activated"'
alias sws='source ~/ros2_ws/install/setup.bash; echo "ros2_ws is sourced!"'
```

확인: `alias` (인자 없이 실행하면 등록된 전체 목록)

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

**등록된 노드** (`setup.py`의 `console_scripts`)

| 노드 | 역할 |
|---|---|
| `hello_node` | 패키지 골격 검증용 |
| `red_object_tf` | 빨간 물체 3D 위치 → TF + Marker 발행 |
| `aruco_board_tf` | ArUco 마커 6DoF Pose → TF 발행 |

**launch**: `ros2 launch vision_pkg vision.launch.py` — 카메라 + 위 두 노드를 한 번에 실행.
`marker_length`는 하드코딩이 아니라 ROS 2 Parameter로 뺐다.

확인: `ros2 pkg list | grep vision_pkg`, `ros2 pkg executables vision_pkg`

## 5. 카메라 (Intel RealSense D435i)

| 항목 | 값 | 확인 방법 |
|---|---|---|
| 모델 | **D435i** | launch 로그 |
| Serial No. | `033422070476` | launch 로그 |
| FW version | 5.17.3.10 | launch 로그 |
| RealSense ROS | **v4.58.3** | launch 로그 |
| LibRealSense | **v2.58.3** | launch 로그 |
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

| 패키지 | 설치 방식 | 용도 |
|---|---|---|
| `ros-humble-desktop` | apt | ROS 2 본체 |
| `ros-humble-realsense2-*` | apt | RealSense ROS 2 드라이버 |
| `python3-colcon-common-extensions` | apt | `colcon build` |
| `terminator` | apt | 터미널 (창 분할용) |
| `language-pack-ko`, `ibus-hangul` | apt | 한글 입력 |
| VS Code | **snap** (`sudo snap install code --classic`) | 에디터 |
| Chrome | `.deb` 직접 설치 | 브라우저 |
| `python3-pip`, `jupyter` | apt / pip3 | Python 도구 |

| 런타임 | 버전 | 확인 명령 |
|---|---|---|
| Python | **3.10.12** | `python3 --version` |
| **OpenCV (`cv2`)** | **4.5.4** | `python3 -c "import cv2; print(cv2.__version__)"` |

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
| 2026-09-03 | 최초 작성. 저장소에 흩어져 있던 환경 정보를 모음. `alias humble` 정의가 문서마다 4가지 버전으로 존재해 어느 게 현재값인지 알 수 없던 문제를 해결하려고 만듦 |
