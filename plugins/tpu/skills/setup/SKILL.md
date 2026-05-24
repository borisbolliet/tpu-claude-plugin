---
description: Create a TPU v6e (or v5e) VM in a given GCP zone. Handles one-time per-region setup (Private Google Access on default subnet, Cloud NAT router for outbound internet from internal-IP VMs, IAP firewall rule for SSH) only if not already present, then runs `gcloud compute tpus tpu-vm create`. Reports VM state and the SSH command to use next.
disable-model-invocation: true
argument-hint: "<tpu-name> [--zone=europe-west4-a] [--accelerator=v6e-1] [--project=tpu-2026]"
allowed-tools: Read Bash(gcloud *) Bash(echo *) Bash(printf *)
---

# Create a TPU VM

Bootstrap a new TPU v6e / v5e VM with the right networking (internal
IP + Cloud NAT + IAP). Mirrors the recipe in
[`borisbolliet/tpu-2026/tpu-setup.md`](https://github.com/borisbolliet/tpu-2026/blob/main/tpu-setup.md)
but generalises across regions and checks each one-time step before
running it.

Arguments in `$ARGUMENTS`:

- `$0` — TPU VM name (required, e.g. `boris`).
- `--zone=ZONE` — e.g. `europe-west4-a`, `us-east5-a`. Default
  `europe-west4-a`.
- `--accelerator=TYPE` — `v6e-1`, `v6e-4`, `v6e-8`, `v5e-1`, ...
  Default `v6e-1`.
- `--project=PROJECT_ID` — GCP project. Default `tpu-2026`.
- `--version=IMAGE` — runtime image. Default `v2-alpha-tpuv6e`
  (correct for any v6e accelerator).

## Steps

1. **Derive region from zone.** `REGION=${ZONE%-?}` strips the
   trailing `-a` / `-b` / `-c`.

2. **Print the plan + estimated cost** before any mutation:
   ```
   project   = tpu-2026
   zone      = europe-west4-a
   region    = europe-west4
   name      = boris
   chip type = v6e-1   (~$2.70/hour on-demand)
   image     = v2-alpha-tpuv6e
   ```
   Wait for the user to confirm if this is the first time in the
   session.

3. **Check + run the one-time per-region setup.** Each step checks
   whether the resource already exists:

   ```bash
   # Private Google Access on default subnet
   PGA=$(gcloud compute networks subnets describe default --region=$REGION \
       --project=$PROJECT_ID --format="value(privateIpGoogleAccess)" 2>/dev/null)
   if [[ "$PGA" != "True" ]]; then
       gcloud compute networks subnets update default \
           --region=$REGION --enable-private-ip-google-access \
           --project=$PROJECT_ID
   fi

   # Cloud NAT router
   if ! gcloud compute routers describe nat-router --region=$REGION \
        --project=$PROJECT_ID --format="value(name)" >/dev/null 2>&1; then
       gcloud compute routers create nat-router \
           --network=default --region=$REGION --project=$PROJECT_ID
       gcloud compute routers nats create nat-config \
           --router=nat-router --region=$REGION \
           --auto-allocate-nat-external-ips --nat-all-subnet-ip-ranges \
           --project=$PROJECT_ID
   fi

   # IAP firewall rule (global; one-time per project)
   if ! gcloud compute firewall-rules describe allow-iap-ssh \
        --project=$PROJECT_ID --format="value(name)" >/dev/null 2>&1; then
       gcloud compute firewall-rules create allow-iap-ssh \
           --network=default --source-ranges=35.235.240.0/20 \
           --allow=tcp:22 --project=$PROJECT_ID
   fi
   ```

4. **Check name collision in the same zone**:
   ```bash
   if gcloud compute tpus tpu-vm describe $TPU_NAME --zone=$ZONE \
       --project=$PROJECT_ID --format="value(name)" >/dev/null 2>&1; then
       echo "ERROR: TPU '$TPU_NAME' already exists in $ZONE"
       gcloud compute tpus tpu-vm list --zone=$ZONE --project=$PROJECT_ID
       exit 1
   fi
   ```

5. **Create the VM**:
   ```bash
   gcloud compute tpus tpu-vm create $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE \
       --accelerator-type=$ACCELERATOR_TYPE \
       --version=$VERSION \
       --internal-ips
   ```
   Typical wall time: 3–5 min.

6. **Verify it came up**:
   ```bash
   gcloud compute tpus tpu-vm describe $TPU_NAME --zone=$ZONE \
       --project=$PROJECT_ID --format="value(state,acceleratorType,health)"
   # expect: READY  v6e-1  HEALTHY
   ```

7. **Report back** with:
   - VM state + accelerator type
   - The SSH command:
     ```bash
     gcloud alpha compute tpus tpu-vm ssh $TPU_NAME \
         --project=$PROJECT_ID --zone=$ZONE --tunnel-through-iap
     ```
   - Suggested next: `/tpu:bootstrap` to install python3.12 + the
     tunix venv on the VM.
   - Cost reminder: the VM is now billing at `~$2.70/hr` (or
     whatever the accelerator type costs); delete when done.

## Pitfalls

- **Quota check fails AFTER the create command runs.** The error
  message looks generic ("creation has failed") — always check
  https://console.cloud.google.com/iam-admin/quotas for the
  region+accelerator family.
- **Wrong runtime image for accelerator family** — `v2-alpha-tpuv6e`
  for v6e, `v2-alpha-tpuv5` for v5p, `v2-alpha-tpuv5-lite` for v5e.
  If you create a v5e with the v6e image, the VM boots but jax sees
  no TPU.
- **`--internal-ips` is intentional**. Don't drop it — VMs with
  public IPs cost more and are needlessly exposed. The IAP tunnel +
  Cloud NAT recipe handles everything an internal-IP VM needs.
- **Zones differ in quota**: v6e-1 quota is per-region, not
  per-project. If you have quota in `us-east5` but not
  `europe-west4`, the create will fail with the same wording. Check
  per-zone before retrying.
