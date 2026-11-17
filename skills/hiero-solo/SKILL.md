---
name: hiero-solo
license: Apache-2.0
metadata:
  source: https://github.com/hiero-hackers/agentic-support-hiero
  author: Hiero Hackers
description: Deploy and manage local or multi-node Hiero/Hedera test networks with the solo CLI. Use when the user wants to spin up a local Hiero network to develop or test an app, run a Hedera/Hiero SDK or EVM tooling (Hardhat, Foundry, ethers.js, MetaMask) against a throwaway network, set up consensus nodes, mirror node, JSON-RPC relay, explorer, or block node, deploy multi-node test networks, reset or upgrade a local network, or run solo in CI. Also use when the user mentions solo, `solo one-shot`, Falcon values files, hiero-solo-action, or replacing hiero-local-node.
---

# Hiero Solo — Test Network Deployment

`solo` is the Hiero Ledger CLI for deploying and managing standalone Hiero/Hedera test networks on Kubernetes (a local Kind cluster by default). It deploys the full stack — consensus node(s), mirror node, JSON-RPC relay, and explorer, optionally a block node — and is the replacement for the deprecated `hiero-local-node`. See the `hiero-info` skill for ecosystem context.

There are three ways to deploy:

- **One-shot** (`solo one-shot single|multi deploy`) — one command brings up a complete network. The right choice for app developers who just want an endpoint to test against, and for CI.
- **Falcon** (`solo one-shot falcon deploy`) — one-shot driven by a YAML values file. Use it for repeatable, version-controlled custom setups.
- **Manual / step-by-step** — deploy each component individually. Use it to debug a single stage, for custom automation, or multi-cluster setups.

> Solo releases frequently (roughly weekly) and renames commands, flags, and default ports between releases. Confirm syntax for the installed version with `solo --version`, `solo --help`, and `solo <group> <subgroup> --help`, and read the actual ports from the deploy output or `solo deployment config ports`. Prefer these over the values below.

## Prerequisites

| Configuration | Min RAM | Recommended RAM | Min CPU | Free disk |
|---------------|---------|-----------------|---------|-----------|
| Single node | 12 GB | 16 GB | 4–6 cores | 20 GB |
| Multi-node (3+) or block node | 16 GB | 24 GB | 8 cores | 20 GB |

- **Node.js ≥ 22** (lts/jod) — npm installs fail with `EBADENGINE` on Node 20 or earlier.
- **Container runtime** — Docker Desktop (macOS/Windows) or Docker Engine (Linux). On Docker Desktop, raise **Settings → Resources** to ≥ 12 GB / 6 CPUs; Docker caps containers independently of host memory. On macOS, Docker Desktop must already be running. If no Docker Engine is found, solo installs Podman on Linux/macOS/WSL2.
- **kubectl, Helm, Kind** — provisioned by solo into `~/.solo/bin` at deploy time (not on `PATH`; install them separately if you want to call them yourself). Solo enforces Kind ≥ v0.29.0 and Kubernetes ≥ v1.32.2.
- **Platforms** — macOS, Linux, Windows natively in PowerShell, or WSL2.

Insufficient Docker memory/CPU is the most common cause of hanging deploys and unhealthy pods — check it first.

Install:

```bash
npm install -g @hiero-ledger/solo@latest   # recommended, all platforms
solo --version
```

- **Homebrew is deprecated** — `brew install hiero-ledger/tools/solo` stopped receiving updates after August 31, 2026. Migrate with `brew uninstall hiero-ledger/tools/solo`, then install via npm.
- The package was also published as `@hashgraph/solo`. If npm fails with `EEXIST` on the `solo` binary, uninstall the other name first (`npm uninstall -g @hashgraph/solo`).
- Pin a version with `npm install -g @hiero-ledger/solo@<version>`. To change a pinned version, uninstall first, then install the other version.
- A global install pre-pulls the release's container images into `~/.solo/cache/images/` (several GB). Set `SOLO_NO_CACHE=true` to skip that.

## One-shot: the fast path for app developers

```bash
# Single consensus node + mirror node + explorer + JSON-RPC relay
solo one-shot single deploy

# Multiple consensus nodes
solo one-shot multi deploy --num-consensus-nodes 3

# Deployment name, versions, components, and path to accounts.json
solo one-shot show deployment

# Pre-funded test accounts (--output json|yaml|wide)
solo one-shot show accounts

# Tear down
solo one-shot single destroy
solo one-shot multi destroy
```

