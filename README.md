# Curator Setup for Jupyter Notebook for Mac (Apple Silicon) in VS Code

This guide covers setting up a Python environment to use the Synapse Curator extension on a Mac with Apple Silicon (M-series chip), running Jupyter notebooks inside VS Code.

> **Tested on:** macOS Sequoia 15.7.4, Apple M4

---

## Prerequisites

- Mac with Apple Silicon (M1/M2/M3/M4)
- VS Code installed ([download here](https://code.visualstudio.com/))
- A Synapse account with a Personal Access Token (PAT)

---

## Quick Install (condensed)

For a completely clean Mac with nothing installed yet. Install the VS Code extensions manually (Step 2), then run:

```bash
# Install Miniforge3 to ~/miniforge3 — do not move this folder later, see Troubleshooting
cd ~/Downloads
curl -fsSL -o Miniforge3.sh "https://github.com/conda-forge/miniforge/releases/latest/download/Miniforge3-MacOSX-arm64.sh"
chmod +x Miniforge3.sh
./Miniforge3.sh -b -p "$HOME/miniforge3"

# Replace <your-shell> with bash or zsh (see Step 1), then restart your terminal
~/miniforge3/bin/conda init <your-shell>
~/miniforge3/bin/mamba shell init --shell <your-shell> --root-prefix=$HOME/miniforge3
```

After restarting your terminal:

```bash
mamba create -n curator_env python=3.13 -y
mamba activate curator_env
pip install --upgrade "synapseclient[curator,pandas]"
pip install ipykernel
python -m ipykernel install --user --name curator_env --display-name "Python (curator_env)"

# Replace YOUR_TOKEN_HERE with your Synapse PAT (see Step 7a)
cat > ~/.synapseConfig << 'EOF'
[authentication]
authtoken = YOUR_TOKEN_HERE
EOF

# Verify everything works end to end
python -c "import synapseclient; syn = synapseclient.Synapse(); syn.login(); print(syn.getUserProfile()['userName'])"
```

Then in VS Code, open a notebook and select the **Python (curator_env)** kernel. The numbered steps below cover each stage in more detail, plus troubleshooting.

---

## Step 1: Check Your Shell

This guide works with either **bash** or **zsh** (macOS default). Check which shell you are using:

```bash
echo $SHELL
```

You will see either `/bin/bash` or `/bin/zsh`. Note which one — you will use it in Step 3 when initializing conda.

---

## Step 2: Install VS Code Extensions

Open VS Code and install the following extensions (`Cmd+Shift+X` to open the Extensions panel):

| Extension | Publisher | Purpose |
|-----------|-----------|---------|
| **Python** (`ms-python.python`) | Microsoft | Python language support |
| **Jupyter** (`ms-toolsai.jupyter`) | Microsoft | Jupyter notebook support in VS Code |

Both are free. The Python extension includes Pylance (IntelliSense and type checking).

---

## Step 3: Install Miniforge3

Miniforge is a lightweight conda distribution that defaults to the `conda-forge` channel and includes `mamba`, a significantly faster package solver than standard conda.

**3a. Download the installer**

Go to: https://github.com/conda-forge/miniforge/releases/latest

Download the file named `Miniforge3-<version>-MacOSX-arm64.sh` (e.g. `Miniforge3-26.1.1-3-MacOSX-arm64.sh`). Make sure to select the `MacOSX-arm64` version — this is for Apple Silicon.

**3b. Run the installer**

In Terminal, substitute the actual filename you downloaded:

```bash
cd ~/Downloads
chmod +x Miniforge3-26.1.1-3-MacOSX-arm64.sh
./Miniforge3-26.1.1-3-MacOSX-arm64.sh -b -p "$HOME/miniforge3"
```

The `-b` flag runs in batch mode (no interactive prompts). The `-p` flag sets the install location.

**3c. Initialize conda and mamba for your shell**

Replace `<your-shell>` with either `bash` or `zsh` based on what you saw in Step 1:

```bash
~/miniforge3/bin/conda init <your-shell>
mamba shell init --shell <your-shell> --root-prefix=$HOME/miniforge3
```

Examples:
- **zsh (macOS default):** `~/miniforge3/bin/conda init zsh` and `mamba shell init --shell zsh --root-prefix=$HOME/miniforge3`
- **bash:** `~/miniforge3/bin/conda init bash` and `mamba shell init --shell bash --root-prefix=$HOME/miniforge3`

**3d. Restart your terminal**, then verify:

```bash
conda --version
mamba --version
```

Both should print version numbers. You should also see `(base)` at the start of your prompt, indicating the base conda environment is active.

---

## Step 4: Create the Curator Environment

```bash
mamba create -n curator_env python=3.13
mamba activate curator_env
```

> **Why not Python 3.14?**
> `synapseclient` 4.13.0 declares support for Python 3.10–3.14 on PyPI, and its CI suite passes on 3.14 — but that CI runs `pytest` directly, which never has an asyncio event loop already running when it calls the library. Jupyter/`ipykernel` does: each notebook cell executes inside an already-running event loop. `synapseclient`'s `async_to_sync()` wrapper (used by `query_schema_registry` and others) explicitly detects that combination — an active event loop **and** Python 3.14+ — and raises `RuntimeError: Python 3.14+ detected an active event loop...` instead of falling back to its usual `nest_asyncio` shim, which no longer works reliably on 3.14. On 3.10–3.13 the `nest_asyncio` fallback still applies, so notebooks work normally. Use 3.13 until an upstream fix (either an async variant of these functions, or a working 3.14 fallback) lands. See [Troubleshooting](#troubleshooting) if you hit this.

---

## Step 5: Install Required Packages

```bash
pip install --upgrade "synapseclient[curator,pandas]"
pip install ipykernel
```

Verify the install worked:

```bash
python -c "import synapseclient; from synapseclient.extensions.curator import query_schema_registry; print('success')"
```

You should see `success` printed.

> **bash users:** Avoid using `!` in one-line terminal commands — bash interprets it as a history expansion character and will error. This does not affect notebook cells.

---

## Step 6: Register as a Jupyter Kernel

This makes `curator_env` available as a selectable kernel inside VS Code notebooks:

```bash
python -m ipykernel install --user --name curator_env --display-name "Python (curator_env)"
```

You should see a message like: `Installed kernelspec curator_env in /Users/<you>/...`

---

## Step 7: Set Up Synapse Authentication

**7a. Create your Synapse Personal Access Token (PAT)**

Go to: https://www.synapse.org/ → your profile menu → Account Settings → Personal Access Tokens → create a new token with all permissions.

**7b. Store the token in your home directory config file**

```bash
cat > ~/.synapseConfig << 'EOF'
[authentication]
authtoken = YOUR_TOKEN_HERE
EOF
```

Replace `YOUR_TOKEN_HERE` with your actual token. Note the single quotes around `EOF` — this prevents the shell from interpreting any special characters in your token.

> **Important:** `~/.synapseConfig` lives in your home directory, **not** inside any project folder. This means it will not be picked up by git and will not be accidentally committed. Never copy your token into a notebook cell or project file.

**7c. Verify authentication**

```bash
python -c "import synapseclient; syn = synapseclient.Synapse(); syn.login(); print(syn.getUserProfile()['userName'])"
```

This should print your Synapse username.

---

## Step 8: Use Curator in a VS Code Notebook

1. Open VS Code
2. Create a new notebook: `Cmd+Shift+P` → type **New Jupyter Notebook** → Enter
3. Select your kernel: click the kernel picker in the top-right corner → choose **Python (curator_env)**
4. In the first cell, authenticate:

```python
import synapseclient
from synapseclient.extensions.curator import (
    query_schema_registry,
    create_record_based_metadata_task,
    create_file_based_metadata_task,
)

syn = synapseclient.Synapse()
syn.login()
```

5. In a new cell, browse available schemas:

```python
results = query_schema_registry(
    synapse_client=syn,
    dcc="ad"
)
print(results)
```

---

## Upgrading synapseclient

To get the latest version of synapseclient and Curator without touching your Python version:

```bash
mamba activate curator_env
pip install --upgrade "synapseclient[curator,pandas]"
```

Check what version you currently have:

```bash
pip show synapseclient
```

If you need to move to a different Python version later, recreate the environment rather than upgrading Python in place — it avoids dependency conflicts:

```bash
mamba deactivate
mamba env remove -n curator_env
mamba create -n curator_env python=<version>
mamba activate curator_env
pip install --upgrade "synapseclient[curator,pandas]"
pip install ipykernel
python -m ipykernel install --user --name curator_env --display-name "Python (curator_env)"
```

Your `~/.synapseConfig` authentication is unaffected — it lives outside the environment. Check the [synapseclient PyPI page](https://pypi.org/project/synapseclient/) for the currently supported Python version range (`Requires-Python`) before picking a version — but avoid 3.14 for notebook use regardless of what that range says; see [Step 4](#step-4-create-the-curator-environment) for why.

---

## Troubleshooting

**`conda: command not found` after install**
Run `~/miniforge3/bin/conda init <your-shell>`, then close and reopen your terminal.

**`mamba activate` fails with "Shell not initialized" / "critical libmamba"**
Run `mamba shell init --shell <your-shell> --root-prefix=$HOME/miniforge3`, then restart terminal.

**`Cannot activate, prefix does not exist at .../curator_env`**
The environment was not created yet. Run `mamba create -n curator_env python=3.13` first.

**`RuntimeError: Python 3.14+ detected an active event loop, which prevents automatic async-to-sync conversion`**
Your `curator_env` is running Python 3.14. `synapseclient` supports 3.14 for general/scripted use (its own test suite runs there fine), but functions like `query_schema_registry` can't auto-convert async to sync when called from inside a *already-running* event loop on 3.14+ — and Jupyter notebooks always have one running. There's no async equivalent of `query_schema_registry` to `await` around instead. Fix: recreate the environment with Python 3.13 (Step 4):
```bash
mamba deactivate
mamba env remove -n curator_env
mamba create -n curator_env python=3.13 -y
mamba activate curator_env
pip install --upgrade "synapseclient[curator,pandas]"
pip install ipykernel
python -m ipykernel install --user --name curator_env --display-name "Python (curator_env)"
```
Then restart the kernel in your notebook (Kernel → Restart) — the old kernel process keeps running the old Python version in memory even after you recreate the environment on disk.

**`No module named ipykernel`**
Run `pip install ipykernel` while inside the activated environment, then re-run the kernel install command from Step 6.

**`bash: !': event not found`**
Bash is interpreting `!` as a history expansion character. Rewrite your print statement to avoid `!`, e.g. use `print('success')`.

**`conda`/`mamba` stopped working, or the kernel can't be found, after everything used to work**
Conda environments hardcode absolute paths at creation time (in script shebangs and activation scripts), so moving the `~/miniforge3` folder — even to another location under your home directory, e.g. into `~/Applications/` — silently breaks it. Signs of this:
- `~/.bash_profile` / `~/.zshrc` sets `MAMBA_ROOT_PREFIX` to a path that no longer exists (`grep MAMBA_ROOT_PREFIX ~/.bash_profile ~/.zshrc`).
- `~/Library/Jupyter/kernels/curator_env/kernel.json` points its `python` argv at a path that doesn't exist (`cat` the file and check).
- Running any script inside the environment (e.g. `pip`) fails with `bad interpreter: No such file or directory`.

Don't try to move the folder back — reinstall Miniforge fresh at `~/miniforge3` and recreate `curator_env` (Steps 3–6). If you have leftover broken installs elsewhere (e.g. in `~/Applications/`), delete them once you've confirmed nothing else depends on them.
