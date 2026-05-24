# Google Cloud TPU helper (Claude Code plugin)

Specialised [Claude Code](https://code.claude.com) assistance for
working with Google Cloud TPU v6e and v5e VMs. Captures the recipes
and pitfalls learned in
[`borisbolliet/tpu-2026`](https://github.com/borisbolliet/tpu-2026)
— including the ones that don't show up in the official docs
(deadsnakes PPA unreachable from internal-IP subnets, `libtpu`
required for jax to see the TPU, flax install order, alpha-track
needed for IAP SSH, ...).

## What you get

- **`/tpu:explain`** — always-on knowledge skill. Auto-loads when
  the conversation is about TPU, gcloud, jax-on-TPU, tunix, flax,
  or running cosmology / ML on Google Cloud accelerators. Covers
  the full architecture (Cloud NAT + IAP + Private Google Access),
  python bootstrap via `uv` (the PPA path is blocked), the
  jax / tunix / qwix / flax / libtpu install order, SSH + Jupyter
  via IAP tunnel, GitHub push via short-lived PAT, and quotas /
  cost guidance.

- **`/tpu:setup <name> [--zone=...] [--accelerator=v6e-1]`** —
  create a new TPU VM. Idempotently sets up the per-region networking
  (Private Google Access, Cloud NAT, IAP firewall) only if not
  already present.

- **`/tpu:bootstrap <name> [--zone=...]`** — SSH into the VM and run
  the `bootstrap.sh` from `tpu-2026`: install `python3.12` via
  `uv`, create `~/venvs/tunix`, install jax + tunix + qwix + flax +
  libtpu in the required order, smoke-test that
  `jax.default_backend() == 'tpu'`.

- **`/tpu:connect <name> [--jupyter] [--tensorboard]`** — generate
  the IAP-tunnelled SSH command with port-forward for Jupyter
  (8888) and / or TensorBoard (6006). The user pastes the command
  in their own terminal (interactive SSH doesn't survive a single
  tool call).

- **`/tpu:delete <name> [--also-network]`** — tear down the VM.
  By default keeps the per-region networking so the next VM comes
  up fast; `--also-network` does the full clean-up.

- **`tpu-engineer` subagent** — end-to-end specialist for heavy
  multi-step TPU work.

## Install

In any Claude Code session, run these three commands:

```
/plugin marketplace add https://github.com/borisbolliet/tpu-claude-plugin.git
/plugin install tpu@tpu-claude-plugin
/reload-plugins
```

After `/reload-plugins`, `/tpu:explain`, `/tpu:setup`, `/tpu:bootstrap`,
`/tpu:connect`, `/tpu:delete` show in `/help` and the `tpu-engineer`
subagent appears in the Agent picker.

To update later (after I push a new commit):

```
/plugin uninstall tpu@tpu-claude-plugin
/plugin marketplace remove tpu-claude-plugin
/plugin marketplace add https://github.com/borisbolliet/tpu-claude-plugin.git
/plugin install tpu@tpu-claude-plugin
/reload-plugins
```

## Environment

- `gcloud` CLI authenticated (`gcloud auth login`) with TPU
  permissions on your project.
- TPU v6e (or v5e) quota in the region you target. See
  https://console.cloud.google.com/iam-admin/quotas.
- The TPU itself is the only paid resource here (Cloud NAT idles at
  a few cents/hour); v6e-1 on-demand is ~$2.7/hour.

## Try it

```
/tpu:setup boris --zone=europe-west4-a --accelerator=v6e-1
/tpu:bootstrap boris
/tpu:connect boris --jupyter
```

When done:

```
/tpu:delete boris
```

## Layout

```
.claude-plugin/marketplace.json    # single-plugin marketplace
plugins/tpu/
  .claude-plugin/plugin.json       # plugin manifest
  skills/
    explain/SKILL.md               # always-on knowledge
    explain/reference.md           # loaded on demand
    setup/SKILL.md                 # /tpu:setup
    bootstrap/SKILL.md             # /tpu:bootstrap
    connect/SKILL.md               # /tpu:connect
    delete/SKILL.md                # /tpu:delete
  agents/
    tpu-engineer.md                # subagent
```

## License

MIT
