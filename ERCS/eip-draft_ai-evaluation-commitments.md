---
title: AI Evaluation Commitments
description: Preregistered blinded evaluation plans with immutable completion, abandonment, and expiry records
author: Shayan Salehi (@shayansal)
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-08-31
requires: 165
---

## Abstract

This ERC defines an on-chain commitment protocol for evaluations of registry-identified AI and autonomous-system releases. Before an evaluation may begin, a submitter posts a blinded, content-addressed commitment to a plan containing the dataset commitment, metrics, thresholds, procedure, environment, stopping rule, and result schema. The commitment later reaches exactly one visible terminal state: completed, abandoned, or expired. Completion opens the plan commitment and anchors a result and evidence bundle. The chain proves that an opaque commitment value was posted in a particular order and that a later valid opening binds the revealed plan to that value; it does not prove that the plan bytes existed when the commitment was posted, that the release coordinate is immutable, or that an evaluation was competently designed, honestly executed, or factually correct.

## Motivation

AI evaluations are unusually vulnerable to selective reporting. A model developer can vary datasets, prompts, seeds, thresholds, exclusions, or stopping rules, then publish only the favorable run. A validator can present a passing score without revealing how many plans were abandoned. A buyer, insurer, marketplace, or safety reviewer cannot distinguish an evaluation designed before results were known from a story assembled afterward.

Ordinary signed reports establish authorship but not a shared publication sequence. A hosted registry remains controlled by its administrator. Ethereum provides an append-only state machine that unrelated organizations and contracts can inspect during procurement, assurance, access control, or settlement without publishing private benchmarks on chain.

Preregistration cannot itself make a benchmark sound or an evaluator honest. The useful primitive is narrower: bind one address to one blinded plan, one chain-and-registry-defined release coordinate, one declared run window, and at most one terminal outcome. Negative process outcomes must remain as observable as successful ones.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Terms

- The **submitter** is the address that preregisters and later completes or abandons an evaluation. This ERC authenticates no organization beyond that address.
- A **release coordinate** is `(block.chainid, releaseRegistry, releaseId)`. It identifies a registry entry on the chain where this registry executes. The referenced release registry, not this ERC, determines whether that entry identifies immutable and exact release content.
- A **campaign** is a submitter-chosen public grouping of attempts for one release and purpose.
- A **plan** fixes evaluation choices before the declared earliest run time.
- An **opening** is `(planHash, salt)` whose hash equals the stored plan commitment.
- A **terminal state** is `Completed`, `Abandoned`, or `Expired`.

### Domain values

Implementations MUST compute each domain as `keccak256` of the exact UTF-8 string shown:

```text
EVALUATION_ID_DOMAIN      = "AI_EVALUATION_ID_V1"
CAMPAIGN_KEY_DOMAIN       = "AI_EVALUATION_CAMPAIGN_V1"
PLAN_DOMAIN               = "AI_EVALUATION_PLAN_V1"
PLAN_COMMITMENT_DOMAIN    = "AI_EVALUATION_PLAN_COMMITMENT_V1"
RESULT_DOMAIN             = "AI_EVALUATION_RESULT_V1"
```

All content hashes in this ERC use EVM `keccak256` over exact bytes.

### Canonical plan

The plan bytes MUST be the following ABI encoding:

```solidity
struct EvaluationPlanV1 {
    address releaseRegistry;
    bytes32 releaseId;
    bytes32 campaignId;
    bytes32 supersedesEvaluationId;
    bytes32 questionHash;
    bytes32 datasetCommitment;
    bytes32 metricsHash;
    bytes32 thresholdsHash;
    bytes32 procedureHash;
    bytes32 environmentHash;
    bytes32 stoppingRuleHash;
    bytes32 exclusionsHash;
    bytes32 resultSchemaHash;
    uint64 earliestRunAt;
    uint64 deadline;
}

planBytes = abi.encode(PLAN_DOMAIN, plan);
planHash = keccak256(planBytes);
```

