---
title: Physical Actuation Authorization
description: An ERC-8001 profile binding robot commands to safety status, identity-stable nonces, and authorization-signer receipts.
author: Shayan Salehi (@shayansal)
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-08-31
requires: 165, 712, 1271, 2098, 5267, 8001
---

## Abstract

This ERC defines a physical-actuation profile for [ERC-8001](./eip-8001.md). It binds a multi-party intent to a device identity reference, release, mission, command, operating envelope, safety policy, typed safety-status snapshot, identity-stable deployment-scoped nonce, authorization-designated signer, and deadline. The profile rechecks every safety-status entry when the intent executes. Execution records an authorization; it does not assert that motion occurred or prove a relationship between the signer and physical hardware. A separate receipt registry records an immutable sequence of assertions signed by the authorization-designated signer. Local interlocks always retain authority over physical actuation.

## Motivation

Robots receive commands from people, agents, fleet services, and other robots. A correctly signed command can still target the wrong firmware, exceed an approved envelope, depend on a superseded safety statement, or be replayed through another workflow. Application-specific messages make it difficult for devices, insurers, investigators, and counterparties to determine exactly what was authorized.

Ethereum can establish when identified parties made a common commitment, when a particular safety-status view was current, and whether an authorization nonce was already consumed. Those guarantees are valuable across organizations that do not share one database. Ethereum cannot observe a motor, payload, person, geofence, emergency stop, or sensor. An authorization-signer signature is attributable evidence from the selected key, not proof of a physical event or hardware identity.

[ERC-8001](./eip-8001.md) already defines participants, typed acceptances, expiry, initiator nonces, and a coordination lifecycle. This ERC specializes that lifecycle for physical commands, adds live safety-status checks and a second replay boundary scoped to the referenced device identity, and separates authorization from outcome assertions by the signer that the approving participants designated.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Scope and definitions

An **authorization** is an executed ERC-8001 intent conforming to this profile. It permits a device to consider a command; it does not require the device to actuate.

A **profile deployment** is the exact `(chainId, profileContract)` coordinate implementing this ERC. Device nonce state is local to this coordinate.

An **action profile** defines the canonical encoding and interpretation of command bytes committed by `commandHash`.

An **operating envelope** is a committed set of physical constraints, such as permitted actuators, workspace, geofence, speed, force, payload, energy, and duration.

A **safety snapshot entry** identifies one current statement in a configured safety-status registry. Any change to that registry's current statement supersedes the entry.

A **device identity reference** is a namespace coordinate selected by the authorizing participants. This ERC does not prove that it denotes particular hardware, that the hardware exists, or that the authorization signer controls it.

An **authorization signer** is the address designated inside the multi-party authorization to sign receipts. Its relationship to a robot, controller, gateway, or secure element is external policy.

A **device receipt** is an authorization-signer assertion about one authorization. It is not a hardware attestation, sensor oracle, or proof that the asserted physical event occurred.

### Constants and references

Implementations MUST use:

```solidity
bytes32 constant PHYSICAL_ACTUATION_TYPE =
    keccak256("PHYSICAL_ACTUATION_AUTHORIZATION_V1");
bytes32 constant PHYSICAL_ACTUATION_PAYLOAD_VERSION =
    keccak256("PHYSICAL_ACTUATION_PAYLOAD_V1");
bytes32 constant PHYSICAL_ACTUATION_SAFETY_SNAPSHOT_VERSION =
    keccak256("PHYSICAL_ACTUATION_SAFETY_SNAPSHOT_V1");
bytes4 constant PHYSICAL_ACTUATION_EXECUTION_MAGIC =
    bytes4(keccak256("PHYSICAL_ACTUATION_AUTHORIZE_V1"));
uint256 constant MAX_SAFETY_SNAPSHOT_ENTRIES = 16;
```

```solidity
struct ResourceRef {
    uint256 chainId;
    address registry;
    bytes32 id;
}

struct DeviceRef {
    ResourceRef identity;
    address authorizationSigner;
}

struct TargetRef {
    uint8 kind;
    uint256 chainId;
    address registry;
    bytes32 id;
}

struct SafetySnapshotEntry {
    address statusRegistry;
    address issuer;
    TargetRef target;
    bytes32 statusId;
    uint64 sequence;
    uint64 maxAge;
}

struct ActuationAuthorization {
    DeviceRef device;
    ResourceRef release;
    bytes32 missionId;
    bytes32 actionProfileId;
    bytes32 commandHash;
    bytes32 operatingEnvelopeHash;
    bytes32 safetyPolicyHash;
    bytes32 recallSnapshotHash;
    bytes32 telemetryPolicyHash;
    bytes32 approverSetHash;
    uint64 actuationNonce;
    uint64 issuedAt;
    uint64 notBefore;
    uint64 deadline;
    uint64 recallValidUntil;
}
```

Every `ResourceRef` field MUST be nonzero. `device.authorizationSigner` and every `bytes32` field of `ActuationAuthorization` MUST be nonzero. `actuationNonce` MUST be nonzero. An exact release reference MUST identify immutable data or a content commitment, not a mutable URI or product name. The participant signatures authorize the selected device identity reference and authorization signer together; they do not prove an underlying hardware binding.

