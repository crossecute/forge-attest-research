# M0 nonce 22 evidence

Collected on 2026-11-05. All addresses, chain records and the producer's test key
are public.

| File | Origin and meaning |
| --- | --- |
| `attestation.json` | Fresh-clone Forge Attest `998d11c9325990a74c534ef406a968c509d1544a` output using `../attest-nonce22.toml`, `--require-live --json`, and an Ethereum archive RPC. Exit code 0. |
| `rejected-claim.json` | The same claim with only the canonical and Safe hash expectations replaced by zero hashes. Exit code 1. |
| `reproduced-safe-tx.json` | The actual `safe-utils` request captured from the unchanged producer's `curl -d` argument. Its SHA-256 matches `expected_output_sha256`. |
| `canonical-safe-tx.json` | The captured request normalized with `--format safe-service --profile safe-utils --safe 0xdcf79C332cB3Fe9d39A830a5f8de7cE6b1BD6fD1 --chain-id 1 --nonce 22`. Its digest matches `expected_canonical_sha256`. |
| `execution-receipt.json` | Historical RPC receipt for Ethereum transaction `0x0e510ee28b99bd2e99ca5e6e61145181fe71964a3bece9723e3417139b91f785`. Status is `0x1`; the Safe success log contains the expected hash. |
| `fork-block.json` | Ethereum block 23834855 from RPC, including its block hash. |
| `ci-run.json` | GitHub metadata for the dedicated unchanged-producer attestation workflow. The inherited M0 integration, gas and coverage workflows are separate. |

The claim pins producer `f712d012dabe705940ba473f754e8e8d32f499db`. Its proposal
script and Foundry configuration are byte-for-byte identical to upstream
`9e194fa9410edaaa467a324a55fa07af0e65f5a8`. The adapter only selects a local curl
capture helper and invokes the original script. The helper writes the request
body unchanged, returns HTTP 201 locally and never calls the real curl binary.

The public key material used by the producer is test private key `1`. The
captured request includes its deterministic local signature and sender. It is
not an owner approval and no proposal is submitted. Those two metadata fields
are not part of the canonical Safe transaction. The existing reader also checks
that the producer's `contractTransactionHash` agrees with the independent hash.

Local checks cover byte-preserving request capture, rejection of a missing body,
Solidity formatting/lint, shell syntax, and equality of the original script and
configuration. Lint reports one advisory note for setting the child process
`PATH`. The original per-call simulation executes during attestation. The
reduced demo has no additional full Safe execution test, historical approval
fixture or custom post-state assertions.