`EvaluationPlanV1` omits a separate chain-ID field because the plan is opened and verified on the evaluation registry's execution chain. Its `releaseRegistry` and `releaseId` fields, together with the current `block.chainid`, encode the complete release coordinate.

Every `bytes32` field in `EvaluationPlanV1` except `supersedesEvaluationId` MUST be nonzero and commit to exact bytes defining its named item. `supersedesEvaluationId` MAY be zero. When it is nonzero, it identifies an earlier evaluation by the same submitter for the same release coordinate and campaign; it neither changes nor invalidates that record. `datasetCommitment` MAY commit to encrypted or sequestered data. `procedureHash` SHOULD cover sampling, randomization, prompting, trial count, and aggregation. `environmentHash` SHOULD cover behavior-affecting evaluator, runtime, tool, hardware, network, and configuration versions. `stoppingRuleHash` and `exclusionsHash` MUST define their rules, including an explicit "none" representation. Metric direction and pass/fail interpretation MUST be fixed by `metricsHash` and `thresholdsHash`.

The plan's release-registry address and release ID, combined with the current `block.chainid`, MUST exactly equal the stored release coordinate. Its campaign ID, earliest run time, and deadline MUST exactly equal the corresponding on-chain record. A plan that fails any of these checks is nonconforming even if its commitment opens successfully.

The submitter MUST choose a nonzero 32-byte salt with at least 128 bits of entropy and compute:

```solidity
planCommitment = keccak256(abi.encode(
    PLAN_COMMITMENT_DOMAIN,
    evaluationId,
    planHash,
    salt
));
```

This salts the plan hash and binds the opening to one evaluation ID. It does not provide information-theoretic secrecy.

### Canonical result

Completed result bytes MUST be encoded as:

```solidity
struct EvaluationResultV1 {
    bytes32 evaluationId;
    bytes32 planHash;
    bytes32 measurementsHash;
    bytes32 verdictHash;
    bytes32 limitationsHash;
    bytes32 evidenceRoot;
}

resultBytes = abi.encode(RESULT_DOMAIN, result);
resultHash = keccak256(resultBytes);
```

Every field MUST be nonzero. `evaluationId`, `planHash`, and `evidenceRoot` MUST equal the values in the on-chain completion record. `measurementsHash` commits to the outputs required by the plan's result schema. `verdictHash` commits to any interpretation; this ERC defines no score scale or pass/fail vocabulary. `limitationsHash` commits to deviations, missing data, uncertainty, and known limitations, including an explicit "none identified" representation when applicable.

The evidence bundle MAY remain encrypted or access-controlled. Its Merkle construction and leaf schema MUST be defined by the bytes committed by `resultSchemaHash`. This ERC treats `evidenceRoot` as opaque.

### Release existence interface

The referenced registry MUST expose:

```solidity
interface IReleaseExistence {
    function releaseExists(bytes32 releaseId) external view returns (bool);
}
```

This selector deliberately makes no claim about the release registry's remaining interface. A consumer MUST evaluate that registry's semantics and authority separately. In particular, "exact release" means only what the selected registry defines: this ERC neither verifies content immutability nor standardizes release manifests.

#### Autonomous Release Lineage adapter

When this ERC is used with a registry conforming to the companion Autonomous Release Lineage proposal, an adapter MUST treat the `manifestHash` in the immutable `Release` record returned for `releaseId` as the manifest commitment for the exact coordinate `(block.chainid, releaseRegistry, releaseId)`. It MUST NOT substitute a URI, publisher assertion, parent record, subject binding, or release digest for that manifest commitment. A consumer claiming exact-release evaluation MUST retrieve the manifest bytes, verify them against that `manifestHash`, and perform the lineage proposal's component-root, count, and dependency-array conformance checks. Registry existence alone establishes only that the record was registered.

### Registry interface

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

