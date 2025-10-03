---
layout: default
title: Network - octomics
permalink: /network/
---

# Network

**The blockchain based network powering community octomics.**

Implementated via the [selfdriven Network Infra Interface ](https://selfdriven.network).

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

Two sovereign Cardano-family networks, each running Ouroboros (L1 = Cardano devnet/preprod; L2 = your PartnerChain):
	•	L1 Cardano: where ADA/tokens are locked in a Plutus bridge “Lockbox”.
	•	L2 PartnerChain (Ouroboros-Praos): where wrapped assets (sADA, sTOKEN) are minted/burned.
	•	Relayers (≥N, threshold K): watch both chains, build canonical BridgeEvent objects, co-sign them, and submit proofs.
	•	Attestation keys: separate from staking/VRF/KES keys; rotated via on-chain governance.

Core dataflow:
	1.	Lock (L1) → Lockbox outputs a new LockedUTxO (datum includes amount, target chain, recipient).
	2.	Relayers see the Lock, form EventID, threshold-sign Attestation.
	3.	PartnerChain Bridge verifies K-of-N signatures, mints sADA to the recipient.
	4.	Reverse direction: Burn (L2) → attested → Unlock (L1).

⸻

### B) Ceremonies & chain parameters (Ouroboros-Praos)

B1. PartnerChain genesis (Conway)

Pick sane defaults for a small, permissioned validator set:
	•	Security: Praos with f=0.2, activeSlotsCoeff ~ 0.2, slotLength = 1s, epochLength = 43200 (≈12h), k=2160.
	•	KES: KES period ~ 1d; automate rotation.
	•	Initial funds: bootstrap treasury + faucet.
	•	Decentralization: start with fixed validator set (N=4–7 pools), move to delegation later.

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
cardano-cli node key-gen                --cold-verification-key-file cold.vkey --cold-signing-key-file cold.skey --operational-certificate-issue-counter cold.counter
cardano-cli node issue-op-cert          --kes-verification-key-file kes.vkey --cold-signing-key-file cold.skey --operational-certificate-issue-counter cold.counter --kes-period 0 --out-file node.opcert

Topology
	•	1× BP per validator; 1–2× relays per validator.
	•	BP peers only with its relays; relays peer with other relays; publish relay addresses in topology.json.

Time sync: enable NTP on every host. Keep KesPeriod rotation on a cron with monitoring.

B2. Attestation key ceremony (for relayers)
	•	Generate Ed25519 attestation keys offline for each relayer.
	•	Publish Attesters.vkey[] on both chains (L1 Lockbox datum; L2 Bridge state).
	•	Store minThreshold K alongside the list (K-of-N).
	•	Version keys via a governance “param cell” so you can rotate without redeploys.

⸻

### C) Bridge contracts (high-level designs)

You’ll implement two Plutus state machines (Aiken/Plutus V2 are both fine). Below are compact schemas you can translate to Aiken or Plutus.

C1. On L1: Lockbox validator (Plutus state machine)

Purpose: take real ADA/tokens into escrow; emit a canonical, hashable event.

Datum (at each LockedUTxO):

type LockDatum = {
  event_id: ByteString,        -- H(blake2b-256): chain_id||tx_id||ix||recipient||amount||nonce
  origin_chain: ByteString,    -- "cardano-l1"
  target_chain: ByteString,    -- "partnerchain-7777"
  recipient_sc: ByteString,    -- bytes of L2 bech32/addr
  asset: AssetClass,           -- ADA or policy+assetname
  amount: Integer,
  attesters_hash: ByteString,  -- H(concat(attester_pubkeys || threshold K))
  nonce: Integer
}

Redeemers:
	•	Lock: creates a new state UTxO with amount locked.
	•	Unlock: spends a previously Burned attested event from L2; checks Attestation.

Attestation structure (Carried in Unlock redeemer):

type Attestation = {
  event_id: ByteString,
  source_chain: ByteString,   -- "partnerchain-7777"
  signatures: [Signature],    -- ed25519 over (domain || event_id || direction)
  bitmap: ByteString,         -- which relayers signed
  threshold: Integer
}

Validation rules (Unlock):
	•	hash(attesters_pubkeys, threshold) equals attesters_hash in current state.
	•	Verify K valid signatures over the domain-separated message.
	•	Confirm the referenced event_id corresponds to a Burn event on L2 (the event content is embedded or reconstructed deterministically).
	•	Recreate the exact LockedUTxO value and release it to the recipient L1 address.

C2. On L2: Bridge & MintPolicy

Two parts:
	1.	Bridge validator (mirrors Lockbox) maintains the attester set & thresholds, processes incoming Lock attestations from L1, and releases/mints sAssets.
	2.	Minting policy for sADA/sTOKEN:
	•	Mint only when called by the Bridge validator with a valid Lock Attestation from L1.
	•	Burn only when called by Bridge with a Burn redeemer (which emits an L2 BurnEvent to later unlock on L1).

L2 Lock → Mint flow:
	•	Input: Attestation{ event_id from L1 Lock }.
	•	Checks K-of-N attesters.
	•	Mints sADA (policy id policy_sADA) to recipient_sc.

L2 Burn → Event flow:
	•	User spends their sADA with Burn redeemer.
	•	Bridge ensures exact burn amount and emits BurnEvent with a new event_id.
	•	Relayers threshold-sign and later Unlock on L1.

⸻

### D) Relayer cluster (threshold attestation)
	•	At least N relayer nodes; require K signatures.
	•	Each relayer is stateless (everything derivable from chain); private key only for attestations.
	•	Consensus between relayers is by deterministic event ordering (EventID), not leader election.

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

const state = { lastPointL1: null, lastPointL2: null };

Promise.resolve()
  .then(() => Promise.all([CardanoClient.connect(), PartnerClient.connect()]))
  .then(([l1, l2]) => {
    function loop() {
      return getEventsSince(l1, 'LOCK', state.lastPointL1)
        .then(lockEvents => Promise.all(lockEvents.map(ev => {
          const id = eventId(ev);
          const m = msgToSign(id);
          return sign(process.env.RELAYER_SK, m).then(sig => ({ id, sig, ev }));
        })))
        .then(partialSigs => submitTx(l2, 'MINT', partialSigs)) // l2 bridge collects ≥K
        .then(() => getEventsSince(l2, 'BURN', state.lastPointL2))
        .then(burnEvents => Promise.all(burnEvents.map(ev => {
          const id = eventId(ev);
          const m = msgToSign(id);
          return sign(process.env.RELAYER_SK, m).then(sig => ({ id, sig, ev }));
        })))
        .then(partialSigs => submitTx(l1, 'UNLOCK', partialSigs))
        .then(() => setTimeout(loop, 800)) // simple poll; replace with websockets in prod
        .catch(err => { console.error(err); setTimeout(loop, 1500); });
    }
    return loop();
  });

In production, move to WebSocket subscriptions (Ogmios on both chains), add replay protection (dedup store), and use aggregate signature (BLS) or canonical multi-sig with explicit bitmap.

⸻

### E) docker-compose.yml (local dual-network + relayer + tooling)

Below is a single-host lab that brings up:
	•	l1-node (Cardano devnet) + Ogmios + Kupo
	•	l2-node (PartnerChain, Ouroboros-Praos) + Ogmios + Kupo
	•	relayer (Node.js)
	•	Optional: db-sync pairs if you need SQL (commented)

You’ll need to drop your generated configs/keys into ./l1/ and ./l2/ as noted.

version: "3.9"
services:
  l1-node:
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
      - ./l1/config:/config:ro
      - ./l1/data:/data
      - ./l1/ipc:/ipc
    ports: ["3001:3001"]

  l1-ogmios:
    image: cardanosolutions/ogmios:v6.5.0
    environment:
      - OGMIOS_NODE=/ipc/node.socket
    volumes:
      - ./l1/ipc:/ipc
    ports: ["1337:1337"]
    depends_on: [l1-node]

  l1-kupo:
    image: cardanosolutions/kupo:v2.8.0
    command: [
      "--node-socket","/ipc/node.socket",
      "--since","origin",
      "--match","*",
      "--workdir","/kupo"
    ]
    volumes:
      - ./l1/ipc:/ipc
      - ./l1/kupo:/kupo
    ports: ["1442:1442"]
    depends_on: [l1-node]

  l2-node:
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
      - ./l2/config:/config:ro        # your PartnerChain genesis/config
      - ./l2/data:/data
      - ./l2/ipc:/ipc
    ports: ["4001:4001"]

  l2-ogmios:
    image: cardanosolutions/ogmios:v6.5.0
    environment:
      - OGMIOS_NODE=/ipc/node.socket
    volumes:
      - ./l2/ipc:/ipc
    ports: ["2337:2337"]
    depends_on: [l2-node]

  l2-kupo:
    image: cardanosolutions/kupo:v2.8.0
    command: [
      "--node-socket","/ipc/node.socket",
      "--since","origin",
      "--match","*",
      "--workdir","/kupo"
    ]
    volumes:
      - ./l2/ipc:/ipc
      - ./l2/kupo:/kupo
    ports: ["2442:2442"]
    depends_on: [l2-node]

  relayer:
    build: ./relayer
    environment:
      - L1_OGMIOS=ws://l1-ogmios:1337
      - L2_OGMIOS=ws://l2-ogmios:2337
      - RELAYER_SK=env:RELAYER_SK_HEX
      - ATTESTERS_JSON=/config/attesters.json
      - THRESHOLD_K=2
    volumes:
      - ./relayer/config:/config:ro
    depends_on: [l1-ogmios, l2-ogmios]

Folder layout you should create:

partnerchain/
  docker-compose.yml
  l1/
    config/{config.json, topology.json, genesis.json, alonzo-genesis.json, conway-genesis.json}
    ipc/  data/  kupo/
  l2/
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
	•	L1: either use preprod configs or make a devnet with magic 42.
	•	L2: create PartnerChain genesis (magic 7777) with your Praos params.
	2.	Start compose: docker compose up -d.
	3.	Deploy contracts:
	•	L1: publish Lockbox with initial attesters_hash, threshold K.
	•	L2: publish Bridge validator + sADA policy bound to Bridge.
	4.	Fund faucet on both networks (UTxO keys created during genesis).
	5.	Test flows:
	•	Lock → Mint: L1 lock 100 ADA to L2 recipient; confirm sADA=100 on L2.
	•	Burn → Unlock: L2 burn 30 sADA; confirm 30 ADA released on L1.

CLI sketches (Lock on L1):

cardano-cli address key-gen --verification-key-file user.vkey --signing-key-file user.skey
RECIP_SC="addr_test1q..."   # L2 recipient (encoded bytes in datum)
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

Relayer logs should show the new Lock event, signatures, and an L2 mint submission.

F2. Testnet (preprod/preview)
	•	Re-deploy Lockbox & Bridge with identical Attester set and threshold.
	•	Run ≥5 relayers across 2+ clouds/AZs.
	•	Stand up 4–7 validators for PartnerChain with distinct stake pools.
	•	Add Ogmios+Kupo for public read APIs.
	•	Add Prometheus/Grafana & alerting (KES expiry, empty relay queue, attestation lag).

F3. Mainnet
	•	Key ceremony for production attesters (HSM/YubiHSM or Hashicorp Vault).
	•	Add governance script to rotate attester set (datum update via DAO or multisig).
	•	Audits (contracts + relayer + node hardening).

⸻

### G) Security, ops, and hardening checklist
	•	Replay & uniqueness
	•	Every event has a unique EventID; keep a Nonce and maintain a spent EventID set in validator state to prevent re-use.
	•	Threshold control
	•	Store attesters_pubkeys + K on-chain (datum), not in off-chain config.
	•	Expose a controlled update path (governance) to change keys/K.
	•	Signature domain separation
	•	Prefix messages with constant domain bytes ("partner-bridge-v1") and direction (LOCK→MINT / BURN→UNLOCK).
	•	Liveness
	•	≥N relayers, geo-distributed; exponential backoff + queueing.
	•	Watchdog: “no attested events in X mins” = page.
	•	Integrity
	•	Double-read events via two indexers (Ogmios + Kupo/db-sync).
	•	KES/VRF
	•	Rotate KES automatically; monitor remainingSlots.
	•	Keep cold keys offline; use air-gapped ceremony for op-certs.
	•	Rate limits
	•	Bridge contract enforces per-tx caps until liquidity & monitoring mature.
	•	Upgrades
	•	Version every datum (e.g., version: 1) to migrate safely.
	•	Keep an escape hatch: emergency pause (requires higher quorum).

⸻

Where to take it next (optional but recommended)
	1.	Aggregate signatures (BLS) to shrink on-chain attestation size and fees.
	2.	Optimistic fraud-proofs: allow anyone to challenge an invalid attestation within a window.
	3.	ZK light client (trust-minimized):
	•	Prove PartnerChain block header validity (Praos rules) with a zk circuit.
	•	Verify succinct proof in a Cardano Plutus verifier (or via a specialized on-chain verifier once available).

⸻

- [Network Kit - Plutus](/network/kit/plutus)

