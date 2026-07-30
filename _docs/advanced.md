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

## 5. GR00T Policy Evaluation Demo (two-container setup)

This demo runs a finetuned GR00T N1.7 policy to control a Unitree G1 robot in the static apple-to-plate task using Arena's **server-client architecture**. Two Docker containers are used:

- **Container 1 (GR00T Server)**: Loads the GR00T N1.7 checkpoint and serves inference over ZeroMQ on port 5555.
- **Container 2 (Arena Client)**: Runs Isaac Sim with the G1 environment. It connects to the server over ZeroMQ to request actions — no GR00T model dependencies inside this container.

No teleoperation data collection or policy training is required. A pre-trained checkpoint is downloaded from HuggingFace.

### Requirements

- All [prerequisites](#prerequisites) satisfied (NVIDIA driver, Docker + container toolkit)
- ~25 GB free disk space for the two Docker images + checkpoint
- `huggingface-cli` on the host:

  ```bash
  pip install huggingface-hub[cli]
  ```

### 5.1 Build the GR00T server image

The Arena client image (`isaaclab_arena:latest`) is built automatically by `./docker/run_docker.sh` — you do not need to build it separately.

```bash
# From the repo root
cd /path/to/IsaacLab-Arena

# GR00T server image (PyTorch 25.04 + Isaac-GR00T) — ~5-10 min
docker build -t isaaclab_arena-gr00t-server:latest -f docker/Dockerfile.gr00t_server .
```

### 5.2 Download the pre-trained checkpoint

```bash
export MODELS_DIR=~/models/isaaclab_arena/static_apple_tutorial
export MODEL_PATH=$MODELS_DIR/gn1x_tuned_static_apple
mkdir -p "$MODEL_PATH"

huggingface-cli download \
   nvidia/GN1x-Tuned-Arena-G1-Static-PickNPlace \
   --repo-type model \
   --local-dir "$MODEL_PATH"
```

### 5.3 Start the GR00T server container

```bash
docker run -d --gpus all --network host --name gr00t-server \
  -v ~/models:/models:ro \
  -v /path/to/IsaacLab-Arena/isaaclab_arena_gr00t/embodiments:/arena_embodiments:ro \
  isaaclab_arena-gr00t-server:latest \
  uv run python gr00t/eval/run_gr00t_server.py \
    --modality-config-path /arena_embodiments/g1/g1_sim_wbc_data_gr00t_n_1_7_config.py \
    --model-path /models/isaaclab_arena/static_apple_tutorial/gn1x_tuned_static_apple \
    --embodiment-tag NEW_EMBODIMENT \
    --device cuda \
    --host 0.0.0.0 \
    --port 5555
```

Check the server is ready:

```bash
docker logs gr00t-server
```

Expected output: `Server Ready and listening on 0.0.0.0:5555`

> **Note**: If you are using a custom checkpoint (from your own training), replace `gn1x_tuned_static_apple` with your checkpoint directory name.

### 5.4 Start the Arena client container

In a **second terminal** on the same machine:

```bash
cd /path/to/IsaacLab-Arena
./docker/run_docker.sh
```

Inside the container, run the policy evaluation with GUI:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
   --viz kit \
   --policy_type isaaclab_arena_gr00t.policy.gr00t_remote_closedloop_policy.Gr00tRemoteClosedloopPolicy \
   --policy_config_yaml_path isaaclab_arena_gr00t/policy/config/g1_static_apple_gr00t_closedloop_config.yaml \
   --remote_host localhost --remote_port 5555 \
   --num_steps 600 \
   --enable_cameras \
   galileo_g1_static_pick_and_place \
   --object apple_01_objaverse_robolab \
   --destination clay_plates_hot3d_robolab \
   --embodiment g1_wbc_agile_joint
```

The Isaac Sim GUI will appear showing the G1 robot in the Galileo lab scene. The GR00T policy controls the robot to pick the apple and place it on the plate.

Expected console output at the end of the evaluation:

```
[Rank 0/1] Metrics: {'success_rate': 1.0, 'object_moved_rate': 1.0, 'num_episodes': 1}
```

### 5.5 Running headless (no GUI)

Replace `--viz kit` with `--viz headless`:

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
   --viz headless \
   --policy_type isaaclab_arena_gr00t.policy.gr00t_remote_closedloop_policy.Gr00tRemoteClosedloopPolicy \
   --policy_config_yaml_path isaaclab_arena_gr00t/policy/config/g1_static_apple_gr00t_closedloop_config.yaml \
   --remote_host localhost --remote_port 5555 \
   --num_steps 600 \
   --enable_cameras \
   galileo_g1_static_pick_and_place \
   --object apple_01_objaverse_robolab \
   --destination clay_plates_hot3d_robolab \
   --embodiment g1_wbc_agile_joint
```

### 5.6 Running with parallel environments

```bash
/isaac-sim/python.sh isaaclab_arena/evaluation/policy_runner.py \
   --viz kit \
   --policy_type isaaclab_arena_gr00t.policy.gr00t_remote_closedloop_policy.Gr00tRemoteClosedloopPolicy \
   --policy_config_yaml_path isaaclab_arena_gr00t/policy/config/g1_static_apple_gr00t_closedloop_config.yaml \
   --remote_host localhost --remote_port 5555 \
   --num_steps 600 \
   --num_envs 5 \
   --enable_cameras \
   galileo_g1_static_pick_and_place \
   --object apple_01_objaverse_robolab \
   --destination clay_plates_hot3d_robolab \
   --embodiment g1_wbc_agile_joint
```

### 5.7 Cleanup

Stop and remove the GR00T server container:

```bash
docker stop gr00t-server && docker rm gr00t-server
```

Exit the Arena container by typing `exit`.

### Troubleshooting

| Symptom | Likely cause | Fix |
|---------|-------------|-----|
| `ConnectionError: Cannot reach GR00T policy server` | Server not ready yet | Wait for `Server Ready` in server logs. Check `docker logs gr00t-server`. |
| `ValueError: Invalid action shape, expected: 23, received: 50.` | Client uses wrong embodiment | Ensure `--embodiment g1_wbc_agile_joint` (not `_pink`). |
| `FileNotFoundError: Model path ... does not exist` | Model volume mount path mismatch | Check `-v` mount matches `--model-path` inside container. |
| Apple falls through shelf on first run | Collision mesh cache not ready | Re-run the command once (cached in `/tmp/Assets/`). |
| Kit permission errors on `/isaac-sim/kit/data/` | Stale state from previous run | Rebuild the Arena image or start from a clean container. |
