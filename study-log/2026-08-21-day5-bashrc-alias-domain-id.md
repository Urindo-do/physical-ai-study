# Day 5 — bashrc·alias 환경 설계 & ROS_DOMAIN_ID (DDS)

**날짜**: 2026-08-21 (2026-08-19~21에 걸친 환경설정 내용 포함)
**진도**: R2R 입문과정 2-3 ~ 2-5 (3개, 누적 12/137)
**환경**: Ubuntu 22.04 (Jammy), Lenovo Legion 5 15IMH05H, GNOME/Wayland

---

## 1. 개발 환경 마무리

### 설치한 것

| 프로그램 | 설치 방식 | 명령 |
|---|---|---|
| VS Code | snap | `sudo snap install code --classic` |
| Chrome | `.deb` 직접 설치 | `wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb` → `sudo apt install ./google-chrome-stable_current_amd64.deb` |
| terminator | apt | `sudo apt install terminator` |

> ⚠️ **필기 정정**: 메모에 `sudo apt install code --classic`이라고 적었는데, **`--classic`은 apt 옵션이 아니라 snap 전용 옵션**이다.
> apt에 붙이면 오류가 난다. VS Code를 snap으로 설치했다면 올바른 명령은 `sudo snap install code --classic`.
> (`--classic` = snap의 격리(sandbox)를 풀고 시스템 전체에 접근하게 하는 옵션. 에디터·IDE처럼 아무 파일이나 열어야 하는
> 프로그램은 격리되면 못 쓰기 때문에 필요하다.)

### snap vs apt — 어느 걸 쓰나

| | snap | apt |
|---|---|---|
| 방식 | 앱 + 의존성을 한 덩어리로 봉인해 배포 | 시스템 라이브러리를 공유하는 전통적 방식 |
| 장점 | 다른 프로그램과 독립적, 자동 업데이트 | 가볍고 시스템 도구에 적합 |
| 단점 | 용량 큼, 격리 때문에 제약 생길 수 있음 | 의존성 충돌 가능("dependency hell") |

- **terminator를 apt로 설치한 이유**: 터미널 관련 프로그램은 시스템과 밀착돼 동작해야 하므로 격리가 걸린 snap보다 apt가 적합.
- 둘 다 정상적인 설치 경로이며, 상황에 맞게 고르면 된다. (Day 2에서 정리한 snap/loop 개념과 이어짐)

### 우분투 데스크톱 사용법 차이

- 우분투는 **바탕화면에 아이콘이 안 생긴다.** Super 키 → Activities 검색으로 실행하고, 자주 쓰면 Dock에 즐겨찾기로 고정.
- **기본 브라우저 설정**은 "다른 프로그램에서 링크를 열 때 어떤 브라우저를 부를지" 정하는 것. 아이콘을 직접 클릭해 여는 것과는 무관하다.

### Python / Jupyter

- Ubuntu 22.04 기본 Python은 **3.10.12** — ROS 2 Humble의 공식 타겟 버전과 정확히 일치. **손댈 필요 없음.**
- **`jupyter notebook` → "command not found" 문제**
  - 원인: pip로 설치된 실행 파일은 `~/.local/bin`에 들어가는데, 이 폴더가 **PATH에 등록되어 있지 않았음.**
  - 해결:
    ```bash
    echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
    source ~/.bashrc
    ```
- Jupyter Notebook의 구조: **로컬 웹 서버(localhost) + 브라우저(클라이언트)**. 코드 실행은 별도 **커널** 프로세스가 맡고, 이 커널이 실제로 CPU/RAM을 쓴다.

---

## 2. 리눅스 핵심 개념 (오늘 제대로 잡은 것)

### PATH

터미널이 명령어를 찾을 때 **뒤지는 폴더 목록**. 파일이 디스크에 실제로 존재해도, 그 폴더가 PATH에 없으면 `command not found`가 뜬다.
→ 위 Jupyter 문제가 정확히 이 케이스였다.

### 리다이렉션 (`>`, `>>`)

- `echo`는 파이썬의 `print`와 같은 개념 — 화면에 출력.
- `>` : 파일에 **덮어쓰기** (기존 내용 날아감 ⚠️)
- `>>` : 파일 끝에 **이어붙이기**

→ `.bashrc`에 설정을 추가할 때 `>>`를 쓰는 이유. `>`를 쓰면 기존 `.bashrc`가 통째로 날아간다.

### `.bashrc`

- **폴더가 아니라 파일.** 홈(`~`) 바로 밑의 숨김 파일 (이름 앞의 점 `.`은 숨김 표시일 뿐).
- 이름 뜻: **bash**(Bourne Again SHell) + **rc**(run commands, "시작할 때 실행할 명령어들")
  = "bash가 시작할 때 자동 실행할 설정 파일"
