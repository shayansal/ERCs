---
title: Autonomous Release Lineage
description: Immutable system release manifests, dependency graphs, lineage, and deployment bindings for autonomous systems
author: Shayan Salehi (@shayansal)
discussions-to: https://ethereum-magicians.org/t/TBD
status: Draft
type: Standards Track
category: ERC
created: 2026-08-31
requires: 165
---

## Abstract

This ERC defines an immutable registry for publisher-asserted releases of autonomous software and machines. A registered release records a canonical system-manifest hash, a Merkle root of its bill of materials, previously registered release dependencies, and an optional publisher-asserted lineage parent. Registration proves who recorded a tuple and when; conformance additionally requires off-chain validation that the committed manifest reproduces that tuple. An optional append-only extension binds a controller-scoped subject, such as an agent deployment or robot fleet unit, to an exact registered release. Neither interface asserts that committed bytes are safe, available, authorized by a parent publisher, or actually running.

## Motivation

An autonomous system is rarely one model. Its behavior can depend on model weights, prompts, policies, tool adapters, retrieval corpora, runtime libraries, firmware, calibration data, verifier keys, and deployment configuration. A mutable metadata URI or a model identifier cannot answer the operationally important question: *which complete system was authorized at a particular time?*

This gap prevents reliable composition across organizations. An auditor may assess one prompt and model pair while an operator deploys another. A robot may retain old firmware after its control service changes. A payment protocol may authorize an agent by identity while being unable to distinguish a reviewed release from an unreviewed update. A recall, assurance policy, procurement rule, or incident investigation therefore has no common, machine-readable release coordinate.

A blockchain is useful here for a narrow reason. The publisher, operator, auditor, insurer, marketplace, and affected user may not share a database administrator. Ethereum gives all of them one ordered, tamper-evident history that contracts can inspect before releasing funds or permissions. A private database can inventory the same artifacts, but it cannot give unaffiliated contracts a neutral record that is atomically usable during settlement. This ERC makes that record possible without putting proprietary artifacts on chain or claiming that a hash establishes quality.

## Specification

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://www.rfc-editor.org/rfc/rfc2119) and [RFC 8174](https://www.rfc-editor.org/rfc/rfc8174).

### Terms

- A **publisher** is the address that registers an immutable release.
- A **component** is one content-addressed item in a release, including a model, prompt, policy, tool adapter, runtime, firmware image, calibration bundle, or configuration bundle.
- A **registered release** is an immutable tuple accepted by a conforming registry. Registry acceptance alone does not prove that referenced manifest bytes are available or conforming.
- A **conforming release** is a registered release whose retrieved manifest bytes match `manifestHash`, decode under this ERC, reproduce the registered component root and counts, and contain the identical dependency array.
- A **release dependency** is another release the publisher asserts is required by the new release. Dependencies are distinct from lineage: a parent is a unilateral derivation claim, while a dependency is a composition claim. Neither claim implies approval by another publisher.
- A **release digest** is deterministic for the exact digest-preimage tuple `(manifestHash, componentsRoot, componentCount, dependenciesRoot, dependencyCount)` and is independent of publisher, nonce, chain, registry, URI, and lineage-parent claim. It is not a globally canonical identity for semantically corresponding releases across registries because `manifestHash` commits to registry-scoped dependency release IDs.
- A **controller** is the address making an append-only claim about the release assigned to a subject.
- A **subject** is the controller-scoped tuple `(subjectNamespace, subjectId)`. This ERC does not confer ownership of an external identity represented by that tuple.

### Domain values

Implementations MUST compute domain values as `keccak256` of the exact UTF-8 strings below:

