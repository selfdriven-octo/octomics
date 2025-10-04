---
layout: default
title: Network - octomics
permalink: /network/
---

# Network

**The blockchain based network powering community octomics.**

Works with the [selfdriven Network Infra Interface](https://www.selfdriven.network/infra-interface/).

An Ouroboros-based drive-chain, with Praos-style leader election, stake-weighted validators.

sweet — here’s a fast, reproducible “single-node local devnet (Conway era)” runbook you can follow today. it uses cardano-testnet to generate a one-node cluster, then layers Ogmios + Kupo on top so your Lucid/relayer code works exactly like preprod/mainnet.

⸻

## Single-node local devnet (Conway era)

- A local Cardano network (fresh chain, 1 BFT node, Conway era).
- cardano-cli in Conway-era mode (CLI moved to “era as top-level command”, e.g. cardano-cli conway ...).  ￼
- Ogmios (WebSocket JSON-RPC bridge) wired to your node.  ￼
- Kupo (fast UTxO indexer) following your node.  ￼

Tip: this is the quickest “all local” path. You can also use prebuilt testnets, but this runbook focuses on your own cluster. The official “custom clusters” page uses cardano-testnet which we’ll rely on below.  ￼

⸻

0) Prereqs
- Git, GHC/Cabal (or Nix), Docker (optional for Ogmios/Kupo).
- Build cardano-node/cardano-cli (or use released binaries).
See “How to run cardano-node”.  ￼
- Build the cardano-testnet helper (bundled in the node repo).  ￼

# clone and build (cabal shown; nix works too)
git clone https://github.com/IntersectMBO/cardano-node.git
cd cardano-node
cabal update
cabal build all
# optional: just build what we need
cabal build cardano-node cardano-cli cardano-testnet  #  [oai_citation:6‡developers.cardano.org](https://developers.cardano.org/docs/get-started/cardano-testnet/?utm_source=chatgpt.com)

Record paths (adjust to your Cabal store):

export CNODE=$(realpath ./dist-newstyle/build/*/*/cardano-node-*/x/cardano-node/build/cardano-node/cardano-node)
export CCLI=$(realpath ./dist-newstyle/build/*/*/cardano-cli-*/x/cardano-cli/build/cardano-cli/cardano-cli)
export CTEST=$(realpath ./dist-newstyle/build/*/*/cardano-testnet-*/x/cardano-testnet/build/cardano-testnet/cardano-testnet)


⸻

1) Spin up a Conway single-node cluster

Create a clean workspace:

mkdir -p $HOME/devnet/conway-one
cd $HOME/devnet/conway-one

Use cardano-testnet to generate a local cluster (1 BFT pool, short slots). The tool writes configs, genesis, keys, and a run script for you. (Exact flags evolve, but the flow is: generate config & genesis → start the node(s).)  ￼

# Example: launch a 1-node cluster (Conway-ready defaults)
$CTEST run \
  --num-pools 1 \
  --testnet-magic 42 \
  --workspace . \
  --start \
  --era conway

What you should see in ./:
- configuration.yaml, byron.genesis.json, shelley.genesis.json, alonzo-genesis.json, conway-genesis.json
- bft1/ (keys, topology, db, socket), logs/
- A run (or run.sh) script that invokes cardano-node run …

If your cardano-testnet subcommand differs, follow its on-screen hints—the developer portal’s “custom clusters” page tracks its usage.  ￼

Sanity check the node socket:

export CARDANO_NODE_SOCKET_PATH=$(pwd)/bft1/node.socket
$CCLI query tip --testnet-magic 42


⸻

2) Use the Conway-era CLI

The CLI shifted to “era as a top-level” command. So for Conway you’ll call:

# examples
$CCLI conway query protocol-parameters --testnet-magic 42
$CCLI conway address key-gen \
  --verification-key-file payment.vkey \
  --signing-key-file payment.skey

This is intentional for Conway; older flags like --babbage-era aren’t used in the new style.  ￼

⸻

3) Fund a dev wallet (from faucet key)

cardano-testnet normally mints an initial UTxO to a prefunded key (printed in the workspace notes or logs). If you need, you can:
- Use the prefunded signing key it generated and send ADA to your payment address you just created, or
- Mint yourself with a simple genesis faucet tx (the tool provides helpers).

Exact file names vary; the cardano-testnet page explains the generated artifacts and paths.  ￼

⸻

