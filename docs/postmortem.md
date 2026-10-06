# Post-mortem

[← README](../README.md) · **English** | [한국어](postmortem_ko.md)

Reconstructed after the contest from the team chat, Notion, workspace snapshots, the shared PC's edit history and ROS logs. Items marked *(estimate)* could not be confirmed from the records.

## Summary

| # | Root cause | What we saw | When | Status on `main` |
|---|---|---|---|---|
| 1 | TEB parameter file never applied (YAML key typo) | Car crawled, got stuck in the maze, could not get around the static obstacle | 08.11 – 08.18 | Fixed (untested in MORAI) |
| 2 | EKF and AMCL both published `map → odom` | `base_link` jittered, localization did not follow the car | 08.07 – 08.12 | Already correct at contest time |
| 3 | Odometry was dead reckoning from commands | `/odom` noisy or frozen; stable but inaccurate | 08.12 – 08.13 | Not addressed |
| 4 | Lane node: fixed throttle, offset steering, crash with one lane | "PD" had no effect; node unusable for real driving | 07.24 – final | Not addressed |

## 1. TEB Parameters Were Never Applied

The top-level key in `wego_2d_nav/params/teb_local_planner_params.yaml` was `TebLocalPlannerRos` (lowercase `os`). `move_base.launch` loaded it with `ns="TebLocalPlannerROS"`, and ROS parameter names are case-sensitive, so the values landed where TEB never reads.

```text
Where TEB reads    : /move_base/TebLocalPlannerROS/max_vel_x                    → not set
Where it was stored: /move_base/TebLocalPlannerROS/TebLocalPlannerRos/max_vel_x
```

TEB therefore ran on its source defaults (`teb_config.h`):

| Parameter | Intended | Default actually used | Observed symptom |
|---|---|---|---|
| `max_vel_x` | 2.0 | **0.4** | Raising `max_vel_x` did not increase speed |
| `min_turning_radius` | 0.755 (Ackermann) | **0** (assumes in-place rotation) | Paths the car could not follow → stuck in the maze |
| `max_global_plan_lookahead_dist` | 2.0 | **1.0 m** | Local plan looked too short a distance ahead → failed static obstacle avoidance |
| `min_obstacle_dist` | 0.6 | 0.5 | — |
| `odom_topic` | `/odometry/filtered` | `odom` | — (no effect: `move_base.launch` remaps `odom` → `/odometry/filtered`) |

Evidence over time:

| Date | Evidence |
|---|---|
| 08.11 | Raising `max_vel_x` from 2 to ~6 changed nothing (Hyojeong Oh) |
| 08.11 18:20 | The borrowed laptop was just as slow → performance hypothesis dropped |
| 08.11 21:00 | **Missed clue**: the TEB code Sihyeon Park pasted contained `saturateVelocity(cfg_.robot.max_vel_x)`. Checking that value at runtime would have exposed the typo |
| 08.12 | rqt screenshots show TEB defaults, not the YAML values |
| 08.12 – 08.13 | Costmap settings from the lecture (applied 08.12) did not help; lowering `cost_scaling_factor` helped slightly (08.13) |
| 08.17 | Raising `max_vel_x` to 100 changed nothing |
| 08.18 | All 17 run logs: `max_vel_x: 0.4`, `No robot footprint model specified` |

The team finally bypassed TEB's cap at the motor level with the ×2 multiplier in `throttle_interpolator.py` on contest day.

## 2. EKF and AMCL Fought over `map → odom`

