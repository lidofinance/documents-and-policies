# Lido on Ethereum Standard Node Operator Protocol – Validator Exits

```markdown!
STATUS: V3.0
```

## A – Purpose
This Standard Node Operator Protocol (SNOP) outlines
* the general validator exit mechanisms available in Ethereum (see [Appendix A](#appendix-a--ethereum-validator-exit-mechanisms)),
* the role that validator exits play in Lido on Ethereum (hereinafter, "Lido"),
* the scope and rules around the usage of this SNOP,
* the algorithmic validator exit order in Lido,
* the framework for permissionless triggerable withdrawals (TWs) of validators from the protocol,
* the responsibilities Node Operators (NOs) using Lido have in processing validator exit requests, and
* the actions that NOs can expect in case of non-conformance with this SNOP.

In Lido, validator exits may be utilized for the following reasons:
* To satisfy unfinalized withdrawal requests from users
* To allow permissionless NOs exits at their discretion
* To reallocate stake across the NO set
* To rotate signing keys
* To maintain NO target, and Staking Module (SM) share limits
* To eject validators regardless of demand in the [Withdrawal Queue (WQ)](https://docs.lido.fi/contracts/withdrawal-queue-erc721), if
  * the rules and expectations of Lido and this SNOP are disregarded
  * permissionless validators underperform for an extended period of time
  * differences arise between principal and agent of a permissionless operation
  * the access to signing keys is lost
  * the security of signing keys is compromised 

## B – Scope
This SNOP applies to Lido, the NOs who use any of the [Staking Router (SR)'s](https://docs.lido.fi/contracts/staking-router) existing SMs — [Curated Module (CM)](https://operatorportal.lido.fi/modules/curated-module), [Simple Distributed Validator Technology Module (SDVTM)](https://operatorportal.lido.fi/modules/simple-dvt-module), and [Community Staking Module (CSM)](https://operatorportal.lido.fi/modules/community-staking-module) — any future SM or staking avenue, and the validators the NOs run as part of the protocol.

The following operational flows and expectations apply to all above-mentioned SMs and staking avenues, except where alternative processes or expectations are defined due to technical or operational differences.

_NOTE: Terminology may vary in meaning depending on context. The below table aims to clarify the usage of key terms for the purposes of this SNOP and within the context of Lido._

Term|Definition|Context
:---:|---|---
Node Operator|An entity, individual, or cluster thereof that operates nodes using any of the SR's SMs or staking avenues to run Ethereum validators|_Solo_:<br>A single entity or individual that operates validators<br><br>_Cluster_:<br>\* An intra-operator cluster by a single entity or individual utilizing [distributed validator technology (DVT)](https://ethereum.org/en/staking/dvt/#distributed-validator-technology) that operates distributed validators (DVs), or<br>\* An inter-operator cluster by multiple entities and/or individuals utilizing DVT that operates DVs
Validator|A validator — also referred to as a "key" in this SNOP — is the technical component of participating in Ethereum's consensus. If active, it represents an effective balance ranging from 16 up to 2048 ETH (in 1 ETH increments), depending on the corresponding 0x01 or 0x02 withdrawal credentials and whether the validator has been slashed. Each validator is controlled by its public-private validator signing key pair and withdrawal credentials, and performs duties for the network — attesting to and proposing new blocks, participating in sync committees, and reporting network violations of the slashing rules|_Standalone_:<br>A validator run by a solo NO on a standalone node and controlled by an unsplit validator key<br><br>_Distributed_:<br>\* A validator run by a solo NO on a multi-node setup and controlled by an unsplit validator key, and/or<br>\* A validator run by a solo NO with distributed remote key management and controlled by the threshold aggregate of the validator key shares generated at the NO's discretion, or<br>\* A DV run by a cluster utilizing DVT and controlled by the threshold aggregate of the validator key shares generated during a distributed key generation (DKG) event

## C – Validator Exit Infrastructure

### C.1 – Exit Order
The validator exit order in Lido follows a deterministic sequence so that exits can be computed independently and trustlessly if needed. The order is programmatic and, in tandem with the [allocation algorithm](https://docs.lido.fi/contracts/staking-router#allocation-algorithm) that distributes inflows, works to organically balance the protocol stake throughout and across NOs within each SM. By default keys are requested to exit from NOs based on the highest number and longest existence of active validators. Other factors that can play a situational role are the `targetLimit` (a per-NO per-SM property, see section [C.1.1](#c11--target-limit)) and the `stakeShareLimit` (an SM-level property, see section [C.1.2](#c12--stake-share-limit)).

With TWs implemented in the protocol (see section [C.2](#c2--triggerable-withdrawals-framework)), the priority of validator exits is determined by the following [sorting predicate](https://github.com/lidofinance/lido-oracle/blob/develop/src/services/exit_order_iterator.py#L32).

Sorting|Staking Module|Node Operator|Validator
:---:|---|---|---
V||Highest number of targeted validators to boosted exit|
V||Highest number of targeted validators to smooth exit|
V|Highest deviation from the exit share limit (i.e., highest positive difference between `currentShare` and `priorityExitShareThreshold`)||
V||Highest number of active validators|
V|||Lowest index

_NOTE: With the adoption of [consolidations](https://eips.ethereum.org/EIPS/eip-7251#allowing-for-multiple-validator-indices-to-be-combined-through-the-protocol) by Lido, criteria currently normalized to the number of active validators — each representing Ethereum's minimal deposit size of 32 ETH — will be mapped to individual keys’ effective balances instead._

#### C.1.1 – Target Limit
The concept of a `targetLimit` and `targetLimitMode` is applied to NOs on a per-SM basis — i.e., the `targetLimit` is not shared by the same entity across different SMs. The `targetLimitMode` determines the approach through which exits are requested, and the `targetLimit` the target number of active validators for an NO in an SM.

With the `targetLimitMode` set to:
* `0`, i.e., disabled, the NO can receive stake without limitation up to its capacity of vetted depositable keys, and its validators have default exit priority.
* `1`, the NO is in "smooth exit mode", and has a limit to its number of active validators. As long as the NO's number of active validators does not exceed the specified `targetLimit`, the NO may receive stake as per the default allocation algorithm. However, once the value is reached, the distribution of further stake is stopped. If the NO's active keys exceed the `targetLimit`, its validators are prioritized for exit when there is demand in the WQ.
* `2`, the NO is in "boosted exit mode", which largely equates to the “smooth exit mode”, but should the NO's active keys exceed the `targetLimit`, exits will be requested regardless of demand in the WQ and requests be prioritized over those of NOs with a `targetLimitMode` of `0` or `1`.

Individual target limits may be implemented by each SM, and are specified by the Lido Community via [Easy Track motion](https://easytrack.lido.fi/) or [on-chain vote](https://vote.lido.fi/).

#### C.1.2 – Stake Share Limit
Each SM has the concept of a `stakeShareLimit`, `priorityExitShareThreshold` and `currentShare`, where `stakeShareLimit <= priorityExitShareThreshold`:
* `stakeShareLimit` is the maximum percentage of total ETH in Lido that can be distributed to an SM during stake allocation
* `priorityExitShareThreshold` represents the stake share threshold beyond which validator exits from an SM are prioritized
* `currentShare` is defined by the number of currently active validators within an SM

Since the total stake in the protocol is dynamic as a result of inflows and outflows, it is possible that SMs temporarily end up with a `currentShare` higher than their `stakeShareLimit`. The actors monitoring Lido (see section [C.2.1](#c21--validator-exit-reporting)) take this into account when determining the next validators to be exited, and if over-allocated, report that an SM must be prioritized for exits (multiple SMs can be prioritized for exits in this manner simultaneously). Thus, each SM is in one of three states at any given time.

When:
* `currentShare < stakeShareLimit`, an SM has default allocation priority, and validators in the SM do not have extra priority for exit
* `stakeShareLimit <= currentShare <= priorityExitShareThreshold`, the SR does not allocate further stake to an SM, but validators in the SM do not have extra priority for exit
* `priorityExitShareThreshold < currentShare`, the SR does not allocate further stake to an SM, and validators in the SM have an increased priority for exit

### C.2 – Triggerable Withdrawals Framework
In the past, Lido had to rely on validator exit reports being submitted by oracles and to place trust in NOs to process corresponding requests. The triggerable withdrawals framework adds the option of permissionless, secure, and verifiable exits that can be initiated from Ethereum's execution layer (EL) (see [Appendix A.3](#a3--triggerable-withdrawal)). This improves the fault tolerance, reduces trust assumptions and strengthens the foundation for permissionless staking. Following are descriptions of key features and components, the complete specifications can be found in [LIP-30](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-30.md).

#### C.2.1 – Validator Exit Reporting
For the framework, the [Validators Exit Bus Oracle (VEBO)](https://docs.lido.fi/contracts/validators-exit-bus-oracle) — the interface for oracle-driven validator exit reporting — has been incorporated in the [Validators Exit Bus (VEB)](https://docs.lido.fi/guides/oracle-spec/validator-exit-bus), which in turn has taken over the role from the VEBO as the main coordinator for such requests in the protocol. Due to this change, requests can now be submitted by a broader set of actors and via more mechanisms: in addition to the established path — where oracles via [HashConsensus](https://docs.lido.fi/contracts/hash-consensus) reconcile the on-chain states of Lido and the validators run using the protocol, as well as the list of keys requested to exit derived from the Exit Order (see section [C.1](#c1--exit-order)) — e.g., also via Easy Track motions that interface with the VEB and can be called by trusted entities like curated NOs or the [Simple DVT Module Committee (SDVTMC)](https://research.lido.fi/t/simple-dvt-module-committee-multisig/6520) to request ad-hoc exits. Whether received directly or through VEBO, the hashes of all reports are stored in the VEB smart contract, which subsequently waits for the underlying data to be revealed. Not only members of the [Lido Oracle Committee](https://docs.lido.fi/guides/oracle-operator-manual#committee-membership), but anyone is allowed to submit report data matching archived hashes. When data is received and unpacked, the VEB records a timestamp linked to the associated hash, enabling anyone to prove that a validator has been requested to exit and when the corresponding event has been emitted via VEB report.

#### C.2.2 – Triggerable Withdrawal Request Submission
With the framework, a permissionless ejection can theoretically be initiated immediately upon the moment a key has been requested to exit. This ensures that Lido can adequately respond to exceptional situations such as those listed in the Purpose (see section [A](#a--purpose)). Moreover, several [security considerations](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-30.md#6-security-considerations) were made during the design of the framework to implement [limits](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-30.md#appendix-a--limit-implementation) and [parameters](https://github.com/lidofinance/lido-improvement-proposals/blob/develop/LIPS/lip-30.md#7-proposed-params) that harden the protocol against various attack vectors, while keeping TWs suitable for a wide range of use cases and economically sensible. In practice, however, the [demand-adjusted request fee](https://eips.ethereum.org/EIPS/eip-7002#rate-limiting-using-a-fee) renders TWs prohibitively expensive compared to voluntary exits (see [Appendix A.1](#a1--voluntary-exit)), making them a fallback mechanism rather than a practical alternative for general validator exit processing.

A Triggerable Withdrawal Request (TWR) can be called in two ways: permissionlessly by anyone specifying an array of keys that have been requested to exit and providing proof of the corresponding VEB reports, or by an NO electing to eject its validators run for the CSM (or any future permissionless SM or staking avenue) at its discretion. Depending on the case, the TWR must be created through either the VEB or ejector specific to the SM or staking avenue (currently, only the CSM Ejector exists).

The former checks that
* the exit data hashes provided match those stored in the contract,
* the underlying data has been revealed, and
* the corresponding exit events have been emitted via VEB report,

while the latter verifies that
* the requested keys belong to the permissionless SM or staking avenue,
* the keys were deposited via the [Deposit Security Module (DSM)](https://docs.lido.fi/contracts/deposit-security-module),
* there are no key duplicates in the request to avoid drying out limits, and
* there is additional logic implemented to confirm that the one calling the TWR is authorized (i.e., owns the keys or is creating a TWR for validators that are permitted, respectively requested to exit).

The subsequent flow is the same for both cases: valid requests are forwarded to the Triggerable Withdrawal Gateway (TWG) — the smart contract responsible for handling triggerable exit requests in Lido — which then
* enforces frame-level rate limits,
* forwards the request to the [Withdrawal Vault](https://docs.lido.fi/contracts/withdrawal-vault), which interacts with the EIP-7002 precompile to trigger the actual exits,
* notifies the associated SM or staking avenue of the exits through the SR, and
* refunds the requester any potential surplus fee provided.

#### C.2.3 – Late Validator Reporting
A late validator refers to a key that has been requested to exit, for which the responsible NO has failed to initiate the exit processing on Ethereum’s consensus layer (CL) within the time frame afforded by Lido (see section [D.3]()). To allow automated enforcement of consequences for NOs non-conformant with the rules and expectations of the SM and this SNOP, late validators must be reportable on-chain. The triggerable withdrawals framework enables this by allowing anyone to permissionlessly submit to the Validator Exit Delay Verifier a CL proof of a validator's continued active state, along with reference to the VEB report that requested the key to exit. The smart contract then confirms with the VEB that the exit has been requested and calculates the difference between the timestamps of the validator state proof and the exit request event emission. In rare cases, like when a key is requested to exit shortly after activation — even before exit initiation is technically possible — the difference is instead calculated between the timestamp of the validator state proof and the moment the key first becomes eligible to exit. The resulting time delta is passed through the SR to the SM, which then — based on its internal logic — decides whether and what further action is warranted.

#### C.2.4 – Automation
For Lido to remain functional when NOs disregard the rules and expectations of the protocol and this SNOP, and nobody makes use of the permissionless interfaces to resolve the situation, automation is required. The triggerable withdrawals framework introduces two off-chain bots for this purpose, which continuously monitor and interact with the system to ensure a smooth user experience.

#### C.2.4.1 – Validator Late Prover Bot
The Validator Late Prover Bot detects and reports validators for which the responsible NOs have failed to initiate exit processing within the afforded time frames.

The bot:
* Continuously tracks validator exit states on the CL
* Compares the current states to the timestamps of the exit request event emissions
* Generates and submits proof if validators have overstayed their exit window
* Operates permissionlessly, ensuring anyone can contribute to protocol safety
* Depends on SMs to expose public interfaces or logic to validate delays and apply consequences

#### C.2.4.2 – Trigger Exits Bot
To ensure withdrawal requests are not unnecessarily stalled, the Trigger Exits Bot initiates ejections of keys that are verifiably late at exiting.

The bot:
* Scans for validators that have been requested to exit by the VEB
* Monitors whether requests have been completed in the afforded time frames
* Estimates and uses optimal gas prices for efficient execution
* Operates permissionlessly, ensuring anyone can help process exits
* Depends on SMs to expose clear rules or public methods for determining which validators are late
* By submitting TWRs through the EL, serves as fallback mechanism if NOs fail to execute voluntary exits on time

## D – Node Operator Responsibilities
Lido contains on-chain signalling mechanisms that serve to notify NOs when to process validator exits. If exits for keys under their operation have been requested via VEB report, NOs have the responsibility to fully, correctly, and timely exit validators as determined by the rules set by and for any of the reasons described in the Purpose (see section [A](#a--purpose)) and Exit Order (see section [C.1](#c1--exit-order)).

To determine whether NOs have appropriately performed the required actions, this SNOP outlines the operative timelines for exit processing, and specifies consequences of non-conformance. Generally, NOs are expected to exit requested validators in a reasonable time frame. The actual mechanisms for validators to be exited are at the discretion of the NOs, with community-built open-source tooling (see [Appendix B](#appendix-b--tooling)) to aid the NOs in identifying and processing signalled requests being available in addition to any internal tooling, APIs, or manual processes NOs may use.

### D.1 – Out of Order Exits
Out of order exits refer to exits that are performed by an NO of either the CM or SDVTM when a validator exit request has not been signaled via VEB report. In such cases, the NO must notify the community that an out of order exit has been processed, how many and which validators have been exited, and the reason for it. NOs of the CM should communicate this via the [Lido Research Forum](https://research.lido.fi), cluster participants of the SDVTM to the SDVTMC. As the CSM is permissionless and validators are [bonded](https://operatorportal.lido.fi/modules/community-staking-module#block-e4a6daadca12480d955524247f03f380), this expectation is not extended to validators contained therein, CSM NOs are free to exit out of order at their discretion.

### D.2 – Operational Continuity
If, at any time, an NO of either the CM or SDVTM is unable to continue operations, e.g. has become insolvent, the NO must notify the community at least 14 days in advance of taking any further action of the circumstances and signal intent to exit all of the validators run using the protocol. NOs of the CM should communicate this via the Lido Research Forum, cluster participants of the SDVTM to the SDVTMC. If the Lido community does not instruct the NO otherwise — e.g., via an unchallenged Easy Track motion or ratified Snapshot vote — the NO must proceed with triggering the exits within the prescribed time frame.<br>One exception to this could be an inter-operator cluster (see section [B](#b--scope)) that has not yet utilized the one-off option to reshare its validator key shares that in Lido can be allowed with community approval — e.g., to replace a participant unable to continue without having to exit the cluster's active DVs and thus promote operational efficiency without compromising the protocol’s security.

### D.3 – Validator Exit Request Processing
The VEB regularly and on an ad-hoc basis settles on-chain reports that indicate which validators should be exited by which NO, and constitutes the validator exit requests signal. Scheduled reports are published at intervals called frames. Each frame spans a duration defined by a specific number of CL epochs (~6.4 min), with the exact length determined by the instance of the HashConsensus smart contract that the VEB is called by. NOs have the responsibility to set up infrastructure to capture and process signalled exit requests pertaining to validators they run using Lido as soon as possible.

For clarity, a validator exit request is considered:
* `signalled` once it is retrievable from a VEB report,
* `processed` once either a voluntary exit message (VEM) for it has been broadcast to the CL (see [Appendix A.1](#a1--voluntary-exit)) or a TWR to the EL (see [Appendix A.3](#a3--triggerable-withdrawal)), and the corresponding transaction has been included in a block proposed to the Beacon Chain (BC), as well as
* `fulfilled` once the validator has been fully exited and withdrawn.

Although the process can be largely automated, to account for differences in infrastructure, working hours, and mechanism timings, the below are the timing constraints regarding the processing of validator exit requests that NOs must adhere to. If NOs are processing signalled requests as soon as available via VEB report, the shortest possible time for a request to go from `signalled` to `processed` will be somewhere within the range of a few minutes to an hour.

To avoid lateness, NOs of the CM and SDVTM, as well as permissionless NOs of the CSM, are afforded a maximum of 96 h for request processing, while Identified Community Stakers (ICS) are granted 120 h.

## E – Consequences of Non-conformance
In case that an NO is not processing validator exit requests timely, the below actions shall be taken. Separate measures may be applied depending on the technical characteristics of the SM within which the request originated. For example, while NOs are known for the permissioned CM and SDVTM, NOs of the permissionless CSM may remain anonymous and reaching the NOs is only possible if a contact is voluntarily shared with the community.

The following table outlines the consequences for non-adherence to validator exit request timing expectations, and for which SM each action applies.

Action|CM|SDVTM|CSM
---|:---:|:---:|:---:
If an NO has a status of delayed, contributors to the Node Operator Mechanisms (NOM) workstream will raise an issue and request remediative action by the NO.|:heavy_check_mark:|:heavy_check_mark:|If NO is identifiable
If an NO has a status of delayed or delinquent, the VEBO will assume that the NO is unresponsive and reroute newly incoming validator exit requests to NOs that are not considered delayed or delinquent. Due to the rerouting of exit requests, LDO token holders should consider - via an ad-hoc vote - overriding the total limit of active keys for the NO such that if the NO resumes in good standing, the NO is not benefiting at the expense of NOs who took over processing of rerouted exit requests.|:heavy_check_mark:|:heavy_check_mark:|:heavy_check_mark:
If an NO has a status of delinquent, contributors to the NOM workstream will raise a formal issue with the NO on the Lido Research Forum.|:heavy_check_mark:|:heavy_check_mark:|If NO is identifiable
While an NO has a status of delinquent, no new stake will be allocated to the NO.|:heavy_check_mark:|:heavy_check_mark:|:heavy_check_mark:
While an NO has a status of delinquent, the NO's share of rewards for the affected AO frame(s) will halve, with the remainder accruing towards the [stETH rebase(s)](https://docs.lido.fi/guides/lido-tokens-integration-guide) instead.|:heavy_check_mark:|:heavy_check_mark:|:x:
While an NO has a status of delinquent, no operator rewards will accrue to the NO for the affected PO reporting frame(s), with all operator rewards accruing towards the stETH rebase(s) instead. The NO still receives rewards related to its bond rebase.|:x:|:x:|:heavy_check_mark:
Once a delinquent NO has processed all signalled exit requests and its `stuckKeysCount` is updated to `0`, the NO may recommence receiving requests. The NO's validator exit request status will revert to `in good standing` after successful expiry of the NO delinquency cooldown period. During that period, the NO will continue to not be allocated new stake and receive halved rewards.|:heavy_check_mark:|:heavy_check_mark:|:x:
Once a delinquent NO has processed all signalled exit requests and its `stuckKeysCount` is updated to `0`, the NO may recommence receiving requests. The NO's validator exit request status will revert to `in good standing` after successful expiry of the current PO frame. During that frame, the NO will continue to neither be allocated new stake, nor receive operator rewards. The NO still receives rewards related to its bond rebase.|:x:|:x:|:heavy_check_mark:
In the most egregious of cases, e.g. being delinquent for weeks on end, LDO token holders at any time may consider an on-chain vote to stop a misbehaving NO. A stopped NO will neither be allocated new stake, nor receive rewards. In case of a delinquent cluster, the [SDVTM Committee](https://research.lido.fi/t/simple-dvt-module-committee-multisig/6520) may propose an Easy Track motion to request the exit of the cluster's remaining active DVs. If the NO continues to remain unresponsive, or a cluster cannot aggregate enough signatures to act, the NO/cluster is considered to have been effectively off-boarded from the CM or SDVTM as appropriate. LDO token holders should take further steps to formalize the exit and, in the case of an SDVTM cluster, to allow responsive participants to form a new cluster if applicable.|:heavy_check_mark:|:heavy_check_mark:|:x:
In case an NO, for any reason, cannot exit a validator, e.g. due to loss of the associated private signing key, the NO is expected to reimburse stakers by supplying the maximum irretrievable balance of the unrecoverable validator -- i.e. 32 ETH, since anything over that can be obtained via partial withdrawal of rewards. Doing so renders the validator in question reimbursed and does not count against the NO in terms of its validator exit request status.</br>**_NOTE:_** With the Ethereum Pectra upgrade, previously unrecoverable validators will become withdrawable through triggerable exits, reducing the cost for NOs to reimburse stakers to the gas and contract transaction fees needed to call the method on the Execution Layer (EL).|:heavy_check_mark:|:heavy_check_mark:|:x:

# Appendix

## Appendix A – Ethereum Validator Exit Mechanisms
Currently, there are two ways a validator can exit from Ethereum's CL. The first is a voluntary exit, a validator may choose to voluntarily stop performing duties for the network by submitting a voluntary exit message (VEM) to the BC. The second mechanism is forced ejection from the Ethereum protocol -- this could be triggered by slashing or an insufficient [effective balance](https://eth2book.info/capella/part2/incentives/balances/). Additionally, the upcoming [Pectra hardfork](https://eips.ethereum.org/EIPS/eip-7600) will introduce a third method of exiting validators, EL withdrawal credentials triggerable withdrawals.

### A.1 – Voluntary Exit
A validator can initiate a voluntary exit at any time, provided that it has not been slashed, is currently active, and has been active for at least 256 epochs (~27 hours). It does so by signing a VEM using the private validator key, and broadcasting the message to the network to have it processed -- either directly through a local CL client or indirectly, e.g. via an external CL node or API. Once the validator has reached the exit epoch and the NO confirmed that the validator was not selected for [delayed sync committee participation](https://hackmd.io/1wM8vqeNTjqt4pC3XoCUKQ), the validator ceases to perform duties and stops receiving rewards and penalties, i.e. enters the withdrawable state. In this state, the validator waits until its index is parsed by the withdrawals sweep operation and for its balance to be withdrawn to the specified withdrawal credentials.

**_NOTE:_** Validators with [BLS 0x00 credentials](https://notes.ethereum.org/@launchpad/withdrawals-faq#Q-What-are-0x00-and-0x01-withdrawal-credentials-prefixes) will need to rotate to 0x01 for the withdrawal to be processed, otherwise they are skipped.

### A.2 – Forced Exit
Forced exits are not relevant for the purposes of this SNOP, but occur either as a result of a validator being slashed or of its effective balance falling below Ethereum's current [ejection balance](https://github.com/ethereum/consensus-specs/blob/dev/specs/phase0/beacon-chain.md#validator-cycle) threshold of 16 ETH.

### A.3 – Triggerable Withdrawal
[EIP-7002](https://eips.ethereum.org/EIPS/eip-7002) introduces a new functionality to Ethereum that will allow EL withdrawal credentials to signal and initiate partial or full withdrawals of related validators on the CL. This will provide increased assurances with respect to the timely and orderly exit of validators for staking solutions, as well as potential countermeasures against malicious or ill-performing NOs. The downside, however, is that Triggerable Withdrawals (TWs) incur on-chain gas for transactions that do not exist on the CL, as well as variable [request fees](https://eips.ethereum.org/EIPS/eip-7002#fee-calculation), and that the combined costs are partially non-refundable in case of accidental underspending. This makes triggerable withdrawals less suitable than the CL-initiated variant for the average validator exit use case, nor are they a cure-all since validators may still misbehave while in the exit queue. For LoE, the introduction of triggerable withdrawals nevertheless represents a key functional advancement in the offering of bond-based permissionless SMs like the CSM.

## Appendix B – Tooling
There is an assortment of community-built open-source tooling available for LoE and Ethereum staking in general, which may be used by NOs to monitor and perform semi- or fully-automated processing of validator exit requests. NOs may choose to utilize open-source "out of the box" implementations, design custom ones, or any combination thereof -- e.g. connecting the Validator Ejector to a custom API, or generating requests in a just-in-time fashion.

Detailed descriptions and the source codes of these tools can be found on the [Node Operator Portal](https://operatorportal.lido.fi/existing-operator-portal/ethereum-onboarding/no-resources-tooling).
