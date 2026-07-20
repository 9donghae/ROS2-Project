# ROS2-Based CQB Multi-Robot System

<p align="center">
  <img src="https://img.shields.io/badge/ROS2-Jazzy-22314E?style=for-the-badge&logo=ros&logoColor=white"/>
  <img src="https://img.shields.io/badge/Python-3776AB?style=for-the-badge&logo=python&logoColor=white"/>
  <img src="https://img.shields.io/badge/TurtleBot3-Burger%20%7C%20Waffle-00A3E0?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Nav2-Autonomous%20Navigation-4CAF50?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/YOLO-Object%20Detection-00FFFF?style=for-the-badge"/>
  <img src="https://img.shields.io/badge/Jetson-Orin%20Nano-76B900?style=for-the-badge&logo=nvidia&logoColor=white"/>
</p>

## 프로젝트 소개

본 프로젝트는 **정찰 로봇, 지원 로봇, 주요 임무 로봇이 역할을 분담하는 ROS2 기반 다중 로봇 시스템**입니다.

정찰 로봇이 Cartographer SLAM으로 공용 지도를 생성하고 위험지도를 유지하면, 주요 임무 로봇은 해당 지도를 전달받아 AMCL과 Nav2를 이용해 자율주행합니다. 정찰 카메라 영상은 Leader의 Jetson으로 전달되며, YOLO 인식 결과를 위험지도와 OMX 자동 조준 파이프라인에 활용합니다.

각 로봇은 서로 다른 ROS Domain에서 동작하며, 주행 명령은 자신의 Domain에서만 발행하도록 구성해 명령 충돌을 방지했습니다.

---

## 시스템 구조

<p align="center">
  <img src="./docs/images/system_flowchart.png" width="100%" alt="ROS2 Project System Flowchart"/>
</p>

### 전체 흐름

```text
Scout22가 SLAM 지도 생성
        ↓
위험지도 생성 및 카메라 프레임 전송
        ↓
Leader20이 공유 지도 수신
        ↓
AMCL + Nav2 기반 자율주행
        ↓
Jetson에서 YOLO 인식 및 OMX AIM 실행
        ↓
PC의 RViz / Viewer / Dashboard에서 상태 확인
```

---

## 로봇별 역할

### 1. Scout22 — Active Scout

- **Model:** TurtleBot3 Burger
- **ROS Domain ID:** `22`
- **Initial Role:** `ACTIVE_SCOUT`
- Cartographer SLAM 기반 기준 지도 생성
- Bayesian Risk Map 유지
- 카메라 프레임을 Leader Jetson의 Flask YOLO 서버로 전송
- 정찰 및 위험지역 탐색 수행

Scout22는 시스템의 기준이 되는 `/map`을 생성합니다. Cartographer가 지도를 소유하는 동안 하위 Nav2는 기본적으로 실행하지 않아 CPU 부하와 `/cmd_vel` 경합을 줄입니다.

### 2. Follower21 — Standby Follower

- **Model:** TurtleBot3 Burger
- **ROS Domain ID:** `21`
- **Initial Role:** `FOLLOWER`
- 평상시 Leader 추종 또는 대기
- Shared Map + AMCL + Nav2 기반 이동
- Scout 장애 발생 시 마지막 Scout 위치로 이동
- 도착 후 `ACTIVE_SCOUT` 역할 인계
- 자신의 Domain에서만 `/cmd_vel` 발행

Follower21은 단순 대기 로봇이 아니라 **장애 발생 시 정찰 임무를 이어받는 예비 정찰 로봇**입니다.

### 3. Leader20 — Mission Leader

- **Model:** TurtleBot3 Waffle
- **ROS Domain ID:** `20`
- **Initial Role:** `LEADER`
- Scout Domain의 지도를 `/map_bridge`로 수신
- 자체 LiDAR/Odometry 기반 AMCL 위치 추정
- Nav2 기반 목표 지점 주행
- Jetson에서 Flask YOLO 서버 실행
- OMX AIM 파이프라인 실행
- 통합 웹 대시보드 실행

Leader는 자체 Cartographer를 실행하지 않고, 현재 Active Scout가 제공하는 공유 지도를 이용해 주행합니다.

---

## 핵심 기능

### Multi-Domain Architecture

| Robot | Model | Domain | Initial Role |
|---|---|---:|---|
| Scout22 | Burger | 22 | ACTIVE_SCOUT |
| Follower21 | Burger | 21 | FOLLOWER |
| Leader20 | Waffle | 20 | LEADER |

- 각 Burger는 자신의 Domain에서 `scan`, `map`, `TF`를 읽습니다.
- 각 로봇은 자신의 `/cmd_vel`만 발행합니다.
- RL Action과 `/cmd_vel`은 Domain 간 Bridge하지 않습니다.
- 지도, 위치, 상태, 위험지도 등 필요한 데이터만 선택적으로 전달합니다.

### SLAM & Shared Map

- Scout22가 Cartographer SLAM으로 기준 지도를 생성
- Leader20은 `/map_bridge`를 통해 공유 지도 수신
- Follower21은 Shared Map 기반 AMCL/Nav2로 대기 또는 추종
- Active Scout 전환 시 Follower가 자체 Cartographer/Risk Stack 실행

### Risk Map

- Bayesian Risk Map 기반 위험도 레이어 유지
- YOLO Detection Topic을 이용해 지도 좌표계에 위험 정보 반영
- Active Scout가 변경되면 현재 정찰원의 Map/Risk Data를 Leader가 선택