interface IAIEvaluationCommitmentRegistry /* is IERC165 */ {
    struct EvaluationPlanV1 {
        address releaseRegistry;
        bytes32 releaseId;
        bytes32 campaignId;
        bytes32 supersedesEvaluationId;
        bytes32 questionHash;
        bytes32 datasetCommitment;
        bytes32 metricsHash;
        bytes32 thresholdsHash;
        bytes32 procedureHash;
        bytes32 environmentHash;
        bytes32 stoppingRuleHash;
        bytes32 exclusionsHash;
        bytes32 resultSchemaHash;
        uint64 earliestRunAt;
        uint64 deadline;
    }

    struct EvaluationResultV1 {
        bytes32 evaluationId;
        bytes32 planHash;
        bytes32 measurementsHash;
        bytes32 verdictHash;
        bytes32 limitationsHash;
        bytes32 evidenceRoot;
    }

    enum EvaluationState {
        None,
        Committed,
        Completed,
        Abandoned,
        Expired
    }

    struct Evaluation {
        address submitter;
        address releaseRegistry;
        bytes32 releaseId;
        bytes32 campaignId;
        bytes32 supersedesEvaluationId;
        bytes32 campaignKey;
        bytes32 planCommitment;
        bytes32 planHash;
        bytes32 resultHash;
        bytes32 evidenceRoot;
        bytes32 reasonHash;
        uint64 submitterNonce;
        uint64 campaignSequence;
        uint64 committedAt;
        uint64 earliestRunAt;
        uint64 deadline;
        uint64 expiredAt;
        uint64 finalizedAt;
        uint64 materializedAt;
        EvaluationState storedState;
    }

    event EvaluationCommitted(
        bytes32 indexed evaluationId,
        bytes32 indexed campaignKey,
        bytes32 indexed releaseId,
        address submitter,
        address releaseRegistry,
        bytes32 campaignId,
        bytes32 planCommitment,
        uint64 submitterNonce,
        uint64 campaignSequence,
        uint64 earliestRunAt,
        uint64 deadline
    );

    event EvaluationCompleted(
        bytes32 indexed evaluationId,
        bytes32 indexed planHash,
        bytes32 indexed resultHash,
        bytes32 evidenceRoot,
        uint64 finalizedAt,
        string planURI,
        string resultURI
    );

    event EvaluationAbandoned(
        bytes32 indexed evaluationId,
        bytes32 indexed planHash,
        bytes32 indexed reasonHash,
        uint64 finalizedAt,
        string planURI
    );

    event EvaluationExpired(
        bytes32 indexed evaluationId,
        uint64 expiredAt,
        uint64 materializedAt
    );

    error InvalidCommitment();
    error InvalidPlan();
    error InvalidRelease();
    error InvalidCampaign();
    error InvalidRunWindow();
    error InvalidOpening();
    error InvalidResult();
    error InvalidURI();
    error UnknownEvaluation(bytes32 evaluationId);
    error UnauthorizedSubmitter(address caller);
    error StaleSubmitterNonce(uint64 expected, uint64 actual);
    error InvalidState(EvaluationState expected, EvaluationState actual);
    error TooEarly(uint64 earliestRunAt);
    error DeadlinePassed(uint64 deadline);
    error DeadlineNotPassed(uint64 deadline);

    function commitEvaluation(
        uint64 expectedSubmitterNonce,
        address releaseRegistry,
        bytes32 releaseId,
        bytes32 campaignId,
        bytes32 planCommitment,
        uint64 earliestRunAt,
        uint64 deadline
    ) external returns (bytes32 evaluationId);

    function completeEvaluation(
        bytes32 evaluationId,
        EvaluationPlanV1 calldata plan,
        bytes32 salt,
        EvaluationResultV1 calldata result,
        string calldata planURI,
        string calldata resultURI
    ) external;