- 터미널을 **열 때마다 자동으로 읽힌다.** 여기 등록한 설정은 이후 여는 모든 새 터미널에 적용됨.
- ⚠️ 수정해도 **이미 열려 있는 터미널에는 반영 안 됨** → `source ~/.bashrc`로 즉시 반영하거나 터미널을 새로 연다.
- ⚠️ 하지만 `source`는 **"새로 추가된 내용"만 반영**할 뿐, 이미 살아있는 세션에 남은 예전 환경변수까지 지우지는 못한다.
  → **완전히 깨끗한 상태로 검증하려면 반드시 새 터미널(새 프로세스)을 열어야 한다.**
  (VS Code 등에서 "재시작하세요"라는 조언이 흔한 것과 같은 원리)

### `source` (= `.`)

- 뜻: **"이 파일 안의 내용을 지금 이 터미널 세션에 직접 적용해라."**
- `./script.sh`(그냥 실행)와의 차이

| | `./script.sh` | `source script.sh` |
|---|---|---|
| 프로세스 | **자식 프로세스**를 새로 만들어 거기서 실행 | 새 프로세스 없이 **지금 세션 자체**에서 실행 |
| 환경변수 변경 | 자식이 끝나면 **사라짐** (부모 터미널에 영향 없음) | 지금 세션에 **그대로 남음** |

- `. 파일경로` 형태의 점(`.`)은 `source`의 줄임 표현.
  → **`.bashrc`의 점(숨김 파일 표시)과는 전혀 다른, 별개의 의미다.** 헷갈리기 쉬운 부분.

### 기타

- `sudo`: 관리자 권한 실행 (윈도우의 "관리자 권한으로 실행"과 같은 개념)
- `dpkg -i` / `apt install ./파일.deb`: 로컬에 받아둔 `.deb` 파일 설치
- alias도 **Tab 자동완성이 된다** — 앞 몇 글자만 독특하면 끝까지 안 쳐도 됨

---

## 3. 2-3 ~ 2-4 — bashrc 자동 실행에서 alias 방식으로 전환

### 왜 바꿨나 (핵심 설계 판단)

- 처음엔 `.bashrc`에 `source /opt/ros/humble/setup.bash`를 **직접 넣어서** 터미널 열 때마다 ROS 2가 자동으로 켜지게 했다. (Day 4에서 한 것)
- **문제**: 이 노트북에서 ROS 말고 파이썬 등 다른 환경도 쓸 수 있는데, ROS 2가 항상 켜져 있으면 나중에 여러 환경(다른 ROS 버전, 가상환경 등)이 섞일 때 충돌을 막을 수단이 없다.
- **해결**: alias로 만들어 **필요할 때만 명시적으로 호출**하는 방식으로 전환.
  → **`conda activate`와 완전히 같은 개념.** 설치·등록은 한 번만, 활성화는 새 터미널마다 매번.

### alias 문법

```bash
alias 이름="실행할 명령"
```

- ⚠️ **`이름`과 `=` 사이에 공백이 있으면 안 된다.** 반드시 붙여 쓴다.
- 명령 안에서 따옴표를 쓰려면 **역슬래시로 이스케이프**한다: `\"`
  (바깥 `"`는 "여기부터 여기까지가 alias 내용"이라는 표시고, 안쪽 `\"`는 "문자로서의 따옴표"라는 뜻)
- 여러 명령을 이어 실행하려면 `;`로 연결한다.

### 최종 alias 구성 (`~/.bashrc` 맨 아래)

```bash
alias rebash="source ~/.bashrc; echo \"bashrc is reloaded!\""
alias ros_domain="export ROS_DOMAIN_ID=13; echo \"ROS_DOMAIN_ID=13\""
alias humble="source /opt/ros/humble/setup.bash; ros_domain; echo \"ROS2 Humble is activated!\""
```

| alias | 역할 | 현재 상태 |
|---|---|---|
| `rebash` | `.bashrc` 수정 후 터미널 재시작 없이 즉시 반영 | ✅ 등록 완료 |
| `ros_domain` | `ROS_DOMAIN_ID` 환경변수 설정 (DDS 통신 그룹 분리) | ❌ **미등록 — 다음에 추가** |
| `humble` | ROS 2 환경 활성화 + 확인 메시지 | ⚠️ 등록됐으나 `ros_domain;` 호출 부분 없음 — **수정 필요** |

### alias 트러블슈팅 — 실제로 겪은 오타들

- **`alias humble`(조회) vs `humble`(실행)은 다른 명령이다.**
  `alias 이름`은 정의만 보여주고, `이름`만 쳐야 실제로 실행된다.
- **오타 1**: `echo\"...\"` — echo와 따옴표 사이 **공백 누락** → `echoROS2...: command not found`
- **오타 2**: `~/bashrc` — **점 누락** → 존재하지 않는 파일을 참조하게 됨
- **교훈**: alias 정의처럼 따옴표·이스케이프가 많은 줄은 저장 후 **`cat ~/.bashrc`로 정확히 들어갔는지 먼저 눈으로 확인**하고 실행하는 습관이 안전하다.
- **검증은 반드시 새 터미널에서**: 같은 세션에서 `source`만 반복하면 이전에 자동 활성화됐던 흔적(예전 PATH 등)이 남아 있어 "고쳤는데 왜 되지?"처럼 헷갈릴 수 있다.

