---
layout: page
title: Running R on Rosalind
permalink: /r/
---

# Running R on Rosalind

This page explains how to set up R in its own conda environment and run R
scripts through Slurm, without conflicting with other environments.

It assumes you have already installed Miniforge (see
[Installing Conda on Rosalind](../conda/)).

The workflow is:

1. Create an R environment (once)
2. Test it interactively on the login node
3. Test it as a Slurm batch job

---

## Step 1: Create the R Environment

Do this **once**, from the login node.

```bash
conda config --set channel_priority strict   # once per account

conda create -n r_test_env -c conda-forge -c bioconda \
  "r-base>=4.5,<4.6" r-tidyverse r-essentials r-data.table
```

Rules that avoid version conflicts:

- **One environment per project.** Never install R into `base`.
- **Install everything in one `conda create` command.** Conda only guarantees
  consistency among the packages it solves together. Adding `r-base` first and
  `r-essentials` later is how most conflicts start.
- **Pin the R minor series, not the patch.** `r-base>=4.5,<4.6` lets conda pick
  the newest 4.5.x that all your packages support. Quote the spec, otherwise the
  shell treats `<` and `>` as redirects.
- **Use `conda-forge` (plus `bioconda` for Bioconductor).** Do not mix in the
  `defaults` channel.
- **Do not also run `module load R`** in the same session or job script.

> **Why R 4.5 and not the newest R?** R 4.6 is on conda-forge, but packages such
> as `r-tidyverse`, `r-ggplot2` and the Bioconductor packages were not yet built
> for it when this page was written. With a pinned `r-base>=4.5,<4.6` you get
> R 4.5.x with all of them. Before changing the pin, check which R series a
> package supports:
>
> ```bash
> conda search -c conda-forge r-tidyverse --info | grep -E "^(version|  - r-base)"
> ```
>
> Look for lines like `r-base >=4.5,<4.6.0a0`.

### Adding packages later

Always dry-run first:

```bash
conda install -n r_test_env <package> --dry-run
```

If the output changes `r-base` or removes packages, stop and create a new
environment with the full package list instead. Roll back with
`conda list -n r_test_env --revisions` and
`conda install -n r_test_env --revision <N>`.

Prefer the conda build of a package (`r-<name>` in lowercase, or
`bioconductor-<name>` on bioconda) over `install.packages()`. Use
`install.packages()` only for packages that conda does not have, and only with
the environment activated.

### Installing Bioconductor packages via bioconda

Bioconductor packages are not on CRAN, so `r-<name>` will not find them. They
are available through the `bioconda` channel as `bioconductor-<name>`, with the
name in **lowercase**. For example:

| In R (`library(...)`) | conda package name |
|---|---|
| `DESeq2` | `bioconductor-deseq2` |
| `edgeR` | `bioconductor-edger` |
| `GenomicRanges` | `bioconductor-genomicranges` |
| `SingleCellExperiment` | `bioconductor-singlecellexperiment` |
| `ComplexHeatmap` | `bioconductor-complexheatmap` |

**1. Make sure the channels are set up (once):**

```bash
conda config --add channels bioconda
conda config --add channels conda-forge
conda config --set channel_priority strict
```

The order matters: `conda-forge` should have higher priority than `bioconda`.
You can also skip this and pass `-c conda-forge -c bioconda` on each command, as
in Step 1.

**2. Search for the package** to confirm the name and which R series it supports:

```bash
conda search -c conda-forge -c bioconda "bioconductor-deseq2" --info | grep -E "^(version|  - r-base)"
```

You can use a wildcard if you are unsure of the exact name:

```bash
conda search -c conda-forge -c bioconda "bioconductor-genomicr*"
```

Look for `r-base >=4.5,<4.6.0a0` in the output. If the package only lists other R
series, it is not built for your environment yet.

**3. Dry-run, then install.** Example with DESeq2:

```bash
conda install -n r_test_env -c conda-forge -c bioconda bioconductor-deseq2 --dry-run
```

Read the output. It should only add packages. If it changes `r-base` or removes
packages, stop and see "Adding packages later" above. Otherwise install:

