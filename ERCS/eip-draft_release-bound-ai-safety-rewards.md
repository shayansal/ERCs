---
title: Release-Bound AI Safety Rewards
description: Prefunded rewards for confidential findings bound to immutable AI releases.
author: Shayan Salehi (@shayansal)
discussions-to: https://ethereum-magicians.org/t/placeholder-release-bound-ai-safety-rewards/0
status: Draft
type: Standards Track
category: ERC
created: 2026-08-31
requires: 20, 165
---

## Abstract

This proposal defines escrow and settlement for safety findings against one exact artificial intelligence (AI) release coordinate and manifest commitment. A sponsor fixes the terms, resolver, confidential contact, and fully funded finite severity schedule at creation.

Researchers establish priority before private disclosure with a salted commitment to the ciphertext hash and claim fields. The resolver acknowledges delivery, then rejects the finding or reserves an award at its selected severity. Reserved funds are immediately payable by any account without publishing the vulnerability.

## Motivation

AI systems combine models, prompts, tools, permissions, data, policies, and software. A finding is meaningful only against a precise release, yet conventional bounty pages can change scope, rewards, adjudicator, or coverage after work begins.

Publishing before remediation can enable policy bypass, unauthorized tool use, unsafe robotic behavior, or data loss. Private reporting instead leaves the researcher dependent on the sponsor's timestamp and willingness to pay.

Ethereum places ordering, immutable terms, reserved funds, and permissionless settlement in one public state machine. It does not decide correctness, severity, or duplication; those remain confidential judgments of the identified resolver.

This proposal does not define general labor bounties, AI identity, a universal severity taxonomy, report hosting, resolver governance, or proof of an off-chain finding.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Definitions

- **Sponsor**: `msg.sender` of program creation, which supplies funding and receives unused funds.
- **Release coordinate**: `(releaseChainId, releaseRegistry, releaseId)`, identifying one registry entry on one chain.
- **Release reference key**: the canonical hash of a release coordinate defined below.
- **Release manifest**: canonical bytes describing the exact covered AI release.
- **Release commitment**: `keccak256(releaseManifestBytes)`.
- **Terms**: canonical bytes describing the program rules.
- **Terms hash**: `keccak256(termsBytes)`.
- **Severity tier**: an immutable identifier, reward amount, and maximum number of awards.
- **Reporter**: the account that establishes a finding's on-chain priority.
- **Committed beneficiary**: the payout address concealed inside the priority commitment and later disclosed to the resolver.
- **Beneficiary**: the current payout address, initially equal to the committed beneficiary.
- **Resolver**: the immutable account or contract designated to evaluate reports.
- **Contact provider**: a contract publishing immutable, versioned encryption contact snapshots.
- **Priority index**: a monotonically increasing integer, starting at one, assigned within a program.

### Release, Terms, and Program Identity

A release coordinate MUST have nonzero `releaseChainId`, `releaseRegistry`, and `releaseId`. The referenced registry MUST define `releaseId` as an immutable entry or immutable content commitment. A mutable name, URI, tag, or product version alone is nonconforming. Implementations MUST compute:

```text
RELEASE_REF_TYPEHASH = keccak256(
    "ReleaseRef(uint256 chainId,address registry,bytes32 id)"
)

releaseRefKey = keccak256(abi.encode(
    RELEASE_REF_TYPEHASH,
    releaseChainId,
    releaseRegistry,
    releaseId
))
```

When the coordinate references an Autonomous Release Lineage registry, `releaseId` MUST be its stored release identifier and `releaseCommitment` MUST equal that release record's `manifestHash`. Another registry remains usable if its documented immutable record identifies the same exact manifest bytes; the terms MUST identify the registry profile and the deterministic procedure for resolving its manifest commitment.

The release manifest MUST distinguish the covered release from every other release. For a model-backed agent or robot, it SHOULD commit to its model or service version, code, policy, tool permissions, and safety-relevant configuration. The terms MUST describe its canonical encoding.

The terms MUST describe scope, prohibited testing, severity and duplicate rules, reporter eligibility, evidence, coordinated disclosure, resolver policy, and decision encoding. They MAY add obligations but MUST NOT contradict on-chain state or funding.

`releaseURI` and `termsURI` MUST be nonempty immutable UTF-8 URI strings from which a client can retrieve the corresponding bytes. Retrieval is not trusted: a client MUST hash the retrieved bytes and compare them with `releaseCommitment` or `termsHash`. URI availability or mutable content at a URI cannot change the committed bytes.