```text
RELEASE_ID_DOMAIN             = "AUTONOMOUS_RELEASE_ID_V1"
RELEASE_DIGEST_DOMAIN         = "AUTONOMOUS_RELEASE_DIGEST_V1"
MANIFEST_DOMAIN               = "AUTONOMOUS_RELEASE_MANIFEST_V1"
SEMANTIC_ID_DOMAIN            = "AUTONOMOUS_SEMANTIC_ID_V1"
COMPONENT_LEAF_DOMAIN         = "AUTONOMOUS_COMPONENT_LEAF_V1"
COMPONENT_EMPTY_DOMAIN        = "AUTONOMOUS_COMPONENT_EMPTY_V1"
COMPONENT_NODE_DOMAIN         = "AUTONOMOUS_COMPONENT_NODE_V1"
DEPENDENCY_LEAF_DOMAIN        = "AUTONOMOUS_DEPENDENCY_LEAF_V1"
DEPENDENCY_EMPTY_DOMAIN       = "AUTONOMOUS_DEPENDENCY_EMPTY_V1"
DEPENDENCY_NODE_DOMAIN        = "AUTONOMOUS_DEPENDENCY_NODE_V1"
SUBJECT_KEY_DOMAIN            = "AUTONOMOUS_SUBJECT_KEY_V1"
BINDING_ID_DOMAIN             = "AUTONOMOUS_BINDING_ID_V1"
```

All hashes in this ERC use EVM `keccak256`. A field described as a hash MUST NOT be interpreted as an identifier for a different digest algorithm.

`systemClass` and every component `kind` MUST be a semantic identifier computed from nonempty, case-sensitive UTF-8 `namespace` and `name` strings as:

```solidity
semanticId = keccak256(abi.encode(
    SEMANTIC_ID_DOMAIN,
    keccak256(bytes(namespace)),
    keccak256(bytes(name))
));
```

The manifest or corresponding descriptor MUST disclose the exact namespace and name strings. Profiles MAY define shared values. This rule permits domain-specific vocabularies without making an unnamespaced short label appear globally standardized.

### Canonical release manifest

The bytes committed by `manifestHash` MUST be the ABI encoding of:

```solidity
struct ComponentV1 {
    bytes32 kind;
    bytes32 artifactHash;
    bytes32 descriptorHash;
    uint32[] dependsOn;
}

abi.encode(
    MANIFEST_DOMAIN,
    systemClass,               // bytes32
    configurationHash,         // bytes32
    components,                // ComponentV1[]
    dependencyReleaseIds       // bytes32[]
)
```

`systemClass`, `configurationHash`, every `kind`, every `artifactHash`, and every `descriptorHash` MUST be nonzero. `artifactHash` MUST equal `keccak256` of the exact artifact bytes. A component descriptor MAY contain retrieval locations, media type, build procedure, supplier identity, hardware constraints, or an alternate digest, but `descriptorHash` MUST equal `keccak256` of the exact descriptor bytes.

`components` MUST contain at least one item. For component index `i`, `dependsOn` MUST be strictly increasing, contain no duplicate, and contain only indices less than `i`. These requirements make the component ordering a topological ordering and prohibit cycles. The same artifact MAY appear in more than one component only when its other component fields differ.

`dependencyReleaseIds` MUST equal the array supplied to `registerRelease`. It MUST be strictly increasing by numeric `bytes32` value and contain no duplicate. This canonical order makes equivalent dependency sets hash identically.

The component leaf at index `i` is:

```solidity
dependsOnHash = keccak256(abi.encode(components[i].dependsOn));
leaf = keccak256(abi.encode(
    COMPONENT_LEAF_DOMAIN,
    components[i].kind,
    components[i].artifactHash,
    components[i].descriptorHash,
    dependsOnHash
));
```

The component tree preserves leaf order. Its leaf count is padded to the smallest power of two greater than or equal to `componentCount` using:

```solidity
emptyLeaf = keccak256(abi.encode(COMPONENT_EMPTY_DOMAIN));
parent = keccak256(abi.encode(COMPONENT_NODE_DOMAIN, left, right));
```

The dependency leaf and tree use the same construction with their corresponding domains:

```solidity
leaf = keccak256(abi.encode(DEPENDENCY_LEAF_DOMAIN, dependencyReleaseId));
emptyLeaf = keccak256(abi.encode(DEPENDENCY_EMPTY_DOMAIN));
parent = keccak256(abi.encode(DEPENDENCY_NODE_DOMAIN, left, right));
```

