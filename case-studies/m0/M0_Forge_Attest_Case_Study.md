# Forge Attest for M0 multisig operations

Recreating Ethereum Safe nonce 22 from the unchanged Foundry proposal script

M0's nonce 22 transaction can be reproduced without changing its original
proposal script or Foundry configuration. The demo captures the request that the
script builds, then uses Forge Attest to compare its complete Safe transaction
with the historical record. The canonical digest and signed Safe hash match
nonce 22 exactly. [1, 3, 6]

## The existing proposal process

The historical `ProposeTransferSwapFacilityOwner` script builds three calls,
simulates each as the Safe through `MultiSigBatchBase`, and proposes a batch
through `safe-utils`. The library encodes a MultiSendCallOnly delegatecall,
derives the Safe hash, signs it and passes a JSON request to the Safe service
using `curl` over FFI. [4, 5]

Forge Attest adds an independent comparison between the reviewed source and the
complete transaction presented for approval. This describes the public code
paths; it is not an account of M0's internal signer procedures.

## What nonce 22 actually does

| Order | Target | Call and effect |
| --- | --- | --- |
| 1 | ProxyAdmin `0x0f38D8A5583f9316084E9c40737244870c565924` | `transferOwnership`: ownership moves to timelock `0x23CA665c8a73292Fc7AC2cC4493d2cE883BBA468`. |
| 2 | SwapFacility `0xB6807116b3B1B321a390594e31ECD6e0076f6278` | `grantRole`: engineering Safe `0xb7A9B5f301eF3bAD36C2b4964E82931Dd7fb989C` receives `DEFAULT_ADMIN_ROLE`. |
| 3 | The same SwapFacility | `renounceRole`: Safe `0xdcf79C332cB3Fe9d39A830a5f8de7cE6b1BD6fD1` relinquishes `DEFAULT_ADMIN_ROLE`. |

The timelock receives ProxyAdmin ownership; the engineering Safe receives
SwapFacility admin rights. The upstream script's historical description does not
make that distinction accurately. Its actual calls and the historical execution
receipt do. [1, 2, 4]

All three inner calls send zero ETH. The outer transaction delegates to
MultiSendCallOnly `0x40A2aCCbd92BCA938b02010E17A5b8929b49130D` with zero Safe gas
and refund fields. It executed successfully on 2025-11-19 at Ethereum block
23,834,856. The receipt records the ownership transfer, role grant, role
renunciation and Safe `ExecutionSuccess` for the expected hash. [1, 2]

## What the demo verifies

| Measure | Result |
| --- | --- |
| Coverage | One complete transaction containing three ordered calls, Ethereum chain ID 1, Safe version 1.3.0, nonce 22. |
| Unchanged source | The proposal script and `foundry.toml` are byte-for-byte identical to upstream `9e194fa9410edaaa467a324a55fa07af0e65f5a8`. |
| Reproducible producer | Original upstream commit `9e194fa9410edaaa467a324a55fa07af0e65f5a8`, Foundry 1.8.1, Solidity 0.8.26 and fork block 23,834,855. |
| Native producer path | The original script builds and simulates the calls, signs with a public test key, and constructs its original `safe-utils` proposal request. |
| Independent verification | Raw and canonical digests are pinned. The `safe-service` reader with `safe-utils` profile checks the request's carried hash. Cast and Solidity independently derive the Safe hash; required service comparison checks the historical record. |
| Diff | Three added files, 87 lines of claim, workflow and instructions. No added executable code and no existing source or Foundry configuration changes. |
| CI | The dedicated M0 workflow passes with native capture and required service comparison. [8] |

The signed Safe transaction hash is:

```text
0xab7012df75fc7907ace6cb632bc3865ded2956c33fa2aab2d4b3d85167d485a6
```

