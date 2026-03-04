---
layout: page
title: Running Jupyter Notebooks on Rosalind
permalink: /jupyter/
---

# Rosalind Partitions and Usage Guidelines

Rosalind provides multiple Slurm partitions designed for different workloads.
Please choose the appropriate partition based on your job size and runtime.

## Available Partitions

| Partition      | Max Time     | Hardware        | Intended Use |
|---------------|-------------|----------------|--------------|
| `debug`       | 2 hours     | CPU nodes      | Quick testing, debugging |
| `cpus`        | 12 hours    | CPU nodes      | Standard CPU workloads (default) |
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

- 2 NVIDIA Blackwell GPUs  
- 384 CPU cores
- 770 GB CPU RAM
- 200 GB GPU RAM  

To request a GPU:

```
#SBATCH --partition=debug-gpu
#SBATCH --gres=gpu:blackwell:1
```
--

### Quick checks 
You can quickly check if you can run on CPU or GPU nodes by running:

```bash
sbatch -p debug --wrap="hostname; sleep 120" --cpus-per-task=32 --mem=10G
```
For GPU nodes:

```bash
sbatch -p debug-gpu --gres=gpu:blackwell:1 --wrap="nvidia-smi -L; sleep 120"
```
# ⚠️ Current CPU2 problem
Our /home directory is currently on CPU2, which means if you run too many CPU jobs on CPU2, you may experience slow performance due to I/O bottlenecks. Please monitor your jobs and avoid overloading CPU2.

# ⚠️ GPU Usage Policy (Important)

GPU nodes are limited and high-demand resources.

Please follow these guidelines:

- **Do NOT submit to GPU partitions unless your code explicitly requires CUDA and requires very high memory**
- If your workload runs on CPU, use:
```
--partition=cpus
```

---

# Running Jupyter Notebooks on Rosalind
This page explains how to launch a Jupyter Lab session on the Rosalind cluster
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

## For GPU Nodes
Follow the same steps but make sure to request a GPU in your SBATCH script as shown above. The Jupyter Lab session will run on the GPU node, and you can use it to run GPU-accelerated code.

```bash
#!/usr/bin/env bash
#SBATCH --job-name=jupyter_gpu
#SBATCH --partition=debug-gpu        # debug-gpu (2h), gpus (12h), batch-gpu (7d)
#SBATCH --gres=gpu:blackwell:1       # Request 1 Blackwell GPU
#SBATCH --time=02:00:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=16
#SBATCH --mem=32G
#SBATCH --output=logs/%x_%j.out

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

Again submit with the following command and follow the same steps to connect.

```bash
sbatch run_jupyter.sbatch /path/to/your/project
```