A program identifier MUST be derived as follows:

```text
PROGRAM_TYPEHASH = keccak256(
    "ReleaseSafetyProgram(uint256 chainId,address escrow,address sponsor,uint256 sponsorNonce,bytes32 releaseRefKey,bytes32 releaseCommitment,bytes32 termsHash,address token,address resolver,address contactProvider,uint8 contactType,uint64 submissionDeadline,uint64 resolutionDeadline,bytes32 tierScheduleHash)"
)

tierScheduleHash = keccak256(abi.encode(tiers))

programId = keccak256(abi.encode(
    PROGRAM_TYPEHASH,
    block.chainid,
    address(escrow),
    sponsor,
    sponsorNonce,
    releaseRefKey,
    releaseCommitment,
    termsHash,
    token,
    resolver,
    contactProvider,
    contactType,
    submissionDeadline,
    resolutionDeadline,
    tierScheduleHash
))
```

An escrow MUST reject reuse of a `sponsorNonce` by the same sponsor. Creation MUST validate the release coordinate, require the supplied `releaseRefKey` to equal the recomputed value, and require nonzero commitments, token, resolver, and contact provider; a resolver address different from the sponsor; and:

```text
block.timestamp < submissionDeadline < resolutionDeadline
```

Different sponsor and resolver addresses expose a separation of addresses, not proof of independent control.

Before accepting funds, creation MUST verify [ERC-165](./eip-165.md) support for `IReleaseSafetyContact`, query a nonzero current version, and validate its identifiers and validity interval.

On success, the escrow MUST store the exact `ProgramData`, emit one `ProgramCreated`, and emit `TierConfigured` for every tier in input order. No operation, upgrade, or administrator may rewrite a program field, redirect its obligations, or bypass payout.

### Severity Schedule, Token, and Funding

The maximum number of tiers is fixed:

```text
MAX_TIERS = 32
```

Creation MUST include between one and `MAX_TIERS` tiers. `maxTierCount()` MUST return `32`. Each tier is:

```solidity
// SPDX-License-Identifier: CC0-1.0
struct SeverityTier {
    bytes32 severityId;
    uint256 reward;
    uint32 maxAwards;
}
```

Within a program, `severityId` MUST be nonzero and unique. `reward` and `maxAwards` MUST be nonzero. The escrow MUST preserve input order and expose every identifier through `tierCount()` and `tierAt()`.

The program token MUST conform to [ERC-20](./eip-20.md), transfer exactly the requested amount, charge no transfer fee, and not rebase balances. At creation, the escrow MUST collect:

```text
totalFunded = sum(tier.reward * tier.maxAwards)
```

from the sponsor. Arithmetic overflow MUST revert. The call MUST revert unless the escrow balance increases by exactly `totalFunded` and the sponsor balance decreases by exactly `totalFunded`. The escrow MUST use a reentrancy guard.

For each token, the escrow MUST maintain:

```text
tokenObligation[token] =
    sum(availableAmount + reservedAmount for all programs using token)
```

Creation increases this obligation by `totalFunded`. A successful payout or refund decreases it by the transferred amount. After every successful state-changing call, the escrow token balance MUST be at least `tokenObligation[token]`. Tokens sent directly to the escrow are excess assets and MUST NOT increase any program's funding, award capacity, or refundable amount.

Funding is segregated by tier. Reserving an award consumes one raw available slot in its awarded tier; funds from one tier MUST NOT satisfy another. For each program:

```text
totalFunded = availableAmount + reservedAmount + paidAmount + refundedAmount
```

`availableAmount` and `reservedAmount` are current obligations. `paidAmount` and `refundedAmount` are cumulative history.

For every tier, `maxAwards` MUST equal `availableAwards + reservedAwards + paidAwards + refundedAwards` as returned by `tier()`.

### Versioned Confidential Contact

A contact provider MUST expose immutable historical snapshots:

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

struct SafetyContactSnapshot {
    uint8 contactType;
    uint64 keyVersion;
    uint64 validFrom;
    uint64 acceptUntil;
    bytes publicKey;
    bytes deliveryData;
}

interface IReleaseSafetyContact {
    event SafetyContactPublished(
        bytes32 indexed programId,
        uint8 indexed contactType,
        uint64 indexed keyVersion,
        bytes32 contactKeyHash,
        uint64 validFrom,
        uint64 acceptUntil
    );