The [demo draft PR](https://github.com/crossecute/evm-m-extensions/pull/1) contains
the pinned claim, instructions and dedicated CI workflow. Forge Attest's native
`capture = "safe-utils"` mode invokes the original script, captures the actual
JSON passed to `curl -d`, and returns a local HTTP 201 response. The supported
curl transport never forwards the proposal. Its endpoint Safe and body nonce
must match exactly one claim; missing, repeated, unclaimed or unsupported
requests fail verification. Capture is a transport adapter, not a network
sandbox for arbitrary producer code. [3, 6]

The original script requires a private key. This demo supplies public test key
`1` to produce its local request signature. That is not an owner approval. No real
key or historical signature fixture is used. The request's sender and local
signature are metadata; the complete canonical Safe transaction still matches
the one its owners signed historically. [3, 5, 6]

## Additional verification and operational benefits

| Review step | Existing public script | With Forge Attest |
| --- | --- | --- |
| Intended calls | Builds the calls and simulates each as the Safe. | Runs that unchanged code at a pinned historical block. |
| Source to request | The proposal code constructs and submits its request. | Captures that exact request locally and checks raw output and canonical digests. |
| Signed identity | `safe-utils` derives the hash for signing. | Cast and a separate Solidity implementation independently derive the complete EIP-712 hash and compare it with the reviewed expectation. |
| Service record | The proposal path submits to the Safe service. | Required service comparison checks the specified Safe and nonce. |
| Review evidence | Source and the executed transaction can be inspected. | A checked-in claim, CI result and machine-readable evidence make the comparison repeatable. |

Changed targets, arguments, call order, Safe, chain, nonce or signed gas/refund
fields change the signed hash. These checks establish transaction equality.
Reviewers still assess whether the permissions change is appropriate. [6]

The reduced demo retains the original per-call simulation. It does not add full
Safe execution with historical signatures or separate post-state assertions.
The historical receipt provides evidence of the actual transaction's execution.

## Adoption and scope

The demo fork is
[crossecute/evm-m-extensions](https://github.com/crossecute/evm-m-extensions).
It retains upstream dependency gitlinks, including `common` revision
`e54273e79cbd8a38b2d2b41ca4123f4e671fcc07` and Recon-Fuzz `safe-utils` revision
`eccb79f80cad0f3ad98137cf3e859aac5d66e425`. Separate dependency forks are not
required. This result does not cover `m-portal` or current M0 nonce and timelock
proposal paths. [3, 5, 7]

For future operations, a reviewed claim and passing attestation can accompany the
operational explanation. The shared capture mode was merged through
[Forge Attest PR 54](https://github.com/crossecute/forge-attest/pull/54) and pinned
in this demo. It removes the need for per-producer wrappers, transport helpers or
JSON export code for this supported path, while reusing the existing request
reader and hash checks. [6]

Operator time, reviewer time and net savings were not measured. The demo adds
no on-chain transaction, calldata field or execution cost. The MultiSend payload
remains 427 bytes, with 516 bytes of outer calldata.

## Reproduction, evidence and limits

Use the [checked-in claim](attest-nonce22.toml) with Forge Attest revision
`27727236af75dda63698d86ef3984b5206309634`:

```bash
export MAINNET_RPC_URL='YOUR_ETHEREUM_ARCHIVE_RPC_URL'
./attest.sh --config /path/to/attest-nonce22.toml --require-live --json
```

The claim clones the pinned producer and installs its historical dependencies.
The [demo instructions](https://github.com/crossecute/evm-m-extensions/blob/feat/attest-nonce-22/attest/README.md)
explain the local transport and public test key. Do not use `--broadcast` or
Forge Attest's signing/submission option `--attest` for this historical demo.

Evidence includes the [passing verdict](evidence/attestation.json),
[captured request](evidence/reproduced-safe-tx.json),
[canonical transaction](evidence/canonical-safe-tx.json),
[execution receipt](evidence/execution-receipt.json),
[fork block](evidence/fork-block.json) and
[dedicated CI metadata](evidence/ci-run.json).

Nonce 22 is already executed. Service comparison checks its historical record,
not a pending proposal. Reproduction requires an archive RPC and required service
comparison depends on the Safe service being available. The inherited M0 test,
gas and coverage workflows are separate from the dedicated attestation check.

Matching the transaction does not authenticate the operator's original source
commit or toolchain. This study does not claim that the original operator ran
this exact revision or used Foundry 1.8.1 in 2025.

## Sources

[[1] Safe service record for nonce 22](https://api.safe.global/tx-service/eth/api/v1/multisig-transactions/0xab7012df75fc7907ace6cb632bc3865ded2956c33fa2aab2d4b3d85167d485a6/)

[[2] Executed Ethereum transaction](https://etherscan.io/tx/0x0e510ee28b99bd2e99ca5e6e61145181fe71964a3bece9723e3417139b91f785)

[[3] Demo draft PR, captured request and claim](https://github.com/crossecute/evm-m-extensions/pull/1)

[[4] Unchanged historical M0 proposal script](https://github.com/m0-platform/evm-m-extensions/blob/9e194fa9410edaaa467a324a55fa07af0e65f5a8/script/ProposeTransferSwapFacilityOwner.s.sol)

[[5] Historical common batch helper](https://github.com/m0-platform/common/blob/e54273e79cbd8a38b2d2b41ca4123f4e671fcc07/script/MultiSigBatchBase.sol), [historical safe-utils proposal flow](https://github.com/Recon-Fuzz/safe-utils/blob/eccb79f80cad0f3ad98137cf3e859aac5d66e425/src/Safe.sol)

[[6] Forge Attest at the merged native-capture revision](https://github.com/crossecute/forge-attest/blob/27727236af75dda63698d86ef3984b5206309634/README.md)

[[7] Research identifying M0 and nonce 22](../../research/foundry-msig-proposal-research.md)

[[8] Successful M0 CI run using merged capture support](https://github.com/crossecute/evm-m-extensions/actions/runs/42104947713)
