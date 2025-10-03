---
layout: default
title: Network - octomics
permalink: /network/kit/plutus
---

## octomics.network Kit - Plutis

[Download the PartnerChain Bridge Kit](https://github.com/selfdriven-octo/octomics/kits/)

What you get:
- aiken/
- validators/lockbox.ak (L1 escrow)
- validators/bridge.ak (L2 bridge)
- policies/mint_sasset.ak (sADA/sTOKEN policy)
- aiken.toml
- relayer/ (Node.js, noble-ed25519, Ogmios WS)
- src/index.js, src/ogmios.js, src/threshold.js, src/util.js, src/sdk.js
- config/attesters.json (K-of-N example)
- Dockerfile, package.json
- docker-compose.yml (L1+L2 nodes, Ogmios, Kupo, Relayer)
- l1/, l2/ folders with placeholders for your configs
- README with quick steps

Quick start
	1.	Drop your real Cardano configs:

- ./l1/config/ → your L1 (devnet/preprod) config.json, topology.json, genesis.json, alonzo-genesis.json, conway-genesis.json
- ./l2/config/ → your PartnerChain (Ouroboros) equivalents (genesis + configs)

	2.	Set attesters + key:

- Edit relayer/config/attesters.json (pubkeys + thresholdK).
- In docker-compose.yml, set RELAYER_SK_HEX (this relayer’s Ed25519 secret key).

	3.	Bring it up:

docker compose up -d

	4.	Compile Aiken contracts:

cd aiken
aiken build

(They’re stubs — wire your real checks where marked TODO.)
	5.	Deploy:

- L1: deploy Lockbox with attesters_hash and threshold in datum.
- L2: deploy Bridge + minting policy bound to Bridge.

	6.	Test flow:

- Lock ADA on L1 → relayer picks it up → mints sADA on L2.
- Burn sADA on L2 → relayer picks it up → unlocks ADA on L1.

Notes
- The relayer uses simple WS stubs; switch to proper Ogmios ChainSync + Kupo queries in src/ogmios.js/src/sdk.js.
- util.js uses sha256 as a placeholder; swap in real blake2b-256 (e.g., @noble/hashes/blake2b) for EventID.
- Mint/burn policy currently trusts Bridge; enforce tx context (inputs/refs) before shipping.
- Keep attester keys in HSM/Vault for prod; rotate via a datum/governance path.
￼