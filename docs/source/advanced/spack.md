# Intro to Spack

![Spack logo](../_static/images/spack.png)

Spack is a multi-platform package manager that builds and installs multiple versions and configurations of software. It works on Linux, macOS, Windows, and many supercomputers. Spack is non-destructive: installing a new version of a package does not break existing installations, so many configurations of the same package can coexist

## Introduction

Spack is a flexible package manager designed specifically for high-performance computing (HPC) environments. Unlike traditional system package managers or Conda, Spack allows users to build, install, and manage multiple versions of scientific software with different compilers, MPI implementations, and build configurations — all side by side.

In HPC systems such as the Lane Cluster, software stacks are often complex:

- Applications depend on specific compiler versions
- MPI libraries must match both hardware and software
- Performance can vary significantly based on build options
- Users frequently need reproducible, isolated environments

Spack addresses these challenges by:

- Explicitly modeling dependencies and build configurations
- Supporting multiple compilers, MPI libraries, and variants
- Allowing reproducible environment definitions via `spack.yaml`
- Avoiding conflicts with system-wide software

On the Lane Cluster, Spack is especially useful for:

- MPI-based scientific applications (e.g., OpenMPI, PETSc)
- Bioinformatics and computational biology tools
- GPU-enabled machine learning frameworks
- Reproducible research workflows shared across users and nodes

---

## Prerequisites

Spack is written in Python and requires a working **Python interpreter (Python 3.6 or newer)** and **Git** to run.

### Python via Miniconda3

Before installing or using Spack on the Lane Cluster, make sure a suitable Python interpreter is available. Load the shared Miniconda3 module:

```bash
module load miniconda3
```

**Using the shared Miniconda3 module is the preferred approach** — it is maintained centrally, avoids duplicating a large installation in every home directory, and guarantees a consistent Python version across login and compute nodes.

If you need a Python installation you control yourself, install Miniconda3 or Anaconda in your home directory instead:

```bash
cd $HOME
wget https://repo.anaconda.com/miniconda/Miniconda3-latest-Linux-x86_64.sh
bash Miniconda3-latest-Linux-x86_64.sh -b -p $HOME/miniconda3
source $HOME/miniconda3/bin/activate
```

Either way, confirm Python is available before continuing:

```bash
python3 --version
```

Spack does **not** require Conda to manage packages and does **not** manage Python environments — Conda is used here only to provide the Python interpreter Spack runs on.

### Other requirements

Git is typically satisfied by the system installation. Spack also needs a C/C++ compiler to build packages; the system GCC toolchain is sufficient to get started.

---

## Installing Spack in Your Home Directory

Spack is not provided as a module on the Lane Cluster. Instead, each user installs their own copy in their home directory. This gives you full control over the version of Spack, the packages you build, and the configuration you use, without affecting other users.

### Clone the Spack repository

```bash
module load miniconda3
cd $HOME
git clone --depth=2 https://github.com/spack/spack.git
```

This creates `$HOME/spack`. The `--depth=2` flag keeps the clone small by skipping most of the Git history.

To install a specific release instead of the development branch:

```bash
cd $HOME
git clone --depth=2 --branch=releases/v1.0 https://github.com/spack/spack.git
```

### Activate Spack in your shell

Spack is activated by sourcing its setup script, which adds the `spack` command to your `PATH`:

```bash
source $HOME/spack/share/spack/setup-env.sh
```

For `csh`/`tcsh` use `setup-env.csh`, and for `fish` use `setup-env.fish`.

### Activate Spack automatically on login

To avoid sourcing the script manually in every session, add it to your shell startup file:

```bash
echo 'module load miniconda3' >> $HOME/.bashrc
echo 'source $HOME/spack/share/spack/setup-env.sh' >> $HOME/.bashrc
```

Both lines are also needed inside SLURM batch scripts, since batch jobs do not always source your interactive shell configuration. Add them explicitly to any job script that uses Spack:

```bash
#!/bin/bash
#SBATCH --partition=pool1
#SBATCH --time=01:00:00

module load miniconda3
source $HOME/spack/share/spack/setup-env.sh

spack install openmpi
```

