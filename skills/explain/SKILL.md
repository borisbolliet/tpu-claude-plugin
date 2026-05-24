---
description: Google Cloud TPU v6e (and v5e) workflow knowledge — VM creation with internal IPs + Cloud NAT + IAP, python3.12 bootstrap via uv (the PPA path is blocked from internal-IP VMs), the jax-from-git / tunix / qwix / flax install order with libtpu, SSH + Jupyter via IAP tunnel, GitHub push from the VM via short-lived PAT. Auto-loads when the conversation is about TPU, gcloud, jax on TPU, tunix, flax, or running cosmology / ML on Google Cloud accelerators.
when_to_use: User mentions TPU, v6e, v5e, gcloud compute tpus, tunix, jax-on-tpu, libtpu, qwix, flax git, IAP tunnel, Cloud NAT, internal-IP VM, Private Google Access, europe-west4 / us-east5 / us-central2 TPU zones, bootstrap.sh, requirements.txt for TPU.
allowed-tools: Read Grep Glob Bash(gcloud *) Bash(ssh *) Bash(scp *) Bash(cat *) Bash(grep *) Bash(ls *) Bash(curl -sSL *)
---

# Google Cloud TPU helper

Help with end-to-end TPU v6e / v5e workflows: create a VM, set up the
network so the VM can reach the internet, bootstrap the python stack,
open an IAP-tunnelled SSH + Jupyter port-forward, and tear down when
done. Recipes adapted from
[`borisbolliet/tpu-2026`](https://github.com/borisbolliet/tpu-2026).

## What the user typically needs

| Phase | Skill | What it does |
| --- | --- | --- |
| First-time per region | `/tpu:setup` | one-time: enable Private Google Access on default subnet, create Cloud NAT, IAP firewall rule, then create the TPU VM |
| Each new VM (same region) | `/tpu:setup` | skips the one-time steps; just creates the VM |
| First login to a new VM | `/tpu:bootstrap` | installs python3.12 via `uv`, creates `~/venvs/tunix`, runs the tunix install order |
| Routine SSH / Jupyter | `/tpu:connect` | opens IAP tunnel SSH (+ optional port-forward for 8888 / 6006) |
| Cleanup | `/tpu:delete` | deletes the VM (does NOT delete NAT / firewall / subnet config) |

## Architecture

The VMs we run are **internal-IP only** (`--internal-ips` on
`gcloud compute tpus tpu-vm create`) for security + cost. This implies:

- **Cloud NAT** is required for outbound internet (pip, curl, git clone
  from github). Without it, every `pip install` times out.
- **IAP tunnel** is required for SSH (no public IP, can't use
  conventional `--ssh`).
- **Private Google Access** must be enabled on the subnet for the VM
  to reach Google Cloud APIs (Secret Manager, GCS, ...) without
  egressing through NAT.

All three are one-time, per-region setup; once configured, you can
create many VMs in that region without re-running them.

## Environment variables

The whole flow is driven by these:

```bash
export PROJECT_ID=tpu-2026                  # change to your GCP project
export ZONE=europe-west4-a                  # or us-east5-a, us-central2-b, ...
export REGION=${ZONE%-?}                    # derives europe-west4 from europe-west4-a
export TPU_NAME=boris                       # your VM name (per-zone-unique)
export ACCELERATOR_TYPE=v6e-1               # or v6e-4, v6e-8, v5e-1, ...
export VERSION=v2-alpha-tpuv6e              # runtime image; matches ACCELERATOR_TYPE
```

Common zones with TPU v6e availability (check GCP docs for current
status, this changes):

- `us-east5-a`, `us-east5-b` (Columbus, OH)
- `europe-west4-a`, `europe-west4-b` (Eemshaven, NL)
- `us-central2-b` (Council Bluffs, IA — older, mostly v4)
- `asia-northeast1-a` (Tokyo)

## One-time per region

```bash
# Private Google Access on the default subnet
gcloud compute networks subnets update default \
  --region=$REGION \
  --enable-private-ip-google-access \
  --project=$PROJECT_ID

# Cloud NAT — outbound internet for internal-IP VMs
gcloud compute routers create nat-router \
  --network=default --region=$REGION --project=$PROJECT_ID
gcloud compute routers nats create nat-config \
  --router=nat-router --region=$REGION \
  --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges \
  --project=$PROJECT_ID

# IAP firewall rule (global; one-time per project, not per region)
gcloud compute firewall-rules create allow-iap-ssh \
  --network=default --source-ranges=35.235.240.0/20 \
  --allow=tcp:22 --project=$PROJECT_ID
```

The `/tpu:setup` skill checks each of these and skips if already
present.

## Create the TPU VM

```bash
gcloud compute tpus tpu-vm create $TPU_NAME \
  --project=$PROJECT_ID --zone=$ZONE \
  --accelerator-type=$ACCELERATOR_TYPE \
  --version=$VERSION \
  --internal-ips
```

Typical create time: ~3–5 min. Status:
```bash
gcloud compute tpus tpu-vm list --zone=$ZONE --project=$PROJECT_ID
```

## SSH

Always via IAP tunnel (no public IP):
```bash
gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
  --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap
```

With port-forward for Jupyter (8888) + TensorBoard (6006):
```bash
gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
  --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap \
  -- -L 8888:localhost:8888 -L 6006:localhost:6006 -N
```

The `--alpha` track is required for `--tunnel-through-iap` on tpu-vm.

## Python bootstrap on the VM

Ubuntu 22.04 on these TPU VMs ships only with `python3.10` and
`python3.11`. The tunix stack needs `python3.12`. The deadsnakes PPA
is **NOT** reachable from internal-IP TPU subnets — `apt-get install
python3.12` fails with `Could not connect to ppa.launchpadcontent.net:443`
even with NAT configured. We use [`uv`](https://github.com/astral-sh/uv)
to fetch a prebuilt CPython from GitHub instead — no sudo, no PPA:

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
export PATH="$HOME/.local/bin:$PATH"
uv python install 3.12
```

Then the venv + tunix install order. Order matters — see
"Pitfalls" below for why each step is in this position:

```bash
"$(uv python find 3.12)" -m venv ~/venvs/tunix
source ~/venvs/tunix/bin/activate
pip install --upgrade pip setuptools wheel

# 1. PyPI batch
pip install python-dotenv kagglehub ipywidgets tensorflow tensorflow_datasets \
            tensorboardX transformers grain huggingface_hub datasets 'numpy>2'

# 2. jax from git (PyPI release lags tunix expectations)
pip install git+https://github.com/jax-ml/jax

# 3. tunix + qwix from git — pulls flax from PyPI as a dep
pip install git+https://github.com/google/tunix git+https://github.com/google/qwix

# 4. Replace the PyPI flax with the GitHub version AFTER tunix installs
pip uninstall -y flax
pip install git+https://github.com/google/flax

# 5. libtpu — REQUIRED for jax to see the TPU.  Pinned in requirements.txt.
pip install libtpu

# Sanity check
python -c "import jax; print(jax.default_backend(), jax.devices())"
# Expect: tpu [TpuDevice(...)]
```

The `/tpu:bootstrap` skill ssh's into the VM and runs the
`bootstrap.sh` from the tpu-2026 repo (which encodes all of the
above).

## Run a Jupyter notebook

Two terminals on the laptop.

**Terminal 1** — port-forward tunnel (stays running):
```bash
gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
  --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap \
  -- -L 8888:localhost:8888 -N
```

**Terminal 2** — SSH in normally and launch Jupyter on the TPU:
```bash
gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
  --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap
# on the TPU:
source ~/venvs/tunix/bin/activate
jupyter lab --no-browser --port=8888 --ip=127.0.0.1
```

Open `http://127.0.0.1:8888/lab?token=...` in the laptop browser.
The kernel to pick is **tunix** (registered by `bootstrap.sh` via
`python -m ipykernel install --user --name tunix`), not the default
`python3`.

## Pitfalls (learned the hard way)

- **`apt-get install python3.12` fails on internal-IP VMs** even
  with NAT, because `ppa.launchpadcontent.net:443` is not in
  Google's NAT egress whitelist. Use `uv python install 3.12`
  instead.
- **`libtpu` is required for jax to see the TPU**. Without it:
  ```
  WARNING:jax._src.xla_bridge:A Google TPU may be present on this
  machine, but either a TPU-enabled jaxlib or libtpu is not
  installed. Falling back to cpu.
  ```
  Installing `jax` from git does NOT pull `libtpu` — you must `pip
  install libtpu` (or the pinned `libtpu-nightly` if a specific
  jaxlib needs it).
- **flax install order**: `pip install git+...tunix` downgrades
  `flax` to the PyPI release as a transitive dep. You must
  `uninstall -y flax` and `pip install git+...flax` AFTER tunix.
- **`transformers` and `huggingface_hub` get downgraded** by the
  tunix install (to 4.57 and 0.36 respectively). This is expected
  — tunix pins specific versions.
- **`python-dotenv` not `dotenv`**: the correct PyPI name is
  `python-dotenv`; the bare `dotenv` package is unmaintained.
- **`setuptools >= 81` breaks tensorboard 2.20** (removed
  `pkg_resources`). Pin `setuptools<81` if you need tensorboard.
- **GitHub push from the VM needs a PAT**: the remote is HTTPS,
  not SSH (no SSH key on the VM by default). Use an inline
  credential helper so the token stays in the env and is never
  written to git config:
  ```bash
  git -c credential.helper='!f() { echo username=USER; echo password=$GITHUB_TOKEN; }; f' \
      push origin main
  ```
- **alpha SDK channel**: `--tunnel-through-iap` for tpu-vm is only
  in the `alpha` track. Use `gcloud alpha compute tpus tpu-vm ssh ...`
  not `gcloud compute tpus tpu-vm ssh ...`.
- **Internal-IP VMs need Cloud NAT for pypi**. Without NAT,
  `pip install jax` times out silently. Easy to miss because the
  symptom looks like a slow connection, not a config error.
- **TPU VM names are zone-unique, not project-unique** — same
  `TPU_NAME` in two different zones is allowed.

## Tear down

```bash
gcloud compute tpus tpu-vm delete $TPU_NAME \
  --project=$PROJECT_ID --zone=$ZONE
```

Note: this does NOT delete the Cloud NAT router, the IAP firewall
rule, or the subnet's Private Google Access setting — those persist
and let the next VM in the same region come up fast. Use
`/tpu:delete` for the right command + a reminder of what stays.

## Cost guidance (rough, as of 2025)

| Accelerator | On-demand $/chip/hour | Notes |
| --- | --- | --- |
| v6e-1  | ~$2.70 | smallest v6e, single chip |
| v6e-4  | ~$10.8 | useful for FSDP / small training |
| v6e-8  | ~$21.6 | full slice |
| v5e-1  | ~$1.20 | older, cheaper, less memory |
| v4-8   | ~$8.0  | legacy |

Real costs vary by region. Quote `gcloud billing accounts list`
+ pricing.cloud.google.com for the precise number. The
`tpu:explain` skill assumes on-demand pricing; spot is ~3× cheaper
but evictions need handling.

## Detailed reference

For full command reference, the bootstrap.sh contents, the
requirements.txt pin list, and the GitHub-push credentials pattern,
see [reference.md](reference.md).