    function currentContactVersion(bytes32 programId, uint8 contactType)
        external view returns (uint64);

    function safetyContact(
        bytes32 programId,
        uint8 contactType,
        uint64 keyVersion
    ) external view returns (SafetyContactSnapshot memory);
}
```

For a `(programId, contactType, keyVersion)`, a published snapshot MUST never change or disappear. Versions MUST start at one and increase by one within each `(programId, contactType)` pair. The provider MUST implement [ERC-165](./eip-165.md) and report `0x7c5ed4f4` for `IReleaseSafetyContact`.

The snapshot hash is:

```text
contactKeyHash = keccak256(abi.encode(
    snapshot.contactType,
    snapshot.keyVersion,
    snapshot.validFrom,
    snapshot.acceptUntil,
    snapshot.publicKey,
    snapshot.deliveryData
))
```

When committing a finding, the escrow MUST query the provider, require the supplied version to equal `currentContactVersion`, require all returned identifiers to match, recompute `contactKeyHash`, and require:

```text
snapshot.validFrom <= block.timestamp
snapshot.acceptUntil >= program.resolutionDeadline
```

The provider operator MUST retain the corresponding private key and delivery capability through `acceptUntil`. `deliveryData` is public routing data for a confidential channel and MUST NOT contain a secret.

Every implementation MUST support `contactType == 1`. It uses HPKE base mode from [RFC 9180](https://www.rfc-editor.org/rfc/rfc9180) with DHKEM(X25519, HKDF-SHA256), HKDF-SHA256, and AES-256-GCM. `publicKey` MUST contain the 32-byte X25519 public key. `deliveryData` MUST contain the UTF-8 bytes of an absolute HTTPS URI with no user information or fragment. The HPKE `info` value is the UTF-8 bytes `RELEASE_SAFETY_FINDING_V1`. The authenticated associated data is:

```text
abi.encode(
    block.chainid,
    address(escrow),
    programId,
    releaseRefKey,
    releaseCommitment,
    reporter,
    committedBeneficiary,
    claimedSeverityId,
    contactKeyHash
)
```

The `releaseRefKey` and `releaseCommitment` in this associated data and in the priority commitment below MUST equal the stored program fields.

The `ciphertextHash` is `keccak256(enc || ct)`, where `enc` and `ct` are the HPKE encapsulated key and ciphertext encodings. For contact type 1, the reporter MUST use HTTP `POST` to send this exact ABI-encoded confidential delivery envelope to `deliveryData`:

```text
abi.encode(
    findingId,
    programId,
    reporter,
    committedBeneficiary,
    claimedSeverityId,
    salt,
    keyVersion,
    contactKeyHash,
    enc,
    ct
)
```

The request body MUST be exactly those ABI-encoded bytes and its `Content-Type` MUST be `application/octet-stream` with no parameters. A client MUST NOT follow redirects; every `3xx` is a delivery failure. Through `acceptUntil`, the endpoint MUST use `findingId` as its idempotency key and return an empty `202 Accepted` after accepting a new envelope or byte-identical retry. It MUST return `409 Conflict` for a different body using that ID; every other status is a failure. After a transport failure or missing `202`, the reporter MAY retry the identical body while the contact remains valid, but MUST NOT re-encrypt or create another finding merely to retry. HTTP `202` proves neither decryption nor resolver review and changes no on-chain state; only `acknowledgeDelivery` records acknowledgement.

The envelope metadata gives the resolver the fields required to reconstruct the authenticated associated data and priority commitment; it is delivered through the confidential route but is not part of the encrypted report plaintext. Other contact types MAY be supported if their complete encryption, envelope, transport, response, retry, and idempotency rules are described in the terms.

### Priority Commitment and Finding Identity

Before delivering a report, the reporter encrypts the complete report for the validated snapshot and computes:

```text
PRIORITY_TYPEHASH = keccak256(
    "ReleaseSafetyFinding(uint256 chainId,address escrow,bytes32 programId,bytes32 releaseRefKey,bytes32 releaseCommitment,address reporter,address committedBeneficiary,bytes32 claimedSeverityId,bytes32 ciphertextHash,bytes32 contactKeyHash,bytes32 salt)"
)

