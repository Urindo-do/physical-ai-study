# physical-ai-study

Physical AI(로봇 + AI) 학습 기록 저장소.
ROS 2 · 3D 비전 센서(ZED / RealSense) · 로봇 매니퓰레이터 제어(MoveIt) · 시뮬레이션(Gazebo)으로 이어지는 과정을 따라가며,
공부한 내용 · 실습한 명령어 · 겪은 문제와 해결 과정을 계속 쌓아간다.

---

## 저장소 구조

```
physical-ai-study/
├── CLAUDE.md                  ← AI 에이전트용 규칙 문서 (구조·배치·커밋 규칙)
├── README.md                  ← 지금 이 파일 (전체 개요 + 진행 상황)
├── .gitattributes             ← 줄바꿈 LF 고정
├── docs/                      ← [진행축] 진도 관리 체크리스트
│   └── ros2-r2r-checklist.md      R2R ROS2 강좌 137개 전체 체크리스트 (진도 단일 진실 공급원)
├── notes/                     ← [지식축] 주제별 개념 정리 (날짜 무관)
│   └── apqp-robot-mass-production.md
│                                  로봇 양산 사이클(APQP) 5단계 실무 프로세스
└── study-log/                 ← [시간축] 날짜별 학습 기록
    ├── images/                               로그에 쓰이는 스크린샷
    ├── 2026-08-18-linux-basics-practice.md   실습 시나리오 (직접 따라 치는 용)
    ├── 2026-08-18-day1-log.md                1일차: 리눅스 환경 진단 + 기초 명령어
    ├── 2026-08-19-day2-log.md                2일차: Ubuntu 22.04 재설치 + 리눅스 개념 심화
    ├── 2026-08-19-day3-ros2-r2r-plan.md      3일차: ROS2 학습 전략 수립 (R2R 커리큘럼 확정)
    ├── 2026-08-20-day4-ros2-intro-install.md 4일차: R2R 입문 착수 + ROS 2 Humble 설치 ✅
    ├── 2026-08-21-day5-bashrc-alias-domain-id.md
    │                                         5일차: bashrc·alias 환경 설계, DDS·ROS_DOMAIN_ID
    ├── 2026-08-24-day8-ros2-turtlesim-assignment.md
    │                                         8일차: 연구실 과제 — turtlesim·토픽·서비스 CLI 실습
    └── 2026-08-26-day10-physical-ai-rl-il-vla.md
                                              10일차: 피지컬 AI 개론 — 강화학습·모방학습·VLA
```

문서는 성격에 따라 세 축으로 나눠 넣는다.

| 폴더 | 축 | 담는 것 | 찾을 때 |
|---|---|---|---|
| `study-log/` | 시간축 | 날짜별 학습 기록 (`YYYY-MM-DD-주제.md`) | "그때 뭐 했더라" |
| `docs/` | 진행축 | 진도 체크리스트·계획 | "어디까지 했더라" |
| `notes/` | 지식축 | 주제별 개념 정리 (파일명에 날짜 없음) | "그거 뭐였더라" |

배치·커밋 규칙 상세는 [`CLAUDE.md`](./CLAUDE.md) §3 참고.

---

## 학습 로드맵

| 단계 | 내용 | 상태 |
|---|---|---|
| 1 | Linux(Ubuntu 22.04) 환경 구축 + 기본 터미널 명령어 | ✅ 완료 |
| 2 | ROS 2 설치 및 기초 개념 (R2R 강좌 4단계, 137개 체크리스트) | 진행 중 |
| 3 | 3D 비전 센서 (ZED 2i / RealSense) | 예정 |
| 4 | 로봇 매니퓰레이터 제어 (MoveIt) | 예정 |
| 5 | 시뮬레이션 (Gazebo) | 예정 |
| — | (병행) 로봇 개발 실무 프로세스 이해 — APQP 양산 사이클 | ✅ 1차 정리 |

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

## 진행 상황 요약 (2026-08-30 기준)