The first deploy takes 10–20 minutes while images are pulled; later deploys take 3–5 minutes. Long pauses without output are normal, and yellow `Stopping port-forward for port [N]` lines are expected.

Useful one-shot flags (verify with `solo one-shot single deploy --help`):

| Flag | Purpose |
|------|---------|
| `--deployment <name>` / `--namespace <ns>` | Name of the deployment and its Kubernetes namespace (both default to `one-shot`) |
| `--num-consensus-nodes N` | Node count (`multi deploy`) |
| `--deploy-mirror-node=false`, `--deploy-explorer=false`, `--deploy-relay=false` | Skip optional components (all default to `true`) |
| `--minimal-setup` | Consensus node(s) and mirror node only — no relay, no explorer |
| `--external-address 0.0.0.0` | Bind address for the port-forwards (for example, to reach solo on a remote host) |
| `--edge` | Use edge (newer) component versions — see [Pinning component versions](#pinning-component-versions) |
| `--debug` | Debug mode (replaces the deprecated `--dev`) |
| `--quiet-mode` | No interactive prompts (CI) |

Each deploy writes its outputs to `~/.solo/one-shot-<deployment>/` — with the default name that is `~/.solo/one-shot-one-shot/`:

- `accounts.json` — operator and pre-funded test accounts
- `forwards` — active port-forwards
- `notes`, `versions` — deployment summary and component versions

Always destroy a network before deploying a new one. A new `one-shot single deploy` replaces the existing components.

### Pinning component versions

One-shot has no per-component version flags. Pin versions with environment variables instead:

```bash
CONSENSUS_NODE_VERSION=v0.73.0 MIRROR_NODE_VERSION=v0.153.1 solo one-shot single deploy
```

Other variables: `RELAY_VERSION`, `EXPLORER_VERSION`, `BLOCK_NODE_VERSION`, `SOLO_CHART_VERSION`. For release candidates, set the `*_EDGE_VERSION` variants (for example, `CONSENSUS_NODE_EDGE_VERSION=v0.74.0-rc.1`) and pass `--edge`. Tag formats differ: consensus, mirror, and block node tags carry a `v` prefix; relay, explorer, and solo chart tags do not. A wrong prefix is the usual cause of image-pull failures. Use environment variables rather than `--*-version` flags so the image cache pulls the matching images.

## Connecting an app to the network

Read the real ports from the end of the deploy output, `~/.solo/one-shot-<deployment>/forwards`, or `solo deployment config ports --deployment <name>`. When a default port is taken, solo picks the next free one and logs `Using available port <port>`.

| Service | Default (Solo ≥ 0.63) | Solo ≤ 0.62 |
|---------|-----------------------|-------------|
| Consensus node gRPC (node account `0.0.3`) | `localhost:35211` | `localhost:50211` |
| Mirror node REST (ingress) | `http://localhost:38081/api/v1/` | `http://localhost:8081` |
| JSON-RPC relay (chain ID `298`) | `http://localhost:37546` | `http://localhost:7546` |
| Explorer UI | `http://localhost:38080` | `http://localhost:8080` |

**Accounts** — `accounts.json` has two parts:

- `systemAccounts[0]` — the operator `0.0.2` with the well-known ED25519 genesis key in DER format (`302e…`). Fine for local development; never reuse it anywhere else.
- `createdAccounts` — pre-funded test accounts in the groups `ecdsa`, `ecdsa-alias`, and `ed25519`. EVM tools (Hardhat, Foundry, ethers.js, MetaMask) need an **ECDSA alias** account; its `privateKey` is already `0x`-prefixed hex.

Configure the SDK with the operator `accountId` and its `privateKey`, not the `publicKey` — using the public key produces `INVALID_SIGNATURE`.

**SDK local-node presets don't match solo.** `Client.forLocalNode()` (JS), `LocalProvider`, and Go's `ClientForName("localhost")` are hardcoded to `50211` / mirror gRPC `5600`. The Java SDK has no local preset at all — `Client.forName("local-node")` throws. Build the network map explicitly:

```javascript
// Hiero JS SDK
const client = Client.fromConfig({
  network: { "127.0.0.1:35211": AccountId.fromString("0.0.3") },
  mirrorNetwork: "127.0.0.1:38081",
  scheduleNetworkUpdate: false, // otherwise the address-book refresh switches back to 50211
});
client.setOperator(process.env.OPERATOR_ID, process.env.OPERATOR_KEY);
```

```java
// Hiero Java SDK
Client client = Client.forNetwork(Map.of("127.0.0.1:35211", AccountId.fromString("0.0.3")));
client.setMirrorNetwork(List.of("127.0.0.1:38081"));
client.setOperator(AccountId.fromString(operatorId), PrivateKey.fromString(operatorKey));
```

In Go, use `hiero.ClientForNetworkV2` with the same addresses. To run unmodified upstream SDK examples, forward the legacy ports instead:

```bash
kubectl port-forward svc/haproxy-node1-svc -n <deployment> 50211:50211 &
kubectl port-forward svc/mirror-1-grpc     -n <deployment> 5600:5600 &
```

For EVM tooling, point the RPC URL at the relay and set `chainId: 298`. Without the chain ID, Hardhat v3 fails with `HHE40000`. The `hardhat-with-solo` and `foundry-with-solo` examples in the solo repository are working starting points.

Verify the endpoints:

```bash
nc -zv localhost 35211                                   # an IPv6 "refused" line on macOS is harmless
curl http://localhost:38081/api/v1/transactions
curl -X POST http://localhost:37546 -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"eth_chainId","params":[],"id":1}'
```

## Managing a running network

Most commands need the deployment name. It defaults to `one-shot`; run `solo one-shot show deployment` to confirm it.

| Task | Command |
|------|---------|
| Stop / start / restart consensus nodes (ledger state kept) | `solo consensus node stop\|start\|restart --deployment <name>` |
| Reset the ledger to genesis without redeploying | `solo ledger system reset --deployment <name>` |
| Create a funded test account | `solo ledger account create --deployment <name> --hbar-amount 100` |
| Show an account including its key | `solo ledger account info --account-id 0.0.1007 --deployment <name> --private-key` |
| Restore port-forwards (after a reboot or a dropped forward) | `solo deployment port-forwards refresh --deployment <name>` |
| Close all port-forwards | `solo deployment port-forwards stop --deployment <name>` |
| Show the namespace and other deployment info | `solo deployment config info --deployment <name>` |
| Collect logs and diagnostics into `~/.solo/logs/` | `solo deployment diagnostics all --deployment <name>` |
| Upgrade the consensus node version | `solo consensus network upgrade --deployment <name> --upgrade-version <vX.Y.Z>` |

- `solo deployment refresh port-forwards` still works but is deprecated.
- `consensus node stop/start` only affect consensus nodes. Mirror node, relay, and explorer have no stop command — their lifecycle is `add`/`destroy`.
- To pause the **whole** network and keep its state, stop the Kind container (`docker stop solo-cluster-control-plane`). Afterwards run `docker start …` and refresh the port-forwards. Never use `kubectl scale --replicas=0`: deleting pods wipes the consensus node's `emptyDir` state.
- A network upgrade keeps the current fee schedule and throttles. To replace them, pass `--simple-fees-schedules-file` / `--throttles-file` from the target release. Topic messages failing with `FAIL_INVALID` after an upgrade usually means this step was skipped.
- The mirror node keeps accumulating history and resource usage grows over time. For long test sessions, destroy and redeploy.

## Falcon: YAML-driven one-shot

Falcon runs the same pipeline as one-shot. Each top-level YAML section maps to one subcommand, and its keys are that subcommand's long CLI flags, including the leading `--`.

| Section | Subcommand |
|---------|-----------|
| `network` | `consensus network deploy` |
| `setup` | `consensus node setup` |
| `consensusNode` | `consensus node start` |
| `mirrorNode` / `explorerNode` / `relayNode` | `mirror\|explorer\|relay node add` |
| `blockNode` | `block node add` (only if `ONE_SHOT_WITH_BLOCK_NODE=true`) |

```yaml
network:
  --consensus-node-version: "v0.71.0"
setup:
  --consensus-node-version: "v0.71.0"
mirrorNode:
  --enable-ingress: true
  --pinger: true
relayNode:
  --node-aliases: "node1"
```

```bash
solo one-shot falcon prepare                       # interactive wizard → ~/.solo/cache/falcon-values.yaml
solo one-shot falcon deploy --values-file falcon-values.yaml [--deployment <name>] [--num-consensus-nodes 3]
solo one-shot falcon destroy --deployment <name>
```

- `--deployment`, `--namespace`, `--cluster-ref`, and `--num-consensus-nodes` only work on the command line; Falcon ignores them inside the YAML.
- With multiple nodes, `relayNode.--node-aliases` must match the generated aliases (`node1,node2,…`).
- Falcon does not accept `--edge`.
- Block node needs consensus node ≥ v0.72.0, block node ≥ v0.29.0, and ≥ 16 GB of Docker memory.
- Start from the annotated `examples/one-shot-falcon/falcon-values.yaml` in the solo repository.
- To test a locally built consensus node, set `setup.--local-build-path` to `<hiero-consensus-node>/hedera-node/data`.

## Manual deployment

Use step-by-step deployment to inspect intermediate state or debug a single component. It assumes a Kind cluster already exists (for example, one left by a previous one-shot run, or created with `kind create cluster -n ${SOLO_CLUSTER_NAME}`).

```bash
export SOLO_CLUSTER_NAME=solo
export SOLO_NAMESPACE=solo-deployment
export SOLO_CLUSTER_SETUP_NAMESPACE=solo-cluster
export SOLO_DEPLOYMENT=solo-deployment
export CONSENSUS_NODE_VERSION=v0.78.1   # pick a current hiero-consensus-node release tag

solo cluster-ref config connect --cluster-ref kind-${SOLO_CLUSTER_NAME} --context kind-${SOLO_CLUSTER_NAME}
solo deployment config create -n "${SOLO_NAMESPACE}" --deployment "${SOLO_DEPLOYMENT}"
solo deployment cluster attach --deployment "${SOLO_DEPLOYMENT}" --cluster-ref kind-${SOLO_CLUSTER_NAME} --num-consensus-nodes 1
solo keys consensus generate --gossip-keys --tls-keys --deployment "${SOLO_DEPLOYMENT}"
solo cluster-ref config setup --cluster-setup-namespace "${SOLO_CLUSTER_SETUP_NAMESPACE}"
solo consensus network deploy --deployment "${SOLO_DEPLOYMENT}"
solo consensus node setup --deployment "${SOLO_DEPLOYMENT}" --consensus-node-version "${CONSENSUS_NODE_VERSION}"
solo consensus node start --deployment "${SOLO_DEPLOYMENT}"
solo mirror node add   --deployment "${SOLO_DEPLOYMENT}" --cluster-ref kind-${SOLO_CLUSTER_NAME} --enable-ingress --pinger --force-port-forward
solo explorer node add --deployment "${SOLO_DEPLOYMENT}" --cluster-ref kind-${SOLO_CLUSTER_NAME} --force-port-forward
solo relay node add    --deployment "${SOLO_DEPLOYMENT}" --node-aliases node1
```

- `--consensus-node-version` replaced `--release-tag`, which has been deprecated since Solo 0.85.
- `--pinger` keeps the mirror importer fed with record files.
- Pass `--application-properties <file>` to `consensus network deploy` to merge custom node properties. Add `# SOLO_ENABLE_OVERWRITE=true` to the file to replace the defaults instead.
- Tear down in reverse: `relay node destroy` → `explorer node destroy --force` → `mirror node destroy --force` → `consensus network destroy --force`. Skipping the order leaves orphaned Helm releases and PVCs.
- Scripts from before Solo v0.44 use flat names (`network deploy`, `node setup`, `mirror-node deploy`, `account create`); see the CLI migration reference in the docs.
- `solo init` no longer exists.

### Command groups

| Group | Subcommands |
|-------|-------------|
| `one-shot` | `single`/`multi` `deploy\|destroy`, `falcon` `prepare\|deploy\|destroy`, `show` `deployment\|accounts` |
| `cluster-ref config` | `connect`, `disconnect`, `list`, `info`, `setup`, `reset` |
| `deployment` | `config` (`create`, `delete`, `list`, `info`, `ports`), `cluster attach`, `port-forwards` (`refresh`, `stop`), `diagnostics` (`all`, `logs`, `debug`) |
| `keys consensus` | `generate` |
| `consensus network` | `deploy`, `destroy`, `freeze`, `upgrade` |
| `consensus node` | `setup`, `start`, `stop`, `restart`, `add`, `update`, `destroy`, `refresh`, `collect-jfr` |
| `consensus dev-node-*`, `consensus state` | Staged node add/update/delete transactions, state download |
| `ledger` | `system init\|reset`, `account create\|update\|info\|predefined`, `file create\|update` |
| `mirror node` / `relay node` / `explorer node` / `block node` | `add`, `destroy`, `upgrade` |
| `cache image` / `cache chart` | `pull`, `load`, `list`, `status`, `clear`, `prune` |
| `rapid-fire` | Load generation against a running network |

## Cleanup

- `solo one-shot single destroy` (or `multi` / `falcon destroy`) removes all Helm releases and the deployment config, but **keeps the Kind cluster** so the next deploy is faster. The command is idempotent; if it fails part-way, run it again.
- For a clean slate, also delete the cluster: `kind delete cluster --name solo-cluster`.
- Full reset — last resort, only when state is corrupted:

```bash
kind get clusters | grep '^solo' | while read c; do kind delete cluster -n "$c"; done   # keep the ^solo filter!
rm -rf ~/.solo   # also deletes the image cache and logs
```

## CI/CD

`solo one-shot single deploy` works in CI the same way as locally. On GitHub-hosted `ubuntu-latest` runners it creates the Kind cluster itself.

```yaml
- uses: actions/setup-node@v4
  with: { node-version: '22' }
- run: npm install -g @hiero-ledger/solo@<pinned-version>   # never @latest in CI
- run: solo one-shot single deploy
- name: Wait for mirror REST
  run: for i in $(seq 1 30); do curl -sf http://localhost:38081/api/v1/accounts | grep -q '"accounts"' && exit 0; sleep 10; done; exit 1
# … run tests …
- if: always()
  run: solo one-shot single destroy --quiet-mode || true
- if: always()
  uses: actions/upload-artifact@v4
  with: { name: solo-logs, path: ~/.solo/logs/* }
```

- Self-hosted runners need ≥ 6 CPUs and 12 GB of memory, plus Kind ≥ v0.29.0 and kubectl ≥ v1.32.2 (for example, via `helm/kind-action`).
- The `hiero-solo-action` GitHub Action wraps the same flow.
- The solo repository's `examples/` (for example, `one-shot-falcon`, `one-shot-local-build`, `hardhat-with-solo`, `state-save-and-restore`) are `Taskfile`-driven references.

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Deploy hangs, pods `Pending` / `CrashLoopBackOff` / unhealthy | Increase Docker memory and CPU, then destroy and redeploy |
| First deploy fails with a PostgreSQL startup timeout | Known intermittent issue: `solo one-shot single destroy` and deploy again |
| `NodesStarted` timeout or SDK `TimeoutException` although the pod is `Running` | Destroy and redeploy; the second attempt usually succeeds |
| Connection refused on a previously working port | `solo deployment port-forwards refresh --deployment <name>`, or check whether solo moved to another port |
| Apple Silicon: "mounts denied" / "path is not shared from the host" | Add `/opt/homebrew` under Docker Desktop **Settings → Resources → File Sharing** |
| MinIO tenant `ImagePullBackOff` | See the troubleshooting page; skip MinIO via environment variables or override the tenant image |
| `failed to setup chart repositories … already exists` | `helm repo list`, then `helm repo remove <stale-repo>` |
| `EEXIST` when running `npm install -g` | Uninstall the other package name (`@hashgraph/solo` / `@hiero-ledger/solo`) first |
| JSON-RPC relay out of memory under load | Raise the relay memory (see troubleshooting → JSON-RPC Relay Out of Memory) |

For bug reports, attach the `~/.solo/logs/` contents from `solo deployment diagnostics all`. The structured log is `~/.solo/logs/solo.ndjson`.

## References

| Resource | URL |
|----------|-----|
| Solo docs | https://solo.hiero.org/docs/ |
| Service endpoints | https://solo.hiero.org/docs/using-solo/endpoints/ |
| Troubleshooting | https://solo.hiero.org/docs/troubleshooting/ |
| Solo repository / releases | https://github.com/hiero-ledger/solo |
| Solo examples | https://github.com/hiero-ledger/solo/tree/main/examples |
| Solo docs source | https://github.com/hiero-ledger/solo-docs |
| hiero-solo-action | https://github.com/hiero-ledger/hiero-solo-action |
