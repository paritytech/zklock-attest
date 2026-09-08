> The following is a prototype, reference implementation, and proof-of-concept. This open source code is provided for research, experimentation, and developer education only. This code has not been audited, is actively experimental, and may contain bugs, vulnerabilities, or incomplete features. Use at your own risk.

# zklock-attest

Public, permanent attestation log for zkLock — an offline physical lock whose trust is
anchored to a Polkadot SDK chain. This repository holds the evidence a lock needs to advance
its BEEFY authority-set trust across validator rotations without being re-provisioned.

The code, protocol, and producer live in the `zklock` repository; this one carries only data
that must stay publicly fetchable forever.

## Layout

Everything is keyed by network id. Each network is a triple — BEEFY relay, Asset Hub
(registry), Bulletin (distribution) — and its directory is self-contained: profile plus log
is everything a verifier needs.

```text
networks/<network>.json                        chain identity, provisioned BEEFY anchor,
                                               SP1 verification key (the trust contract)
handover-log/<network>/handover-<setid>.json   one attestation per authority set, append-only
handover-log/<network>/index.json              per-entry size, Blake2b-256 hash, Bulletin CID,
                                               plus the network id and anchor it belongs to
```

Networks currently published: `pnv2` (Paseo Next V2). Adding a network adds one profile and
one log directory; nothing is shared across networks.

An attestation is a BEEFY commitment signed by authority set N together with the MMR leaf
naming set N+1's keyset commitment. It can only be captured while set N is live (one relay
session, about an hour) — but once captured it is permanent evidence: a lock that missed any
number of sessions replays the segment it missed, in order, and catches up.

## Why a repository, not a chain

Attestations are permanent; the Bulletin chain retains data for 201,600 blocks (14 days).
Because attestation bytes are fixed and Bulletin is content-addressed, anyone holding these
files can re-store them and obtain the same CID recorded in `index.json` — expiry costs
availability, never the reference. Git is the durable home; Bulletin is an on-demand cache.

Cost: 1,704 bytes per authority set, roughly 15 MB per year per network.

## Verify

From a `zklock` checkout:

```sh
cargo run -p zklock-certificate -- verify-handover-log <path-to>/handover-log/pnv2
```

This replays every entry against the network's pinned trust anchor and reports the authority
set reached. Each entry must verify under the set the previous entry established; a segment
with a gap is rejected rather than skipping a rotation.

## Integrity

- Entries are append-only; an existing `handover-<setid>.json` never changes.
- `index.json` records each entry's byte size, Blake2b-256 content hash, and CID.
- The signing chain bottoms out at each network's provisioned BEEFY authority set, published
  in `networks/<network>.json` and pinned in the `zklock` repository — this repository asserts
  lineage; locks carry their own anchor.
- Profiles are mutable (endpoints rot, anchors get re-dated); attestations never are.
