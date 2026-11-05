# Forge Attest for M0 multisig operations

Recreating Ethereum Safe nonce 22 from the Foundry proposal script

M0's nonce 22 transaction can be reproduced from its Foundry script and bound to
the complete transaction its Safe owners signed. This demo regenerates the three
SwapFacility administration calls, verifies their state changes on a historical
Ethereum fork, and passes Forge Attest's independent hash checks and required
Safe Transaction Service comparison. The resulting Safe hash matches the
executed transaction exactly. [1, 2, 3]

## The existing proposal process

The historical `ProposeTransferSwapFacilityOwner` script builds three calls,
simulates them as the Safe through `MultiSigBatchBase`, then proposes the batch
through `safe-utils`. That library encodes a MultiSendCallOnly delegatecall,
derives the Safe transaction hash, signs it and submits the proposal to the Safe
Transaction Service. These are useful existing controls. [4, 5]

The additional review question is whether the complete transaction presented for
approval is the one the reviewed source produces. Forge Attest makes that check
repeatable from a pinned source revision, toolchain, chain state and expected
hashes. This comparison describes the public code paths; it is not an account of
M0's internal signer procedures. [3, 4, 5]

## What nonce 22 actually does

| Order | Target | Call and effect |
| --- | --- | --- |
| 1 | ProxyAdmin `0x0f38D8A5583f9316084E9c40737244870c565924` | `transferOwnership`: ownership moves to timelock `0x23CA665c8a73292Fc7AC2cC4493d2cE883BBA468`. |
| 2 | SwapFacility `0xB6807116b3B1B321a390594e31ECD6e0076f6278` | `grantRole`: engineering Safe `0xb7A9B5f301eF3bAD36C2b4964E82931Dd7fb989C` receives `DEFAULT_ADMIN_ROLE`. |
| 3 | The same SwapFacility | `renounceRole`: proposing Safe `0xdcf79C332cB3Fe9d39A830a5f8de7cE6b1BD6fD1` relinquishes `DEFAULT_ADMIN_ROLE`. |

The distinction between the two recipients matters: the timelock receives
ProxyAdmin ownership, while the engineering Safe receives SwapFacility admin
rights. The upstream script's historical description did not make that
distinction accurately. The reconstruction checks the actual calls and resulting
permissions. [1, 2, 4]

All three inner calls send zero ETH. The outer transaction delegates to
MultiSendCallOnly `0x40A2aCCbd92BCA938b02010E17A5b8929b49130D` with zero Safe gas
and refund fields. It executed successfully on 2025-11-19 at Ethereum block
23,834,856. The receipt records the ownership transfer, role grant, role
renunciation and Safe `ExecutionSuccess` for the expected hash. [1, 2]

## What the demo demonstrated

| Measure | Verified result |
| --- | --- |
| Coverage | One complete Safe transaction containing three ordered calls, Ethereum chain ID 1, Safe version 1.3.0, nonce 22. |
| Historical source | Upstream `9e194fa9410edaaa467a324a55fa07af0e65f5a8`, including its pinned `common` and `safe-utils` dependencies. |
| Reproducible producer | Demo commit `361b8c2334887af62d751726c3bbbe53e802a179`, Foundry 1.8.1, Solidity 0.8.26 and fork block 23,834,855. |
| Simulation and execution | M0's original per-call simulation passes. Full Safe execution with the two public historical signatures also passes, advances the nonce from 22 to 23 and satisfies all three permissions checks. |
| Independent verification | Raw output and canonical digests match their pins. Cast, Forge Attest's Solidity cross-check, the historical Safe contract and the Safe service agree on the signed hash. |
| Rejection checks | Changed calldata cannot use the historical approvals. Incorrect canonical and signed-hash expectations produce `NOT ATTESTED` and exit code 1. |
| CI result | Successful replay and attestation job in 64 seconds, including setup, dependency installation, fork tests and required service comparison. [8] |

The Safe transaction hash is:

```text
0xab7012df75fc7907ace6cb632bc3865ded2956c33fa2aab2d4b3d85167d485a6
```

