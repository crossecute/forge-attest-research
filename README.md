# research

Research notes and case studies behind crossecute projects. Each project has its own section below.

## forge-attest

Research behind [`forge-attest`](https://github.com/crossecute/forge-attest): how EVM teams build, propose, and review multisig transactions with Foundry scripts, and a case study of applying `forge-attest` to one of them.

| Path | What it is |
|---|---|
| [`research/foundry-msig-proposal-research.md`](research/foundry-msig-proposal-research.md) | Survey of about 60 public repos that use Foundry scripts for Safe multisig work, grouped by style, with a dependency index and a comparison of the five reusable tools (Base `MultisigScript`, Optimism `MultisigTask`, Zeus `MultisigBuilder`, `forge-safe`, `safe-utils`). Research date 2026-11-04. |
| [`case-studies/frax/Frax_Forge_Attest_Case_Study.md`](case-studies/frax/Frax_Forge_Attest_Case_Study.md) | Case study of `forge-attest` on Frax's Aurora OFT multisig operations: five Fraxtal-to-Aurora batches (30 calls, Safe nonces 554 to 558) reproduced and hash-verified in one 69-second CI run, a comparison with the current manual review steps, and a proposed rollout. |

Findings that matter for `forge-attest`:

- Most teams submit a script's output by one of three routes: a library that posts to the Safe Transaction Service (`forge-safe`, `safe-utils`, Zeus CLI), a Safe Transaction Builder JSON file that signers import (ether.fi, Royco, Boundless and others), or signatures collected off-chain and sent by a facilitator (Base, Optimism). The JSON-file route is the one with a separate upload step between script and Safe.
- Base is the only framework found that checks the signed hashes and expected state changes against a separate validation file before signing. Optimism checks state changes in Solidity. The other tools leave that to each project.
- Nonce handling differs by tool and is a place where a queued transaction can be replaced or collide: `forge-safe` queues behind pending proposals, `safe-utils` uses the on-chain nonce, and Zeus leaves it to Safe protocol-kit.

Method and limits: candidates came from GitHub code search, which is incomplete. Large orgs such as Morpho, Uniswap, Aave/BGD and Lido returned nothing in the index, which does not show they avoid this pattern. The research file lists exactly what was read in full, what was read by function inventory, and what was not read.

## protocol

Research behind [`crossecute/protocol`](https://github.com/crossecute/protocol). The protocol's normative documents stay in that repository; this section holds the findings about other people's systems, which go stale on their release schedule rather than ours.

| Path | What it is |
|---|---|
| [`research/provider-research.md`](research/provider-research.md) | What each transport (LayerZero, CCIP, Hyperlane, Wormhole, OP Stack, Arbitrum, Avalanche) guarantees about replay, how ERC-7786 attaches, and the reasoning behind each native binding. The design influences and candidate additions each suggests, with the status of each. Version-pinned to the other projects' source as read on the dates given in the file. |
