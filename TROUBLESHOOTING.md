# Troubleshooting

Known issues when running the Docker + MuJoCo simulation, and how they are fixed.

| # | Problem | Fixed in |
|---|---|---|
| 1 | Nodes can't discover each other (launch hangs) | `env/cyclone_dds.xml` |
| 2 | `pycollada` missing, so MJCF conversion crashes | `docker/Dockerfile` |
| 3 | WSL2 / Docker Desktop needs a different DDS profile | `env/cyclone_dds_wsl.xml`, `docker/run_container.sh` |
| 4 | First launch in a new container is slow | not fixed (workaround below) |
| 5 | Harmless warnings | nothing to fix |

---

## 1. Nodes can't discover each other (launch hangs)

**Symptom:** Launch never finishes. `ros2_control_node` keeps printing
`Waiting for data on 'robot_description' topic`, and the spawners time out
with `Could not contact service /controller_manager/list_controllers`.

**Cause:** CycloneDDS finds other nodes using multicast. On some hosts the
loopback interface (`lo`) doesn't support multicast. When that happens,
CycloneDDS may also pick the wrong network interface (for example WiFi).

**Fix:** `env/cyclone_dds.xml` pins DDS to `lo` and finds nodes by direct
(unicast) messages to `localhost`:

```xml
<General>
    <Interfaces>
        <NetworkInterface name="lo"/>
    </Interfaces>
</General>
<Discovery>
    <Peers>
        <Peer address="localhost"/>
    </Peers>
</Discovery>
```

This works because the container runs with `--net host`. `env/` is
bind-mounted, so no rebuild is needed.

**Quick check:** run `ros2 topic pub /test std_msgs/String "data: hi"` in one
terminal and `ros2 topic echo /test` in another. You should see the message.

---

## 2. `pycollada` missing, so MJCF conversion crashes

**Symptom:** Same hang as #1, and the log shows
`ImportError: missing pip install pycollada`.

**Cause:** `mujoco_ros2_control` converts the URDF to MuJoCo format (MJCF),
and reading the `.dae` meshes needs `pycollada`. The Python venv it uses
doesn't install `pycollada` and gets recreated in every new container.

**Fix:** `pycollada` is installed in the image (`docker/Dockerfile`):

```dockerfile
RUN pip3 install --no-input --no-cache-dir --break-system-packages pycollada
```

Rebuild the image once with `./docker/build_image.sh`.

---

## 3. WSL2 / Docker Desktop needs a different DDS profile

The `lo` setup from #1 doesn't fit WSL2, where the network runs over `eth0`.
When the container starts, `docker/run_container.sh` checks whether it's
running on WSL2 and picks the profile:

- **Native Linux:** `env/cyclone_dds.xml`
- **WSL2:** `env/cyclone_dds_wsl.xml` (uses `eth0` with multicast)

The choice is passed into the container as `CYCLONEDDS_URI`.

**Not yet tested on WSL2.** If it hangs there:

- Check that `echo $CYCLONEDDS_URI` ends in `cyclone_dds_wsl.xml`.
- Check that `ip link` shows an interface named `eth0`. If it has a
  different name, update the `NetworkInterface` entry in `env/cyclone_dds_wsl.xml`.

---

## 4. First launch in a new container is slow

The first launch in a new container installs `trimesh`, `mujoco` and
`obj2mjcf` into a Python venv. That takes 60–90 s. It can occasionally time
out with `Timeout waiting for /mujoco_robot_description topic` followed by a
crash.

**Workaround:** wait for the install to finish, then launch again. Later
launches in the same container are fast.

To avoid it completely, install those three packages in the Dockerfile too.

---

## 5. Harmless warnings

- **`Could not enable FIFO RT scheduling policy`:** the container has no
  realtime scheduling. This doesn't matter in simulation, only on real Franka
  hardware (fix: `--ulimit rtprio=99 --cap-add=sys_nice`).
- **`Unknown tag/attribute ...` during URDF→MJCF conversion:** these are
  MuJoCo-specific tags that the converter reads in a second pass. Expected.
- **`/moveit_rviz2` appears twice in `ros2 node list`:** a known MoveIt RViz
  quirk.
- **`Jump back in time` TF warnings at startup:** these appear once while
  sim time settles, then stop.
