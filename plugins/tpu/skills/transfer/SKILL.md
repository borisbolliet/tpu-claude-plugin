---
description: Copy files between a TPU VM and the user's laptop via IAP-tunnelled scp. Typical use is pulling tarballs / outputs off a VM before deleting it, or pushing a dataset / script up. Knows the alpha-track command, --recurse, --tunnel-through-iap, internal-IP gotchas, and prints the exact command for the user to run from their own terminal.
argument-hint: "<tpu-name>:<remote-path> <local-path>   (or reverse for push)   [--zone=europe-west4-a] [--project=tpu-2026]"
allowed-tools: Read Bash(gcloud *) Bash(echo *) Bash(printf *) Bash(ls *)
---

# Copy files to / from a TPU VM (IAP-tunnelled `scp`)

Generate the right `gcloud alpha compute tpus tpu-vm scp` command for
the given source / destination. The transfer itself runs in the user's
terminal — this skill prepares and prints the command, then explains
how to verify the result.

Arguments in `$ARGUMENTS`:

- `$0` — source path. Either `<tpu-name>:<remote-path>` (pull from
  VM) or a local path (push to VM).
- `$1` — destination path. Mirror of `$0`: local path for a pull,
  `<tpu-name>:<remote-path>` for a push.
- `--zone=ZONE` — default `europe-west4-a`.
- `--project=PROJECT_ID` — default `tpu-2026`.
- `--no-recurse` — omit `--recurse`. Default is to include it (safe
  for both files and directories).

## Steps

1. **Verify the VM is reachable**:

   ```bash
   gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
       --zone=$ZONE --project=$PROJECT_ID \
       --tunnel-through-iap --command='echo ok'
   ```

   If this hangs or errors with `connect to host 10.x.x.x port 22:
   Operation timed out`, see Pitfalls below — almost always IAP
   isn't set up.

2. **Build the scp command** (default direction: pull from VM):

   ```bash
   gcloud alpha compute tpus tpu-vm scp --recurse \
       --zone=$ZONE --project=$PROJECT_ID \
       --tunnel-through-iap \
       $TPU_NAME:$REMOTE_PATH \
       $LOCAL_PATH
   ```

   For a push, swap source and destination:

   ```bash
   gcloud alpha compute tpus tpu-vm scp --recurse \
       --zone=$ZONE --project=$PROJECT_ID \
       --tunnel-through-iap \
       $LOCAL_PATH \
       $TPU_NAME:$REMOTE_PATH
   ```

3. **Verify the transfer**:

   - File count: `ls -la $LOCAL_PATH` on the laptop side should
     match `ls -la $REMOTE_PATH` on the VM side.
   - For tarballs / checksummed payloads, run `sha256sum` on both
     sides — the values must match. If you transferred from /tmp on
     a VM that's about to be deleted, this is your last chance to
     catch corruption.

## Common recipes

### Pull every tarball in `/tmp/upload` to a local backup dir

```bash
mkdir -p ~/tpu-backup
gcloud alpha compute tpus tpu-vm scp --recurse \
    --zone=europe-west4-a --project=tpu-2026 \
    --tunnel-through-iap \
    boris:/tmp/upload/ \
    ~/tpu-backup/
```

Leaves `~/tpu-backup/upload/*.tar.gz` on the laptop.

### Pull a single big file (model checkpoint)

```bash
gcloud alpha compute tpus tpu-vm scp \
    --zone=europe-west4-a --project=tpu-2026 \
    --tunnel-through-iap \
    boris:~/checkpoints/step-50000.npz \
    ~/checkpoints/
```

Omit `--recurse` for single files; it still works with it, just
unnecessary noise.

### Push a local script / dataset to the VM

```bash
gcloud alpha compute tpus tpu-vm scp --recurse \
    --zone=europe-west4-a --project=tpu-2026 \
    --tunnel-through-iap \
    ~/my-experiment/ \
    boris:~/
```

## Pitfalls

- **`scp` without `--tunnel-through-iap` fails on internal-IP VMs**.
  These TPU VMs sit on a subnet with Private Google Access and no
  external IP. Your laptop cannot route to `10.x.x.x` directly. The
  symptom is `ssh: connect to host 10.164.0.8 port 22: Operation
  timed out`. Always add `--tunnel-through-iap`.

- **`gcloud compute tpus tpu-vm scp` (non-alpha) does NOT accept
  `--tunnel-through-iap`** as of late 2025 / early 2026. Always use
  the `alpha` channel for tpu-vm scp/ssh. The error from the stable
  channel is explicit:

  ```
  ERROR: unrecognized arguments: --tunnel-through-iap
   --tunnel-through-iap flag is available in one or more alternate
   release tracks. Try: gcloud alpha compute tpus tpu-vm ssh
  ```

- **First `alpha` invocation prompts for component install**. Say
  yes; subsequent runs are silent.

- **IAP firewall rule + API must exist on the project**. If even the
  step-1 ssh test (`--command='echo ok'`) hangs, the project needs:

  ```bash
  gcloud services enable iap.googleapis.com --project=$PROJECT_ID
  gcloud compute firewall-rules create allow-iap-ssh \
      --project=$PROJECT_ID \
      --direction=INGRESS --action=ALLOW --rules=tcp:22 \
      --source-ranges=35.235.240.0/20
  ```

  (35.235.240.0/20 is Google's IAP egress range — Google-published
  and stable.)

- **`--recurse` is required for directories**. Without it, scp will
  silently skip directory sources and only transfer top-level files
  in the source path. Default to including it.

- **IAP tunnel is slow without NumPy on the laptop**. The first run
  prints a `WARNING: To increase the performance of the tunnel,
  consider installing NumPy ...`. For a few hundred MB it's fine
  either way; for multi-GB transfers, install NumPy into whatever
  python `gcloud` uses (often `/usr/bin/python3` or the
  Cloud-SDK-bundled one). See
  https://cloud.google.com/iap/docs/using-tcp-forwarding#increasing_the_tcp_upload_bandwidth.

- **Sudden disconnects mid-transfer**: IAP tunnels can drop on
  flaky networks. `scp` resumes nothing — it restarts from zero.
  For multi-GB payloads, prefer `rsync` over a manually-opened IAP
  ssh tunnel, or split the payload into ~500 MB pieces with
  `split -b 500M` and `cat` them back together on the destination.

- **Don't put secrets in `/tmp` and forget them**. `/tmp` on a TPU
  VM is wiped on VM delete (and sometimes on reboot). If the
  payload matters, move it to `~/` before transfer and confirm.

- **Wrong source / destination order silently overwrites**. `scp
  A B` copies A → B. Double-check which side is which, especially
  for pushes — pushing an empty local directory over a populated
  remote one will wipe the remote.

## When to suggest an alternative to scp

- **Files already on GitHub**: `gh release download` from the
  laptop is one command, no networking setup. Prefer it when the
  files exist as release assets or repo content.
- **Pulling from many VMs in parallel**: write a small loop over
  `gcloud alpha compute tpus tpu-vm scp ...` rather than reaching
  for `rclone` / `rsync` — the IAP tunnel setup amortises and the
  loop is simpler than configuring a sync tool.
- **Pushing the same payload to many VMs**: stage it once in a GCS
  bucket and `gsutil cp` from each VM — IAP scp scales linearly,
  GCS scales O(1).
