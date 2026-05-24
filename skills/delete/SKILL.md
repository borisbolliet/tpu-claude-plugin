---
description: Delete a TPU VM and (optionally) the per-region networking that was set up alongside it. By default deletes ONLY the VM — Cloud NAT, IAP firewall, and Private Google Access stay so the next VM in the same region comes up fast. Reports what stayed so the user can decide whether to clean up further.
disable-model-invocation: true
argument-hint: "<tpu-name> [--zone=europe-west4-a] [--also-network]"
allowed-tools: Read Bash(gcloud *) Bash(echo *)
---

# Tear down a TPU VM

Stop billing on a TPU VM. By default deletes ONLY the VM; the
per-region networking (Cloud NAT, IAP firewall, Private Google
Access) stays so the next VM in the same region comes up without
re-running `/tpu:setup`.

Arguments in `$ARGUMENTS`:

- `$0` — TPU VM name (required).
- `--zone=ZONE` — default `europe-west4-a`.
- `--project=PROJECT_ID` — default `tpu-2026`.
- `--also-network` — ALSO delete the Cloud NAT router, the IAP
  firewall rule, and toggle Private Google Access off. ONLY do this
  if you are abandoning the region entirely; the next VM here will
  need `/tpu:setup` to re-run those steps.

## Steps

1. **Confirm the VM exists**:
   ```bash
   gcloud compute tpus tpu-vm describe $TPU_NAME --zone=$ZONE \
       --project=$PROJECT_ID --format="value(state,acceleratorType,createTime)"
   ```
   If it doesn't exist, report and exit.

2. **Show what will be deleted + ask for confirmation**:
   ```
   About to DELETE:
     VM:  $TPU_NAME (in $ZONE, accelerator $ACCEL, created $CREATED)
   ```
   If `--also-network`, append:
   ```
     Cloud NAT router:        nat-router (region $REGION)
     IAP firewall rule:       allow-iap-ssh (project-wide)
     Private Google Access:   will be DISABLED on default subnet ($REGION)
   ```
   Wait for explicit confirmation before mutating.

3. **Delete the VM**:
   ```bash
   gcloud compute tpus tpu-vm delete $TPU_NAME \
       --project=$PROJECT_ID --zone=$ZONE --quiet
   ```
   Wall time: ~30 seconds. The VM stops billing as soon as the
   delete starts.

4. **(If `--also-network`)** delete per-region networking:
   ```bash
   gcloud compute routers nats delete nat-config \
       --router=nat-router --region=$REGION --project=$PROJECT_ID --quiet
   gcloud compute routers delete nat-router \
       --region=$REGION --project=$PROJECT_ID --quiet
   gcloud compute firewall-rules delete allow-iap-ssh \
       --project=$PROJECT_ID --quiet
   gcloud compute networks subnets update default --region=$REGION \
       --no-enable-private-ip-google-access --project=$PROJECT_ID
   ```

5. **Report back**:
   - What was deleted (VM ± network).
   - What stayed (and why):
     ```
     Kept (for the next VM in $REGION):
       - Cloud NAT router "nat-router"
       - IAP firewall rule "allow-iap-ssh"
       - Private Google Access on default/$REGION
     ```
   - Total VM cost since creation (compute from `createTime` and
     the rate from `--accelerator-type`).

## Pitfalls

- **Delete is final and immediate** — no soft-delete, no recovery.
  Confirm before mutating, especially for VMs with unsaved work.
- **The VM's local disk is wiped**. Anything in `/home/<user>/` is
  gone unless you copied it off (e.g. to GCS) or pushed to GitHub.
  Remind the user before deletion if the VM is recent.
- **Cloud NAT keeps billing a small idle cost** when there's no VM
  using it (~$0.045/hour per NAT gateway + ~$0.045/hour per public
  IP). For frequent rebuilds, leave it. For long pauses, use
  `--also-network`.
- **IAP firewall is project-wide, not region-specific**. Deleting
  it affects every region.
