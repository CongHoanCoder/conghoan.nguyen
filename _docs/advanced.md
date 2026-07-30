---
layout: doc
title: Setup Guide — Isaac Lab-Arena (Docker)
description: >
  This chapter covers advanced topics, such as offline support and custom JS builds. Codings skills are recommended.
hide_description: true
---

This chapter covers advanced topics, such as offline support and custom JS builds. Codings skills are recommended.
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