`commandHash` MUST equal `keccak256(commandBytes)`, where `commandBytes` is the canonical encoding declared by `actionProfileId`. The other profile hashes commit to canonical encodings declared by their named profiles. Their preimages need not be public.

`TargetRef.kind` MUST be in the inclusive range 1 through 4. All other `TargetRef` fields MUST be nonzero. Kinds are `1 = RELEASE`, `2 = DEVICE`, `3 = FLEET`, and `4 = CAPABILITY`, matching the safety-status interface.

### Authorization and snapshot hashes

Implementations MUST compute:

```solidity
bytes32 constant RESOURCE_REF_TYPEHASH = keccak256(
    "ResourceRef(uint256 chainId,address registry,bytes32 id)"
);
bytes32 constant DEVICE_KEY_TYPEHASH = keccak256(
    "DeviceKey(bytes32 identityKey)"
);
bytes32 constant DEVICE_AUTHORIZATION_KEY_TYPEHASH = keccak256(
    "DeviceAuthorizationKey(bytes32 deviceKey,address authorizationSigner)"
);
bytes32 constant TARGET_REF_TYPEHASH = keccak256(
    "TargetRef(uint8 kind,uint256 chainId,address registry,bytes32 id)"
);
bytes32 constant ACTUATION_AUTHORIZATION_TYPEHASH = keccak256(
    "ActuationAuthorization(bytes32 deviceAuthorizationKey,bytes32 releaseKey,bytes32 missionId,bytes32 actionProfileId,bytes32 commandHash,bytes32 operatingEnvelopeHash,bytes32 safetyPolicyHash,bytes32 recallSnapshotHash,bytes32 telemetryPolicyHash,bytes32 approverSetHash,uint64 actuationNonce,uint64 issuedAt,uint64 notBefore,uint64 deadline,uint64 recallValidUntil)"
);
bytes32 constant ACTUATION_AUTHORIZATION_DOMAIN_TYPEHASH = keccak256(
    "PhysicalActuationAuthorizationDomain(uint256 chainId,address profileContract,bytes32 authorizationStructHash)"
);

resourceKey = keccak256(abi.encode(
    RESOURCE_REF_TYPEHASH, ref.chainId, ref.registry, ref.id
));
deviceKey = keccak256(abi.encode(
    DEVICE_KEY_TYPEHASH,
    resourceKey(device.identity)
));
deviceAuthorizationKey = keccak256(abi.encode(
    DEVICE_AUTHORIZATION_KEY_TYPEHASH,
    deviceKey,
    device.authorizationSigner
));
targetKey = keccak256(abi.encode(
    TARGET_REF_TYPEHASH, target.kind, target.chainId, target.registry, target.id
));
authorizationStructHash = keccak256(abi.encode(
    ACTUATION_AUTHORIZATION_TYPEHASH,
    deviceAuthorizationKey,
    resourceKey(release),
    missionId,
    actionProfileId,
    commandHash,
    operatingEnvelopeHash,
    safetyPolicyHash,
    recallSnapshotHash,
    telemetryPolicyHash,
    approverSetHash,
    actuationNonce,
    issuedAt,
    notBefore,
    deadline,
    recallValidUntil
));
authorizationHash = keccak256(abi.encode(
    ACTUATION_AUTHORIZATION_DOMAIN_TYPEHASH,
    block.chainid,
    address(this),
    authorizationStructHash
));
```

`deviceKey` depends only on `device.identity` and is therefore stable when an authorization signer changes. `deviceAuthorizationKey` separately binds that stable key to the signer selected by the participants. The profile MUST key `lastActuationNonce` by `deviceKey`; changing the selected signer MUST NOT reset or partition nonce history.

The safety entries MUST contain between 1 and `MAX_SAFETY_SNAPSHOT_ENTRIES` elements. `statusRegistry`, `issuer`, `statusId`, `sequence`, and `maxAge` MUST be nonzero. Each `statusRegistry` MUST be configured as trusted by the profile deployment. Entries MUST be strictly ascending and unique by:

```solidity
entryKey = keccak256(abi.encode(statusRegistry, issuer, targetKey));
```

The snapshot commitment MUST be:

```solidity
recallSnapshotHash = keccak256(abi.encode(
    PHYSICAL_ACTUATION_SAFETY_SNAPSHOT_VERSION,
    safetyEntries
));
```

### Current safety-status interface

This profile independently defines and requires `IAutonomousSafetyStatusQuery`. Its [ERC-165](./eip-165.md) interface identifier is `0x468323fd`. The companion Autonomous System Safety Status proposal is the preferred full implementation because it adds signed immutable statements and history, but this profile does not depend on that proposal receiving a number and does not add it to `requires`.

Each configured status registry MUST report support for `0x468323fd` and implement the functions with the same ABI tuple layout for `TargetRef`:

