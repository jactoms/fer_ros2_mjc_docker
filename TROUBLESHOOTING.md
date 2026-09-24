# Session Troubleshooting Log — fer_ros2_mjc_docker

Record of everything found and fixed while getting the Docker + MuJoCo
simulation running per the top-level `README.md`. Kept as a reference for
future setup issues on this or similar hosts.

## Summary

| # | Problem | Blocking? | Fix location | Persists across container restart? |
|---|---|---|---|---|
| 1 | CycloneDDS multicast discovery fails on this host | Yes — hangs both launch files forever | `env/cyclone_dds.xml` | Yes (host file, bind-mounted) |
| 2 | `pycollada` missing → MJCF conversion crashes | Yes — hangs both launch files, spawners time out | `docker/Dockerfile` | Yes — image rebuilt and re-verified |
| 3 | No realtime (`SCHED_FIFO`) scheduling in container | No | not fixed | N/A |
| 4 | Duplicate `/moveit_rviz2` node name | No (cosmetic) | not fixed (upstream MoveIt RViz quirk) | N/A |
| 5 | "Unknown tag/attribute" warnings during URDF→MJCF conversion | No (expected) | none needed | N/A |
| 6 | "Jump back in time" TF warnings at startup | No (transient) | none needed | N/A |
| 7 | DDS discovery fix (#1) stopped working intermittently | Yes | `env/cyclone_dds.xml` | Yes (host file, bind-mounted) |
| 8 | First launch after a fresh container is much slower | No (by design) | none needed — informational | N/A |
| 9 | Loopback-pinned DDS profile (#1/#7) doesn't suit WSL2 / Docker Desktop hosts | Yes, on WSL2 | `env/cyclone_dds_wsl.xml`, `docker/run_container.sh` | Yes (host files; profile picked at container start) |

Fixes #1, #2, and #7 are committed on the `adaptive_handover_skill` branch
(commit `5fdb37f`). The WSL2 profile (#9) is in the next commit, `f9d12ae`.

---

## 1. CycloneDDS multicast discovery fails on this host

**Symptom:** Both `fer_mujoco_ros2_control.launch.py` and
`fer_mujoco_moveit.launch.py` hung indefinitely. `ros2_control_node` printed
`Waiting for data on 'robot_description' topic to finish initialization`
forever, and controller spawners timed out with
`Could not contact service /controller_manager/list_controllers`. All the
expected processes (`robot_state_publisher`, `ros2_control_node`, etc.) were
alive and healthy — they just never discovered each other.

**Root cause:** ROS 2 / CycloneDDS uses multicast for node discovery (SPDP)
by default. On this host, the loopback interface's flags
(`/sys/class/net/lo/flags` = `0x9`) show `IFF_MULTICAST` is **not** set —
confirmed directly with `ros2 multicast send` / `ros2 multicast receive`,
which never exchanged a single datagram. Since the container runs with
`--net host`, all ROS 2 nodes — even ones in the *same* container — rely on
that broken loopback multicast path, so discovery silently never completes.
This is a host networking quirk, not a bug in the repo.

**Fix:** Added a unicast discovery fallback to `env/cyclone_dds.xml`:

```xml
<Discovery>
    <Peers>
        <Peer address="localhost"/>
    </Peers>
</Discovery>
```

This tells CycloneDDS to also try unicast SPDP against `localhost`,
independent of whether multicast works. Verified with a plain
`ros2 topic pub` / `ros2 topic echo` round trip, then with a full launch —
all nodes appeared in `ros2 node list` within seconds.

**Alternative (not applied):** `sudo ip link set lo multicast on` would fix
it at the OS level, but doesn't persist across reboots without a netplan/
systemd change, needs a password we can't supply non-interactively, and
wouldn't help if this repo is ever run on a different host with the same
issue. The XML fix travels with the repo and works regardless of host
config, so it's the one that was kept.

**Persistence:** `env/` is bind-mounted into the container
(`docker/run_container.sh`), so this fix is live immediately — no image
rebuild needed, and it survives closing/restarting the container. On native
Linux hosts `run_container.sh` points `CYCLONEDDS_URI` at this file
(see issue #9 for the WSL2 exception).

**Update:** this fix alone turned out to be incomplete — see issue #7 below,
which adds the missing piece (pinning the network interface).

---

## 2. Missing `pycollada` dependency breaks URDF→MJCF conversion

**Symptom:** Worked in the first container of the session, then **stopped
working after the user closed the container and started a fresh one from
the same image** — same hang symptoms as issue #1 (`Waiting for data on
'robot_description'...`), even though the DDS fix above was already in
place and confirmed via `ros2 node list`/`ros2 multicast`.

**Root cause:** `mujoco_ros2_control`'s `robot_description_to_mjcf.sh`
lazily creates a Python virtualenv on first run, at
`~/.ros/ros2_control/.venv` (`--system-site-packages`), and pip-installs
`trimesh`, `mujoco`, `obj2mjcf` into it — but **not** `pycollada`, which
`trimesh` needs to parse the `.dae` (Collada) mesh files that
`franka_description` ships. The script crashed with:

```
ImportError: missing `pip install pycollada`
```

Critically, `~/.ros` is **not** one of the bind-mounted host directories in
`docker/run_container.sh` (only `ros2_ws`, `env`, `data`, and
`.claude_container` are mounted). So the very first container of this
session must have had `pycollada` installed into that venv by an earlier,
unrecorded interactive session — that fix lived only inside that specific
container's writable layer. The moment the user closed it
(`run_container.sh` uses `--rm`, so the container is deleted on exit) and
started a new one, that ephemeral fix vanished and the underlying,
never-actually-fixed image bug reappeared.

**Fix (applied twice):**
1. **Immediate unblock** (session-only, evaporates again on next container
   restart): installed directly into the running container's venv —
   ```bash
   source ~/.ros/ros2_control/.venv/bin/activate
   pip3 install --no-input --no-cache-dir --disable-pip-version-check pycollada
   ```
2. **Durable fix**: added to `docker/Dockerfile` (system-wide, so the
   `--system-site-packages` venv inherits it automatically on every fresh
   container, with no runtime pip install needed at all):
   ```dockerfile
   RUN pip3 install --no-input --no-cache-dir --break-system-packages pycollada
   ```
   (`--break-system-packages` is required on Ubuntu 24.04/Python 3.12,
   which this image is built on — confirmed via `pip3 --version` and
   `/etc/os-release` inside the container.)

**Persistence:** The Dockerfile change required an image rebuild
(`./docker/build_image.sh`) to take effect. The rebuild was run during this
session (~92s for the `colcon build` step; layer caching made the rest
fast) and re-verified: a brand-new container, created fresh from the
rebuilt image with zero manual steps, has `pycollada` importable
immediately (`python3 -c "import collada"` succeeds with no prior pip
install), and a full launch completes with no `ImportError`.

---

## 3. No realtime (`SCHED_FIFO`) scheduling in the container

**Symptom:** `controller_manager` logs on every launch:
```
Could not enable FIFO RT scheduling policy: with error number <1>(Operation not permitted)
```

**Root cause:** `ulimit -r` inside the container is `0` and `CapEff` is
`0000000000000000` — `docker/run_container.sh` passes `--privileged` but
never raises the realtime-priority ulimit (`--ulimit rtprio=...`) or adds
`--cap-add=sys_nice`, so `--privileged` alone doesn't grant `SCHED_FIFO`.

**Impact:** None for this pure-simulation use case — the control loop was
verified to work correctly (see below) at normal OS scheduling priority.
Per `DDS_Profiles.md`'s own framing, this specifically matters when driving
**real Franka hardware**, where missing a 1ms control window trips the
arm's safety lock. Not fixed, since it doesn't affect the simulation and
fixing it is only relevant if this setup is later pointed at real hardware.

---

## 4. Duplicate `/moveit_rviz2` node name

**Symptom:** `ros2 node list` (with the MoveIt launch file) shows
`/moveit_rviz2` twice, and `ros2` warns: *"Be aware that there are nodes in
the graph that share an exact name."*

**Root cause:** Verified there is only **one** `rviz2` process running (not
a leftover from testing) — the MoveIt RViz `MotionPlanningDisplay` plugin
internally spins up a second `rclcpp::Node` reusing its parent's node name.
This is a known, cosmetic upstream MoveIt/RViz behavior, unrelated to
anything in this repo or to the DDS fix.

**Fix:** None needed.

---

## 5. "Unknown tag/attribute" warnings during URDF→MJCF conversion

**Symptom:** Dozens of lines like:
```
Unknown attribute "gear_ratio" in .../joint[@name='fer_joint1']/dynamics
Unknown tag "mujoco_inputs" in /robot[@name='fer']
```

**Root cause:** These come from a generic URDF-schema validator flagging
custom, MuJoCo-specific extension tags/attributes (`mujoco_inputs`,
`ros2_control`, `dynamics/gear_ratio`, `dynamics/K`, etc.) that
`fer_ros2_mjc_bringup`'s Xacro wrapper injects on purpose. A second parsing
pass explicitly consumes those same tags to build the MuJoCo model
(confirmed in the log: `Parsing MuJoCo elements from...`), and the
resulting model loaded with the correct body/geom counts and rendered/
controlled correctly.

**Fix:** None needed — expected, by-design warnings.

---

## 6. "Jump back in time" TF warnings at startup

**Symptom:** A burst of `rviz2`/`tf2_buffer` warnings about detecting a
backwards time jump and resetting, right around when controllers activate.

**Root cause:** Not fully isolated. Did not recur during later clean
launches or during a live trajectory-execution test, so it looks like a
one-time startup transient (e.g. sim-time settling as `robot_state_publisher`
republishes joint transforms once controllers take over) rather than a
persistent problem.

**Fix:** None applied — flagged as the one item not fully root-caused,
should it reappear consistently in the future.

---

## 7. DDS discovery fix (#1) stopped working intermittently

**Symptom:** After confirming issue #1's fix worked, a *later* container —
including a completely fresh one created from scratch, with zero prior ROS
activity — went right back to `ros2 topic pub`/`ros2 topic echo` never
exchanging a message, and full launches hanging exactly like issue #1
again. This happened even though `env/cyclone_dds.xml` still had the
`<Peer address="localhost"/>` entry and was confirmed correctly mounted and
loaded (`$CYCLONEDDS_URI` and the file contents were checked directly
inside the affected container).

**Root cause:** The original fix only told CycloneDDS *who* to unicast
discovery packets to (`localhost`) — it never told it *which network
interface* to send that traffic through. Without an explicit pin,
CycloneDDS's interface autoselection is free to choose the host's WiFi
interface (`wlp1s0`, the default-route interface) instead of `lo`, and nothing
guarantees that choice stays stable across the session (WiFi
reconnects, DHCP renewals, etc. can all change what autoselection prefers).
When it picks the wrong interface, the `localhost` unicast hint gets sent
out a path that never reaches the loopback-bound peer, and discovery fails
exactly as if the Peers fix were never applied — even though the config
file is untouched and correct.

**Fix:** Pinned the interface explicitly in `env/cyclone_dds.xml`:
```xml
<General>
    <Interfaces>
        <NetworkInterface name="lo"/>
    </Interfaces>
</General>
```
Justified since the container always runs with `--net host` — there is no
scenario where this setup needs to reach another machine, so binding
exclusively to loopback is always correct here, not just a workaround.

**Verification:** Removed the affected container entirely and created a
brand-new one. Immediately after boot (before anything else could pollute
the DDS graph), a `ros2 topic pub`/`echo` round trip succeeded on the first
try, with the log explicitly confirming the mechanism:
```
selected interface "lo" is not multicast-capable: disabling multicast
data: hello3
```
Followed by a full `fer_mujoco_ros2_control.launch.py` run completing
cleanly (`Model body count: 10`, all expected controllers active, no
errors) in that same fresh container.

**Persistence:** Same file as issue #1 (`env/cyclone_dds.xml`, bind-mounted)
— survives container restarts with no image rebuild needed.

---

## 8. First launch after a fresh container is much slower (and can look like a crash)

**Symptom:** On a container that has never run the simulation before, the
first `ros2 launch` can take 60–90+ seconds before `ros2_control_node` and
the MJCF-conversion script show any progress, both sitting blocked on a
futex the whole time. Once during this session, that delay was long enough
that `ros2_control_node` hit its own internal wait-timeout for
`/mujoco_robot_description`, logged `Timeout waiting for
/mujoco_robot_description topic. Aborting...`, and then crashed with a
`corrupted double-linked list` glibc heap-corruption abort — a real bug in
how that abort path cleans up, but only reachable by losing this race.

**Root cause:** Same underlying mechanism as issue #2 —
`robot_description_to_mjcf.sh` lazily builds `~/.ros/ros2_control/.venv`
and pip-installs `trimesh`, `mujoco`, `obj2mjcf` into it on first use, and
`~/.ros` isn't persisted across containers. Even with `pycollada` now baked
into the image, those three packages still have to be downloaded and
installed fresh in every new container the first time the launch file
runs. That install can occasionally take long enough to lose the race
against `ros2_control_node`'s fixed timeout, triggering the crash above.
On a second attempt in the *same* container (venv already warm), the same
launch completed normally.

**Fix:** None applied — this is inherent to the upstream lazy-install
design, not something introduced by this repo. If it recurs and is
disruptive, the options are: (a) bake `trimesh`, `mujoco`, and `obj2mjcf`
into the Dockerfile as well (same pattern as the `pycollada` fix, trades a
larger image for eliminating this delay entirely), or (b) bind-mount
`~/.ros/ros2_control/.venv` from the host so it's built once and reused
across containers. Neither was applied since the crash was a one-time
occurrence tied to a timing race, not a guaranteed failure.

**Workaround if hit:** kill the launch, wait for the pip install to finish
in the background (check `~/.ros/ros2_control/.venv` for `trimesh`,
`mujoco`, `obj2mjcf`), and relaunch — the second attempt won't pay the
install cost again for that container's lifetime.

---

## 9. Loopback-pinned DDS profile doesn't suit WSL2 / Docker Desktop hosts

**Context:** Added on the `adaptive_handover_skill` branch (commit
`f9d12ae`) so the same repo also runs under WSL2 with Docker Desktop, and
not only on the native Linux host the fixes above were developed on.

**Problem:** The fixes for #1 and #7 tune `env/cyclone_dds.xml` for one
specific native Linux host: discovery is forced onto `lo` with a
`localhost` unicast peer, because that host's loopback has no multicast.
Under WSL2 + Docker Desktop, networking goes through a virtual `eth0`
interface instead, so a profile pinned to `lo` is the wrong choice there.
Before this change there was also no way to choose a different profile per
host: the Dockerfile hard-codes
`ENV CYCLONEDDS_URI=file:///home/${USER}/env/cyclone_dds.xml`.

**Fix:**
1. New profile `env/cyclone_dds_wsl.xml`. It uses the same buffer, message
   size, fragment size, and shared-memory settings as the native profile,
   but pins discovery to `eth0` with multicast enabled and has no
   `localhost` peer:
   ```xml
   <Interfaces>
       <NetworkInterface name="eth0" multicast="true"/>
   </Interfaces>
   ```
2. `docker/run_container.sh` now picks the profile when the container
   starts and passes it in, overriding the Dockerfile's default:
   ```bash
   CYCLONEDDS_CONFIG="cyclone_dds.xml"
   if grep -qi microsoft /proc/version; then
       CYCLONEDDS_CONFIG="cyclone_dds_wsl.xml"
   fi
   # ...
   -e CYCLONEDDS_URI=file:///home/${CONTAINER_USER}/env/${CYCLONEDDS_CONFIG}
   ```
   On native Linux nothing changes: it still uses `cyclone_dds.xml` with
   fixes #1 and #7.

**Persistence:** Both files live on the host (`env/` is bind-mounted), so
no image rebuild is needed. The profile is chosen each time the container
starts. Only containers started through `run_container.sh` get this choice;
a container started any other way falls back to the Dockerfile's
`cyclone_dds.xml`.

**Verification status:** Not verified on a WSL2 host during these
sessions. If discovery hangs there (same symptoms as #1), check inside the
container:
- `echo $CYCLONEDDS_URI` should end in `cyclone_dds_wsl.xml`.
- `ip link` should list an interface called `eth0`. If it has a different
  name, update the `NetworkInterface` entry.

After that, the `ros2 topic pub` / `ros2 topic echo` round trip from #1 is
a quick end-to-end test.

---

## Functional verification performed

- `fer_mujoco_ros2_control.launch.py`: MuJoCo window opens, `joint_state_broadcaster`,
  `joint_effort_traj_controller`, and `gripper_position_controller` go
  active — matching the README's description exactly.
- `fer_mujoco_moveit.launch.py`: `move_group`, planning RViz, and the MuJoCo
  render all came up together with full node discovery.
- Sent a real `FollowJointTrajectory` goal to
  `/joint_effort_traj_controller/follow_joint_trajectory` (moved
  `fer_joint1` to 1.0 rad) — action returned `SUCCEEDED`, and
  `/joint_states` plus a screenshot confirmed MuJoCo's physics actually
  moved the arm to the commanded pose (not just a controller reporting
  success).
- Re-tested from a **fresh** container (closed and restarted) to catch
  exactly the regression the user hit — reproduced issue #2 above, fixed it,
  and re-verified the launch succeeds end-to-end again.
- Rebuilt the Docker image (`./docker/build_image.sh`) with the `pycollada`
  fix baked in, then created a **brand-new container from scratch** (no
  prior state at all) and confirmed: `pycollada` importable with zero
  manual steps, DDS discovery working via the interface-pinned config
  (issue #7), and a full `fer_mujoco_ros2_control.launch.py` launch
  completing cleanly end-to-end (`Model body count: 10`, all expected
  controllers active, screenshot confirming MuJoCo rendered correctly).

## Current state

Both fixes (`env/cyclone_dds.xml`, `docker/Dockerfile`) are in place and
committed on the `adaptive_handover_skill` branch, the image has been rebuilt with the `pycollada` fix included, and a genuinely
fresh container was verified end-to-end after the rebuild. No outstanding
manual steps remain for a normal `./docker/run_container.sh` →
`ros2 launch ...` workflow. The only residual caveat is issue #8 above
(slower, occasionally crash-prone first launch per fresh container) —
informational only, not something requiring action unless it becomes
disruptive. The WSL2 profile (issue #9) is in place for WSL2 hosts but has
not been tested on one yet.
