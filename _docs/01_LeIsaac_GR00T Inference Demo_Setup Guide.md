---
layout: doc
title: LeIsaac + GR00T Inference Demo — Setup Guide
description: >
  A comprehensive guide to setting up a two-environment system for running policy inference in IsaacLab using a GR00T N1.5 model served via a separate inference service.
video: https://youtu.be/DSIHzQhY04Y
hide_description: true
---

A comprehensive guide to setting up a two-environment system for running policy inference in IsaacLab using a GR00T N1.5 model served via a separate inference service.

---

## Architecture Overview



### Inference-Only Mode

```
┌─────────────────────────┐         TCP :5555         ┌──────────────────────────────┐
│  Environment 1          │ ◄─────────────────────►   │  Environment 2               │
│  gr00t_server           │                           │  leisaac_envhub              │
│                         │                           │                              │
│  - Python 3.10          │                           │  - Python 3.11               │
│  - Isaac-GR00T          │                           │  - IsaacSim 5.1.0            │
│  - GR00T N1.5 model     │                           │  - IsaacLab 2.3.0            │
│  - inference_service.py │                           │  - LeIsaac                   │
│  ────────────────       │                           │  - policy_inference.py       │
│  Starts server on       │                           │  ────────────────             │
│  tcp://0.0.0.0:5555     │                           │  Connects as client          │
└─────────────────────────┘                           └──────────────────────────────┘
```
### Demo

<div style="background-color: #f0f0f0; padding: 8px; border-radius: 4px;">

| LiftCube | PickOrange |
|:--------:|:----------:|
| <video src="/assets/video/groot_leisaac_cube.mp4" controls width="320" height="180"><br/>  Your browser does not support the video tag.<br/></video> | <video src="/assets/video/groot_leisaac_orange.mp4" controls width="320" height="180"><br/>  Your browser does not support the video tag.<br/></video> |

</div>


### Teacher-Student Recording Mode

```
┌─────────────────────────┐         TCP :5555         ┌──────────────────────────────────────────┐
│  Environment 1          │ ◄─────────────────────►   │  Environment 2                           │
│  gr00t_server           │                           │  leisaac_envhub                          │
│  (Teacher)              │                           │                                          │
│                         │                           │  - teacher_generate.py                   │
│  Serves GR00T N1.5     │                           │    ┌─────────────────────┐               │
│  as remote policy       │                           │    │ IsaacSim Simulation │               │
│                         │                           │    │  ┌───────────────┐  │               │
│                         │                           │    │  │ SO101 Robot   │  │               │
│                         │                           │    │  │ Pick Orange   │  │               │
│                         │                           │    │  │ Environment   │  │               │
│                         │                           │    │  └───────┬───────┘  │               │
│                         │                           │    │          │          │               │
│                         │                           │    │          ▼          │               │
│                         │                           │    │  ┌───────────────┐  │               │
│                         │                           │    │  │  Recorder     │  │               │
│                         │                           │    │  │  (HDF5 /      │  │               │
│                         │                           │    │  │  LeRobot)     │  │               │
│                         │                           │    │  └───────┬───────┘  │               │
│                         │                           │    └─────────┼─────────┘               │
│                         │                           │              ▼                          │
│                         │                           │    ./datasets/teacher_demos.hdf5        │
│                         │                           │              │                          │
│                         │                           │              ▼                          │
│                         │                           │    ┌──────────────────────┐             │
│                         │                           │    │  Convert & Train    │             │
│                         │                           │    │  Student Policy     │             │
│                         │                           │    │  (Diffusion / ACT / │             │
│                         │                           │    │   Pi0 / SmolVLA)    │             │
│                         │                           │    └──────────────────────┘             │
└─────────────────────────┘                           └──────────────────────────────────────────┘
```

---

## Prerequisites

- **OS:** Ubuntu 20.04 / 22.04
- **GPU:** NVIDIA GPU with CUDA 12.8 support (tested on T1000 8 GB)
- **Driver:** ≥ 570.86 (for CUDA 12.8)
- **Storage:** ~30 GB free for environments, assets, and model checkpoints
- **Conda:** Installed and initialized (`conda init`)

