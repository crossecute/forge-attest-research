# EVM teams that use Foundry scripts for multisig proposals

Research date: 2026-11-04. Scope: public GitHub repos where a Foundry script (`.s.sol` or a `Script` base contract) builds, simulates, proposes, or executes a Safe (Gnosis Safe) multisig transaction. Every entry below was opened and read; see "Method and limits".

## Method and limits

Candidates came from GitHub code search (`gh search code`) for Safe-specific identifiers: `multisig-transactions`, `execTransaction`, `MultisigScript`, `MultisigBuilder`, `BatchScript`, `SafeHelper`, `safe-utils`, `forge-safe`, `encodeMultiSend`. For each repo I confirmed it is public and not archived, then downloaded the linked file.

How much of each file I read:
- Files under about 400 lines (most project scripts) were read start to finish.
- About a dozen files are large: Base `MultisigScript` (1,052 lines), Optimism `MultisigTask` (1,068), `safe-utils` `Safe.sol` (1,290), Olympus `BatchScriptV2` (875), the ether.fi script (2,754), Union `Deploy.s.sol` (2,912), and the OP Stack `Deploy.s.sol` files in opBNB and Fraxtal. For these I read the header documentation, the full function inventory, and every section that touches Safe signing, proposing, or execution, not each line.
- Dependency information comes from each repo's `.gitmodules`, `foundry.toml`, `remappings.txt`, and `soldeer.lock` where they exist. For Union and Pareto I did not find where `safe-utils` comes from.
- For the comparison I also read the tools that consume script output: the Zeus CLI ([Layr-Labs/zeus](https://github.com/Layr-Labs/zeus), commit `8c9569d`), Base's [eip712sign](https://github.com/base/eip712sign) (commit `15e556f`, all 371 lines of `main.go`), and Base's [task-signing-tool](https://github.com/base/task-signing-tool) (commit `e18a2b0`, README plus the validation, simulation, and signing code paths). I did not read CI around these scripts or the Safe Transaction Service.

Limits: GitHub code search is incomplete. Large orgs I checked (Morpho, Uniswap, Aave/BGD, Lido, Pendle, Compound, Hyperlane, Across, Liquity, Sablier, Ethena, Scroll, Mantle, LayerZero) returned no Safe-proposal scripts in the index. That is not evidence they do not use Foundry for this.

## Style key

The key groups scripts by how they get a transaction in front of signers. The code is used in the master table.

| Code | Style | How it works | Needs network or FFI |
|---|---|---|---|
| S1 | Offline sign, approve, run | The script builds the Safe tx and prints data to sign. Signers sign off-chain. A facilitator runs the script again with the collected signatures (Base `MultisigScript`: `sign`, `verify`, `approve`, `simulate`, `run`). Supports nested Safes and state-override simulation. | No Safe API. Optional `eip712sign` helper or Tenderly for simulation. |
| S2 | Prank and emit | The script runs the actions as the multisig with `vm.startPrank` and emits `ZeusRequireMultisig`. The Zeus CLI runs the script with `forge script --json`, finds that event, takes the successful calls the script contract made from the traces, builds one Safe tx with Safe protocol-kit, signs it, and proposes it to the Safe Transaction Service. Tests run the same code as an EOA. | The Zeus CLI (Node 22, `forge`, a Safe API key). |
| S3 | forge-safe `BatchScript` | `addToBatch(...)` then `executeBatch(send)`. Builds a MultiSend, signs the Safe tx hash (local key, or Ledger through `cast wallet sign` over FFI, selected by `WALLET_TYPE`), and posts it to the Safe Transaction Service with `curl` through the `surl` library. | `--ffi`, `cast`, `curl`, Safe API access. |
| S4 | safe-utils client | `Safe.Client` from Recon-Fuzz `safe-utils`. `proposeTransaction` / `proposeTransactions` sign (private key, Ledger, or Trezor through `cast`) and post to the Transaction Service using the `solidity-http` library. | `--ffi`, `cast`, Safe API access. |
| S5 | Custom client | The team wrote its own Transaction Service or signing code, with no shared Safe library. | Varies. |
| S6 | Output only | The script produces something signers import or paste: a Safe Transaction Builder JSON file, MultiSend calldata, or a batch description. Nothing is posted. | None, or FFI to write files. |
| S7 | Direct or pre-signed execution | The script calls `execTransaction` itself, using owner keys during deployment or signatures collected earlier. No proposal step. | Signers' keys or signatures. |

Add-on features noted in the tables: **TL** (wraps the action in a timelock `schedule` / `scheduleBatch`), **NS** (nested Safes), **Sim** (fork simulation before proposing), **HW** (Ledger or Trezor signing in the script).

## Master table

Dependency column says what the project uses and how it is pulled in: **submodule** (git submodule), **vendored** (file copied into the repo), **soldeer** (pinned in `foundry.toml`), **fork** (submodule pointing at the team's own fork).

### S1: Offline sign, approve, run

| Project | Script | Dependency | Notes |
|---|---|---|---|
| Base | [base/contracts: scripts/universal/MultisigScript.sol](https://github.com/base/contracts/blob/HEAD/scripts/universal/MultisigScript.sol), tasks in [base/contract-deployments](https://github.com/base/contract-deployments) | The framework itself. `contract-deployments` carries a copy of `MultisigScript.sol` inside task folders (code search hits under `archive/`). | NS, Sim. Uses `ICBMulticall`, `Signatures`, `StateDiff`, and `Simulation` helpers. Most active repo found (pushed 2026-11-04). |
| Optimism (superchain-ops) | [src/tasks/MultisigTask.sol](https://github.com/ethereum-optimism/superchain-ops/blob/HEAD/src/tasks/MultisigTask.sol), [src/tasks](https://github.com/ethereum-optimism/superchain-ops/tree/HEAD/src/tasks) | **submodule** `lib/base-contracts`; imports `Signatures`, `Simulation`, `IGnosisSafe` from `@base-contracts/script/universal`. | NS (depth 2), Sim. TOML task configs and templates, storage-write and balance-change allowlists, Tenderly simulation output, Multicall3 `aggregate3Value`, `approveHash` for nested approvals. |
| Optimism (presigner) | [script/CallPause.s.sol](https://github.com/ethereum-optimism/presigner/blob/HEAD/script/CallPause.s.sol) | Remaps `@base-contracts` to `lib/base-contracts`; extends `MultisigBuilder`. | Builds one pre-signed `pause` call. |
| Blast | [SafeBuilder.sol](https://github.com/blast-io/blast/blob/HEAD/blast-optimism/packages/contracts-bedrock/scripts/universal/SafeBuilder.sol) | Inherited from Optimism `contracts-bedrock`. | Approved-hashes method plus Multicall3. Without `--broadcast` it prints a Tenderly URL. Older ancestor of the Base style. |

### S2: Prank and emit (Zeus)

| Project | Script | Dependency | Notes |
|---|---|---|---|
| EigenLayer | [zeus-templates: MultisigBuilder.sol](https://github.com/Layr-Labs/zeus-templates/blob/HEAD/src/templates/MultisigBuilder.sol), [eigenlayer-contracts: 1-updateGenerator.s.sol](https://github.com/Layr-Labs/eigenlayer-contracts/blob/HEAD/script/operations/update-generator/1-updateGenerator.s.sol), [script/releases](https://github.com/Layr-Labs/eigenlayer-contracts/tree/HEAD/script/releases) | **submodule** `lib/zeus-templates` | `_runAsMultisig()` with a `prank(Env.opsMultisig())` modifier, then `execute()`. `testScript()` replays the action and asserts the end state. |
| EigenLayer (eigenx) | [2-acceptAdminAndSetLimits.s.sol](https://github.com/Layr-Labs/eigenx-contracts/blob/HEAD/script/releases/v1.0.4-init/2-acceptAdminAndSetLimits.s.sol) | **submodule** `lib/zeus-templates` | Same pattern, numbered release steps. |
| EigenLayer (EigenHopper) | [2-queueGrantMintingRights.s.sol](https://github.com/Layr-Labs/EigenHopper/blob/HEAD/script/deploy/init-deploy/2-queueGrantMintingRights.s.sol) | Imports Zeus through `eigenlayer-contracts/lib/zeus-templates`. | TL. Queues a timelock `schedule` whose payload is an executor-Safe `execTransaction` wrapping a MultiSend (built with Zeus `Encode`). |

### S3: forge-safe `BatchScript`

| Project | Script | Dependency | Notes |
|---|---|---|---|
| forge-safe (source) | [ind-igo/forge-safe: src/BatchScript.sol](https://github.com/ind-igo/forge-safe/blob/HEAD/src/BatchScript.sol) | Needs `surl` | The pattern. `CHAIN` and `WALLET_TYPE` env vars, Ledger via `cast` over FFI, posts to the Transaction Service. 129 stars, last push 2025-02. |
| Ion Protocol | [script/actions/Pause.sol](https://github.com/paxoslabs/ion-protocol/blob/HEAD/script/actions/Pause.sol) | **submodule** `ind-igo/forge-safe` | Pause and unpause contracts built on the older `executeBatch(SAFE, send)` signature. Last push 2024-07. |
| Infrared | [UpgradeInfrared.s.sol](https://github.com/infrared-dao/contracts/blob/HEAD/script/upgrades/infrared/v1.2/UpgradeInfrared.s.sol) | **fork** `sandybradley/forge-safe` | Proxy upgrade plus initializer in one batch. |
| Twyne | [ParamChangeScript.s.sol](https://github.com/0xTwyne/twyne-contracts-v1/blob/HEAD/script/ParamChangeScript.s.sol) | **fork** `bunbun295/forge-safe` | Sets max liquidation LTV and deploys a new IRM, batched. Has TODO placeholders, so it is a template for the next change. |
| Sense Finance | [PointTokenVault.s.sol](https://github.com/sense-finance/point-tokenization-vault/blob/HEAD/contracts/script/PointTokenVault.s.sol) | **fork** `sense-finance/forge-safe` | Deploy scripts plus `executeBatch(safe, true)` admin actions. |
| Storm Labs (Cove) | [OpsMultisigScript.s.sol](https://github.com/Storm-Labs-Inc/multisig-scripts/blob/HEAD/script/ops/OpsMultisigScript.s.sol) | **fork** `Storm-Labs-Inc/forge-safe` | Abstract base for the ops Safe. The example batch is commented out, so it is scaffolding. |
| eodata | [ExecuteAny.s.sol](https://github.com/eodata/target-contracts/blob/HEAD/script/multisig/ExecuteAny.s.sol) | **fork** `Eoracle/forge-safe` | Generic "send arbitrary calls from the multisig" with a `send` flag. |
| Axis Finance | [script/ops/lib/BatchScript.sol](https://github.com/Axis-Fi/axis-core/blob/HEAD/script/ops/lib/BatchScript.sol), [Batch.s.sol](https://github.com/Axis-Fi/axis-core/blob/HEAD/script/ops/Batch.s.sol) | **vendored** BatchScript, `Surl`, `DelegatePrank` | `Batch.s.sol` wraps it with `CHAIN`, `MS` env and `env.json` address lookups. |
| Superform | [BatchScript.sol](https://github.com/superform-xyz/superform-core/blob/HEAD/script/forge-scripts/safe/BatchScript.sol) | **vendored** BatchScript and `DelegatePrank`; **submodule** `memester-xyz/surl` | Same shape as Axis. |
| Flat Money | [BatchScript.sol](https://github.com/FlatMoney/flat-money-contracts/blob/HEAD/script/utils/BatchScript.sol) | **vendored** BatchScript, `Surl`, `DelegatePrank` | Last push 2025-07. |
| Yearn (stb) | [scripts/lib/BatchScript.sol](https://github.com/yearn/yearn-stb/blob/HEAD/scripts/lib/BatchScript.sol) | **vendored** BatchScript, `Surl`, `DelegatePrank` | |
| Yearn (yb) | [script/utils/SafeHelper.sol](https://github.com/yearn/yearn-yb/blob/HEAD/script/utils/SafeHelper.sol) | **vendored** SafeHelper; **submodule** `surl` | Variant file with a different name and a `Test` import, no `DelegatePrank`. Same `SAFE_API_*` constants. |
| Resupply | [script/utils/SafeHelper.sol](https://github.com/resupplyfi/resupply/blob/HEAD/script/utils/SafeHelper.sol) | **vendored** SafeHelper; **submodule** `surl` | Same variant as Yearn yb. |
| 3Jane | [script/utils/SafeHelper.sol](https://github.com/3jane-protocol/moneymarket-contracts/blob/HEAD/script/utils/SafeHelper.sol) | **vendored** SafeHelper and `Surl` | Same variant. Uses the `api.safe.global/tx-service` URL. |
| Octant | [script/helpers/BatchScript.sol](https://github.com/golemfoundation/octant-v2-core/blob/HEAD/script/helpers/BatchScript.sol) | **vendored** BatchScript; **soldeer** `surl` pinned to a commit | Comments the Safe API migration. |
| YieldNest (cross-chain) | [script/BatchScript.s.sol](https://github.com/yieldnest/yieldnest-cross-chain/blob/HEAD/script/BatchScript.s.sol) | **vendored** BatchScript; imports `surl` | HW, Sim. |
| Pike Markets | [script/SafeScript.sol](https://github.com/nutsfinance/pike-local-markets/blob/HEAD/script/SafeScript.sol) | **vendored** SafeScript; **submodule** `surl` | forge-safe derived. |

### S4: safe-utils client

| Project | Script | Dependency | Notes |
|---|---|---|---|
| safe-utils (source) | [Recon-Fuzz/safe-utils: src/Safe.sol](https://github.com/Recon-Fuzz/safe-utils/blob/HEAD/src/Safe.sol), [SafeScriptBase.sol](https://github.com/Recon-Fuzz/safe-utils/blob/HEAD/src/SafeScriptBase.sol) | Needs `solidity-http`, `safe-smart-account` | `Safe.Client`, `proposeTransaction(s)`, `simulateTransaction*`, `getSafeTxHash(es)`, `validateProposedTransaction*`, approve-hash helpers. Ledger or Trezor selected with `HARDWARE_WALLET`. Targets `api.safe.global/tx-service`. |
| Olympus DAO | [BatchScriptV2.sol](https://github.com/OlympusDAO/olympus-v3/blob/HEAD/src/scripts/ops/lib/BatchScriptV2.sol), [src/scripts/ops](https://github.com/OlympusDAO/olympus-v3/tree/HEAD/src/scripts/ops) | **soldeer** `safe-utils` 0.0.23, `solidity-http` 0.0.7, `surl` 1.0.0 | HW, Sim. One script can propose to a Safe, send from an EOA, or run on an Anvil or Tenderly fork. Reads batch args from a file. Adds post-batch validation hooks. |
| M0 | [m-portal: MultiSigBatchBase.sol](https://github.com/m0-platform/m-portal/blob/HEAD/script/MultiSigBatchBase.sol), [common: SafeNonce.sol](https://github.com/m0-platform/common/blob/HEAD/script/SafeNonce.sol), [evm-m-extensions: ProposeTimelockUpgradeBase.sol](https://github.com/m0-platform/evm-m-extensions/blob/HEAD/script/upgrade/ProposeTimelockUpgradeBase.sol) | **submodule** `Recon-Fuzz/safe-utils` in m-portal; **fork** `m0-platform/safe-utils` in `common`; `evm-m-extensions` gets it through `lib/common` | TL, Sim. Batch base with `vm.prank` simulation, a next-free-nonce helper that reads pending proposals, and a timelock `scheduleBatch` proposal. Reused by downstream forks (Seismic, 1Money). |
| Telcoin | [tel-v3: UnpauseMigration.s.sol](https://github.com/Telcoin-Association/tel-v3/blob/HEAD/script/mainnet/write/UnpauseMigration.s.sol), [forge-deploy-utils: DeployBase.sol](https://github.com/Telcoin-Association/forge-deploy-utils/blob/HEAD/src/DeployBase.sol) | **submodule** `Recon-Fuzz/safe-utils` in `forge-deploy-utils`; `tel-v3` reaches it through that library | Sim. `DeployBase` extends `SafeScriptBase`, so every deployment and admin action is a Safe proposal. `SAFE_NONCE_OFFSET` queues behind pending txs. |
| Centrifuge | [script/utils/GateProposal.s.sol](https://github.com/centrifuge/protocol/blob/HEAD/script/utils/GateProposal.s.sol) | **submodule** `Recon-Fuzz/safe-utils` (plus `create3-gate`) | HW. Gets the Ledger address with `cast wallet address --ledger` over FFI, then proposes one plain call into the CREATE3 gate. |
| Aave Labs (v4 risk stewards) | [RiskStewardsBase.s.sol](https://github.com/aave-dao/aave-v4-risk-stewards/blob/HEAD/scripts/RiskStewardsBase.s.sol), [RiskStewardsArc.s.sol](https://github.com/aave-dao/aave-v4-risk-stewards/blob/HEAD/scripts/networks/RiskStewardsArc.s.sol) | **submodule** `Recon-Fuzz/safe-utils`; `aave-helpers` | Sim. `run(broadcastToSafe, generateDiffReport, skipTimelock)` simulates, prints calldata, can write a diff report, and can propose one MultiSend batch. On chains with no registered MultiSend (Arc) it only prints calldata. Small repo; not Aave's main governance flow. |
| Espresso Systems | [MultiSigTransfer.s.sol](https://github.com/EspressoSystems/espresso-tee-contracts/blob/HEAD/scripts/MultiSigTransfer.s.sol) | **submodule** `Recon-Fuzz/safe-utils` | HW. Starts a 2-step ownership transfer from an EOA, then proposes `acceptOwnership` to the multisig. |
| Rheo | [ProposeSafeTxSetTotalAssetsCap.s.sol](https://github.com/rheo-xyz/very-liquid-vaults/blob/HEAD/script/ProposeSafeTxSetTotalAssetsCap.s.sol) | **submodule** `safe-utils` (`@safe-utils` remap) | TL, HW. One script per proposal. Wraps each vault change in an OpenZeppelin `TimelockController.schedule`, proposed by the governance Safe. |
| Pareto | [USP: script/ManagerSpells.s.sol](https://github.com/pareto-credit/USP/blob/HEAD/script/ManagerSpells.s.sol) | `safe-utils`; source not found in the files I checked | HW. `IS_EOA` flag switches between sending from an EOA and proposing to the timelock multisig. |
| Union | [evm/scripts/Deploy.s.sol](https://github.com/unionlabs/union/blob/HEAD/evm/scripts/Deploy.s.sol) | `@safe-utils/Safe.sol`; source not found in the files I checked | `useSafe` flag. In the upgrade flow it proposes `upgradeToAndCall` to the Safe; otherwise it sends directly. Dry-run mode. |
| treb-sol (Trebuchet) | [GnosisSafeSender.sol](https://github.com/trebuchet-org/treb-sol/blob/HEAD/src/internal/sender/GnosisSafeSender.sol) | **fork** `trebuchet-org/safe-utils` | HW. Queues a script's calls and proposes them as a batch. Proposer can be a private key, Ledger, or Trezor. A tool for other teams. |

### S5: Custom client

| Project | Script | Dependency | Notes |
|---|---|---|---|
| Euler | [evk-periphery: script/utils/SafeUtils.s.sol](https://github.com/euler-xyz/evk-periphery/blob/HEAD/script/utils/SafeUtils.s.sol) | None shared. Own `Surl.sol` copy; comments say "inspired by forge-safe". | Sim. Reads status, owners, delegates, pending txs, and next nonce from the Transaction Service. Writes a Safe tx JSON and a Transaction Builder JSON per tx, and builds MultiSend batches. It does not post a proposal (also fits S6). |
| Lend | [script/common/LendMultisig.s.sol](https://github.com/lendxyz/lend-contracts/blob/HEAD/script/common/LendMultisig.s.sol) | None. Single library using FFI to `cast` and `curl`. | HW. Signs EIP-712 typed data with `cast wallet sign --ledger`, takes the owner from `--sender`, finds the next nonce, and posts to the service. Nothing is signed or proposed without `--broadcast`. |
| pkxp-forge-safe | [src/SafeScript.s.sol](https://github.com/panukettu/pkxp-forge-safe/blob/HEAD/src/SafeScript.s.sol) | Bun TypeScript helper (`ts/ffi.ts`) | Sim. `simulateAndSign`, `signBatch`, `proposeBatch`, `sendBatch`. Personal tool, 2 stars, last push 2024-09. |
| Axie / Ronin | [foundry-deployment-kit: script/utils/Helpers.sol](https://github.com/axieinfinity/foundry-deployment-kit/blob/HEAD/script/utils/Helpers.sol) | None shared | Sim. Safe tx hash and nonce helpers; reads the Transaction Service for pending nonces. Does not propose. |

### S6: Output only

| Project | Script | Dependency | Notes |
|---|---|---|---|
| ether.fi | [script/upgrades/security-upgrades/transactions.s.sol](https://github.com/etherfi-protocol/smart-contracts/blob/HEAD/script/upgrades/security-upgrades/transactions.s.sol) | Own `writeSafeJson` in `@scripts/utils` | TL, Sim. Builds ordered batches for the operation multisig and two timelocks (10-day upgrade, 2-day ops), writes six Safe JSON files, and runs many post-upgrade checks. Uses FFI to confirm the git commit is clean. |
| Royco | [ParameterUpdateBase.sol](https://github.com/roycoprotocol/royco-dawn/blob/HEAD/script/update/base/ParameterUpdateBase.sol) | Own JSON writer; OpenZeppelin `AccessManager` | TL, Sim. Forks each chain, simulates schedule, warp, then execute, and writes schedule, execute, and cancel Safe JSON files per chain. |
| Boundless | [contracts/scripts/Manage.Router.s.sol](https://github.com/boundless-xyz/boundless/blob/HEAD/contracts/scripts/Manage.Router.s.sol) | Own JSON writer through FFI | `GNOSIS_EXECUTE=true` queues admin calls and writes a Safe Transaction Builder JSON for the multisig to import. |
| EthStrat | [script/lib/SafeBatchLib.sol](https://github.com/ethstrat/contracts/blob/HEAD/script/lib/SafeBatchLib.sol) | Own library | Writes Transaction Builder JSON with decoded method names and arguments, and re-derives calldata to check the decoded form matches. |
| Signals | [PrepareSafeUpgrade.s.sol](https://github.com/signals-protocol/v1-contract/blob/HEAD/script/actions/PrepareSafeUpgrade.s.sol) | None | Deploys new implementations, writes MultiSend calldata files and a plan JSON, and prints steps to paste into the Safe UI. |
| Agglayer (Polygon) | [ObsoleteRollupType.s.sol](https://github.com/agglayer/agglayer-contracts/blob/HEAD/script/forge/obsolete-rollup-type/ObsoleteRollupType.s.sol) | None | TL. Reads a JSON input and generates calldata for either a multisig or a timelock (`scheduleBatch` / `executeBatch`). Not Safe-specific. |
| YieldNest (kernel) | [script/BatchScript.sol](https://github.com/yieldnest/yieldnest-kernel-lrt/blob/HEAD/script/BatchScript.sol) | **vendored**, stripped copy of forge-safe | Sim. Keeps `addToBatch` (with a `vm.prank` simulation) and `getBatch()` (MultiSend data). No signing or posting. |
| Arcadia (Pragma Labs) | [safe-signer: SafeTransactionBuilder.sol](https://github.com/arcadia-finance/safe-signer/blob/HEAD/transaction_builder/script/utils/SafeTransactionBuilder.sol) | Based on forge-safe (rewritten) | Encodes `multiSend(bytes)` data per Safe. |
| Alien Finance | [multisig-tx-builder: Example.s.sol](https://github.com/AlienFinance/multisig-tx-builder/blob/HEAD/script/Example.s.sol) | Imports `forge-safe/BatchScript.sol` | Example batching an ERC-20 transfer. 0 stars, last push 2024-03. |

### S7: Direct or pre-signed execution

| Project | Script | Dependency | Notes |
|---|---|---|---|
| Pantos | [SubmitSafeTxs.s.sol](https://github.com/pantos-io/ethereum-contracts/blob/HEAD/script/SubmitSafeTxs.s.sol) | `@safe` contracts | Reads `<chain>_flat_output.json` of already-signed SafeTxs and calls `execTransaction` for each. Last push 2025-02. |
| Celo | [SafeSetGames.s.sol](https://github.com/celo-org/celo-monorepo/blob/HEAD/packages/op-tooling/impls/SafeSetGames.s.sol) | `safe-contracts`, Multicall3 | Computes the Safe tx hash for a Multicall3 delegatecall. Runs `execTransaction` if `SIG` signatures are provided; otherwise it only prints the hash. |
| opBNB | [packages/contracts-bedrock/scripts/Deploy.s.sol](https://github.com/bnb-chain/opbnb/blob/HEAD/packages/contracts-bedrock/scripts/Deploy.s.sol) | Inherited from Optimism | `_callViaSafe` runs `Safe.execTransaction` during deployment and config. |
| Fraxtal | [Deploy.s.sol](https://github.com/FraxFinance/fraxtal-contracts/blob/HEAD/lib/optimism/packages/contracts-bedrock/scripts/Deploy.s.sol) | Vendored Optimism lib | Same `_callViaSafe` pattern. |
| HOPR | [SingleAction.s.sol](https://github.com/hoprnet/contracts/blob/HEAD/ethereum/contracts/script/SingleAction.s.sol) | `SafeSuiteLibV141` in repo | Calls `execTransaction` on node-operator Safes (module setup, channel approvals). Operational, not a governance multisig. |
| 0x (Settler) | [script/DeploySafes.s.sol](https://github.com/0xProject/0x-settler/blob/HEAD/script/DeploySafes.s.sol) | Own Safe and MultiSend interfaces | Deploys Safes and configures them (owners, threshold, modules, guard) with direct `execTransaction` calls. |
| Gardens (1Hive) | [pkg/contracts/script/AddOwner.s.sol](https://github.com/1Hive/gardens-v2/blob/HEAD/pkg/contracts/script/AddOwner.s.sol) | Test helper `SafeSetup` | Signs and executes `addOwnerWithThreshold` on a threshold-1 council Safe with one member's key. |

## Dependency index

Which shared code each project pulls in, and how.

| Shared code | Role | Used by |
|---|---|---|
| `ind-igo/forge-safe` (+ `memester-xyz/surl`) | S3 `BatchScript` | Direct submodule: Ion. Forks: Infrared (`sandybradley`), Twyne (`bunbun295`), Sense, Storm Labs, eodata (`Eoracle`). Vendored copies: Axis, Superform, Flat Money, Yearn stb, Yearn yb, Resupply, 3Jane, Octant, YieldNest cross-chain, Pike. Stripped copy: YieldNest kernel. Imported by name: Alien Finance. Rewritten from it: Euler, Arcadia. |
| `Recon-Fuzz/safe-utils` (+ `solidity-http`, `safe-smart-account`) | S4 client | Submodule: M0 m-portal, Espresso, Centrifuge, Aave v4 stewards, Rheo, Telcoin `forge-deploy-utils`. Forks: `m0-platform/safe-utils`, `trebuchet-org/safe-utils`. Soldeer pin: Olympus. Through another library: Telcoin `tel-v3` (`forge-deploy-utils`), M0 `evm-m-extensions` (`common`). Source not found: Union, Pareto. |
| Base `MultisigScript` / `MultisigBuilder` | S1 framework | Base `contract-deployments` (copies), Optimism `superchain-ops` and `presigner` (submodule `lib/base-contracts`). Blast `SafeBuilder` is an older Optimism ancestor. |
| `Layr-Labs/zeus-templates` | S2 framework | `eigenlayer-contracts`, `eigenx-contracts` (submodules), `EigenHopper` (through `eigenlayer-contracts`). |
| [`base/eip712sign`](https://github.com/base/eip712sign) | Signer binary for S1 | Base `contract-deployments` (installed at v0.0.16 by `make deps`), Optimism `superchain-ops` (installed at v0.0.11 by `just install`). Base also uses [`base/task-signing-tool`](https://github.com/base/task-signing-tool), cloned into the task repo. |
| [Zeus CLI](https://github.com/Layr-Labs/zeus) (+ Safe protocol-kit, api-kit) | Consumer of S2 scripts | EigenLayer repos that register upgrades with Zeus. |
| OpenZeppelin `TimelockController` / `AccessManager` | Timelock layer | Rheo, M0, ether.fi, Royco, EigenHopper, Agglayer. |
| Multicall3 or Safe MultiSend | Batching | Base and Optimism use Multicall3 variants. S3 and S4 use Safe `MultiSend` / `MultiSendCallOnly`. |
| Nothing shared | S5 to S7 | Euler, Lend, pkxp-forge-safe, Axie, ether.fi, Royco, Boundless, EthStrat, Signals, Agglayer, Pantos, Celo, HOPR, 0x. |

Observations from the table:
- Two families account for most of the S3 and S4 entries. The forge-safe family is mostly copied or forked into each repo, so fixes upstream do not reach them. The safe-utils family is mostly a submodule or a pinned version.
- Olympus uses safe-utils (`BatchScriptV2`). Axis, which has a forge-safe style copy, does not.
- The S1 and S2 frameworks come from chain and protocol teams (Base, Optimism, EigenLayer). The S3 and S4 libraries show up in application repos.

## Comparison of the five reusable options

These are the shared tools other teams build on: [Base `MultisigScript`](https://github.com/base/contracts/blob/HEAD/scripts/universal/MultisigScript.sol), [Optimism `MultisigTask`](https://github.com/ethereum-optimism/superchain-ops/blob/HEAD/src/tasks/MultisigTask.sol), [Zeus `MultisigBuilder`](https://github.com/Layr-Labs/zeus-templates/blob/HEAD/src/templates/MultisigBuilder.sol), [`forge-safe`](https://github.com/ind-igo/forge-safe/blob/HEAD/src/BatchScript.sol), and [`safe-utils`](https://github.com/Recon-Fuzz/safe-utils/blob/HEAD/src/Safe.sol). Facts come from the source files, the repo READMEs, and the GitHub API on 2026-11-04.

| | Base `MultisigScript` | Optimism `MultisigTask` | Zeus `MultisigBuilder` | `forge-safe` | `safe-utils` |
|---|---|---|---|---|---|
| Style | S1 | S1 | S2 | S3 | S4 |
| Who runs it | A facilitator runs `sign`, then `approve` (nested Safes), then `run` with the collected signatures. | Same split: `simulate`, `approve`, `execute`, driven by `just` commands. | One proposer runs `zeus deploy run`. The CLI keeps a phase state machine (`multisig_start`, `multisig_wait_signers`, `multisig_execute`, `multisig_wait_confirm`) and the deploy is resumed with `--resume` between phases. | One proposer runs `executeBatch(true)` once. | One proposer runs `proposeTransaction(s)` once. |
| Signing | Signers sign printed EIP-712 data off-chain with Base's `eip712sign` (Ledger, Trezor, mnemonic, or private key), driven by the `task-signing-tool` web app. The facilitator passes signatures as a hex argument and runs `approve` / `run` with `forge script --ledger`. | Same `eip712sign` binary (installed at v0.0.11 by the repo's `justfile`). `just sign` uses a Ledger by default or a Foundry keystore (`USE_KEYSTORE`), and checks that the signing address is a current Safe owner unless `SKIP_SIGNER_OWNER_CHECK` is set. | In the Zeus CLI, per strategy: Ledger (viem, EIP-712 typed data; blind signing must be on), private key (viem), or a local web page (MetaMask, Ledger, GridPlus). Private-key signing prompts for the key, and the CLI redacts it from logged output. | `cast wallet sign` through a `bash -c` FFI call. Wallet is `LOCAL` or `LEDGER`. For `LOCAL` the private key is placed in the command string passed to bash. | `cast wallet sign` over FFI for Ledger or Trezor (`HARDWARE_WALLET`, default Ledger). With no derivation path it signs with `vm.sign` for a key Foundry already holds (for example from `--private-key` or a keystore). |
| Where it ends | Signatures go to an on-chain `execTransaction` sent by the facilitator. No Safe API. | Same. | POSTs to the Safe Transaction Service through `@safe-global/api-kit` (API key required). The CLI then polls confirmations and execution, records the result in the metadata repo, and can propose a same-nonce rejection tx to cancel. Other strategies execute on-chain: an `execTransaction` with a pre-approved signature, described in the code as for 1-of-N Safes only. | POSTs to the Safe Transaction Service. Other owners then confirm in the Safe UI. | Same. Uses `api.safe.global/tx-service`. |
| Simulation | Fork simulation with state overrides for threshold, nonce, and approvals; `_postCheck`, `_postSign`, `_postApprove`, `_postRun` hooks; can print a Tenderly link. | Fork simulation plus a state-diff check against allowlists of expected storage writes and balance changes; Tenderly data; stacked simulation of other pending tasks. | The CLI runs `zeus test` on the script before the multisig step and halts the deploy if any `test*` function fails. The consumer repos' `testScript()` runs the same code as an EOA and asserts the result. | Each `addToBatch` runs the call with `vm.prank(safe)` so a revert shows up early. The whole-batch simulation call is commented out. | `simulateTransaction*` functions, plus `validateProposedTransaction*` to compare a posted tx with expected values. `SafeScriptBase` switches between simulate-only and propose depending on whether `--broadcast` is set. |
| Nonce | Safe's on-chain nonce. Override with `SAFE_NONCE` or `SAFE_NONCE_<ADDRESS>`. | Set in the task's TOML `[stateOverrides]`. The README says choosing the nonce needs care. | Zeus does not set one. It takes whatever Safe protocol-kit assigns in `createTransaction`; I did not trace that library. | Latest transaction in the Safe API plus one, so a new proposal queues behind pending ones. | On-chain nonce only. M0 and Telcoin each added their own handling (a pending-proposals helper, a `SAFE_NONCE_OFFSET` variable). |
| Nested Safes | Yes, any depth. | Yes, up to depth 2 in the README examples. | No. One Safe per script, taken from the event. | No. | No nested flow found. |
| Timelock | No. Teams add their own. | No. Teams add their own. | README mentions an `OpsTimelockBuilder`, but at HEAD the templates folder holds only `EOADeployer` and `MultisigBuilder`. EigenHopper builds the timelock call by hand. | No. | No. M0 and Rheo wrap it in their own scripts. |
| Batching | `ICBMulticall` and Multicall3-style calls. | Multicall3 (`aggregate3Value`, with a delegatecall variant). | Safe protocol-kit builds a batch from the calls the script made; the event sets Call or DelegateCall. Zeus's own `MultisigBuilder` doc says it uses `MultiSendCallOnly`. | Safe `MultiSend` at a fixed address, as a delegatecall. | `MultiSendCallOnly`, with address tables for v1.3.0 and v1.4.1 and zkSync variants. |
| Chains | Any chain with a Safe and the multicall contract. | Any chain, per task config. | Any. | Five chain IDs hard-coded in the file (the README lists three). | Large chain-ID table for the Safe service. |
| What you must run | `forge`, `eip712sign` (Go), the `task-signing-tool` Next.js app, and (for the Base repo) `mise` and Bun. Task-origin signing also needs `ottr-cli`. | `forge`, `mise`, `just`, `eip712sign`, a Tenderly environment variable. | `forge`, Node 22, the Zeus CLI, a GitHub metadata repo with the Zeus Deployer app installed, `zeus login`, and a Safe API key. | `forge --ffi`, `bash`, `cast`, `curl` (through `surl`). | `forge --ffi`, `cast`, and `solidity-http`. |
| License | MIT | MIT | BUSL-1.1 (templates and CLI both have a LICENSE file with the Business Source License text) | MIT in the source header; no LICENSE file in the repo | MIT in the source header; no LICENSE file in the repo |
| Activity | 330 stars, 9 releases, latest v8.4.0 on 2026-10-16. Signer tools: `eip712sign` 102 stars, pushed 2026-10-23; `task-signing-tool` pushed 2026-11-03. | 99 stars, no releases, last push 2026-11-04. | Templates: 2 stars, last push 2025-09-19. CLI: 10 stars, last commit on the default branch 2025-11-07, last push 2026-01-27, published on npm, has unit tests and a docs site. | 129 stars, 34 forks, last commit 2024-12-10, 12 open issues. | 20 stars, 24 releases, latest v0.0.24 on 2026-11-04. Has a test folder with differential and validation tests. |

### What the comparison shows

- **Three trust models.** Base and Optimism never send anything to a Safe service: signers check data from a local simulation and a facilitator submits the signatures. `forge-safe`, `safe-utils`, and the Zeus CLI post a signed proposal, and the other owners confirm it in the Safe UI, so they depend on the Safe API being reachable. Zeus differs from the other two in that the script itself signs nothing: the CLI reads the script's calls out of the forge traces and builds the Safe tx.
- **Nonce handling is the main place S3 and S4 differ.** `forge-safe` queues behind pending proposals by reading the API. `safe-utils` uses the on-chain nonce, and two teams built their own workaround. Anyone proposing more than one transaction before the first executes needs to plan for this.
- **Base builds the review step into its tooling.** The `task-signing-tool` re-runs the script, compares the result with a separate validation file, and only then signs (see the next section). Optimism's `MultisigTask` does a similar check in Solidity against allowlists. The S3 and S4 libraries and the Zeus CLI simulate or test but leave state checks to each project (for example Aave Labs's diff report, or `testScript()` assertions in EigenLayer's repos).
- **Maintenance differs.** `safe-utils` and Optimism had commits on 2026-11-04 and Base on 2026-10-16. `forge-safe` has no commit since 2024-12-10, and many teams carry their own copy or fork. Zeus Templates has had no push since 2025-09-19, the Zeus CLI none since 2026-01-27, and both are BUSL-1.1.
- **Not covered here.** I did not read the Safe Transaction Service, Safe protocol-kit (which decides Zeus's nonce), or the `ottr-cli` used for Base's task-origin signatures.

### What happens between the script and the signature

**Base (`task-signing-tool` and `eip712sign`).** Signers do not sign what the script prints directly. The flow I read in the code:
1. A task carries a per-signer validation JSON with the forge command to run, the RPC URL, the expected domain hash and message hash, the expected state overrides, the expected storage changes (key, before, after), and optional balance changes.
2. The web app (run locally) executes that forge command with `RECORD_STATE_DIFF=true`, reads the `stateDiff.json` the script writes, and compares simulated hashes and state changes with the expected ones. Mismatches are shown to the signer.
3. Task-origin validation is on by default: three X.509 signatures (task creator, Base facilitator, Security Council facilitator) over a deterministic tarball of the task's config folder, checked with `ottr-cli`. A task can opt out with `skipTaskOriginValidation`.
4. For a Ledger, the app calls `eip712sign --ledger --data 0x1901<domainHash><messageHash>`, so the device signs two hashes and cannot show the transaction fields. The check that those hashes match what was reviewed happens in the app. For a Trezor, the app builds the full `SafeTx` typed data, refuses to sign unless it hashes to the validated hashes, and the device shows each field.
5. `eip712sign` itself does no review. It runs the command (or reads stdin), takes the hex or typed-data JSON between the markers `vvvvvvvv` and `^^^^^^^^`, prints the domain and message hashes, and signs. For `forge script` commands it adds `--sender` automatically, and it retries once if a Ledger wallet closes. Only one signer type is allowed per run.

**Zeus CLI.** After `zeus test` passes, the multisig phase runs the script, builds the Safe tx, shows the calls in a table, asks the signer to confirm the typed data (`checkShouldSignGnosisMessage`), and signs. For a Ledger it first checks that the device address is an owner of the Safe. There is no separate expected-state file: the safety checks are the script's own `test*` functions. The CLI also keeps a record of the commit each upgrade was registered at, so every party runs the same code.

**Optimism.** `just sign` runs the task simulation, prints the data to sign, and signs it through `eip712sign` or a Foundry keystore, after checking the signer is a current Safe owner. The state-diff checks are in `MultisigTask`, not in a separate app.

## Adjacent repos that are not proposal scripts

| Repo | What it is |
|---|---|
| [KintoXYZ/kinto-core: script/actions/multisig.s.sol](https://github.com/KintoXYZ/kinto-core/blob/HEAD/script/actions/multisig.s.sol) | Not Safe. Sends a Kinto Wallet (ERC-4337) user operation signed by two private keys read from a JSON file. |
| [blockful/dao-proposals: src/ens/helpers/SafeHelper.sol](https://github.com/blockful/dao-proposals/blob/HEAD/src/ens/helpers/SafeHelper.sol) | Builds `execTransaction` calldata with a pre-approved signature so the ENS Timelock can run actions on the ENS Endowment Safe from a DAO proposal. A DAO-proposal helper, not a multisig proposal flow. |
| [Hypernative-Labs/hypernative-guard: script/GenerateSafeHash.s.sol](https://github.com/Hypernative-Labs/hypernative-guard/blob/HEAD/script/GenerateSafeHash.s.sol) | Demo that signs sample Safe txs against a mainnet fork to show the guard. |
| [CumpsD/stflip-contracts: script/ops/AcceptGov.s.sol](https://github.com/CumpsD/stflip-contracts/blob/HEAD/script/ops/AcceptGov.s.sol) | Entire file is commented out (it used forge-safe). |
| [lidofinance/clutch: src/exec/SafeExec.sol](https://github.com/lidofinance/clutch/blob/HEAD/src/exec/SafeExec.sol) | Library for a dry run with a mock Aragon voting contract. |

Mirrors and forks with no independent usage (for example `anxolin/evk-periphery`, `dumebi042/Yearnfinance`, `VULCAN2019/lagoon-v0`, `Secure3Audit/code_Mantle_V2_Public`, audit-contest copies) and repos where "Safe" matched ERC-1155 `safeBatchTransferFrom` were left out.

## Follow-ups worth doing

- Read Safe protocol-kit's nonce logic (it decides Zeus's nonce) and `ottr-cli` (Base's task-origin signatures).
- Find where Union and Pareto get `safe-utils`, and check whether the S3 copies track upstream fixes.
- Search for large DAOs (Morpho, Aave, Lido, Uniswap) whose scripts may sit outside the GitHub index.
