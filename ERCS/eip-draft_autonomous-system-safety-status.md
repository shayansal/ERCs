---
title: Autonomous System Safety Status
description: Issuer-scoped safety dispositions for exact autonomous-system releases, devices, fleets, and capabilities.
author: Shayan Salehi (@shayansal)
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-08-31
requires: 165, 712, 1271, 2098, 5267
---

## Abstract

This ERC defines a registry for issuer-scoped safety status statements about exact autonomous-system releases, devices, fleets, and capabilities. A statement identifies its target in an explicit chain and registry namespace, carries an expiring disposition, links to the prior statement through a strictly increasing sequence, and can commit to restrictions, evidence, remediation, and a replacement target. Statements are signed using typed structured data and may be submitted by relayers. The registry exposes a fail-closed freshness query for contracts and autonomous devices. A statement is attributable to its issuer; it is not universal truth, proof of safety, or a real-time emergency-stop mechanism.

## Motivation

An autonomous system is assembled from components operated by different parties. A model publisher may withdraw a release, a robot manufacturer may recall a device family, a fleet operator may suspend particular units, and a regulator or safety laboratory may restrict a capability. Today these decisions are commonly published through vendor websites, private fleet databases, emails, or application-specific contract flags. A consumer cannot reliably determine which exact artifact a notice covers, whether it has been superseded, or whether a cached response is still fresh.

A single global status would be misleading. Different issuers have different authority, scopes, evidence, and risk tolerances. A manufacturer can state that it recalls a device; the registry cannot establish that the manufacturer is legitimate or that the stated defect exists. Consumers therefore need attributable statements and an explicit policy for which issuers they trust.

Autonomous devices also experience intermittent connectivity. A signed, expiring statement can be checked without trusting the delivery channel, while a monotonic sequence lets a device reject a previously observed status after learning of a newer one. Short validity windows bound, but do not eliminate, the risk of replay while disconnected.

This ERC supplies the missing current-state primitive: exact target identifiers, issuer attribution, linear supersession, bounded freshness, machine-readable disposition, and remedy linkage. It deliberately leaves discovery of legitimate issuers, assessment of evidence, and millisecond-scale physical safety to other systems.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Definitions

An **issuer** is the address signing a status statement. Issuer identity and authority are application policy, not registry conclusions.

A **target** is an exact release, device, fleet, or capability identified within an on-chain registry namespace.

A **disposition** is the issuer's current safety instruction for a target.

A **current statement** is the statement with the greatest accepted sequence for one `(issuer, targetKey)` pair.

A **fresh statement** is current, already effective, unexpired, and no older than a consumer-supplied maximum age.

### Target identifiers

Implementations MUST use these enums and structures:

```solidity
enum TargetKind {
    UNSPECIFIED,
    RELEASE,
    DEVICE,
    FLEET,
    CAPABILITY
}

struct TargetRef {
    TargetKind kind;
    uint256 chainId;
    address registry;
    bytes32 id;
}
```

`chainId` identifies the chain on which `registry` defines `id`. `registry` is the namespace authority, not necessarily the contract implementing this ERC. `id` MAY be a token identifier encoded as `bytes32`, a content commitment, or another immutable identifier defined by `registry`.

`kind` MUST NOT be `UNSPECIFIED`. `chainId`, `registry`, and `id` MUST be nonzero. Registries defining `id` semantics MUST document a deterministic encoding. A mutable name or URI alone MUST NOT be treated as an exact target.

The target key MUST be computed as:

```solidity
bytes32 constant TARGET_REF_TYPEHASH = keccak256(
    "TargetRef(uint8 kind,uint256 chainId,address registry,bytes32 id)"
);

targetKey = keccak256(abi.encode(
    TARGET_REF_TYPEHASH,
    uint8(target.kind),
    target.chainId,
    target.registry,
    target.id
));
```

Equal `id` values in different registry, chain, or kind namespaces identify different targets.

### Dispositions and severity

Implementations MUST use:

```solidity
enum Disposition {
    UNSPECIFIED,
    ACTIVE,
    RESTRICTED,
    SUSPENDED,
    RECALLED
}

enum Severity {
    NONE,
    LOW,
    MEDIUM,
    HIGH,
    CRITICAL
}
```