priorityCommitment = keccak256(abi.encode(
    PRIORITY_TYPEHASH,
    block.chainid,
    address(escrow),
    programId,
    releaseRefKey,
    releaseCommitment,
    reporter,
    committedBeneficiary,
    claimedSeverityId,
    ciphertextHash,
    contactKeyHash,
    salt
))
```

`committedBeneficiary`, `claimedSeverityId`, and `ciphertextHash` MUST be nonzero, and `committedBeneficiary` MUST differ from the escrow address. `claimedSeverityId` MUST identify a configured tier. `salt` MUST contain at least 128 bits of cryptographic entropy.

The reporter calls `commitFinding(programId, priorityCommitment, keyVersion, contactKeyHash)` before delivering the envelope. The call MUST occur while `block.timestamp < submissionDeadline`. The escrow MUST reject a zero commitment or a previously used `(programId, reporter, priorityCommitment)` tuple. It MUST NOT treat use of the same opaque commitment by a different reporter as use by the committed reporter. It stores the reporter and validated contact snapshot fields, assigns the next priority index, and derives:

```text
findingId = keccak256(abi.encode(
    programId,
    priorityIndex,
    reporter,
    priorityCommitment
))
```

The fixed preimage is not a signed-message format. The transaction authenticates the reporter, and explicit chain and escrow fields prevent cross-deployment replay. Reporter-scoped reuse rejection prevents double submission without allowing a mempool copier to block the committed reporter; a copy stored under another reporter cannot be acknowledged. A reporter SHOULD wait for suitable chain finality before private delivery.

### Delivery, Resolution, and Duplicate References

A finding permits exactly these transitions:

```text
None -> Committed
Committed -> Delivered | Rejected | Expired
Delivered -> Reserved | Rejected | Expired
Reserved -> Paid
```

Only the designated resolver MAY acknowledge, reserve, or reject a finding. Acknowledgement and resolution MUST occur while `block.timestamp < resolutionDeadline`.

After receiving the encrypted report, the resolver calls:

```text
acknowledgeDelivery(
    findingId,
    claimedSeverityId,
    committedBeneficiary,
    ciphertextHash,
    salt
)
```

The escrow MUST reconstruct the priority commitment using the stored reporter, program, and contact hash. It MUST require a beneficiary that is nonzero and not the escrow, a configured claimed tier, an exact match, and state `Committed`. It then stores the disclosed fields, sets both beneficiary fields to `committedBeneficiary`, changes state to `Delivered`, and emits `FindingDeliveryAcknowledged`. This proves that the resolver supplied a valid preimage; it does not prove report truth, route availability, or correct decryption.

To accept a delivered finding, the resolver calls `reserveAward(findingId, awardedSeverityId, decisionHash)`. The awarded severity MAY differ from the claimed severity. The escrow MUST require state `Delivered`, a nonzero `decisionHash`, a configured awarded tier, and an available slot. It atomically decrements that tier's raw available awards, increments its reserved awards, moves the exact reward from available to reserved accounting, stores the awarded severity, reward, and decision hash, and changes state to `Reserved`. No token call occurs during reservation.

`decisionHash` MUST equal `keccak256(decisionBytes)`, where the decision encoding is defined in the immutable terms. The decision SHOULD state the disposition, claimed and awarded severities, evidence class, conflicts disclosed by the resolver, and any duplicate reference without exposing confidential report content.

Where two materially equivalent findings both qualify under the immutable duplicate policy, the resolver MUST prefer the one with the lower priority index. The resolver MAY reject a finding in `Committed` or `Delivered` state with `rejectFinding(findingId, decisionHash, duplicateOf)`. Rejection consumes no tier capacity. A nonzero `duplicateOf` MUST identify a finding in the same program with a lower priority index and state `Reserved` or `Paid`; otherwise the call MUST revert. The escrow enforces only this direction and accepted-reference rule. Material equivalence and qualification remain resolver trust decisions.

The reporter MAY call `updateBeneficiary(findingId, newBeneficiary)` in `Delivered` or `Reserved` state. `newBeneficiary` MUST be nonzero and MUST differ from the escrow. The committed beneficiary remains stored for commitment verification, while subsequent payout uses the updated beneficiary. Each change MUST emit `BeneficiaryUpdated`.

At or after `resolutionDeadline`, any account MAY expire a `Committed` or `Delivered` finding. Expiry consumes no capacity. Reserved, rejected, expired, and paid states are final except that a reserved finding can become paid.

### Immediate Payout and Constant-Time Refund

Any account MAY call `pay` for a `Reserved` finding immediately after reservation, including in the same block. An implementation MUST NOT impose a payout delay. Before the external token call, it MUST set state to `Paid`, decrement the awarded tier's reserved awards and program reserved accounting, increment the tier's paid awards and program paid accounting, and decrement `tokenObligation`. The entire call MUST revert unless the escrow balance decreases by exactly the reward, the current beneficiary balance increases by exactly the reward, and the remaining escrow balance covers the remaining token obligation. The caller receives no fee.

At or after `resolutionDeadline`, any account MAY call `refundRemainder`. The escrow MUST set a program-level `refunded` flag, move `availableAmount` to `refundedAmount`, reduce `tokenObligation`, and transfer exactly that amount to the sponsor using the same balance checks. It MUST NOT iterate over tiers. A second call MUST revert or return without state changes.

After refund, `tier()` MUST report zero `availableAwards` and report the tier's stored raw available count as `refundedAwards`. Before refund, `refundedAwards` is zero. Reserved award counts and payouts are unaffected. This program-level interpretation provides constant-time refund while preserving exact per-tier history.

### Interfaces

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

interface IReleaseSafetyRewards {
    enum FindingState {
        None,
        Committed,
        Delivered,
        Reserved,
        Rejected,
        Expired,
        Paid
    }

    struct SeverityTier {
        bytes32 severityId;
        uint256 reward;
        uint32 maxAwards;
    }

    struct ProgramData {
        uint256 releaseChainId;
        address releaseRegistry;
        bytes32 releaseId;
        bytes32 releaseRefKey;
        bytes32 releaseCommitment;
        bytes32 termsHash;
        bytes32 tierScheduleHash;
        string releaseURI;
        string termsURI;
        address sponsor;
        uint256 sponsorNonce;
        address token;
        address resolver;
        address contactProvider;
        uint8 contactType;
        uint64 submissionDeadline;
        uint64 resolutionDeadline;
        bool refunded;
        uint32 tierCount;
    }

    struct TierData {
        uint256 reward;
        uint32 maxAwards;
        uint32 availableAwards;
        uint32 reservedAwards;
        uint32 paidAwards;
        uint32 refundedAwards;
    }

    struct FindingData {
        bytes32 programId;
        address reporter;
        bytes32 priorityCommitment;
        uint256 priorityIndex;
        uint256 committedBlock;
        uint64 committedAt;
        uint64 keyVersion;
        bytes32 contactKeyHash;
        FindingState state;
        bytes32 claimedSeverityId;
        bytes32 awardedSeverityId;
        address committedBeneficiary;
        address beneficiary;
        uint256 reward;
        bytes32 ciphertextHash;
        bytes32 salt;
        bytes32 decisionHash;
        bytes32 duplicateOf;
    }

    event ProgramCreated(
        bytes32 indexed programId,
        address indexed sponsor,
        bytes32 indexed releaseRefKey,
        uint256 releaseChainId,
        address releaseRegistry,
        bytes32 releaseId,
        bytes32 releaseCommitment,
        bytes32 termsHash,
        bytes32 tierScheduleHash,
        string releaseURI,
        string termsURI,
        uint256 sponsorNonce,
        address token,
        address resolver,
        address contactProvider,
        uint8 contactType,
        uint64 submissionDeadline,
        uint64 resolutionDeadline,
        uint256 totalFunded
    );
    event TierConfigured(
        bytes32 indexed programId,
        bytes32 indexed severityId,
        uint32 index,
        uint256 reward,
        uint32 maxAwards
    );
    event FindingCommitted(
        bytes32 indexed findingId,
        bytes32 indexed programId,
        address indexed reporter,
        bytes32 priorityCommitment,
        uint256 priorityIndex,
        uint64 keyVersion,
        bytes32 contactKeyHash
    );
    event FindingDeliveryAcknowledged(
        bytes32 indexed findingId,
        bytes32 indexed claimedSeverityId,
        address indexed committedBeneficiary,
        bytes32 ciphertextHash,
        bytes32 salt
    );
    event AwardReserved(
        bytes32 indexed findingId,
        bytes32 indexed awardedSeverityId,
        address indexed beneficiary,
        bytes32 claimedSeverityId,
        uint256 reward,
        bytes32 decisionHash
    );
    event FindingRejected(
        bytes32 indexed findingId,
        bytes32 indexed duplicateOf,
        bytes32 decisionHash
    );
    event BeneficiaryUpdated(
        bytes32 indexed findingId,
        address indexed oldBeneficiary,
        address indexed newBeneficiary
    );
    event FindingExpired(bytes32 indexed findingId);
    event AwardPaid(
        bytes32 indexed findingId,
        address indexed beneficiary,
        uint256 reward
    );
    event RemainderRefunded(
        bytes32 indexed programId,
        address indexed sponsor,
        uint256 amount
    );

    function createProgram(
        uint256 releaseChainId,
        address releaseRegistry,
        bytes32 releaseId,
        bytes32 releaseRefKey,
        bytes32 releaseCommitment,
        string calldata releaseURI,
        bytes32 termsHash,
        string calldata termsURI,
        address token,
        address resolver,
        address contactProvider,
        uint8 contactType,
        uint64 submissionDeadline,
        uint64 resolutionDeadline,
        uint256 sponsorNonce,
        SeverityTier[] calldata tiers
    ) external returns (bytes32 programId);

    function commitFinding(
        bytes32 programId,
        bytes32 priorityCommitment,
        uint64 keyVersion,
        bytes32 contactKeyHash
    ) external returns (bytes32 findingId);

    function acknowledgeDelivery(
        bytes32 findingId,
        bytes32 claimedSeverityId,
        address committedBeneficiary,
        bytes32 ciphertextHash,
        bytes32 salt
    ) external;

    function reserveAward(
        bytes32 findingId,
        bytes32 awardedSeverityId,
        bytes32 decisionHash
    ) external;

    function rejectFinding(
        bytes32 findingId,
        bytes32 decisionHash,
        bytes32 duplicateOf
    ) external;

    function updateBeneficiary(bytes32 findingId, address newBeneficiary) external;
    function expireFinding(bytes32 findingId) external;
    function pay(bytes32 findingId) external;
    function refundRemainder(bytes32 programId) external;

    function maxTierCount() external pure returns (uint32);
    function tierCount(bytes32 programId) external view returns (uint32);
    function tierAt(bytes32 programId, uint32 index) external view returns (bytes32);
    function tier(bytes32 programId, bytes32 severityId)
        external view returns (TierData memory);
    function program(bytes32 programId) external view returns (ProgramData memory);
    function finding(bytes32 findingId) external view returns (FindingData memory);
    function findingState(bytes32 findingId) external view returns (FindingState);
    function accounting(bytes32 programId)
        external view
        returns (
            uint256 totalFunded,
            uint256 availableAmount,
            uint256 reservedAmount,
            uint256 paidAmount,
            uint256 refundedAmount
        );
    function tokenObligation(address token) external view returns (uint256);
}
```