For zero dependencies, `dependenciesRoot` MUST equal `keccak256(abi.encode(DEPENDENCY_EMPTY_DOMAIN))`. There is no zero-component root because zero components are invalid. A proof MUST contain exactly `log2(nextPowerOfTwo(count))` siblings. Starting at the leaf, bit `k` of the zero-based index determines whether the running hash is the left or right child at level `k`.

### Release registry interface

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

interface IAutonomousReleaseRegistry /* is IERC165 */ {
    struct Release {
        address publisher;
        bytes32 releaseDigest;
        bytes32 manifestHash;
        bytes32 componentsRoot;
        bytes32 dependenciesRoot;
        bytes32 parentReleaseId;
        uint32 componentCount;
        uint32 dependencyCount;
        uint64 publisherNonce;
        uint64 registeredAt;
        string manifestURI;
    }

    event ReleaseRegistered(
        bytes32 indexed releaseId,
        address indexed publisher,
        bytes32 indexed parentReleaseId,
        bytes32 releaseDigest,
        bytes32 manifestHash,
        bytes32 componentsRoot,
        bytes32 dependenciesRoot,
        uint32 componentCount,
        uint32 dependencyCount,
        uint64 publisherNonce,
        string manifestURI
    );

    error InvalidReleaseField();
    error InvalidManifestURI();
    error UnknownRelease(bytes32 releaseId);
    error InvalidDependencyOrder();

    function registerRelease(
        bytes32 manifestHash,
        bytes32 componentsRoot,
        uint32 componentCount,
        bytes32 parentReleaseId,
        bytes32[] calldata dependencyReleaseIds,
        string calldata manifestURI
    ) external returns (bytes32 releaseId);

    function getRelease(bytes32 releaseId)
        external
        view
        returns (Release memory release);

    function releaseExists(bytes32 releaseId) external view returns (bool);

    function nextPublisherNonce(address publisher)
        external
        view
        returns (uint64);

    function computeReleaseDigest(
        bytes32 manifestHash,
        bytes32 componentsRoot,
        uint32 componentCount,
        bytes32 dependenciesRoot,
        uint32 dependencyCount
    ) external pure returns (bytes32 releaseDigest);

    function verifyComponent(
        bytes32 releaseId,
        uint32 index,
        bytes32 kind,
        bytes32 artifactHash,
        bytes32 descriptorHash,
        uint32[] calldata dependsOn,
        bytes32[] calldata proof
    ) external view returns (bool);

