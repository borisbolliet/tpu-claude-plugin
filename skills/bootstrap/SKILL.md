---
description: SSH into an existing TPU VM and install the python3.12 + jax + tunix + flax + libtpu stack. Idempotent — safe to re-run. Uses `uv` to fetch python3.12 (the deadsnakes PPA is unreachable from internal-IP TPU subnets), then runs the install in the order required for tunix to not stomp on flax.
disable-model-invocation: true
argument-hint: "<tpu-name> [--zone=europe-west4-a] [--project=tpu-2026]"
allowed-tools: Read Bash(gcloud *) Bash(ssh *) Bash(scp *)
---

# Bootstrap the python venv on a TPU VM

Run the `bootstrap.sh` recipe from `borisbolliet/tpu-2026` on a
freshly-created TPU VM. Installs python3.12 via uv, creates
`~/venvs/tunix`, and installs the jax / tunix / qwix / flax / libtpu
stack in the required order.

Arguments in `$ARGUMENTS`:

- `$0` — TPU VM name (required).
- `--zone=ZONE` — default `europe-west4-a`.
- `--project=PROJECT_ID` — default `tpu-2026`.

## Steps

1. **Verify the VM is reachable** via IAP tunnel:
   ```bash
   gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap \
       -- "echo OK"
   ```
   If this fails, the VM isn't ready (still creating, or the IAP
   firewall rule is missing — re-run `/tpu:setup`).

2. **Install uv + python3.12** on the VM:
   ```bash
   gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap \
       -- "set -e
           if ! command -v uv >/dev/null 2>&1; then
               curl -LsSf https://astral.sh/uv/install.sh | sh
               echo 'export PATH=\"\$HOME/.local/bin:\$PATH\"' >> ~/.bashrc
           fi
           export PATH=\"\$HOME/.local/bin:\$PATH\"
           uv python install 3.12"
   ```
   Do NOT try `sudo apt install python3.12` — the PPA is blocked
   from internal-IP subnets.

3. **Clone the tpu-2026 repo and run `bootstrap.sh`**:
   ```bash
   gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap \
       -- "set -e
           cd ~
           if [[ ! -d tpu-2026 ]]; then
               git clone https://github.com/borisbolliet/tpu-2026.git
           fi
           cd tpu-2026 && ./bootstrap.sh"
   ```

4. **Smoke test**: confirm jax sees the TPU.
   ```bash
   gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap \
       -- "source ~/venvs/tunix/bin/activate && \
           python -c 'import jax; print(jax.default_backend(), jax.devices())'"
   # expect: tpu [TpuDevice(...)]
   ```

5. **Optional**: pull secrets from Secret Manager into `~/.env` and
   wire `~/.bashrc` to auto-activate the venv on login. Ask the
   user before doing this — it requires a pre-existing secret named
   `tunix-env`:
   ```bash
   gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap \
       -- "set -e
           gcloud secrets versions access latest --secret=tunix-env \
               --project=$PROJECT_ID > ~/.env && chmod 600 ~/.env
           echo 'source ~/venvs/tunix/bin/activate' >> ~/.bashrc
           echo '[ -f ~/.env ] && set -a && source ~/.env && set +a' >> ~/.bashrc"
   ```

6. **Report back**: jax backend + device list, venv path, suggested
   next (`/tpu:connect --jupyter` for a notebook session).

## Pitfalls

- **`apt-get install python3.12` does NOT work** on internal-IP
  subnets. `ppa.launchpadcontent.net:443` is not in Google's NAT
  egress whitelist. Always use `uv python install 3.12`.
- **`libtpu` is REQUIRED for jax to see the TPU**, but
  `pip install git+...jax` does NOT pull it. The bootstrap script's
  `pip install -r requirements.txt` step pins it.
- **flax install order**: `pip install git+...tunix` downgrades
  `flax` to PyPI release; you must `uninstall -y flax && pip
  install git+...flax` AFTER tunix to get the GitHub version. The
  bootstrap script handles this.
- **First `pip install` after VM creation takes ~5 min** because the
  pip cache is cold and the wheels come from PyPI over Cloud NAT.
  Subsequent installs are much faster.
- **Cloud NAT must be in place** before this skill runs — if it
  isn't, every `pip install` hangs. Run `/tpu:setup` first.