```solidity
interface IAutonomousSafetyStatusQuery /* is IERC165 */ {
    struct TargetRef {
        uint8 kind;
        uint256 chainId;
        address registry;
        bytes32 id;
    }

    function currentStatusId(
        address issuer,
        bytes32 targetKey
    ) external view returns (bytes32 statusId);

    function currentSequence(
        address issuer,
        bytes32 targetKey
    ) external view returns (uint64 sequence);

    function isFreshAndActive(
        address issuer,
        TargetRef calldata target,
        uint64 maxAge
    ) external view returns (bool);
}
```

The exact selector calculation is:

```text
0xf2fe210e  currentStatusId(address,bytes32)
0x497aad21  currentSequence(address,bytes32)
0xfd07afd2  isFreshAndActive(address,(uint8,uint256,address,bytes32),uint64)
XOR         0x468323fd
```

A conforming query registry MUST maintain at most one current record for each `(issuer, targetKey)` pair. With no record, `currentStatusId` and `currentSequence` MUST return zero and `isFreshAndActive` MUST return `false`. The first record MUST have a nonzero identifier and sequence one. Any change to its disposition, issue time, effective time, expiry, or committed meaning MUST atomically replace it with a different nonzero identifier and sequence exactly one greater; sequence overflow MUST be rejected. A status identifier's referent MUST remain immutable. At one block, `currentStatusId` and `currentSequence` MUST describe the same record.

`isFreshAndActive` MUST revert when `target` violates the target rules above. Otherwise it MUST return `true` only when the current record is explicitly active and unrestricted, is for the supplied issuer and exact `targetKey`, has `issuedAt <= effectiveAt <= block.timestamp < validUntil`, has `maxAge > 0`, and satisfies `block.timestamp - issuedAt <= maxAge`. It MUST return `false` for a missing, restricted, suspended, recalled, future-effective, expired, or stale record. All times are Unix seconds. Implementations MAY obtain these semantics from a larger status registry or a conforming adapter.

For every entry, the profile MUST require that `currentStatusId(issuer, targetKey) == statusId`, `currentSequence(issuer, targetKey) == sequence`, and `isFreshAndActive(issuer, target, maxAge) == true`. A revert, malformed return, missing code, unconfigured registry, or failed comparison MUST invalidate the snapshot. The profile MUST perform these checks both when proposing and immediately within execution. Thus, publication of any newer current statement before execution invalidates the saved snapshot even if the newer statement is less restrictive.

### ERC-8001 payload mapping

The [ERC-8001](./eip-8001.md) fields MUST be:

```text
AgentIntent.coordinationType  = PHYSICAL_ACTUATION_TYPE
AgentIntent.coordinationValue = 0
AgentIntent.expiry            = authorization.deadline

CoordinationPayload.version          = PHYSICAL_ACTUATION_PAYLOAD_VERSION
CoordinationPayload.coordinationType = PHYSICAL_ACTUATION_TYPE
CoordinationPayload.coordinationData = abi.encode(authorization, safetyEntries)
CoordinationPayload.conditionsHash   = actuationConditionsHash
CoordinationPayload.timestamp        = authorization.issuedAt
CoordinationPayload.metadata         = bytes(0)
```

The profile contract MUST retain ERC-8001's [EIP-712](./eip-712.md) domain `{name: "ERC-8001", version: "1", chainId, verifyingContract: profileContract}` and MUST expose that domain through [ERC-5267](./eip-5267.md).

The payload hash MUST use ERC-8001's field-wise `getPayloadHash` semantics, exactly:

```solidity
AgentIntent.payloadHash = keccak256(abi.encode(
    payload.version,
    payload.coordinationType,
    payload.coordinationData,
    payload.conditionsHash,
    payload.timestamp,
    payload.metadata
));
```

It MUST NOT be computed as `keccak256(abi.encode(payload))`.

`coordinationData` MUST decode as exactly one `(ActuationAuthorization, SafetySnapshotEntry[])` tuple and MUST equal `abi.encode(decodedAuthorization, decodedEntries)` byte-for-byte. This rejects noncanonical or trailing data.

`actuationConditionsHash` MUST be:

```solidity
bytes32 constant ACTUATION_CONDITIONS_TYPEHASH = keccak256(
    "ActuationConditions(bytes32 operatingEnvelopeHash,bytes32 safetyPolicyHash,bytes32 recallSnapshotHash,bytes32 telemetryPolicyHash,uint64 recallValidUntil)"
);
actuationConditionsHash = keccak256(abi.encode(
    ACTUATION_CONDITIONS_TYPEHASH,
    authorization.operatingEnvelopeHash,
    authorization.safetyPolicyHash,
    authorization.recallSnapshotHash,
    authorization.telemetryPolicyHash,
    authorization.recallValidUntil
));
```

`authorization.approverSetHash` MUST equal `keccak256(abi.encodePacked(intent.participants))`. Participants inherit ERC-8001's strictly ascending, unique address rule. A deployment policy MUST define the organizational roles represented by those addresses.

For this coordination type, `acceptCoordination` MUST additionally reject an `AcceptanceAttestation` unless `attestation.conditionsHash == actuationConditionsHash`. This binds every participant's signed conditions to the same canonical actuation conditions.

### Profile validation and execution

