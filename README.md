# Egress Ward: agentic egress security on Dell Pro Precision GB10

Egress Ward watches the traffic leaving your network, flags suspicious egress with
deterministic rules and an Isolation Forest model, and has a local AI security agent
investigate each finding and recommend a block. Telemetry, detection and inference all
stay on one Dell Pro Precision GB10, with no cloud fallback. A person stays in charge: a
suggested block is applied only after an analyst approves it.

## Watch it first

The overview video is under [Releases](../../releases).

## How it works

| # | Stage | What happens | Check or human gate |
|---|---|---|---|
| 1 | Capture | A Squid egress proxy on the GB10 logs each request (destination, bytes, verdict). The collector posts canonical events to the SQLite ingestion API. For demos, `just demo-pipeline` and `just demo-live` generate synthetic traffic. | Events follow `contracts/event.schema.json`; `just contracts-check` and `just fixtures-check` validate the examples and fixtures. |
| 2 | Detect | Deterministic rules and the promoted Isolation Forest score each window and store rolling risk assessments for every event. | Detection keeps running on rules alone if the model artifact is missing or corrupt. Training writes measured false and true positive rates to `services/processing/artifacts/training-report.json`. |
| 3 | Investigate | The OpenClaw security agent reads the evidence through the `egress-ward-ingestion` MCP tools and records an investigation, using a local Qwen model served by vLLM behind LiteLLM. | Only five MCP tools are allowed (`query_events`, `get_evidence`, `get_rules`, `submit_investigation`, `recommend_policy`). The agent cannot approve or enforce anything. |
| 4 | Recommend | If the evidence warrants it, the agent submits a pending `deny_destination` recommendation. | The contract rejects any action type that is not on the allow list. The recommendation stays pending. |
| 5 | Approve | An analyst approves or rejects the recommendation in the dashboard. | **Human gate.** Nothing is blocked without an explicit analyst decision. |
| 6 | Enforce | Approved destinations go on ingestion's denylist. `infra/gb10/squid/sync-denylist.sh` writes it for Squid and reloads the proxy, so the repeated transfer is blocked. | Malformed entries are refused. If ingestion can't be reached, the current denylist stays in place. |

## Hardware

**Built and run on**

| Component | Built and run on |
|---|---|
| System | Dell Pro Precision GB10 |
| GPU (with memory) | NVIDIA GB10 Grace Blackwell superchip, 128 GB unified LPDDR5x (shared by CPU and GPU) |
| CPU | 20-core NVIDIA Grace Arm CPU (GB10) |
| Memory | 128 GB unified LPDDR5x (shared by CPU and GPU) |
| Storage | 4 TB NVMe SSD |
| OS | NVIDIA DGX OS |
| Software | OpenClaw gateway agent, OpenShell, FastMCP, LiteLLM, SQLite ingestion API, Isolation Forest/autoencoder detection, React dashboard (pnpm), uv, just (versions not stated; NemoClaw is not installed on the GB10 deployment) |

## What's in this repo

| Path | What |
|---|---|
| `agents/<name>/` | Self-contained agent projects (`hello-agent`, `business-agent`, `security-agent`): NemoClaw manifest (`agents.yaml`), OpenShell policy (`policy.yaml`), tests. They share the root `uv.lock`. |
| `services/ingestion/` | Durable SQLite ingestion API: events, findings, recommendations, decisions, denylist, GPU and memory telemetry, and the `egress-ward-ingestion` MCP server. |
| `services/processing/` | Deterministic rules and Isolation Forest/autoencoder detection pipeline (`egress-ward-process` CLI). |
| `services/collector/` | Reads the Squid access log and posts events to ingestion. |
| `services/demo/` | Demo traffic generation and the live and simulate CLIs (`just s demo ...`). |
| `services/dashboard/` | Egress Ward React dashboard, with its own pnpm lockfile and build. |
| `contracts/` | JSON Schemas and examples for events, findings, recommendations, approvals and enforcement results. |
| `fixtures/` | Canonical normal and suspicious event fixtures and Squid log samples. |
| `libs/agentkit/` | Shared Python package: FastAPI service factory and `agents.yaml`/`policy.yaml` validation. |
| `libs/contracts-py/` | Python models for the contracts, with schema-drift tests. |
| `libs/policies/` | Reusable OpenShell policy fragments. |
| `libs/skills/` | Shared OpenClaw/Claude skills (fan-out limits, data boundaries, egress audit, local inference routing), packaged in `claude-plugin/`. |
| `infra/gb10/` | GB10 deployment: vLLM and LiteLLM compose files, Squid proxy and denylist sync, Ansible playbooks, model notes. |
| `infra/openclaw/` | OpenClaw desired state: settings, the security agent's workspace (identity, memory, tools, prompts) and the vulnerability-intelligence cron job. |
| `scripts/` | Demo pipeline, demo data seed and clear, `doctor.sh` readiness checks, agent deploy and scaffolding. |
| `docs/` | Architecture, design decisions (ADRs), ports, roadmap and work plans. |

## Setup

Linux. The full stack runs on the Dell Pro Precision GB10; the local demo also runs on
a Linux or macOS workstation.

