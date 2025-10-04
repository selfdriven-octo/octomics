---
layout: default
title: Network - octomics
permalink: /network/
---

# Network

**The blockchain based network powering community octomics.**

Works with the [selfdriven Network Infra Interface](https://www.selfdriven.network/infra-interface/).

An Ouroboros-based drive-chain, with Praos-style leader election, stake-weighted validators.

Sections:
- (A) Architecture
- (B) Ceremonies & Params
- (C) Contracts
- (D) Relayers
- (E) Docker-compose stack
- (F) Runbook
- (G) Security

*All examples assume Cardano Conway era.*

⸻

### A) Reference architecture (Ouroboros partner chain)

Two sovereign Cardano-family networks, each running Ouroboros - the Cardano mainnet & the Octomics network:
- Cardano mainnet: where ADA/tokens are locked in a Plutus bridge “Lockbox”.
- Octomics Network "OctoChain" (Ouroboros-Praos): where wrapped assets (ADA, Octos) are minted/burned.
- Relayers (≥N, threshold K): watch both chains, build canonical BridgeEvent objects, co-sign them, and submit proofs.
- Attestation keys: separate from staking/VRF/KES keys; rotated via on-chain governance.

Core dataflow:
	1.	Lock (mainnet) → Lockbox outputs a new LockedUTxO (datum includes amount, target chain, recipient).
	2.	Relayers see the Lock, form EventID, threshold-sign Attestation.
	3.	OctoChain Bridge verifies K-of-N signatures, mints sADA to the recipient.
	4.	Reverse direction: Burn (OctoChain) → attested → Unlock (Cardano Mainnet).

⸻

### B) Ceremonies & chain parameters (Ouroboros-Praos)

B1. OctoChain genesis (Conway)

Pick sane defaults for a small, permissioned validator set:
- Security: Praos with f=0.2, activeSlotsCoeff ~ 0.2, slotLength = 1s, epochLength = 43200 (≈12h), k=2160.
- KES: KES period ~ 1d; automate rotation.
- Initial funds: bootstrap treasury + faucet.
- Decentralization: start with fixed validator set (N=4–7 pools), move to delegation later.

Create genesis (example):

cardano-cli conway genesis create \
  --testnet-magic 7777 \
  --gen-genesis-keys 1 \
  --gen-utxo-keys 4 \
  --start-time now \
  --slot-length 1 \
  --active-slots-coeff 0.2 \
  --security-param 2160

Generate pool keys for each validator:

# For each validator i:
cardano-cli node key-gen-VRF --verification-key-file vrf.vkey --signing-key-file vrf.skey
cardano-cli node key-gen-KES --verification-key-file kes.vkey --signing-key-file kes.skey
cardano-cli node key-gen --cold-verification-key-file cold.vkey --cold-signing-key-file cold.skey --operational-certificate-issue-counter cold.counter
cardano-cli node issue-op-cert --kes-verification-key-file kes.vkey --cold-signing-key-file cold.skey --operational-certificate-issue-counter cold.counter --kes-period 0 --out-file node.opcert

Topology
- 1× BP per validator; 1–2× relays per validator.
- BP peers only with its relays; relays peer with other relays; publish relay addresses in topology.json.

Time sync: enable NTP on every host. Keep KesPeriod rotation on a cron with monitoring.

B2. Attestation key ceremony (for relayers)
- Generate Ed25519 attestation keys offline for each relayer.
- Publish Attesters.vkey[] on both chains (mainnet Lockbox datum; OctoChain Bridge state).
- Store minThreshold K alongside the list (K-of-N).
- Version keys via a governance “param cell” so you can rotate without redeploys.

⸻

### C) Bridge contracts (high-level designs)

You’ll implement two Plutus state machines (Aiken/Plutus V2 are both fine). Below are compact schemas you can translate to Aiken or Plutus.

C1. On Mainnet: Lockbox validator (Plutus state machine)

Purpose: take real ADA/tokens into escrow; emit a canonical, hashable event.

Datum (at each LockedUTxO):

type LockDatum = {
  event_id: ByteString,        -- H(blake2b-256): chain_id||tx_id||ix||recipient||amount||nonce
  origin_chain: ByteString,    -- "cardano-mainnet"
  target_chain: ByteString,    -- "octochain-8888"
  recipient_sc: ByteString,    -- bytes of  bech32/addr
  asset: AssetClass,           -- ADA or policy+assetname
  amount: Integer,
  attesters_hash: ByteString,  -- H(concat(attester_pubkeys || threshold K))
  nonce: Integer
}

Redeemers:
- Lock: creates a new state UTxO with amount locked.
- Unlock: spends a previously Burned attested event from octochain; checks Attestation.

Attestation structure (Carried in Unlock redeemer):

type Attestation = {
  event_id: ByteString,
  source_chain: ByteString,   -- "octochain-8888"
  signatures: [Signature],    -- ed25519 over (domain || event_id || direction)
  bitmap: ByteString,         -- which relayers signed
  threshold: Integer
}

Validation rules (Unlock):
- hash(attesters_pubkeys, threshold) equals attesters_hash in current state.
- Verify K valid signatures over the domain-separated message.
- Confirm the referenced event_id corresponds to a Burn event on OctoChain (the event content is embedded or reconstructed deterministically).
- Recreate the exact LockedUTxO value and release it to the recipient mainnet address.

C2. On OctoChain: Bridge & MintPolicy

Two parts:
1.	Bridge validator (mirrors Lockbox) maintains the attester set & thresholds, processes incoming Lock attestations from mainnet, and releases/mints sAssets.
2.	Minting policy for ADA/Octos:
  - Mint only when called by the Bridge validator with a valid Lock Attestation from mainnet.
  - Burn only when called by Bridge with a Burn redeemer (which emits an octochain BurnEvent to later unlock on mainnet).

OctoChain Lock → Mint flow:
- Input: Attestation{ event_id from mainnet Lock }.
- Checks K-of-N attesters.
- Mints sADA (policy id policy_ADA) to recipient_sc.

OctoChain Burn → Event flow:
- User spends their ADA with Burn redeemer.
- Bridge ensures exact burn amount and emits BurnEvent with a new event_id.
- Relayers threshold-sign and later Unlock on mainent.

⸻

### D) Relayer cluster (threshold attestation)
- At least N relayer nodes; require K signatures.
- Each relayer is stateless (everything derivable from chain); private key only for attestations.
- Consensus between relayers is by deterministic event ordering (EventID), not leader election.

EventID (stable hashing)

EventID = blake2b-256(
  network_id || direction || tx_id || tx_index || asset_policy || asset_name ||
  amount || sender || recipient || epoch_no || nonce
)

Domain-separate Lock vs Burn.

Node.js skeleton (watch–attest–submit; .then() style)

import { blake2b } from 'blakejs';
import { createHash } from 'crypto';
import { CardanoClient, PartnerClient, submitTx, getEventsSince } from './sdk.js';
import { sign, verifyMany, toBitmap } from './threshold.js';

const ATTEST_DOMAIN = Buffer.from('partner-bridge-v1');

function eventId(e) {
  const enc = Buffer.concat([
    Buffer.from(e.networkId), Buffer.from(e.direction),
    Buffer.from(e.txId, 'hex'), Buffer.from(Uint8Array.of(e.txIndex)),
    Buffer.from(e.assetPolicy, 'hex'), Buffer.from(e.assetName),
    Buffer.from(e.amount.toString()), Buffer.from(e.sender), Buffer.from(e.recipient),
    Buffer.from(e.epoch.toString()), Buffer.from(e.nonce.toString()),
  ]);
  return Buffer.from(blake2b(enc, null, 32)).toString('hex');
}

function msgToSign(event_id) {
  return createHash('blake2b512').update(Buffer.concat([ATTEST_DOMAIN, Buffer.from(event_id,'hex')])).digest();
}

const state = { lastPointmainnet: null, lastPointoctochain: null };

Promise.resolve()
  .then(() => Promise.all([CardanoClient.connect(), PartnerClient.connect()]))
  .then(([mainnet, octochain]) => {
    function loop() {
      return getEventsSince(mainnet, 'LOCK', state.lastPointmainnet)
        .then(lockEvents => Promise.all(lockEvents.map(ev => {
          const id = eventId(ev);
          const m = msgToSign(id);
          return sign(process.env.RELAYER_SK, m).then(sig => ({ id, sig, ev }));
        })))
        .then(partialSigs => submitTx(octochain, 'MINT', partialSigs)) // octochain bridge collects ≥K
        .then(() => getEventsSince(octochain, 'BURN', state.lastPointoctochain))
        .then(burnEvents => Promise.all(burnEvents.map(ev => {
          const id = eventId(ev);
          const m = msgToSign(id);
          return sign(process.env.RELAYER_SK, m).then(sig => ({ id, sig, ev }));
        })))
        .then(partialSigs => submitTx(mainnet, 'UNLOCK', partialSigs))
        .then(() => setTimeout(loop, 800)) // simple poll; replace with websockets in prod
        .catch(err => { console.error(err); setTimeout(loop, 1500); });
    }
    return loop();
  });

In production, move to WebSocket subscriptions (Ogmios on both chains), add replay protection (dedup store), and use aggregate signature (BLS) or canonical multi-sig with explicit bitmap.

⸻

### E) docker-compose.yml (local dual-network + relayer + tooling)

Below is a single-host lab that brings up:
- mainnet-node (Cardano devnet) + Ogmios + Kupo
- octonet-node (OctoChain, Ouroboros-Praos) + Ogmios + Kupo
- relayer (Node.js)
- Optional: db-sync pairs if you need SQL (commented)

You’ll need to drop your generated configs/keys into ./mainnet/ and ./octochain/ as noted.

version: "3.9"
services:
  mainnet-node:
    image: inputoutput/cardano-node:8.11.0
    command: [
      "run",
      "--topology","/config/topology.json",
      "--database-path","/data/db",
      "--socket-path","/ipc/node.socket",
      "--host-addr","0.0.0.0",
      "--port","3001",
      "--config","/config/config.json"
    ]
    volumes:
      - ./mainnet/config:/config:ro
      - ./mainnet/data:/data
      - ./mainnet/ipc:/ipc
    ports: ["3001:3001"]

  mainnet-ogmios:
    image: cardanosolutions/ogmios:v6.5.0
    environment:
      - OGMIOS_NODE=/ipc/node.socket
    volumes:
      - ./mainnet/ipc:/ipc
    ports: ["1337:1337"]
    depends_on: [mainnet-node]

  mainnet-kupo:
    image: cardanosolutions/kupo:v2.8.0
    command: [
      "--node-socket","/ipc/node.socket",
      "--since","origin",
      "--match","*",
      "--workdir","/kupo"
    ]
    volumes:
      - ./mainnet/ipc:/ipc
      - ./mainnet/kupo:/kupo
    ports: ["1442:1442"]
    depends_on: [mainnet-node]

  octonet-node:
    image: inputoutput/cardano-node:8.11.0
    command: [
      "run",
      "--topology","/config/topology.json",
      "--database-path","/data/db",
      "--socket-path","/ipc/node.socket",
      "--host-addr","0.0.0.0",
      "--port","4001",
      "--config","/config/config.json"
    ]
    volumes:
      - ./octochain/config:/config:ro        # your PartnerChain genesis/config
      - ./octochain/data:/data
      - ./octochain/ipc:/ipc
    ports: ["4001:4001"]

  octonet-ogmios:
    image: cardanosolutions/ogmios:v6.5.0
    environment:
      - OGMIOS_NODE=/ipc/node.socket
    volumes:
      - ./octochain/ipc:/ipc
    ports: ["2337:2337"]
    depends_on: [octochain-node]

  octonet-kupo:
    image: cardanosolutions/kupo:v2.8.0
    command: [
      "--node-socket","/ipc/node.socket",
      "--since","origin",
      "--match","*",
      "--workdir","/kupo"
    ]
    volumes:
      - ./octochain/ipc:/ipc
      - ./octochain/kupo:/kupo
    ports: ["2442:2442"]
    depends_on: [octochain-node]

  relayer:
    build: ./relayer
    environment:
      - mainnet_OGMIOS=ws://mainnet-ogmios:1337
      - octochain_OGMIOS=ws://octochain-ogmios:2337
      - RELAYER_SK=env:RELAYER_SK_HEX
      - ATTESTERS_JSON=/config/attesters.json
      - THRESHOLD_K=2
    volumes:
      - ./relayer/config:/config:ro
    depends_on: [mainnet-ogmios, octochain-ogmios]

Folder layout you should create:

octochain/
  docker-compose.yml
  mainnet/
    config/{config.json, topology.json, genesis.json, alonzo-genesis.json, conway-genesis.json}
    ipc/  data/  kupo/
  octonet/
    config/{config.json, topology.json, genesis.json, alonzo-genesis.json, conway-genesis.json}
    ipc/  data/  kupo/
  relayer/
    Dockerfile
    package.json
    src/{index.js, sdk.js, threshold.js}
    config/attesters.json

Minimal relayer/Dockerfile:

FROM node:20-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev
COPY src ./src
COPY config ./config
CMD ["node","src/index.js"]

relayer/src/threshold.js (sketch):

import { createSign, createVerify } from 'crypto';

export function sign(skHex, msg) {
  // use libsodium or noble-ed25519 in practice
  return Promise.resolve(fakeSign(skHex, msg)); // placeholder
}
export function verifyMany(pubkeys, msg, sigs, bitmap, k) {
  // verify sigs against pubkeys set by bitmap, count ≥k
  return true;
}
export function toBitmap(indexes, n) { /*...*/ }


⸻

### F) Runbook (local → testnet → mainnet)

F1. Local
	1.	Generate both networks:
- mainnet: either use preprod configs or make a devnet with magic 42.
- octochain: create OctoChain genesis (magic 8888) with your Praos params.
	2.	Start compose: docker compose up -d.
	3.	Deploy contracts:
- mainnet: publish Lockbox with initial attesters_hash, threshold K.
- octonet: publish Bridge validator + sADA policy bound to Bridge.
	4.	Fund faucet on both networks (UTxO keys created during genesis).
	5.	Test flows:
- Lock → Mint: mainnet lock 100 ADA to octonet recipient; confirm sADA=100 on octochain.
- Burn → Unlock: octochain burn 30 sADA; confirm 30 ADA released on mainnet.

CLI sketches (Lock on mainnet):

cardano-cli address key-gen --verification-key-file user.vkey --signing-key-file user.skey
RECIP_SC="addr_test1q..."   # octochain recipient (encoded bytes in datum)
AMT=100000000               # 100 ADA
# Build datum JSON with recipient_sc, asset, amount, nonce...
cardano-cli transaction build \
  --tx-in <tx#ix> \
  --tx-out "$(LOCKBOX_ADDR)+$AMT" \
  --tx-out-datum-json-file lock-datum.json \
  --change-address $(cat user.addr) \
  --testnet-magic 42 \
  --out-file tx.body

cardano-cli transaction sign --tx-body-file tx.body --signing-key-file user.skey --out-file tx.signed
cardano-cli transaction submit --tx-file tx.signed --testnet-magic 42

Relayer logs should show the new Lock event, signatures, and an octochain mint submission.

F2. Testnet (preprod/preview)
- Re-deploy Lockbox & Bridge with identical Attester set and threshold.
- Run ≥5 relayers across 2+ clouds/AZs.
- Stand up 4–7 validators for PartnerChain with distinct stake pools.
- Add Ogmios+Kupo for public read APIs.
- Add Prometheus/Grafana & alerting (KES expiry, empty relay queue, attestation lag).

F3. Mainnet
- Key ceremony for production attesters (HSM/YubiHSM or Hashicorp Vault).
- Add governance script to rotate attester set (datum update via DAO or multisig).
- Audits (contracts + relayer + node hardening).

⸻

### G) Security, ops, and hardening checklist
- Replay & uniqueness
- Every event has a unique EventID; keep a Nonce and maintain a spent EventID set in validator state to prevent re-use.
- Threshold control
- Store attesters_pubkeys + K on-chain (datum), not in off-chain config.
- Expose a controlled update path (governance) to change keys/K.
- Signature domain separation
- Prefix messages with constant domain bytes ("partner-bridge-v1") and direction (LOCK→MINT / BURN→UNLOCK).
- Liveness
- ≥N relayers, geo-distributed; exponential backoff + queueing.
- Watchdog: “no attested events in X mins” = page.
- Integrity
- Double-read events via two indexers (Ogmios + Kupo/db-sync).
- KES/VRF
- Rotate KES automatically; monitor remainingSlots.
- Keep cold keys offline; use air-gapped ceremony for op-certs.
- Rate limits
- Bridge contract enforces per-tx caps until liquidity & monitoring mature.
- Upgrades
- Version every datum (e.g., version: 1) to migrate safely.
- Keep an escape hatch: emergency pause (requires higher quorum).

⸻

Next:
	1.	Aggregate signatures (BLS) to shrink on-chain attestation size and fees.
	2.	Optimistic fraud-proofs: allow anyone to challenge an invalid attestation within a window.
	3.	ZK light client (trust-minimized):
- Prove PartnerChain block header validity (Praos rules) with a zk circuit.
- Verify succinct proof in a Cardano Plutus verifier (or via a specialized on-chain verifier once available).

⸻

- [Network Kit - Plutus](/network/kit/plutus)

⸻

## OctoChain Lab Kit V3

### 1.	Aggregate signatures (BLS)

- aggregator/ (Rust) uses blst to aggregate partial signatures → a single G2 aggregate. It returns:
- agg_sig_g2 (hex-compressed)
- agg_hash (blake3 transcript over ordered partials)
- Intent: keep the on-chain payload tiny (one aggregate) and pair it with either:
- the optimistic flow (no on-chain pairing checks — see #2), or
- a future SNARK proof that the aggregate is valid (see #3).

### 2.	Optimistic fraud-proof bridge

- Aiken module: aiken_optimistic/validators/optimistic_bridge.ak
- Pattern: Claim → Challenge → Finalize with a time window.
- Claim creates an escrow claim UTxO holding:
- event_id, msg = domain || event_id
- agg_sig (BLS aggregate, opaque to chain)
- attesters_root, threshold, window_slots, created_at_slot, challenged=false
- Challenge (anyone): before deadline, flip challenged (you’ll add bonds/slashing).
- Finalize: after deadline and if not challenged, allow the spend that performs Unlock/Mint.
- How to use:
- Gate your existing Lockbox/Bridge spend by requiring a reference input to a finalized claim UTxO for the same event_id.
- Add a challenge bond output in Challenge tx; pay out bond if valid, slash if frivolous.

### 3.	ZK light client (Praos) — Noir skeleton

- circuits/praos-light/ is a Noir project stub you can extend to:
- Verify a batch of PartnerChain headers meet Praos rules (VRF, hash links).
- Produce a succinct proof with public outputs:
- headers_root (commitment),
- agg_sig_commitment (committee/attester commitment),
- epoch/chain ids.
- Intended endgame:
- Verify this SNARK on Cardano via a Plutus SNARK verifier or specialized on-chain verifier when available.
- This makes BLS aggregation “trust-minimized”: chain checks only the SNARK, not pairings.

⸻

###  How the pieces fit
- Fast path (today):
Off-chain relayers create a BLS aggregate and post a Claim. If nobody challenges before the deadline, Finalize releases funds. Small on-chain footprint, cheap gas.
- Trust-minimized path (future):
Off-chain creates BLS aggregate and a SNARK proving the aggregate is valid for the posted headers_root / committee. Cardano verifies the SNARK; no challenge period needed.

⸻

Where to wire in your existing v2 kit
- Keep v2 Lockbox / Bridge / MintPolicy for business logic.
- Add the optimistic claim as a gating reference:
- mainnet Unlock path requires a finalized claim referencing the Burn on octonet.
- octonet Mint path requires a finalized claim referencing the Lock on mainnet.

⸻

Next:
- Add challenge bonds & slashing flows (escrow, payouts).
- Implement real time window using slot/time conversions and reference inputs.
- Swap the placeholder BLS blobs for actual aggregate calls to aggregator/ (expose a small HTTP or CLI).
- Upgrade relayer to:
- gather partials → call aggregator → include agg_sig_g2,
- post Claim, poll window, and post Finalize.
- Flesh out Noir:
- VRF gadget & header hashing,
- batch rules for Praos leader election,
- produce verifier inputs for a Plutus SNARK verifier interface.

⸻

## OctoChain Lab Kit V4

### 1. Challenge-bond mechanics (Aiken scaffold)
- New module: aiken_optimistic_bonds/validators/optimistic_bridge_bonds.ak
- Flow:
- Claim{claimer_pkh} — requires a claimer bond output in the same tx.
- Challenge{challenger_pkh, reason} — requires a challenger bond output and flips challenged.
- Finalize — after window, if not challenged, requires refund to claimer_pkh.
- It’s a scaffold: the helper has_pkh_output(...) just checks for any output to the PKH. Swap it for strict checks (exact lovelace, token accounting, reference inputs, script hash guards, time window via slot/time conversions).
	2.	BLS Aggregator CLI (Rust)
- Folder: aggregator/
- Aggregates partial signatures (G2) against PKs (G1) using blst.
- Usage:

cd aggregator
cargo run --release -- --input partials.json
# or
cat partials.json | cargo run --release --

- Input partials.json:

[
  { "pk_g1": "hex...", "sig_g2": "hex..." },
  { "pk_g1": "hex...", "sig_g2": "hex..." }
]

- Output:

{ "agg_sig_g2": "hex...", "agg_hash": "hex..." }

### How to wire this into your bridge
- On Lock → Mint (mainnet→octonet):
	1.	Relayers collect partials → run aggregator → produce agg_sig_g2.
	2.	Post Claim on octonet (bridge consumes it via reference input later).
	3.	If no Challenge before deadline → Finalize; mint sADA + refund claimer bond.
- On Burn → Unlock (octonet→mainnet):
	1.	Same aggregator step on the burn event.
	2.	Post Claim on mainnet; if unchallenged after window → Finalize; unlock ADA + refund claimer bond.

### Next (hardening steps):
- Replace placeholder bond checks with:
- exact lovelace equality,
- correct recipients (claimer/challenger PKH),
- optional fee split to treasury.
- Add resolution path if challenged:
- Either require a SNARK proof (trust-minimized route), or
- Require an on-chain verifiable refutation (e.g., mismatched amounts/recipient), paying out the challenger’s bond and slashing claimer’s.
- Hook the v3 Noir Praos light client once you’re ready to go SNARK-first (no challenge window needed).
- In the relayer, call the Rust aggregator (small HTTP or CLI), embed agg_sig_g2 in the Claim, and schedule Finalize after window_slots.

⸻

## OctoChain Lab Kit V5

Tightened the Aiken validator and wired the relayer to the Rust aggregator with an auto-finalize loop.
- Download v5 Kit (strict bonds + time window + Node wrapper)
- For reference, earlier drops:
  - v4 (bonds + aggregator CLI)
  - v3 (optimistic + BLS + ZK skeleton)
  - v2 (ed25519 threshold + relayer upgrades)

### What v5 adds
- Aiken (strict optimistic with bonds + time windows)
- Exact lovelace checks for claimer/challenger bonds.
- Enforces Claim lower bound ≥ created_at_time_ms, Challenge upper bound ≤ deadline, Finalize lower bound ≥ deadline and challenged == False.
- File: aiken_optimistic_bonds_strict/validators/optimistic_bridge_bonds_strict.ak.
- Relayer Node wrapper
- Calls the Rust BLS aggregator (AGG_BIN) to produce a single agg_sig_g2.
- Posts Claim with strict bond values and auto-schedules Finalize after WINDOW_MS.
- Files: relayer/src/index.js, relayer/src/sdk.js, relayer/src/ogmios.js, relayer/src/util.js.

Next bits I can do if you want
- Replace placeholder PKHs and integrate CSL transaction construction for Claim/Challenge/Finalize.
- Add resolution path that pays challenger and slashes claimer on successful disputes.
- Flesh out the Noir Praos light client (VRF gadget, header hash chain) and sketch a Plutus verifier interface.

If you want CSL wiring next, tell me your preferred stack (pure CSL via @dcspark/cardano-multiplatform-lib, Lucid, Mesh) and I’ll bake it into the relayer with real submit calls. ￼

⸻

## OctoChain Lab Kit V6

### What’s in v6
- relayer/src/lucid.js
- Creates a Lucid instance for L1/L2 via Kupmios(ogmios, kupo) with network from env.
- Encodes the ClaimDatum schema so we can inline it at the optimistic-bridge script.
- relayer/src/sdk-lucid.js
- submitClaimTx(chain, direction, event, claim, bonds): builds & submits a tx that:
- sets validFrom(created_at_time_ms),
- pays to the optimistic script address with inline datum (your claim),
- adds a claimer bond output (per our scaffold validator).
- submitFinalizeTx(chain, direction, claimRef): spends a claim UTxO with a Finalize redeemer after the window (placeholder selection logic; wire Kupo filter by event_id).
- relayer/src/index-lucid.js
- Relayer loop that:
- calls the Rust BLS aggregator (AGG_BIN) to aggregate partials,
- posts Claim via Lucid,
- auto-schedules Finalize after WINDOW_MS.

Env you’ll want to set

# L1 (Cardano) and L2 (PartnerChain)
L1_OGMIOS_URL=ws://localhost:1337
L1_KUPO_URL=http://localhost:1442
L1_NETWORK=0
L1_PAYMENT_SK_HEX=<>   # hex or bech32-priv acceptable by Lucid

L2_OGMIOS_URL=ws://localhost:2337
L2_KUPO_URL=http://localhost:2442
L2_NETWORK=0
L2_PAYMENT_SK_HEX=<>

# Optimistic bridge script addresses (L1/L2)
CLAIM_SCRIPT_ADDRESS_L1=addr_test1...
CLAIM_SCRIPT_ADDRESS_L2=addr_test1...

# Relayer settings
DOMAIN_HEX=706172746e65722d6272696467652d7631   # "partner-bridge-v1"
WINDOW_MS=1800000
CLAIMER_BOND=2000000
CHALLENGER_BOND=2000000
AGG_BIN=./aggregator/target/release/partnerchain-aggregator
CLAIMER_PKH_HEX=<payer keyhash hex>

### Next improvements:
- The bond model in the strict Aiken scaffold still “pays to claimer PKH” to satisfy checks. For real security, change it to script-locked bond escrow, and update submitClaimTx to lock the bond at that script.
- In submitFinalizeTx, replace the naive utxosAt() selection with a Kupo query that filters for the claim UTxO by event_id encoded in datum.
- Add real Finalize redeemer serialization matching your Aiken constructors.
- If you’re ready, I can:
- add the bond-escrow script and wire the Lucid builders to lock/release bonds correctly,
- add Kupo REST queries (by datum JSON path) to find the exact claim,
- or translate the current JS to TypeScript with strict types for all datum/redeemer schemas.

⸻

## OctoChain Lab Kit V7

Done. I switched bonds to a script-escrow and wired Lucid to lock/refund them, plus added a Kupo filter scaffold.
- Download v7 Kit (bond escrow + Lucid wiring + Kupo filter)
- For context, previous drops:
- v6 (Lucid wiring)
- v5 (strict bonds + auto finalize)
- v4 (bonds + aggregator CLI)
- v3 (optimistic + BLS + ZK skeleton)
- v2 (ed25519 threshold + relayer upgrades)

What changed in v7
- Aiken bond-escrow validator (aiken_bond_escrow/validators/escrow.ak)
- Two redeemers: Finalize (refund claimer) and ChallengeWin (pay challenger).
- Checks exact lovelace value and recipient PKH. In production, bind spend to a finalized-claim reference.
- Lucid SDK
- submitClaimTx now:
- Pays to the claim script with inline claim datum.
- Locks the claimer bond at the escrow script with an EscrowDatum.
- submitFinalizeTx:
- Finds the claim UTxO (via Kupo helper, placeholder).
- Collects claim + escrow UTxOs and refunds claimer in the same tx.
- Kupo helper
- findClaimUtxoByEventId(kupoUrl, scriptAddress, eventIdHex) scaffolds fetching UTxOs and picking the right one. Replace with a precise datum filter using Kupo’s API or your indexer.

Next steps
- Add a dedicated challenge transaction:
- lock challenger bond at escrow (role=CHALLENGER_BOND),
- route ChallengeWin payouts appropriately.
- Bind escrow spends to the finalized claim using a reference input check (datum equality).
- Implement Kupo datum filtering (exact endpoint wiring) and decode inline datums from CBOR to match event_id.
- Upgrade relayer to use index-lucid.js end-to-end with real event extraction from Ogmios/Kupo.

⸻

## OctoChain Lab Kit V8

Added the Challenge path and reference-claim binding scaffolds, plus a Lucid submitter for challenges.

What v8 adds
- Claim validator (Aiken)
- Challenge spend must be before the deadline, re-output the claim at the same script with challenged=true, and include a challenger bond output (escrow recommended). (Datum validation of the re-output is marked TODO where you’ll decode and check fields.)
- Escrow validator (Aiken)
- Now expects a reference input to a claim with matching event_id when paying out (scaffold has_reference_claim to fill in).
- Lucid SDK
- submitChallengeTx(...) consumes the claim, re-creates it challenged, and locks a challenger bond at the escrow script.
- submitFinalizeTx(...) collects both the claim and the escrow and refunds the claimer after the window.
- Kupo helper
- Basic fetch + placeholder selection; slot in precise datum-based filtering.

Finish:
- Implement reference-claim checks in both validators (read reference inputs, decode datums, compare event_id + status).
- Define exact redeemer encodings and use them instead of Data.void() in Lucid.
- Add ChallengeWin resolution path:
- pay challenger from challenger-escrow and optionally slash claimer’s bond if fraud is proven,
- gate via proof (on-chain checks or SNARK verification, per your roadmap).

⸻

## OctoChain Lab Kit V9

Pushed the next pass with datum checks and a ChallengeWin path.
- Download v9 Kit (datum-checked Challenge + ref-claim escrow gating + ChallengeWin)

What’s new
- Aiken – Claim validator
- On Challenge, it now inspects the re-output’s inline datum and ensures it matches the original datum with only challenged flipped to true. (I left decode_claim_datum as a scaffold—swap in from_data<ClaimDatum> in your real project.)
- Aiken – Escrow validator
- Escrow spends require a reference claim; the scaffold ref_claim_matches is ready to enforce event_id and challenge status.
- Lucid SDK
- submitChallengeTx(...) re-outputs the claim as challenged=true and locks a challenger bond at escrow.
- submitChallengeWinTx(...) spends the challenger escrow using the claim as a reference, paying the challenger.
- submitFinalizeTx(...) collects the claim + claimer escrow and includes the claim as a reference input.
- Kupo helper
- New findUtxoByPredicate(...) for more precise selection; wire in inline-datum decoding to match event_id.

Short path to “prod ready”
1.	In Aiken:
- Replace scaffold decoders with real from_data for both Claim & Escrow datums.
- In escrow, fully enforce: reference-claim’s event_id equals this escrow’s, and challenged is False (Finalize) or True (ChallengeWin).
2.	In the relayer:
- Encode real redeemers instead of Data.void() (constructors must match Aiken).
- Decode inline datums from Kupo (CBOR) to pick exact UTxOs (by event_id & role).
3.	Economics:
- Add payout logic: on ChallengeWin, pay challenger; optionally slash claimer escrow.
- Treasury split / fees if desired.

⸻

## OctoChain Lab Kit V10

Replaced placeholder redeemers with concrete encodings and added inline-datum decoding + precise UTxO selection.
- Download v10 Kit (real redeemers + Kupo datum decoding)

Highlights
- Redeemers (Lucid Constr) aligned to Aiken order:
- Claim: Claim(claimer_pkh) = Constr(0, [...]), Challenge(challenger_pkh, reason) = Constr(1, [...]), Finalize() = Constr(2, []).
- Escrow: Finalize() = Constr(0, []), ChallengeWin() = Constr(1, []).
- Kupo helpers now parse inline datum CBOR to find the exact UTxOs:
- findClaimUtxoByEventId(kupoUrl, claimAddr, eventIdHex)
- findEscrowUtxo(kupoUrl, escrowAddr, roleStr, eventIdHex)
- SDK updates:
- All submitters (Claim, Challenge, Finalize, ChallengeWin) pass proper redeemers.
- Finalize/ChallengeWin include the claim UTxO as a reference input for escrow validation.

Next:
- swap the Aiken scaffolds for proper from_data<...> decoding in the on-chain code,
- add unit tests (Aiken test harness + JS e2e against a devnet with Ogmios+Kupo),
- or wire the ZK light client stub to gate Finalize instead of (or in addition to) the time window. ￼

⸻

## OctoChain Lab Kit V11

What changed in v11
- Aiken (on-chain)
- Swapped the scaffolds for real from_data decoding in both Claim and Escrow validators.
- Finalize is now gated by either:
- time window expiry or
- a ZK verifier reference UTxO carrying ZkDatum(event_id, epoch, ok=true).
- Included a dummy ZK verifier script (aiken_zk_verifier/) as a stand-in for your Noir/Plonk verifier.
- Tests
- Aiken test stubs for both validators (ready to expand with full Tx contexts).
- JS tests (relayer/tests/datum_redeemer.test.mjs) that round-trip datum encode/decode and verify redeemer encodings.
- Relayer / Lucid
- submitFinalizeTx(..., { zk: true }) will read a ZK proof UTxO (via ZK_SCRIPT_ADDRESS_*) instead of waiting out the window.
- Kupo helpers now decode inline datum CBOR to select exact claim/escrow UTxOs by event_id (and escrow role), plus a finder for ZK proof UTxOs.

Wire-up checklist
	1.	Addresses & keys
- Set CLAIM_SCRIPT_ADDRESS_*, ESCROW_SCRIPT_ADDRESS_*, ZK_SCRIPT_ADDRESS_*.
- Provide *_PAYMENT_SK_HEX, *_OGMIOS_URL, *_KUPO_URL.
	2.	Redeemer ordering
- Ensure your compiled Aiken constructors match:
- Claim: Claim(0), Challenge(1), Finalize(2)
- Escrow: Finalize(0), ChallengeWin(1)
	3.	ZK flow (stub → real)
- Replace the dummy ZK verifier with your actual verifier that only creates a proof UTxO when the SNARK validates.
- Option: mint a proof NFT at the verifier and require it as the reference input instead of decoding inline datum.
	4.	Tests you’ll likely add next
- Aiken: scenario tests—Claim→no challenge→Finalize (time); Claim→Challenge→ChallengeWin; Claim→ZK proof→Finalize (no time).
- JS: Kupo parsing of real responses; integration against a devnet Ogmios+Kupo.

Next:
- script a Noir→proof→verifier UTxO pipeline stub,
- or add full Aiken Tx-context tests (using fixtures) to enforce each state transition end-to-end. ￼

⸻

## OctoChain Lab Kit V12

What’s in v12
- Noir circuit stub (noir/partnerchain_praos_lc/)
- Nargo.toml + src/main.nr scaffold for a Praos-style light client. It currently just commits (event_id, epoch) — swap in real header/VRF checks when ready.
- Verifier UTxO creator (relayer/src/create-zk-utxo.js)
- After (stub) verifying a Noir proof, it mints a UTxO at your ZK_SCRIPT_ADDRESS_* carrying ZkDatum{ event_id, epoch, ok:true }.
- Usage: node src/create-zk-utxo.js <event_id_hex> <epoch> <proofPath>
- Aiken state-machine tests (scaffold) (aiken_tests_tx/)
- Three test files outline tx contexts for:
- Claim → Finalize (time path)
- Claim → Challenge → ChallengeWin
- Claim → Finalize (ZK path)

How to wire it together
	1.	Build a test proof (stub):
- cd noir/partnerchain_praos_lc && nargo check && nargo prove p --witness witness.toml
	2.	Post a zk_ok UTxO:
- cd relayer && node src/create-zk-utxo.js <event_id_hex> <epoch> noir/partnerchain_praos_lc/proofs/p.proof
	3.	Finalize on L1 with proof:
- Call your v11 submitFinalizeTx(..., { zk: true }) — the claim script accepts via the reference ZK UTxO.

Next upgrades I can tackle
- Replace the Noir stub with a real Praos/VRF circuit and a tiny verifier service.
- Flesh out the Aiken tests into executable transaction fixtures (covering bond flows, challenge timing, payouts).
- Integrate proof NFT minting to make the ZK reference uniquely bindable per event_id.

⸻

## OctoChain Kit V0.1

Heck yeah — I’ve packaged a Version 0.1 you can start running and poking today, with both the Praos/VRF circuit outline and a minimal verifier service wired into the zk flow.
- Download PartnerChain Bridge Kit — v0.1

What you’re getting in v0.1
- Noir “Praos LC” circuit (stub but structured): noir/partnerchain_praos_lc/
- Public inputs laid out: event_id, epoch, header_hash, parent_hash, slot, vrf_output, vrf_threshold.
- Witness placeholders: vrf_secret, (future) inclusion proof.
- Returns a commitment; constraints are dummies you’ll swap for Poseidon + VRF gadget.
- witness.example.toml and a short README for nargo commands.
- Verifier Service (dev-only): verifier-service/
- Tiny Express server with POST /verify that shells out to nargo verify.
- You pass { event_id_hex, epoch, proof_path }; it returns { ok: true } only if the proof verifies.
- This sits between proving and minting the zk_ok UTxO—so you can test the trust-minimized finalize path.
- Relayer helper → zk_ok UTxO: relayer/src/create-zk-utxo.js
- Calls the verifier-service; only on ok does it build a Lucid tx to mint a ZK-proof UTxO at your ZK_SCRIPT_ADDRESS_*.
- Works with Ogmios+Kupo (Kupmios provider). Env-var driven.
- Dev scaffolding
- docker-compose.example.yml with placeholders for Ogmios/Kupo + verifier.
- RUNBOOK.md with step-by-step to run the whole thing locally.
- Top-level README.md & VERSION.txt.

Quick start (local)
1.	spin up the verifier:

cd verifier-service
cp .env.example .env
npm i
npm run dev  # :8787


2.	make a (stub) proof:

cd noir/partnerchain_praos_lc
cp witness.example.toml witness.toml  # fill values
nargo check
nargo prove p --witness witness.toml
nargo verify p --proof proofs/p.proof


3.	mint a zk_ok UTxO (guarded):

cd relayer
npm i
export ZK_SCRIPT_ADDRESS_L1=addr_test1...
export L1_OGMIOS_URL=ws://localhost:1337
export L1_KUPO_URL=http://localhost:1442
export L1_NETWORK=0
export L1_PAYMENT_SK_HEX=<hex>
export VERIFIER_URL=http://localhost:8787/verify

node src/create-zk-utxo.js <event_id_hex> <epoch> ../noir/partnerchain_praos_lc/proofs/p.proof


4.	finalize on L1 with ZK:
- Use the v11 submitFinalizeTx(..., { zk: true }) from earlier. It will read that zk_ok UTxO as a reference input.

What’s left to toggle into “real”
- Replace Noir’s XOR placeholders with:
- Poseidon (or similar) for commitments,
- a proper Praos/VRF gadget (or a Schnorr/ed25519 VRF you prefer),
- inclusion proof path (Merkle/Accumulator).
- Harden the verifier-service (validate the proof’s public inputs, authenticate clients, add logs).
- Deploy your Aiken validators (from v11), set CLAIM_SCRIPT_ADDRESS_*, ESCROW_SCRIPT_ADDRESS_*, ZK_SCRIPT_ADDRESS_*.

Next:
- drop in specific Poseidon hash + a recommended VRF gadget plan for Noir, or
- add a Dockerfile for the verifier-service and a Compose profile that also boots a preprod Ogmios+Kupo pair for you. ￼