---

## 4. 2-5 — ROS 1 vs ROS 2의 결정적 차이: DDS와 도메인 ID

### DDS (Data Distribution Service)란

중앙 관리자 없이 노드들이 네트워크에서 **서로 자동으로 찾아** 메시지를 주고받는 국제 표준 통신 미들웨어.
OMG(Object Management Group) 표준이며, 실시간성·규모가변성·안정성·고성능을 목표로 항공우주·국방 등에서 검증된 규격이다.

| | ROS 1 | ROS 2 |
|---|---|---|
| 통신 구조 | **`roscore`라는 중앙 관리자 필수** | **DDS 채택 — 중앙 관리자 없음** |
| 단일 장애점 | roscore가 죽으면 **전체 마비** | 각 노드가 알아서 서로 발견 → 더 안정적 |

### 트레이드오프: 같은 네트워크면 전부 섞인다

- DDS는 같은 네트워크(같은 공유기)에 있으면 **기본적으로 서로 다 발견해버린다.**
- → 실습실처럼 여러 명이 같은 와이파이를 쓰면 **남의 talker/listener까지 내 화면에 섞여 들어온다.**
  (Day 4에서 강사가 경고했던 그 상황)

### 해결책: `ROS_DOMAIN_ID`

```bash
export ROS_DOMAIN_ID=13
```

- 같은 네트워크 안에 있어도 **도메인 번호가 다르면 별개 그룹으로 인식** → 안 섞인다.
- 값은 **0~101 범위 내 임의의 숫자**. 우리는 13을 쓰기로 함.
- 혼잡 문제 자체는 QoS 설정, Discovery Server(선택적 경량 중계) 등으로도 조절 가능.

### 💭 생각해본 것 — LLM 오케스트레이션과의 관계

- **DDS/ROS 2는 "메시지를 빠르고 안정적으로 전달"하는 통신 인프라 층**이다. 그 자체에 지능은 없다.
- "상황을 판단해서 필요한 노드를 알아서 찾고 관리하는 LLM"이라는 아이디어는 **그 위에 얹는 의사결정 층**이다.
- 즉 **서로 대체 관계가 아니라 함께 쓰는 구조** — LLM이 두뇌, ROS 2/DDS가 신경망.
  Physical AI 분야의 실제 연구 방향과도 일치하는 그림.

---

## 5. 다음에 할 일 (리눅스 노트북 켜면 이것만)

- [x] `rebash`는 이미 만들어놨으니 다시 만들 필요 없음
- [ ] `nano ~/.bashrc`로 열어서 `ros_domain` alias 새로 추가
  ```bash
  alias ros_domain="export ROS_DOMAIN_ID=13; echo \"ROS_DOMAIN_ID=13\""
  ```
- [ ] 기존 `humble` alias에 `ros_domain;` 호출 추가하도록 수정
  ```bash
  alias humble="source /opt/ros/humble/setup.bash; ros_domain; echo \"ROS2 Humble is activated!\""
  ```
- [ ] 저장 후 **`cat ~/.bashrc`로 오타(따옴표/공백) 없이 정확히 들어갔는지 먼저 확인**
- [ ] **새 터미널을 열어** 검증:
  1. `ros2` → `command not found` 확인 (아직 활성화 전이므로 정상)
  2. `humble` 실행
  3. `echo $ROS_DOMAIN_ID` → `13` 나오는지 확인
  4. `ros2` 다시 → 정상 작동 확인
- [ ] 그 다음 진도: 입문 3-1 Turtlesim 실행하기 ~ 3-9 ROS2 Action

---

## 6. 오늘의 회고

- 진도상으로는 3개(약 23분)뿐이지만, **환경을 어떻게 설계할지에 대한 판단**을 배운 날이었다.
  "자동으로 항상 켜두기"에서 "필요할 때만 부르기"로 바꾼 게 핵심 — 지금은 ROS 하나뿐이라 차이가 없어 보여도,
  나중에 환경이 늘어났을 때 충돌을 막아주는 구조를 미리 깔아둔 것.
- `source` vs `./실행`, `.bashrc`의 점 vs `source`의 점처럼 **비슷하게 생겼는데 완전히 다른 것들**을 구분한 것도 수확.
- 오타 2개로 막혔던 경험에서 "저장 후 `cat`으로 눈 검증 → 그다음 실행" 순서를 습관으로 삼기로 함.

---

## 참고 자료

- [R2R 무작정 따라하는 ROS2 - 입문과정 (핑크랩 PinkLAB)](https://www.youtube.com/playlist?list=PL0xYz_4oqpvhj4JaPSTeGI2k5GQEE36oi)
- [The ROS_DOMAIN_ID — ROS 2 공식 문서](https://docs.ros.org/en/humble/Concepts/Intermediate/About-Domain-ID.html)
- [ROS 2 Humble 공식 설치 문서](https://docs.ros.org/en/humble/Installation/Ubuntu-Install-Debians.html)