    function verifyDependency(
        bytes32 releaseId,
        uint32 index,
        bytes32 dependencyReleaseId,
        bytes32[] calldata proof
    ) external view returns (bool);
}
```

The `IAutonomousReleaseRegistry` interface ID is `0xa13eda3b`, the XOR of these function selectors in declaration order:

```text
0xd0fc8c33  registerRelease(bytes32,bytes32,uint32,bytes32,bytes32[],string)
0x7f698e92  getRelease(bytes32)
0x3f415772  releaseExists(bytes32)
0x4911c386  nextPublisherNonce(address)
0x4ec28ba1  computeReleaseDigest(bytes32,bytes32,uint32,bytes32,uint32)
0x57f5bc28  verifyComponent(bytes32,uint32,bytes32,bytes32,bytes32,uint32[],bytes32[])
0x61cc7be7  verifyDependency(bytes32,uint32,bytes32,bytes32[])
```

The registry MUST implement [ERC-165](./eip-165.md) and return `true` for `type(IAutonomousReleaseRegistry).interfaceId`.

`registerRelease` MUST reject zero `manifestHash`, zero `componentsRoot`, zero `componentCount`, a URI whose UTF-8 byte length is zero or greater than 2,048, a dependency count that exceeds `uint32`, and a timestamp or nonce that does not fit `uint64`. It MUST verify that `parentReleaseId`, when nonzero, already exists in the same registry. A parent is asserted only by the new publisher; its existence MUST NOT be treated as consent, endorsement, compatibility, or inherited trust by the parent publisher. The registry MUST also verify that every dependency already exists in the same registry and that the dependency array is strictly increasing. Because parents and dependencies must pre-exist, both the lineage relation and inter-release dependency relation are acyclic.

The registry MUST compute `dependenciesRoot` from the supplied array. `computeReleaseDigest` MUST return, and registration MUST compute:

```solidity
releaseDigest = keccak256(abi.encode(
    RELEASE_DIGEST_DOMAIN,
    manifestHash,
    componentsRoot,
    componentCount,
    dependenciesRoot,
    uint32(dependencyReleaseIds.length)
));
```

The digest deliberately excludes publisher, nonce, chain, registry, URI, and `parentReleaseId`. It identifies identical digest-preimage tuples even when publishers or lineage claims differ. It is deterministic only for that tuple, not a globally canonical semantic-release identity: because `manifestHash` contains registry-scoped dependency release IDs, corresponding releases represented in different registries can have different digests.

The registry MUST then use the publisher's current nonce and compute:

```solidity
releaseId = keccak256(abi.encode(
    RELEASE_ID_DOMAIN,
    block.chainid,
    address(this),
    msg.sender,
    publisherNonce,
    releaseDigest,
    parentReleaseId
));
```

The registry MUST reject a derived ID that already exists. On success it MUST store `msg.sender` as publisher, the computed release digest, every supplied or derived `Release` field, set `registeredAt` to `uint64(block.timestamp)`, increment the publisher nonce, and emit exactly one `ReleaseRegistered`. No release field may subsequently change or be deleted. Multiple registrations MAY have the same release digest but MUST have different release IDs. `releaseExists` reports registration existence only; it MUST return `false` rather than revert for an unknown identifier. `getRelease` MUST revert with `UnknownRelease` for one.

The registry intentionally does not receive or inspect the complete manifest bytes. A release is conforming only if the retrievable bytes match `manifestHash`, decode as specified, reproduce `componentsRoot` and `componentCount`, and contain the identical dependency array. Consumers MUST perform those checks before treating a registered release as conforming. Registration, `releaseExists`, and a matching release digest do not substitute for those checks.

`verifyComponent` MUST reject a `dependsOn` array that is not strictly increasing or contains an index greater than or equal to the component index. It MUST compute `dependsOnHash = keccak256(abi.encode(dependsOn))` before computing the leaf. `verifyComponent` and `verifyDependency` MUST return `false` for an invalid component dependency array, incorrect proof, or out-of-range index and MUST revert with `UnknownRelease` only when the release is absent. They MUST NOT fetch any URI. A successful proof establishes inclusion in the registered root; it does not establish that every manifest component was encoded conformingly.

### Optional Subject Binding Extension

The subject binding extension is OPTIONAL. Implementations that expose it MUST conform to this section; a release registry remains conforming without it. The extension MAY share a contract with the release registry or be deployed separately.

```solidity
// SPDX-License-Identifier: CC0-1.0
pragma solidity ^0.8.24;

interface IAutonomousReleaseBinding /* is IERC165 */ {
    struct Binding {
        address controller;
        bytes32 subjectNamespace;
        bytes32 subjectId;
        address releaseRegistry;
        bytes32 releaseId;
        bytes32 previousBindingId;
        bytes32 reasonHash;
        uint64 sequence;
        uint64 boundAt;
        bool active;
    }

    event SubjectReleaseBound(
        bytes32 indexed subjectKey,
        bytes32 indexed bindingId,
        bytes32 indexed releaseId,
        address controller,
        bytes32 subjectNamespace,
        bytes32 subjectId,
        address releaseRegistry,
        bytes32 previousBindingId,
        uint64 sequence
    );

    event SubjectReleaseDeactivated(
        bytes32 indexed subjectKey,
        bytes32 indexed bindingId,
        address indexed controller,
        bytes32 subjectNamespace,
        bytes32 subjectId,
        bytes32 previousBindingId,
        bytes32 reasonHash,
        uint64 sequence
    );

    error InvalidSubject();
    error InvalidReleaseRegistry();
    error ReleaseNotFound(bytes32 releaseId);
    error StalePreviousBinding(bytes32 expected, bytes32 actual);
    error UnknownBinding(bytes32 bindingId);

