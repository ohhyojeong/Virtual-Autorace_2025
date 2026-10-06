# 사후 분석

[← README](../README_ko.md) · [English](postmortem.md) | **한국어**

대회 후 팀 채팅, Notion, 작업 공간 스냅샷, 공용 PC 편집 이력·ROS 로그로 재구성함, *(추정)* 항목은 기록으로 확인 불가

## 요약

| # | 근본 원인 | 관찰 증상 | 시기 | `main` 상태 |
|---|---|---|---|---|
| 1 | TEB 파라미터 파일 미적용 (YAML 키 오타) | 저속, 미로 정지, 정적 장애물 우회 불가 | 08.11~08.18 | 수정 완료 (MORAI 미검증) |
| 2 | EKF·AMCL 모두 `map → odom` 발행 | `base_link` 떨림, 위치 추정이 차량을 따라가지 못함 | 08.07~08.12 | 대회 당시 이미 정상 설정 |
| 3 | 명령 기반 추측항법 odometry | `/odom` 잡음·정지, 안정적이나 부정확 | 08.12~08.13 | 미조치 |
| 4 | 차선 노드: 고정 스로틀, 조향 오프셋, 차선 1개일 때 종료 | "PD" 효과 없음, 실제 주행 불가 | 07.24~본선 | 미조치 |

## 1. TEB 파라미터 미적용

`wego_2d_nav/params/teb_local_planner_params.yaml`의 최상위 키는 `TebLocalPlannerRos` (소문자 `os`), `move_base.launch`는 `ns="TebLocalPlannerROS"`로 파일 로드 → ROS 파라미터 이름의 대소문자 구분으로 TEB가 읽지 않는 위치에 값 저장

```text
Where TEB reads    : /move_base/TebLocalPlannerROS/max_vel_x                    → not set
Where it was stored: /move_base/TebLocalPlannerROS/TebLocalPlannerRos/max_vel_x
```

TEB는 소스 코드 기본값 (`teb_config.h`)으로 실행됨:

| 파라미터 | 의도한 값 | 실제 사용한 기본값 | 관찰 증상 |
|---|---|---|---|
| `max_vel_x` | 2.0 | **0.4** | `max_vel_x`를 높여도 속도 증가 없음 |
| `min_turning_radius` | 0.755 (Ackermann) | **0** (제자리 회전 가정) | 실제 차량이 따라갈 수 없는 경로 → 미로 정지 |
| `max_global_plan_lookahead_dist` | 2.0 | **1.0 m** | 지역 경로의 전방 탐색 거리 부족 → 정적 장애물 회피 실패 |
| `min_obstacle_dist` | 0.6 | 0.5 | — |
| `odom_topic` | `/odometry/filtered` | `odom` | — (영향 없음: `move_base.launch`에서 `odom` → `/odometry/filtered` remap) |

시기별 근거:

| 날짜 | 근거 |
|---|---|
| 08.11 | `max_vel_x` 2 → 약 6 변경 효과 없음 (오효정) |
| 08.11 18:20 | 대여 노트북도 같은 저속 → 성능 가설 제외 |
| 08.11 21:00 | **놓친 단서**: 박시현이 붙여넣은 TEB 코드에 `saturateVelocity(cfg_.robot.max_vel_x)` 포함, 실행 중 해당 값 확인 시 오타 발견 가능했음 |
| 08.12 | rqt 스크린샷에 YAML 값 대신 TEB 기본값 표시 |
| 08.12~08.13 | 강의 costmap 설정 (08.12 적용) 효과 없음, `cost_scaling_factor` 하향 후 소폭 개선 (08.13) |
| 08.17 | `max_vel_x` 100 변경 효과 없음 |
| 08.18 | 주행 로그 17개 모두 `max_vel_x: 0.4`, `No robot footprint model specified` |

대회 당일 `throttle_interpolator.py`의 모터 배율 ×2로 TEB 속도 제한 우회

## 2. EKF·AMCL의 `map → odom` 충돌