```bash
conda install -n r_test_env -c conda-forge -c bioconda bioconductor-deseq2
```

**4. Check it loads:**

```bash
conda activate r_test_env
Rscript -e 'library(DESeq2); cat("DESeq2", as.character(packageVersion("DESeq2")), "\n")'
```

Installing several packages at once is better than one at a time, because the
solver then considers them together:

```bash
conda install -n r_test_env -c conda-forge -c bioconda \
  bioconductor-deseq2 bioconductor-edger bioconductor-genomicranges
```

If you know up front that you need Bioconductor packages, add them to the Step 1
`conda create` command instead, so the whole environment is solved in one go.

---

## Step 2: Test Interactively

On the login node, activate the environment and run a quick check:

```bash
conda activate r_test_env

Rscript -e 'cat(R.version.string, "\n"); print(.libPaths()); library(tidyverse); library(data.table); cat("OK\n")'
```

You should see:

- `R version 4.5.x`
- `.libPaths()` listing **only** paths inside your environment, for example
  `.../miniforge3/envs/r_test_env/lib/R/library`
- `OK` at the end

The messages about functions being masked (for example `dplyr::filter()` masks
`stats::filter()`) are normal.

If `.libPaths()` shows other paths such as `~/R/...`, a stray library is leaking
in. Fix it with:

```bash
export R_LIBS_USER=""
```

---

## Step 3: Test as a Slurm Batch Job

Create a file named `test_R_batch.sbatch` (protip: keep it in your
`batch_scripts` folder) and paste the following:

```bash
#!/usr/bin/env bash
#SBATCH --job-name=r_test
#SBATCH --partition=cpus
#SBATCH --time=00:05:00
#SBATCH --nodes=1
#SBATCH --ntasks=1
#SBATCH --cpus-per-task=2
#SBATCH --mem=4G
#SBATCH --output=logs/%x_%j.out

set -eo pipefail

# Make sure the logs directory exists (since --output points there)
mkdir -p logs

ENV_NAME="r_test_env"
MAMBA="${HOME}/miniforge3/bin/mamba"

# Activate the environment (the job script is bash, so hard-code bash)
eval "$("${MAMBA}" shell hook --shell bash)"
mamba activate "${ENV_NAME}"

# Sanity checks: fail early if the wrong R is picked up
echo "env: ${CONDA_DEFAULT_ENV}"
which Rscript
Rscript -e 'cat(R.version.string, "\n"); print(.libPaths()); library(tidyverse); library(data.table); cat("OK\n")'
```

Notes:

- Replace `ENV_NAME` with your own environment name. If it is empty, `mamba
  activate` silently activates `base` and R will not be found.
- `MAMBA` assumes Miniforge is installed in `~/miniforge3`, which is the default
  from the [conda install guide](../conda/).
- Use `--shell bash`, not `$SHELL`. Slurm passes on the `$SHELL` of wherever you
  submitted from, which may be `zsh`, and `zsh` syntax can break a bash job
  script.
- To run your own script, replace the last line with
  `Rscript /Polaris/<LabName>/<project>/your_script.R`. Keep input data and
  results in your lab's `/Polaris/<LabName>/` storage rather than your home.
- If the job fails with an `unbound variable` error during activation, remove
  `u` from `set -eo pipefail` (as written above it is already left out).

Submit the job from the directory containing the script:

```bash
sbatch test_R_batch.sbatch
```

### Check the result

`squeue` only lists pending and running jobs. This test takes a few seconds, so
it is usually finished before you can see it. Check the final state and output:

```bash
sacct -j <jobid> --format=JobID,JobName,State,ExitCode,Elapsed,NodeList
cat logs/r_test_<jobid>.out
```

A working job shows:

- `State` is `COMPLETED` and `ExitCode` is `0:0`
- `env: r_test_env` (not `base`)
- `which Rscript` points inside `~/miniforge3/envs/r_test_env/bin/`
- The same R version and library path as your interactive test, ending with `OK`

If the job shows a different R from your interactive session, it is picking up
`module load R` or a different `PATH`.

---

## Running R in Jupyter (R kernel)

