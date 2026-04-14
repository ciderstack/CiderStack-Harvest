<p align="center">
  <picture>
    <img alt="Harvest" src="https://ciderstack.sfo3.cdn.digitaloceanspaces.com/Images/ciderstack-logo-512.png" width="200">
  </picture>
</p>

<h3 align="center">CI/CD orchestrator for macOS virtual machines</h3>

<p align="center">
  Schedule jobs, manage ephemeral GitHub Actions runners, and orchestrate fleets of macOS VMs — all from a single binary.
</p>

<p align="center">
  <a href="#quickstart">Quickstart</a> &middot;
  <a href="#features">Features</a> &middot;
  <a href="#configuration">Configuration</a> &middot;
  <a href="#github-actions-runners">GitHub Runners</a> &middot;
  <a href="#sdk">SDK</a> &middot;
  <a href="#api-reference">API</a>
</p>

---

## What is CiderStack Harvest (formely Harvest)?

CiderStack Harvest is the orchestration layer for [CiderStack](https://ciderstack.com) macOS infrastructure. It turns a fleet of Mac machines into a programmable CI/CD platform with:

- **Ephemeral macOS VMs** — Clone a template, run your build, tear it down. Every job gets a clean environment.
- **GitHub Actions integration** — Self-hosted macOS runners that auto-register, run one job, and disappear.
- **Fleet management** — Pair nodes, monitor health, manage snapshots, execute commands — all through one dashboard.

Ships as a single compiled binary with a built-in web UI. No containers, no JVM, no external databases.

## Quickstart

### Run

```bash
cd Harvest-v*
./Harvest
```

That's it. Harvest creates its SQLite database and P-256 identity keypair on first launch. Open [http://localhost:8470](http://localhost:8470) to set up your admin account.

### Pair a node

Once the UI is running, pair your first Mac:

1. On the Mac running [CiderStack Fleet](https://ciderstack.com), note the pairing code from the Fleet UI
2. In Harvest, go to **Nodes** and click **Pair Node**
3. Enter the Mac's IP address and pairing code

Harvest will exchange cryptographic keys with the node and begin monitoring it.

## Features

### Job Scheduling

Submit CI/CD jobs that clone a template VM, execute build steps over SSH, and clean up automatically.

```bash
curl -X POST http://localhost:8470/api/jobs \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "build-ios-app",
    "templateVM": "xcode-16-clean",
    "steps": [
      { "name": "install", "command": "cd /src && npm ci", "timeout": 300 },
      { "name": "build",   "command": "cd /src && xcodebuild -scheme App", "timeout": 600 },
      { "name": "test",    "command": "cd /src && xcodebuild test -scheme App", "timeout": 600 }
    ]
  }'
```

Jobs are queued, assigned to the best available node (considering CPU, memory, and chip type), and executed in isolated VMs.

### GitHub Actions Runners

Create pools of ephemeral macOS runners that register with GitHub, pick up exactly one job, and self-destruct:

```bash
curl -X POST http://localhost:8470/api/runner-pools \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "ios-runners",
    "org": "your-github-org",
    "templateVM": "xcode-16-runner",
    "labels": ["macOS", "apple-silicon", "xcode-16"],
    "targetSize": 3
  }'
```

Harvest maintains the pool at `targetSize`, automatically provisioning new runners as jobs consume them. Each runner gets a fresh VM clone — no cross-job contamination.

Use them in your workflows:

```yaml
# .github/workflows/build.yml
jobs:
  build:
    runs-on: [self-hosted, macOS, apple-silicon, xcode-16]
    steps:
      - uses: actions/checkout@v4
      - run: xcodebuild -scheme MyApp -sdk iphoneos
```

### Fleet Management

Full lifecycle control over your macOS VM fleet:

| Capability | Description |
|---|---|
| **Node pairing** | Cryptographic P-256 ECDSA handshake, manager auto-discovery |
| **Live monitoring** | CPU, memory, disk, VM counts — polled and cached |
| **VM operations** | Clone, start, stop, delete, snapshot, restore |
| **Command execution** | Run arbitrary commands on VMs over SSH |
| **Persistent pools** | Keep a set of VMs pre-cloned and ready, with TTL-based recycling |
| **Prometheus metrics** | `/api/metrics` endpoint for Grafana dashboards |

### Alerts & Notifications

- Health alerts when nodes go offline or resources run low
- Slack integration for job status and failure notifications
- In-dashboard alert feed with acknowledgment

## Configuration

Copy `.env.example` to `.env` and customize:

```bash
# Server
PORT=8470                              # HTTP + WebSocket port

# GitHub integration
GITHUB_PAT=ghp_...                     # Personal access token (org admin + runners)
GITHUB_WEBHOOK_SECRET=your-secret       # For push/PR event verification

# Notifications
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
SLACK_CHANNEL=#ci-notifications

# Networking
Harvest_PUBLIC_URL=https://ci.example.com  # URL that VMs use to reach Harvest

# Tuning
HEALTH_CHECK_INTERVAL=30               # Node polling (seconds)
RUNNER_JOB_TIMEOUT=14400                # Max runner job duration (seconds, default 4h)
LOG_LEVEL=info                          # debug | info | warn | error
```

All configuration is optional. Harvest works out of the box for local development with no env vars set.

### Data directory

Harvest stores all state in the `data/` directory relative to the binary:

```
data/
  orchestrator.db          # SQLite database (WAL mode)
  identity/
    identity.json          # Node ID + creation timestamp
    private.key            # P-256 ECDSA private key
  builds/                  # Local build artifacts
```

## Architecture

```
                          +-----------+
                          |   Harvest |
                          |  :8470    |
                          +-----+-----+
                                |
               +----------------+----------------+
               |                |                |
         +-----+-----+   +-----+-----+   +-----+-----+
         |  Mac Mini  |   |  Mac Pro  |   | Mac Studio|
         |  M4 Pro    |   |  M2 Ultra |   |  M4 Max   |
         |  :9473     |   |  :9473    |   |  :9473    |
         +-----+-----+   +-----+-----+   +-----+-----+
               |                |                |
            [VMs]            [VMs]            [VMs]
```

Harvest communicates with each Mac over the **Fleet RPC protocol** (HTTP POST to `:9473/rpc/<Method>`). Each Mac runs [CiderStack Fleet](https://ciderstack.com) which manages the Apple Virtualization framework, VM lifecycle, and SSH tunneling.

**Constraints:**
- Max 2 concurrent VMs per Mac (Apple hardware limitation)
- macOS VMs require Apple Silicon (or Intel with specific macOS versions)
- Chip-aware scheduling routes jobs to the right hardware

## SDK

The TypeScript/JavaScript SDK provides programmatic access:

```bash
npm install @ciderstack/fleet-sdk
```

```typescript
import { FleetClient } from "@ciderstack/fleet-sdk";

const client = new FleetClient({
  host: "192.168.1.100",
  port: 9473,
  apiToken: "your-fleet-token",
});

// List VMs on a node
const vms = await client.listVMs();

// Clone and start a VM
const vm = await client.cloneVM("template-vm-id", "build-1234");
await client.startVM(vm.id);

// Run a command
const result = await client.execCommand(vm.id, "xcodebuild -version");
console.log(result.stdout);

// Clean up
await client.stopVM(vm.id);
await client.deleteVM(vm.id);
```

## API Reference

All endpoints are served at `http://localhost:8470`. Authenticated endpoints require a `Bearer` token or active session cookie.

### Health & Monitoring

| Method | Path | Auth | Description |
|---|---|---|---|
| `GET` | `/api/health` | No | Orchestrator status + uptime |
| `GET` | `/api/metrics` | No | Prometheus-format metrics |
| `GET` | `/api/fleet/overview` | No | Fleet-wide health summary |

### Nodes

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/nodes` | List paired nodes with stats |
| `GET` | `/api/nodes/:id` | Node detail with live stats |
| `POST` | `/api/nodes/pair` | Pair a new node |
| `DELETE` | `/api/nodes/:id` | Unpair node |

### VMs

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/vms` | List all VMs across fleet |
| `POST` | `/api/vms/:nodeId/clone` | Clone a VM |
| `POST` | `/api/vms/:nodeId/:vmId/start` | Start VM |
| `POST` | `/api/vms/:nodeId/:vmId/stop` | Stop VM |
| `DELETE` | `/api/vms/:nodeId/:vmId` | Delete VM |
| `POST` | `/api/vms/:nodeId/:vmId/exec` | Execute command on VM |
| `GET` | `/api/vms/:nodeId/:vmId/snapshots` | List snapshots |
| `POST` | `/api/vms/:nodeId/:vmId/snapshots` | Create snapshot |

### Jobs

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/jobs` | List jobs (filterable by status) |
| `POST` | `/api/jobs` | Submit a job |
| `GET` | `/api/jobs/:id` | Job detail + logs |
| `POST` | `/api/jobs/:id/cancel` | Cancel a queued job |

### Runner Pools

| Method | Path | Description |
|---|---|---|
| `GET` | `/api/runner-pools` | List runner pools |
| `POST` | `/api/runner-pools` | Create ephemeral runner pool |
| `PATCH` | `/api/runner-pools/:id` | Update pool (resize) |
| `DELETE` | `/api/runner-pools/:id` | Delete pool |

### Webhooks

| Method | Path | Description |
|---|---|---|
| `POST` | `/api/webhooks/github` | GitHub push/PR events (HMAC verified) |

## System Requirements

| Platform | Architecture | Status |
|---|---|---|
| macOS 13+ | Apple Silicon (arm64) | Fully supported |
| macOS 13+ | Intel (x64) | Supported |
| Linux | x86_64 | Supported |
| Linux | ARM64 | Supported |

The Harvest binary itself runs on any of the above. **VM operations** require macOS nodes running CiderStack Fleet with Apple Virtualization framework support.

## License

Proprietary. See [LICENSE](LICENSE) for details.

---

<p align="center">
  Built by <a href="https://ciderstack.com">CiderStack</a>
</p>