1. Install the toolchain pinned in `mise.toml`: `just`, `uv`, Node.js and `pnpm`.
2. Put this repo on the machine and, from its root, copy the environment template:
   `cp .env.example .env`. Set your own values for `LITELLM_MASTER_KEY` and
   `LITELLM_API_KEY`, and set `MGMT_BIND_ADDR` to the GB10's
   management-network address.
3. Install dependencies: `just each setup` and `just dashboard-setup`.
4. Check and test: `just check` and `just each test`.
5. Run the local demo: `just demo-pipeline`. It starts the ingestion API, posts 100
   synthetic events, scores them, stores the finding for investigation and opens the
   dashboard at `http://127.0.0.1:8300`. For continuous traffic with a suspicious burst
   every third cycle, run `just s ingestion run` in one terminal and `just demo-live` in
   another.
6. Reset between runs: `just demo-pipeline` resets `data/demo-pipeline.db` at start
   (set `DEMO_PIPELINE_RESET=0` to keep it). Against a running ingestion API,
   `just demo-clear` removes only the synthetic demo events and the records derived
   from them; `just demo-seed` adds them back.
7. On the GB10: add SSH aliases `hack` and `dell-gb10` for the box (see
   `infra/gb10/README.md`), populate the model cache, then run `just doctor local`,
   `just gb10-up` (OpenClaw), `just gb10-app-up` (ingestion, processing, dashboard) and
   `just gb10-squid-up` (egress proxy). Point client laptops at port 3128 on the GB10.
   To add an agent project, run `just new my-agent`. To deploy one, gate locally with
   `just check && just each test`, then run `just deploy hello-agent gb10.local` (add
   `--image` to ship a built image).
8. The full checklist, with the redeploy loop, model switching and Squid setup, is in
   [`infra/gb10/README.md`](infra/gb10/README.md). Installation steps for the whole
   appliance are in [`docs/exfiltration-protection-architecture.md`](docs/exfiltration-protection-architecture.md)
   §11.

## Known limits

- This is a prototype, not a production product.
- It has only been run on the Dell Pro Precision GB10.
- Demo traffic is synthetic and deterministic.
- The CVE, asset, model and analyst-feedback pages show fixture data by default. Set
  `VITE_ENABLE_DEMO_PAGES=false` to hide them.
- Blocks are only recommended until a person approves them.
- Dell renamed the project, its packages and CLI (`egress-ward-processing`,
  `egress-ward-process`, `egress-ward-ingestion`) and its example hostnames, and
  edited `uv.lock` to match. Dell has not run the team's code in this copy: the
  checks and tests (`just each check`, `just each test`) were not re-run after the
  rename. Run them before you rely on the demo.
- Squid sees a request only during or after it happens, so the first transfer completes.
  It is the repeated transfer, after approval and a denylist reload, that is blocked
  (`docs/design/0002-squid-denylist-is-the-enforcement-point.md`).
- On the GB10 the agent runs in an OpenClaw gateway service; NemoClaw is not installed
  in that deployment (`docs/design/0001-openclaw-gateway-replaces-nemoclaw.md`). The
  NemoClaw manifests under `agents/` are scaffolds.
- The OpenClaw gateway dashboard binds to all interfaces on port 18789 and the
  deployment does not enable a host firewall. Restrict access with your own network
  filtering.
- HTTPS traffic shows only the destination host and byte counts, not paths or
  payloads (no TLS interception).
- The `provision.sh models` step expects a USB model export. Otherwise, download the
  pinned Hugging Face revisions listed in `infra/gb10/README.md`.

## Credits

- **Created by the v-x team:** Rohan Hasabe, Brian Cripe, Shrivara K and Devashish.
  Original repo: [ric03uec/squidward](https://github.com/ric03uec/squidward). If this
  demo is useful to you, please star it.
- The project was originally named SquidWard. Dell retitled it Egress Ward and named
  the platform throughout as the Dell Pro Precision GB10.
- The team used BeadHive to plan and track the build.
- Silicon partner: NVIDIA.
- Upstream licence: MIT, Copyright (c) 2026 Devashish. It is kept in [`LICENSE`](LICENSE)
  and [`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md). No third-party repositories
  are bundled. Dependencies install from their own registries under their own licences.
- AI-generated content: parts of the code and docs were written with AI coding
  assistants during the build (see `infra/openclaw/promptlog.md` and `docs/epics/`).
  The security agent's findings and recommendations are AI-generated at run time.
- NVIDIA, Grace, Blackwell and DGX are trademarks of NVIDIA Corporation. Other
  product names (including Squid, vLLM, LiteLLM, Qwen, OpenClaw, React and Hugging
  Face) are trademarks of their respective owners. They are named only to describe
  the software this demo uses. Their use does not imply endorsement of Dell or of this
  demo by their owners, or endorsement of their products by Dell.

## Support

Logan Lawler, Dell Technologies: logan.lawler@dell.com

## License

MIT. See [`LICENSE`](LICENSE). The upstream copyright notice is kept, with a line added
for Dell Inc.'s changes. Upstream notices are listed in
[`THIRD_PARTY_NOTICES.md`](THIRD_PARTY_NOTICES.md).