The rewards contract MUST implement ERC-165 and report `0xb909a538` for `IReleaseSafetyRewards`. All getters MUST revert for an unknown program, tier, finding, or out-of-range tier index rather than returning plausible zero data.

A conforming core escrow MUST NOT silently add reporter allowlists, reporter bonds, submission fees, or sponsor-controlled cancellation. Such policies MAY be implemented by a separately detectable extension or wrapper and MUST be disclosed in the terms. They MUST NOT weaken obligations of a core program already created.

## Rationale

### Exact Release and Immutable Retrieval References

One program per release prevents a rolling model or policy from obscuring what was tested. The coordinate identifies the registry entry; the manifest commitment identifies its bytes. This lets Autonomous Release Lineage compose directly while other registries integrate without treating equal local IDs as global. URI fields aid retrieval without making storage trustworthy or permanent.

### Fully Funded Finite Capacity

Finite tier counts expose maximum obligations and prevent lower-severity awards from consuming critical capacity. Enumeration and refund counters let clients reconstruct every slot without unbounded refund loops.

### Confidential Priority and Delivery Acknowledgement

Plaintext reports are unsafe, while delivering first lets a recipient copy the report or dispute priority. Commitment records only an opaque hash at a chain-ordered index. A later valid resolver opening binds the revealed beneficiary, claimed severity, ciphertext hash, contact snapshot, and salt to that earlier hash; it does not prove that ciphertext bytes or a report existed when the commitment was posted. Resolver acknowledgement shows knowledge of the commitment preimage before adjudication without revealing the report. It is evidence of workflow, not proof of vulnerability validity or delivery.

