---
layout: default
title: Network - octomics
permalink: /network/kit/plutus
---

## OctoChain Kit - Plutus

[Download the OctoChain Bridge Kit](https://github.com/selfdriven-octo/octomics/kits/)

What you get:
- aiken/
- validators/lockbox.ak (mainnet escrow)
- validators/bridge.ak (octonet bridge)
- policies/mint_sasset.ak (sADA/sTOKEN policy)
- aiken.toml
- relayer/ (Node.js, noble-ed25519, Ogmios WS)
- src/index.js, src/ogmios.js, src/threshold.js, src/util.js, src/sdk.js
- config/attesters.json (K-of-N example)
- Dockerfile, package.json
- docker-compose.yml (mainnet+octonet nodes, Ogmios, Kupo, Relayer)
- mainnet/, octonet/ folders with placeholders for your configs
- README with quick steps

Quick start
	1.	Drop your real Cardano configs:

- ./mainnet/config/ → your mainnet (devnet/preprod) config.json, topology.json, genesis.json, alonzo-genesis.json, conway-genesis.json
- ./octonet/config/ → your PartnerChain (Ouroboros) equivalents (genesis + configs)

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

- mainnet: deploy Lockbox with attesters_hash and threshold in datum.
- octonet: deploy Bridge + minting policy bound to Bridge.

	6.	Test flow:

- Lock ADA on mainnet → relayer picks it up → mints sADA on octonet.
- Burn sADA on octonet → relayer picks it up → unlocks ADA on mainnet.

Notes
- The relayer uses simple WS stubs; switch to proper Ogmios ChainSync + Kupo queries in src/ogmios.js/src/sdk.js.
- util.js uses sha256 as a placeholder; swap in real blake2b-256 (e.g., @noble/hashes/blake2b) for EventID.
- Mint/burn policy currently trusts Bridge; enforce tx context (inputs/refs) before shipping.
- Keep attester keys in HSM/Vault for prod; rotate via a datum/governance path.
￼