### Confirm Spack is available

```bash
spack --version
```

If the command returns a version number, Spack has been installed successfully and is ready for use.

### Where Spack stores its files

By default, everything Spack creates lives under your home directory:

| Path | Contents |
|------|----------|
| `$HOME/spack` | The Spack source tree and package recipes |
| `$HOME/spack/opt/spack` | Installed packages |
| `$HOME/.spack` | User configuration and cached data |

Builds can consume a significant amount of disk space. If your home directory has a quota, point the install tree and build staging area at a larger filesystem by editing `$HOME/.spack/config.yaml`:

```yaml
config:
  install_tree:
    root: /path/to/large/filesystem/$user/spack
  build_stage:
    - /path/to/large/filesystem/$user/spack-stage
```

### Updating Spack

Because Spack is a Git repository, updating is a pull:

```bash
cd $HOME/spack
git pull
```

## Basic Usage

### Search for available packages

```bash
spack list
spack search openmpi
```

### Install a package

```bash
spack install openmpi
```

### List installed packages

```bash
spack find
```

### Load a package into the current environment

```bash
spack load openmpi
```

### Unload a package

```bash
spack unload openmpi
```

## Environment Management

Spack environments provide a reproducible way to manage project-specific dependencies.

### Create and activate an environment

```bash
spack env create mpi-env
spack env activate mpi-env
```

### Add packages to the environment

```bash
spack add openmpi
```

### Resolve dependencies and install all packages

```bash
spack concretize
spack install
```

Each environment is defined by a spack.yaml file, which can be shared with collaborators to reproduce the same software stack on other nodes or systems.

## Example Workflow 1: Installing OpenMPI

This example demonstrates installing and testing OpenMPI using Spack on a compute node in pool1.

Request an interactive compute node salloc -p pool1 --time=01:00:00

### Create and activate OpenMPI an environment

```bash
module load miniconda3
source $HOME/spack/share/spack/setup-env.sh
spack env create openmpi-env
spack env activate openmpi-env
```

### Install OpenMPI

```bash
spack add openmpi
spack concretize
spack install
```

### Verify the installation

```bash
spack load openmpi
mpirun --version
```

If the command returns a version number, OpenMPI has been built and linked successfully on the compute node.

## Example Workflow 2: Bioinformatics Workflow

**Use case:**  
A researcher needs a reproducible and isolated environment for RNA-seq analysis using tools such as HISAT2, Samtools, and Python scientific libraries.

**Approach:**  
Spack environments are used to define package specifications, resolve complex dependency chains, and install compatible versions in a unified prefix.

```bash
spack env create bio-env
spack env activate bio-env
spack add hisat2 samtools python@3.10 py-biopython py-numpy py-pandas
spack concretize
spack install
```

### Integration with Conda and Apptainer

Spack is designed to complement other software management tools commonly used in HPC environments, rather than replace them.

- **Conda** is well suited for Python-centric workflows, rapid experimentation, and lightweight data analysis. It excels at managing Python packages and prebuilt binaries but offers limited control over compilers and low-level system dependencies.

- **Spack** is optimized for building performance-critical, compiled software such as MPI libraries, numerical solvers, and GPU-enabled frameworks. It provides fine-grained control over compilers, build variants, and dependency resolution.

- **Apptainer (Singularity)** can be used to containerize Spack-built software, enabling portability across nodes or clusters while preserving optimized native libraries.

A common pattern is to use **Spack to build core system-level dependencies** (e.g., MPI, CUDA, math libraries) and layer **Conda environments or Apptainer containers** on top for application-level dependencies and reproducibility.

---

## Best Practices

- Use **Spack environments (`spack env`)** to define reproducible software stacks for projects and research workflows.

- Prefer building and testing packages on **compute nodes** rather than login nodes to avoid resource limitations and build failures.

- Reuse existing builds whenever possible to reduce compilation time and disk usage:
  
```bash
spack install --reuse
```

Periodically remove unused packages to reclaim disk space:

```bash
spack gc --all
```

## References

- Spack documentation, *Spack: A package manager for HPC*, available at: [https://spack.readthedocs.io/en/latest/index.html]
