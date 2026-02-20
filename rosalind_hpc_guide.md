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

> **Important:** Never run compute-intensive work directly on the login node. All non-trivial work must go through SLURM.

### Partitions

Rosalind organizes nodes into *partitions* (queues). Common partitions include:

- **cpu** — CPU-only nodes for general workloads (Mostly favored for typical bioinformatics and classical ML tasks)
- **gpu** — GPU nodes; request only when your workload genuinely needs GPU acceleration
- **interactive** (if available) — short interactive sessions for debugging

Check currently available partitions with:

```bash
sinfo -s
```

### Shared Storage

| Path | Description | Use for |
|------|-------------|---------|
| `/home/<username>` | Your personal home directory (limited quota) | Scripts, conda environments, small config files |
| `/data` (NAS) | Shared lab network-attached storage (larger quota) | Datasets, results, shared project files |

---

## 2. Getting Started: Login and Environment Setup

### Logging In

```bash
ssh <username>@rosalind.example.edu   # replace with actual hostname
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
/home/<username>/          Personal home — scripts, envs, config (limited space)
/data/                     Shared NAS — datasets, results, project files
/data/<project>/           Per-project directories (coordinate with lab)
/tmp/ (on compute node)    Fast local scratch — use for temporary I/O during jobs
```

**Conventions:**

- Keep large raw data in `/data`, not in `/home`.
- Do not store large intermediate files in `/home`; you will hit quota limits and affect other users.
- Clean up `/tmp` scratch files after your job finishes.

---

## 4. SLURM Basics

SLURM is the job scheduler that allocates cluster resources fairly among all users.

### Interactive Session (`srun`)

Use `srun` when you need an interactive shell on a compute node — for example, for debugging.

```bash
# Short interactive CPU session (30 min, 4 CPUs, 8 GB RAM)
srun --partition=cpu --cpus-per-task=4 --mem=8G --time=00:30:00 --pty bash
```

### Batch Job (`sbatch`)

For production runs, write a job script and submit it with `sbatch`.

**Example batch script (`run_analysis.sh`):**

```bash
#!/bin/bash
#SBATCH --job-name=my_analysis
#SBATCH --partition=cpu
#SBATCH --cpus-per-task=8
#SBATCH --mem=32G
#SBATCH --time=04:00:00
#SBATCH --output=logs/%j_out.txt
#SBATCH --error=logs/%j_err.txt

# Activate environment
conda activate my_env

# Run your analysis
python run_pipeline.py --input /data/project/input --output /data/project/output
```

Submit the script:

```bash
sbatch run_analysis.sh
```

### Key `#SBATCH` Directives

| Directive | Meaning | Example |
|-----------|---------|---------|
| `--partition` | Which partition/queue to use | `--partition=cpu` |
| `--cpus-per-task` | Number of CPU cores | `--cpus-per-task=8` |
| `--mem` | Total memory for the job | `--mem=32G` |
| `--time` | Maximum wall-clock time (HH:MM:SS) | `--time=04:00:00` |
| `--gres=gpu:N` | Number of GPUs (GPU partition only) | `--gres=gpu:1` |
| `--output` / `--error` | Log file paths | `--output=logs/%j.out` |
| `--job-name` | Label your job | `--job-name=preprocess` |

---

## 5. Requesting Resources Responsibly

> **This is the single most impactful thing you can do for the lab and the cluster.**

### Why Over-Requesting Hurts Everyone

SLURM allocates resources exclusively. If you request 64 CPUs and only use 4, those 60 CPUs sit idle and **nobody else can use them**. This:

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
#SBATCH --gres=gpu:4          # only one GPU is used
```

This blocks 64 CPUs, 500 GB RAM, and 4 GPUs for a week — for everyone else on the cluster.

#### ✅ Good: Right-sized request

```bash
#SBATCH --cpus-per-task=4     # matches script's actual thread count
#SBATCH --mem=25G             # peak usage ~20 GB + small buffer
#SBATCH --time=03:00:00       # prior test run took ~2 hours
```

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

### GPU Request Examples

#### ❌ Bad: Requesting a GPU for work that doesn't need it

```bash
#SBATCH --partition=gpu
#SBATCH --gres=gpu:1
#SBATCH --mem=64G

# Running this on a GPU that completes in 3 minutes on CPU:
python preprocess_counts.py --input raw.h5ad --output processed.h5ad
```

#### ❌ Bad: Requesting multiple GPUs for a single-GPU workload

```bash
#SBATCH --gres=gpu:4    # your training script only uses one GPU
```

#### ✅ Good: Justified GPU request for a large training run

```bash
#SBATCH --partition=gpu
#SBATCH --gres=gpu:1
#SBATCH --cpus-per-task=8
#SBATCH --mem=64G
#SBATCH --time=12:00:00

conda activate deep_learning_env
python train_model.py \
    --model large_transformer \
    --data /data/project/train_set \
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
   srun --partition=cpu --cpus-per-task=4 --mem=16G --time=00:30:00 --pty bash
   conda activate my_env
   python my_pipeline.py --input /data/project/tiny_test --output /tmp/test_out
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
| `/home/<username>` | Scripts, conda envs, config | Strictly limited quota — do not store data here |
| `/data` (NAS) | Raw data, results, shared files | Large quota; treat as long-term lab storage |
| `/tmp` on compute node | Temporary I/O during a job | Fast local disk; **clean up after your job** |

### Avoid Unnecessary Duplication

- **Do not copy large raw datasets** into your personal directories; symlink to `/data` instead:
  ```bash
  ln -s /data/shared/genomes/hg38 ~/refs/hg38
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
du -sh /data/project/

# Check overall NAS usage (if available)
df -h /data
```

---

## 9. Monitoring and Managing Jobs

### Checking the Queue

```bash
# See all your jobs
squeue -u $USER

# See all jobs in a partition
squeue -p cpu

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

- Confirm you are on the GPU partition: `squeue -u $USER` — check `PARTITION` column.
- Make sure `--gres=gpu:N` is in your job script.
- Check CUDA availability inside your job:
  ```python
  import torch
  print(torch.cuda.is_available())   # Should be True
  ```

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
Login:          ssh <user>@rosalind.example.edu
Interactive:    srun --partition=cpu --cpus-per-task=4 --mem=8G --time=00:30:00 --pty bash
Submit job:     sbatch my_job.sh
Check queue:    squeue -u $USER
Cancel job:     scancel <JOBID>
Job stats:      sacct -j <JOBID> --format=JobID,MaxRSS,Elapsed,ExitCode
Node info:      sinfo -s
Disk usage:     du -sh /data/myproject/
```

---

*Maintained by the Sarkar Lab. For corrections or additions, open a pull request or contact the lab's HPC point of contact.*