Separating claimed and awarded severity lets researchers commit their claim while preserving resolver reclassification. Separating committed and current beneficiaries preserves commitment integrity while allowing recovery from an unusable payout address.

### Designated Resolver and Immediate Settlement

Hashes cannot decide correctness or semantic duplication. A named resolver exposes that trust boundary. Reservation removes sponsor discretion, and permissionless immediate payout removes a second approval. Programs needing appeals can designate a resolver contract implementing them.

### General Bounty and Contact Formats

[ERC-1081](./eip-1081.md) models general bounties and permits broader data, actor, deadline, and payout workflows. This proposal neither replaces nor extends it. Release-specific safety disclosure adds immutable release and terms commitments, confidential priority, versioned contact binding, finite tier capacity, and exact escrow accounting.

[ERC-5437](./eip-5437.md) provides an encryption-key, contact-type, and delivery-data pattern but no release scope, priority, version-bound snapshot, escrow, resolution, or payout. A contact adapter can expose its data through `IReleaseSafetyContact`, but must add immutable historical versions and validity intervals.

### Hats Finance and Hats Protocol

Hats Finance is direct prior art for funded noncustodial bounty vaults, encrypted submissions, on-chain hash evidence, committee adjudication, and payout. This proposal does not claim those mechanisms are new. Its interoperability contribution is a common exact-release identifier, immutable content-addressed terms and tier schedule, a versioned contact snapshot, a fixed priority preimage, and standardized discoverable accounting.