The numeric order of `Disposition` is increasing restrictiveness. `ACTIVE` means the issuer has published no restrictions under the referenced statement. `RESTRICTED` means operation requires the separately committed restrictions. `SUSPENDED` means operation is temporarily disallowed. `RECALLED` means the issuer directs removal from the applicable service or operating scope.

These values are issuer assertions. They MUST NOT be interpreted as an objective safety score or a legal finding.

### Status statement

```solidity
struct SafetyStatus {
    address issuer;
    TargetRef target;
    uint64 sequence;
    uint64 issuedAt;
    uint64 effectiveAt;
    uint64 validUntil;
    Disposition disposition;
    Severity severity;
    bytes32 reasonCode;
    bytes32 restrictionsHash;
    bytes32 evidenceHash;
    bytes32 remediationHash;
    TargetRef replacementTarget;
    bytes32 previousStatusId;
}
```

`reasonCode` is an application-defined, domain-separated code. `restrictionsHash` commits to the exact restrictions for a `RESTRICTED` status. `evidenceHash` commits to evidence considered by the issuer. `remediationHash` commits to repair, validation, or other remediation information. `replacementTarget` identifies an exact recommended replacement, when one exists. Hash preimages and their availability are outside this ERC.

An absent replacement MUST be the all-zero `TargetRef`: `UNSPECIFIED`, chain ID zero, registry zero, and ID zero. A present replacement MUST satisfy the target-identifier rules. A partially zero replacement is invalid. `replacementTargetKey` elsewhere in this ERC means zero for the absent sentinel and `targetKey(replacementTarget)` otherwise.

### Interface

A compliant registry MUST implement [ERC-165](./eip-165.md) and the following interface:

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

interface IAutonomousSystemSafetyStatus /* is IERC165 */ {
    enum TargetKind { UNSPECIFIED, RELEASE, DEVICE, FLEET, CAPABILITY }
    enum Disposition { UNSPECIFIED, ACTIVE, RESTRICTED, SUSPENDED, RECALLED }
    enum Severity { NONE, LOW, MEDIUM, HIGH, CRITICAL }

    struct TargetRef {
        TargetKind kind;
        uint256 chainId;
        address registry;
        bytes32 id;
    }

    struct SafetyStatus {
        address issuer;
        TargetRef target;
        uint64 sequence;
        uint64 issuedAt;
        uint64 effectiveAt;
        uint64 validUntil;
        Disposition disposition;
        Severity severity;
        bytes32 reasonCode;
        bytes32 restrictionsHash;
        bytes32 evidenceHash;
        bytes32 remediationHash;
        TargetRef replacementTarget;
        bytes32 previousStatusId;
    }

    event SafetyStatusPublished(
        bytes32 indexed targetKey,
        address indexed issuer,
        bytes32 indexed statusId,
        uint64 sequence,
        uint8 disposition,
        uint8 severity,
        uint64 validUntil,
        bytes32 replacementTargetKey
    );

    error InvalidTarget();
    error InvalidDisposition();
    error InvalidTimeWindow();
    error InvalidSequence(uint64 expected, uint64 supplied);
    error InvalidPreviousStatus(bytes32 expected, bytes32 supplied);
    error InvalidIssuerSignature();
    error MissingReasonOrEvidence();
    error MissingRestrictions();
    error MissingRemediation();
    error InvalidReplacement();
    error UnknownStatus(bytes32 statusId);

    function publishStatus(
        SafetyStatus calldata status,
        bytes calldata issuerSignature
    ) external returns (bytes32 statusId);

    function getCurrentStatus(
        address issuer,
        TargetRef calldata target
    ) external view returns (bytes32 statusId, SafetyStatus memory status);

    function getStatus(
        bytes32 statusId
    ) external view returns (SafetyStatus memory status);

    function statusExists(bytes32 statusId) external view returns (bool);

    function currentSequence(
        address issuer,
        bytes32 targetKey
    ) external view returns (uint64);

    function currentStatusId(
        address issuer,
        bytes32 targetKey
    ) external view returns (bytes32 statusId);

    function isFreshAndActive(
        address issuer,
        TargetRef calldata target,
        uint64 maxAge
    ) external view returns (bool);

    function targetKey(
        TargetRef calldata target
    ) external pure returns (bytes32);

    function hashStatus(
        SafetyStatus calldata status
    ) external view returns (bytes32 statusId);
}
```

For consumers that need only the fail-closed current-status queries, the three-function subset is named `IAutonomousSafetyStatusQuery`:

```solidity
interface IAutonomousSafetyStatusQuery /* is IERC165 */ {
    enum TargetKind { UNSPECIFIED, RELEASE, DEVICE, FLEET, CAPABILITY }

