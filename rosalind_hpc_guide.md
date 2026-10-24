---
layout: page
title: Rosalind HPC Server Guide
permalink: /guide/
---

# Rosalind HPC Server — User Guide

**Audience:** New lab members  
**Tone:** Professional, practical, collaborative

---

## Table of Contents

1. [System Architecture](#1-system-architecture)
2. [Getting Started: Login and Environment Setup](#2-getting-started-login-and-environment-setup)
3. [Filesystem Conventions](#3-filesystem-conventions)
4. [SLURM Basics](#4-slurm-basics)
5. [Requesting Resources Responsibly](#5-requesting-resources-responsibly)
6. [GPU Usage Policy](#6-gpu-usage-policy)
7. [Testing Jobs Before Scaling](#7-testing-jobs-before-scaling)
8. [Storage Best Practices](#8-storage-best-practices)
9. [Monitoring and Managing Jobs](#9-monitoring-and-managing-jobs)
10. [Troubleshooting](#10-troubleshooting)

---

## 1. System Architecture

Rosalind is a shared high-performance computing (HPC) cluster. Understanding its layout helps you use it responsibly.

### Nodes

| Node type | Purpose | What you should do here |
|-----------|---------|------------------------|
| **Login node** | Entry point for all users | Edit files, submit jobs, light scripting |
| **CPU compute nodes** | General-purpose computation | Most bioinformatics, preprocessing, classical ML |
| **GPU compute nodes** | Accelerated computation | Large deep learning training, heavy spatial-omics modeling, large-scale simulation |

> **Important:** Never run compute-intensive work directly on the login node. All non-trivial work must go through SLURM. The login node is small (16 CPU threads) and shared by everyone.

### Hardware

| Node(s) | Count | CPUs | CPU RAM | GPUs |
|---------|-------|------|---------|------|
| `ROSALIND-CPU1`, `ROSALIND-CPU2` | 2 | 384 threads (2 × 96-core AMD EPYC 9645) | ~770 GB | none |
| `ROSALIND-GPU1`, `ROSALIND-GPU2` | 2 | 384 threads (2 × 96-core AMD EPYC 9645) | ~1 TB | 2 × NVIDIA RTX PRO 6000 Blackwell Server Edition (~96 GB each) |

That is 4 GPUs in total. Each compute node also has a local ~880 GB disk at `/tmp`.

### Partitions

Rosalind organizes nodes into *partitions* (queues):

| Partition      | Max Time     | Nodes           | Intended Use |
|---------------|-------------|----------------|--------------|
| `debug`       | 2 hours     | CPU nodes      | Quick testing, debugging (highest priority) |
| `cpus`        | 12 hours    | CPU nodes      | Standard CPU workloads (**default**) |
| `batch`       | 7 days      | CPU nodes      | Long CPU runs (lowest priority) |
| `debug-gpu`   | 2 hours     | GPU nodes      | Short GPU testing (highest priority) |
| `gpus`        | 12 hours    | GPU nodes      | Standard GPU workloads |
| `batch-gpu`   | 7 days      | GPU nodes      | Long GPU runs (lowest priority) |

Partition names are exactly as shown. There is no `cpu` or `gpu` partition: use `cpus` and `gpus`.

Check partitions and node state with:

```bash
sinfo -s          # partition summary
sinfo -N -l       # one line per node, with state and drain reason
```

**Per-user limits.** Slurm QOS limits apply: the `normal` QOS caps a user at 240 CPUs at once, and the `priority` QOS at 480. Check yours with `sacctmgr show assoc user=$USER format=Account,QOS`.

### Shared Storage

Your login directory is `/home/NAShome/<username>/` (the `$HOME` variable points there). It sits on the shared network storage and is visible from the login node and all compute nodes. Other mounts:

| Path | Description | Use for |
|------|-------------|---------|
| `/home/NAShome/<username>/` | Your home directory on the shared network storage (~97 TB pool shared by everyone) | Scripts, conda environments, small config files |
| `/Polaris/<LabName>/` (one directory per lab) | Per-lab network storage, large | Datasets, results, project files for that lab |
| `/mnt/launas/` | Lau lab NAS (network, large) | Lab archives and shared data |
| `/mnt/htan/` | HTAN data share (SMB/CIFS mount) | HTAN datasets |
| `/home/NAS/` | Older shared NAS (~7 TB) | Legacy data; ask before adding new data |
| `/tmp` (on each compute node) | Fast local NVMe disk (~880 GB per node) | Temporary I/O during a job |

> Check current free space at any time with `df -h /home/NAShome /Polaris/<LabName>`. These are network filesystems, so heavy small-file I/O from many jobs at once will be slow.

---

## 2. Getting Started: Login and Environment Setup

### Logging In

```bash
ssh <username>@rosalind-login   # replace with actual hostname
```

Use SSH key authentication when possible to avoid repeated password prompts.

### Activating Conda Environments

Rosalind uses [conda](https://docs.conda.io/) for software environment management.

```bash
# List available environments
conda env list

# Activate an environment
conda activate <env_name>

# Deactivate
conda deactivate
```

**Always activate your environment before submitting jobs.** When running batch jobs, activate the environment explicitly inside your job script (see [Section 4](#4-slurm-basics)).

### First-time Setup Recommendations

- Add your SSH public key to `~/.ssh/authorized_keys` for passwordless login.
- Initialize conda in your shell profile (only needs to be done once):
  ```bash
  conda init bash   # or zsh, etc.
  ```
- Set up `~/.bashrc` or `~/.bash_profile` to activate your default environment, if needed.

---

## 3. Filesystem Conventions

```
/home/NAShome/<username>/    Personal home — scripts, envs, config
/Polaris/<LabName>/          Lab storage — datasets, results, project files
/Polaris/<LabName>/<project>/  Per-project directories (coordinate with lab)
/mnt/launas/, /mnt/htan/     Group data shares (see Section 1)
/tmp/ (on compute node)      Fast local scratch — use for temporary I/O during jobs
```

**Conventions:**

- Keep large raw data in your lab's `/Polaris/<LabName>/` directory, not in your home.
- Do not store large intermediate files in your home; the home pool is shared by everyone and filling it affects all users.
- Everything under `/home`, `/Polaris` and `/mnt` is network storage. Jobs doing millions of small reads and writes should work in `/tmp` on the compute node and copy results back at the end.
- Clean up `/tmp` scratch files after your job finishes.

---

## 4. SLURM Basics

SLURM is the job scheduler that allocates cluster resources fairly among all users.

### Interactive Session (`srun`)

Use `srun` when you need an interactive shell on a compute node — for example, for debugging.

```bash
# Short interactive CPU session (30 min, 4 CPUs, 8 GB RAM)
srun --partition=cpus --cpus-per-task=4 --mem=8G --time=00:30:00 --pty bash
```

### Batch Job (`sbatch`)

For production runs, write a job script and submit it with `sbatch`.

**Example batch script (`run_analysis.sh`):**

```bash
#!/bin/bash
#SBATCH --job-name=my_analysis
#SBATCH --partition=cpus
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --output=logs/%j_out.txt
#SBATCH --error=logs/%j_err.txt

# Activate environment
conda activate my_env

# Run your analysis
python run_pipeline.py --input /Polaris/<LabName>/project/input --output /Polaris/<LabName>/project/output
```

> The `logs/` directory must exist **before** you submit (`mkdir -p logs`). Slurm will not create it, and a job that cannot open its log file fails.

Submit the script:

```bash
sbatch run_analysis.sh
```

### Key `#SBATCH` Directives

| Directive | Meaning | Example |
|-----------|---------|---------|
| `--partition` | Which partition/queue to use | `--partition=cpus` |
| `--cpus-per-task` | Number of CPU threads | `--cpus-per-task=8` |
| `--mem` | Total memory for the job (**always set this**, see Section 5) | `--mem=32G` |
| `--time` | Maximum wall-clock time (HH:MM:SS) | `--time=04:00:00` |
| `--gres=gpu:blackwell:N` | N whole GPUs, exclusive to your job (GPU partitions only) | `--gres=gpu:blackwell:1` |
| `--gres=shard:N` | N *shares* of a GPU, shared with other jobs (GPU partitions only, see Section 6) | `--gres=shard:2` |
| `--output` / `--error` | Log file paths | `--output=logs/%j.out` |
| `--job-name` | Label your job | `--job-name=preprocess` |

---

## 5. Requesting Resources Responsibly

> **This is the single most impactful thing you can do for the lab and the cluster.**

### Why Over-Requesting Hurts Everyone

Nodes are shared: SLURM gives each job the CPU threads, memory and GPUs it asked for, and other jobs can use the rest of the node. But whatever you request is reserved for you. If you request 64 CPUs and only use 4, those 60 CPUs sit idle and **nobody else can use them**. This:

- Increases wait time for all lab members
- Wastes expensive shared hardware
- May cause your job to wait longer in the queue (SLURM prioritizes jobs that fit available slots)

### How to Estimate What You Need

1. **Run a small test first** (see [Section 7](#7-testing-jobs-before-scaling)).
2. Check peak memory usage with `sacct` or `/usr/bin/time -v`:
   ```bash
   /usr/bin/time -v python myscript.py 2>&1 | grep "Maximum resident"
   ```
3. Add a ~20–25% buffer above your measured peak — not 2–10×.

### Good vs. Bad Requests

#### ❌ Bad: Over-requesting resources

```bash
#SBATCH --cpus-per-task=64    # script uses 4 threads
#SBATCH --mem=500G            # actual usage is ~20 GB
#SBATCH --time=7-00:00:00     # job finishes in 2 hours
#SBATCH --gres=gpu:blackwell:2  # both GPUs of a node, but only one is used
```

This blocks 64 CPUs, 500 GB RAM, and both GPUs of a node for a week — for everyone else on the cluster.

#### ✅ Good: Right-sized request

```bash
#SBATCH --cpus-per-task=4     # matches script's actual thread count
#SBATCH --mem=25G             # peak usage ~20 GB + small buffer
#SBATCH --time=03:00:00       # prior test run took ~2 hours
```

### Always Set `--mem`

If you do not set `--mem` (or `--mem-per-cpu`), your job is given a default of about 2.7 GB per CPU thread you requested. That default is a safe fallback, not a good estimate of what your job needs: a job that needs more will be killed with an out-of-memory error. Set `--mem` explicitly in every script.

### Memory Guidelines

| Workload type | Typical memory range |
|---------------|----------------------|
| Light preprocessing / parsing | 4–16 GB |
| Standard bioinformatics tools | 16–64 GB |
| Large genome assembly / alignment | 64–256 GB |
| Deep learning training (CPU side) | 32–128 GB |

### CPU / Thread Guidelines

- Set `--cpus-per-task` to match the number of threads your tool actually uses.
- Most tools expose a `--threads` or `-t` argument — match it to your `--cpus-per-task`.
- Multi-node parallelism (MPI) is rarely needed for typical bio/ML workloads; use single-node multi-threading instead.

---

## 6. GPU Usage Policy

### GPUs Are a Premium Shared Resource

Rosalind's GPUs are large, high-end accelerators shared across the entire lab. They are intended **only** for workloads that genuinely require GPU acceleration — workloads where the GPU provides substantial speedup that a CPU cannot reasonably provide.

**Appropriate GPU workloads:**
- Large-scale deep learning model training (e.g., foundation models, large transformers, diffusion models)
- Heavy spatial-omics or single-cell deep learning pipelines (e.g., scVI, Tangram, DeepST at scale)
- Large-scale simulation or molecular dynamics with GPU acceleration
- Production training runs that have already been tested and tuned

**Inappropriate GPU workloads — use CPU instead:**
- Data preprocessing, format conversion, quality control
- Classical ML models (random forest, SVM, logistic regression, XGBoost on typical datasets)
- Small neural network experiments or proof-of-concept runs
- Debugging or iterating on model code
- Running inference on small batches
- Any workload where a CPU run would complete in under 30 minutes

### For Lightweight Experimentation and Prototyping

If you are exploring, prototyping, or teaching and need a GPU for interactive iteration, **use an external resource instead of Rosalind's GPU nodes:**

- [Google Colab](https://colab.research.google.com/) — free GPU tier suitable for small experiments and tutorials
- [Kaggle Kernels](https://www.kaggle.com/code) — free GPU access for notebooks
- Lab-owned smaller workstations (if available)

These platforms are better suited for experimentation and free up Rosalind's GPUs for production-scale jobs that truly need them.

### Whole GPU or a Share of a GPU?

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

A job with shards sees exactly one GPU, as device `0` (`CUDA_VISIBLE_DEVICES=0`). Your job runs on one physical GPU, and its other shards are used by other people's jobs.

**What shards do *not* do.** A shard is a scheduling quota, not a hardware limit:

- **GPU memory is not enforced.** If your job allocates more than its share, the other jobs on that card can run out of memory and crash with a CUDA out-of-memory error, or you can. Keep your usage under about 24 GB per shard you requested, and request more shards if you need more memory.
- **Compute is shared.** If other jobs are busy on the same GPU, yours will run slower than on a whole GPU.
- A whole-GPU request (`gpu:blackwell:1`) can only start on a GPU that has no shard jobs on it, so it may wait while shard jobs are running.

```bash
#SBATCH --partition=gpus
#SBATCH --gres=shard:2          # about half of a GPU (~48 GB)
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
```

> Setting `CUDA_VISIBLE_DEVICES` yourself is not needed and can make your job use the wrong GPU. Leave it as Slurm sets it.

### GPU Request Examples

#### ❌ Bad: Requesting a GPU for work that doesn't need it

```bash
#SBATCH --partition=gpus
#SBATCH --gres=gpu:blackwell:1
#SBATCH --mem=64G

# Running this on a GPU that completes in 3 minutes on CPU:
python preprocess_counts.py --input raw.h5ad --output processed.h5ad
```

#### ❌ Bad: Requesting both GPUs of a node for a single-GPU workload

```bash
#SBATCH --gres=gpu:blackwell:2    # your training script only uses one GPU
```

#### ✅ Good: A share of a GPU for a notebook or small model

```bash
#SBATCH --partition=gpus
#SBATCH --gres=shard:1            # ~24 GB of GPU memory, shared
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
```

#### ✅ Good: Justified whole-GPU request for a large training run

```bash
#SBATCH --partition=gpus
#SBATCH --gres=gpu:blackwell:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=64G
#SBATCH --time=12:00:00

conda activate deep_learning_env
python train_model.py \
    --model large_transformer \
    --data /Polaris/<LabName>/project/train_set \
    --epochs 100 \
    --batch_size 256
```

### Before Requesting a GPU, Ask Yourself

1. Does my code explicitly use CUDA / GPU acceleration?
2. Have I profiled or tested on CPU — is the GPU truly necessary for this run?
3. Am I running a final/production job, or am I still debugging?
4. Could I run this on Google Colab or another external resource instead?

If the answer to (1) or (2) is no, or if you are still in the debugging stage, **use a CPU node**.

### Collaborative Responsibility

Our GPUs are expensive and finite. When you hold a GPU node idle or run a trivial job on it, colleagues with legitimate GPU-intensive workloads may be blocked for hours. We rely on everyone's good judgment to keep the cluster usable for the whole lab.

---

## 7. Testing Jobs Before Scaling

Before submitting a large production job, always validate with a small test:

1. **Subset your input:** Take a small slice of your data (e.g., first 1,000 rows, one chromosome, one sample).
2. **Run interactively on CPU first:**
   ```bash
   srun --partition=cpus --cpus-per-task=4 --mem=16G --time=00:30:00 --pty bash
   conda activate my_env
   python my_pipeline.py --input /Polaris/<LabName>/project/tiny_test --output /tmp/test_out
   ```
3. **Profile resource usage** using `htop` (in another terminal) or `/usr/bin/time -v`.
4. **Estimate wall time** by extrapolating from your test run.
5. **Scale up:** Adjust your `sbatch` directives to match measured needs plus a buffer.

This practice catches bugs early, prevents wasted queue time, and gives you accurate resource estimates.

---

## 8. Storage Best Practices

### Use the Right Storage Tier

| Storage | Best for | Notes |
|---------|----------|-------|
| `/home/NAShome/<username>` | Scripts, conda envs, config | Shared pool for all users — do not store large data here |
| `/Polaris/<LabName>` | Raw data, results, shared files | Large lab storage; treat as long-term lab storage |
| `/tmp` on compute node | Temporary I/O during a job | Fast local disk; **clean up after your job** |

### Avoid Unnecessary Duplication

- **Do not copy large raw datasets** into your personal directories; symlink to the lab directory instead:
  ```bash
  ln -s /Polaris/<LabName>/shared/genomes/hg38 ~/refs/hg38
  ```
- Coordinate with labmates to share reference files (genome indices, model weights) rather than each person downloading their own copy.
- When a project is complete or paused, archive or compress large intermediate files.

### Scratch Disk Usage

For jobs with heavy intermediate I/O, use the compute node's local `/tmp`:

```bash
# In your job script
TMPDIR=/tmp/${SLURM_JOB_ID}
mkdir -p $TMPDIR

# ... run your pipeline using $TMPDIR for intermediates ...

# Clean up at the end
rm -rf $TMPDIR
```

### Checking Your Usage

```bash
# Check home directory usage
du -sh ~/

# Check a project directory
du -sh /Polaris/<LabName>/project/

# Check free space on the shared storage
df -h /home/NAShome /Polaris/<LabName>
```

---

## 9. Monitoring and Managing Jobs

### Checking the Queue

```bash
# See all your jobs
squeue -u $USER

# See all jobs in a partition
squeue -p cpus

# Compact summary of all jobs
squeue -a
```

### Viewing Completed Job Stats

```bash
# Summary of recent jobs (last day)
sacct -u $USER --starttime=$(date -d "1 day ago" +%Y-%m-%d) \
      --format=JobID,JobName,Partition,State,Elapsed,MaxRSS,ReqMem,CPUTime

# Detailed info for a specific job
sacct -j <JOBID> --format=JobID,MaxRSS,Elapsed,ExitCode
```

`MaxRSS` shows peak memory usage — use it to right-size future jobs.

### Monitoring Running Jobs

```bash
# Connect to a compute node where your job is running
squeue -u $USER         # note the NODELIST column
ssh <node_name>         # then on that node:
htop -u $USER           # real-time CPU/memory view
```

### Cancelling Jobs

```bash
# Cancel a specific job
scancel <JOBID>

# Cancel all your jobs
scancel -u $USER
```

### Estimating Queue Wait Time

```bash
# Show expected start time for pending jobs
squeue -u $USER --start
```

---

## 10. Troubleshooting

### Job Fails Immediately

- Check your error log: `cat logs/<JOBID>_err.txt`
- Common causes: conda environment not activated, wrong file path, typo in script.
- Test the command interactively first with `srun`.

### Out-of-Memory (OOM) Error

```
slurmstepd: error: Detected 1 oom-kill event(s)
```

- Increase `--mem` in your script.
- Profile with `sacct -j <JOBID> --format=MaxRSS` on a previous run to see peak usage.

### Job Stuck in Queue (`PENDING`)

```bash
squeue -u $USER -o "%.18i %.9P %.20j %.8u %.8T %.10M %.9l %.6D %R"
```

The **REASON** column explains why (e.g., `Resources`, `Priority`, `QOSMaxWallDurationPerJobLimit`).

Common fixes:
- Reduce resource requests (especially memory, GPUs, or walltime) to improve scheduling.
- Check if the partition is fully occupied: `sinfo -p <partition>`.

### CUDA / GPU Not Found

- Confirm you are on a GPU partition (`debug-gpu`, `gpus` or `batch-gpu`): `squeue -u $USER` — check the `PARTITION` column.
- Make sure your script requests a GPU: `--gres=gpu:blackwell:1` (whole GPU) or `--gres=shard:N` (a share of one).
- Check CUDA availability inside your job:
  ```python
  import torch
  print(torch.cuda.is_available())   # Should be True
  ```
- The GPUs are NVIDIA Blackwell cards. Use a PyTorch / TensorFlow / JAX build made for **CUDA 12.8 or newer**; older builds will not run kernels on these GPUs. The system `nvcc` is CUDA 12.0 and cannot compile for Blackwell, so install the CUDA toolkit in your conda environment if you need to compile CUDA code.

### GPU Out-of-Memory on a Shared GPU

If you use `--gres=shard:N` and see `CUDA out of memory` even though your own job is small, another job on the same GPU may be using more than its share. Check with `nvidia-smi` inside your job. If it keeps happening, request more shards or use a whole GPU (`--gres=gpu:blackwell:1`).

### "Invalid generic resource (gres) specification"

`sbatch` rejected your `--gres` value. Use exactly `--gres=gpu:blackwell:N` or `--gres=shard:N`, and submit from the login node.

### Conda Environment Not Found

```bash
conda env list    # verify the env exists
```

If the environment was created under a different user or path, check the full path and reference it explicitly:

```bash
conda activate /path/to/envs/my_env
```

### Getting Help

- Check SLURM documentation: `man sbatch`, `man squeue`, `man sacct`
- Consult the [SLURM documentation](https://slurm.schedmd.com/documentation.html)
- Ask in the lab Slack / discussion channel before opening a ticket — labmates may have seen the same issue

---

## Quick Reference Card

```
Login:          ssh <user>@rosalind-login
Interactive:    srun --partition=cpus --cpus-per-task=4 --mem=8G --time=00:30:00 --pty bash
Submit job:     sbatch my_job.sh
Partitions:     debug, cpus (default), batch  |  debug-gpu, gpus, batch-gpu
Whole GPU:      #SBATCH --gres=gpu:blackwell:1
Share of GPU:   #SBATCH --gres=shard:1   (~24 GB; shard:2 ~48 GB; shared)
Check queue:    squeue -u $USER
Cancel job:     scancel <JOBID>
Job stats:      sacct -j <JOBID> --format=JobID,MaxRSS,Elapsed,ExitCode
Node info:      sinfo -s
Disk usage:     du -sh /Polaris/<LabName>/myproject/
```

---

*Maintained by the Sarkar Lab. For corrections or additions, open a pull request or contact the lab's HPC point of contact.*
