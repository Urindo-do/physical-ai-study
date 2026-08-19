# physical-ai-study

Physical AI(로봇 + AI) 학습 기록 저장소.
ROS 2 · 3D 비전 센서(ZED / RealSense) · 로봇 매니퓰레이터 제어(MoveIt) · 시뮬레이션(Gazebo)으로 이어지는 과정을 따라가며,
공부한 내용 · 실습한 명령어 · 겪은 문제와 해결 과정을 계속 쌓아간다.

---

## 저장소 구조

```
physical-ai-study/
├── README.md                  ← 지금 이 파일 (전체 개요 + 진행 상황)
└── study-log/                 ← 날짜별 학습 기록
    ├── 2026-08-18-linux-basics-practice.md   실습 시나리오 (직접 따라 치는 용)
    └── 2026-08-18-day1-log.md                1일차 학습 로그 (배운 것 정리)
```

새 학습 기록은 `study-log/YYYY-MM-DD-주제.md` 형식으로 계속 추가한다.

---

## 학습 로드맵

| 단계 | 내용 | 상태 |
|---|---|---|
| 1 | Linux(Ubuntu) 환경 구축 + 기본 터미널 명령어 | 진행 중 |
| 2 | ROS 2 설치 및 기초 개념 | 예정 |
| 3 | 3D 비전 센서 (ZED 2i / RealSense) | 예정 |
| 4 | 로봇 매니퓰레이터 제어 (MoveIt) | 예정 |
| 5 | 시뮬레이션 (Gazebo) | 예정 |

---

## 핵심 용어 정리

| 용어 | 설명 |
|---|---|
| **Linux** | 운영체제의 핵심 엔진(커널) 이름. 그 위에 여러 배포판이 존재 |
| **Ubuntu** | 리눅스 계열 배포판 중 하나. 로봇공학 분야의 사실상 표준 환경 |
| **ROS 2** | Robot Operating System 2. 로봇 소프트웨어를 모듈 단위로 만들고 서로 통신시키는 프레임워크 |
| **MoveIt 2** | 로봇 팔(매니퓰레이터)의 동작 계획(motion planning)을 담당. 관절을 몇 도씩 돌려야 목표 위치에 충돌 없이 도달하는지 계산 |
| **Gazebo** | 로봇 시뮬레이터. 물리 법칙(중력·충돌·마찰)이 적용된 가상 환경에서 실제 하드웨어 없이 테스트 |
| **snap** | 우분투의 패키지 배포 방식. 앱과 의존성을 하나의 이미지 파일로 묶어 배포하며, loop 장치로 마운트됨 |

---

## 진행 상황 요약 (2026-08-18 기준)

- 학습 환경: 랩 지급 노트북 (Lenovo Legion, hostname `tuolong-20`)
- 설치된 OS: **Ubuntu 20.04.6 LTS (Focal Fossa)**
- 과제 요구 버전은 22.04였으나, 지도 교원 판단으로 **20.04 유지 결정** → 재설치 보류
- ⚠️ 알아둘 점: Ubuntu 20.04를 지원하는 ROS 2 배포판(Foxy)은 2023년 5월에 지원 종료(EOL).
  향후 ROS 2 · MoveIt · Gazebo 실습 단계에서 22.04(Humble)로의 이전이 필요해질 가능성 있음.

---

## 참고 자료

- [[ROS2] Windows에 Ubuntu 듀얼부팅 설치 | 강티처 EP1](https://www.youtube.com/watch?v=sWmrAb1MJvI)
- [MoveIt 2 Documentation (Humble)](https://moveit.picknik.ai/humble/index.html)
- [ROS 2 Foxy / ROS Melodic EOL 안내 (Ubuntu 공식 블로그)](https://ubuntu.com/blog/ros-foxy-ros-melodic-eol)