---

## Environment 1 — GR00T Inference Server

This environment runs the GR00T N1.5 model as a gRPC/TCP service.

### 1. Create the Conda Environment

```bash
conda create -n gr00t_server python=3.10
conda activate gr00t_server
```

### 2. Install CUDA Toolkit

```bash
conda install -c "nvidia/label/cuda-12.8.1" cuda-toolkit
```

### 3. Install PyTorch

```bash
pip install -U torch==2.7.0 torchvision==0.22.0 \
    --index-url https://download.pytorch.org/whl/cu128
```

### 4. Clone and Install Isaac-GR00T

```bash
cd /home/cong/Desktop/code/vla_robotics
git clone https://github.com/NVIDIA/Isaac-GR00T.git
cd Isaac-GR00T
git checkout 4af2b622892f7dcb5aae5a3fb70bcb02dc217b96

pip install --upgrade setuptools
pip install -e .[base]
pip install --no-build-isolation flash-attn==2.7.1.post4
```

### 5. Download the Fine-Tuned Model

```bash
huggingface-cli download LightwheelAI/leisaac-pick-orange-v0 \
    --local-dir ./models/leisaac-pick-orange-v0
```

### 6. Start the Inference Server

```bash
conda activate gr00t_server
cd /home/cong/Desktop/code/vla_robotics/Isaac-GR00T

python scripts/inference_service.py --server \
    --model-path ./models/leisaac-pick-orange-v0 \
    --embodiment-tag new_embodiment \
    --data-config so100_dualcam \
    --port 5555
```

Wait for the following confirmation message:

```
Server is ready and listening on tcp://0.0.0.0:5555
```

> **⚠️ Keep this terminal running** — the server must be active before launching the client.

---

## Environment 2 — LeIsaac Simulation Client

This environment contains IsaacLab, IsaacSim, and LeIsaac. It runs the simulated environment and connects to the GR00T server for policy inference.

### 1. Create the Conda Environment

```bash
conda create -n leisaac_envhub python=3.11
conda activate leisaac_envhub
```

### 2. Install CUDA Toolkit

```bash
conda install -c "nvidia/label/cuda-12.8.1" cuda-toolkit
```

### 3. Install PyTorch

```bash
pip install -U torch==2.7.0 torchvision==0.22.0 \
    --index-url https://download.pytorch.org/whl/cu128
```

### 4. Install IsaacSim

```bash
pip install --upgrade pip
pip install "isaacsim[all,extscache]==5.1.0" \
    --extra-index-url https://pypi.nvidia.com
```

### 5. Clone LeIsaac with Submodules

```bash
mkdir -p /home/cong/Desktop/code/vla_robotics/lerobot_example
cd /home/cong/Desktop/code/vla_robotics/lerobot_example
git clone https://github.com/LightwheelAI/leisaac.git --recursive
cd leisaac
```

### 6. Install IsaacLab Dependencies

```bash
sudo apt install cmake build-essential

cd dependencies/IsaacLab
./isaaclab.sh --install
cd ../..
```

### 7. Install LeIsaac

```bash
pip install -e source/leisaac
```

> **Optional:** For LeRobot integration (data conversion, lerobot recorder):
> ```bash
> pip install -e "source/leisaac[lerobot]"
> pip install numpy==1.26.0
> ```

---

## Asset Preparation