A conforming profile MUST implement `IAgentCoordination` from ERC-8001 and `IPhysicalActuationProfile` below. It MUST preserve ERC-8001's validation and lifecycle rules.

`proposeCoordination` MUST additionally reject when:

- a reference, required commitment, or snapshot entry is invalid;
- a field or payload mapping above is incorrect;
- any configured status check fails;
- `issuedAt > block.timestamp` or `issuedAt > notBefore`;
- `notBefore >= deadline` or `deadline > recallValidUntil`; or
- `actuationNonce <= lastActuationNonce(deviceKey)`.

Several proposed intents MAY contain the same unused nonce. Only one can execute successfully. The identity-scoped actuation nonce is scoped to `(block.chainid, address(profile), deviceKey)`. It does not form a global nonce across profile deployments.

For this type, `executionData` MUST equal `abi.encode(PHYSICAL_ACTUATION_EXECUTION_MAGIC)`. `executeCoordination` MUST additionally reject when:

- `block.timestamp < notBefore`;
- `block.timestamp >= deadline` or `block.timestamp >= recallValidUntil`;
- `actuationNonce <= lastActuationNonce(deviceKey)`; or
- any execution-time current safety-status check fails.

After all checks, execution MUST atomically update the nonce keyed by stable `deviceKey`, store the complete authorization, authorization hash, and safety entries under `intentHash`, emit `ActuationAuthorized` with the computed `deviceKey` and `deviceAuthorizationKey`, and complete the ERC-8001 transition to `Executed`. It MUST return `(true, abi.encode(authorizationHash))`. A conforming profile MUST NOT return `false` for an otherwise valid execution. Any profile-hook, storage, or validation failure MUST revert the entire call so that neither authorization state nor ERC-8001 lifecycle state advances partially.

### Authorization profile interface

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

interface IPhysicalActuationProfile /* is IERC165 */ {
    struct ResourceRef { uint256 chainId; address registry; bytes32 id; }
    struct DeviceRef { ResourceRef identity; address authorizationSigner; }
    struct TargetRef { uint8 kind; uint256 chainId; address registry; bytes32 id; }
    struct SafetySnapshotEntry {
        address statusRegistry;
        address issuer;
        TargetRef target;
        bytes32 statusId;
        uint64 sequence;
        uint64 maxAge;
    }
    struct ActuationAuthorization {
        DeviceRef device;
        ResourceRef release;
        bytes32 missionId;
        bytes32 actionProfileId;
        bytes32 commandHash;
        bytes32 operatingEnvelopeHash;
        bytes32 safetyPolicyHash;
        bytes32 recallSnapshotHash;
        bytes32 telemetryPolicyHash;
        bytes32 approverSetHash;
        uint64 actuationNonce;
        uint64 issuedAt;
        uint64 notBefore;
        uint64 deadline;
        uint64 recallValidUntil;
    }

    event ActuationAuthorized(
        bytes32 indexed intentHash,
        bytes32 indexed deviceKey,
        bytes32 indexed missionId,
        bytes32 deviceAuthorizationKey,
        bytes32 authorizationHash,
        uint64 actuationNonce,
        uint64 deadline
    );

    function getActuation(bytes32 intentHash) external view returns (
        bool authorized,
        bytes32 authorizationHash,
        ActuationAuthorization memory authorization
    );
    function getSafetySnapshot(
        bytes32 intentHash
    ) external view returns (SafetySnapshotEntry[] memory safetyEntries);
    function lastActuationNonce(bytes32 deviceKey) external view returns (uint64);
    function isSafetyStatusRegistry(address registry) external view returns (bool);
    function deviceKey(ResourceRef calldata identity) external pure returns (bytes32);
    function deviceAuthorizationKey(
        DeviceRef calldata device
    ) external pure returns (bytes32);
    function hashAuthorization(
        ActuationAuthorization calldata authorization
    ) external view returns (bytes32);
}
```

The interface identifier is `0xd3744de5`, the XOR of these seven selectors:

```text
0x223d627a  getActuation(bytes32)
0xc7498090  getSafetySnapshot(bytes32)
0x9cd27640  lastActuationNonce(bytes32)
0xc98e3b89  isSafetyStatusRegistry(address)
0x2a1f6546  deviceKey((uint256,address,bytes32))
0x001f6192  deviceAuthorizationKey(((uint256,address,bytes32),address))
0x495ce612  hashAuthorization((((uint256,address,bytes32),address),(uint256,address,bytes32),bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,bytes32,uint64,uint64,uint64,uint64,uint64))
XOR         0xd3744de5
```

The profile MUST report ERC-165 support for `0xd3744de5`. `getActuation` MUST return `false`, a zero hash, and a zero-valued authorization before execution, and the exact immutable stored values after execution. `getSafetySnapshot` MUST return an empty array before execution and the exact immutable entries afterward. This makes the live checks reconstructible without depending on archival transaction calldata.

### Separate receipt registry

Receipts MUST be handled by a contract separate from the ERC-8001 profile. This prevents one contract from claiming the ERC-8001 EIP-712 domain and a different receipt domain through the single [ERC-5267](./eip-5267.md) discovery interface.

```solidity
interface IPhysicalActuationReceiptRegistry /* is IERC165 */ {
    enum DevicePhase {
        NONE,
        ACKNOWLEDGED,
        STARTED,
        COMPLETED,
        FAILED,
        ABORTED,
        REJECTED
    }

