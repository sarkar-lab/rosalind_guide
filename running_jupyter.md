---
layout: page
title: Running Jupyter Notebooks on Rosalind
permalink: /jupyter/
---

# Running Jupyter Notebooks on Rosalind

This page explains the resources Rosalind offers, how to choose between a **whole GPU** and a **shared GPU**, and how to run a Jupyter Lab session on the cluster.

## Contents

1. [Partitions and Hardware](#partitions-and-hardware)
2. [Whole GPU or a Share of a GPU?](#whole-gpu-or-a-share-of-a-gpu)
   - [Shared vs full at a glance](#shared-vs-full-at-a-glance)
   - [Which one should I use for Jupyter?](#which-one-should-i-use-for-jupyter)
3. [GPU Usage Policy](#gpu-usage-policy-important)
4. [How Jupyter Runs on Rosalind](#how-jupyter-runs-on-rosalind)
5. [Step 1: SSH Into Rosalind](#step-1-ssh-into-rosalind)
6. [Step 2: Create the SBATCH Script](#step-2-create-the-sbatch-script)
7. [Step 3: Create SSH Tunnel](#step-3-create-ssh-tunnel)
8. [Running Jupyter on a GPU Node](#running-jupyter-on-a-gpu-node)

---

## Partitions and Hardware

Rosalind provides multiple Slurm partitions designed for different workloads.
Please choose the appropriate partition based on your job size and runtime.

### Available Partitions

| Partition      | Max Time     | Hardware        | Intended Use |
|---------------|-------------|----------------|--------------|
| `debug`       | 2 hours     | CPU nodes      | Quick testing, debugging |
| `cpus`        | 12 hours    | CPU nodes      | Standard CPU workloads (default) |
| `batch`       | 7 days      | CPU nodes      | Long CPU runs (lowest priority) |
| `debug-gpu`   | 2 hours     | GPU nodes      | Short GPU testing |
| `gpus`        | 12 hours    | GPU nodes      | Standard GPU workloads |

### CPU Nodes
Each CPU node provides:

- 384 CPU cores  
- ~770 GB RAM  

CPU nodes are shared resources. Jobs request specific cores and memory:
```
#SBATCH --cpus-per-task=32
#SBATCH --mem=64G
```

Please avoid requesting excessive cores or memory unless your workload requires it.

### GPU Nodes

Each GPU node provides:

- 2 NVIDIA RTX PRO 6000 Blackwell GPUs (~96 GB GPU memory each)
- 384 CPU cores
- ~1 TB CPU RAM

To request a GPU, choose either a **whole GPU** (`--gres=gpu:blackwell:1`) or a **share of a GPU** (`--gres=shard:N`, a good fit for notebooks). See [Whole GPU or a Share of a GPU?](#whole-gpu-or-a-share-of-a-gpu) below.

### Quick checks 
You can quickly check if you can run on CPU or GPU nodes by running:

```bash
sbatch -p debug --wrap="hostname; sleep 120" --cpus-per-task=32 --mem=10G
```
For GPU nodes:

```bash
sbatch -p debug-gpu --gres=gpu:blackwell:1 --wrap="nvidia-smi -L; sleep 120"
```

## Whole GPU or a Share of a GPU?

Rosalind has **4 GPUs** (2 per GPU node), each with about 96 GB of GPU memory. There are two ways to request GPU time:

| You ask for | Meaning | Use it for |
|-------------|---------|------------|
| `--gres=gpu:blackwell:1` | One **whole** GPU, exclusively yours | Large training runs that need the full card's memory or speed |
| `--gres=shard:N` | `N` quarter-GPU *shares*, **shared** with other jobs | Notebooks, development, inference, small to medium training |

**How shards work.** Each GPU is split into 4 shards, so a node has 8 shards. One shard is roughly a quarter of a GPU, about **24 GB** of GPU memory. Up to 4 single-shard jobs can run on the same GPU at once, and up to 16 across the cluster.

| Request | About how much GPU memory |
|---------|---------------------------|
| `--gres=shard:1` | ~24 GB |
| `--gres=shard:2` | ~48 GB |
| `--gres=shard:3` | ~72 GB |
| `--gres=shard:4` | a full GPU's worth (but it is still shared; use `gpu:blackwell:1` if you need exclusive use) |

### Shared vs full at a glance

| | **Shared GPU** (`--gres=shard:N`) | **Whole GPU** (`--gres=gpu:blackwell:1`) |
|---|---|---|
| GPU memory | about 24 GB per shard (`shard:2` is about 48 GB). Not enforced. | The whole card, about 96 GB |
| Speed | Shared with up to 3 other jobs on the same GPU | Full speed, no neighbours |
| Neighbours | Yes. A neighbour using too much memory can make your job crash | None |
| Waiting | Can start whenever a share is free | Needs a GPU with no shard jobs on it, so it may wait longer |
| Best for | Notebooks, development, inference, small to medium training | Large training runs, benchmarks, anything that needs more than about 48 GB |

A job with shards sees exactly one GPU, as device `0` (`CUDA_VISIBLE_DEVICES=0`). Your job runs on one physical GPU, and its other shards are used by other people's jobs.

**What shards do *not* do.** A shard is a scheduling quota, not a hardware limit:

- **GPU memory is not enforced.** If your job allocates more than its share, the other jobs on that card can run out of memory and crash with a CUDA out-of-memory error, or you can. Keep your usage under about 24 GB per shard you requested, and request more shards if you need more memory.
- **Compute is shared.** If other jobs are busy on the same GPU, yours will run slower than on a whole GPU.
- A whole-GPU request (`gpu:blackwell:1`) can only start on a GPU that has no shard jobs on it, so it may wait while shard jobs are running.

> Setting `CUDA_VISIBLE_DEVICES` yourself is not needed and can make your job use the wrong GPU. Leave it as Slurm sets it.

### Which one should I use for Jupyter?

- **If your notebook needs a GPU, use a shared one.** A notebook sits idle much of the time, so it does not need a whole card to itself. (If your code runs fine on CPU, use a CPU node instead; see the [GPU Usage Policy](#gpu-usage-policy-important).) Start with `shard:1` (about 24 GB), or `shard:2` (about 48 GB) if your model needs more.
- **Use a whole GPU only if you need it:** your job needs more than about 48 GB of GPU memory, or you are timing or benchmarking and need no neighbours.
- Whichever you choose, keep your GPU memory use within what you asked for, and **shut your notebook down when you are done** so the GPU is free for others.

Ready-to-paste headers for both are in [Running Jupyter on a GPU Node](#running-jupyter-on-a-gpu-node). For GPU usage policy and more examples, see the [HPC Server User Guide](../guide/#7-whole-gpu-or-a-share-of-a-gpu).

---

## GPU Usage Policy (Important)

GPU nodes are limited and high-demand resources.

Please follow these guidelines:

- **Do NOT submit to GPU partitions unless your code explicitly requires CUDA and requires very high memory**
- If your workload runs on CPU, use:
```
--partition=cpus
```
- If you do need a GPU for a notebook, prefer a **shared** GPU (`--gres=shard:N`) over a whole one. See [Whole GPU or a Share of a GPU?](#whole-gpu-or-a-share-of-a-gpu).

---

## How Jupyter Runs on Rosalind
This section explains how to launch a Jupyter Lab session on the Rosalind cluster
using Slurm and connect to it from your local machine.

The workflow is:

1. SSH into `rosalind-login`
2. Submit a Slurm job
3. Create an SSH tunnel from your laptop
4. Open Jupyter in your browser

---

## Step 1: SSH Into Rosalind

From your local machine:

```bash
ssh <username>@rosalind-login
```
Replace `<username>` with your Vanderbilt username.

---

## Step 2: Create the SBATCH Script

On `rosalind-login`, create a file (protip: name it `run_jupyter.sbatch` and keep in `batch_scripts` folder) and copy the following:


script you can copy and adapt to run
Jupyter Lab on the Rosalind cluster. The snippet is embedded below with
comments explaining what to edit. Copy the block into a file on the login
node (for example `run_jupyter.sbatch`), update the SBATCH headers, then
submit with `sbatch run_jupyter.sbatch /path/to/project`.


Paste the following:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=jupyter_cpu
#SBATCH --partition=cpus
#SBATCH --time=04:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=32G
#SBATCH --output=logs/%x_%j.out
#SBATCH --mail-type=END,FAIL

set -euo pipefail

# -----------------------------
# Usage: sbatch jupyter_cpu.sbatch /path/to/workdir
# -----------------------------
TARGET_DIR="${1:-$PWD}"

mkdir -p logs

# Pick a random high port (avoid common collisions)
PORT="$(shuf -i 10000-20000 -n 1)"
NODE="$(hostname -s)"
USER="$(whoami)"

# -----------------------------
# Generate token ourselves so it ALWAYS appears in the SLURM log
# -----------------------------
export PYTHONUNBUFFERED=1

TOKEN="$(python - <<'PY'
import secrets
print(secrets.token_hex(24))
PY
)"

echo ""
echo "============================================================"
echo "Jupyter will run on compute node: ${NODE}"
echo "Port: ${PORT}"
echo ""
echo "From your LOCAL machine, run (in a new terminal):"
echo "  ssh -N -L ${PORT}:127.0.0.1:${PORT} -J ${USER}@rosalind-login ${USER}@${NODE}"
echo ""
echo "Then open your browser at:"
echo "  http://127.0.0.1:${PORT}/lab?token=${TOKEN}"
echo "Token:"
echo "  ${TOKEN}"
echo "============================================================"
echo ""

CONDA="${HOME}/miniforge3/bin/conda"
# Start Jupyter; keep port fixed; send logs to the SLURM output

cd "${TARGET_DIR}"
echo -e "\n\nRunning JupyterLab...\n\n"

# Bind to localhost for safety on shared nodes; access via SSH tunnel above.
"${CONDA}" run -n spatial \
  jupyter lab --no-browser --port="${PORT}" --ip=127.0.0.1 \
  --ServerApp.token="${TOKEN}" \
  --ServerApp.password="" \
  --ServerApp.port_retries=0 \
  --ServerApp.log_level=INFO \
  2>&1 | tee -a "logs/jupyter_${USER}_${PORT}.log"
```
Make sure to update the SBATCH headers according to your needs (partition, time, resources).
Then submit the job:

```bash
sbatch run_jupyter.sbatch /path/to/your/project
```

Check the job status with `squeue -u <username>` and wait for it to start running.

## Step 3: Create SSH Tunnel
Open a **new terminal** on your local machine and run the command printed in the job output:

```
ssh -N -L <PORT>:<COMPUTE_NODE>:<PORT> <username>@rosalind-login
```
Replace `<PORT>`, `<COMPUTE_NODE>`, and `<username>` with the values from your job output.
Then open your browser:
```
http://localhost:<PORT>
```

## Running Jupyter on a GPU Node

Use the **same script as in [Step 2](#step-2-create-the-sbatch-script)** and the same steps to connect. The only change is the `#SBATCH` header at the top: request either a shared GPU or a whole GPU, as described in [Whole GPU or a Share of a GPU?](#whole-gpu-or-a-share-of-a-gpu). The Jupyter Lab session then runs on the GPU node, and you can use it to run GPU-accelerated code.

### Shared GPU (recommended for notebooks)

```bash
#!/usr/bin/env bash
#SBATCH --job-name=jupyter_gpu_shared
#SBATCH --partition=gpus             # debug-gpu (2h), gpus (12h)
#SBATCH --gres=shard:2               # a share of a GPU: shard:1 ~ 24 GB, shard:2 ~ 48 GB
#SBATCH --time=04:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --output=logs/%x_%j.out
```

### Whole GPU

```bash
#!/usr/bin/env bash
#SBATCH --job-name=jupyter_gpu
#SBATCH --partition=debug-gpu        # debug-gpu (2h), gpus (12h)
#SBATCH --gres=gpu:blackwell:1       # 1 whole Blackwell GPU, exclusively yours
#SBATCH --time=02:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=32G
#SBATCH --output=logs/%x_%j.out
```

Everything below the header (the part that picks a port, creates the token and starts Jupyter Lab) is the same as in Step 2. Submit and connect the same way:

```bash
sbatch run_jupyter.sbatch /path/to/your/project
```
