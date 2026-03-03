---
layout: page
title: Installing Conda on Rosalind
permalink: /conda/
---

# Miniforge + JupyterLab Setup (Linux)

This guide explains how to:

1. Install `Miniforge3-Linux-x86_64.sh`
2. Create a conda environment
3. Install `pandas`, `jupyterlab`, etc.
4. Run JupyterLab on a remote server
5. (Optional) Run JupyterLab inside a Slurm job

---

# 1. Install Miniforge

```bash
cd ~

# Download installer
wget -O Miniforge3-Linux-x86_64.sh \
  https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-Linux-x86_64.sh

# Run installer
bash Miniforge3-Linux-x86_64.sh
```

During installation:
- Accept license
- Choose install location (default is fine)
- Say **yes** to "initialize conda"

Restart your shell:

```bash
exec $SHELL
```

Verify:

```bash
which conda
conda --version
```

---

# 2. Configure Conda (Recommended)

```bash
conda config --add channels conda-forge
conda config --set channel_priority strict
conda config --set auto_activate_base false
```

Open a new shell or run:

```bash
exec $SHELL
```

---

# 3. Create Jupyter Environment

```bash
conda create -n test_jupyter python=3.12 -y
conda activate test_jupyter
```

Install core packages:

```bash
conda install -y mamba

mamba install -y \
  pandas jupyterlab
```

Test installation:

```bash
python -c "import pandas as pd; print(pd.__version__)"
```

Test is in a slurm job:

```bash
sbatch <<'EOT'
#!/usr/bin/env bash
#SBATCH --job-name=test
#SBATCH --partition=cpus
#SBATCH --time=00:10:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=4
#SBATCH --mem=16G
#SBATCH --output=logs/%x_%j.out

set -euo pipefail

# Ensure logs directory exists (since --output points there)
mkdir -p logs

CONDA="${HOME}/miniforge3/bin/conda"

# Run in the env without activating (shell-agnostic)
"$CONDA" run -n test_jupyter python -c "import pandas as pd; print(pd.__version__)"
EOT
```
Check output with `squeue -u <username>` and `cat logs/test_<jobid>.out`.


Register kernel for Jupyter:

```bash
python -m ipykernel install --user --name test_jupyter --display-name "Python (test_jupyter)"
```

A full check for Jupyter-lab from the test jupyter via a batch file 
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
"${CONDA}" run -n test_jupyter \
  jupyter lab --no-browser \
  --port="${PORT}" --ip=127.0.0.1 \
  --ServerApp.token="${TOKEN}" \
  --ServerApp.password="" \
  --ServerApp.port_retries=0 \
  --ServerApp.log_level=INFO \
  2>&1

```
 Then do the following 
```bash
cat logs/logs/jupyter_cpu_105.out
```
and follow the steps listed in the output to connect to JupyterLab.

For detailed instructions, visit the [Jupyter setup guide](/jupyter/).