No novelty claim is made for encryption, escrow, bounty adjudication, or off-chain vulnerability reports in isolation. The distinct standardization boundary is their composition with exact-release binding, confidential chain-ordered priority commitments, immutable finite tier obligations, designated resolution, and prefunded exact-transfer payouts.

Hats Protocol is a separate role and authority system. It can control a resolver contract, but is not itself the bounty mechanism. Resolver membership and governance remain outside this proposal and must be disclosed by the terms.

### Rejected Alternatives

A sponsor database cannot independently prove unchanged terms, ordering, or escrow. Automatic truth judgments invite fabricated claims, while public evidence leaks vulnerabilities. Per-finding appeals and on-chain severity taxonomies add governance without making confidential facts machine-verifiable. The bounded design instead commits promises and funds, then makes one designated off-chain judgment auditable.

## Backwards Compatibility

This proposal introduces new interfaces and does not change existing token, bounty, contact, identity, or role contracts. Existing systems can integrate through adapters. A prior bounty does not become conforming without the exact release and terms commitments, versioned contact validation, fixed priority preimage, funded finite schedule, state transitions, and accounting guarantees defined here.

## Test Cases

1. Create a program whose nonzero release coordinate recomputes the supplied `releaseRefKey` and whose release and terms bytes reproduce the supplied hashes. For an Autonomous Release Lineage coordinate, confirm the commitment equals its stored `manifestHash`. With tiers `(100, 1)` and `(10, 2)`, exactly 120 is funded, accounting returns `(120, 120, 0, 0, 0)`, and both tiers remain enumerable.
2. Creation reverts for a zero release-coordinate field, mismatched release reference key, reused sponsor nonce, zero commitment, empty URI, zero or duplicate severity identifier, zero reward, zero award count, zero or more than 32 tiers, identical sponsor and resolver addresses, or deadlines not satisfying the strict order.
3. Publish contact version 1, then version 2. A commitment naming version 1 reverts after version 2 is current. A commitment naming version 2 reverts if any snapshot field hashes differently, it is not yet valid, or `acceptUntil` is before the resolution deadline.
4. Commit a correct priority commitment containing the program's release reference key before the submission deadline. It receives priority one. Reusing it from the same reporter reverts. A call at the deadline reverts. A mempool copier may store the opaque value under a different reporter, but cannot acknowledge it and cannot block or open the original reporter's tuple.
5. POST the exact envelope to a type-1 HTTPS contact. A new delivery and byte-identical retry return empty `202` responses; a changed body for the same finding returns `409`; and redirects are not followed. None changes finding state. A resolver acknowledgement with the correct claimed severity, committed beneficiary, ciphertext hash, contact hash, and salt enters `Delivered`. Changing any committed field reverts. Acknowledgement by another account or after the deadline reverts.
6. Reserve the delivered finding at a different awarded severity. Only the awarded tier loses a slot; its reward moves from available to reserved; and both severity identifiers remain discoverable. Double reservation and an unavailable tier revert.
7. Reject a finding with a duplicate reference. A reference to another program, an equal or higher priority, or a finding not yet reserved or paid reverts. A lower-priority reserved reference succeeds without consuming a slot.
8. Immediately after reservation, an unrelated account calls `pay`. Exactly the reward reaches the current beneficiary, reserved accounting and `tokenObligation` decrease, and paid accounting increases. No caller fee or delay applies.
9. The reporter changes the beneficiary in `Delivered`, then again in `Reserved`. Events preserve the changes, the committed beneficiary remains unchanged, and payout reaches only the latest nonzero beneficiary. Another caller and a post-payment update revert.
10. After the resolution deadline, refund a program with available and reserved awards. One constant-time call refunds exactly `availableAmount`, sets the program flag, and leaves reserved funds payable. Tier getters report raw unconsumed slots as refunded and zero available. A second call changes nothing or reverts.
11. Funding with a fee-on-transfer token, or one that changes either endpoint by an inexact amount, reverts. A token returning false or invoking a reentrant callback cannot create inconsistent state.
12. If a previously accepted token later rebases downward or changes transfer behavior, a payout or refund fails its exact-balance or solvency checks and reverts. The escrow does not claim it can restore the missing assets.
13. At the resolution deadline, any account can expire a `Committed` or `Delivered` finding. Reservation, acknowledgement, and rejection then revert, while already reserved findings remain payable.