    function abandonEvaluation(
        bytes32 evaluationId,
        EvaluationPlanV1 calldata plan,
        bytes32 salt,
        bytes32 reasonHash,
        string calldata planURI
    ) external;

    function markExpired(bytes32 evaluationId) external;

    function getEvaluation(bytes32 evaluationId)
        external
        view
        returns (
            Evaluation memory evaluation,
            EvaluationState effectiveState
        );

    function evaluationExists(bytes32 evaluationId)
        external
        view
        returns (bool);

    function effectiveStateOf(bytes32 evaluationId)
        external
        view
        returns (EvaluationState);

    function nextSubmitterNonce(address submitter)
        external
        view
        returns (uint64);

    function campaignCount(
        address submitter,
        address releaseRegistry,
        bytes32 releaseId,
        bytes32 campaignId
    ) external view returns (uint64);

    function computeEvaluationId(address submitter, uint64 nonce)
        external
        view
        returns (bytes32);

    function computeCampaignKey(
        address submitter,
        address releaseRegistry,
        bytes32 releaseId,
        bytes32 campaignId
    ) external view returns (bytes32);

    function verifyPlanOpening(
        bytes32 evaluationId,
        EvaluationPlanV1 calldata plan,
        bytes32 salt
    ) external view returns (bool);
}
```

The `IAIEvaluationCommitmentRegistry` interface ID is `0x57bc37f1`, the XOR of these function selectors in declaration order. Solidity tuple notation is used for the two canonical structs:

```text
0x0fff5f1a  commitEvaluation(uint64,address,bytes32,bytes32,bytes32,uint64,uint64)
0x3a6cd277  completeEvaluation(bytes32,(address,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,uint64,uint64),bytes32,(bytes32,bytes32,bytes32,bytes32,bytes32,bytes32),string,string)
0x08b0c9f8  abandonEvaluation(bytes32,(address,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,uint64,uint64),bytes32,bytes32,string)
0x87e0aeb2  markExpired(bytes32)
0xfce1ae6e  getEvaluation(bytes32)
0x1f00cf8c  evaluationExists(bytes32)
0x0fc830c9  effectiveStateOf(bytes32)
0x63bac275  nextSubmitterNonce(address)
0x80b41f86  campaignCount(address,address,bytes32,bytes32)
0xe29dd585  computeEvaluationId(address,uint64)
0x7c3e4249  computeCampaignKey(address,address,bytes32,bytes32)
0x7cfbc6c2  verifyPlanOpening(bytes32,(address,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,uint64,uint64),bytes32)
```

The registry MUST implement [ERC-165](./eip-165.md) and return `true` for `type(IAIEvaluationCommitmentRegistry).interfaceId`.

Where more than one condition fails, this ERC does not prescribe which listed error is returned.

### Commitment

`computeEvaluationId` MUST return:

```solidity
keccak256(abi.encode(
    EVALUATION_ID_DOMAIN,
    block.chainid,
    address(this),
    submitter,
    nonce
))
```

`computeCampaignKey` MUST return:

```solidity
keccak256(abi.encode(
    CAMPAIGN_KEY_DOMAIN,
    block.chainid,
    address(this),
    submitter,
    releaseRegistry,
    releaseId,
    campaignId
))
```

`commitEvaluation` MUST reject zero `releaseRegistry`, `releaseId`, `campaignId`, or `planCommitment`. `expectedSubmitterNonce` MUST equal `nextSubmitterNonce(msg.sender)` or the call MUST revert with `StaleSubmitterNonce`. This compare-and-set rule lets the submitter compute the evaluation ID used in the blinded commitment without a concurrent transaction silently consuming another nonce. The registry MUST require code at `releaseRegistry` and perform a `staticcall` to `releaseExists(releaseId)`. The call MUST return exactly one canonical ABI-encoded `true`; a revert, malformed return, or `false` MUST cause `InvalidRelease`.

`earliestRunAt` MUST be strictly greater than the current `block.timestamp`, and `deadline` MUST be strictly greater than `earliestRunAt`. The deadline is exclusive: submitter finalization is allowed only while `block.timestamp < deadline`. The current time, both supplied times, the submitter nonce, and campaign count MUST fit in `uint64` without overflow.

The registry MUST derive `evaluationId` using `msg.sender` and the accepted nonce, reject the ID if it already exists, then increment that nonce. It MUST increment the count for the derived campaign key and store the new value as the one-based `campaignSequence`. It MUST set `committedAt` to `uint64(block.timestamp)`, `expiredAt` to `deadline`, `storedState` to `Committed`, `supersedesEvaluationId`, all terminal hashes, `finalizedAt`, and `materializedAt` to zero, store the remaining inputs, and emit exactly one `EvaluationCommitted`. No function may delete an evaluation or alter its release coordinate, campaign, commitment, nonce, sequence, run window, or `expiredAt`.

### Completion

Only the stored submitter MAY call `completeEvaluation`. The stored state MUST be `Committed`; `block.timestamp` MUST be at least `earliestRunAt` and strictly less than `deadline`. `salt` MUST be nonzero. The registry MUST validate every required plan field, require the plan's release coordinate, campaign, earliest time, and deadline to equal the stored values, then compute:

```solidity
planHash = keccak256(abi.encode(PLAN_DOMAIN, plan));
```

The registry MUST use that computed hash and the supplied salt to reproduce `planCommitment`. If `plan.supersedesEvaluationId` is nonzero, the referenced evaluation MUST exist, MUST differ from the current evaluation, MUST have the same submitter, release coordinate, and campaign, and MUST have a lower `submitterNonce`. This reference does not transition the earlier record.

The registry MUST validate every `EvaluationResultV1` field, require `result.evaluationId == evaluationId`, `result.planHash == planHash`, and then compute:

```solidity
resultHash = keccak256(abi.encode(RESULT_DOMAIN, result));
```

Each URI's UTF-8 byte length MUST be no greater than 2,048; either URI MAY be empty when bytes are delivered out of band.

On success the registry MUST store the computed `planHash` and `resultHash`, `result.evidenceRoot`, and `plan.supersedesEvaluationId`; set `storedState` to `Completed`; set both `finalizedAt` and `materializedAt` to `uint64(block.timestamp)`; and emit exactly one `EvaluationCompleted`. It MUST NOT accept another terminal transition.

The bytes obtained from nonempty `planURI` and `resultURI` MUST hash to `planHash` and `resultHash`, respectively, for a consumer to treat them as conformant. URI resolution is not performed by the contract.

### Abandonment

Only the stored submitter MAY call `abandonEvaluation`. The stored state MUST be `Committed`, and `block.timestamp` MUST be strictly less than `deadline`. `salt` and `reasonHash` MUST be nonzero. The registry MUST perform the same full-plan validation, plan-hash computation, opening check, and nonzero supersession-reference validation required by `completeEvaluation`. `planURI` MAY be empty and otherwise MUST be no more than 2,048 UTF-8 bytes.

On success the registry MUST store the computed `planHash`, `reasonHash`, and `plan.supersedesEvaluationId`; set `storedState` to `Abandoned`; set both `finalizedAt` and `materializedAt` to `uint64(block.timestamp)`; and emit exactly one `EvaluationAbandoned`. `reasonHash` commits to exact bytes explaining the abandonment; the registry does not constrain their schema. Abandonment MUST NOT be represented as completion or expiry.

### Expiry and views

Any address MAY call `markExpired` when the stored state is `Committed` and `block.timestamp >= expiredAt`. It MUST set `storedState` to `Expired`, set `finalizedAt` to `expiredAt`, set `materializedAt` to `uint64(block.timestamp)`, and emit exactly one `EvaluationExpired(evaluationId, expiredAt, materializedAt)`. It MUST NOT reveal or modify the plan commitment.

An unmarked record is effectively expired beginning at `expiredAt`, which equals the committed `deadline`. `effectiveStateOf` and the second return value of `getEvaluation` MUST return `Expired` whenever `storedState` is `Committed` and `block.timestamp >= expiredAt`. Otherwise they MUST return `storedState`. Before `markExpired`, its stored state remains `Committed`, `finalizedAt` and `materializedAt` remain zero, and `expiredAt` supplies the logical terminal time. `markExpired` exists to create a terminal event and materialize that state for downstream contracts without changing the logical expiry time.

`getEvaluation` and `effectiveStateOf` MUST revert with `UnknownEvaluation` for an absent ID. `evaluationExists` MUST return `false` for one. `verifyPlanOpening` MUST recompute the plan hash from the supplied struct and apply the same plan-field, duplicated-record, and supersession checks as finalization. It MUST return `false`, not revert, for an absent evaluation, zero salt, invalid plan, invalid supersession reference, or commitment mismatch.

### State transition table

| Stored state | Caller | Condition | Next state |
| --- | --- | --- | --- |
| `None` | anyone | valid commitment | `Committed` |
| `Committed` | submitter | valid plan, result, opening, and `earliestRunAt <= now < deadline` | `Completed` |
| `Committed` | submitter | valid plan and opening, and `now < deadline` | `Abandoned` |
| `Committed` | anyone | `now >= expiredAt` | `Expired` |
| any terminal state | anyone | any transition | revert |

## Rationale

### Submitter-authored records

The committing address alone can report completion or abandonment, avoiding false attribution through an unaccepted evaluator field. Multiple parties can use a jointly authorized contract account or name participants inside the plan.

### Blinding with visible negative states

Publishing a detailed plan before a safety evaluation can contaminate a benchmark or expose proprietary data. A salted commitment proves only that an opaque commitment value was posted at a particular on-chain time and position. A later valid opening binds the revealed canonical plan hash to that prior value and evaluation ID. It does not prove that the plan bytes existed, were possessed, or were complete when the commitment was posted. Completion and voluntary abandonment open the commitment. Involuntary expiry preserves the unopened commitment because nobody can recover a submitter's salt. In all cases the attempt and its campaign sequence remain visible.

### Campaigns expose repeated attempts

The one-based sequence exposes repeated registrations under one submitter, release, and campaign. It cannot stop address or campaign changes; policies can require a recognized submitter and previously disclosed campaign ID.

### Separation from scoring and proof systems

[ERC-8126](./eip-8126.md) defines categories and risk scores for AI agent verification. This ERC defines neither; it can preregister any evaluation methodology. [ERC-7992](./eip-7992.md) verifies a proof of a particular model inference, while an evaluation may involve many trials, human judgments, physical behavior, or non-provable safety properties. Such a proof can be one evidence leaf without replacing preregistration.

[ERC-8004](./eip-8004.md) provides agent-linked validation requests and permits multiple validator responses for progressive or updated status. This ERC is a separate registry because it is release-coordinate based, submitter-authored, plan-blinded, and single-finalization, and because abandonment and automatic expiry are first-class outcomes. An adapter can place an evaluation ID and terminal result hash in an ERC-8004 request or response, but ERC-8004 conformance alone does not satisfy this ERC's preregistration or lifecycle rules.

[ERC-8220](./eip-8220.md) defines interfaces for registering, governing, and evaluating AI-agent compliance. [ERC-8294](./eip-8294.md) defines operator-diverse validation networks that plug into ERC-8004. Those proposals coordinate governance, compliance, validators, or their outputs; they do not establish this ERC's submitter-authored blinded plan, run window, campaign sequence, or visible `Completed`, `Abandoned`, and `Expired` outcomes. Their decisions or validator aggregates can be referenced as evaluation evidence without replacing the preregistration lifecycle.

[ERC-8274](./eip-8274.md) defines interfaces for verifying AI inference proofs. [ERC-8281](./eip-8281.md) anchors observation commitments for later verification. [ERC-8299](./eip-8299.md) binds received input, its sanitization pipeline, and the input actually executed. [ERC-8354](./eip-8354.md) proves a pre-execution allow or deny verdict against a policy that remains confidential. Each can supply a procedure, input-provenance, policy-verdict, observation, proof, or evidence artifact to an evaluation. None records that an opaque evaluation plan was committed before a bounded run or makes abandonment and expiry part of the public record.

[ERC-8273](./eip-8273.md) lets an authorized attestor atomically issue and consume transaction-scoped authorization for an agent action while retaining an audit record with an evidence hash. This ERC neither gates an action nor authorizes attestors. A completed evaluation's ID or result hash can serve as ERC-8273 evidence under an integration-defined capability, while this registry preserves the earlier blinded plan and negative outcomes. The two lifecycles are therefore complementary rather than interchangeable. [ERC-7512](./eip-7512.md) represents audit reports; this ERC records the earlier experimental commitment and its lifecycle rather than asserting that an audit occurred.

*Recomputable Verification Receipts*, currently a working proposal in PR #1980, defines content-addressed identities for claims, evidence sets, verification profiles, canonical results, and independent recomputation outcomes. A receipt or recomputation-result digest can be committed through this ERC's result and evidence fields. That proposal addresses reproducibility after artifacts exist; this ERC addresses prior blinded registration, run windows, and exactly one visible terminal state, so neither lifecycle substitutes for the other.

[ERC-8299](./eip-8299.md), also known as WYRIWE ("What You Read Is What You Execute"), explains how an executed input relates to what was received. Its commitments or typed attestations can be included in a plan's procedure, environment, or evidence schema. This ERC instead explains when an opaque evaluation commitment was posted and how a later plan opening and outcome relate to it.

### Hashes and events rather than on-chain reports

Fixed hashes keep large or confidential materials off chain. URIs aid discovery but are not authoritative. Stored terminal hashes remain directly usable by downstream contracts.

### Strict single finalization

Updated results would recreate outcome shopping under one ID. Corrections and reruns therefore require a new commitment. `supersedesEvaluationId` gives that new plan a machine-readable pointer without mutating or hiding the prior outcome.

## Backwards Compatibility

This ERC changes no existing interface. Evaluation services can adopt it by committing before execution and hashing their report formats into the plan and result envelopes. Existing registries can reference evaluation IDs without contract changes.

## Test Cases

In the following cases, `H(x)` means `keccak256(bytes(x))`. Alice is the submitter, release R exists in registry G, current time is 1,000, and Alice's next nonce is zero.

1. Alice computes `evaluationId` for nonce zero and commits to a canonical plan P with expected nonce zero, campaign `H("red-team-q3")`, earliest time 1,100, deadline 1,500, zero `supersedesEvaluationId`, and a valid salt. The record is `Committed`, `expiredAt` is 1,500, campaign sequence is one, committed time is 1,000, and Alice's nonce becomes one.
2. A commit with earliest time 1,000, deadline no greater than its earliest time, zero campaign, a registry with no code, an unknown release, or a `releaseExists` call returning 32-byte value `2` reverts.
3. At time 1,099, completion reverts. At 1,100, Alice supplies the full P and a full canonical result. The registry recomputes both hashes, verifies all duplicated fields and the opening, stores the result's evidence root, and reaches `Completed`; `finalizedAt` and `materializedAt` equal 1,100. Another terminal call reverts.
4. Completion with a different salt, any changed plan field, a result naming another evaluation ID or plan hash, or a supplied result with any zero field reverts. A caller cannot select an unrelated `resultHash` or `evidenceRoot` because both are derived from the supplied result struct.
5. Alice's second commitment using expected nonce one and the same release and campaign receives sequence two and a different evaluation ID. A transaction still expecting nonce zero reverts. The second plan may name the first ID in `supersedesEvaluationId`; an unknown ID, later nonce, different submitter, release, or campaign is rejected when the plan is opened.
6. Before its deadline, Alice abandons a committed evaluation with the full plan, correct salt, and `reasonHash = H("fixture leak")`. The recomputed plan hash and reason hash are stored, result fields remain zero, and `finalizedAt` equals `materializedAt`. A zero reason or invalid plan opening reverts.
7. For a record with deadline and `expiredAt` 1,500, `effectiveStateOf` returns `Expired` at exactly 1,500 while `storedState` remains `Committed` and both finalization timestamps remain zero. Any caller may then call `markExpired`; the stored state becomes `Expired`, `finalizedAt` is 1,500, and `materializedAt` is the marking time.
8. Completion and abandonment are permitted through time 1,499 and not at 1,500. Expiry is unavailable at 1,499 and effective at 1,500.
9. Empty plan and result URIs are accepted on completion, but a consumer cannot claim to have retrieved their bytes from the event. A 2,049-byte URI reverts with `InvalidURI`.

## Security Considerations

### Limited claim

The chain proves that an address posted one opaque commitment value before a declared time and could not replace that record's terminal outcome. A later valid opening computationally binds the revealed canonical plan hash to that prior value under the collision and second-preimage resistance of `keccak256`; it does not prove that the plan bytes existed or were possessed when the value was posted. It also does not prove that evaluation began after `earliestRunAt`, that the committed release was actually tested, that data were unmodified, that measurements are accurate, or that a verdict follows from them. Contracts using results must establish evaluator trust or verify evidence separately.

### Cherry-picking outside the registry

A submitter can omit experiments, change addresses or campaigns, or test privately first. Sequencing exposes only attempts under the same key. High-assurance policies should prescribe the submitter, release, campaign, minimum lead time, and evidence access.

### Weak salts and plan guessing

Low-entropy salts allow dictionary attacks against predictable plans, datasets, or thresholds. A nonzero check is not an entropy check. Submitters must generate salts using a cryptographically secure random source and protect them until finalization. Losing a salt makes completion and voluntary abandonment impossible, leaving expiry as the only outcome.

### Timestamp assumptions

Run windows use `block.timestamp`. Validators or sequencers can influence timestamps within the underlying chain's rules, and different chains have different finality and reorganization properties. Policies should require a lead time and confirmation depth proportionate to risk. A one-second on-chain lead does not demonstrate meaningful preregistration in an off-chain process.

### Malicious release registries

The existence check proves only that the specified contract returned `true`. A malicious or upgradeable registry can lie or later change semantics. Consumers must authenticate the release-registry deployment and retain the full `(chainId, registry, releaseId)` coordinate. The evaluation registry deliberately does not import another registry's trust model.

### Content availability and equivocation off chain

An event URI can disappear, deny access, or serve different bytes. Only matching content is valid. Reviewers should retain exact bytes; commitments do not create availability.

### Benchmark leakage and privacy

Opening the plan hash and salt can enable confirmation of guessed plan bytes even when `planURI` is empty. Completed and abandoned evaluations should not open commitments whose plan encoding contains secrets in plaintext. Sensitive elements should appear as hiding commitments or encrypted-content hashes. Public metadata, timing, campaign grouping, and the existence of an evaluation may themselves be sensitive.

### Flooding and misleading records

Attackers can create many campaigns or misleading off-chain labels. Integrators must key trust to the submitter address and registry, not a display name. Indexers should preserve terminal states for recognized campaigns.

### URI and parsing denial of service

Referenced documents can still be hostile or enormous. Consumers should bound bytes, nesting, decompression, and proofs, decode ABI strictly, and compare duplicated on-chain fields.

### Contract implementation safety

Terminal state must precede any implementation-specific external interaction. Release checks must be static and strictly decoded; counters must revert rather than wrap. Upgradeable deployments should disclose authority and migration behavior.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