| Time | What happened |
|---|---|
| 08.07 | EKF `world_frame` switched from `odom` to `map` earlier that day, then a clean SLAM map was built. The switch most likely fixed the overlapping maps (see [Other Problems](#other-problems-along-the-way)) but started the conflict below *(estimate)* |
| 08.07 – 08.12 | The robot_localization example's `ekf_se_odom` had `world_frame: map` (upstream default is `odom`). With that setting the EKF publishes `map → odom`, **the same transform AMCL publishes** |
| 08.12 14:42 | Problems listed (Hyojeong Oh): odd local plan, `/odom` x-velocity jumping while stopped, `base_link` TF not following localization |
| 08.12 14:47 | Chaeyeon Kim: switch the frame back from `map` to `odom` |
| 08.12 15:28 | After the change: jitter gone, but localization still lost |
| 08.12 18:01 | Localization started following again without further changes — cause unknown |
| 08.12 18:53 | TF stable, but odometry stopped updating; around then Chaeyeon Kim saw in the rqt TF view that `base_link` was not connected *(Chaeyeon Kim's recollection)* |
| 08.13 12:49 | Hyojeong Oh's experiments: `vesc_to_odom/publish_tf: True` + `pub_odom.py` → no jitter, inaccurate · `True` + EKF → TF jitter · `False` + EKF → TF not connected at all |

Reading of the evidence:
- Two publishers of `map → odom` (EKF with `world_frame: map`, and AMCL) overwrote each other → `base_link` jitter.
- With the EKF on `map` and `vesc_to_odom` TF switched off, nobody published `odom → base_link` → odometry frozen, TF tree disconnected.
- The contest-day configuration was clean: `world_frame: odom`, only the EKF publishes `odom → base_link`, `vesc_to_odom` TF off.

## 3. Odometry Was Dead Reckoning from Commands

- `vesc_to_odom` integrates the motor speed from `/sensors/core` and **the steering command we sent** (`/sensors/servo_position_command`); it never measures the car's actual motion.
- `speed_to_erpm_gain` was 1000; MORAI's documented scale (300 per 1 km/h) implies about 1080, an ~8% speed error *(estimate)*.
- The x-velocity jumping while stopped, seen on 08.12, cannot be explained from the remaining records (no `/odom` recording exists).
- The 08.18 AMCL file shared before the run used `recovery_alpha_slow/fast = 0.001`, which practically disables random-particle recovery: once AMCL lost the car it could hardly recover.

## 4. Lane-Following Node Issues

The `lane_following` node (from 07.24 – 07.26) was ported from the team's SEA:ME hackathon code for a PiRacer car: `compute_pd_control` (Chaeyeon Kim, 07.17) and `plothistogram` / `sliding_window` (Sihyeon Park, 07.16). The port changed three lines in `compute_pd_control`, and its problems remain in the final code:

| Issue | Effect |
|---|---|
| `throttle = max(min(abs(pd), 0.25), 0.25)` | Speed is always 0.25; in the hackathon version the clamp was 0–1.0 and the PD term really scaled speed |
| Steering = deviation/180 clipped to ±0.4, minus 0.23 | Simple proportional steering; −0.23 is the **PiRacer's neutral steering trim**, carried over unchanged |
| Sends steering 0.0 when going straight | PiRacer steering is centred near 0, but MORAI's servo uses 0.5 as straight on a 0–1 range, so 0.0 is a full-lock command *(estimate)* |
| `sliding_window` returns a single value when only one lane is found | Unpacking error → node stops |
| 1 Hz loop plus `waitKey` delays | Far too slow for closed-loop driving |
| White mask only | The yellow-lane mask added on 07.26 did not survive into the final node |

Also on 07.26: the traffic-light stop attempt had a topic typo (`/GetTraficLightStatus`) and its stop command was overwritten by the main loop; the bird's-eye-view and calibration version was dropped the same evening, and the reason was not recorded.

## 5. If the Typo Had Been Fixed *(estimate)*

Fixing the TEB key alone mostly helps driving: top speed 0.4 → 2.0 m/s without the ×2 hack, a real turning radius (0.755 m) instead of in-place rotation, and a 2 m lookahead. The missions beyond driving depended on prototype code that teammates had written in Notion but never merged into the final launch.

| Mission | Prototype code (owner) | What would still have blocked it | Verdict |
|---|---|---|---|
| Maze exit | Final stack | Untested TEB values: `min_obstacle_dist` 0.6 (08.16 used 0.25) may block narrow corridors; maze-exit goal heading reversed (178°), hidden while TEB allowed in-place rotation | Likely in both runs |
| Delivery | `run_delivery_step` (Sihyeon Park) | Final version reads undefined `self.has_delivery` / `self.delivery_route` → crash; reach radius 2.0 m vs the rule's 1.2 m; target hard-coded as `unique_id == 50` | Possible after small fixes, never run in MORAI |
| Static obstacle | None — TEB only | Fixed 2 m lookahead on 1 m road waypoints may pull the car back into the blocked lane | Plausible |
| Dynamic obstacle (pedestrian) | LiDAR DBSCAN clustering → `/obstacle_class`, `/detected_car_vel` (Hyojeong Oh) | Detection only, no stop logic; versions that fed TEB `obstacles` would make the car avoid the pedestrian, which the rules count as a fail | Not without new code |
| Roundabout | None | No yielding or lane-keeping logic | Not without new code |
| Traffic light | Subscriber node (Hyojeong Oh, Sihyeon Park) | Topic `/getTrafficLightStatus` vs actual `/GetTrafficLightStatus`; stop-line position left at (0, 0); stops on status 16 (green); publishes `traffic_light_vel`, which nothing subscribes to | Not without 4 fixes |

Most likely outcome: out of the maze in both runs and past the static obstacle, then a stop or penalty at the first pedestrian or traffic light. The prototypes show the team had started every mission; what was missing was the integration step — a node that switches behaviour by mission and owns the final `cmd_vel`.

## Planned vs Built

| Item | Preliminary design (06.23–24) | Final run (08.18) | Why it changed |
|---|---|---|---|
| Middleware | Nav2-based (ROS2 assumed) | ROS1 Noetic | Rulebook: Ubuntu 20.04 / ROS Noetic, rosbridge |
| Simulator assumed | Gazebo / CARLA | MORAI | Revealed in July |
| Sensor focus | Cameras (upper: YOLO, lower: lanes) | 2D LiDAR | Mission 1 is a maze with walls; rulebook bans absolute position |
| Localization | SLAM with IMU + GPS | gmapping map + AMCL | GPS banned by the rules |
| Global planner | PRM | GlobalPlanner (Dijkstra) | Came with the lecture's navigation stack |
| Local planner | TEB (fallback DWA + Pure Pursuit) | TEB | Kept |
| Behavior | Behavior Tree | Maze → road switch script | No time for traffic light, delivery and obstacle logic |
| Perception | YOLO + OpenCV | Lane node only (not used in the final run) | YOLO never implemented |

## Results Against the Rules

| Mission (rulebook) | What the rules asked | What we did |
|---|---|---|
| 1. SLAM & Navigation | Own SLAM map, no manual localization, deliver 2 objects to 1 target (stay within 1.2 m for 2 s) | Map and AMCL worked; maze exited in run 2; delivery skipped |
| 2–3. Obstacles | Pedestrian: stop and wait (avoiding = fail); static: change lanes | Stopped at the first static obstacle (run 2) |
| 4. Roundabout | Enter without collision, stay off the centre line | Not reached |
| 5. Traffic light | Stop within 0.3 m of the line on red, go within 5 s on green | Not reached, no logic in the final stack |

The two-PC Ethernet setup (08.05 – 08.07) was not a team choice: the rulebook runs MORAI on the organizer's PC and each team's algorithm on its own laptop, wired together (fixed IPs 192.168.12.10 / .20).

## Other Defects

| Defect | Impact | Status on `main` |
|---|---|---|
| TEB YAML key `TebLocalPlannerRos` ≠ namespace `TebLocalPlannerROS` | TEB ran on defaults | Fixed |
| Quaternion `w`/`z` swapped in the maze-exit goal (`navigation_client.py`) | Goal heading 178°, opposite to the road direction (0°) | Fixed |
| Maze → Road switch condition uses `or` instead of `and` | Can switch early when only x or only y is close | Fixed |
| `use_dijkstra` outside the GlobalPlanner namespace, costmap `inflation_unknow` typo | Those settings were ignored (carried over from the lecture example) | Fixed |
| Road waypoints hard-coded to `~/ws/waypoint_1m.csv` | Only worked on the shared PC | Fixed (moved into `wego/waypoints/`) |
| Goal result never checked (ABORTED also advances) | A failed waypoint is silently skipped | Open |
| AMCL recovery effectively off (`recovery_alpha_* = 0.001`) | No self-recovery after losing the pose | Open |
| Lane node issues (section 4) | Lane following unusable | Open |
| Laptop performance | move_base control loop ~2 Hz versus a 10–20 Hz target | Hardware |

Fixes on `main` are in one commit ("Fix bugs found in the post-contest analysis"), checked for syntax and parameter placement only — **not driven in MORAI**.

## Other Problems Along the Way

| When | Problem | Outcome |
|---|---|---|
| 07.20 – 07.23 | MORAI crashed in a VM (no GPU access) | Ran MORAI on Windows + ROS in WSL2 |
| 08.03 – 08.04 | First SLAM maps overlapped (a second map under the first), gmapping kept exiting | *(estimate)* With `ekf=true`, both the EKF (`world_frame: odom`) and `pub_odom.py` published `odom → base_link`, so gmapping's odometry jumped between two estimates. Switching the EKF to `world_frame: map` on 08.07 left `pub_odom.py` as the only publisher → clean map, and the EKF's new `map → odom` became problem 2 |
| 08.06 | Two PCs connected over Ethernet but `rostopic list` empty on the ROS PC | Worked on 08.07; fix not recorded |
| 08.13 | Lanes as costmap obstacles block multiple and crossing lanes | Abandoned; waypoints (the original plan) kept, via-points designed but not integrated |
| 08.16 | Sudden costmap change on the shared PC | Cause not recorded; stalls later traced to the planner, not the map |

## Parameter Tuning History

| Parameter | 08.11 | 08.13 | 08.16 | Final (08.18) |
|---|---|---|---|---|
| common `inflation_radius` / `cost_scaling_factor` | 0.4 / 10 | 0.25 / 13 | 0.15 / 3 | 0.15 / 10 |
| global `inflation_radius` | 0.2 | 0.5 | 0.15 | 0.15 |
| obstacle layer | Voxel | Voxel | Voxel | **ObstacleLayer** |
| TEB `max_vel_x` | 2.0 | 2.0 | 2.0 | 2.0 |
| TEB `min_obstacle_dist` | 0.4 | 0.6 | 0.25 | 0.6 |
| TEB `weight_kinematics_nh` | 1000 | 1000 | 1000 | 200 |
| `controller_frequency` / `patience` | 10 / 7 | 10 / 7 | 5 / 7 | 20 / 2 |
| AMCL particles (min/max) | 900/2000 | 800/2500 | 800/2500 | 500/2000 |

- Values are from the files in each snapshot. TEB rows were never applied (section 1), except values changed live in rqt_reconfigure.
- On 08.13 Sihyeon Park said lowering `cost_scaling_factor` helped, while the 08.13 snapshot file shows 13 (up from 10); the improvement may have come from an rqt change or a different file.

## Lessons

- When tuning does nothing, check what the node actually reads (`rosparam get`, rqt) before changing more values.
- One transform, one publisher: list who publishes each TF edge before adding an EKF.
- Log raw topics (`/odom`, `/tf`, `/sensors/core`) during experiments; without recordings the odometry noise could not be explained afterwards.