| 시각 | 내용 |
|---|---|
| 08.07 | 같은 날 앞서 EKF `world_frame`을 `odom` → `map`으로 변경 후 정상 SLAM 지도 생성, 변경으로 지도 겹침 해소 ([개발 중 기타 문제](#개발-중-기타-문제) 참고) 및 아래 충돌 시작 *(추정)* |
| 08.07~08.12 | robot_localization 예제의 `ekf_se_odom`에 `world_frame: map` 설정 (upstream 기본값 `odom`) → EKF가 **AMCL과 같은 변환** `map → odom` 발행 |
| 08.12 14:42 | 문제 목록 (오효정): 이상한 지역 경로, 정지 중 `/odom` x속도 튐, `base_link` TF가 위치 추정을 따라가지 못함 |
| 08.12 14:47 | 김채연: 프레임 `map` → `odom` 복구 제안 |
| 08.12 15:28 | 변경 후 떨림 해소, 위치 추정은 여전히 상실 |
| 08.12 18:01 | 추가 변경 없이 위치 추적 복구 — 원인 불명 |
| 08.12 18:53 | TF 안정, odometry 갱신 중단, 비슷한 시각 김채연이 rqt TF에서 `base_link` 연결 끊김 확인 *(김채연 회고)* |
| 08.13 12:49 | 오효정 실험: `vesc_to_odom/publish_tf: True` + `pub_odom.py` → 떨림 없음·부정확 · `True` + EKF → TF 떨림 · `False` + EKF → TF 완전 연결 끊김 |

근거 해석:
- `map → odom` 발행자 2개 (`world_frame: map`인 EKF·AMCL)가 서로 덮어씀 → `base_link` 떨림
- EKF가 `map`이고 `vesc_to_odom` TF가 꺼져 있으면 `odom → base_link` 발행자 부재 → odometry 정지, TF 트리 연결 끊김
- 대회 당일 설정 정상: `world_frame: odom`, EKF만 `odom → base_link` 발행, `vesc_to_odom` TF 비활성화

## 3. 명령 기반 추측항법 odometry

- `vesc_to_odom`은 `/sensors/core`의 모터 속도·**팀이 보낸 조향 명령** (`/sensors/servo_position_command`)을 적분함, 차량의 실제 움직임 측정 없음
- `speed_to_erpm_gain` 1000, MORAI 명시 규격 (1 km/h당 300)은 약 1080에 해당 → 속도 오차 약 8% *(추정)*
- 08.12 정지 중 x속도 튐은 남은 기록으로 설명 불가 (`/odom` 기록 없음)
- 08.18 주행 전 공유한 AMCL 파일의 `recovery_alpha_slow/fast = 0.001`은 무작위 입자 복구를 사실상 비활성화 → AMCL 위치 상실 후 복구 어려움

## 4. 차선 추종 노드 문제

`lane_following` 노드 (07.24~07.26)는 팀의 SEA:ME 해커톤 PiRacer 차량 코드에서 이식함: `compute_pd_control` (김채연, 07.17), `plothistogram` / `sliding_window` (박시현, 07.16), 이식 시 `compute_pd_control` 3줄 변경, 아래 문제가 최종 코드에도 남아 있음:

| 문제 | 영향 |
|---|---|
| `throttle = max(min(abs(pd), 0.25), 0.25)` | 속도 항상 0.25, 해커톤 버전은 제한 범위 0~1.0으로 PD 항이 실제 속도 조절 |
| 조향 = deviation/180을 ±0.4로 제한 후 0.23 차감 | 단순 비례 조향, −0.23은 **PiRacer 중립 조향 보정값**을 그대로 사용 |
| 직진 시 조향 0.0 전송 | PiRacer 조향 중심은 0 부근, MORAI 서보는 0~1 범위에서 직진 값 0.5 → 0.0은 최대 조향 명령 *(추정)* |
| 차선 1개 검출 시 `sliding_window`가 값 1개 반환 | unpacking 오류 → 노드 종료 |
| 1 Hz 루프 + `waitKey` 지연 | 폐루프 주행에 너무 느림 |
| 흰색 마스크만 사용 | 07.26 추가한 노란 차선 마스크는 최종 노드에 미포함 |

07.26 신호등 정지 시도에도 토픽 오타 (`/GetTraficLightStatus`)·메인 루프의 정지 명령 덮어쓰기 발생, 같은 날 저녁 BEV(버드아이뷰)·캘리브레이션 버전 제외, 이유 기록 없음

## 5. 오타 수정 시 예상 *(추정)*

TEB 키만 수정해도 주행 성능 위주로 개선 — ×2 우회 없이 최고 속도 0.4 → 2.0 m/s, 제자리 회전 대신 실제 회전 반경 (0.755 m), 전방 탐색 거리 1 → 2 m

주행 외 미션은 팀원들이 Notion에 작성했으나 최종 launch에 통합하지 않은 프로토타입 코드에 의존

| 미션 | 프로토타입 코드 (담당) | 남은 장애 요인 | 판단 |
|---|---|---|---|
| 미로 탈출 | 최종 스택 | 미검증 TEB 값: `min_obstacle_dist` 0.6 (08.16에는 0.25 사용)으로 좁은 통로 차단 가능, 미로 출구 목표 방향 반대 (178°) — TEB 제자리 회전 허용 시 드러나지 않음 | 두 시도 모두 가능성 높음 |
| 배달 | `run_delivery_step` (박시현) | 최종 버전에서 미정의 `self.has_delivery` / `self.delivery_route` 참조 → 오류로 종료, 도달 반경 2.0 m (규정 1.2 m), 목표 `unique_id == 50`으로 하드코딩 | 소규모 수정 후 가능, MORAI 실행 이력 없음 |
| 정적 장애물 | 없음 — TEB만 사용 | 1 m 간격 도로 waypoint에 고정 전방 탐색 거리 2 m 사용 시 차량이 막힌 차로로 복귀할 가능성 | 통과 가능성 있음 |
| 동적 장애물 (보행자) | LiDAR DBSCAN 클러스터링 → `/obstacle_class`, `/detected_car_vel` (오효정) | 감지만 구현, 정지 로직 없음 — TEB에 `obstacles`를 전달한 버전은 보행자 회피 주행 유도, 규정상 실패 | 새 코드 필요 |
| 회전교차로 | 없음 | 양보·차선 유지 로직 없음 | 새 코드 필요 |
| 신호등 | 구독 노드 (오효정, 박시현) | 토픽 `/getTrafficLightStatus`와 실제 `/GetTrafficLightStatus` 불일치, 정지선 위치 (0, 0) 그대로, 상태 16 (초록불)에 정지, 구독자 없는 `traffic_light_vel` 발행 | 4개 수정 필요 |

가장 가능성 높은 결과: 두 시도 모두 미로 탈출·정적 장애물 통과 후 첫 보행자나 신호등에서 정지 또는 감점

프로토타입으로 모든 미션의 개발 착수 확인, 부족했던 단계는 통합 — 미션별 행동 전환과 최종 `cmd_vel` 제어를 담당하는 노드 필요

## 계획과 구현

| 항목 | 예선 설계 (06.23~06.24) | 본선 (08.18) | 변경 이유 |
|---|---|---|---|
| 미들웨어 | Nav2 기반 (ROS2 가정) | ROS1 Noetic | 규정서: Ubuntu 20.04 / ROS Noetic, rosbridge |
| 예상 시뮬레이터 | Gazebo / CARLA | MORAI | 7월 공개 |
| 주 센서 | 카메라 (상부 YOLO, 하부 차선) | 2D LiDAR | 미션 1은 벽이 있는 미로, 규정서 절대좌표 금지 |
| 위치 추정 | IMU + GPS 기반 SLAM | gmapping 지도 + AMCL | 규정상 GPS 금지 |
| 전역 플래너 | PRM | GlobalPlanner (Dijkstra) | 강의 내비게이션 스택에 포함 |
| 지역 플래너 | TEB (예비안 DWA + Pure Pursuit) | TEB | 유지 |
| 행동 | Behavior Tree | 미로 → 도로 전환 스크립트 | 신호등·배달·장애물 로직 구현 시간 부족 |
| 인식 | YOLO + OpenCV | 차선 노드만 (본선 미사용) | YOLO 미구현 |

## 규정 대비 결과

| 미션 (규정서) | 요구 사항 | 수행 결과 |
|---|---|---|
| 1. SLAM & Navigation | 자체 SLAM 지도, 수동 위치 추정 금지, 물체 2개를 목표 지점 1개에 배달 (1.2 m 이내 2초 유지) | 지도·AMCL 동작, 2차 미로 탈출, 배달 미수행 |
| 2–3. 장애물 | 보행자: 정지·대기 (회피 시 실패), 정적 장애물: 차선 변경 | 첫 정적 장애물에서 정지 (2차) |
| 4. 회전교차로 | 충돌 없이 진입, 중앙선 침범 금지 | 미도달 |
| 5. 신호등 | 빨간불에 정지선 0.3 m 이내 정지, 초록불에 5초 이내 출발 | 미도달, 최종 스택에 로직 없음 |

PC 2대 이더넷 구성 (08.05~08.07)은 팀 자체 선택이 아닌 규정서 요구 사항임: 대회 측 PC에서 MORAI, 팀별 노트북에서 알고리즘 실행·유선 연결 (고정 IP 192.168.12.10 / .20)

## 기타 결함

| 결함 | 영향 | `main` 상태 |
|---|---|---|
| TEB YAML 키 `TebLocalPlannerRos` ≠ namespace `TebLocalPlannerROS` | TEB 기본값 사용 | 수정 완료 |
| 미로 출구 목표 quaternion `w`/`z` 뒤바뀜 (`navigation_client.py`) | 목표 방향 178°, 도로 방향 (0°)과 반대 | 수정 완료 |
| 미로 → 도로 전환 조건에 `and` 대신 `or` 사용 | x 또는 y 하나만 가까워도 조기 전환 가능 | 수정 완료 |
| GlobalPlanner namespace 밖의 `use_dijkstra`, costmap `inflation_unknow` 오타 | 해당 설정 무시 (강의 예제에서 이어진 설정) | 수정 완료 |
| 도로 waypoint 경로 `~/ws/waypoint_1m.csv`로 고정 | 공용 PC에서만 동작 | 수정 완료 (`wego/waypoints/`로 이동) |
| 목표 결과 미확인 (ABORTED도 다음으로 진행) | 실패한 waypoint를 알림 없이 건너뜀 | 미해결 |
| AMCL 복구 사실상 비활성화 (`recovery_alpha_* = 0.001`) | 위치·자세 상실 후 자체 복구 불가 | 미해결 |
| 차선 노드 문제 (4절) | 차선 추종 불가 | 미해결 |
| 노트북 성능 | move_base 제어 루프 목표 10–20 Hz 대비 실제 약 2 Hz | 하드웨어 |

`main` 수정 사항은 커밋 1개에 포함 ("Fix bugs found in the post-contest analysis"), 문법·파라미터 위치만 확인 — **MORAI 주행 미검증**

## 개발 중 기타 문제

| 시기 | 문제 | 결과 |
|---|---|---|
| 07.20~07.23 | VM에서 MORAI 종료 (GPU 접근 불가) | Windows의 MORAI + WSL2의 ROS 실행 |
| 08.03~08.04 | 초기 SLAM 지도 겹침 (첫 지도 아래 두 번째 지도), gmapping 반복 종료 | *(추정)* `ekf=true`에서 EKF (`world_frame: odom`)와 `pub_odom.py` 모두 `odom → base_link` 발행 → gmapping odometry가 두 추정값 사이에서 변동, 08.07 EKF를 `world_frame: map`으로 변경 → `pub_odom.py`만 발행하여 정상 지도 생성, EKF의 새 `map → odom`은 문제 2로 연결 |
| 08.06 | 이더넷으로 PC 2대 연결했으나 ROS PC의 `rostopic list` 비어 있음 | 08.07 정상 동작, 수정 기록 없음 |
| 08.13 | costmap 장애물로 처리한 차선이 다중·교차 차선 통과 차단 | 포기, 원래 계획인 waypoint 유지, via-points 설계·미통합 |
| 08.16 | 공용 PC의 갑작스러운 costmap 변화 | 원인 기록 없음, 이후 정지 원인을 맵 대신 플래너 문제로 확인 |

## 파라미터 튜닝 이력

| 파라미터 | 08.11 | 08.13 | 08.16 | 최종 (08.18) |
|---|---|---|---|---|
| 공통 `inflation_radius` / `cost_scaling_factor` | 0.4 / 10 | 0.25 / 13 | 0.15 / 3 | 0.15 / 10 |
| 전역 `inflation_radius` | 0.2 | 0.5 | 0.15 | 0.15 |
| 장애물 레이어 | Voxel | Voxel | Voxel | **ObstacleLayer** |
| TEB `max_vel_x` | 2.0 | 2.0 | 2.0 | 2.0 |
| TEB `min_obstacle_dist` | 0.4 | 0.6 | 0.25 | 0.6 |
| TEB `weight_kinematics_nh` | 1000 | 1000 | 1000 | 200 |
| `controller_frequency` / `patience` | 10 / 7 | 10 / 7 | 5 / 7 | 20 / 2 |
| AMCL 입자 수 (최소/최대) | 900/2000 | 800/2500 | 800/2500 | 500/2000 |

- 각 스냅샷 파일의 값 기준, TEB 행은 미적용 (1절), rqt_reconfigure로 실행 중 변경한 값은 예외
- 08.13 박시현은 `cost_scaling_factor` 하향 후 개선 보고, 다만 08.13 스냅샷 파일은 13 (10에서 증가) → rqt 변경·다른 파일에서 개선됐을 가능성

## 배운 점

- 튜닝 효과가 없으면 값을 더 바꾸기 전에 노드가 실제 읽는 값 확인 (`rosparam get`, rqt)
- 변환 1개당 발행자 1개: EKF 추가 전 TF 연결별 발행자 정리
- 실험 중 원본 토픽 (`/odom`, `/tf`, `/sensors/core`) 기록, 기록 부재로 대회 후 odometry 잡음 설명 불가
