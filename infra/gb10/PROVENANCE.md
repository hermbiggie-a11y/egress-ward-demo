# gb10: what ships with the device vs. what this deployment adds

**The rule:** anything that arrives with the Dell Pro Precision GB10 is the
**pre-installed** baseline, and this repo does not manage it. Everything this
deployment adds is reproduced by `provision.sh` and the Ansible playbooks.

## Pre-installed

### Factory image

Ubuntu Server 24.04.3 LTS "Noble Numbat" arm64 (NVIDIA DGX OS). `nsys` /
`nsys-ui` (Nsight Systems) are part of the image.

### First boot / NVIDIA-Dell OOBE

The NVIDIA DGX OS stack lands at first boot:

| What |
|---|
| NVIDIA driver (open), CUDA toolkit 13.0 |
| `dgx-dashboard` (the `:11000` web UI), `dgx-oobe`, `dgx-oobe-desktop` |
| `nvidia-ai-workbench`, `/opt/NVIDIA AI Workbench` |
| **Docker + `docker-compose-plugin` + `docker-buildx-plugin`** |
| Mellanox/RDMA (`rdma-core`, `mstflint`, `perftest`, `ucx-utils`) |
| `linux-nvidia-hwe-24.04` kernel |
| `dell-dgx-wallpaper`, `dgx-oobe-customize` (Dell OEM repo) |
| GNOME, Firefox, snapd bases |

Docker is **not** something this deployment installs — it comes with the box.
Note that OOBE **purged `unattended-upgrades`** on the device inventoried here,
so that machine did not self-patch.

### Access

`dell` is the only non-system account and has passwordless sudo. Review
`~/.ssh/authorized_keys` on a re-issued device and keep only the keys you need.

---

## Added by this deployment — not documented here

Everything this deployment adds is **adopted into IaC**, not described in
prose: `provision.sh` reproduces it and records where each piece came from,
and `README.md` covers the config and the redeploy loop.

One thing worth stating because its absence surprises people: `nemoclaw` is
**not** installed by this deployment. The serving stack is plain
Docker + vLLM + LiteLLM.
