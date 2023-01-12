# ERC4337 -- Networking

**Notice**: This document is work-in-progress for researchers and implementers.

This document contains the networking specification of the bundler software for ERC4337. 

It consists of four main sections:

1. A specification of the network fundamentals.
2. A specification of the three network interaction *domains* of the bundler: (a) the gossip domain, (b) the discovery domain, and (c) the Req/Resp domain.
3. The rationale and further explanation for the design choices made in the previous two sections.
4. An analysis of the maturity/state of the libp2p features required by this spec across the languages in which bundler software are being developed.

## Table of contents
<!-- TOC -->
<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->

- [Network fundamentals](#network-fundamentals)
  - [Transport](#transport)
  - [Encryption and identification](#encryption-and-identification)
  - [Protocol Negotiation](#protocol-negotiation)
- [Bundler interaction domains](#bundler-interaction-domains)
  - [Configuration](#configuration)
  - [MetaData](#metadata)
  - [The gossip domain: gossipsub](#the-gossip-domain-gossipsub)
    - [Topics and messages](#topics-and-messages)
      - [Global topics](#global-topics)
        - [`topic_1`](#topic_1)
        - [`topic_2`](#topic_2)
    - [Encodings](#encodings)
  - [Container Specifications](#container-specs)
  - [The Req/Resp domain](#the-reqresp-domain)
    - [Protocol identification](#protocol-identification)
    - [Req/Resp interaction](#reqresp-interaction)
      - [Requesting side](#requesting-side)
      - [Responding side](#responding-side)
    - [Encoding strategies](#encoding-strategies)
      - [SSZ-snappy encoding strategy](#ssz-snappy-encoding-strategy)
    - [Messages](#messages)
      - [Status](#status)
      - [Goodbye](#goodbye)
      - [Ping](#ping)
      - [GetMetaData](#getmetadata)
- [Design decision rationale](#design-decision-rationale)
- [libp2p implementations matrix](#libp2p-implementations-matrix)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->
<!-- /TOC -->

# Network fundamentals

This section outlines the specification for the networking stack in ERC4337 bundlers.

## Transport

All implementations MUST support the TCP libp2p transport, and it MUST be enabled for both dialing and listening (i.e. outbound and inbound connections). The libp2p TCP transport supports listening on IPv4 and IPv6 addresses (and on multiple simultaneously).

Bundlers must support listening on at least one of IPv4 or IPv6.
Bundlers that do _not_ have support for listening on IPv4 SHOULD be cognizant of the potential disadvantages in terms of
Internet-wide routability/support. Bundlers MAY choose to listen only on IPv6, but MUST be capable of dialing both IPv4 and IPv6 addresses.

All listening endpoints must be publicly dial-able, and thus not rely on libp2p circuit relay, AutoNAT, or AutoRelay facilities.
(Usage of circuit relay, AutoNAT, or AutoRelay will be specifically re-examined soon.)

Nodes operating behind a NAT, or otherwise un dial-able by default (e.g. container runtime, firewall, etc.),
MUST have their infrastructure configured to enable inbound traffic on the announced public listening endpoint.

## Encryption and identification

The [Libp2p-noise](https://github.com/libp2p/specs/tree/master/noise) secure channel handshake with `secp256k1` identities will be used for encryption.

As specified in the libp2p specification, Bundlers MUST support the `XX` handshake pattern.

## Protocol Negotiation

Bundlers MUST use exact equality when negotiating protocol versions to use and MAY use the version to give priority to higher version numbers.

Bundlers MUST support [multistream-select 1.0](https://github.com/multiformats/multistream-select/)
and MAY support [multiselect 2.0](https://github.com/libp2p/specs/pull/95) when the spec solidifies.
Once all Bundlers have implementations for multiselect 2.0, multistream-select 1.0 MAY be phased out.

# Bundler interaction domains

## Configuration

This section outlines constants that are used in this spec.

| Name | Value | Description |
|---|---|---|


## MetaData

Bundlers MUST locally store the following `MetaData`:

```
(
  
)
```

Where

- .
- .


## The gossip domain: gossipsub

Bundlers MUST support the [gossipsub v1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md) libp2p Protocol
including the [gossipsub v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md) extension.

### Topics and messages

Topics are plain UTF-8 strings and are encoded on the wire as determined by protobuf (gossipsub messages are enveloped in protobuf messages).
Topic strings have form: `/erc4337/UserOpsWithEntryPoint/Name/Encoding`.
This defines both the type of data being sent on the topic and how the data field of the message is encoded.

- `UserOpsWithEntryPoint` - the lowercase hex-encoded (no "0x" prefix) bytes of `entry_point_contract_address` + `user_operation` 

Each gossipsub [message](https://github.com/libp2p/go-libp2p-pubsub/blob/master/pb/rpc.proto#L17-L24) has a maximum size of `GOSSIP_MAX_SIZE`.
Bundlers MUST reject (fail validation) messages that are over this size limit.
Likewise, Bundlers MUST NOT emit or propagate messages larger than this limit.

The `message-id` of a gossipsub message MUST be the following 20 byte value computed from the message data:
* If `message.data` has a valid snappy decompression, set `message-id` to the first 20 bytes of the `SHA256` hash of
  the concatenation of `MESSAGE_DOMAIN_VALID_SNAPPY` with the snappy decompressed message data,
  i.e. `SHA256(MESSAGE_DOMAIN_VALID_SNAPPY + snappy_decompress(message.data))[:20]`.
* Otherwise, set `message-id` to the first 20 bytes of the `SHA256` hash of
  the concatenation of `MESSAGE_DOMAIN_INVALID_SNAPPY` with the raw message data,
  i.e. `SHA256(MESSAGE_DOMAIN_INVALID_SNAPPY + message.data)[:20]`.

*Note*: The above logic handles two exceptional cases:
(1) multiple snappy `data` can decompress to the same value,
and (2) some message `data` can fail to snappy decompress altogether.

The payload is carried in the `data` field of a gossipsub message, and varies depending on the topic:

| Name                             | Message Type              |
|----------------------------------|---------------------------|
| `UserOperationWithEntryPoint`    | ``                        |
| `NewPooledUserOperationsHashes`  | ``                        |


Bundlers MUST reject (fail validation) messages containing an incorrect type, or invalid payload.

When processing incoming gossip, Bundlers MAY de-score or disconnect peers who fail to observe these constraints.

For any optional queueing, Bundlers SHOULD maintain maximum queue sizes to avoid DoS vectors.


#### Global topics

There are two primary global topics used to propagate user operations (`UserOperationWithEntryPoint`)
and aggregate user operation hashes (`NewPooledUserOperationsHashes`) to all nodes on the network.

##### `UserOperationWithEntryPoint`

The `UserOperationWithEntryPoint` topic is the concatenation of EntryPoint address and UserOperation message serialized using SSZ

##### `NewPooledUserOperationsHashes`

The `NewPooledUserOperationsHashes` topic is used solely for propagating to all the connected nodes on the networks. One or more UserOps that have appeared in the network and which have not yet been included in a block are propagated to a fraction of the nodes connected to the network.


##### `GetPooledUserOps`

The `GetPooledUserOps` requests UserOps from the recipients UserOp mempool for a given EntryPoint contract address. The recommended soft limit for GetPooledUserOps requests is 256 hashes (8 KiB). The recipient may enforce an arbitrary limit on the response (size or serving time), which must not be considered a protocol violation.

##### `PooledUserOperations`

The `PooledUserOperations` is a response to the `GetPooledUserOps`, returning the requested UserOperations from the local pool. The items in the list are UserOps in the format described in ERC4337 specification. 

## Constants

| Name                             | Message Type              |
|----------------------------------|---------------------------|
| `MAX_OPS_PER_REQUEST`            | `uint64(2**10)` = 1024    |


## Container Specifications

The following types are SimpleSerialize (SSZ) containers.

#### `UserOp`

```python
class UserOp(Container):
    sender: Address
    nonce: uint256
    init_code: bytes
    call_data: bytes
    call_gas_limit: uint256
    verification_gas_limit: uint256
    pre_verification_gas: uint256
    max_fee_per_gas: uint256
    max_priority_fee_per_gas: uint256
    paymaster_and_data: bytes
    signature: bytes
```

#### `UserOperationWithEntryPoint`

```python
class UserOperationWithEntryPoint(Container):
    entry_point_contract: Address
    verified_at_block_hash: uint256
    chain_id: uint256
    user_operations: List[UserOp, MAX_OPS_PER_REQUEST]
```

#### `GetPooledUserOps`

```python
class GetPooledUserOps(Container):
    request_id: uint256
    chain_id: uint256
    entry_point_address: List[Address, MAX_OPS_PER_REQUEST]
```

#### `PooledUserOperations`

```python
class PooledUserOperations(Container):
    chain_id: uint256
    entry_point_address: Address
    user_operations: List[UserOp, MAX_OPS_PER_REQUEST]
```