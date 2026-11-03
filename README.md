# Debian on WSL2 Setup & Configuration Guide: Rootless Docker, NVIDIA Container Toolkit, Developer Setup

> This guide provides a modular, step-by-step workflow to install Debian on WSL2, configure Rootless Docker Engine, and optionally integrate the NVIDIA Container Toolkit for GPU acceleration if your host machine has an NVIDIA GPU.

> [!IMPORTANT]
> **Before you start**
> * Windows 11 with WSL 2.4.4 or newer (`wsl --version`).
> * VS Code installed on Windows with the WSL extension if you want `code .` (see the [VS Code WSL guide](https://code.visualstudio.com/docs/remote/wsl)).
> * The guide uses `D:\WSL` as an example; replace it if you do not have a `D:` drive.
> * The NVIDIA phase is optional and needs a current Windows NVIDIA driver.

---

## Phase 1: PowerShell Initial Setup (Windows Host)

Run these commands in a **PowerShell** prompt on Windows.

```powershell
# ------------------------------------------------------------------------------
# STEP 1: First-Time Setup & Windows Prerequisites
# ------------------------------------------------------------------------------
# NOTE: If WSL is not yet installed on Windows, running this command will
# enable the required Windows features and prompt for a PC restart.
# If prompted, restart your PC and rerun this block.

# --location requires the target directory to exist. Create it first.
New-Item -ItemType Directory -Force -Path "D:\WSL\Debian" | Out-Null

wsl --install Debian --name Debian --web-download --location "D:\WSL\Debian" --no-launch

# ------------------------------------------------------------------------------
# STEP 2: Initialize Instance & Create User
# ------------------------------------------------------------------------------
# Launch the instance to trigger the initial configuration prompt where you create
# your non-root UNIX username and password.

wsl -d Debian
```

> [!NOTE]
> `--name` and `--location` require WSL 2.4.4 or newer. If WSL is already installed, run `wsl --update` before the install command. If WSL is not installed yet, the first run enables it and may request elevation once; after the restart, rerun the block from a normal PowerShell prompt.

Inside the Debian shell, configure `/etc/wsl.conf` and the VS Code mounts. `$USER` is expanded automatically, and `WINUSER` is auto-detected while `/mnt/c` is still mounted:

```bash
# Write /etc/wsl.conf (unquoted EOF lets $USER expand)
sudo tee /etc/wsl.conf >/dev/null <<EOF
[boot]
systemd=true

[automount]
enabled=false
mountFsTab=true

[interop]
appendWindowsPath=false

[network]
hostname=debian

[user]
default=$USER
EOF

# VS Code Remote-WSL reads these two Windows folders. Mount only them, read-only.
# The mountpoint directories are created after the WSL restart, before `mount -a`.
WINUSER="$(/mnt/c/Windows/System32/cmd.exe /c 'echo %USERNAME%' 2>/dev/null | tr -d '\r')"
if [ -z "$WINUSER" ]; then
  WINUSER="$(sed -n 's#^c:/Users/\([^/]*\)/vscode-remote-wsl.*#\1#p' /etc/fstab | head -n 1)"
fi
WINUSER="${WINUSER:-YourWindowsUsername}"

if ! grep -q 'vscode-remote-wsl' /etc/fstab; then
  sudo tee -a /etc/fstab >/dev/null <<EOF
c:/Users/$WINUSER/.vscode/extensions  /mnt/c/Users/$WINUSER/.vscode/extensions  drvfs defaults,ro 0 0
c:/Users/$WINUSER/vscode-remote-wsl   /mnt/c/Users/$WINUSER/vscode-remote-wsl   drvfs defaults,ro 0 0
EOF
fi

# Detect the VS Code install folder so `code .` works without extra steps.
CODE_ROOT=""
for p in "/mnt/c/Users/$WINUSER/AppData/Local/Programs/Microsoft VS Code" \
         "/mnt/c/Program Files/Microsoft VS Code"; do
  if [ -d "$p" ]; then
    CODE_ROOT="$p"
    break
  fi
done

if [ -n "$CODE_ROOT" ]; then
  CODE_WIN="$(wslpath -w "$CODE_ROOT" | sed 's#\\#/#g')"
  CODE_WIN_ESC="${CODE_WIN// /\\040}"
  CODE_ROOT_ESC="${CODE_ROOT// /\\040}"
  if ! grep -qF "$CODE_WIN_ESC" /etc/fstab; then
    printf '%s  %s  drvfs defaults,ro 0 0\n' "$CODE_WIN_ESC" "$CODE_ROOT_ESC" |
      sudo tee -a /etc/fstab >/dev/null
  fi
fi
```

> [!TIP]
> * **Username:** `$USER` is expanded by your current shell before `sudo` runs, so the file gets the UNIX username you created in Step 2. Run the block as your normal user, not through `sudo -i` or as root, or `$USER` will be `root`.
> * **VS Code Integration:** The VS Code Remote-WSL extension needs `%USERPROFILE%\.vscode\extensions` and `%USERPROFILE%\vscode-remote-wsl`. With automount disabled, the two `/etc/fstab` entries above provide exactly those folders, read-only; the rest of `C:` stays hidden. The mountpoints are created after the restart.
> * **Terminal `code` Command:** Because `appendWindowsPath=false` isolates your Linux `$PATH` from Windows, `code .` is not available in your WSL terminal. The block below auto-detects the Windows VS Code CLI, adds an alias to `~/.bashrc`, and adds the VS Code install folder to `/etc/fstab` as a read-only mount. Install VS Code on Windows before running this block.

```bash
# Remove a previous copy (makes re-pasting safe)
if grep -q '^# >>> vscode cli >>>$' ~/.bashrc &&
   grep -q '^# <<< vscode cli <<<$' ~/.bashrc; then
  awk '
    /^# >>> vscode cli >>>$/ { skip=1; next }
    /^# <<< vscode cli <<<$/ { skip=0; drop_blank=1; next }
    skip { next }
    drop_blank && /^$/ { drop_blank=0; next }
    { drop_blank=0; print }
  ' ~/.bashrc > ~/.bashrc.tmp &&
    cat ~/.bashrc.tmp > ~/.bashrc &&
    rm -f ~/.bashrc.tmp
fi

cat << 'EOF' >> ~/.bashrc
# >>> vscode cli >>>
# Alias `code` to the Windows VS Code CLI so `code .` works from WSL.
for _code_bin in \
  /mnt/c/Users/*/AppData/Local/Programs/Microsoft\ VS\ Code/bin/code \
  "/mnt/c/Program Files/Microsoft VS Code/bin/code"; do
  if [ -x "$_code_bin" ]; then
    alias code="\"$_code_bin\""
    break
  fi
done
unset _code_bin
# <<< vscode cli <<<
EOF

source ~/.bashrc
```

Restart WSL from PowerShell to apply `wsl.conf`:

```powershell
wsl --shutdown
wsl -d Debian
```

### Phase 1a: Apply the VS Code Mounts (After Restart)

After the restart, create the mountpoints and apply the fstab entries:

```bash
WINUSER="$(sed -n 's#^c:/Users/\([^/]*\)/vscode-remote-wsl.*#\1#p' /etc/fstab | head -n 1)"

# Create every /mnt/c mountpoint listed in fstab, unescaping \040.
sudo -v
sudo awk '$2 ~ /^\/mnt\/c\// { gsub(/\\040/, " ", $2); print $2 }' /etc/fstab |
while IFS= read -r mp; do
  sudo mkdir -p "$mp"
done

sudo mount -a

findmnt "/mnt/c/Users/$WINUSER/.vscode/extensions"
findmnt "/mnt/c/Users/$WINUSER/vscode-remote-wsl"

VSCODE_MOUNT="$(awk '$1 ~ /VS.040Code/ { gsub(/\\040/, " ", $2); print $2; exit }' /etc/fstab)"
if [ -n "$VSCODE_MOUNT" ]; then
  findmnt "$VSCODE_MOUNT"
fi
```

> [!NOTE]
> * The first boot after disabling automount happens before these directories exist, so the fstab entries may fail once. Creating them and running `sudo mount -a` fixes it; they persist for later boots.
> * Install VS Code and the WSL extension on Windows first; `.vscode\extensions` must exist for the mount to succeed.
> * If Windows did not expose `%USERNAME%`, replace `YourWindowsUsername` in `/etc/fstab` first, then rerun the commands above.
> * `%USERPROFILE%\vscode-remote-wsl` must exist on Windows. Starting one WSL connection from VS Code normally creates it; if it is missing, connect once, then run `sudo mount -a` again.
> * If the VS Code WSL log reports a different server path, such as `AppData\Local\Temp\vscode-remote-wsl`, mount that folder instead of `%USERPROFILE%\vscode-remote-wsl`.
> * `code .` is wired up automatically when the VS Code install folder is found. If VS Code is in a non-standard path, add its folder to `/etc/fstab` and create the mountpoint manually.

---

## Phase 2: Backup & Restore Cheat Sheet (PowerShell)

Run these PowerShell commands on the Windows host to snapshot or restore your Debian environment.

Back up the instance:

```powershell
wsl --terminate Debian
New-Item -ItemType Directory -Force -Path "D:\WSL\_backup" | Out-Null

$backup = "D:\WSL\_backup\DebianInitialStart.tar"
Remove-Item -Force $backup -ErrorAction SilentlyContinue
wsl --export Debian $backup
```

> [!WARNING]
> The restore below is destructive. `wsl --unregister` deletes the current Debian instance and every change made after the snapshot. Only run it when you intend to roll back.

```powershell
wsl --unregister Debian
wsl --import Debian D:\WSL\Debian\ D:\WSL\_backup\DebianInitialStart.tar --version 2
wsl -d Debian
```

---

## Phase 3: Docker Rootless Installation

This script installs Docker Engine and configures it in **Rootless mode**. All users (with or without NVIDIA GPUs) should complete this step.

Run the following block in your Debian terminal to generate, make executable, and run `setup_docker_rootless.sh`:

```bash
cat << 'EOF' > setup_docker_rootless.sh
#!/usr/bin/env bash
set -euo pipefail

if [ ! -d /run/systemd/system ]; then
  echo "systemd is not running. Enable it in /etc/wsl.conf and restart WSL first." >&2
  exit 1
fi

echo "=== STEP 1: Updating System Packages & Prerequisites ==="
sudo apt-get update && sudo apt-get upgrade -y
sudo apt-get install -y \
  ca-certificates \
  curl \
  gnupg \
  uidmap \
  unzip \
  slirp4netns \
  dbus-user-session

echo "=== STEP 2: Setting up Docker Official Repository ==="
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | \
  sudo gpg --yes --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
sudo apt-get install -y \
  docker-ce \
  docker-ce-cli \
  containerd.io \
  docker-buildx-plugin \
  docker-compose-plugin \
  docker-ce-rootless-extras

echo "=== STEP 3: Disabling System Docker Daemon (Rootless Prep) ==="
sudo systemctl disable --now docker.service docker.socket || true
sudo systemctl stop docker.socket || true
sudo rm -f /var/run/docker.sock

echo "=== STEP 4: Configuring Docker Rootless Mode ==="
dockerd-rootless-setuptool.sh install --force

# Export DOCKER_HOST into bashrc if not already present
if ! grep -q "DOCKER_HOST" ~/.bashrc; then
  {
    echo '# Rootless Docker socket (added by the WSL setup guide)'
    echo 'export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock'
  } >> ~/.bashrc
fi
export DOCKER_HOST=unix:///run/user/$(id -u)/docker.sock

# Enable lingering so user-space services persist
sudo loginctl enable-linger "$USER"

systemctl --user enable --now docker
systemctl --user restart docker

echo "=== Docker Rootless Setup Complete! ==="
EOF

chmod +x setup_docker_rootless.sh
./setup_docker_rootless.sh
source ~/.bashrc
```

---

## Phase 4: NVIDIA Container Toolkit (Optional)

> [!NOTE]
> Only run this phase if your host system has an NVIDIA GPU and you require GPU acceleration inside Docker containers. If your machine uses integrated graphics, an AMD GPU, or you do not need GPU support, **skip this phase**.

> [!IMPORTANT]
> Install the NVIDIA driver on **Windows**, not inside WSL. Before running this phase, `nvidia-smi` must already work in the Debian shell:
>
> ```bash
> nvidia-smi
> ```
>
> If that command fails, fix the Windows driver / WSL GPU setup first. The NVIDIA Container Toolkit cannot make a missing WSL GPU driver work. See the [CUDA on WSL guide](https://docs.nvidia.com/cuda/wsl-user-guide/index.html).

Run the following block in your Debian terminal to generate, make executable, and run `setup_nvidia_toolkit.sh`:

```bash
cat << 'EOF' > setup_nvidia_toolkit.sh
#!/usr/bin/env bash
set -euo pipefail

echo "=== STEP 1: Installing NVIDIA Container Toolkit Repository & Packages ==="
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --yes --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -fsSL https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list >/dev/null

sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit

echo "=== STEP 2: Configuring NVIDIA Runtime for Rootless Docker ==="
# Ensure rootless docker config directory exists
mkdir -p ~/.config/docker

# Configure NVIDIA toolkit to update rootless daemon.json (NOT /etc/docker/daemon.json)
nvidia-ctk runtime configure --runtime=docker --config="$HOME/.config/docker/daemon.json"

systemctl --user restart docker

echo "=== STEP 3: Configure NVIDIA for Rootless Docker ==="
# Rootless Docker cannot manage cgroup device rules; this disables
# NVIDIA's cgroup enforcement for the runtime.
sudo nvidia-ctk config --set nvidia-container-cli.no-cgroups --in-place

echo "=== NVIDIA Container Toolkit Setup Complete! ==="
EOF

chmod +x setup_nvidia_toolkit.sh
./setup_nvidia_toolkit.sh
source ~/.bashrc
```

> [!NOTE]
> This `no-cgroups` setting is required for rootless Docker. If GPU access still fails, see the WSL2 troubleshooting notes in Phase 5.

---

## Phase 5: Verification & Testing

### 1. Verify Docker Rootless Mode (All Systems)

Run the following to confirm Docker is running in rootless mode:

```bash
docker info | grep -i "rootless"
```

*Expected output includes `rootless` (under `Security Options`).*

### 2. Test Hello-World Container (All Systems)

```bash
docker run --rm hello-world
```

### 3. Test NVIDIA GPU Access (NVIDIA Machines Only)

If you completed Phase 4, test GPU pass-through inside a container:

```bash
docker run --rm --gpus all nvidia/cuda:13.0.2-base-ubuntu22.04 nvidia-smi
```

> [!NOTE]
> The CUDA 13 image needs a recent Windows NVIDIA driver. If it fails with a driver mismatch, use an older CUDA tag that matches your installed driver.

> [!TIP]
> **Troubleshooting the GPU test:**
>
> 1. Confirm `nvidia-smi` works in the Debian shell. If it does not, the Windows driver / WSL GPU setup is the problem, not the container toolkit.
> 2. Confirm the rootless `no-cgroups` setting was applied:
>    ```bash
>    grep -n "no-cgroups" /etc/nvidia-container-runtime/config.toml
>    ```
>    It should show `no-cgroups = true`.
> 3. If the container fails with `Failed to initialize NVML: GPU access blocked by the operating system`, this is a known WSL2-specific conflict: some WSL2 driver/toolkit combinations need `no-cgroups = false` for NVML access, while rootless Docker generally needs `true` for cgroup device rules. Try `no-cgroups = false` and retest:
>    ```bash
>    sudo nvidia-ctk config --set nvidia-container-cli.no-cgroups=false --in-place
>    docker run --rm --gpus all nvidia/cuda:13.0.2-base-ubuntu22.04 nvidia-smi
>    ```
>    If neither setting works, verify the Windows NVIDIA driver, WSL kernel, and NVIDIA Container Toolkit are current. Rootless GPU access on WSL2 depends on all three.

---

## Phase 6: Developer CLI & Runtime Tooling

Install common developer CLIs, prompt styling, package managers, and language runtimes (`uv`, `bun`, `fnm`, `Antigravity CLI`, `OpenCode`, and `oh-my-posh`).

> [!IMPORTANT]
> Oh My Posh is initialized at the **bottom** of `~/.bashrc` so another prompt framework cannot override it. Put other shell configuration above that line; Phases 7 and 10 reorder automatically.

```bash
# Clipboard utility for pasting images from Windows into agy/opencode
sudo apt update && sudo apt install -y wl-clipboard

# 1. Astral uv (Python manager) & Python runtimes
curl -LsSf https://astral.sh/uv/install.sh | sh
source ~/.bashrc
uv python install 3.11 3.12 3.13 3.14
uv --version

# 2. Bun JavaScript runtime
curl -fsSL https://bun.com/install | bash
source ~/.bashrc
bun --version

# 3. fnm (Fast Node Manager) & Node.js LTS
curl -fsSL https://fnm.vercel.app/install | bash
source ~/.bashrc
fnm install lts-latest
fnm use lts-latest
node --version
npm --version

# 4. pnpm
bun add -g pnpm

# 5. Antigravity CLI
curl -fsSL https://antigravity.google/cli/install.sh | bash
agy --version

# 6. OpenCode
curl -fsSL https://opencode.ai/v2/install | bash
source ~/.bashrc
opencode --version

# 7. Oh My Posh & theme (the Nerd Font is installed on Windows; see the note below)
curl -fsSL https://ohmyposh.dev/install.sh | bash -s
mkdir -p ~/.config/oh-my-posh/
cp -r ~/.cache/oh-my-posh/themes/. ~/.config/oh-my-posh/themes/

if ! grep -q "oh-my-posh init bash" ~/.bashrc; then
  printf '\n# Oh My Posh prompt (kept last so nothing overrides it)\neval "$(oh-my-posh init bash --config ~/.config/oh-my-posh/themes/emodipt-extend.omp.json)"\n' >> ~/.bashrc
fi

source ~/.bashrc
```

> [!IMPORTANT]
> **Install the Nerd Font on Windows, not in WSL.** Windows Terminal renders the prompt, so a font installed only inside Debian is invisible to it.
>
> In a PowerShell prompt on Windows:
>
> ```powershell
> winget install JanDeDobbeleer.OhMyPosh --source winget
> oh-my-posh font install RobotoMono
> ```
>
> Then set Windows Terminal to use it. Open Settings (`Ctrl+Shift+,`), add this under `profiles.defaults` in `settings.json`, and restart Windows Terminal:
>
> ```json
> {
>   "profiles": {
>     "defaults": {
>       "font": {
>         "face": "RobotoMono Nerd Font"
>       }
>     }
>   }
> }
> ```
>
> If `oh-my-posh font install` is unavailable, install the font manually from [Nerd Fonts](https://www.nerdfonts.com/font-downloads) and use the exact family name shown in Windows Settings.

---

## Phase 7: OpenCode v2 Web UI & Server

OpenCode v2 serves the web UI from its shared background service. This phase sets the server password once and adds an `opencode-server` launcher that starts the service. After the first browser pairing, the browser uses a session cookie; you do not enter the password or pair again on every start.

### 1. Set the server password

Run this once:

```bash
# Replace with your own password. Quote it if it contains spaces or special characters.
opencode service set password YourSecurePasswordHere
```

The password is stored in `~/.config/opencode/service.json`, which is already private:

```bash
ls -l ~/.config/opencode/service.json
```

Expected output:

```text
-rw------- 1 username username ... service.json
```

> [!NOTE]
> Changing the service password stops the background service so the new setting can take effect. The next `opencode-server` run starts it again.

### 2. Add the launcher to `~/.bashrc`

Paste this block once. It is safe to re-paste: it removes the previous copy first, writes the functions, and keeps the Oh My Posh `eval` line at the very bottom.

```bash
# Remove a previous copy (makes re-pasting safe)
if grep -q '^# >>> opencode helpers >>>$' ~/.bashrc &&
   grep -q '^# <<< opencode helpers <<<$' ~/.bashrc; then
  awk '
    /^# >>> opencode helpers >>>$/ { skip=1; next }
    /^# <<< opencode helpers <<<$/ { skip=0; drop_blank=1; next }
    skip { next }
    drop_blank && /^$/ { drop_blank=0; next }
    { drop_blank=0; print }
  ' ~/.bashrc > ~/.bashrc.tmp &&
    cat ~/.bashrc.tmp > ~/.bashrc &&
    rm -f ~/.bashrc.tmp
fi

cat << 'EOF' >> ~/.bashrc
# >>> opencode helpers >>>
# Start the shared OpenCode service and print the local web UI URL.
opencode-server() {
  opencode service start || return

  local url
  url="$(opencode service status)" || return
  url="${url/127.0.0.1/localhost}"
  url="${url/0.0.0.0/localhost}"
  printf 'Open %s\n' "$url"
}

# Print a one-time browser pairing link for the current service URL.
opencode-pair() {
  local url
  url="$(opencode service status)" || return
  url="${url/127.0.0.1/localhost}"
  url="${url/0.0.0.0/localhost}"
  opencode pair --url "$url"
}

# Stop the shared OpenCode service.
opencode-server-stop() {
  opencode service stop
}
# <<< opencode helpers <<<
EOF

# Keep Oh My Posh as the very last line
if grep -q 'oh-my-posh init bash' ~/.bashrc; then
  omp_init="$(grep 'oh-my-posh init bash' ~/.bashrc | tail -n 1)"
  awk -v omp="$omp_init" \
    'index($0, "oh-my-posh init bash") == 0 { print } END { print ""; print omp }' \
    ~/.bashrc > ~/.bashrc.tmp &&
    cat ~/.bashrc.tmp > ~/.bashrc &&
    rm -f ~/.bashrc.tmp
fi

source ~/.bashrc
```

The block ends with `source ~/.bashrc`, so the functions are available immediately. `opencode-server` starts the shared service and prints its current URL. `opencode-pair` prints a one-time login link for that URL. `opencode-server-stop` stops it when you want to free the RAM.

> [!IMPORTANT]
> Keep the functions above `eval "$(oh-my-posh init bash ...)"` so the prompt still initializes last. The block above enforces this automatically, so you do not need to edit `~/.bashrc` by hand.

### 3. Start the server

```bash
opencode-server
```

OpenCode starts the shared service and prints its current URL, for example:

```text
Open http://localhost:49374
```

Open that URL in Windows.

On the first visit — or after the 30-day session expires — sign in once:

```bash
opencode-pair
```

Open the printed link in the browser. It is one-time and expires in 5 minutes. After that, the browser session lasts 30 days, so normal use is just `opencode-server` and opening the URL.

> [!NOTE]
> The helpers read the address from `opencode service status` and replace `127.0.0.1` or `0.0.0.0` with `localhost`, because a Windows browser cannot reach the WSL loopback address directly. If you change the service port, the helpers follow it automatically.

### 4. Stop the server

```bash
opencode-server-stop
```

This stops the shared service and frees its RAM. The browser session cookie remains, but the web UI will not respond until you run `opencode-server` again. After restarting, just open the URL; you only need `opencode-pair` again if the 30-day session has expired.

Check whether it is running with:

```bash
opencode service status
```

> [!NOTE]
> The shared service does not start on boot. It starts when you run `opencode-server` or when another OpenCode client needs it. If another client starts it automatically, that is normal.

> [!TIP]
> The service listens on loopback. If you expose it to a network, use an SSH tunnel or a TLS reverse proxy; do not send the password over plain HTTP on an untrusted network.

---

## Phase 8: GitHub CLI & Git Configuration

Install the GitHub CLI, authenticate your account, and set global Git identity:

```bash
# Debian's gh package is old; use GitHub's official apt repository.
(type -p wget >/dev/null || (sudo apt update && sudo apt install -y wget)) \
  && sudo mkdir -p -m 755 /etc/apt/keyrings \
  && out=$(mktemp) && wget -nv -O$out https://cli.github.com/packages/githubcli-archive-keyring.gpg \
  && cat $out | sudo tee /etc/apt/keyrings/githubcli-archive-keyring.gpg >/dev/null \
  && rm -f $out \
  && sudo chmod go+r /etc/apt/keyrings/githubcli-archive-keyring.gpg \
  && echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | \
       sudo tee /etc/apt/sources.list.d/github-cli.list >/dev/null \
  && sudo apt update \
  && sudo apt install -y gh

# Authenticate with GitHub (follow terminal prompts)
gh auth login

# Set global Git identity. The noreply address must use your numeric GitHub ID.
git config --global user.name "Full Name"
git config --global user.email "$(gh api user --jq '.id')+$(gh api user --jq '.login')@users.noreply.github.com"
```

---

## Phase 9: WSLg Desktop Libraries & Build Dependencies

Install essential build utilities, Wayland/X11 client libraries, OpenGL/EGL runtime packages for WSLg GUI apps, and configure FUSE for AppImage support:

```bash
sudo apt update && sudo apt install -y \
    curl \
    wget \
    binutils \
    patchelf \
    build-essential \
    libfuse2 \
    fuse3 \
    file \
    desktop-file-utils \
    libxkbcommon0 \
    libxkbcommon-x11-0 \
    libxcb-cursor0 \
    libxcb-icccm4 \
    libxcb-image0 \
    libxcb-keysyms1 \
    libxcb-randr0 \
    libxcb-render-util0 \
    libxcb-shape0 \
    libxcb-shm0 \
    libxcb-sync1 \
    libxcb-xfixes0 \
    libxcb-xinerama0 \
    libxcb-xkb1 \
    libwayland-client0 \
    libwayland-cursor0 \
    libwayland-egl1 \
    libegl1 \
    libgl1 \
    libopengl0 \
    libdbus-1-3 \
    libfontconfig1 && \
sudo ln -sf /usr/bin/fusermount3 /usr/bin/fusermount
```

> [!NOTE]
> The `fusermount` symlink is a compatibility shim for AppImages that still call FUSE2's `fusermount`; Debian ships `fusermount3`.

## Phase 10: Upgrade script

Paste the block below **once** into your Debian terminal. It writes the helper into `~/.bashrc`, then moves the Oh My Posh `eval` line back to the very bottom so the prompt keeps working. Re-pasting is safe: the old copy of the helper is removed first, and a single `upgrade` command is left behind.

```bash
# Remove a previously added copy of the helper (makes re-pasting safe)
if grep -q '^# >>> system upgrade helper >>>$' ~/.bashrc &&
   grep -q '^# <<< system upgrade helper <<<$' ~/.bashrc; then
  awk '
    /^# >>> system upgrade helper >>>$/ { skip=1; next }
    /^# <<< system upgrade helper <<<$/ { skip=0; drop_blank=1; next }
    skip { next }
    drop_blank && /^$/ { drop_blank=0; next }
    { drop_blank=0; print }
  ' ~/.bashrc > ~/.bashrc.tmp &&
    cat ~/.bashrc.tmp > ~/.bashrc &&
    rm -f ~/.bashrc.tmp
fi

cat << 'EOF' >> ~/.bashrc
# >>> system upgrade helper >>>
# Print each upgrade command before running it.
_upgrade_step() {
  printf '\n\033[1;34m==> Running: %s\033[0m\n' "$*"
  "$@"
}

# Upgrade apt packages and the user-level toolchains installed in this guide.
upgrade() {
  local failed=0

  _upgrade_step sudo apt update || failed=1
  _upgrade_step sudo apt upgrade -y || failed=1
  _upgrade_step fnm install --lts || failed=1
  _upgrade_step fnm default lts-latest || failed=1
  _upgrade_step fnm use lts-latest || failed=1
  _upgrade_step npm install -g npm@latest || failed=1
  _upgrade_step bun upgrade || failed=1
  _upgrade_step bun update -g pnpm || failed=1
  _upgrade_step uv self update || failed=1
  _upgrade_step uv python upgrade 3.11 3.12 3.13 3.14 || failed=1
  _upgrade_step agy update || failed=1
  _upgrade_step opencode upgrade || failed=1
  _upgrade_step oh-my-posh upgrade || failed=1

  if [ "$failed" -ne 0 ]; then
    printf '\nSome upgrade steps failed; see the output above.\n' >&2
    return 1
  fi
}
# <<< system upgrade helper <<<
EOF

# Keep the Oh My Posh init as the very last line of ~/.bashrc
if grep -q 'oh-my-posh init bash' ~/.bashrc; then
  omp_init="$(grep 'oh-my-posh init bash' ~/.bashrc | tail -n 1)"
  awk -v omp="$omp_init" \
    'index($0, "oh-my-posh init bash") == 0 { print } END { print ""; print omp }' \
    ~/.bashrc > ~/.bashrc.tmp &&
    cat ~/.bashrc.tmp > ~/.bashrc &&
    rm -f ~/.bashrc.tmp
fi

source ~/.bashrc
```

Then run it any time with:

```bash
upgrade
```

> [!NOTE]
> The block appends the helper and *then* re-adds the Oh My Posh line, so Oh My Posh still runs last. If you ever add shell config by hand, put it above the `eval "$(oh-my-posh init bash ...)"` line.