The [demo draft PR](https://github.com/crossecute/evm-m-extensions/pull/1) contains
the replay, fork tests, pinned claim and CI workflow. The original proposal entry
point and replay share one batch builder, preserving the original targets,
arguments and order. The replay uses the historical `safe-utils` encoder; it does
not copy transaction calldata from the service fixture. Only public historical
signatures are read from that fixture. [3, 6]

## Additional verification and operational benefits

| Review step | Existing public script | With the demo's Forge Attest claim |
| --- | --- | --- |
| Intended calls | Builds the calls in Solidity and simulates them with the Safe as caller. | Reuses that builder and simulation, then asserts all three permissions changes. |
| Complete Safe execution | The historical helper simulates target calls individually. | Also executes the full MultiSend through the Safe with historical owner signatures on the local fork. |
| Source to transaction | The proposal code builds and signs its transaction. | Independently reruns the pinned producer and checks both exact output bytes and the canonical transaction digest. |
| Signed identity | `safe-utils` derives the transaction hash for signing. | Cast and a separate Solidity implementation derive the same complete EIP-712 hash and compare it with the reviewed expectation. |
| Service record | The proposal path submits to the Safe service. | Required service comparison checks the transaction at the specified Safe and nonce. |
| Review evidence | Source and the executed transaction can be inspected. | Checked-in claims, replay tests and machine-readable verdicts make the comparison repeatable. |

Changes to targets, arguments, call order, the Safe, chain, nonce or signed
gas/refund fields change the signed transaction hash. Reviewers can compare the
full expected hash independently of the transaction interface. The demo's explicit
state assertions add evidence about the intended effects for this particular
operation. Neither hash equality nor those assertions replace review of whether
the permissions change itself is appropriate. [3, 6]

## Adoption across M0 codebases

The demo fork is
[crossecute/evm-m-extensions](https://github.com/crossecute/evm-m-extensions).

The implemented demo covers the nonce 22 script in `evm-m-extensions`. It retains
historical dependency gitlinks, including Recon-Fuzz's `safe-utils` revision
`eccb79f80cad0f3ad98137cf3e859aac5d66e425`. These dependencies come from upstream;
separate `common` and `safe-utils` forks are not required. The demo does not use
`m-portal` or validate the current M0 `SafeNonce` and timelock proposal paths.
[3, 5, 7]

For future operations, a suitable next step is a deterministic export before
signing, with a reviewed claim and passing CI beside the operational explanation.
Proposing owners and later signers can then regenerate the same transaction
locally. Forge Attest supports verified proposal and confirmation workflows, but
this demo exercises reproduction and verification only. It requests no new
signature and submits no proposal. [6]

The expected saving is repeated mechanical comparison across operators and
signers. Operator time, review time and net savings were not measured. The demo
adds no on-chain transaction, calldata field or execution cost: the reproduced
MultiSend payload remains 427 bytes, with 516 bytes of outer calldata.

## Reproduction, evidence and limits

Use the [checked-in claim](attest-nonce22.toml) with Forge Attest revision
`998d11c9325990a74c534ef406a968c509d1544a`:

```bash
export MAINNET_RPC_URL='YOUR_ETHEREUM_ARCHIVE_RPC_URL'
./attest.sh --config /path/to/attest-nonce22.toml --require-live --json
```

The claim clones the pinned producer and installs its historical dependencies.
The [demo instructions](https://github.com/crossecute/evm-m-extensions/blob/feat/attest-nonce-22/attest/README.md)
also provide the direct replay and test commands. CI runs the fork tests and the
same claim with required service comparison. The
[successful run](https://github.com/crossecute/evm-m-extensions/actions/runs/42092875046)
and [saved run metadata](evidence/ci-run.json) record the result. [3, 8]

Local evidence includes the [passing verdict](evidence/attestation.json),
[rejected claim verdict](evidence/rejected-claim.json),
[regenerated transaction](evidence/reproduced-safe-tx.json),
[canonical transaction](evidence/canonical-safe-tx.json),
[execution receipt](evidence/execution-receipt.json) and
[fork block](evidence/fork-block.json). Verification was performed on 2026-11-05.

Nonce 22 is already executed. A passing service comparison refers to its
historical record, not a pending proposal. Reproduction requires an Ethereum
archive RPC; required service verification also depends on the Safe service being
available. The demo does not broadcast to Ethereum.

The selected historical source reproduces the signed transaction exactly. The
service record does not authenticate the operator's original source commit or
toolchain, so this case study does not claim that the operator ran this exact
revision or used Foundry 1.8.1 in 2025.

## Sources

[[1] Safe service record for nonce 22](https://api.safe.global/tx-service/eth/api/v1/multisig-transactions/0xab7012df75fc7907ace6cb632bc3865ded2956c33fa2aab2d4b3d85167d485a6/)

[[2] Executed Ethereum transaction](https://etherscan.io/tx/0x0e510ee28b99bd2e99ca5e6e61145181fe71964a3bece9723e3417139b91f785)

[[3] Demo draft PR, claim, replay and tests](https://github.com/crossecute/evm-m-extensions/pull/1)

[[4] Historical M0 proposal script](https://github.com/m0-platform/evm-m-extensions/blob/9e194fa9410edaaa467a324a55fa07af0e65f5a8/script/ProposeTransferSwapFacilityOwner.s.sol)

[[5] Historical common batch helper](https://github.com/m0-platform/common/blob/e54273e79cbd8a38b2d2b41ca4123f4e671fcc07/script/MultiSigBatchBase.sol), [historical safe-utils encoder and proposal flow](https://github.com/Recon-Fuzz/safe-utils/blob/eccb79f80cad0f3ad98137cf3e859aac5d66e425/src/Safe.sol)

[[6] Forge Attest at the pinned verifier revision](https://github.com/crossecute/forge-attest/blob/998d11c9325990a74c534ef406a968c509d1544a/README.md)

[[7] Research identifying M0's codebases and nonce 22](../../research/foundry-msig-proposal-research.md)

[[8] Successful replay and Forge Attest CI run](https://github.com/crossecute/evm-m-extensions/actions/runs/42092875046)