USD scene and robot assets are **not** included in the Git repository. Download them from the [LeIsaac Releases](https://github.com/LightwheelAI/leisaac/releases) page.

### Scene Assets

| Scene | Release Tag | Download Link |
|-------|-------------|---------------|
| Kitchen with Orange | `v0.1.0` | [Download](https://github.com/LightwheelAI/leisaac/releases/tag/v0.1.0) |
| Table with Cube | `v0.1.2` | [Download](https://github.com/LightwheelAI/leisaac/releases/tag/v0.1.2) |
| Lightwheel Bedroom | `v0.2.0` | [Download](https://github.com/LightwheelAI/leisaac/releases/tag/v0.2.0) |
| Lightwheel Loft | `v0.3.0` | [Download](https://github.com/LightwheelAI/leisaac/releases/tag/v0.3.0) |

### Robot Assets

Robot USD files are included in the same releases or can be downloaded from the [HuggingFace repository](https://huggingface.co/LightwheelAI/leisaac_env).

### Expected Directory Structure

After extraction, the `assets/` folder should look like:

```
leisaac/assets/
├── robots/
│   └── so101_follower.usd
└── scenes/
    └── kitchen_with_orange/
        ├── scene.usd              # 38 MB scene file
        ├── assets/
        └── objects/
            ├── Orange001/
            ├── Orange002/
            ├── Orange003/
            └── Plate/
```

---

## Fixing Dependency Conflicts

Due to the pip dependency resolver limitations, the following packages may have version conflicts. Install the versions expected by IsaacSim:

```bash
pip install click==8.1.7 psutil==5.9.8 typing_extensions==4.12.2
```

> These are warnings, not errors — most functionality works correctly without these fixes.

---

## Running the Demo

### Step 1 — Start the Inference Server

In **Terminal 1**:

```bash
conda activate gr00t_server
cd /home/cong/Desktop/code/vla_robotics/Isaac-GR00T

python scripts/inference_service.py --server \
    --model-path ./models/leisaac-pick-orange-v0 \
    --embodiment-tag new_embodiment \
    --data-config so100_dualcam \
    --port 5555
```

### Step 2 — Launch the Simulation Client

In **Terminal 2** (after the server reports ready):

```bash
conda activate leisaac_envhub
cd /home/cong/Desktop/code/vla_robotics/lerobot_example/leisaac

python scripts/evaluation/policy_inference.py \
    --task=LeIsaac-SO101-PickOrange-v0 \
    --eval_rounds=10 \
    --policy_type=gr00tn1.5 \
    --policy_host=localhost \
    --policy_port=5555 \
    --policy_timeout_ms=5000 \
    --policy_action_horizon=16 \
    --policy_language_instruction="Pick up the orange and place it on the plate" \
    --device=cuda \
    --enable_cameras
```

---

## Teacher-Student Workflow: Record Dataset with GR00T

This workflow uses the GR00T N1.5 policy as a **teacher** to generate demonstration data in simulation, which is then used to train a **student** policy (Diffusion, ACT, Pi0, SmolVLA, etc.).

### Data Flow

```
GR00T Teacher (server)  ──►  IsaacSim + LeIsaac (client)
                                    │
                                    ▼
                              teacher_generate.py
                              ───────────────────
                              Runs GR00T policy in simulation,
                              records (obs, action, done) automatically
                                    │
                                    ▼
                              ./datasets/teacher_demos.hdf5
                                    │
                                    ▼
                              scripts/convert/isaaclab2lerobot.py
                                    │
                                    ▼
                              LeRobot Dataset (HF Hub / local)
                                    │
                                    ▼
                              lerobot-train --policy=<student>
```

### Step 1 — Start the GR00T Server (Terminal 1)

```bash
conda activate gr00t_server
cd /home/cong/Desktop/code/vla_robotics/Isaac-GR00T

python scripts/inference_service.py --server \
    --model-path ./models/leisaac-pick-orange-v0 \
    --embodiment-tag new_embodiment \
    --data-config so100_dualcam \
    --port 5555
```

### Step 2 — Collect Demonstrations (Terminal 2)

```bash
conda activate leisaac_envhub
cd /home/cong/Desktop/code/vla_robotics/lerobot_example/leisaac

python scripts/datagen/state_machine/teacher_generate.py \
    --task=LeIsaac-SO101-PickOrange-v0 \
    --device=cuda \
    --enable_cameras \
    --record \
    --dataset_file=./datasets/teacher_demos.hdf5 \
    --num_demos=100 \
    --policy_host=localhost \
    --policy_port=5555 \
    --policy_type=gr00tn1.5 \
    --policy_action_horizon=16 \
    --policy_language_instruction="Pick up the orange and place it on the plate"
```

| Argument | Description |
|----------|-------------|
| `--num_demos` | Number of **successful** episodes to collect (0 for infinite) |
| `--dataset_file` | Output path for recorded HDF5 (default: `./datasets/teacher_demos.hdf5`) |
| `--record` | Enable recording (required) |
| `--resume` | Resume appending to an existing dataset |
| `--use_lerobot_recorder` | Record directly in LeRobot format (skip conversion step) |
| `--lerobot_dataset_repo_id` | HuggingFace repo ID (required with `--use_lerobot_recorder`) |

The recorder captures observations and actions automatically on every `env.step()`. Only **successful** episodes (where the task is completed) are saved; failed/timeout episodes are discarded.

### Step 3 — Convert to LeRobot Format

After collection, convert the HDF5 dataset to LeRobot format for training:

```bash
# Install lerobot if not already done
pip install -e "source/leisaac[lerobot]"

# Convert HDF5 → LeRobot Dataset v2
python scripts/convert/isaaclab2lerobot.py \
    --task_name=LeIsaac-SO101-PickOrange-v0 \
    --repo_id=myuser/teacher-dataset \
    --hdf5_root=./datasets \
    --hdf5_files=teacher_demos.hdf5

# Or for LeRobot Dataset v3
python scripts/convert/isaaclab2lerobotv3.py \
    --task_name=LeIsaac-SO101-PickOrange-v0 \
    --repo_id=myuser/teacher-dataset \
    --hdf5_root=./datasets \
    --hdf5_files=teacher_demos.hdf5
```

### Step 4 — Train a Student Policy

Once the dataset is in LeRobot format, train any supported student policy:

```bash
# Diffusion Policy
lerobot-train --policy=diffusion --dataset.repo_id=myuser/teacher-dataset

# ACT (Action Chunking Transformer)
lerobot-train --policy=act --dataset.repo_id=myuser/teacher-dataset

# Pi0
lerobot-train --policy=pi0 --dataset.repo_id=myuser/teacher-dataset

# SmolVLA
lerobot-train --policy=smolvla --dataset.repo_id=myuser/teacher-dataset
```

### Student Model Comparison

| Model | Type | Strengths | Weaknesses | Recommended Demos |
|-------|------|-----------|------------|-----------------|
| **Diffusion** | Diffusion over actions | Multimodal (multiple valid strategies), robust, smooth | Slower inference, more compute | 50–200 |
| **ACT** | Transformer (CVAE) | Precise, smooth, good for fine manipulation | Requires action chunking, less multimodal | 50–200 |
| **Pi0** | VLA (Vision-Language-Action) | Generalizes well, language-conditioned | Large model, heavy compute | 100–1000+ |
| **SmolVLA** | Small VLA | Fast inference, runs on low-end GPUs | Less capable than larger models | 100–500 |

> **Key insight:** The recording format is identical for all student models — record once with GR00T, then train any student from the same dataset.

### Recording Directly to LeRobot Format (Skip Step 3)

To bypass the HDF5 → LeRobot conversion step, use `--use_lerobot_recorder`:

```bash
python scripts/datagen/state_machine/teacher_generate.py \
    --task=LeIsaac-SO101-PickOrange-v0 \
    --device=cuda --enable_cameras \
    --record \
    --use_lerobot_recorder \
    --lerobot_dataset_repo_id=myuser/teacher-dataset \
    --lerobot_dataset_fps=30 \
    --num_demos=100 \
    --policy_host=localhost --policy_port=5555 \
    --policy_language_instruction="Pick up the orange and place it on the plate"
```

The dataset is saved directly in LeRobot format and can be used immediately for training without conversion.

### Source Code Modifications for Successful-Only Recording

By default, the original LeIsaac source records **all** episodes (both success and failure). The following changes were made to the source code to record only successful episodes:

#### 1. `teacher_generate.py` — Termination & Export Mode

**File:** `scripts/datagen/state_machine/teacher_generate.py`

- **Removed** the lines that nullified `time_out` and `success` termination terms. The task's native termination logic (e.g. `mdp.task_done` for PickOrange, `mdp.time_out`) now works as designed — episodes end when the task is complete or the timeout is reached.
- **Changed** both HDF5 and LeRobot recorder export modes from `EXPORT_ALL` / `EXPORT_ALL_RESUME` to `EXPORT_SUCCEEDED_ONLY` / `EXPORT_SUCCEEDED_ONLY_RESUME`. Only episodes where the `success` termination term triggered (not timeouts) are saved to disk.
- **Fixed** the episode success check: `episode_success = not reset_time_outs[0]` ensures timeouts are not incorrectly counted as successful.

#### 2. `recorder_manager.py` — StreamingRecorderManager

**File:** `source/leisaac/leisaac/enhance/managers/recorder_manager.py`

- **Added** `EXPORT_SUCCEEDED_ONLY` and `EXPORT_SUCCEEDED_ONLY_RESUME` to the supported export modes assertion.
- **Added** resume handling for `EXPORT_SUCCEEDED_ONLY_RESUME` mode.
- **Added** succeeded-only filtering in `export_episodes()`: episodes are written to the dataset file only when the export mode allows it AND the episode was successful.
- **Fixed** a cache leak: `_clear_episode_cache` is now called unconditionally after each episode (not only on write), preventing failed episode data from accumulating across resets.

---

## Available Tasks

| Task ID | Description | Robot |
|---------|-------------|-------|
| `LeIsaac-SO101-PickOrange-v0` | Pick three oranges and place on plate | Single-Arm SO101 |
| `LeIsaac-SO101-LiftCube-v0` | Lift the red cube up | Single-Arm SO101 |
| `LeIsaac-SO101-CleanToyTable-v0` | Pick two letter E objects into box | Single-Arm SO101 |
| `LeIsaac-SO101-FoldCloth-BiArm-v0` | Fold a cloth | Bi-Arm SO101 |
| `LeIsaac-LeKiwi-CleanupTrash-v0` | Pick up trash from floor | LeKiwi mobile |

---

## Troubleshooting

| Problem | Likely Cause | Solution |
|---------|-------------|----------|
| `No module named 'tyro'` | GR00T deps not installed | Run `pip install -e .[base]` in Isaac-GR00T |
| `cannot import name 'runtime_version'` | Wrong `protobuf` version | Server env needs `protobuf==4.25.1` (set by `pip install -e .[base]`) |
| `Service is not running` | Server not started | Start server first in Terminal 1 |
| `Failed to open layer @scene.usd@` | USD assets missing | Download scene assets from Releases page |
| `joint between static bodies` | Scene USD warnings | PhysX warnings — usually benign |
| GPU out of memory | T1000 8 GB limited | Close other GPU processes; consider running server on separate machine |

---

## Version Compatibility Matrix

| LeIsaac | IsaacSim | IsaacLab | Python | CUDA | PyTorch |
|---------|----------|----------|--------|------|---------|
| v0.4.x | 5.1.0 | 2.3.0 | 3.11 | 12.8 | 2.7.0 |
| v0.3.x | 5.0.0 | 2.2.1 | 3.11 | 12.8 | 2.7.0 |
| v0.2.x | 4.5.0 | 2.1.1 | 3.10 | 11.8 | 2.5.1 |

---

## References

- [LeIsaac Documentation](https://lightwheelai.github.io/leisaac/)
- [IsaacLab Documentation](https://isaac-sim.github.io/IsaacLab/)
- [Isaac-GR00T Repository](https://github.com/NVIDIA/Isaac-GR00T)
- [LeRobot Documentation](https://huggingface.co/docs/lerobot/)
- [Sample Policy on HuggingFace](https://huggingface.co/LightwheelAI/leisaac-pick-orange-v0)
