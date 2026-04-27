# Conda Guide

**Purpose:** What conda is, why sheepsoc uses it, and the commands you actually need.

| Key | Value |
|---|---|
| Install | Miniconda at `~/infrastructure/miniconda3/` |
| Environments | `openwebui` (Python 3.11), `sheepsoc` (Python 3.12), `datawrangler`, `base` |

## What Conda Is (Plain English)

**Conda** is an environment manager for Python. An *environment* is a self-contained Python install with its own packages, isolated from everything else on the system.

Think of it like this:

- System Python is *everybody's* Python. If two projects need different versions of a library, they fight.
- A conda environment is *one project's* Python. Each environment has its own interpreter and its own installed packages.
- You switch between them with `conda activate <name>`. Your shell prompt shows the active environment in parentheses: `(sheepsoc) pmabry@sheepsoc:~$`.

**Miniconda** is the lightweight version of conda — just the environment manager, no pre-installed scientific stack. That is what is installed here, at `~/infrastructure/miniconda3/`.

!!! note "Mental Model"
    When the prompt shows `(sheepsoc)`, running `python` runs the sheepsoc env's Python, and `pip install X` installs into that env only. The moment you `conda deactivate`, you're back to plain shell Python and those packages are invisible. Nothing you do inside an env can break the system Python — that's the whole point.

## The Environments on This Box

| Env Name | Python | Used For | Notes |
|---|---|---|---|
| `openwebui` | 3.11 | **Primary production env.** Runs the `open-webui.service` systemd unit — the main AI interface on sheepsoc. | `elasticsearch==8.19.3` pip-installed manually — not bundled; must reinstall if env is rebuilt. |
| `sheepsoc` | 3.12 | **Legacy / prototype.** Was used for the CLI RAG prototype; now used only as the Jupyter kernel and for ad-hoc embedding experiments. | The CLI RAG app (`~/repositories/sheepsoc`) in this env is no longer maintained. Do not use it for production RAG queries. |
| `datawrangler` | — | Separate environment for data wrangling. | — |
| `base` | — | Miniconda's own base — *don't* install project stuff here. | — |

To see them yourself:

```bash
pmabry@sheepsoc:~$ conda env list
# conda environments:
#
base                     /home/pmabry/infrastructure/miniconda3
sheepsoc                 /home/pmabry/infrastructure/miniconda3/envs/sheepsoc
openwebui                /home/pmabry/infrastructure/miniconda3/envs/openwebui
datawrangler             /home/pmabry/infrastructure/miniconda3/envs/datawrangler
```

!!! warning "Careful"
    The `openwebui` environment has `elasticsearch==8.19.3` installed via pip (not via conda). This package is not bundled with OpenWebUI and must be reinstalled manually if the env is ever deleted and recreated:

    ```bash
    conda activate openwebui && pip install elasticsearch==8.19.3
    ```

    Without it, `open-webui.service` will fail to start with an `ImportError`.

## Daily Commands

```bash
# Switch INTO an environment
pmabry@sheepsoc:~$ conda activate sheepsoc
(sheepsoc) pmabry@sheepsoc:~$

# Leave the environment (go back to base / plain shell)
(sheepsoc) pmabry@sheepsoc:~$ conda deactivate

# List environments
pmabry@sheepsoc:~$ conda env list

# List packages in the currently-active env
(sheepsoc) pmabry@sheepsoc:~$ conda list

# Which Python am I actually using right now?
(sheepsoc) pmabry@sheepsoc:~$ which python
/home/pmabry/infrastructure/miniconda3/envs/sheepsoc/bin/python
```

## How Conda Gets Into Your Shell

For `conda activate` to work, conda's shell integration has to be loaded. This is normally done once in `~/.bashrc`. On this machine, that line uses the **static loader**:

```bash
# At the bottom of ~/.bashrc
source ~/infrastructure/miniconda3/etc/profile.d/conda.sh
```

The static loader is used here specifically because the default `conda shell.bash hook` subprocess was hanging login for ~5 minutes (fixed 2026-04-18). Do not replace it with the hook form.

If you land in a shell where conda isn't initialized (e.g., a minimal systemd unit or `sh -c`), source it manually:

```bash
pmabry@sheepsoc:~$ source ~/infrastructure/miniconda3/etc/profile.d/conda.sh
pmabry@sheepsoc:~$ conda activate sheepsoc
```

## Installing Packages

Two installers. Use the right one:

| Tool | When to Use |
|---|---|
| `conda install <pkg>` | Anything with compiled binaries (numpy, scipy, pytorch, hdf5 libraries) — conda handles the native deps. Prefer this for heavy scientific stack. |
| `pip install <pkg>` | Pure-Python packages or anything not in conda channels. Used from **inside** an active conda env — pip installs into the env, not the system. |

```bash
# Activate first, always
pmabry@sheepsoc:~$ conda activate sheepsoc

# conda-style
(sheepsoc) pmabry@sheepsoc:~$ conda install numpy pandas

# pip-style (the env's pip, so it stays in the env)
(sheepsoc) pmabry@sheepsoc:~$ pip install langchain
```

!!! warning "Careful"
    Mixing `conda install` and `pip install` in the same env mostly works, but can occasionally conflict. Rule of thumb: let conda handle the scientific stack first, then pip for the rest.

## Creating a New Environment

```bash
# Create a fresh env with a specific Python version
pmabry@sheepsoc:~$ conda create -n myenv python=3.12

# Activate it
pmabry@sheepsoc:~$ conda activate myenv

# Install what you need
(myenv) pmabry@sheepsoc:~$ conda install numpy
(myenv) pmabry@sheepsoc:~$ pip install whatever-else
```

Environments live under `~/infrastructure/miniconda3/envs/<name>/`. If you ever wonder how much disk a given env is taking, that's the directory to check.

## Removing an Environment

```bash
# Deactivate first if you're inside it
(myenv) pmabry@sheepsoc:~$ conda deactivate

pmabry@sheepsoc:~$ conda env remove -n myenv
```

!!! note "Reversibility"
    Before removing an env that has any chance of being needed again, export it: `conda env export -n myenv > myenv.yaml`. You can rebuild from the YAML with `conda env create -f myenv.yaml`.

## Common Pitfalls

### `conda: command not found`

Conda's shell integration isn't loaded. Source it:

```bash
source ~/infrastructure/miniconda3/etc/profile.d/conda.sh
```

### Packages installed, but `python` doesn't see them

Almost always because you are not in the env you think you are. Verify:

```bash
$ conda env list           # which env has the *
$ which python
$ python -c "import sys; print(sys.prefix)"
```

### Pip installed to the wrong place

If your prompt doesn't show `(envname)`, you're not in a conda env — pip installed to either base or system Python. Activate the env first, then reinstall.

### Slow login after editing .bashrc

If login hangs for minutes, the conda init line may have reverted to the subprocess `conda shell.bash hook` form. Replace it with the static loader (`source …/profile.d/conda.sh`). This was fixed on 2026-04-18.
