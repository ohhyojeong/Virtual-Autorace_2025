# Timeline

[← README](../README.md) · **English** | [한국어](timeline_ko.md)

## Before the Contest (2025)

| Date | Event | Notes |
|---|---|---|
| 05.21 | Yoonju Jeong found the contest and proposed entering; all agreed | Simulator assumed to be CARLA at the time |
| 06.23 | Notice: preliminary presentation the next day at 11:30; roles split for the slides (Chaeyeon Kim control/path following, Sihyeon Park sensors/perception, Yoonju Jeong planning, Hyojeong Oh behavior/risk handling) | Sihyeon Park posted architecture drafts v1 and v2 |
| 06.24 | Tech stack, plan, slides and script finished; presentation at 11:30 | Outcome not recorded; the team later competed in the final |
| 07.07 | Pre-training started; MORAI accounts set up (2 team accounts) | Training looked ROS1-based |
| 07.08 | Plan: run MORAI on own laptops, dual-boot on a spare PC as fallback | Never needed — solved with WSL2 on 07.22–23 |
| 07.16 – 07.17 | Lane-detection code written for the SEA:ME hackathon (sliding window, PD function) | Reused in the contest lane node on 07.26 |

## Development Timeline (2025)

| Date | Work | Notes |
|---|---|---|
| 07.20 | MORAI crashes in a VM (no GPU access) | Dual-boot looked necessary at first |
| 07.21 – 07.22 | First meetings; MORAI sensors checked; two-PC idea proposed | |
| 07.22 – 07.23 | **MORAI on Windows + ROS Noetic in WSL2** works; roles assigned | Became the team standard; this laptop = shared PC |
| 07.24 | First lane node (raw camera → `/commands/vel`); missing camera node found | rosbridge provides the camera topic |
| 07.26 | Three separate lane-node versions: MORAI port, traffic-light attempt, BEV + calibration; right-lane bug fixed 17:05 | Only the MORAI port survived |
| 07.29 – 07.31 | Offline training on MORAI Cloud | Instructor suggested camera-based driving |
| 07.30 | Perception direction: HOG example tested, YOLO figures looked up | Deep learning chosen, not implemented |
| 08.03 – 08.04 | WeGo packages, static TF, `pub_odom`; first SLAM maps distorted, gmapping exiting | LiDAR TF check suggested 08.05 |
| 08.05 – 08.06 | Two-PC link connects but no topics | |
| 08.07 | Two-PC link works; SLAM map built (world frame `map`) | |
| 08.07 – 08.10 | Navigation packages and waypoint pipeline on Sihyeon Park's laptop; global planner design note (08.10) | Branch `history/sihyeon-laptop-2025-08` |
| 08.11 | Moved to a borrowed laptop; still slow → performance hypothesis dropped; first maze exit (maze-only goals) | `max_vel_x` 2 → 6 had no effect |
| 08.12 | Local plan odd, `/odom` jumping, `base_link` jitter → frame switched `map` → `odom` (15:28) → tracking back (18:01) → odometry stopped updating | Speed still low |
| 08.13 | Odometry experiments; `cost_scaling_factor` lowered (slight improvement); lanes-in-costmap judged infeasible | |
| 08.14 – 08.16 | Waypoint planner research; via-points design; sudden costmap change on the shared PC (08.16) | Stalls traced to the planner, not the map (08.16) |
| 08.16 – 08.17 | Stack moved back to the shared PC; TEB built from source; maze → road client; `max_vel_x` 100 tried | |
| 08.18 | AMCL/EKF tuning (reverted on the shared PC), **final map**, ×2 motor multiplier, final run | |

## Contest Day (8/18) Reconstruction

| Time | Event | Source |
|---|---|---|
| 01:19–03:44 | ~30 AMCL edits and EKF tuning | VS Code |
| 09:18 | 03:36 AMCL tuned version shared with the team (tuning by Sihyeon Park). At the same time the shared PC's AMCL/EKF files were reverted to the 8/17 values | Team chat, VS Code |
| 10:03–10:16 | Map rebuilt with gmapping (Chaeyeon Kim), saved 10:16:08 → path-planning failures dropped sharply afterwards | ROS logs, Chaeyeon Kim's recollection |
| 10:26 | Chaeyeon Kim in the contest waiting room | Team chat |
| 12:14–12:16 | Chaeyeon Kim re-recorded the road waypoints (`amcl_waypoint_today2.csv`, from `/amcl_pose`). The shared-PC logs show no driving stack at that time, so it appears to have been recorded on another PC | CSV timestamps, confirmed by Chaeyeon Kim |
| 14:26–15:35 | navigation_client experiments (publishing the path directly, changing the switch point) → reverted with undo | VS Code |
| **15:34:56** | **Sihyeon Park uncommented `input_rpm = msg.data * 2.0` in `throttle_interpolator.py`** | VS Code, confirmed by Chaeyeon Kim |
| 15:38:32 | Run with ×2 → 15:41:00 GOAL Reached at the maze exit → oscillation / Aborting / NO PATH | ROS logs |
| 15:40 | ×3, controller 20 Hz / patience 2 | VS Code |
| 15:42:19 | Re-run with ×3 → 15:44:26 GOAL Reached | ROS logs |
| 15:42:51 | Saved ×4 (no run recorded on this PC afterwards) | VS Code |

The records cannot pin down exactly which runs were the official 1st and 2nd attempts.

**Operators confirmed**: the 15:34 ×2 uncomment and ×3/×4 changes were Sihyeon Park's; the 12:14 waypoint re-recording was Chaeyeon Kim's (confirmed by Chaeyeon Kim). The operator of the 14:26–15:35 navigation_client experiment (publishing a Path directly to `/move_base/GlobalPlanner/plan`, using today2.csv, switch point 15.70/−9.51 → reverted with undo at 15:35) is classified as **joint team work** (operator cannot be identified). That experimental code matches the code at the top of the Notion main page.

None of these items are mentioned in the chat.
