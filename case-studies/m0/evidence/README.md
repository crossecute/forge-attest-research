# M0 nonce 22 evidence

Collected on 2026-11-05. All addresses, chain records and the producer's test key
are public.

| File | Origin and meaning |
| --- | --- |
| `attestation.json` | Fresh-clone Forge Attest `6ba05810e4252c87805af1c518fab6e228ea96f8` output using `../attest-nonce22.toml`, `--require-live --json`, and an Ethereum archive RPC. Exit code 0. |
| `reproduced-safe-tx.json` | The native `safe-utils` request from the unchanged producer's curl POST. Its SHA-256 matches `expected_output_sha256` and the native capture verdict. |
| `canonical-safe-tx.json` | The request normalized by the existing `safe-service`/`safe-utils` reader. Its digest matches `expected_canonical_sha256`. |
| `execution-receipt.json` | Historical receipt for Ethereum transaction `0x0e510ee28b99bd2e99ca5e6e61145181fe71964a3bece9723e3417139b91f785`. Status is `0x1`; the Safe success log contains the expected hash. |
| `fork-block.json` | Ethereum block 23834855 from RPC, including its block hash. |
| `ci-run.json` | Successful native-capture M0 run 42100053825. The inherited M0 integration, gas and coverage workflows are separate. |

The claim runs original upstream producer revision
`9e194fa9410edaaa467a324a55fa07af0e65f5a8` directly. No M0 wrapper, capture helper,
JSON writer or modified Solidity source is used. The pinned verifier stores the
captured request outside the producer checkout. Its supported curl interceptor
returns HTTP 201 locally and never forwards the request.

Public test private key `1` satisfies the original script's local signing step.
The captured request includes its deterministic signature and sender, which are
metadata outside the canonical Safe transaction. It is not an owner approval.
The existing reader checks the carried `contractTransactionHash` against the
independently derived hash. No proposal was submitted.

The [Forge Attest feature CI](https://github.com/crossecute/forge-attest/actions/runs/42099920207)
passes the Solidity and full shell suites, including 37 native-capture checks.
Those cover missing, duplicate, unclaimed and unsupported requests, ignored
transport failures, carried-hash tampering, multiple proposals, RPC/claim chain
mismatch, invalid manifests, both configuration layouts, and logical output
names that coincide with source paths. The native M0 run independently verifies
raw and canonical digests, cast/Solidity hashes and the historical service record.

The original per-call simulation executes during attestation. The demo does not
add full Safe execution with historical signatures or custom post-state
assertions. Capture is not a network sandbox; the reviewed bare-curl path is the
supported transport.
