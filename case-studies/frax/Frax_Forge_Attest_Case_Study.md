# Forge Attest for Frax multisig operations

Aurora OFT demo and adoption across the organization

Frax can deterministically bind multisig transactions to simple, human-readable Foundry scripts. The Aurora demo demonstrates that Forge Attest can regenerate five real transaction batches, verify their complete Safe transaction hashes, and record a repeatable result in CI. The operational benefit is to use verified transactions for proposals and signing across the organization. [2, 3, 4]

## The current review process

The Aurora signer notes already provide meaningful review material. They identify transaction files, Safe nonces and hashes, explain the intended calls, and include manual calldata decoding examples. Signers can use that material alongside the repository and Safe interface to assess the operation. [1]

The remaining work is to establish that the submitted payload is exactly what the reviewed code produces. A written explanation, a decoded example, and a displayed transaction each contribute evidence, but a reviewer still needs to connect them to the complete batch. Forge Attest automates that connection and makes the result repeatable for every signer. [3, 4]

## What the demo demonstrated

| Measure | Verified result |
| - | - |
| Coverage | Five Fraxtal-to-Aurora OFT batches for frxUSD, frxETH, sfrxETH, sfrxUSD and WFRAX. |
| Payloads | 30 underlying calls across Safe nonces 554 through 558. |
| Reproduction | Pinned producer commit, Foundry version and historical Fraxtal block 41874116. |
| Verification | Canonical transaction digests and Safe hashes matched all five claims, with independent cast and Solidity hash derivations. |
| CI result | Successful job in 69 seconds, including setup, dependencies and verification. [2, 3] |

A single demo PR contains the producer setup, the standard Forge Attest action and the claims for all five transaction batches. Each batch is verified in the same CI run. [2, 3]

## Additional security and operational benefits

| Review step | Manual Verification | With Forge Attest |
| - | - | - |
| Code to payload | Reviewer connects the script, exported file and queued batch through inspection and manual checks. | Rerun the pinned producer and compare the complete normalized transaction to reviewed expectations. |
| Batch integrity | Manual decoding can inspect individual calls. Full coverage depends on the review performed. | A changed target, amount, calldata, call order or omitted call changes the digest. Every generated output must be claimed or explicitly set aside. |
| Safe identity | Operators check the selected Safe, chain, nonce and transaction envelope. | Bind the payload to explicit chain, Safe, nonce and MultiSend settings and verify the resulting Safe hash. |
| Submitting a batch | Selecting and uploading a JSON file creates a separate opportunity to submit the wrong file. | The proposal command submits the transaction just attested, removing file selection and upload from that path. |
| Signer verification | A signer may use the frontend display, the notes and locally run tools to identify the transaction. | Derive the signed hash locally from the reviewed source and config, independently of the frontend display. |
| Review record | Notes and transaction links describe the operation. Repeating the full check requires reviewer effort. | A pinned claim, CI verdict and derived hashes provide repeatable evidence that can accompany the explanation. |

These controls reduce reliance on the person assembling or uploading the batch. A substituted JSON file or a wrong submission can be detected by comparing its full hash with the independently derived expectation. When proposing through Forge Attest, the signature is generated over the verified transaction itself. A frontend cannot change that signed payload without invalidating the signature. [4]

## Where the time savings come from

The expected saving is repeated mechanical verification across batches and signers. CI reproduces all five batches in one run. Each owner can reuse the reviewed claim to check the same payload, and the proposal workflow removes JSON upload and manual transfer of transaction details. Review time can focus on the intended change, authority and protocol effects. [3, 4]

The 69-second result measures one CI run. Manual reviewer time, rollout effort and net savings were not measured. To quantify the benefit in production, record active operator and signer minutes, corrections to submissions, and time to collect approvals for the next comparable operation.

## Verified proposals and signing across the organization

An owner can run Forge Attest with the reviewed claim and a supported wallet, including a hardware wallet. After verification, the first owner proposes the attested transaction to the configured Safe Transaction Service. Later owners run the same command to add confirmations to the same hash. Before signing, the tool checks owner eligibility, nonce use and conflicting queued transactions. It verifies the signature and requires a final live queue comparison. [4]

This eliminates the wrong-file upload error within that workflow and removes the need to trust a frontend to identify the payload being signed. Operators review the claim and confirm the signing request on a trusted device. [4]

## Proposed adoption

Adopt checked-in Forge Attest claims and passing CI for multisig operations across the organization. Keep each operational explanation alongside that evidence. After validating the relevant Transaction Service integrations, have proposing owners and subsequent signers run the same reviewed claims locally. Measure active review effort and submission corrections to establish the operational return.

## Evidence and scope

The comparison uses the public Aurora signer notes on HackMD and the demo repository. Descriptions of manual handoffs are process analysis based on those notes and the workflow described for this case study.

[[1] Frax Aurora OFT signer notes](https://hackmd.io/@dhruvin-frax/teardown-aurora)

[[2] Demo PR](https://github.com/pegahcarter/frax-oft-upgradeable/pull/1)

[[3] Successful five-claim CI run](https://github.com/pegahcarter/frax-oft-upgradeable/actions/runs/41876795192)

[[4] Forge Attest usage and threat model at the demo tool revision](https://github.com/crossecute/forge-attest/blob/81f37c317170759989575a43dacd7d94ea19efbf/README.md)
