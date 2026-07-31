---
layout: doc
title: Setup Guide — Isaac Lab-Arena (Docker)
description: >
  This chapter covers advanced topics, such as offline support and custom JS builds. Codings skills are recommended.
hide_description: true
---

This chapter covers advanced topics, such as offline support and custom JS builds. Codings skills are recommended.


# Setup Guide — Isaac Lab-Arena (Docker)

## Prerequisites

- Linux (Ubuntu 22.04+)
- NVIDIA GPU with [supported driver](https://docs.isaacsim.omniverse.nvidia.com/6.0.0/installation/requirements.html)
- Docker + [NVIDIA Container Toolkit](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/install-guide.html)
- Git

## 1. Clone with submodules

```bash
git clone git@github.com:isaac-sim/IsaacLab-Arena.git
cd IsaacLab-Arena
git submodule update --init --recursive
```

## 2. Start the container

```bash
./docker/run_docker.sh
```

This builds the Docker image (`isaaclab_arena:latest`) on first run, then starts and attaches to the container. Idempotent — subsequent runs just attach.

### Common flags

| Flag | Purpose |
|------|---------|
| `-c` | Include cuRobo (compiles CUDA, adds ~10 min) |
| `-r` | Force rebuild the image |
| `-R` | Force rebuild without cache |
| `-d <path>` | Mount custom dataset directory (default: `~/datasets`) |
| `-m <path>` | Mount custom models directory (default: `~/models`) |
| `-e <path>` | Mount custom eval directory (default: `~/eval`) |
| `-s <suffix>` | Container name suffix (for parallel clones) |

### Examples

```bash
# Base (recommended for development)
./docker/run_docker.sh

# Custom mounts + rebuild
./docker/run_docker.sh -r -d ~/my_datasets -m ~/my_models
```

## 3. Verify

Inside the container (already attached), or from outside:

```bash
# From inside the attached container:
/isaac-sim/python.sh -c 'import isaaclab_arena; print(isaaclab_arena.__file__)'

# From outside (find your container first):
ARENA_CONTAINER=$(docker ps --filter "volume=$(pwd)" --format '{{.Names}}' | head -1)
docker exec "$ARENA_CONTAINER" su $(id -un) -c \
  "/isaac-sim/python.sh -c 'import isaaclab_arena; print(isaaclab_arena.__file__)'"
```

Expected output: a path under `/workspaces/isaaclab_arena/`.

## 4. Run a test rollout

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
  --policy_type zero_action --num_steps 20 cube_goal_pose
```

Optional GUI visualizer:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
  --viz kit --policy_type zero_action --num_steps 200 cube_goal_pose
```

## Running commands from outside the container

```bash
ARENA_CONTAINER=$(docker ps --filter "volume=$(pwd)" --format '{{.Names}}' | head -1)
docker exec "$ARENA_CONTAINER" su $(id -un) -c \
  "cd /workspaces/isaaclab_arena && /isaac-sim/python.sh your_script.py"
```

**Important:** Inside `docker exec`, always use `/isaac-sim/python.sh` explicitly — the `python` alias is not available outside the container.

## Container management

- Each repo clone gets its own container (name derived from directory name).
- Running `./docker/run_docker.sh` again re-attaches to the existing container.
- To stop: `exit` from the attached shell, or `docker kill <container-name>`.
- One container per clone — parallel clones run independently.

---

## 5. GR00T Inference Demo in Isaac Sim (server-client)

This demo runs a finetuned GR00T N1.6 policy to control a Unitree G1 humanoid robot in the Galileo lab locomanipulation task (pick a box from a shelf and place it in a bin). It uses a **server-client architecture**:

- **Terminal 1 (GR00T Server, host)**: Loads the GR00T N1.6 checkpoint and serves inference over ZeroMQ on port 5555. Runs in a conda environment outside Docker.
- **Terminal 2 (Arena Client, Docker container)**: Runs Isaac Sim with the G1 environment, connects to the server, and renders the GUI via `--viz kit`.

No teleoperation data collection or policy training is required. A pre-trained checkpoint is downloaded from HuggingFace.

### Requirements

- All [prerequisites](#prerequisites) satisfied (NVIDIA driver, Docker + container toolkit)
- Conda (Miniconda / Anaconda) on the host
- ~15 GB free disk space for the checkpoint + Docker image

### 5.1 Set up the GR00T conda environment (host)

```bash
# Create and activate a dedicated conda env
conda create -n gr00t_server python=3.10 -y
conda activate gr00t_server

# Install PyTorch (match your CUDA version, e.g. cu124)
pip install torch torchvision --index-url https://download.pytorch.org/whl/cu124

# Install GR00T package in editable mode
cd submodules/Isaac-GR00T
pip install -e . --no-build-isolation
```

If `flash-attn` fails to build, install it manually after PyTorch:

```bash
pip install flash-attn==2.7.4.post1 --no-build-isolation
```

### 5.2 Download the pre-trained checkpoint

```bash
export MODELS_DIR=~/models/isaaclab_arena/locomanipulation_tutorial
mkdir -p $MODELS_DIR/checkpoint-20000

huggingface-cli download \
   --revision gn1_6 \
   nvidia/GN1x-Tuned-Arena-G1-Loco-Manipulation \
   --local-dir $MODELS_DIR/checkpoint-20000
```

### 5.3 Start the GR00T server (Terminal 1, host)

```bash
cd submodules/Isaac-GR00T
conda activate gr00t_server

python gr00t/eval/run_gr00t_server.py \
    --modality-config-path /absolute/path/to/IsaacLab-Arena/isaaclab_arena_gr00t/embodiments/g1/g1_sim_wbc_data_config.py \
    --model-path ~/models/isaaclab_arena/locomanipulation_tutorial/checkpoint-20000 \
    --embodiment-tag NEW_EMBODIMENT \
    --device cuda \
    --host 0.0.0.0 \
    --port 5555
```

Wait for: `Server Ready and listening on 0.0.0.0:5555`

Replace `/absolute/path/to/IsaacLab-Arena` with the absolute path to your Arena clone.

### 5.4 Start the Arena client with GUI (Terminal 2, Docker)

In a **second terminal** on the same machine:

```bash
cd /path/to/IsaacLab-Arena
./docker/run_docker.sh
```

Inside the container, run the policy evaluation with the Isaac Sim GUI:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
   --viz kit \
   --policy_type isaaclab_arena_gr00t.policy.gr00t_remote_closedloop_policy.Gr00tRemoteClosedloopPolicy \
   --policy_config_yaml_path isaaclab_arena_gr00t/policy/config/g1_locomanip_gr00t_closedloop_config.yaml \
   --remote_host 127.0.0.1 --remote_port 5555 \
   --num_steps 1500 \
   --enable_cameras \
   galileo_g1_locomanip_pick_and_place \
   --object brown_box \
   --embodiment g1_wbc_joint
```

The Isaac Sim GUI window will appear showing the G1 robot in the Galileo lab scene. The GR00T policy controls the robot to pick the brown box from the shelf and place it in the blue bin.

Expected console output at the end of the evaluation:

```
[Rank 0/1] Metrics: {'success_rate': 1.0, 'num_episodes': 1}
```

> **First run**: The scene assets (Galileo lab, G1 robot, objects) download from Omniverse on first launch. This takes a few minutes and may show `[omni.client.python] Detected a blocking function` warnings — this is normal. Assets are cached to `/tmp/Assets/` for subsequent runs.

### 5.5 Running headless (no GUI)

Replace `--viz kit` with `--viz headless`:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
   --viz headless \
   --policy_type isaaclab_arena_gr00t.policy.gr00t_remote_closedloop_policy.Gr00tRemoteClosedloopPolicy \
   --policy_config_yaml_path isaaclab_arena_gr00t/policy/config/g1_locomanip_gr00t_closedloop_config.yaml \
   --remote_host 127.0.0.1 --remote_port 5555 \
   --num_steps 1500 \
   --enable_cameras \
   galileo_g1_locomanip_pick_and_place \
   --object brown_box \
   --embodiment g1_wbc_joint
```

### 5.6 Cleanup

```bash
# Terminal 1: Stop the server (Ctrl+C)
conda deactivate

# Terminal 2: Exit the container (type 'exit')
```

### Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|------|
| `ValueError: The checkpoint...model type Gr00tN1d7` | Checkpoint is GR00T N1.7 but submodule only supports N1.6 | Use the N1.6 checkpoint (`--revision gn1_6` on `GN1x-Tuned-Arena-G1-Loco-Manipulation`) |
| `shape mismatch: [50, 50] cannot be broadcast to [1, 40, 50]` | Action horizon mismatch between server modality config and client YAML | Ensure server uses `g1_sim_wbc_data_config.py` (horizon 50) and client uses `g1_locomanip_gr00t_closedloop_config.yaml` (horizon 50) |
| `ConnectionError: Cannot reach GR00T policy server` | Server not ready yet | Wait for `Server Ready` in server logs |
| GUI shows empty grid / assets not loading | First-time Omniverse asset download in progress | Wait 5-10 minutes; assets cache to `/tmp/Assets/` for next time |

---

## 6. GR00T DROID Pick-and-Place Demo (table-top, simpler)

This demo runs a **pre-trained GR00T N1.6-DROID** foundation model (no fine-tuning, no checkpoint download) to control a DROID robot (Franka arm + Robotiq gripper) that picks a Rubik's cube off a maple table and places it into a bowl. It reuses the same server-client architecture as Section 5, but uses a different model and a much simpler single-robot table-top environment.

- **Terminal 1 (GR00T Server, host)**: Loads `nvidia/GR00T-N1.6-DROID` and serves inference over ZeroMQ on port 5555. Weights fetch automatically from HuggingFace on first launch.
- **Terminal 2 (Arena Client, Docker container)**: Runs Isaac Sim with the `pick_and_place_maple_table` environment and renders the GUI via `--viz kit`.

### Requirements

Same as Section 5 (conda env `gr00t_server` already set up in 5.1).

> **Note:** The server can only run one model at a time on port 5555. If you previously ran the Section 5 G1 server, stop it (Ctrl+C) before starting the DROID server below.

### 6.1 Start the GR00T server (Terminal 1, host)

```bash
cd submodules/Isaac-GR00T
conda activate gr00t_server

python gr00t/eval/run_gr00t_server.py \
    --model-path nvidia/GR00T-N1.6-DROID \
    --embodiment-tag OXE_DROID \
    --device cuda \
    --host 0.0.0.0 \
    --port 5555
```

Wait for: `Server Ready and listening on 0.0.0.0:5555`

GR00T N1.6-DROID ships with its own modality config (action horizon 32), so `--modality-config-path` is omitted. The first launch downloads the model weights from HuggingFace (~several GB) and caches them locally.

### 6.2 Optional sanity check: zero-action (Terminal 2, container)

Verifies the environment loads before running the real policy:

```bash
cd /path/to/IsaacLab-Arena
./docker/run_docker.sh
```

Inside the container:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
  --viz kit --policy_type zero_action --num_steps 50 \
  pick_and_place_maple_table \
  --embodiment droid_abs_joint_pos \
  --pick_up_object rubiks_cube_hot3d_robolab \
  --destination_location bowl_ycb_robolab \
  --hdr home_office_robolab
```

> **First run**: The scene assets (maple table, DROID robot, Rubik's cube, bowl, HDR) download from Omniverse on first launch — a few minutes with `[omni.client.python] Detected a blocking function` warnings. This is normal.

### 6.3 Run the GR00T closed-loop demo (Terminal 2, container)

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
  --viz kit \
  --policy_type isaaclab_arena_gr00t.policy.gr00t_remote_closedloop_policy.Gr00tRemoteClosedloopPolicy \
  --policy_config_yaml_path isaaclab_arena_gr00t/policy/config/droid_manip_gr00t_closedloop_config.yaml \
  --remote_host 127.0.0.1 --remote_port 5555 \
  --language_instruction "Pick up the Rubik's cube and place it in the bowl." \
  --enable_cameras \
  --num_episodes 3 \
  pick_and_place_maple_table \
  --embodiment droid_abs_joint_pos \
  --pick_up_object rubiks_cube_hot3d_robolab \
  --destination_location bowl_ycb_robolab \
  --hdr home_office_robolab
```

Key differences from the G1 demo in Section 5:

- Uses `droid_manip_gr00t_closedloop_config.yaml` (OXE_DROID embodiment, action horizon 32).
- Uses `--num_episodes 3` instead of `--num_steps`, so the run terminates when episodes complete rather than after a fixed step count.
- `--language_instruction` sets the natural-language instruction sent to the model.
- `--embodiment droid_abs_joint_pos` — N1.6-DROID's default modality config uses absolute joint positions (not `droid_rel_joint_pos`).
- `--enable_cameras` is required — GR00T needs camera observations.

After each episode Arena prints whether the pick-and-place succeeded. The pre-trained model is zero-shot, so expect a low success rate — the point is to demo the closed-loop pipeline.

### 6.4 Running headless (no GUI)

Replace `--viz kit` with `--viz headless`:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
  --viz headless \
  --policy_type isaaclab_arena_gr00t.policy.gr00t_remote_closedloop_policy.Gr00tRemoteClosedloopPolicy \
  --policy_config_yaml_path isaaclab_arena_gr00t/policy/config/droid_manip_gr00t_closedloop_config.yaml \
  --remote_host 127.0.0.1 --remote_port 5555 \
  --language_instruction "Pick up the Rubik's cube and place it in the bowl." \
  --enable_cameras \
  --num_episodes 3 \
  pick_and_place_maple_table \
  --embodiment droid_abs_joint_pos \
  --pick_up_object rubiks_cube_hot3d_robolab \
  --destination_location bowl_ycb_robolab \
  --hdr home_office_robolab
```

### 6.5 Optional: batch evaluation across variations

Runs nine jobs sequentially — each varying the object, background HDR, and destination — within a single Isaac Sim process, reporting per-job success rates:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/experiment_runner.py \
  --viz kit \
  --eval_jobs_config isaaclab_arena_environments/eval_jobs_configs/droid_pnp_srl_gr00t_jobs_config.json
```

### 6.6 Swap objects, backgrounds, and destinations

From the baseline command, swap any of:

- `--pick_up_object` — e.g. `mustard_bottle_hot3d_robolab`, `mug_hot3d_robolab`, `soup_can_hot3d_robolab`, `tomato_soup_can_ycb_robolab`
- `--destination_location` — `bowl_ycb_robolab`, `wooden_bowl_hot3d_robolab`
- `--hdr` — e.g. `billiard_hall_robolab`, `empty_warehouse_robolab`, `garage_robolab`
- `--light_intensity` — dome light brightness

### 6.7 Cleanup

```bash
# Terminal 1: Stop the server (Ctrl+C)
conda deactivate

# Terminal 2: Exit the container (type 'exit')
```

---

## 7. OpenPi pi05 Pick-and-Place Demo (recommended for `pick_and_place_maple_table`)

The GR00T N1.6-DROID model in Section 6 is zero-shot on the maple-table task and often fails to pick the cube. **OpenPi pi05** (Physical Intelligence, fine-tuned on DROID) is the recommended alternative — it succeeds on most `pick_and_place_maple_table` variations out of the box (docs report 1.0 success rate on 8/9 variations).

It reuses the same server-client architecture, but replaces the GR00T server with an **openpi inference server** over WebSocket:

- **Terminal 1 (OpenPi Server, host)**: Runs `pi05` (or `pi0`) in a self-contained Docker image, serving inference on port 8000.
- **Terminal 2 (Arena Client, Docker container)**: Runs Isaac Sim with the `pick_and_place_maple_table` environment and sends observations / receives actions over WebSocket.

### Requirements

- All [prerequisites](#prerequisites) satisfied
- ~30 GB free disk space (19 GB image + 11 GB checkpoint) and ~24 GB free VRAM on a single GPU

> **Note:** pi05 is a JAX model needing ~15-20 GB of VRAM. The `run_openpi_server.sh` script pins the server to **GPU 0** (`--gpus '"device=0"'`). If your primary GPU is not index 0 (e.g. you have a small secondary card), edit that flag to select the correct device.

### 7.1 Start the OpenPi server (Terminal 1, host)

```bash
./isaaclab_arena_openpi/docker/run_openpi_server.sh
```

- **First invocation** builds the `isaaclab_arena_openpi-server` Docker image (~3 min, ~19 GB), then downloads the ~11 GB pi05 checkpoint into the container.
- **Subsequent runs** reuse the cached image and checkpoint.
- `-v pi0` serves the smaller pi0 variant instead of pi05; `-r` forces a rebuild.

Wait for:

```
INFO:websockets.server:server listening on 0.0.0.0:8000
```

Leave the terminal running.

### 7.2 Optional sanity check: zero-action (Terminal 2, container)

Same as Section 6.2 — verifies the environment loads before running the real policy:

```bash
cd /path/to/IsaacLab-Arena
./docker/run_docker.sh
```

Inside the container:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
  --viz kit --policy_type zero_action --num_steps 50 \
  pick_and_place_maple_table \
  --embodiment droid_abs_joint_pos \
  --pick_up_object rubiks_cube_hot3d_robolab \
  --destination_location bowl_ycb_robolab \
  --hdr home_office_robolab
```

> **First run**: The scene assets (maple table, DROID robot, Rubik's cube, bowl, HDR) download from Omniverse on first launch — a few minutes with `[omni.client.python] Detected a blocking function` warnings. This is normal.

### 7.3 Run the pi05 closed-loop demo (Terminal 2, container)

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
  --viz kit \
  --policy_type isaaclab_arena_openpi.policy.pi0_remote_policy.Pi0RemotePolicy \
  --num_episodes 3 \
  --enable_cameras --num_envs 1 \
  --language_instruction "Pick up the Rubik's cube and place it in the bowl." \
  pick_and_place_maple_table \
    --embodiment droid_abs_joint_pos \
    --pick_up_object rubiks_cube_hot3d_robolab \
    --destination_location bowl_ycb_robolab \
    --hdr home_office_robolab
```

Defaults: `--openpi_embodiment_adapter droid`, `--policy_variant pi05`, `--remote_host localhost`, `--remote_port 8000`. Pass `--remote_host` only if the server is on a different machine.

The Arena Kit window will show the DROID arm picking the Rubik's cube and placing it in the bowl. pi05 succeeds on most variations zero-shot.

### 7.4 Supported variants

| `--policy_variant` | Model | Checkpoint | Pair with `--embodiment` |
|--------------------|-------|------------|--------------------------|
| `pi05` (default)   | `pi05_droid_jointpos_polaris` | `gs://openpi-assets-simeval/pi05_droid_jointpos` | `droid_abs_joint_pos` |
| `pi0`              | `pi0_droid_jointpos_polaris`  | `gs://openpi-assets-simeval/pi0_droid_jointpos`  | `droid_abs_joint_pos` |

Serve `pi0` by running `./isaaclab_arena_openpi/docker/run_openpi_server.sh -v pi0`.

### 7.5 Optional: batch evaluation across variations

Runs nine jobs sequentially — each varying the object, background HDR, and destination — within a single Isaac Sim process, reporting per-job success rates:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/experiment_runner.py \
  --viz kit \
  --enable_cameras \
  --experiment_config isaaclab_arena_environments/experiment_configs/droid_pnp_srl_openpi_experiment.yaml
```

> The experiment config adds cameras to each environment, while `--enable_cameras` enables camera support in Isaac Sim before the experiment is loaded — both are required.

### 7.6 Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|------|
| `jaxlib.xla_extension.XlaRuntimeError: RESOURCE_EXHAUSTED: Out of memory while trying to allocate ...` | JAX placed the model on a GPU that is too small (e.g. an 8 GB secondary card) | Pin the server to your large GPU: edit `run_openpi_server.sh`'s `--gpus '"device=0"'` to the correct index |
| `ConnectionRefusedError` from the Arena client | Server not ready or port mismatch | Wait for `server listening on 0.0.0.0:8000`; confirm `--remote_port 8000` |

### 7.7 Cleanup

```bash
# Terminal 1: Stop the server (Ctrl+C)
# Terminal 2: Exit the container (type 'exit')
```