    function bindSubject(
        bytes32 subjectNamespace,
        bytes32 subjectId,
        address releaseRegistry,
        bytes32 releaseId,
        bytes32 expectedPreviousBindingId
    ) external returns (bytes32 bindingId);

    function deactivateSubject(
        bytes32 subjectNamespace,
        bytes32 subjectId,
        bytes32 expectedPreviousBindingId,
        bytes32 reasonHash
    ) external returns (bytes32 bindingId);

    function subjectKey(
        address controller,
        bytes32 subjectNamespace,
        bytes32 subjectId
    ) external pure returns (bytes32);

    function currentBinding(
        address controller,
        bytes32 subjectNamespace,
        bytes32 subjectId
    ) external view returns (bytes32 bindingId);

    function getBinding(bytes32 bindingId)
        external
        view
        returns (Binding memory binding);

    function bindingExists(bytes32 bindingId) external view returns (bool);
}
```

The `IAutonomousReleaseBinding` interface ID is `0x2b9b2a5a`, the XOR of these function selectors in declaration order:

```text
0x9885f538  bindSubject(bytes32,bytes32,address,bytes32,bytes32)
0x15b2b9c8  deactivateSubject(bytes32,bytes32,bytes32,bytes32)
0xa8595c3c  subjectKey(address,bytes32,bytes32)
0x1198c8a5  currentBinding(address,bytes32,bytes32)
0xf55cafcb  getBinding(bytes32)
0xea315df8  bindingExists(bytes32)
```

A binding registry MUST implement ERC-165 and advertise `type(IAutonomousReleaseBinding).interfaceId`.

`subjectNamespace` and `subjectId` MUST be nonzero. `subjectKey` MUST return:

```solidity
keccak256(abi.encode(
    SUBJECT_KEY_DOMAIN,
    controller,
    subjectNamespace,
    subjectId
))
```

For `bindSubject`, the controller and stored `Binding.controller` MUST be `msg.sender`, and `releaseId` MUST be nonzero. `releaseRegistry` MUST contain deployed code. Strict static calls to `supportsInterface(type(IAutonomousReleaseRegistry).interfaceId)` and `releaseExists(releaseId)` MUST each return exactly one canonical ABI-encoded `true`. A revert, malformed return, or `false` MUST cause the binding call to revert. Interface support confirms only the claimed ABI; it does not establish release conformance or registry trust. `expectedPreviousBindingId` MUST equal `currentBinding` for `subjectKey(msg.sender, subjectNamespace, subjectId)` and MUST be zero for the first binding. This compare-and-set rule prevents two concurrent updates from silently overwriting each other.

Each successful transition increments a per-subject `uint64` sequence from zero; the first record has sequence one, and overflow MUST revert. A binding ID MUST be:

```solidity
keccak256(abi.encode(
    BINDING_ID_DOMAIN,
    block.chainid,
    address(this),
    subjectKey,
    sequence,
    releaseRegistry,
    releaseId,
    expectedPreviousBindingId,
    reasonHash,
    active
))
```

For activation, `active` is `true`, `reasonHash` is zero, and the supplied release fields are stored. For deactivation, the controller MUST be `msg.sender`, `active` is `false`, `releaseRegistry` and `releaseId` are zero, and `reasonHash` MUST be nonzero. `boundAt` MUST be `uint64(block.timestamp)`. Every binding record MUST be immutable and retained after it ceases to be current. A derived binding ID that already exists MUST be rejected. `currentBinding` MUST return zero before a subject's first transition. `getBinding` MUST revert for an unknown ID; `bindingExists` MUST return `false` for one.

The controller scope is intentional. A binding says only that `controller` made the claim. A consumer using an external identity registry MUST independently verify that the controller was authorized for that identity at the binding time.

## Rationale

### Whole-system releases rather than model records

[ERC-7992](./eip-7992.md) commits a model, circuit, and verification key for provable inference. That is valuable for one computation but does not identify prompts, tools, firmware, or deployment configuration. This ERC instead names the complete operational release and does not verify inference correctness.

[ERC-8257](./eip-8257.md) discovers individual agent tools and permits their metadata to change. A tool may be one leaf in this ERC's bill of materials. Making releases immutable prevents a historical approval from silently following a tool update.

[ERC-8303](./eip-8303.md) exposes a smart contract implementation version string for runtime discovery. It identifies the version reported by one contract, whereas this ERC commits an immutable release manifest spanning mixed models, prompts, tools, firmware, policies, configuration, and contract components, plus release dependencies and optional deployment bindings. An ERC-8303-versioned contract can therefore be one component of a release; its version string is not a substitute for the release coordinate or manifest commitment.

### Separation from identity and trust scores

[ERC-8004](./eip-8004.md) provides agent identity, mutable discovery metadata, reputation, and validation hooks. A controller can derive a subject namespace and identifier from such an identity, but this ERC neither mints identity nor judges trust. A relying contract can require both an accepted identity and an accepted release binding.

### Append-only updates

Updating a release in place would destroy the exact coordinate needed by audits, recalls, and incident response. New registrations therefore receive new IDs. `releaseDigest` supplies a deterministic key for the registered tuple without erasing publisher-specific registration history; it is not a cross-registry semantic identity. Optional binding records are also append-only, while a current pointer makes ordinary reads inexpensive. The expected-previous parameter makes update races explicit.

### Explicit dependencies and Merkle components

Storing every component would make large software bills of materials expensive. A Merkle root permits inexpensive inclusion checks. Release dependencies are supplied during registration because checking that they already exist is what gives the cross-release graph an enforceable acyclic order. This differs from [ERC-6224](./eip-6224.md), which manages live smart-contract dependency injection and upgrades rather than immutable off-chain system artifacts.

### Relationship to Package Registries and BOM Formats

[ERC-2678](./eip-2678.md) defines an EthPM JSON package manifest for smart-contract sources, builds, deployments, and dependencies. [ERC-1319](./eip-1319.md) defines registries for named EthPM package releases. This ERC does not replace either format. It addresses mixed autonomous systems whose committed components can include models, prompts, policies, remote-service descriptors, firmware, calibration, hardware constraints, and smart-contract packages; it also defines ordered component proofs, publisher-asserted lineage, and an optional subject-binding extension. An EthPM manifest can be one component or descriptor in a release.

SPDX and CycloneDX define broad off-chain software and AI bill-of-materials schemas and vocabularies. This ERC does not recreate those schemas or claim that its minimal `ComponentV1` is a complete BOM. A component descriptor can commit to an exact SPDX or CycloneDX document. The on-chain contribution here is a common immutable registration coordinate, digest, dependency ordering, and inclusion-proof boundary for whichever detailed BOM format a profile selects.

### Publisher-Asserted Parentage

Parentage records a new publisher's derivation claim without modifying the parent. Requiring the parent to pre-exist prevents cycles but cannot prove derivation or permission. Consumers must evaluate the child independently and must not inherit the parent's publisher identity, audit status, license, reputation, or authorization.

### No revocation flag

Facts about what was released must not disappear when a key is compromised or a product is recalled. Deprecation, recall, assurance, and policy decisions are separate claims that can reference the immutable release coordinate. A publisher can issue a corrected child release, and a controller can deactivate a subject, without rewriting history.

## Backwards Compatibility

This ERC introduces a new release registry and an optional binding extension without changing existing token, identity, package, BOM, or agent interfaces. Existing systems can adopt it by publishing releases and adding a release reference to their current metadata. The controller-scoped subject scheme permits integration without modifying the underlying identity contract. Contracts that do not understand these interfaces continue to operate unchanged.

## Test Cases

In these cases, `H(x)` means `keccak256(bytes(x))`, Alice is the publisher, and all calls occur on one registry.

1. Alice registers release A with `manifestHash = H("manifest-a")`, one component, no parent, and no dependencies. The call succeeds, `dependencyCount` is zero, `dependenciesRoot` equals `keccak256(abi.encode(DEPENDENCY_EMPTY_DOMAIN))`, `releaseDigest` matches the specified digest preimage, nonce zero is embedded in A's ID, and Alice's next nonce becomes one.
2. Registering the same fields again succeeds with nonce one and produces a different release ID but the same release digest. Neither record changes.
3. Registration with zero `componentCount`, an empty URI, an unknown parent, an unsorted dependency array `[B, A]` where `B > A`, or duplicate dependencies `[A, A]` reverts.
4. After A exists, Alice registers B with parent A and dependency array `[A]`. The call succeeds. Attempting to register a release that depends on an identifier not yet registered reverts, so B cannot later become an ancestor of A.
5. For a four-component manifest, a component at index two has a two-sibling proof and `dependsOn = [0, 1]`. `verifyComponent` returns `true` for the committed fields, `false` after changing `artifactHash`, `false` for `[1, 0]` or `[2]`, `false` with one sibling, and `false` for index four.
6. Controller C binds `(H("agent-identity"), bytes32(uint256(42)))` to release A with expected previous ID zero. The release registry reports both the required ERC-165 interface and release existence. The returned record has sequence one and becomes current. A second transaction also expecting zero reverts with `StalePreviousBinding`; a registry lacking the interface or returning malformed Boolean data is rejected.
7. C binds the same subject to B while supplying A's binding ID. The record has sequence two and points back to the first binding. Both records remain queryable.
8. C deactivates the subject with sequence-two binding as expected previous and `reasonHash = H("maintenance")`. The sequence-three record is current and inactive. Deactivation with a zero reason reverts.
9. Another controller using the same namespace and subject ID receives a different subject key and cannot alter C's current binding.

## Security Considerations

### Commitments are not verification

A matching hash proves byte equality, not safety, authorship, license, quality, or execution. A malicious publisher can register harmful artifacts and a conformant-looking manifest. A binding proves a controller claim, not that a device loaded the release. Relying systems need attestation, inspection, monitoring, or governance appropriate to their risk.

### Availability and confidential artifacts

Ethereum retains hashes and records, not referenced bytes. A URI can fail, become access-controlled, or serve unavailable content. Consumers should obtain and retain the manifest and required artifacts before accepting a release. Confidential artifacts can remain encrypted or privately distributed, but parties without the bytes cannot validate their commitments.

### Mutable and external behavior

An artifact hash cannot freeze an external API, changing retrieval corpus, sensor environment, hardware defect, or nondeterministic service. Publishers should represent each behavior-affecting dependency as a content-addressed component or release and should disclose unavoidable dynamic inputs in descriptors. Consumers must not infer reproducibility merely from manifest conformance.

### Merkle and encoding hazards

Implementations must preserve ABI encoding, domain separation, child order, padding, and exact proof length. Substituting sorted-pair trees, packed encoding, or a different empty-leaf rule can make proofs non-interoperable or ambiguous. Off-chain builders must validate topological indices and reproduce both roots before publication.

### Publisher and controller compromise

Compromised keys can publish releases or bind subjects, and immutable records cannot be erased. Publishers and controllers should use contract accounts with suitable key rotation and authorization policies. Consumers should evaluate the authority at the record's time and combine release history with separate compromise or recall signals.

### Namespace impersonation

Anyone can choose a namespace and subject ID. Controller scoping prevents overwriting but not misleading claims. An integration that maps a binding to an NFT, robot serial number, or organizational asset must authenticate the controller through that system's authority rules. Display software should show the controller and binding-registry address, not only a human-readable subject label.

### Cross-chain and registry scope

Release and binding IDs include chain ID and registry address. The same manifest on another chain is a different release coordinate. Bridges and indexers must preserve the full `(chainId, registry, id)` tuple. Chain reorganizations and L2 timestamp or finality assumptions apply to `registeredAt` and `boundAt`.

### Resource exhaustion

Dependency validation is linear in the submitted dependency count. Implementations may impose a documented maximum count, but must reject over-limit registrations rather than truncate them. URI length is capped to limit storage griefing. Consumers parsing manifests should bound nested arrays and descriptor sizes.

## Copyright

Copyright and related rights waived via [CC0](../LICENSE.md).