## Reference Implementation

No reference implementation is included. The interfaces, exact encodings, state machine, accounting rules, and test cases permit independent implementations before code is added under the proposal's assets directory.

## Security Considerations

Release and terms fields commit hash values; matching retrieved bytes demonstrate consistency with those values. They do not prove that bytes were available at creation, or prove deployment, authorship, legality, availability, or safety. Clients must retrieve the documents, reproduce their hashes, validate the release coordinate-to-manifest mapping, and determine whether an observed system actually matches the release manifest.

The designated resolver can misclassify severity, reject valid reports, accept invalid ones, conceal conflicts, or misjudge duplicates. Distinct sponsor and resolver addresses do not prove distinct controllers. A sponsor or resolver can also self-deal through another reporter address. Terms should identify controllers, governance, conflicts, availability expectations, and evidence practices. A resolver contract can add quorum or appeal rules, but this interface does not attest to their quality.

Confidentiality depends on endpoint security, authenticated encryption, fresh HPKE randomness, salt entropy, contact-provider integrity, and retention of historical private keys. The chain reveals timing, reporter, contact version, eventual claimed and awarded severity, beneficiary, reward, and ciphertext hash. Deterministic encryption, reused randomness, low-entropy salts, or separately published plaintext commitments can permit correlation or guessing. Key compromise exposes reports; key deletion can make them unreadable. On-chain delivery data is public and must never contain credentials.

A contact provider may publish an unusable key or route, fail to retain a key, or violate its historical-snapshot promise. The escrow's hash proves which public snapshot was checked, not that private delivery worked. An HTTP `202` is only the endpoint's transport response. Resolver acknowledgement proves knowledge of a valid preimage, not that ciphertext or report bytes existed at commitment time, who authored them, how they arrived, whether they decrypted correctly, or whether their claims are true.

Immediate private delivery after one block creates reorganization and front-running risk. Reporters should choose confirmation depth based on value and harm. Commitment binding prevents common cross-chain replay and beneficiary redirection, but cannot stop resolver leakage, coerced disclosure, or delivery of bytes outside the committed ciphertext.

The exact-transfer and non-rebasing restriction is a trust assumption about token code and governance, not a capability inferred from the ERC-20 interface. A token can upgrade, blacklist, pause, rebase, lie about balances, or become insolvent. Exact endpoint deltas and global obligations fail closed but cannot repair a negative rebase or force a blocked transfer. Programs should use narrowly governed, well-reviewed assets. Direct token transfers create unallocated excess and may be irrecoverable unless a separate excess-recovery rule cannot touch obligations.

Implementations require overflow-safe funding arithmetic, checks-effects-interactions ordering, reentrancy protection, exact balance-delta checks, and atomic rollback on transfer mismatch. No sponsor, resolver, upgrader, token callback, refund path, or excess-recovery function may access reserved obligations. Upgradeable deployments must make program-level immutability credible; an ERC-165 response alone does not prove that property.

Permissionless commitments enable spam and storage growth. The core does not solve this with hidden eligibility gates. Frontends can filter, sponsors can price resolver capacity into program design, and separately detectable extensions can add disclosed controls without changing existing obligations.

Finally, reward terms can incentivize hazardous testing. Terms should identify prohibited production tests, physical safety boundaries for robotics, data-handling restrictions, legal authorization, and coordinated-disclosure expectations. A reward does not authorize access, deployment, or experimentation that the tester is not otherwise permitted to perform.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
