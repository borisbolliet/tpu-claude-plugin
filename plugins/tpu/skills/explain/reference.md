# TPU — detailed reference

Load when the SKILL.md summary isn't enough.

## Full `bootstrap.sh` (from tpu-2026)

```bash
#!/usr/bin/env bash
# Set up the tunix venv on a TPU VM that already has python3.12 installed.
# Prereq: python3.12 must already be on PATH (or findable via `uv python find 3.12`).
set -euo pipefail

REPO_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)
VENV=${VENV:-$HOME/venvs/tunix}

# Locate python3.12 — installed via `uv python install 3.12` on these VMs
export PATH="$HOME/.local/bin:$PATH"
if command -v python3.12 >/dev/null 2>&1; then
  PYTHON312=$(command -v python3.12)
elif command -v uv >/dev/null 2>&1 && PYTHON312=$(uv python find 3.12 2>/dev/null); then
  :
else
  echo "ERROR: python3.12 not found. Run: curl -LsSf https://astral.sh/uv/install.sh | sh && uv python install 3.12"
  exit 1
fi

# Create venv
"$PYTHON312" -m venv "$VENV"
source "$VENV/bin/activate"
pip install --upgrade pip setuptools wheel

# Install order matters — see SKILL.md "Pitfalls"
pip install python-dotenv kagglehub ipywidgets tensorflow tensorflow_datasets \
            tensorboardX transformers grain huggingface_hub datasets 'numpy>2'
pip install git+https://github.com/jax-ml/jax
pip install git+https://github.com/google/tunix git+https://github.com/google/qwix
pip uninstall -y flax
pip install git+https://github.com/google/flax
pip install -r "$REPO_DIR/requirements.txt"   # pins libtpu + tensorboard + others

# Register the venv as a Jupyter kernel
python -m ipykernel install --user --name tunix --display-name "tunix"
```

## Verifying jax sees the TPU

```python
import jax
print("backend:", jax.default_backend())   # expect: tpu
print("devices:", jax.devices())           # expect: [TpuDevice(id=0, ...)]
print("device count:", jax.device_count()) # expect: number of chips
```

If the backend is `cpu`:
1. `pip install libtpu` (most common fix)
2. Check `pip list | grep -E "jax|libtpu"` — both must be present
3. `dmesg | grep TPU` — confirm the TPU hardware is visible

## VM image versions

| Accelerator | `--version` flag |
| --- | --- |
| v6e-* | `v2-alpha-tpuv6e` |
| v5p-* | `v2-alpha-tpuv5` |
| v5e-* | `tpu-vm-tf-2.16.1-pjrt` (or `v2-alpha-tpuv5-lite`) |
| v4-*  | `tpu-vm-tf-2.16.1-pod` |

The `v2-alpha-tpuv6e` image ships with python3.10 + python3.11, and
the deadsnakes PPA preconfigured (but unreachable from internal-IP
subnets — see SKILL.md).

## Common `requirements.txt` pins for tunix

```
libtpu                       # required; jax-from-git doesn't pull this
tensorboard                  # explicit; TF 2.21 dropped the hard dep
setuptools<81                # tensorboard 2.20 still imports pkg_resources
jupyterlab
ipykernel
```

Everything else flows from the tunix install graph (`pip install
git+...tunix` pulls flax + protobuf + safetensors + ml_dtypes + ...).

## Secrets via Secret Manager

```bash
# Write a secret (one-time, from your laptop):
gcloud secrets create tunix-env --project=tpu-2026
echo -e "GITHUB_TOKEN=ghp_...\nKAGGLE_USERNAME=...\nKAGGLE_KEY=..." | \
    gcloud secrets versions add tunix-env --data-file=- --project=tpu-2026

# Read on the TPU (uses Private Google Access — no NAT egress):
gcloud secrets versions access latest --secret=tunix-env --project=tpu-2026 > ~/.env
chmod 600 ~/.env
```

Wire it into `~/.bashrc`:
```bash
echo '[ -f ~/.env ] && set -a && source ~/.env && set +a' >> ~/.bashrc
```

## GitHub push pattern from a TPU VM

The remote is HTTPS (no SSH key on the VM by default). The token
should live in `GITHUB_TOKEN` from `~/.env` and be passed inline via
a credential helper — never written to disk:

```bash
git -c credential.helper='!f() { echo username=YOUR_GH_USER; echo password=$GITHUB_TOKEN; }; f' \
    push origin main
```