- ✅ **ROS 2 Humble 설치 완료** (2026-08-20) — talker/listener 통신 및 `rqt_graph`로 검증
- ✅ ROS 2 환경을 `.bashrc` 자동 실행이 아닌 **alias 호출 방식**으로 전환 (환경 충돌 방지)
- ✅ **연구실 과제 4종 수행 완료** (2026-08-24) — turtlesim 실행, 토픽 Pub/Sub, `topic pub`으로
  원형 궤적, `service call /spawn`으로 거북이 소환. 토픽(N:M 방송)과 서비스(1:1 요청-응답)의
  차이를 실물로 확인 → [Day8 로그](./study-log/2026-08-24-day8-ros2-turtlesim-assignment.md)
- ✅ **`ros_domain` alias 등록 완료** (2026-08-26) — Day5부터의 이월 작업 해소.
  새 터미널에서 `humble` 한 번으로 ROS 2 환경 + `ROS_DOMAIN_ID=13` 동시 적용 확인
- ✅ **피지컬 AI 이론 수업 정리** (2026-08-26) — 강화학습 → 모방학습(BC·DAgger·ACT·Diffusion Policy)
  → VLA(RT 계열·OpenVLA·π0·GR00T) 계보. 논문 원문 대조로 필기 6건 정정
  → [Day10 로그](./study-log/2026-08-26-day10-physical-ai-rl-il-vla.md)
- ✅ **로봇 양산 사이클(APQP) 정리** (2026-08-30) — 기획 → 개발설계 → Proto → Pilot → 양산
  5단계와 게이트 심의 구조. ROS 2 학습 내용이 실무 문서(ICD·DFMEA·Fail-Safe 명세)의 어디에
  대응하는지 매핑표 작성. 공개 사양 대조로 필기 정정 3건
  → [APQP 노트](./notes/apqp-robot-mass-production.md) — `notes/` 축을 이때 신설
- R2R 입문과정 12/35 진행 중 (전체 **12/137**) — 8/22~8/26은 R2R 영상 진도 없음
  (주말·연구실 과제·이론 수업). 다음은 묶음 3(3-1 Turtlesim ~ 3-9 Action)
- ⏳ 이월: 과제 1 인증 캡처 재촬영 (거북이 창과 rqt_graph 창이 겹치지 않게 한 화면에)

- 학습 환경: 랩 지급 노트북 (Lenovo Legion 5, hostname `urindodo-Lenovo-Legion-5-15IMH05H`)
- 설치된 OS: **Ubuntu 22.04.5 LTS (Jammy Jellyfish)** — Day2에서 20.04 → 22.04 재설치 완료 (ROS 2 Humble 공식 지원 버전)
- 한/영 입력 전환 정상 작동 확인 (ibus-hangul `initial-input-mode` 설정으로 해결)
- 2단계(ROS 2) 학습 자료로 **R2R(민형기 강사님, 핑크랩 PinkLAB) 무료 강좌**를 채택. 4단계
  (입문·응용·심화·실전, 총 137개 필수 강의, 약 25시간 34분) 완주 목표 — **날짜 일정 없이
  체크박스를 지워나가는 방식**(이유는 체크리스트 상단 참고). 세부 진도는
  [`docs/ros2-r2r-checklist.md`](./docs/ros2-r2r-checklist.md)

---

## 참고 자료

- [[ROS2] Windows에 Ubuntu 듀얼부팅 설치 | 강티처 EP1](https://www.youtube.com/watch?v=sWmrAb1MJvI)
- [MoveIt 2 Documentation (Humble)](https://moveit.picknik.ai/humble/index.html)
- [ROS 2 Foxy / ROS Melodic EOL 안내 (Ubuntu 공식 블로그)](https://ubuntu.com/blog/ros-foxy-ros-melodic-eol)
- [R2R 무작정 따라하는 ROS2 - 입문과정 (핑크랩 PinkLAB, 민형기 강사님)](https://www.youtube.com/playlist?list=PL0xYz_4oqpvhj4JaPSTeGI2k5GQEE36oi)
- [ROS 2 Humble 공식 설치 문서 (docs.ros.org)](https://docs.ros.org/en/humble/Installation/Alternatives/Ubuntu-Install-Binary.html)