    struct DeviceReceipt {
        address authorizationRegistry;
        bytes32 intentHash;
        bytes32 authorizationHash;
        bytes32 previousReceiptId;
        uint64 sequence;
        uint64 actuationNonce;
        DevicePhase phase;
        uint64 deviceTime;
        bytes32 telemetryRoot;
        bytes32 resultHash;
        bytes32 localSafetyStateHash;
    }

    event DeviceReceiptRecorded(
        bytes32 indexed actuationKey,
        bytes32 indexed receiptId,
        bytes32 indexed deviceKey,
        address authorizationRegistry,
        bytes32 intentHash,
        uint64 sequence,
        uint8 phase,
        bytes32 telemetryRoot,
        uint64 recordedAt
    );

    event AuthorizationProfileTrustSet(
        address indexed authorizationRegistry,
        bool trusted
    );

    error UnknownReceipt(bytes32 receiptId);
    error UntrustedAuthorizationProfile(address authorizationRegistry);
    error InvalidAuthorizationRecord();

    function submitDeviceReceipt(
        DeviceReceipt calldata receipt,
        bytes calldata signerSignature
    ) external returns (bytes32 receiptId);
    function getReceipt(bytes32 receiptId) external view returns (
        DeviceReceipt memory receipt
    );
    function receiptExists(bytes32 receiptId) external view returns (bool);
    function getLatestReceipt(
        address authorizationRegistry,
        bytes32 intentHash
    ) external view returns (bytes32 receiptId, DeviceReceipt memory receipt);
    function hashDeviceReceipt(
        DeviceReceipt calldata receipt
    ) external view returns (bytes32 receiptId);
    function isAuthorizationProfile(
        address authorizationRegistry
    ) external view returns (bool);
}
```

The receipt interface identifier is `0xd0c0411e`, the XOR of these six selectors:

```text
0x394c967e  submitDeviceReceipt((address,bytes32,bytes32,bytes32,uint64,uint64,uint8,uint64,bytes32,bytes32,bytes32),bytes)
0xfcecbb61  getReceipt(bytes32)
0xf9d969c9  receiptExists(bytes32)
0x7d683e22  getLatestReceipt(address,bytes32)
0x25cc16be  hashDeviceReceipt((address,bytes32,bytes32,bytes32,uint64,uint64,uint8,uint64,bytes32,bytes32,bytes32))
0xb41d2d54  isAuthorizationProfile(address)
XOR         0xd0c0411e
```

Both contracts MUST implement ERC-165 independently. The receipt registry MUST report support for `0xd0c0411e`.

The receipt registry MUST maintain an explicit allowlist of trusted authorization-profile addresses and expose its current decision through `isAuthorizationProfile`. Every addition or removal after deployment MUST emit `AuthorizationProfileTrustSet`. The configuration mechanism and authority MAY be immutable, access-controlled, or externally governed, but MUST be documented by the deployment. Removing a profile MUST block new receipts without changing already stored history.

`actuationKey` MUST equal `keccak256(abi.encode(block.chainid, authorizationRegistry, intentHash))`. The latest-receipt pointer MUST be scoped by `actuationKey`. `getLatestReceipt` MUST return a zero identifier and zero-valued receipt before the first entry, and the exact latest entry afterward. Every accepted receipt MUST also be stored immutably by `receiptId`; `getReceipt` MUST return the complete receipt and MUST revert with `UnknownReceipt` for an unknown identifier. `receiptExists` MUST report whether that immutable entry exists. No receipt may be deleted or overwritten.

### Authorization-signer receipt typed data and transitions

The receipt registry MUST use [EIP-712](./eip-712.md) with domain `{name: "Physical Actuation Receipt", version: "1", chainId, verifyingContract: receiptRegistry}` and MUST expose it through ERC-5267.

```solidity
bytes32 constant DEVICE_RECEIPT_TYPEHASH = keccak256(
    "DeviceReceipt(address authorizationRegistry,bytes32 intentHash,bytes32 authorizationHash,bytes32 previousReceiptId,uint64 sequence,uint64 actuationNonce,uint8 phase,uint64 deviceTime,bytes32 telemetryRoot,bytes32 resultHash,bytes32 localSafetyStateHash)"
);
```

`receiptId` MUST be the full EIP-712 digest of those fields in that order. For an EOA authorization signer, the registry MUST accept the 65-byte `(r,s,v)` form with `v` equal to 27 or 28 and MAY accept the 64-byte [EIP-2098](./eip-2098.md) compact form. It MUST enforce low-`s` signatures and recover `authorization.device.authorizationSigner`. For a contract authorization signer, it MUST call [ERC-1271](./eip-1271.md) on that address with `(receiptId, signerSignature)` and accept only magic value `0x1626ba7e`; revert, malformed return, and every other value are invalid. Any account MAY relay a valid receipt. Successful verification proves only that the authorization-designated signer made the assertion.

Before accepting a receipt, the registry MUST require all of the following:

1. `isAuthorizationProfile(receipt.authorizationRegistry) == true` and `authorizationRegistry` contains code.
2. A strict static call to `authorizationRegistry.supportsInterface(0xd3744de5)` succeeds and returns exactly the canonical ABI encoding of `true`.
3. A strict static call to `authorizationRegistry.getActuation(receipt.intentHash)` succeeds, decodes canonically, and returns `authorized == true` plus the complete stored authorization.
4. The returned authorization's `actuationNonce` equals `receipt.actuationNonce`.
5. The registry locally recomputes `resourceKey`, stable `deviceKey`, `deviceAuthorizationKey`, and `authorizationStructHash` using the formulas in this ERC, then computes:

```solidity
expectedAuthorizationHash = keccak256(abi.encode(
    ACTUATION_AUTHORIZATION_DOMAIN_TYPEHASH,
    block.chainid,
    receipt.authorizationRegistry,
    authorizationStructHash
));
```

6. The hash returned by `getActuation`, `receipt.authorizationHash`, and `expectedAuthorizationHash` are identical.
7. `signerSignature` validates for the returned authorization's `device.authorizationSigner` as specified above.

The receipt registry MUST perform the hash calculation itself and MUST NOT rely on the authorization profile's `hashAuthorization` result. These checks bind a receipt to an allowlisted profile deployment, stable device identity key, designated authorization signer, and exact authorization rather than merely to a reusable `intentHash`. ERC-165 support is an interface claim, not evidence that the allowlisted implementation is honest.

The phase MUST not be `NONE`; `deviceTime` and `localSafetyStateHash` MUST be nonzero. The first receipt MUST use sequence 1 and a zero previous identifier. Each later receipt MUST use `previousReceiptId == latestReceiptId`, `sequence == previous.sequence + 1`, and `deviceTime >= previous.deviceTime`. If the previous sequence is `type(uint64).max`, submission MUST revert.

Only these transitions are valid:

```text
NONE         -> ACKNOWLEDGED | REJECTED
ACKNOWLEDGED -> STARTED | FAILED | ABORTED
STARTED      -> COMPLETED | FAILED | ABORTED
```

`COMPLETED`, `FAILED`, `ABORTED`, and `REJECTED` are terminal. No receipt may follow them. `ACKNOWLEDGED` and `STARTED` MUST be recorded before the authorization deadline. A terminal receipt MAY be recorded later to tolerate delayed delivery.

Receipt evidence fields MUST use these canonical combinations:

| Phase | `telemetryRoot` | `resultHash` |
|---|---|---|
| `ACKNOWLEDGED` | `bytes32(0)` | `bytes32(0)` |
| `STARTED` | `bytes32(0)` | `bytes32(0)` |
| `COMPLETED` | nonzero and not `EMPTY_TELEMETRY_ROOT` | nonzero |
| `FAILED` | `EMPTY_TELEMETRY_ROOT` or a nonempty root | nonzero |
| `ABORTED` | `EMPTY_TELEMETRY_ROOT` or a nonempty root | nonzero |
| `REJECTED` | `EMPTY_TELEMETRY_ROOT` | nonzero |

Thus only `COMPLETED` requires at least one committed telemetry leaf. A failed or aborted action can legitimately produce no telemetry, and a rejection occurs before a start receipt. A nonempty root MUST be constructed from at least one leaf under the telemetry algorithm below; arbitrary nonzero values are invalid.

After validation, the registry MUST reject a duplicate `receiptId`, store the complete receipt by identifier, update the latest pointer, emit `DeviceReceiptRecorded` with the locally computed stable `deviceKey` and `recordedAt == uint64(block.timestamp)`, and return `receiptId`. `deviceTime` is an authorization-signer assertion and MUST NOT replace on-chain recording time.

Authorization lookup and ERC-1271 validation invoke external contracts. Submission MUST be non-reentrant, or the implementation MUST re-evaluate the authorization, latest receipt, sequence, previous identifier, terminal state, and duplicate identifier after external calls and immediately before its atomic state update.

### Telemetry root

```solidity
struct TelemetryLeaf {
    uint64 index;
    uint64 observedAt;
    bytes32 channelId;
    bytes32 valueHash;
}