Convenient alias for `~/.bashrc`:
```bash
alias gp='git -c credential.helper="!f() { echo username=$GH_USER; echo password=$GITHUB_TOKEN; }; f" push'
```

Most `~/.bashrc` returns early in non-interactive shells, so
`GITHUB_TOKEN` only loads in interactive sessions. For scripts,
source `~/.env` explicitly.

## Networking layout (per region)

```
   project default VPC
       |
       +-- subnet "default" (one per region; auto-mode by default)
       |     |
       |     +-- Private Google Access  ON  ─── reach Google APIs without NAT
       |     |
       |     +-- TPU VM (internal IP only)
       |
       +-- Cloud Router "nat-router" (one per region)
       |     |
       |     +-- NAT config "nat-config" (auto-allocate external IPs)
       |
       +-- Firewall rule "allow-iap-ssh"  (global; 35.235.240.0/20 -> tcp:22)
```

This is the *minimum* network footprint for an internal-IP TPU VM
that needs (a) outbound internet for pip/pip-from-git, (b) SSH via
IAP, (c) Secret Manager / GCS access via Private Google Access.

## Quotas + regions

TPU v6e on-demand quotas are LOW by default (often 0 or 1 chip per
region). To increase:

1. https://console.cloud.google.com/iam-admin/quotas
2. Filter by "TPU v6e Pod" (or your accelerator family)
3. Pick the region; request the chip count you need
4. SLAs vary — a v6e-1 bump is usually <24h, v6e-256 can take a week

`gcloud quotas list --service=compute.googleapis.com \
  --project=$PROJECT_ID --filter="quotaInfo.quotaId~v6e"` gives the
machine-readable list but the human web UI is faster for one-shots.

## Quick troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| `gcloud compute tpus tpu-vm create` hangs at "creating" >10min | quota exhausted | check `gcloud compute tpus tpu-vm list`, increase quota |
| `pip install` hangs / times out | no Cloud NAT in this region | run the Cloud NAT one-time setup |
| `apt-get install python3.12` fails (`ppa.launchpadcontent.net`) | PPA blocked from internal-IP subnet | use `uv python install 3.12` |
| `jax.default_backend() == 'cpu'` on a TPU VM | `libtpu` not installed | `pip install libtpu` |
| SSH "Permission denied" | not using `--tunnel-through-iap` | use the alpha-track SSH command |
| `scp: connect to host 10.x.x.x port 22: Operation timed out` | scp tried the internal IP, no IAP tunnel | use `gcloud alpha compute tpus tpu-vm scp --tunnel-through-iap`; see the `transfer` skill |
| `unrecognized arguments: --tunnel-through-iap` for scp/ssh | wrong gcloud release track | always use `gcloud alpha compute tpus tpu-vm ...` for tpu-vm ssh/scp |
| scp silently skipped a directory | omitted `--recurse` | re-run with `--recurse` |
| `from cobaya import LoggedError` fails | CWD has a `cobaya/` folder shadowing the install | `cd /tmp` first |

## File transfer (scp via IAP)

For copying files between a TPU VM (internal-IP only) and your
laptop, use the alpha-track `scp` over the IAP tunnel — same
auth path as the SSH command. Concrete recipes + push direction
+ pitfalls live in the `transfer` skill (`/tpu:transfer`); the
one-line summary:

```bash
# pull
gcloud alpha compute tpus tpu-vm scp --recurse \
    --zone=$ZONE --project=$PROJECT_ID --tunnel-through-iap \
    $TPU_NAME:$REMOTE_PATH $LOCAL_PATH

# push (swap src and dst)
gcloud alpha compute tpus tpu-vm scp --recurse \
    --zone=$ZONE --project=$PROJECT_ID --tunnel-through-iap \
    $LOCAL_PATH $TPU_NAME:$REMOTE_PATH
```

Three points the troubleshooting table can't capture in one line:

- Single files work without `--recurse`; the flag is required only
  for directories and harmless otherwise — default to including it.
- The IAP tunnel is slow without NumPy in the laptop's gcloud python
  (the first run prints a warning with a link). Fine for hundreds of
  MB, worth fixing for multi-GB.
- `scp` does not resume mid-transfer. For multi-GB payloads on a
  flaky link, prefer `rsync` over a manually-opened IAP tunnel, or
  `split -b 500M` the source and `cat` it back on the destination.