### Camera & YOLO Offloading

- Scout의 카메라 프레임을 Leader Jetson으로 전송
- Leader의 Flask YOLO 서버에서 추론 수행
- 추론 결과를 Scout Domain의 `/risk/yolo_detections`로 전달
- Scout의 연산 부하를 줄이고 Jetson의 GPU 성능 활용

### Autonomous Navigation

- Leader/Follower: AMCL + Nav2 기반 위치 추정 및 자율주행
- Leader Dashboard에서 Nav2 Path 시각화
- 현재 경로의 Frame, Pose Count, 시작·종료 좌표, Age 확인

### Failover

1. Active Scout의 장애 감지
2. Follower가 마지막 Scout 위치로 이동
3. 도착 후 `ACTIVE_SCOUT`으로 역할 전환
4. Cartographer, Risk Map, Camera Sender 활성화
5. 정찰 및 위험지도 생성을 이어서 수행

### OMX AIM

- 목표 방향 기반 대략 조준
- YOLO 기반 표적 검출
- 영상 중심 오차 기반 정밀 조준
- 안정 상태 확인
- OMX AIM 동작 수행

---

## 패키지 구성

```text
src/
├── bayesian_risk_map/       # 위험지도 생성 및 관리
├── flask_yolo_bridge/       # 카메라 프레임 전송 및 YOLO 연동
├── fleet_bringup/           # 다중 로봇 주행 스택
├── multi/                   # 다중 로봇 관련 기능
├── omx_aim/                 # OMX 자동 조준
├── region_mapper/           # 공간/영역 매핑
├── system_bringup/          # 역할별 전체 시스템 통합 실행
└── turtlebot3_rl_training/  # 강화학습 기반 정찰 정책
```

---

## 실행 방법

### 1. 공통 환경 설정

```bash
cd ~/ROS2-Project
source /opt/ros/jazzy/setup.zsh
source install/setup.zsh
```

### 2. Scout22 실행

```bash
export ROS_DOMAIN_ID=22
export TURTLEBOT3_MODEL=burger

ros2 launch system_bringup field_robot.launch.py \
  robot_name:=scout22 \
  domain_id:=22 \
  initial_role:=ACTIVE_SCOUT
```

### 3. Follower21 실행

```bash
export ROS_DOMAIN_ID=21
export TURTLEBOT3_MODEL=burger

ros2 launch system_bringup field_robot.launch.py \
  robot_name:=follower21 \
  domain_id:=21 \
  initial_role:=FOLLOWER
```

### 4. Leader20 실행

```bash
export ROS_DOMAIN_ID=20
export TURTLEBOT3_MODEL=waffle

ros2 launch system_bringup field_robot.launch.py \
  initial_role:=LEADER \
  robot_name:=leader \
  domain_id:=20 \
  risk_domain_id:=22
```

### 5. PC Debug Viewer 실행

```bash
ros2 launch system_bringup pc.launch.py
```

PC에서는 기본적으로 YOLO 서버를 실행하지 않습니다. 컴포넌트 디버깅이 필요한 경우에만 다음 옵션을 사용합니다.

```bash
ros2 launch system_bringup pc.launch.py start_yolo_server:=true
```

---

## 주요 Launch 파일

| Launch File | Description |
|---|---|
| `field_robot.launch.py` | Leader, Active Scout, Follower의 통합 진입점 |
| `system.launch.py` | 역할별 내부 전체 스택 실행 |
| `pc.launch.py` | PC에서 RViz 및 디버그 Viewer 실행 |
| `viewer.launch.py` | Fleet Marker와 Risk Map을 RViz에서 확인 |

---

## 초기 배치

공유 지도 좌표계를 기준으로 다음 위치에서 시작합니다.

| Robot | Initial Pose |
|---|---|
| Leader20 | `(0.00, 0.00, 0.0)` |
| Scout22 | `(0.00, +0.20, 0.0)` |
| Follower21 | `(0.00, -0.20, 0.0)` |

세 로봇 모두 `yaw=0.0`, 즉 `+x` 방향을 바라봅니다.

---

## 사용 기술

- **Robotics:** ROS2 Jazzy, TurtleBot3 Burger, TurtleBot3 Waffle
- **Mapping:** Cartographer SLAM, Bayesian Risk Map
- **Localization:** AMCL, TF/TF2, Odometry
- **Navigation:** Nav2, Path Planning, Lifecycle Management
- **AI / Vision:** YOLO, Flask YOLO Bridge, Camera Streaming
- **Edge Computing:** Jetson Orin Nano
- **Control:** OMX AIM
- **Visualization:** RViz2, Web Dashboard
- **Development:** Python, Linux/Ubuntu, Git

---

## 주요 성과

- 세 로봇을 서로 다른 ROS Domain으로 분리한 다중 로봇 구조 구축
- Scout가 생성한 지도를 Leader가 공유받아 주행하는 Shared Map 구조 구현
- YOLO 추론을 Leader Jetson으로 Offloading하여 Scout 연산 부하 감소
- Active Scout 장애 시 Follower가 정찰 역할을 인계하는 Failover 구현
- 위험지도, Robot Marker, Nav2 Path를 통합 Dashboard와 RViz에서 시각화
- `system_bringup`을 통한 역할별 전체 시스템 일괄 실행

---

## Repository

- GitHub: `https://github.com/sangaje/ROS2-Project`
