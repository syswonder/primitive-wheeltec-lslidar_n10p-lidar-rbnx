# N10P LSLIDAR — Robonix Primitive

LSLIDAR N10P 2D laser scanner primitive for Robonix. Owns the
`robonix/primitive/lidar/*` capability namespace.

## Overview

| Field | Value |
|-------|-------|
| Package | `robonix.primitive.n10p_lslidar` |
| Version | 0.1.0 |
| Hardware | LSLIDAR N10P (2D laser scanner) |
| ROS 2 distro | Humble |
| License | Apache-2.0 |

## Capabilities

| Capability | Transport | Topic | QoS |
|------------|-----------|-------|-----|
| `robonix/primitive/lidar/driver` | gRPC | — | — |
| `robonix/primitive/lidar/lidar` | topic_out | `/scan` | reliable |
| `robonix/primitive/lidar/lidar_snapshot` | MCP | — | one-shot LaserScan |

Publishes `sensor_msgs/LaserScan` with `frame_id=laser`. The `base_link → laser`
TF edge comes from the URDF via `robot_state_publisher` — the lidar primitive
itself does not publish it (unless `extrinsics` is explicitly configured).

## Configuration

All values below live under this package's `config:` key in
`robonix_manifest.yaml`.

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| `scan_topic` | string | `/scan` | LaserScan topic published by the lidar driver. The sentinel waits for the first message on this topic before declaring ready. |
| `frame_id` | string | `laser` | TF frame_id written in the LaserScan header. Must match the frame name in the physical URDF. |
| `parent_frame` | string | `base_link` | Parent frame for optional `static_transform_publisher`. Only used when `extrinsics` is provided. |
| `extrinsics` | dict | `null` | Optional 6-DoF mount pose `{x, y, z, roll, pitch, yaw}` (metres, radians). When present, a `static_transform_publisher` is spawned (`parent_frame → frame_id`). Omit when the chassis primitive or soma URDF already publishes this edge — double-publishing causes `TF_REPEATED_DATA`. |
| `launch_package` | string | `turn_on_wheeltec_robot` | ROS 2 package containing the lidar launch file. |
| `launch_file` | string | `wheeltec_lidar.launch.py` | Launch file inside `launch_package`. |
| `sentinel_timeout_s` | float | `30.0` | Maximum wait for the first LaserScan before failing init. N10P typically starts streaming within 5–10 seconds. |
| `retries` | int | `3` | Number of respawn attempts if the scan stream doesn't appear. |

### Example manifest config

```yaml
# In robonix_manifest.yaml:
primitives:
  - name: lidar
    path: ./n10p_lslidar
    config:
      scan_topic: /scan
      frame_id: laser
      sentinel_timeout_s: 30.0
      retries: 3
      # extrinsics:                         # omit — URDF publishes this edge
      #   x: 0.12
      #   y: 0.0
      #   z: 0.25
      #   roll: 0.0
      #   pitch: 0.0
      #   yaw: 0.0
```

## Lifecycle

```
REGISTERED → INACTIVE → ACTIVE
                │
                ▼
           SHUTTING_DOWN
```

1. **on_init** — parse config → spawn `ros2 launch wheeltec_lidar.launch.py` →
   sentinel waits for first LaserScan on `scan_topic` → optionally spawn
   `static_transform_publisher` → declare `topic_out` capabilities.
2. **on_shutdown** — kill static_transform_publisher → kill lslidar subprocess
   (SIGTERM, then SIGKILL after 5s timeout).

## Self-heal

The N10P may fail to start its data stream on rare occasions (hardware
initialisation race). The primitive detects this — no `LaserScan` within
`sentinel_timeout_s` — and respawns the driver up to `retries` times:

```
spawn lslidar
    │
    ▼
wait for first LaserScan (sentinel_timeout_s)
    │
    ├── seen ──► declare capabilities, init complete
    │
    └── timeout ──► kill lslidar, respawn (attempt + 1)
                         │
                         └── after retries exhausted → return Err
```

This lets a remote deploy recover without manually power-cycling the lidar.

## Dependencies

- **ROS 2**: sourced at `/opt/ros/humble/setup.bash`
- **`turn_on_wheeltec_robot`** workspace on the ROS package path (provides
  `wheeltec_lidar.launch.py`)
- **`robot_description`** primitive (or equivalent URDF) — publishes the
  `base_link → laser` TF edge so downstream consumers see a complete TF tree
- **`rbnx`** CLI on PATH for build and codegen
- Python 3.10+ with `rclpy`, `sensor_msgs`, `robonix_api`

## TF integration

```
base_footprint
    └── base_link                          (chassis primitive)
            └── laser                      (URDF via robot_state_publisher)
```

The lidar primitive itself does **not** publish the `base_link → laser` TF
edge — the URDF model from `robot_description` handles it. Set `extrinsics`
only if you're running without a URDF or need to override the mount pose.

## Package structure

```
n10p_lslidar/
├── README.md
├── package_manifest.yaml      # Robonix manifest (build, start, capabilities)
├── config.spec                # Config schema documentation for deployers
├── capabilities/              # Capability descriptors (auto-generated)
├── scripts/
│   ├── build.sh               # rbnx codegen + colcon build
│   └── start.sh               # Source ROS 2 + overlays, launch main.py
└── n10p_lslidar/
    ├── __init__.py
    └── main.py                # Primitive lifecycle: init → sentinel → shutdown
```

## Build & run

```bash
# Build (codegen + colcon)
rbnx build -p ./n10p_lslidar

# Or directly:
bash scripts/build.sh

# Run (standalone, outside rbnx boot):
bash scripts/start.sh
```