    struct TargetRef {
        TargetKind kind;
        uint256 chainId;
        address registry;
        bytes32 id;
    }

    function currentSequence(
        address issuer,
        bytes32 targetKey
    ) external view returns (uint64);

    function currentStatusId(
        address issuer,
        bytes32 targetKey
    ) external view returns (bytes32 statusId);

    function isFreshAndActive(
        address issuer,
        TargetRef calldata target,
        uint64 maxAge
    ) external view returns (bool);
}
```

The full `IAutonomousSystemSafetyStatus` ERC-165 interface identifier is the XOR of its nine function selectors above. Events and errors do not contribute to the identifier. The `IAutonomousSafetyStatusQuery` identifier is `0x468323fd`, the XOR of:

```text
0x497aad21  currentSequence(address,bytes32)
0xf2fe210e  currentStatusId(address,bytes32)
0xfd07afd2  isFreshAndActive(address,(uint8,uint256,address,bytes32),uint64)
```

The registry MUST report support for both the full interface and `0x468323fd` through ERC-165. The duplicate enum and struct declarations above have the same canonical ABI tuple as the full interface and do not change any selector.

### Typed-data encoding and signatures

Status signatures MUST use [EIP-712](./eip-712.md) with this domain:

```text
name: "Autonomous System Safety Status"
version: "1"
chainId: the registry deployment chain
verifyingContract: the registry address
```

The registry MUST expose this domain through [EIP-5267](./eip-5267.md). Domain introspection is not included in the custom interface identifier.

The type hash and struct hash MUST be computed exactly as follows:

```solidity
bytes32 constant SAFETY_STATUS_TYPEHASH = keccak256(
    "SafetyStatus(address issuer,TargetRef target,uint64 sequence,uint64 issuedAt,uint64 effectiveAt,uint64 validUntil,uint8 disposition,uint8 severity,bytes32 reasonCode,bytes32 restrictionsHash,bytes32 evidenceHash,bytes32 remediationHash,TargetRef replacementTarget,bytes32 previousStatusId)TargetRef(uint8 kind,uint256 chainId,address registry,bytes32 id)"
);

replacementTargetStructHash = keccak256(abi.encode(
    TARGET_REF_TYPEHASH,
    uint8(status.replacementTarget.kind),
    status.replacementTarget.chainId,
    status.replacementTarget.registry,
    status.replacementTarget.id
));

statusStructHash = keccak256(abi.encode(
    SAFETY_STATUS_TYPEHASH,
    status.issuer,
    targetKey(status.target),
    status.sequence,
    status.issuedAt,
    status.effectiveAt,
    status.validUntil,
    uint8(status.disposition),
    uint8(status.severity),
    status.reasonCode,
    status.restrictionsHash,
    status.evidenceHash,
    status.remediationHash,
    replacementTargetStructHash,
    status.previousStatusId
));