4) Add Ogmios (WebSocket bridge)

Option A — Docker (easiest): run the combined cardano-node-ogmios image pointed at your existing node data (mount your config/db/socket). See Ogmios “running with Docker”.  ￼

docker run --rm -p 1337:1337 \
  -v $(pwd)/configuration.yaml:/config/config.yaml \
  -v $(pwd)/bft1/db:/data/db \
  -v $(pwd)/bft1/node.socket:/ipc/node.socket \
  -e OGMIOS_NETWORK="privnet" \
  -e OGMIOS_NODE_SOCKET="/ipc/node.socket" \
  -e OGMIOS_NODE_CONFIG="/config/config.yaml" \
  cardanosolutions/cardano-node-ogmios:latest

Option B — Native: build Ogmios and run it against your node (--node-socket, --node-config). The developer portal gives the basic flags.  ￼

When it’s up, WebSocket should answer on ws://localhost:1337. (Your Lucid Kupmios provider will target this.)

⸻

5) Add Kupo (fast UTxO indexer)

Kupo follows your node via the socket and serves an HTTP API (default :1442). It’s lightweight and perfect for local flows.  ￼

Docker (simple):  ￼

docker run --rm -p 1442:1442 \
  -v $(pwd)/bft1/node.socket:/node.socket \
  -v $(pwd)/configuration.yaml:/config/config.yaml \
  cardanosolutions/kupo:latest \
  --node-socket /node.socket \
  --node-config /config/config.yaml \
  --workdir /var/lib/kupo \
  --since origin \
  --match .*

--since origin indexes from genesis; for faster starts you can point to a recent slot. Kupo docs explain matching patterns / performance knobs.  ￼

Sanity check:

curl http://localhost:1442/health


⸻

6) Wire your app (Lucid / Kupmios)

In your code, point the Kupmios provider to Ogmios+Kupo:

const ogmios = "ws://localhost:1337";
const kupo   = "http://localhost:1442";
// Lucid.new(new Kupmios(ogmios, kupo), "Preprod")  // network tag doesn't matter on private nets

- Ogmios gives chain sync/tx submit; Kupo gives fast UTxO lookups.  ￼

⸻

7) Quick smoke tests
	1.	Query tip:

$CCLI query tip --testnet-magic 42

	2.	Make an address & send a tx (Conway CLI form):

$CCLI conway address key-gen \
  --verification-key-file payment.vkey \
  --signing-key-file payment.skey

$CCLI conway address build \
  --payment-verification-key-file payment.vkey \
  --testnet-magic 42 > payment.addr

# use prefunded key/addr from the workspace to send to payment.addr
# (exact path printed by cardano-testnet)

	3.	Check via Kupo:

curl "http://localhost:1442/v1/utxo/$(cat payment.addr)"


⸻

8) Handy structure & envs

export CARDANO_NODE_SOCKET_PATH=$(pwd)/bft1/node.socket
export OGMIOS_URL=ws://localhost:1337
export KUPO_URL=http://localhost:1442
export TESTNET_MAGIC=42

Workspace tree (typical):

conway-one/
  configuration.yaml
  conway-genesis.json
  ...
  bft1/
    db/               # chain state
    node.socket       # IPC for CLI/Ogmios/Kupo
    topology.json


⸻

9) Troubleshooting (fast)
- CLI complains about era flags → you’re using pre-Conway syntax; prefer cardano-cli conway ....  ￼
- Ogmios can’t connect → verify node.socket path, same user/permissions; check Ogmios docs for Docker bind-mounts.  ￼
- Kupo stays empty → confirm --node-socket and --node-config paths; use --since origin for a brand new chain.  ￼
- No funds → use the prefunded payment key from the generated workspace or regenerate the cluster; the “custom clusters” guide explains the default funds.  ￼

⸻

10) Optional: run everything with Compose

Once you’re happy with the local workspace, wire a compose file that:
- runs your existing node (or mounts its db/socket),
- starts Ogmios (1337) and Kupo (1442),
- sets volumes to your configuration.yaml and node.socket.

The official pages for Ogmios/Kupo show Docker usage & tags so you can pin versions.  ￼

⸻

References
- Custom local clusters with cardano-testnet.  ￼
- Conway-era CLI changes (era as top-level).  ￼
- Ogmios overview / Docker.  ￼
- Kupo manual / Docker.  ￼
- General node run guide.  ￼

⸻

