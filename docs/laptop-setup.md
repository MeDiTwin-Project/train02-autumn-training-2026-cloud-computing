# Working on your own laptop

During the training you work on your **training VM**: every tool is installed there, at the right
version, and your access to GitLab, the cluster and the object store is ready. This page is for
working through the exercises **on your own laptop** instead, for example to catch up on a day you
missed. It lists what to install, how to bring your access over from the VM, and how to check that
everything works.

You still need your trainee account (`traineeNN` and claim code): the laptop replaces the VM, not
the services the exercises use.

## 1. Operating system

Every command in the material is a `bash` command for Linux.

| Laptop | What to use |
|---|---|
| **Linux** | Your normal terminal. Ubuntu 24.04 or a recent Fedora/Rocky work. |
| **macOS** | The Terminal app, with [Homebrew](https://brew.sh/) for a few packages. |
| **Windows** | **WSL 2** with Ubuntu: open PowerShell as administrator, run `wsl --install -d Ubuntu-24.04`, restart, then open *Ubuntu* from the Start menu. Run **every** command of this page and of the course inside Ubuntu, not in PowerShell. |

You need administrator rights on the laptop to install Docker and the system packages.

## 2. What to install, and for which day

Use **these versions**: they are the ones on the training VMs, and the ones the exercises were tested
with.

| Tool | Version | Used on |
|---|---|---|
| Git | 2.40 or later | Day 4, Day 5 |
| Docker, with Compose v2 | Docker 27 or later | Days 2–5 |
| Python | 3.12 | Days 1, 4, 5 |
| `kubectl` | 1.33.3 | Days 3–5 |
| Helm | 3.18.6 | Days 3–5 |
| Trivy | **0.69.3** | Day 4 |
| cosign | 2.4.1 | Day 4 (section 4.8) |
| Dagger CLI | **0.18.10** | Day 4 (section 8B) |
| s3cmd | any recent | Day 1, Day 5 |
| `jq`, `curl` | any recent | Days 1, 4, 5 |
| Visual Studio Code | any recent | every day |

:::{warning} Trivy: exactly 0.69.3
Trivy 0.69.4, and the Docker images 0.69.5 and 0.69.6, were published by attackers in March 2026
([Day 4, section 4.5](https://training.pages.data.meditwin-project.eu/meditwin-autumn-training-2026/pipeline-security/#d4-4-5)). Do not install "the latest Trivy" from a
package manager without checking its version.
:::

:::{note} Dagger: the CLI version must match the engine
The Dagger CLI starts an engine of its own version, and the course module was written for 0.18.10.
A newer CLI regenerates the module differently, and the Day 4 steps no longer match.
:::

### 2.1 Docker

- **Linux:** install Docker Engine from the [official instructions](https://docs.docker.com/engine/install/)
  for your distribution, including the `docker-compose-plugin`. Then
  `sudo usermod -aG docker $USER`, log out and in again.
- **macOS:** install [Docker Desktop](https://docs.docker.com/desktop/setup/install/mac-install/) and
  start it once.
- **Windows:** install [Docker Desktop](https://docs.docker.com/desktop/setup/install/windows-install/),
  and in *Settings → Resources → WSL integration* enable it for **Ubuntu-24.04**.

Check, in the terminal you will use for the course:

```bash
docker run --rm hello-world
docker compose version
```

### 2.2 System packages

**Ubuntu (also inside WSL):**

```bash
sudo apt update
sudo apt install -y git curl jq python3.12 python3.12-venv
```

**macOS:**

```bash
brew install git jq python@3.12
```

### 2.3 `kubectl`, Helm, Trivy, cosign and Dagger, at the course versions

This script downloads the five tools from their official release pages into `~/.local/bin`. It works
on Linux and macOS, on Intel/AMD and on ARM (Apple Silicon) laptops.

```bash
cat > ~/install-course-tools.sh <<'EOF'
#!/usr/bin/env bash
set -euo pipefail
BIN="${BIN:-$HOME/.local/bin}"
mkdir -p "$BIN"
case "$(uname -s)" in
  Linux)  OS=linux;  TOS=Linux ;;
  Darwin) OS=darwin; TOS=macOS ;;
  *) echo "Unsupported system: $(uname -s)"; exit 1 ;;
esac
case "$(uname -m)" in
  x86_64|amd64)  ARCH=amd64; TARCH=64bit ;;
  arm64|aarch64) ARCH=arm64; TARCH=ARM64 ;;
  *) echo "Unsupported CPU: $(uname -m)"; exit 1 ;;
esac
TMP=$(mktemp -d); trap 'rm -rf "$TMP"' EXIT; cd "$TMP"

curl -fsSLo "$BIN/kubectl" "https://dl.k8s.io/release/v1.33.3/bin/$OS/$ARCH/kubectl"
curl -fsSL "https://get.helm.sh/helm-v3.18.6-$OS-$ARCH.tar.gz" | tar -xz
mv "$OS-$ARCH/helm" "$BIN/helm"
curl -fsSL "https://github.com/aquasecurity/trivy/releases/download/v0.69.3/trivy_0.69.3_${TOS}-${TARCH}.tar.gz" | tar -xz trivy
mv trivy "$BIN/trivy"
curl -fsSLo "$BIN/cosign" "https://github.com/sigstore/cosign/releases/download/v2.4.1/cosign-$OS-$ARCH"
curl -fsSL "https://github.com/dagger/dagger/releases/download/v0.18.10/dagger_v0.18.10_${OS}_${ARCH}.tar.gz" | tar -xz dagger
mv dagger "$BIN/dagger"
chmod +x "$BIN/kubectl" "$BIN/helm" "$BIN/trivy" "$BIN/cosign" "$BIN/dagger"
echo "Installed in $BIN: kubectl helm trivy cosign dagger"
EOF
bash ~/install-course-tools.sh
```

If your shell does not find them afterwards, add the folder to your `PATH`:

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc     # macOS with zsh: ~/.zshrc
source ~/.bashrc
```

### 2.4 The Python environment

On the VM, a virtual environment called `course` is active when you log in. Create the same one:

```bash
python3.12 -m venv ~/venvs/course
source ~/venvs/course/bin/activate
pip install --upgrade pip
pip install python-openstackclient openstacksdk cookiecutter pyyaml pytest requests s3cmd \
  scikit-learn==1.9.1 numpy==2.5.4 scipy==1.18.1 joblib==1.6.0 \
  flask==3.1.3 gunicorn==26.2.0 boto3==1.43.115 prometheus-client==0.26.0
```

The pinned versions matter on Day 5: the model is a Python pickle, and the service can load it only
with the scikit-learn version that wrote it.

Activate it in every new terminal, or make it automatic:

```bash
echo 'source ~/venvs/course/bin/activate' >> ~/.bashrc       # macOS with zsh: ~/.zshrc
```

## 3. Bring your access over from the VM

The VM holds four credentials that the laptop needs. The simplest way is to copy them from the VM,
while you still have it. Replace `traineeNN` and `<vm-ip>` with yours, and use the SSH key you
claimed for the VM ([Instructions, step 4](https://training.pages.data.meditwin-project.eu/meditwin-autumn-training-2026/instructions/)).

:::{danger} These files are passwords
Keep them in your home directory, readable only by you. Never put them in a Git repository, a shared
folder or a chat.
:::

### 3.1 GitLab over SSH (Days 4 and 5)

Copy the GitLab key and its `~/.ssh/config` entry:

```bash
scp -i ~/.ssh/meditwin-training-2026.pem cloud-user@<vm-ip>:.ssh/gitlab_training ~/.ssh/
chmod 600 ~/.ssh/gitlab_training
cat >> ~/.ssh/config <<'EOF'
Host gitlab.data.meditwin-project.eu
  User git
  IdentityFile ~/.ssh/gitlab_training
  IdentitiesOnly yes
EOF
ssh -T git@gitlab.data.meditwin-project.eu      # Welcome to GitLab, @traineeNN!
```

Then tell Git who you are, as on the VM:

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
git config --global init.defaultBranch main
```

:::{tip} Port 22 blocked on your network?
Some networks block SSH. Clone and push over HTTPS instead, with your trainee username and a GitLab
personal access token (scopes `read_repository`, `write_repository`) as the password, as described
in [Day 4, section 1](https://training.pages.data.meditwin-project.eu/meditwin-autumn-training-2026/git/).
:::

### 3.2 The cluster: `kubectl` and Helm (Days 3–5)

Download your kubeconfig from Rancher:

1. Log in to `https://rancher-training.apps.meditwin-project.eu` as `traineeNN`, with your claim
   code (**Log in with Local User**).
2. Open the training cluster: the **EMG** tile on the left.
3. In the top bar, press **Download KubeConfig** (the sheet-of-paper icon; hold the pointer over the
   icons to see their names), and save the file.

```bash
mkdir -p ~/.kube
mv ~/Downloads/ecmwf-meditwin-training.yaml ~/.kube/config     # the file name may differ
chmod 600 ~/.kube/config
kubectl config set-context --current --namespace=traineeNN
kubectl get pods            # "No resources found in traineeNN namespace." is a success
```

The file contains a token tied to your Rancher account. Alternatively, copy the VM's:
`scp -i ~/.ssh/meditwin-training-2026.pem cloud-user@<vm-ip>:.kube/config ~/.kube/config`.

### 3.3 The container registry (Days 2–5)

```bash
docker login gitlab.data.meditwin-project.eu:6443 -u traineeNN     # password: your GitLab password
```

### 3.4 The object store (Days 1 and 5)

Copy your `s3cmd` configuration, which holds your access key and secret key:

```bash
scp -i ~/.ssh/meditwin-training-2026.pem cloud-user@<vm-ip>:.s3cfg ~/.s3cfg
chmod 600 ~/.s3cfg
s3cmd ls s3://meditwin-training-2026-traineeNN
```

Day 5 reads the same keys from `~/.s3cfg` to create a Kubernetes Secret, so the file must be there.

The OpenStack application credential of [Day 1, section 7](https://training.pages.data.meditwin-project.eu/meditwin-autumn-training-2026/apis/) is an instructor demo:
there is nothing to set up for it.

## 4. What differs from the VM

**ARM laptops (Apple Silicon, some Windows laptops).** An image you build on the laptop is for ARM,
and the training cluster runs Intel/AMD nodes: a Pod with that image fails with `exec format error`.
When an image you build **by hand** goes to the cluster (Day 2 push, Day 3 deploy), build it for the
cluster:

```bash
docker build --platform linux/amd64 -t <image> .
```

Images built by the GitLab pipelines (Days 4 and 5) are already right: the runners are Intel/AMD.

**The Python environment is not active by default**, unless you added it to `~/.bashrc` (2.4).

**Ports on `localhost`.** Days 2 and 5 start services on ports such as 8000 and 8080. On a laptop
they may already be in use by another program: stop it, or change the port on the left of `-p`.

**Network.** The laptop must reach `gitlab.data.meditwin-project.eu` (HTTPS, SSH port 22, registry
port 6443), `rancher-training.apps.meditwin-project.eu`, `s3.training.meditwin-project.eu` and the
services under `*.training.meditwin-project.eu`. A company VPN or proxy can block some of them.

## 5. Check everything

Run this in the terminal you will use for the course. Every line should print a version or `ok`.

```bash
git --version
docker version --format 'docker {{.Server.Version}}' && docker compose version --short
python --version                                   # Python 3.12.x, from ~/venvs/course
python -c "import sklearn, flask, boto3; print('python packages ok, scikit-learn', sklearn.__version__)"
kubectl version --client | head -1                 # v1.33.3
helm version --short                               # v3.18.6
trivy --version | head -1                          # Version: 0.69.3
cosign version 2>&1 | grep GitVersion              # v2.4.1
dagger version                                     # v0.18.10
s3cmd ls s3://meditwin-training-2026-traineeNN >/dev/null && echo "object store ok"
ssh -T git@gitlab.data.meditwin-project.eu         # Welcome to GitLab, @traineeNN!
kubectl auth can-i create deployments              # yes
```

If one line fails, go back to its section above. If you are stuck, ask in the course channel with
the command and its full output, never with a key or a password.