statusId = keccak256(abi.encodePacked(
    hex"1901",
    domainSeparator,
    statusStructHash
));
```

Each nested `TargetRef` is represented by its EIP-712 struct hash. The primary target's struct hash equals its target key. The absent replacement still hashes its all-zero fields inside `SafetyStatus`; only its event-facing replacement target key is zero. `domainSeparator` is the EIP-712 separator for the domain above.

For an issuer without code, the registry MUST accept canonical low-`s` secp256k1 signatures in 65-byte `(r,s,v)` form and [EIP-2098](./eip-2098.md) 64-byte compact form. In 65-byte form, `v` MUST be 27 or 28. The recovered address MUST equal `issuer`. For an issuer with code, the registry MUST call [ERC-1271](./eip-1271.md) `isValidSignature(statusId, issuerSignature)` and accept only a successful call returning magic value `0x1626ba7e`; a revert or malformed return is invalid. Any account MAY relay a valid statement; `msg.sender` does not acquire issuer authority.

### Publication rules

`publishStatus` MUST perform all of the following checks before changing state:

1. The target satisfies the target-identifier rules.
2. `issuer` is nonzero and `disposition` is not `UNSPECIFIED`.
3. `issuedAt <= effectiveAt <= block.timestamp < validUntil`.
4. The issuer signature validates the computed `statusId`.
5. For a first statement, `sequence == 1` and `previousStatusId == bytes32(0)`.
6. For a later statement, the current sequence is less than `type(uint64).max`, `sequence == current.sequence + 1`, and `previousStatusId == currentStatusId`.
7. `replacementTarget` is the absent sentinel or a valid target whose key differs from the statement's target key.

An `ACTIVE` statement MUST use `Severity.NONE`, `reasonCode == bytes32(0)`, and `restrictionsHash == bytes32(0)`.

A `RESTRICTED`, `SUSPENDED`, or `RECALLED` statement MUST use nonzero `reasonCode`, nonzero `evidenceHash`, and a severity other than `NONE`. A `RESTRICTED` statement MUST additionally use a nonzero `restrictionsHash`.

If a new disposition is numerically less restrictive than the current disposition, `remediationHash` MUST be nonzero. This rule makes an issuer's relaxation attributable to a committed remedy; it does not prove that the remedy works.

On success, the registry MUST reject an already stored `statusId`, store the complete statement immutably under `statusId`, replace the current-statement pointer for the `(issuer, targetKey)` pair, emit `SafetyStatusPublished` with the computed replacement target key, and return `statusId`. No stored statement may be deleted or rewritten.

ERC-1271 signature validation invokes untrusted code. Publication MUST be non-reentrant, or the implementation MUST re-evaluate the current identifier, sequence, previous identifier, and duplicate identifier after signature validation and immediately before its atomic state update.

### Query and freshness semantics

`getCurrentStatus` MUST return the current statement and its `statusId`. If no statement exists, it MUST return `bytes32(0)` and a zero-valued statement.

`getStatus` MUST return the immutable statement stored under `statusId` and revert with `UnknownStatus` when it does not exist. `statusExists` MUST return `false` rather than revert for an unknown ID. Together with `SafetyStatusPublished`, these functions make every accepted statement discoverable and reconstructible.

`currentSequence` MUST return zero when no statement exists.

`currentStatusId` MUST return the current statement ID for `(issuer, targetKey)` and zero when no statement exists. It permits safety gates to compare a previously accepted checkpoint without decoding the full statement.

`isFreshAndActive` MUST return `true` only when all of these conditions hold:

- a statement exists;
- its disposition is `ACTIVE`;
- `effectiveAt <= block.timestamp < validUntil`;
- `maxAge > 0`; and
- `block.timestamp - issuedAt <= maxAge`.

It MUST return `false` rather than revert for a missing, stale, expired, restricted, suspended, or recalled statement. It MUST revert only when `target` itself is invalid. Publication permits only already-effective statements; `effectiveAt` remains in the query for signature reconstruction and defensive validation.

A safety-gating integration claiming conformance with this ERC MUST treat a missing, stale, expired, `SUSPENDED`, or `RECALLED` statement as non-permissive. It MUST also treat `RESTRICTED` as non-permissive unless it implements and validates the restrictions profile committed by `restrictionsHash`.

When several configured targets apply to one operation, such as a release, device, fleet, and capability, the integration MUST evaluate every required `(issuer, target)` pair. A fresh `ACTIVE` statement for one pair MUST NOT override a non-permissive result for another pair.

### Offline checkpoint handling

The EIP-712 statement and issuer signature form a transport-independent checkpoint. An offline consumer claiming conformance MUST verify the domain, issuer, exact target, signature, time window, and its configured `maxAge`. It MUST persist the highest accepted sequence per `(registry deployment, issuer, targetKey)` in rollback-resistant storage and reject lower sequences.

An offline signature does not prove that no higher sequence exists on-chain. A consumer that cannot obtain a sufficiently recent view of registry state MUST enter its locally defined safe state when its checkpoint becomes stale. Local collision avoidance, force limits, protective stops, and emergency-stop circuits MUST continue to operate independently of this registry.

## Rationale

### Issuer-scoped rather than global

The registry records who said what. It does not choose a single authority because a model publisher, manufacturer, fleet operator, laboratory, and public authority may all publish relevant but different statements. Applications can pin one issuer, require several, or apply jurisdiction-specific policy without changing the interface.

### Exact targets and linear supersession

Including kind, chain, registry, and identifier prevents a statement about one model version or device token from silently applying to another. The sequence and previous-status link produce one issuer-controlled chain per target. A simple mutable boolean would erase the path by which a recall was issued and later cleared.

### Expiring status

An indefinite `ACTIVE` value becomes unsafe when an issuer disappears or a delivery path is censored. Mandatory expiry and consumer-selected maximum age force deployments to state their freshness tolerance. They do not solve offline revocation; local safe behavior remains necessary.

### Relationship to adjacent ERCs

[ERC-7992](./eip-7992.md) registers machine-learning model commitments and permits model deprecation. It does not define issuer-scoped status for devices, fleets, or capabilities, nor an expiring current-state query. An ERC-7992 model can be represented as a `RELEASE` target by encoding its `modelId` in the referenced registry namespace.

[ERC-8196](./eip-8196.md) revokes wallet policies and contains on-chain agent actions. [ERC-8226](./eip-8226.md) freezes regulated agent mandates. Those controls govern particular execution authorities; this ERC publishes portable safety disposition about the referenced system target.

[ERC-8328](./eip-8328.md) is a general append-only compliance event log. It can record recall history, but it intentionally does not prescribe safety disposition, strict per-issuer sequence, mandatory expiry, or fail-closed current-state semantics. Implementations can emit or mirror status history into an ERC-8328 log without replacing this registry's current-state query.

[ERC-8320](./eip-8320.md) defines role-gated, versioned claims about regulated assets. A safety statement could be carried in one of its payload schemas, but its maker-checker lifecycle, multiple simultaneously active claims, optional non-expiry, and registry-admin authority do not provide this ERC's direct per-issuer, fail-closed status query. This ERC consequently remains an attributable safety-state primitive rather than a general claim registry.

[ERC-8240](./eip-8240.md) defines trust infrastructure for factual attestations, decision trails, accountability, risk signals, and asset passports. Its aggregate or economic risk signals can inform an issuer's decision, but they do not provide this ERC's exact heterogeneous target coordinate, issuer-scoped linear supersession, mandatory expiry, or signed offline checkpoint. An adapter can translate a risk alert into a safety statement only by naming the issuer, exact target, validity window, and evidence commitment required here.

[ERC-8307](./eip-8307.md) standardizes emergency-state management and discovery for the implementing smart contract, while [ERC-8308](./eip-8308.md) standardizes callable on-chain emergency-response functions and their event trail. [ERC-8343](./eip-8343.md) exposes irreversible deactivation of the implementing contract. Those interfaces describe or change the operational state of a contract; this ERC carries an issuer-attributed, expiring disposition about an exact external model, release, device, fleet, capability, or other target. It neither invokes their emergency functions nor claims their real-time response semantics.

[ERC-8391](./eip-8391.md) exposes token-level lifecycle, reference-market, valuation, and primary-window status for tokenized assets. It reports properties of a tokenized asset through its implementing contract, whereas this ERC permits multiple issuers to publish independently signed, monotonically superseding safety statements about heterogeneous targets. Token status can inform policy, but it does not replace issuer attribution, evidence commitments, freshness, or offline checkpoint verification.

[ERC-7777](./eip-7777.md) defines robot identity and charter interfaces. [ERC-4519](./eip-4519.md) and [ERC-6956](./eip-6956.md) bind tokens to physical assets. Those interfaces can supply target namespaces; none defines an interoperable recall state.

### No emergency-stop semantics

Block production, finality, network access, indexing, and transaction inclusion are too slow or uncertain for real-time motion safety. This ERC belongs in the supervisory control plane: it distributes attributable, versioned safety disposition. It is not a motion controller or emergency stop.

## Backwards Compatibility

This ERC introduces a new interface and does not change existing identity, token, agent-wallet, or model-registry behavior. Existing systems can adopt it through a companion registry and map their identifiers into `TargetRef`. Systems that do not query the registry are unaffected. An integration should not infer conformance merely because an asset or agent implements an adjacent ERC.

## Test Cases

The following cases use a registry deployed on chain `1`, an issuer at `0x1111111111111111111111111111111111111111`, and this release target:

```text
kind     = RELEASE
chainId  = 1
registry = 0x2222222222222222222222222222222222222222
id       = 0xaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
```

1. **First active statement:** Submit sequence `1`, zero previous status, `issuedAt == effectiveAt == 1_800_000_000`, `validUntil == 1_800_003_600`, `ACTIVE`, `NONE`, and zero reason, restriction, and replacement fields at block time `1_800_000_010`. A valid issuer signature returns the EIP-712 digest as `statusId`, emits sequence `1`, and makes `isFreshAndActive(..., 300)` return `true`.
2. **Wrong signer:** Change only the signing key. Publication reverts with `InvalidIssuerSignature`, and sequence remains `1`.
3. **Skipped sequence:** Submit sequence `3` with the current status identifier. Publication reverts with `InvalidSequence(2, 3)`.
4. **Broken chain:** Submit sequence `2` with a nonzero value other than the current status identifier. Publication reverts with `InvalidPreviousStatus`.
5. **Restricted without details:** Submit a correctly signed sequence `2` statement with `RESTRICTED` and zero `restrictionsHash`. Publication reverts with `MissingRestrictions`.
6. **Valid recall:** Submit sequence `2`, the correct previous identifier, `RECALLED`, `CRITICAL`, nonzero reason and evidence commitments, and a complete replacement `TargetRef` whose key differs from the recalled target. The current disposition becomes `RECALLED`; `isFreshAndActive` returns `false` for every maximum age, while `getStatus(statusId)` returns the complete immutable statement.
7. **Unexplained relaxation:** Submit sequence `3` changing `RECALLED` to `ACTIVE` with zero `remediationHash`. Publication reverts with `MissingRemediation`.
8. **Remediated relaxation:** Repeat case 7 with a nonzero remediation commitment and otherwise valid active fields. Publication succeeds and links to sequence `2`.
9. **Stale checkpoint:** At block time `1_800_000_500`, an otherwise current active statement issued at `1_800_000_000` returns `false` for `maxAge == 300` even if its `validUntil` has not elapsed.
10. **Aggregate failure:** A release status is fresh and active, while the configured device status is missing. A conforming safety gate produces a non-permissive result.

Implementations should additionally test sequence overflow, partial-zero replacement rejection, unknown and superseded historical getters, contract issuers using ERC-1271, target namespace separation, exact boundary behavior at `effectiveAt` and `validUntil`, and both advertised ERC-165 interface identifiers.

## Security Considerations

### Issuer authenticity and authority

A valid signature proves control of an issuer address, not that the issuer is the manufacturer, publisher, regulator, owner, or other legitimate authority for a target. Consumers need an independently governed issuer policy. Contract-wallet issuers and threshold-controlled accounts reduce single-key risk.

### Compromised and unavailable issuers

A compromised issuer can publish a false active status or denial-of-service recall. An unavailable issuer cannot renew expiring statements. Expiry intentionally converts silence into a non-permissive result for conforming gates. Deployments should define issuer rotation and emergency recovery outside this interface without allowing a registry administrator to forge issuer signatures.

### Replay, rollback, and censorship

The EIP-712 domain prevents cross-chain and cross-registry replay. On-chain sequence checks prevent an old statement from replacing a new one. Offline consumers remain vulnerable if they never learn that a higher sequence exists, especially after storage rollback. Short validity windows, rollback-resistant sequence storage, redundant chain access, and a local safe state limit this risk. No expiry choice eliminates it.

Transaction censorship or chain unavailability can delay a recall. Safety-critical operators should distribute the same signed statement through independent channels and configure devices to stop or degrade safely when freshness expires.

### Evidence and remedy claims

Evidence, restrictions, and remediation fields are commitments only. The registry does not establish availability, validity, completeness, causal connection, or successful repair. A less restrictive statement with a remediation commitment records issuer accountability but cannot prove the target is safe.

### Identifier substitution

A misleading registry can map a human-readable product name to different bytes over time. Consumers should require immutable, content-addressed release identifiers and authenticated device identifiers. UIs should display chain, registry, kind, and identifier together rather than a name alone.

### Privacy

Device, fleet, vulnerability, and remediation information can be sensitive. Public target identifiers may reveal ownership or location, and low-entropy hashed evidence can be guessed. Deployments should use opaque identifiers and commitments with adequate secret randomness, keep confidential preimages off-chain, and avoid encoding personal or precise location data.

### Gas and denial of service

The fixed-size statement bounds per-call storage and hashing cost. Integrations checking many issuers and target layers can still be made uneconomic by an unbounded policy. Consumer policies should cap the number of required checks. The registry does not iterate across issuers or targets.

### Physical safety boundary

An on-chain result can be stale, censored, reorganized, or unavailable. It cannot sense a person, stop a motor, or validate a physical repair. Local interlocks, certified control logic, watchdogs, collision avoidance, and emergency stops remain authoritative and should default to safe behavior independently of Ethereum.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
