# Provider research

What we have found out about transports we do not control and about standards that are still drafts, and what each finding suggests for the protocol's design. None of it is a requirement.

**Check how stale this is before relying on it.** This file and the provider spec were one document. They were split because they go out of date for different reasons. The spec changes when the protocol changes. Everything here goes stale when somebody else ships a release: it makes version-pinned claims about other teams' contracts, verified by reading their source on a particular day and true only until they redeploy.

Nothing in this file is normative. Where a finding produced an obligation, the obligation lives in the spec and is cited from here by id. Ids such as P7, R3.5 and C29 refer to [`provider-spec.md`](https://github.com/crossecute/protocol/blob/main/docs/provider-spec.md) in `crossecute/protocol`: P1 to P15 are the provider prerequisites, R the rules a binding follows, C the compliance tests. This file was last aligned with that spec at protocol commit `1a223df` (2026-11-04). Terms used without definition (conduit, transceiver, bootstrap, path A and path B) are the spec's.

## Design influences and potential additions

| Finding | Influence on the design | Status |
| --- | --- | --- |
| Eight of ten transports dedupe a successful message; the two raw signature primitives (Wormhole Core, Avalanche Warp) do not ([§1](#1-what-each-transport-guarantees-about-replay)) | Replay protection is the transport's job; a binding on a primitive that lacks it supplies its own, per receiving conduit (P7, R3.5, R3.6, C29 to C31) | In the spec |
| Transports differ on whether a failed message can be retried and with what gas (§1) | A failure must survive as a retry, so a binding never swallows one (R3.7); gas a sender names is not a floor for the retry (R1.5) | In the spec |
| Idempotency exists at the transport for every candidate except the raw primitives (§1) | No protocol-level request id or per-message nonce | Settled |
| Canonical rollup bridges have the chain's own trust model but a 7-day L2 -> L1 leg and aliased senders ([§2](#2-canonical-rollup-and-subnet-transports)) | Cost to state wherever an L2 is offered as a conduit home; a binding must un-alias or sit at a layer that does | OP Stack bound; Arbitrum and Avalanche ICM are open candidates |
| Chain-level creation allowlists key on `tx.origin` (§2) | A per-destination prerequisite, checked against the relayer's signing key (P15) | In the spec |
| ERC-7786 has no quote, a possible second send step, and no exactly-once ([§3](#3-erc-7786-as-a-transport)) | The core contracts speak 7786; a provider's native SDK is preferred over its 7786 gateway where it has both | In the spec |
| OpenZeppelin's ERC-7930 library is a draft that would replace an in-house one (§3) | Candidate swap, blocked by the draft status and the pinned OpenZeppelin version | Open |
| Hyperlane's client contracts need OpenZeppelin 4.x ([§5](#5-hyperlane-as-a-native-binding)) | Vendor only the OpenZeppelin-free interfaces and libraries | Followed |
| Wormhole's Standard Relayer is deprecated ([§6](#6-wormhole-core-vs-the-relayer-are-two-different-bindings)) | The binding uses Core plus the Executor; the on-chain Executor quote router is deployed on two chains only, so quotes are signed | In use; revisit when the router is wider |
| An OFT can carry a message when a team already operates one ([§9](#9-layerzero-oft-as-a-carrier)) | A transceiver-less carrier binding trusts only the OFT's own verification | In use |
| Avalanche ICM is a sub-minute mesh that cannot reach Ethereum and charges relayers in an ERC-20 (§2) | Fails P8 as it stands; a candidate only if a deployment anchors on the C-Chain | Open |
| Axelar dedupes (§1) but was not read against P1 to P15 | Unknown | Open |
| NEAR Chain Signatures is signing, not messaging ([Parked](#parked-near-intents-and-chain-signatures)) | Needs a design for how a destination trusts an MPC-derived address | Parked |

| Section | Pinned to | Goes stale when |
| --- | --- | --- |
| [1. Transport replay](#1-what-each-transport-guarantees-about-replay) | source as read: LayerZero V2, Hyperlane, CCIP, Axelar, Wormhole core, OP `CrossDomainMessenger`, Arbitrum `AbsOutbox`, Warp `contract.go`, `TeleporterMessenger`. Arbitrum's L1->L2 retryable is ArbOS Go and remains unread | any of them changes how a delivered message is marked consumed |
| [2. Canonical transports](#2-canonical-rollup-and-subnet-transports) | OP Stack, Arbitrum and Avalanche; the aliasing and same-address findings are from source, the latency and fee figures from documentation | a fault-proof window changes, Superchain interop ships, or ICM changes its fee model |
| [3. ERC-7786](#3-erc-7786-as-a-transport) | ERC-7786 as of OpenZeppelin 5.5.0, whose `draft-IERC7786` is the interface vendored; `draft-InteroperableAddress` for comparison | The ERC changes |
| [4. CCIP](#4-ccip-as-a-native-binding) | `smartcontractkit/ccip`, `ccip-develop` branch, commit `171f9f0c` | Chainlink changes `CCIPReceiver`'s authentication, `Client`'s struct shapes, or the Router/OnRamp/OffRamp split |
| [5. Hyperlane](#5-hyperlane-as-a-native-binding) | `hyperlane-xyz/hyperlane-monorepo`, `main` branch, commit `983831f6`; pinned dependency versions read from `solidity/remappings.txt` (OZ `4.9.3`) | Hyperlane bumps its own OZ pin past a version the protocol can share, or changes `MailboxClient`/`Router`'s shape |
| [6. Wormhole](#6-wormhole-core-vs-the-relayer-are-two-different-bindings) | `wormhole-foundation/wormhole`, `main` branch, commit `2df4000c` (`IWormhole.sol`); `wormhole-foundation/wormhole-solidity-sdk`, `main` branch, commit `2cb855ea` | Wormhole Core adds general-message dedupe (it does not have it today), or the Executor's request or quote shape changes |
| [7. OP Stack](#7-op-stack-as-a-native-binding) | `ethereum-optimism/optimism`, `develop` branch, commit `0abfb166` (`ICrossDomainMessenger.sol`); commit `dfe4f947` (`IL2ToL2CrossDomainMessenger.sol`, `ICrossL2Inbox.sol`) | Optimism changes `relayMessage`'s calling convention, how `xDomainMessageSender` or `crossDomainMessageSource` is scoped, or interop's message format |
| [8. LayerZero](#8-layerzero-as-a-native-binding) | `@layerzerolabs/oapp-evm-upgradeable@0.1.3`, `@layerzerolabs/oapp-evm@0.4.1`; per-file commits are pinned in the protocol repository's vendor script | LayerZero moves OApp's storage namespace or its peer check |
| [9. LayerZero OFT](#9-layerzero-oft-as-a-carrier) | `devtools` and `LayerZero-v2` at the commits that script pins; frxUSD's deployment as read on 2026-10-25 | LayerZero changes how `composeFrom` is encoded or a compose is queued, or Frax re-peers or replaces frxUSD |

---

## 1. What each transport guarantees about replay

What each candidate provider actually does about a message being delivered twice, checked against the deployed source rather than the marketing. This is the evidence behind P7 and R3.5.

| Provider | Dedupes a successful message | Mechanism | Failed message retryable |
| --- | --- | --- | --- |
| **LayerZero V2** | Yes | `inboundPayloadHash[receiver][srcEid][sender][nonce]`, cleared by `_clearPayload` before the receiver is called | Yes, `lzReceive` is permissionless and a revert rolls the clear back |
| **Hyperlane** | Yes | `deliveries[messageId]` in `Mailbox`, written before `handle()`; `require(delivered(_id) == false, "Mailbox: already delivered")` | Yes, `process()` again; the `handle` call is plain, so a revert rolls the write back |
| **CCIP** | Yes | `s_executionStates[sourceChainSelector][seqNum]`; `SUCCESS` is terminal | Yes, manual execution from the `FAILURE` state |
| **Axelar** | Yes | the gateway marks a `commandId` consumed inside `validateContractCall`, which cannot be called twice | Yes, the whole `execute` reverts, so the approval survives |
| **Wormhole (core)** | **No** | `parseAndVerifyVM` verifies signatures and nothing else; the core contract keeps no record of consumed VAAs | n/a, replay is the integrator's problem |
| **OP Stack (canonical)** | Yes | `successfulMessages[versionedHash]` in `CrossDomainMessenger`, written only after the call returns true; `failedMessages` takes the rest | Yes, `relayMessage` again, and it `require`s `failedMessages[versionedHash]` for any caller that is not the other messenger |
| **Arbitrum, L2->L1** | Yes | `Outbox.spent`, a packed bitmap over the withdrawal's merkle index; `recordOutputAsSpent` reverts `AlreadySpent` | Yes, `executeBridgeCall` bubbles the revert, so a failure rolls the spent bit back |
| **Arbitrum, L1->L2** † | Yes | retryable ticket id, redeemable once | Yes, manual redeem inside the ticket lifetime, about 7 days |
| **Avalanche Warp** | **No** | the precompile verifies a BLS aggregate over the source L1's validator set and nothing else; the word "replay" does not appear in it | n/a, replay is the integrator's problem |
| **Avalanche ICM (Teleporter)** | Yes | `_receivedMessageNonces[messageID]`, written before execution; `receivedFailedMessageHashes` is a separate record of a delivered message whose execution failed | Yes, `retryMessageExecution`: the only one that retries execution without re-delivery |

**† One row is still unverified.** Arbitrum's retryable redeem is implemented in ArbOS, in Go, and `ArbRetryableTx` in `nitro-contracts` is an interface only, so nothing in that repository states the guarantee. Every other row was read from source: OP Stack's `CrossDomainMessenger` (`develop`), Arbitrum's `AbsOutbox` and `AddressAliasHelper` (`main`), the Warp precompile's `contract.go` (`master`), and `TeleporterMessenger` (`main`).

**Eight of the ten provide it, by three shapes, and the difference matters to a binding author.**

**Mark first, then call plainly:** LayerZero, Hyperlane, Axelar, and Arbitrum's outbox. The write-first order is reentrancy protection; the plain call is what gives retry, because the revert that fails the payload also rolls the mark back. One state write, both properties. Arbitrum is the clearest instance: `recordOutputAsSpent` runs before `executeTransactionImpl`, and `executeBridgeCall` re-throws the callee's revert data verbatim, so a failed withdrawal is simply not spent.

**Call, then record which way it went:** CCIP and the OP Stack. `relayMessage` makes a low-level call and branches, `successfulMessages` on success and `failedMessages` on failure, and the failed entry is what a later replay is required to come from. Two state writes, and the failure is a recorded state of its own rather than an absence.

**Mark delivery first, record execution separately:** Avalanche ICM alone, and it is the strongest of the three. `_markMessageReceived` writes the nonce before `_handleInitialMessageExecution` runs, so the message can never be delivered twice; the execution then happens through a bare `call` whose boolean is checked, and a failure is stored as `receivedFailedMessageHashes[messageID]`. Delivery and execution are separate facts, so `retryMessageExecution` re-runs a failed payload with no new bridge message. Every other transport here conflates them and needs the message re-delivered.

**The second shape looks like the thing R3.7 forbids, and is not.** That rule says a binding must never wrap the delivery call in `try/catch`, because a binding that swallows a failure consumes the message and drops the payload. A transport that catches a failure and records it as replayable has done the opposite: it is providing P6, not defeating it. The distinction is whether the failure survives the catch, and on both of these it does.

**Two of the ten are outliers, for one reason.** Wormhole's core layer and Avalanche's Warp precompile are both signature-verification primitives. They prove a message was authorised by a validator set and stop there, leaving delivery, ordering and dedupe to whatever is built on top. Wormhole's own guidance says integrators must implement replay protection themselves, offering the VAA digest or the `(emitterChain, emitterAddress, sequence)` triple as the key. Warp's equivalent is the Teleporter layer above it, which is why ICM appears separately in the table and does provide it.

*Influence on the design.* A binding on either primitive carries a guarantee the other eight inherit for free, so it is the one place where a binding holds a security property rather than a translation. Binding to ICM rather than to raw Warp avoids that on Avalanche. R3.5 states it: the transport's guarantee is imported, and a binding on a primitive supplies the dedupe itself, per receiving conduit (R3.6).

**A retry is not held to the gas the sender paid for.** Read 2026-10-21, for the five providers bound. LayerZero's `EndpointV2.lzReceive` and Hyperlane's `Mailbox.process` call the receiver with no gas argument, so it gets what the retry transaction brings; the options gas a LayerZero sender names only tells the executor what to send. CCIP's `manuallyExecute` takes a per-message override and refuses one below the message's own limit (`InvalidManualExecutionGasLimit`, OffRamp 1.6.2; `InvalidGasLimitOverride` in `execute`, OffRamp 2.0.0). OP Stack's `relayMessage` requires only `hasMinGas(minGasLimit, ...)` and calls the target with `gasleft() - RELAY_RESERVED_GAS`, and `baseGas` sizes the deposit so a call short of `minGasLimit` is recorded in `failedMessages` rather than lost. Wormhole's retry is open to anyone at any gas. So the gas a sender names can be left to the sender: too little costs a retry, never the message (R1.5). A retry can also bring value: `Mailbox.process` and `EndpointV2.lzReceive` are payable and forward `msg.value` to the receiver (read 2026-10-22, on each repository's `main`), while CCIP's receiver call carries none. That is what lets a bootstrap stuck on an empty report float pay its own report (R7.5). Not checked: which OffRamp version each CCIP lane runs.

**What this settles.** The protocol needs no `requestId` and no per-message nonce of its own. Correlation never needed one: the home derives the reporting conduit from the authenticated origin plus the stated `(owner, salt)`, and the conduit pins the first report. Idempotency does need one, and it exists at the transport for every candidate except the two raw signature primitives. Adding a protocol-level id would put a field on every channel plus a growing set on every receiver to buy something eight of ten already give. The right shape is to state the requirement, test it per binding (C29 to C31), and make the two that lack it carry the cost themselves.

Sources, for the five unmarked rows:
[LayerZero `EndpointV2.sol`](https://github.com/LayerZero-Labs/LayerZero-v2/blob/main/packages/layerzero-v2/evm/protocol/contracts/EndpointV2.sol),
[Hyperlane `Mailbox.sol`](https://github.com/hyperlane-xyz/hyperlane-monorepo/blob/main/solidity/contracts/Mailbox.sol),
[CCIP manual execution](https://docs.chain.link/ccip/concepts/manual-execution),
[Axelar Executable](https://docs.axelar.dev/dev/general-message-passing/executable),
[Wormhole core contracts](https://wormhole.com/docs/products/messaging/guides/core-contracts/).

---

## 2. Canonical rollup and subnet transports

Avalanche, the OP Stack and the Arbitrum stack differ from the providers above in kind rather than degree, and the replay matrix is the wrong lens for them. They are not third-party attestation networks: they are a chain's own bridge, so **the trust model is the chain's own** and there is no validator set to compromise separately from the chain itself. Against P4 and the protocol's security argument that is a strict improvement on every provider in §1.

What disqualifies or constrains them is elsewhere, in three properties the matrix does not capture.

### Directionality and latency

| | L1 -> L2 | L2 -> L1 | L2 <-> L2 |
| --- | --- | --- | --- |
| **OP Stack** | minutes, one step | **about 7 days**, two steps: prove, then finalize after the fault-proof window | not without Superchain interop (`L2ToL2CrossDomainMessenger`, and only inside a shared dependency set) |
| **Arbitrum** | minutes, retryable ticket, auto-redeem when funded | **about 7 days**, `ArbSys.sendTxToL1` then `Outbox.executeTransaction` after the challenge window | no |
| **Avalanche ICM** | n/a: C-Chain and Avalanche L1s, not Ethereum | n/a | **seconds, any-to-any**, the one mesh in this table |

**The asymmetry looks fatal and mostly is not.** A conduit is one-directional by construction: its home sends and every destination is a leaf, and the only return leg is the receiver report. That report fires only where a destination's address derivation diverges from Ethereum's, which is false on both rollup stacks, since they use Ethereum's CREATE2 formula. So a conduit homed on Ethereum reaching OP Stack and Arbitrum chains over their canonical bridges never needs the slow direction.

**It is fatal the other way round.** A conduit homed on an L2 that reaches its L1 over the canonical bridge puts every bootstrap and every payload through the seven-day window. Any chain can be a conduit's home, so nothing refuses this. It is a cost to state wherever an L2 is offered as a home, and a reason to reach the L1 over a mesh provider instead.

### Address aliasing, which is the concrete trap

An L1 contract that deposits to an L2 does not arrive as itself. Arbitrum and the OP Stack's `OptimismPortal` both add `0x1111000000000000000000000000000000001111` to the sender, so the L2 sees an aliased address. The receiving side compares the sender against a transmitter contract's address, so **a naive canonical binding fails every inbound message** unless it un-aliases first.

The two stacks differ in whether the binding has to. OP Stack's `CrossDomainMessenger` un-aliases and exposes the original sender through `xDomainMessageSender()`, so a binding at that layer sees the real address; one built directly on `OptimismPortal` does not. Arbitrum has no equivalent, so undoing the alias is the binding's job. The offset is Arbitrum's `AddressAliasHelper.OFFSET` character for character, so one constant serves all three cases.

Undoing an alias has two traps. It applies to L1 -> L2 only, so applying it to a withdrawal corrupts a sender that arrived unaliased. And the arithmetic wraps, so undoing an alias that was never applied yields a well-formed address belonging to nobody, rather than reverting. Decide from the direction of the message, never from the value. Exactly one input undoes to the zero address, the offset itself, so a zero check on the result buys nothing.

### Fees and quotes

| | Fee at source, native? | `view` quote? |
| --- | --- | --- |
| **OP Stack** | yes, L2 gas bought through the deposit | **no**: you state `minGasLimit` and pay for it, and there is nothing to ask for a price |
| **Arbitrum** | yes | **partly**: `Inbox.calculateRetryableSubmissionFee(dataLength, baseFee)` is a view, and `NodeInterface.estimateRetryableTicket` covers the L2 gas |
| **Avalanche ICM** | **no**: the relayer incentive is an ERC-20 | n/a |

Arbitrum is the only one of the three that partly satisfies P9. OP Stack has no quote function, but its native fee is zero (see the correction in [§7](#7-op-stack-as-a-native-binding)), so a binding quotes zero rather than failing. Avalanche ICM fails P8 outright: a per-chain ERC-20 fee reintroduces exactly the funding matrix the protocol exists to remove, so a binding would have to pay relayers some other way or accept that signers hold a fee token per destination.

### None of them is a fan-out

The providers in §1 are meshes: one binding reaches every chain they support. A canonical bridge connects one L2 to one L1, and Avalanche Warp connects Avalanche L1s to each other and to nothing else. For Arbitrum and Avalanche, neither of which is bound, that means one provider registration and one transceiver pair per rollup, not one for the stack. It is also what makes the trust argument worth having: a payload to one chain trusts that chain's bridge and nothing else, instead of one attestation network for every destination at once. The cost is N deployments, N graded chain entries, and N sets of routes.

**OP Stack is the exception.** Its L1 side needs only one provider for every OP Stack chain, because the destination is which messenger contract is called, so a write-once table of the messenger that reaches each chain is enough. The trust argument survives because a delivery's origin is the chain of the messenger that called, never anything in the message: a compromised Base bridge can only deliver as Base. L2 to L2 is a separate provider over Superchain interop's `L2ToL2CrossDomainMessenger` predeploy, which names a destination by chain id and reports the source chain on delivery (`crossDomainMessageSource`). So OP Stack is two providers, and the second is not deployable until interop is live.

### Chain-level deployment permissioning, which breaks bootstrap and not sends

subnet-evm ships an optional `ContractDeployerAllowList` precompile at `0x0200000000000000000000000000000000000000`. A chain enables it in genesis, and from then on only allowlisted addresses may create contracts. It is opt-in, so most chains do not have it; **two that LayerZero supports do.** This is the evidence behind P15.

| | precompile | role of an arbitrary address |
| --- | --- | --- |
| Avalanche C-Chain | absent (`eth_getCode` empty) | n/a, deployment is open |
| **DFK Chain** | **active** | `0`, None |
| **Dexalot** | **active** | `0`, None |

**It gates `tx.origin`, not the contract performing the CREATE**, which decides how bad this is, and DFK says so in the error text. Probing it three ways:

```
top-level deploy, non-allowlisted from   ->  "tx.origin 0x1234… is not authorized
                                              to deploy a contract"
CREATE2 via Arachnid's factory, same from ->  execution reverted
CREATE2 via Arachnid's factory, from 0x0  ->  0xb2e363e5…da1e80a1   (0x0 has role Enabled)
```

The factory is a contract doing the create, and it fails or succeeds purely on who originated the call. So allowlisting the protocol's own contract buys nothing.

**What that costs is bootstrap, and only bootstrap.** A conduit is created inside the inbound delivery callback, so `tx.origin` there is whoever submitted the delivery transaction: the provider's relayer or executor, not the protocol and not the owner. On such a chain, bootstrap works only if that relayer is allowlisted, and P6's permissionless retry is gone with it, since an arbitrary party retrying a failed bootstrap is exactly an arbitrary `tx.origin`. Ordinary sends are untouched, since a send creates no contract.

**On these two chains today, LayerZero's executor cannot deploy, so bootstrap cannot happen.** Not degraded: impossible. The Executor contract is the wrong thing to check, since the gate is on the origin, and the origin is the off-chain key that drives it. Recent deliveries on both chains come from one signing EOA, and it holds no role on either:

| | LayerZero executor contract | its signing EOA `0xe93685f3…` |
| --- | --- | --- |
| DFK Chain | `None` | **`None`** |
| Dexalot | `None` | **`None`** |

**And no arbitrary conduit can fix that for itself.** Granting is Admin-only, and the precompile refuses everyone else by name:

```
setEnabled from a random EOA  ->  cannot modify allow list: modify address: 0x1234…,
                                  from role: NoRole, to role: EnabledRole
setEnabled from 0x0           ->  same, though 0x0 holds Enabled on DFK and CAN deploy
setAdmin  from a random EOA   ->  same, to role: AdminRole
```

The second line is the useful one: **Enabled is not Admin**. An address that may deploy still may not grant, so there is no route through a friendly party. Making either chain reachable requires that chain's own admin to allowlist the provider's executor key, a governance ask to a third party, outside the control of both the protocol and LayerZero.

**It is also keyed on an operational key.** If the provider rotates its executor signing address, bootstrap breaks again with no change on either side and no event anyone here would see. A deployment that depends on this arrangement has a liveness dependency on somebody else's key-rotation policy matching somebody else's governance queue.

*Influence on the design.* The shape to check for is any chain policy keyed on `tx.origin`, not the precompile address, because everything the protocol does on a destination happens inside somebody else's transaction. P15 states it as a per-destination prerequisite, checked by reading the allowlist for the relayer's signing key rather than for any contract.

**Teleporter imposes a same-address property the protocol already relies on.** `receiveCrossChainMessage` requires `warpMessage.originSenderAddress == address(this)`: the messenger accepts a message only from a messenger at its own address on the source chain. That is the conduit-is-its-own-peer rule one layer down, arrived at independently, so an ICM binding would inherit a deployment constraint the protocol was going to impose anyway.

**DFK's CREATE2 is EIP-1014**: the address it returned is the one computed for that salt and initcode.

**Aurora is a parity chain, checked rather than assumed.** NEAR itself runs WASM and has no EVM, so Solidity does not run on it; Aurora is an EVM implemented as a NEAR contract, and it presents as `eip155` chain 1313161554. The question that decides whether it needs anything of its own is whether its CREATE2 is EIP-1014, and it is. Arachnid's factory is deployed there at the usual address, and `eth_call`ing it executes on Aurora's own engine rather than on a local simulation, so it answers directly. Three salts over the conduit proxy's initcode returned exactly the addresses EIP-1014 predicts:

```
salt 00000000…   0x5a6eee7a8eb1d36ef7336bd24c57b975fb9ebc15
salt c0ffee01     0xb2e363e52060ca5f20a59fac76cf1960da1e80a1
salt deadbeef…   0x70e2fc1339425ad82497b92828ac23248b804297
```

So Aurora needs nothing chain-specific. NEAR itself is a different question (see [Parked](#parked-near-intents-and-chain-signatures)).

**The same trick does not settle Tron.** Arachnid's factory relies on a pre-signed Ethereum transaction and is absent from both Tron mainnet and Shasta, so there is no live CREATE2 factory to `eth_call`. java-tron's source settles the formula instead; the conduit bytecode hash still needs a funded deployment.

**Avalanche is the odd one and the interesting one.** ICM is a real mesh, sub-minute and bidirectional, a better shape than anything else here; it just cannot reach Ethereum. It is the transport to reach for if a deployment ever anchors on the C-Chain, and irrelevant otherwise.

---

## 3. ERC-7786 as a transport

OpenZeppelin ships `interfaces/draft-IERC7786.sol` and `crosschain/ERC7786Recipient.sol`, and the protocol arrived at nearly the same shape independently: an opaque `bytes` payload to a recipient named by an interoperable address, with an opaque per-send options blob. So the question is worth answering once rather than rediscovering per provider.

**The protocol implements the standard directly rather than adopting OpenZeppelin's helper.** Its transmitter is an ERC-7786 gateway source and its receiver an ERC-7786 recipient. OpenZeppelin's `CrosschainLinked(Upgradeable)` is not adopted: it sits behind `Bytes.sol` and its four `mcopy` sites, so it cannot compile at the `paris` target, and independently its per-contract links table and gateway allowlist would replace the shared-transceiver routing and bypass the registry's provenance dial. The interfaces are held as a copy of OpenZeppelin 5.5.0's. The `draft-` prefix is upstream saying it may change the API in a minor release, and these two interfaces are the protocol's ABI (an event topic, a selector, an argument order), so a change should be a reviewed diff on the protocol's schedule rather than a side effect of a dependency bump.

### The one thing that does not map, and how it resolves

ERC-7786 addresses a recipient as a binary interoperable address: chain and address in one ERC-7930 blob. The protocol's chain name is a `chainKey`, which is `keccak256(<canonical chain identifier>)` and therefore one-way. A chainKey cannot produce a 7786 recipient.

**Store the chain identifier in the route.** Then the reverse index is correct by construction rather than by configuration, since `keccak256(identifier) == chainKey` is the definition of a chainKey, and the route holds a chain identifier instead of a provider-native id. A 7786 gateway then needs no provider-native chain id, which is what the route table existed to hold.

### Where each seam attaches

| Protocol hook | ERC-7786 |
| --- | --- |
| send | `gateway.sendMessage{value: value}(recipient, payload, attributes)`, one to one. Pay from the amount the transmitter allots, never from `msg.value` |
| quote | **nothing.** See below |
| inbound | `receiveMessage(receiveId, sender, payload)`, splitting the sender envelope into a bare chain identifier for the route and a parsed address for the sender |
| attributes | passed straight through |

### What ERC-7786 gives up

1. **No quote, at all.** The interface is `supportsAttribute` and `sendMessage`. A payable send with no way to ask its price fails P9 and removes the whole quote surface. It also removes every send: the transmitter prices each one on chain (R2.6), so a binding over a gateway with no quote extension cannot send. This is the largest cost and the reason to prefer a native SDK where one exists.
2. **`sendMessage` may not complete the send.** It returns a `sendId`, and a non-zero value means further gateway-specific, non-standardised action is required. A binding must either handle the second step or restrict itself to gateways that return zero, and say which.
3. **No mandated exactly-once.** The standard defines a `receiveId` for correlation but requires nothing about replay, so R3.5 stays a per-gateway question rather than being answered by the standard. The `receiveId` is free where the protocol's own channels carry no id; the protocol concluded none was needed (§1).

### The other draft worth knowing about

`utils/draft-InteroperableAddress.sol` is OpenZeppelin's ERC-7930. It covers the same ground as the protocol's own ERC-7930 library, with `formatEvmV1`, `parseEvmV1`, and `try` and calldata variants. Replacing the in-house library with it is a real candidate: audited, maintained, and one fewer library to own.

Two things block a straight swap. It is a `draft-`, which OpenZeppelin excludes from its API stability guarantee and may change in a minor release, while conduits are frozen against exact bytes. And the in-house `parseStrict` enforces strictness the registry depends on, rejecting non-minimal `eip155` references and trailing bytes; whether `parseV1` matches has not been checked. Neither is a reason not to do it; both are reasons it is its own task with its own test vectors.

**Declined for now**, separately: moving onto this draft would mean bumping the pinned OpenZeppelin past 5.4.0, and that bump is the same one that breaks `AccessControlEnumerableUpgradeable`'s compilation at `paris` from 5.5.0 onward. The dependency the swap would need is the dependency the pin cannot survive, so the pin is kept.

---

## 4. CCIP as a native binding

Chainlink CCIP, read the same way LayerZero was in [§8](#8-layerzero-as-a-native-binding): what a native binding would inherit and what it owes R3.3 and R5. Source: `smartcontractkit/ccip`, branch `ccip-develop`, commit `171f9f0c`, reading `contracts/src/v0.8/ccip/applications/CCIPReceiver.sol`, `contracts/src/v0.8/ccip/interfaces/{IAny2EVMMessageReceiver,IRouterClient}.sol`, and `contracts/src/v0.8/ccip/libraries/Client.sol`.

**One Router, both directions.** A chain's CCIP `Router` implements `IRouterClient` (`getFee(selector, message) view`, `ccipSend(selector, message) payable`) for the send side, and is the only contract CCIP will ever call `ccipReceive` from. One gateway grant covers both, the same as every other candidate.

**The chain identifier is a `uint64` selector, not an EVM chain id.** Same shape as LayerZero's `eid` and Hyperlane's `domain`: a native CCIP binding needs its own chainKey to selector table under R5.

**The wire addresses are ABI-encoded, not raw bytes.** `Client.EVM2AnyMessage.receiver` is `abi.encode(address)` for an EVM destination, and `Client.Any2EVMMessage.sender` is `abi.decode`d the same way for an EVM source. Neither is a raw 20-byte slice, which is what LayerZero and a bare EVM address use, so sender narrowing has to decode, not slice.

**A native `view` quote, satisfying P9 outright.** `getFee` prices the exact message with no fallback needed, unlike ERC-7786 (no quote at all) or OP Stack (no quote function, and a native fee of zero). `feeToken = address(0)` pays in native currency through `msg.value`, the only choice consistent with P8: CCIP also supports paying in LINK, and using it would reintroduce the per-chain funding-token matrix P8 exists to avoid.

**`CCIPReceiver` carries zero storage and zero external dependency.** It holds one `immutable` (`i_ccipRouter`, set in the constructor, the same one-constructor-argument-on-the-implementation shape as LayerZero's endpoint; see [§8](#8-layerzero-as-a-native-binding)) and vendors its own tiny copy of `IERC165` rather than importing OpenZeppelin's. No storage-layout question, no OpenZeppelin-version question. It is the cleanest of the candidates to fit into a proxy layout.

**No R3.3-style exception is needed, provided a binding skips `CCIPReceiver` itself.** `CCIPReceiver.ccipReceive` is gated by `onlyRouter` alone (a transport-identity check) and asserts nothing about who sent the message on the source chain. Unlike LayerZero's `_lzReceive`, which authenticates the peer before handing control to the application and cannot be bypassed without forking the receive path, CCIP's base contract leaves the source-chain sender entirely to the application. So a binding that implements `IAny2EVMMessageReceiver.ccipReceive` directly, gated by the gateway role, and authenticates the source chain and sender only through the protocol's own origin check, has exactly one origin check in exactly one place.

**Licensing is mixed, and matters for what gets vendored.** The infrastructure contracts the protocol only ever calls (`Router.sol`, `FeeQuoter.sol`, the on-ramp and off-ramp internals) are `BUSL-1.1`. The application-facing files a binding needs (`CCIPReceiver.sol`, `IAny2EVMMessageReceiver.sol`, `IRouterClient.sol`, `Client.sol`) are each individually tagged `MIT` in their own SPDX header. Worth a second look before mainnet, as any vendored license is, but the files needed are not the BUSL ones.

**Replay is already covered.** §1 records CCIP's dedupe (`s_executionStates[selector][seqNum]`, `SUCCESS` terminal, manual re-execution from `FAILURE`).

---

## 5. Hyperlane as a native binding

Hyperlane, read the same way. What a native binding would inherit, and the one finding that rules out inheriting most of it. Source: `hyperlane-xyz/hyperlane-monorepo`, branch `main`, commit `983831f6`, reading `solidity/contracts/{Mailbox.sol,PackageVersioned.sol}`, `solidity/contracts/client/{MailboxClient,Router}.sol`, `solidity/contracts/interfaces/{IMailbox,IMessageRecipient}.sol`, `solidity/contracts/libs/TypeCasts.sol`, and `solidity/remappings.txt`.

**One Mailbox, both directions.** `dispatch`/`quoteDispatch` (send) and `process` (the permissionless relay call that verifies the message's ISM and then calls the recipient's `handle`) live on the same `Mailbox` contract. One gateway grant covers both.

**The chain identifier is a `uint32` domain, not reliably an EVM chain id.** It conventionally equals the EVM chain id for EVM chains, but that is a convention, not a guarantee the protocol may depend on, and it is not meaningful for a non-EVM chain. A native binding needs its own chainKey to domain table under R5, as with LayerZero's `eid` and CCIP's selector.

**A native `view` quote.** `IMailbox.quoteDispatch(domain, recipient, body[, hookMetadata, hook])` prices the exact send, satisfying P9 outright, the same shape as CCIP's `getFee`.

**Addresses are `bytes32`, via a pure library.** `TypeCasts.addressToBytes32` and `bytes32ToAddress` are alignment-preserving casts with no imports and nothing to collide with.

**The most important finding: `MailboxClient` and `Router` cannot be inherited into the protocol's proxies at all, and it is not a style question.** Both bring `OwnableUpgradeable`, and `hyperlane-monorepo`'s own `remappings.txt` pins `@openzeppelin/contracts-upgradeable` at 4.9.3. `MailboxClient._MailboxClient_initialize` calls `__Ownable_init()` with no argument, which is OpenZeppelin 4.x's signature; the protocol is pinned to 5.4.0, whose `OwnableUpgradeable` takes an explicit initial owner and has no zero-argument overload. The two versions cannot occupy one inheritance graph: Solidity will not compile two incompatible definitions of the same base reached through different import paths. This is a harder wall than the ERC-7930 swap question in §3: that one is blocked by a compile-target collision (`paris` against `mcopy`) that a version choice could in principle resolve; this one is blocked by two shipped, incompatible major versions of the same contract.

**The fix costs nothing, because the useful parts have no OpenZeppelin dependency at all.** `IMailbox.sol`, `IMessageRecipient.sol`, `IInterchainSecurityModule.sol`, and `TypeCasts.sol` are plain interfaces and a pure library. `IMailbox` imports `IPostDispatchHook`, itself a plain interface, and `StandardHookMetadata` carries the refund address, so the set to copy is six files. A binding copies those, holds the `IMailbox` address as its own immutable, and implements `IMessageRecipient.handle(uint32 origin, bytes32 sender, bytes calldata message)` itself, gated by the gateway role rather than `MailboxClient`'s `onlyMailbox`. Since neither `MailboxClient` nor `Router` is inherited, their `uint256[48] __GAP` storage reservations never enter the protocol's layout.

**Skipping `Router` is independently correct.** `Router.handle` checks an enrolled-router-per-domain mapping before calling the application's `_handle`, the same shape rejected for OpenZeppelin's `CrosschainLinked` in [§3](#3-erc-7786-as-a-transport): its own gateway allowlist would replace the shared-transceiver routing and bypass the registry's provenance dial.

**No R3.3-style exception is needed, provided `Router` is skipped.** `Mailbox.process` authenticates that a message passed its configured ISM before calling `handle`, a statement about the message, not about which contract sent it on the source chain. The per-domain peer check is `Router`'s own optional layer, not `Mailbox`'s. So a binding built directly on `IMessageRecipient` and `IMailbox` has exactly one origin check in exactly one place, unlike LayerZero, where the peer check is inside `_lzReceive` and cannot be opted out of short of forking the receive path.

**Verification is the ISM, and the Mailbox owner can replace it.** A destination's default ISM is mapped per origin, so what verifies a message can change without any change on the protocol's side. Recheck it before relying on a lane.

**Licensing is uniform and clean.** Every file read here is `MIT OR Apache-2.0`. No BUSL question, unlike CCIP's mixed repository.

**Replay is already covered.** §1 records Hyperlane's dedupe (`deliveries[messageId]` in `Mailbox`, written before `handle()`, guarded by `already delivered`).

---

## 6. Wormhole: Core vs. the Relayer are two different bindings

Wormhole is qualitatively different from the other candidates, and §1's replay matrix already flags it as one of only two outliers (with Avalanche's raw Warp precompile) that leave dedupe to the integrator. "Wormhole" also names two separable products with almost nothing in common at the interface level: Core, the guardian-signed message primitive, and a delivery layer built on top. The delivery layer used to be the Standard Relayer. Wormhole now marks the Standard Relayer deprecated and points integrators to the Executor framework; its automatic delivery ended on 2026-04-01. The protocol's binding therefore takes Core with the Executor for delivery. The Relayer is kept below as the comparison that explains why. Source: `wormhole-foundation/wormhole`, `main`, commit `2df4000c`, reading `ethereum/contracts/interfaces/IWormhole.sol`; and `wormhole-foundation/wormhole-solidity-sdk`, `main`, commit `2cb855ea`.

### Core (`IWormhole`): the trust-minimized primitive, and the hard case

**`publishMessage` names no destination.** `publishMessage(nonce, payload, consistencyLevel) payable returns (sequence)` takes no target chain and no target address. It only emits `LogMessagePublished`; the guardian network observes that event off chain and signs a VAA attesting to `(emitterChainId, emitterAddress, sequence, payload)`. Where that VAA goes, and whether it goes anywhere, is decided entirely off protocol, by whoever chooses to do something with it.

**There is no delivery, so there is no delivery-inclusive quote.** `messageFee()` is a flat, tiny anti-spam fee paid on the source chain, unrelated to destination gas. Every other provider surveyed prices the full round trip in one native-currency number; bare Core has nothing to ask. A binding on it needs a quote from somewhere else, which is what the Executor supplies below.

**No push callback, and therefore no caller to gate.** `parseAndVerifyVM(encodedVM) view returns (vm, valid, reason)` is a pure verification function. Wormhole Core never calls anything. A binding has to expose its own entry point that accepts a raw VAA from whoever submits it, calls `parseAndVerifyVM` itself, and proceeds only if `valid`. Authenticity comes entirely from the guardian signatures inside the VAA, checked against `vm.emitterChainId` and `vm.emitterAddress`, with nothing checking who submitted the transaction. This is the first candidate whose entry point has no role check in front of the protocol's inbound seam.

**No replay protection, confirmed against the full interface, not assumed.** There is no `delivered`/`consumed` mapping for an ordinary message anywhere in `IWormhole`. `governanceActionIsConsumed` exists, and it is scoped to governance actions only. A binding on bare Core is therefore the first to actually trigger R3.5 ("a binding whose transport does not provide it MUST supply it"): it needs its own consumed-VAA-hash map, held on the binding's own receiver the way a peer table or an eid table lives on a binding.

**Bootstrap is the one place bare Core's permissionless-relay model is a fit rather than a cost.** Bootstrap is already designed to be callable by anyone willing to pay for it; a permissionlessly submittable VAA composes with that directly.

### The Standard Relayer (`IWormholeRelayer`): deprecated, and the comparison case

A separate product built on top of Core, not a mode of it, and confusingly also called "Wormhole" in most integration guides.

| Protocol hook | Wormhole Relayer |
| --- | --- |
| send | `sendPayloadToEvm(targetChain, targetAddress, payload, receiverValue, gasLimit){value}`: an explicit destination, finally |
| quote | `quoteEVMDeliveryPrice(targetChain, receiverValue, gasLimit).nativePriceQuote`: destination-inclusive, in this chain's native currency |
| gateway role | granted to the Relayer contract, as with the other providers |
| inbound | `IWormholeReceiver.receiveWormholeMessages(payload, additionalMessages, sourceAddress, sourceChainId, deliveryHash)`: a real push callback |

**Dedupe exists at this layer.** `deliveryAttempted(bytes32 deliveryHash) view returns (bool)` tracks delivery the way LayerZero, CCIP and Hyperlane do, so a binding on the Relayer would not have needed the consumed-hash map bare Core requires. (Checked against the implementation: `executeDelivery` refuses a hash already in `deliverySuccessBlock`; a failed delivery can be retried. A reverting `receiveWormholeMessages` does not revert the delivery transaction. The implementation was read at `wormhole-foundation/wormhole` `932a2e0a2c`, the parent of the commit that deleted it from that repo.)

**The chain id is `uint16`, the "Wormhole Chain ID" format**, a fourth provider-native width, which is why the protocol's chain-id table is `uint256`-widened rather than sized to any one provider.

**The trust assumption moves.** Message authenticity still rests on the guardian-signed VAA underneath. But delivery liveness also depends on a delivery-provider marketplace (a default provider, typically Wormhole Labs-operated, or an alternate one named by address) actually submitting the transaction. LayerZero's executor, CCIP's off-ramp and Hyperlane's relayer are also third parties, so this is not unique to Wormhole, but it is a fact to grade the way the registry grades a counterpart's address claim, not to wave through because the signature underneath is sound.

### Core plus the Executor: what the binding uses

| Protocol hook | Wormhole Core + Executor |
| --- | --- |
| send | `ICoreBridge.publishMessage{messageFee}` then `IExecutor.requestExecution{price}` with the caller's signed EQ01 quote. The Executor pays everything it is sent to the quote's payee, so the binding refunds the rest itself. Publish-only stops after Core, and whoever holds the VAA delivers it |
| quote | `messageFee()` plus the price the quote states for the gas, computed on chain with the Executor's own formula (`ExecutorQuoter.estimateQuote`) |
| inbound | `IVaaV1Receiver.executeVAAv1(vaa)`, permissionless; verified via `parseAndVerifyVM` |
| replay | the binding's own consumed-hash set, as the Core section anticipated |

Two facts the Relayer used to cover. A VAA names no destination, so the binding prefixes the payload with `(targetChain, targetAddress)` and checks both. And the gateway role has no caller to name, so it is granted to the Core bridge and checked by membership. Sources: `wormhole-solidity-sdk @ 2cb855ea` (`interfaces/ICoreBridge.sol`, `interfaces/IExecutor.sol`, `Executor/Request.sol`, `Executor/Integration.sol`) and `wormholelabs-xyz/example-messaging-executor @ 55f94274` (`Executor.sol`; the quote layout in `README.md`, the formula in `ExecutorQuoter.sol`). The on-chain quote path (`ExecutorQuoterRouter`, wormhole-solidity-sdk#118) is deployed only on Polygon and Monad, so the binding takes signed quotes, Wormhole's production path. A wider router deployment would let the quote move on chain.

*Influence on the design.* Neither variant needed a change to the shared base contracts: the send and quote hooks are per-binding overrides, and the inbound seam is reusable by any entry point regardless of its own gating. The Core binding is real, additional, self-contained work (a permissionless entry point and a replay-protection map, both local to the binding), not a gap in what already existed.

---

## 7. OP Stack as a native binding

[§2](#2-canonical-rollup-and-subnet-transports) covers latency, aliasing, fees, and the no-fan-out property at the level of what constrains a canonical rollup bridge generally. This section is the interface-level follow-up, read the same way CCIP, Hyperlane, and Wormhole were. Source: `ethereum-optimism/optimism`, `develop`, commit `0abfb166`, reading `packages/contracts-bedrock/interfaces/universal/ICrossDomainMessenger.sol`; and, for Superchain interop, commit `dfe4f947` reading `IL2ToL2CrossDomainMessenger.sol` and `ICrossL2Inbox.sol`.

**No chain id, and no chain-id table, because there is no fan-out to name.** `sendMessage(address _target, bytes memory _message, uint32 _minGasLimit) external payable` takes no destination chain at all. Each OP Stack chain has its own dedicated `L1CrossDomainMessenger` at its own address on L1; the destination is which messenger contract you call, not an argument to it. So the L1 to L2 provider names a chain by the address of the messenger that reaches it: on L1 one row per OP Stack chain, and on each OP Stack chain one row, the L1, reached through the `L2CrossDomainMessenger` predeploy. The same transceiver runs on both sides.

**No on-chain quote, confirmed against the full interface.** There is no `quote`-shaped function anywhere in `ICrossDomainMessenger`. `baseGas(message, minGasLimit)` exists, but it returns a `uint64` gas overhead, not a native-currency price; it still has to be combined with an off-chain gas price to produce a `msg.value`. This confirms §2's finding at the interface level: a binding's quote has nothing to answer from, and the correction below is what keeps the binding usable, since a transmitter prices every send on chain (R2.6).

**Correction: the native quote is zero, not missing.** Checked against `CrossDomainMessenger.sendMessage` and `ResourceMetering` at the same commit: a deposit's L2 gas is paid by burning L1 gas in the sending transaction, and `sendMessage` forwards `msg.value` to the target as bridged ETH (`relayMessage` calls the target with `_value`). Nothing is charged in `msg.value`, so the binding requires zero value and quotes zero, which is what a balance-delta measurement (R2.2.2) would report. `baseGas` remains the only price-shaped function, and it prices gas, not currency.

**The trap: the authenticated sender is retrieved by a callback, never carried as an argument, and treating it as one would be a real vulnerability rather than a style choice.** `relayMessage(nonce, sender, target, value, minGasLimit, message)` makes a low-level call to `target` with `message` as calldata. Whoever called `sendMessage` on the origin domain chose that calldata in full, since `sendMessage` is permissionless and anyone may target the protocol's receiver with it. Every other provider surveyed hands the authenticated origin to the callback as a value the transport computed (LayerZero's `Origin`, Hyperlane's `sender`, CCIP's `message.sender`, Wormhole's `sourceAddress`). OP Stack does not: nothing about `message`'s own bytes is trustworthy, because the caller who published it wrote every byte of it. The only fact the messenger guarantees is retrievable separately, by calling `xDomainMessageSender()` on the messenger during the execution `relayMessage` triggers. A binding must call this itself and must not accept a sender as part of `message`'s own payload; an entry point that trusted a self-declared sender would let anyone impersonate the transmitter by encoding a claim to that effect in `message`. The comparison against the expected source still happens as for any provider; what changes is where the value being compared comes from.

**Binding at the messenger layer is what makes aliasing someone else's problem.** §2 notes that `OptimismPortal` requires undoing `AddressAliasHelper`'s offset yourself, while `CrossDomainMessenger` un-aliases internally and hands back the real address through `xDomainMessageSender()`. Binding one layer up means the alias-undoing path stays unused. It would only be needed by a binding built directly on `OptimismPortal`, which this is deliberately not.

**Interop (L2 to L2) is a second provider with a different shape.** `L2ToL2CrossDomainMessenger` names a destination by chain id and reports the source chain on delivery (`crossDomainMessageSource`), so unlike the L1 messengers it takes a chain argument. A message executes only if its source is in the destination's dependency set, so a chain may be routed only within its set. `sendMessage` is non-payable and `relayMessage` permissionless, so delivery depends on someone relaying. Whether a source log expires after some window, ending its retry, is not settled. It imports only `ICrossL2Inbox`.

**No divergent variant, unlike the other providers.** zkSync and Tron need their own transceiver variants because their CREATE2 formulas differ from Ethereum's. An OP Stack chain runs standard `op-geth` and Ethereum's own CREATE2 formula, so it is always the parity case.

**No OpenZeppelin dependency and no storage.** `ICrossDomainMessenger` is a plain interface with zero imports. A transmitter stays plain `OwnableUpgradeable`, the same conclusion reached for CCIP, Hyperlane, and Wormhole: there is no SDK `Ownable` to merge with, unlike LayerZero's OApp.

**Replay is already covered**, as §1 records: `successfulMessages`/`failedMessages` in `CrossDomainMessenger`, the call-then-record-which-way-it-went shape shared with CCIP. Licensing: MIT, like the rest of `contracts-bedrock`.

---

## 8. LayerZero as a native binding

The first binding, and the one the others were read against. Verified against `@layerzerolabs/oapp-evm-upgradeable@0.1.3` (it imports interfaces from `@layerzerolabs/oapp-evm`, 0.4.1 at the time of checking). None of the packages the binding depends on has a dedicated repository that isolates just those files: `@layerzerolabs/oapp-evm-upgradeable` and `@layerzerolabs/oapp-evm` live in `LayerZero-Labs/devtools`, `@layerzerolabs/oft-evm` has no repository of its own, and `@layerzerolabs/lz-evm-protocol-v2` has no standalone package repository. Their actual home is those monorepos, so the files are copied at pinned commits rather than taken as dependencies. `IOFT`, `OFTComposeMsgCodec` and `ILayerZeroComposer` are copied too, for the OFT carrier ([§9](#9-layerzero-oft-as-a-carrier)).

### What was found in the package

```solidity
abstract contract OAppCoreUpgradeable is IOAppCore, OwnableUpgradeable {
    ILayerZeroEndpointV2 public immutable endpoint;
    constructor(address _endpoint) { endpoint = ILayerZeroEndpointV2(_endpoint); }
    function __OAppCore_init(address _delegate) internal onlyInitializing {
        if (_delegate == address(0)) revert InvalidDelegate();
        endpoint.setDelegate(_delegate);
    }
}

abstract contract OAppUpgradeable is OAppSenderUpgradeable, OAppReceiverUpgradeable {
    constructor(address _endpoint) OAppCoreUpgradeable(_endpoint) {}
    function __OApp_init(address _delegate) internal onlyInitializing {
        __OAppCore_init(_delegate);
        __OAppReceiver_init_unchained();
        __OAppSender_init_unchained();
    }
}
```

**One constructor argument, and it does not matter.** It is on the implementation, and an implementation's address lives in the proxy's ERC-1967 storage slot rather than in its initcode, so the conduit proxy stays argument-free and no derived address moves (R8.1). The endpoint being an `immutable` is correct rather than merely tolerable: every conduit on a chain uses the same endpoint, which is what an implementation-level immutable expresses.

**It fits the protocol's layout without collisions.** `OAppCoreUpgradeable` keeps its peer mapping in ERC-7201 namespaced storage (`OAPP_CORE_STORAGE_LOCATION`), as do `OAppOptionsType3`, `PreCrime`, and the simulator, so it cannot collide with the protocol's own storage however that layout changes (R8.2).

**`Ownable` is deliberately left uninitialized.** The package says so: *"Ownable is not initialized here on purpose. It should be initialized in the child contract to accommodate the different version of Ownable."* Since it derives OpenZeppelin's `OwnableUpgradeable`, the same one the transmitter uses, initializing the owner and the sender side composes rather than collides, and the two are one owner rather than two authorities. A binding must not add a third.

### Where each seam attaches

The hooks are ERC-7786-shaped, so a native LayerZero binding has one translation: the recipient arrives as an ERC-7930 envelope and the eid has to come back out of it.

| Protocol hook | LayerZero |
| --- | --- |
| send | `endpoint.send{value}(MessagingParams(eid, peer, payload, options, false), refund)` through the linked message library, not `_lzSend`, whose `_payNative` requires `msg.value` to equal the fee: `eid` from the binding's own chainKey to eid table, `options` decoded from `attributes`, `refund` from the transmitter's refund address |
| quote | `endpoint.quote(MessagingParams(...), address(this)).nativeFee`, over the same `eid` and `options` the send resolves |
| gateway role | granted to `address(endpoint)` in the conduit's initializer |
| inbound | `_lzReceive`, with the route the stored chain identifier for `origin.srcEid` and the sender narrowed per R4.2 |
| conduit initialization | must build `__OApp_init(delegate)` and the peer, since the conduit locks in the same call |

**A native binding reintroduces a codec, and the eid table with it.** ERC-7786 removed the protocol's need for a provider id, not LayerZero's: `MessagingParams` still takes a `uint32`. So a native LayerZero binding keeps its own chainKey to eid mapping under R5, where a gateway binding keeps none.

**The peer value is the destination's counterpart, not `address(this)`.** The two agree wherever Ethereum's CREATE2 formula holds, which made the shortcut tempting; they differ on zkSync and Tron, where deriving the peer names an address holding no receiver. One entry per destination, read from a table rather than computed.

**Check a pathway's default verification before relying on it.** The protocol's own checks found that pathways touching Fraxtal and zkSync default to a DVN that is dead, so both ends of such a lane must pin a working one.

### Two consequences, both settled

- **`__OAppCore_init` calls `endpoint.setDelegate(_delegate)`, and the delegate is `address(this)`.** Inside the initializer's delegatecall that is the conduit, so each conduit is its own delegate and no other party can reconfigure it. The cost is that every conduit creation touches the endpoint, which is real gas on the bootstrap path and the concrete form of "peers and any send-side security configuration are per-user rather than shared". Recorded as R6.4, which generalises it: any provider-side authority over a conduit is the conduit.
- **OApp authenticates inbound before the protocol's code runs.** `lzReceive` does `if (address(endpoint) != msg.sender) revert OnlyEndpoint(...)` then `if (_getPeerOrRevert(_origin.srcEid) != _origin.sender) revert OnlyPeer(...)`. That is the R3.3 exception: for a 1:1 pairing there is nothing extra to verify, and the binding documents the exception where it takes it. The receiver checks the gateway role on top, so revoking the gateway still disconnects LayerZero. LayerZero is the only binding that takes the exception.

---

## 9. LayerZero OFT as a carrier

The `layerzero-oft` binding, held to [`layerzero-oft-spec.md`](https://github.com/crossecute/protocol/blob/main/docs/layerzero-oft-spec.md). It reads, and copies, only `IOFT`, `OFTComposeMsgCodec` and `ILayerZeroComposer` (§8). It carries a conduit's messages through an OFT its own team operates, using the OFT's compose path as the delivery mechanism.

### What LayerZero guarantees

Read on 2026-10-24 and 2026-10-25 in `LayerZero-Labs/devtools` (`packages/oft-evm`) and `LayerZero-Labs/LayerZero-v2` (`MessagingComposer.sol`, `PacketV1Codec.sol`, `ExecutorOptions.sol`, `ExecutorFeeLib.sol`), on main and at the pinned commits. `OFTCore`, `OFTMsgCodec` and `MessagingComposer` are not copied: nothing compiles against them.

- `OFTMsgCodec.encode` writes `sendTo | amountSD | addressToBytes32(msg.sender) | composeMsg` when `composeMsg` is non-empty. The caller of `send` cannot set the third field, which is what makes `composeFrom` an authentication.
- `OFTCore._lzReceive` credits `to`, then calls `endpoint.sendCompose(to, guid, 0, m)` with `m = nonce(8) | srcEid(4) | amountLD(32) | composeFrom(32) | composeMsg`: `composeFrom` at `[44:76]`, the payload at `[76:]`.
- `sendCompose` keys the queue by `msg.sender`, so the `_from` `lzCompose` reports is whoever queued it, and any contract can queue a compose for any target. A destination therefore has to know which `_from` is the real OFT before it trusts a first compose, which is why a conduit's destinations are bound into its address.
- `lzCompose` writes `RECEIVED_MESSAGE_HASH` before it calls the composer, so a success is final and a revert rolls the mark back (P7, R3.5).
- `msgInspector` runs only on the send side, in `_buildMsgAndOptions`.
- An `lzCompose` executor option is `index(2) | gas(16)` under option type 3. The executor refuses options with no `lzReceive` gas (`Executor_ZeroLzReceiveGasProvided`), so an OFT must supply that gas in its enforced options for a send with a compose message.
- `_debitView` refuses a received amount below `minAmountLD` (`SlippageExceeded`). A message-only send of 0 with a minimum of 0 never hits it.
- `PacketV1Codec` puts the nonce at `[1:9]`, the guid at `[81:113]` and the message at `[113:]`.

### Why there is no transceiver

The first design kept LayerZero's transceiver for the bootstrap. It would have put the governor's peering and DVN pins on the path that tells a destination which OFT to trust, so a weak pin there could stand up a receiver bound to the wrong OFT. Without a transceiver the conduit trusts only its own OFT's verification, the endpoint, and immutable code, at the cost of a destination set fixed at creation. An OFT whose address were the same on every chain could avoid that cost, but no assumption here relies on one.

### frxUSD, as checked

Read over public RPCs on 2026-10-25, from the addresses at https://docs.frax.com/protocol/crosschain/addresses:

| Chain | Contract | Endpoint eid | Peers |
| --- | --- | --- | --- |
| Fraxtal (252) | lockbox `0x96A394058E2b84A89bac9667B19661Ed003cF5D4` | 30255 | Ethereum's lockbox, Arbitrum's OFT |
| Ethereum (1) | lockbox `0x566a6442A5A6e9895B9dCA97cC7879D632c6e4B0` | 30101 | Fraxtal's lockbox |
| Arbitrum (42161) | OFT `0x80Eede496655FB9047dd39d9f418d5483ED600df` | 30110 | Fraxtal's lockbox |

All three use the endpoint at `0x1a44076050125825900e736c501f859c50fE728c`. The Fraxtal lockbox's enforced options give a send with a compose message 250,000 lzReceive gas toward both destinations, and it sets no `msgInspector`. Frax's contracts use opcodes after `paris`, so a fork test against them runs under `cancun`. A bootstrap from Fraxtal to Ethereum with the default 1,000,000 lzCompose gas quoted 7.08 FRAX at Fraxtal block 43,069,455.

### Cost

Computed, not measured, except where stated. Against the plain binding a message carries 72 more bytes (`sendTo`, `amountSD`, `composeFrom`) and pays for a second executor step. A bootstrap adds 192 bytes of conduit fields, `ceil(log2 n)` proof words for `n` destinations and 96 bytes of offset and length words. Creation stores two slots per destination and two for the conduit. The destination's gas is measured: a bootstrap through the factory costs 651,291 to 659,121 and a payload 34,246 against mocks, and 649,133 and 42,138 through frxUSD into Ethereum.

*Influence on the design.* An existing OFT is a delivery path that needs no new provider relationship, at the price of a fixed destination set and of trusting the OFT's operator, whose owner, delegate and upgrade admin can stop the conduit's channel.

---

## Parked: NEAR Intents and Chain Signatures

**Not ruled out, but it needs its own research pass before it belongs beside the bindings above.** Explored informally, not source-verified the way §4 to §8 are.

NEAR Intents itself (the `defuse`/Verifier contract plus its PoA token bridge, in `near/intents`) is a swap-settlement ledger, not a message-passing transport: no arbitrary-call delivery, and its one data-carrying primitive (`PoaFactory.ft_deposit`'s `msg`, NEP-141's transfer-and-call) is gated to a small permissioned role and only ever reaches a NEAR contract's callback. It is not a candidate transport in the shape this protocol needs.

The more plausible angle is Chain Signatures itself (`v1.signer`, NEAR's MPC threshold-signing contract, `crates/mpc` in the same repository), which signs an arbitrary payload for a key derived from a NEAR account, for any chain its curve covers: general-purpose, not restricted to token transfers. But it does not fit the transceiver shape at all: there is no message delivered and no gateway to authenticate, since a Chain-Signatures transaction is broadcast directly by whoever holds the signature and is indistinguishable on the destination chain from any other EOA's transaction. Adopting it would mean deciding how a destination chain trusts a NEAR-MPC-derived address as "the conduit" at all: an architecture question upstream of "add a binding", not a peer of the bindings above.

**Not required by, and must not block, the seven bindings that exist.** Recorded here so it is not rediscovered under time pressure. The next step, if ever pursued, is source-verified research in the format above, of `v1.signer` and NEAR's validator threshold-signing scheme itself, not of the Intents/Verifier application layer built on top of it.