bytes32 constant TELEMETRY_LEAF_TYPEHASH = keccak256(
    "TelemetryLeaf(uint64 index,uint64 observedAt,bytes32 channelId,bytes32 valueHash)"
);
bytes32 constant EMPTY_TELEMETRY_ROOT =
    keccak256("PHYSICAL_ACTUATION_EMPTY_TELEMETRY_V1");
```

Leaves MUST be ordered by contiguous `index` starting at zero. A leaf is `keccak256(abi.encodePacked(bytes1(0x00), abi.encode(TELEMETRY_LEAF_TYPEHASH, index, observedAt, channelId, valueHash)))`. An internal node is `keccak256(abi.encodePacked(bytes1(0x01), left, right))`. At a level with an odd node count, the final node MUST be paired with itself. The empty set root is `EMPTY_TELEMETRY_ROOT`.

`valueHash` MAY commit to plaintext canonical bytes or encrypted bytes as declared by `telemetryPolicyHash`. That policy MUST identify required channels, sampling, encodings, encryption, retention, and evidence-access rules. A valid root proves commitment consistency only, not sensor correctness or availability.

## Rationale

### Reusing ERC-8001

ERC-8001 already provides a multi-party proposal, acceptance, expiry, initiator replay, and execution lifecycle. This profile adds physical semantics without creating an incompatible generic intent protocol. Exact field-wise payload hashing and participant conditions binding are retained so profile-aware and generic ERC-8001 tooling agree on the same signed data.

### Authorization, actuation, and reporting are distinct

On-chain execution records permission and consumes a profile-scoped, identity-stable actuation nonce. It does not call a motor controller. A separate receipt registry distinguishes approval, signer acknowledgement, claimed start, and claimed outcome. Separation also gives the authorization and receipt protocols unambiguous EIP-712 verifying contracts.

### Live supersession checks

A hash of a safety snapshot alone cannot reveal a recall published after proposal. Typed entries let the profile verify the exact statement at proposal and again during execution. Treating every newer statement as superseding avoids asking a generic coordinator to interpret severity-specific policy.

### Relationship to adjacent ERCs

[ERC-4519](./eip-4519.md) supports physical asset addresses and owner or user engagement. [ERC-7777](./eip-7777.md) supplies robot hardware identity and governance rules. [ERC-6956](./eip-6956.md) represents assets that may lack signing capability. Coordinates from those systems can be used as `device.identity`, while the participants separately designate an authorization signer such as a controller, gateway, or oracle. This ERC does not infer that the signer is the referenced asset or controls it.

[ERC-8196](./eip-8196.md) constrains AI-agent wallet transactions and logs on-chain actions. It does not define a physical command envelope, safety-status checkpoint, identity-stable actuation nonce, or authorization-signer outcome. An agent wallet may participate in this profile while the actuating system retains independent local controls.

[ERC-8301](./eip-8301.md) records AI-agent workflow execution, tasks, and results. Its generic execution record does not define this profile's physical operating envelope, exact device and release binding, live multi-target safety snapshot, stable-device nonce, or authorization-signer receipt transitions.

[ERC-8312](./eip-8312.md) meters bounded agent actions against a verifiable state cursor across calls. [ERC-8370](./eip-8370.md) defines inheritable agent mandates whose authority and constraints can flow to descendants. Those standards bound or delegate agent authority; they do not specialize a multi-party authorization as a physical command or require its safety snapshot to remain current at execution. A participant governed by either standard can still take part in this profile.

[ERC-8269](./eip-8269.md) defines a scoped, expiring lease between an agent-memory subject and a hardware or runtime body, plus portable capsule and credential-broker conventions. That lease can supply a device relationship or identity reference, but it does not authorize an individual actuation, consume a deployment-scoped device nonce, recheck live safety statements, or attribute a sequenced device receipt.

[ERC-8380](./eip-8380.md) couples a single-use agent execution credential to an authorized action through a nullifier guard. Its at-most-once credential-consumption property is distinct from the identity-stable actuation nonce, multi-party ERC-8001 approval, physical envelope, live safety checks, and signer-attributed receipts defined here. Neither interface proves that physical motion occurred.

*Recomputable Verification Receipts*, currently a working proposal in PR #1980, focuses on evidence that a computation can be independently rerun or checked. Such evidence can be committed by `resultHash`; it does not prove physical occurrence. These related-work references intentionally create no normative dependency.

## Backwards Compatibility

This ERC is additive. It does not change ERC-8001 typed data, status codes, or base interface. A generic ERC-8001 deployment is not profile-compliant without the additional payload checks, status-registry configuration, authorization storage, and nonce state. Receipt registries are independent and may serve multiple compliant profile deployments. Existing robot identity contracts can be referenced without modification.

## Test Cases

1. Compute an ERC-8001 payload hash using the six field-wise values. Confirm it matches `getPayloadHash` and differs from `keccak256(abi.encode(payload))` for dynamic data.
2. Propose a canonical authorization with two sorted participants, two sorted safety entries, and a nonzero authorization signer. Confirm every snapshot ID and sequence is current and the snapshot hash matches the embedded entries.
3. Change an entry's registry, issuer, target, sequence, status identifier, or maximum age without recomputing all commitments. Proposal must revert.
4. Submit an otherwise valid participant acceptance whose `conditionsHash` differs from `actuationConditionsHash`. Acceptance must revert without advancing status.
5. After all acceptances, publish a newer safety statement for one entry. Execution must revert and the ERC-8001 intent must remain unexecuted.
6. Execute before `notBefore`, at `deadline`, and at `recallValidUntil`. Each call must revert without consuming the identity-scoped actuation nonce.
7. Execute a valid intent. Confirm atomic storage of the authorization and safety entries, nonce advancement, `Executed` status, event fields, and `(true, abi.encode(authorizationHash))`. Confirm the hash changes on another chain or profile address.
8. Change only `authorizationSigner`. Confirm `deviceKey` is unchanged, `deviceAuthorizationKey` and `authorizationHash` change, and the same stable device nonce high-water mark still applies.
9. Attempt a second execution for the same profile and device identity with an equal or lower actuation nonce but a fresh signer and ERC-8001 initiator nonce. It must revert.
10. Allowlist the profile in a separate receipt registry, verify ERC-165 ID `0xd3744de5`, locally recompute its authorization hash, and submit a receipt signed by the authorization-designated signer. Confirm the immutable getter and latest pointer.
11. Repeat case 10 with an unallowlisted profile, false or malformed ERC-165 response, locally mismatched hash, or signature from a key other than the authorization-designated signer. Each must revert without changing receipt state.
12. Record canonical `ACKNOWLEDGED`, `STARTED`, and `COMPLETED` receipts. Nonzero evidence in either nonterminal phase and zero or empty completion telemetry must revert. Confirm `REJECTED` requires the empty root, while `FAILED` and `ABORTED` accept either empty or nonempty telemetry with a nonzero result.
13. Test skipped sequences, wrong previous IDs, decreasing device time, invalid phase jumps, post-terminal receipts, 65-byte and EIP-2098 EOA signatures, high-`s` rejection, ERC-1271 success and failure, sequence overflow, exact interface IDs, and malformed external returns.

## Security Considerations

### Physical safety remains local

Ethereum latency, finality, censorship resistance, and availability are unsuitable for inner-loop control or emergency stopping. A robot should treat authorization as one input to a local safety supervisor. Hardware stops, watchdogs, force and speed limits, collision avoidance, geofencing, human override, and certified control logic must remain able to reject or interrupt every authorized action. The device should recheck locally available recall and safety data immediately before motion, because chain execution and physical receipt can be separated in time.

### Registry trust and safety races

The on-chain execution check prevents a superseded configured statement from silently passing. It cannot make a malicious issuer or registry truthful, prevent a recall on an unconfigured registry, or cover a statement published after execution. Configuration governance is therefore safety-critical. Devices should pin the expected chain, profile, status registries, issuers, and confirmation policy and fail closed when freshness cannot be established.

The receipt registry's authorization-profile allowlist is a separate trust root. A malicious allowlisted profile can invent authorizations or designated signers. ERC-165 checking and local hash recomputation prevent interface confusion and inconsistent records; they cannot establish that an allowlisted profile applied honest policy. Trust changes should use governance, delay, and monitoring appropriate to the physical consequence.

### Authorization signers and delayed receipts

A receipt signature proves control of the authorization-designated key only. It does not prove possession of uncloned hardware, correct firmware, a connection to `device.identity`, or truthful sensors. Where deployments assert such a relationship, hardware-protected keys, measured boot, rotation, revocation, and independent attestation may be needed outside this ERC. Because terminal receipts may arrive after the deadline, a compromised historical authorization signer can attempt to backfill claims. Consumers should apply a bounded reporting window or key-validity policy appropriate to the consequence; this ERC does not define a universal one.

### Deployment-scoped nonces

Actuation nonces are not global. An identity used across multiple chains or profile contracts has a separate high-water mark for every `(chainId, profileContract, deviceKey)` coordinate. Changing only the authorization signer does not create a new nonce stream. Migration requires explicit state transfer or a policy that rejects prior deployments. Losing contract, bridge, gateway, or actuating-system nonce state can enable replay. Actuating systems should persist acted-on nonces in rollback-resistant storage.

The authorization hash binds a deployment address, not its runtime bytecode. Proxy upgrades can therefore change validation logic without changing the coordinate. Safety-critical consumers should pin an immutable implementation or independently govern and audit upgrade authority.

### Receipts and telemetry are assertions

A `COMPLETED` receipt does not prove motion, delivery, or sensor accuracy. Physical causation can require independent sensors, observers, secure hardware, calibration, and investigation. A Merkle root proves consistency with later-presented leaves, not that data are correct or available.

Receipt-chain linearity applies within one receipt registry. Two registries can record parallel histories for the same authorization. Consumers should pin a canonical receipt-registry coordinate or explicitly reconcile every registry they accept.

### Profile interpretation and availability

`actionProfileId`, envelope, safety, telemetry, and result commitments are intentionally profile identifiers rather than global schemas. This preserves extensibility but leaves semantic interoperability to separately governed profiles. Unknown, unavailable, ambiguous, or unbounded profile data should fail closed. Low-entropy private data should use salted commitments or encryption; hashing alone does not provide confidentiality.

### Coordinator and participant compromise

ERC-8001 proves that configured addresses accepted an intent; it does not prove the right organizations or people were configured. A compromised participant set can authorize harmful commands. Devices should pin audited profile deployments and independently validate the full authorization, safety snapshot, participant policy, deadline, and nonce rather than trusting an event alone.

### Timing, reorganizations, and denial of service

Block timestamps have limited precision and device clocks may be wrong. A robot acting before sufficient finality may rely on an authorization later removed by reorganization. Required margins and confirmation depth depend on physical consequence. Attackers can propose conflicting intents, withhold acceptance, make registry calls revert, or relay receipts near deadlines. Implementations should bound parsing and external-call gas and never make receipt submission necessary for entering a safe state.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
