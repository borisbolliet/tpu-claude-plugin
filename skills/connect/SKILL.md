---
description: Open an IAP-tunnelled SSH session to a TPU VM, optionally with port-forward for Jupyter (8888) and TensorBoard (6006). Knows the alpha-track command + zone + project flags; can also launch the SSH command for the user to paste into their own terminal (interactive sessions don't survive an MCP tool invocation).
argument-hint: "<tpu-name> [--zone=europe-west4-a] [--jupyter] [--tensorboard]"
allowed-tools: Read Bash(gcloud *) Bash(echo *) Bash(printf *)
---

# SSH (with optional port-forwarding) into a TPU VM

Generate the right `gcloud alpha compute tpus tpu-vm ssh` command
for the given VM, with optional `-L 8888` / `-L 6006` port-forwards
for Jupyter + TensorBoard. Interactive SSH itself runs in the user's
terminal — this skill prepares the command and prints it.

Arguments in `$ARGUMENTS`:

- `$0` — TPU VM name (required).
- `--zone=ZONE` — default `europe-west4-a`.
- `--project=PROJECT_ID` — default `tpu-2026`.
- `--jupyter` — add `-L 8888:localhost:8888`.
- `--tensorboard` — add `-L 6006:localhost:6006`.
- `--detach` — append `-N` so the tunnel stays open without a shell
  (use for the port-forward-only terminal pattern).

## Steps

1. **Verify the VM exists + is reachable**:
   ```bash
   gcloud compute tpus tpu-vm describe $TPU_NAME --zone=$ZONE \
       --project=$PROJECT_ID --format="value(state,acceleratorType,health)"
   ```
   If state is not `READY`, abort with an explanation.

2. **Build the SSH command**:
   ```bash
   CMD="gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap"
   if --jupyter or --tensorboard:
       CMD="$CMD --"
       [[ $JUP    ]] && CMD="$CMD -L 8888:localhost:8888"
       [[ $TB     ]] && CMD="$CMD -L 6006:localhost:6006"
       [[ $DETACH ]] && CMD="$CMD -N"
   ```

3. **Tell the user what to do next**:
   - Without port-forward: paste the command in their terminal to
     get a shell on the VM.
   - With port-forward + `--detach`: open a SECOND terminal and run
     the plain SSH command to actually get a shell. The detached
     terminal is just the tunnel.
   - Mention what to do once logged in:
     ```bash
     # If Jupyter:
     source ~/venvs/tunix/bin/activate
     jupyter lab --no-browser --port=8888 --ip=127.0.0.1
     # then open http://127.0.0.1:8888/lab?token=... in your laptop browser

     # If TensorBoard:
     source ~/venvs/tunix/bin/activate
     tensorboard --logdir <your-log-dir> --port=6006 --host=127.0.0.1
     # then open http://localhost:6006
     ```

## Pitfalls

- **Interactive SSH cannot run inside a single Claude tool call**.
  The skill prepares the command; the user pastes it to their
  terminal.
- **`gcloud compute tpus tpu-vm ssh` (non-alpha) does NOT support
  `--tunnel-through-iap`**. Always use the `alpha` channel for
  tpu-vm SSH.
- **Jupyter token**: the URL has a `?token=...` query string that
  changes each launch. Always copy the full URL from the Jupyter
  startup log, not just the hostname.
- **Port conflict**: if 8888 or 6006 is already bound on the
  laptop, the SSH tunnel command silently succeeds but the local
  connection refuses. Kill the conflicting process or pick a
  different local port:
  `-L 8889:localhost:8888` and open `localhost:8889`.
- **Single tunnel for multiple ports**: prefer one SSH session with
  multiple `-L` flags over two separate tunnels.