To use R in a Jupyter notebook you need two things: the R kernel
(`r-irkernel`) and a registered kernel spec so JupyterLab can find it.
`r-essentials` already includes `r-irkernel`, so the environment from Step 1
has the kernel package. There are two ways to use it.

### Option A: Run JupyterLab from the R environment (simplest)

Put JupyterLab in the same environment as R. If you know you will use Jupyter,
add `jupyterlab` to the Step 1 command so everything is solved together:

```bash
conda create -n r_test_env -c conda-forge -c bioconda \
  "r-base>=4.5,<4.6" r-tidyverse r-essentials r-data.table jupyterlab
```

If the environment already exists, dry-run first and stop if `r-base` changes:

```bash
conda install -n r_test_env jupyterlab --dry-run
conda install -n r_test_env jupyterlab
```

Register the kernel **inside** the environment, so it travels with it:

```bash
conda activate r_test_env
Rscript -e 'IRkernel::installspec(name = "r45", displayname = "R 4.5 (r_test_env)", user = FALSE)'
jupyter kernelspec list
```

`jupyter kernelspec list` should show `r45` with a path under
`~/miniforge3/envs/r_test_env/share/jupyter/kernels/`.

Then launch JupyterLab as described in
[Running Jupyter Notebooks on Rosalind](../jupyter/), with one change in the
sbatch script: run it in the R environment instead of `spatial`.

```bash
# before
"${CONDA}" run -n spatial \
# after
"${CONDA}" run -n r_test_env \
```

When JupyterLab opens, choose **R 4.5 (r_test_env)** from the Launcher or from
the kernel menu of a notebook.

### Option B: Add the R kernel to an existing Jupyter environment

If you already run JupyterLab from another environment (for example `spatial`)
and just want R as an extra kernel, register the R environment's kernel for
your user. JupyterLab in the other environment will then list it.

```bash
conda activate r_test_env
Rscript -e 'IRkernel::installspec(name = "r45", displayname = "R 4.5 (r_test_env)", user = TRUE)'
```

This writes the kernel spec to `~/.local/share/jupyter/kernels/r45/`. Check that
`argv` in its `kernel.json` points to R inside `r_test_env`, not to a different R:

```bash
cat ~/.local/share/jupyter/kernels/r45/kernel.json
```

The R packages still come from `r_test_env`, so keep installing them there.

### Remove or rename a kernel

```bash
jupyter kernelspec list
jupyter kernelspec remove r45
```

### Troubleshooting

- **The R kernel is missing in JupyterLab:** run `jupyter kernelspec list` from
  the same environment that launches JupyterLab. If `r45` is not listed, the
  kernel was registered somewhere that Jupyter does not search; register it again
  with the matching `user =` setting above.
- **Kernel starts but dies immediately:** open a terminal in the same
  environment and run `R -e 'IRkernel::main()'` to see the error. A common cause
  is a leaked library path; set `export R_LIBS_USER=""` and check `.libPaths()`.
- **Wrong R version in a notebook:** run `R.version.string` and `.libPaths()` in
  a cell. They should match your Step 2 test.

---

## Tips

- **Storage:** environments and package caches are large, and the home pool is
  shared by everyone (see [Shared Storage](../guide/#shared-storage) in the HPC
  Server User Guide). Keep conda environments in your home by default, and keep
  datasets and results in your lab's storage under `/Polaris/<LabName>/`, not
  in your home. If your home is tight, move conda's package cache and
  environments to your lab storage (check with your lab first):

  ```bash
  conda config --add pkgs_dirs /Polaris/<LabName>/<username>/conda_pkgs
  conda config --add envs_dirs /Polaris/<LabName>/<username>/conda_envs
  ```

- **Many small files:** `/home`, `/Polaris` and `/mnt` are network storage. If an
  R job reads or writes huge numbers of small files, work in `/tmp` on the
  compute node and copy the results back at the end of the job.

- **Python and R together** (for example `rpy2`): put both in the same
  environment and install them in a single `conda create` command.
- **Share your environment** with a labmate:

  ```bash
  conda env export --from-history > environment.yml
  conda env create -f environment.yml
  ```
