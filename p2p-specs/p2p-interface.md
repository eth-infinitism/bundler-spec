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
        - [`user_ops_with_entry_point`](#user_ops_with_entry_point)
        - [`pooled_user_ops_hashes`](#pooled_user_ops_hashes)
    - [Encodings](#encodings)
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
- [The discovery domain: discv5](#the-discovery-domain-discv5)
    - [Integration into libp2p stacks](#integration-into-libp2p-stacks)
    - [ENR structure](#enr-structure)
      - [Mempool subnet bitfield](#mempool-subnet-bitfield)
      - [`account_abstraction` field](#account_abstraction-field)
- [Container Specifications](#container-specs)
- [Design decision rationale](#design-decision-rationale)

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

## Multiplexing
Like Ethereum consensus clients p2p specification, the Bundlers ought to support both multiplexer implementations of mplex and yamux. Their protocol IDs are, respectively: /mplex/6.7.0 and /yamux/1.0.0.

Clients MUST support mplex and MAY support yamux. If both are supported by the client, yamux MUST take precedence during negotiation. See the Rationale section below for tradeoffs.

# Bundler network interaction domains

## Configuration

This section outlines constants that are used in this spec.

| Name | Value | Description |
|---|---|---|
| `GOSSIP_MAX_SIZE`    | `2**20` (= 1048576, 1 MiB) | The maximum allowed size of uncompressed gossip messages. |
| `MAX_OPS_PER_REQUEST`| `256` | Maximum number of UserOps in a single request. |
| `RESP_TIMEOUT`	     | `10s` | The maximum time for complete response transfer. |
| `TTFB_TIMEOUT`       | `5s` | The maximum time to wait for first byte of request response (time-to-first-byte). |


## MetaData

Bundlers MUST locally store the following `MetaData`:

```
(
  seq_number: uint64
  bundler_nets: Bitvector[BUNDLER_SUBNET_COUNT]
)
```

Where

`seq_number` is a `uint64` starting at 0 used to version the node's metadata. If any other field in the local `MetaData` changes, the node MUST increment `seq_number` by 1.
`bundler_nets` is a Bitvector representing the node's persistent mempool subnet subscriptions.


## The gossip domain: gossipsub

Bundlers MUST support the [gossipsub v1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.0.md) libp2p Protocol
including the [gossipsub v1.1](https://github.com/libp2p/specs/blob/master/pubsub/gossipsub/gossipsub-v1.1.md) extension.

### Topics and messages

Topics are plain UTF-8 strings and are encoded on the wire as determined by protobuf (gossipsub messages are enveloped in protobuf messages).
Topic strings have form: `/erc4337/UserOpsWithEntryPoint/Name/Encoding`.
This defines both the type of data being sent on the topic and how the data field of the message is encoded.

- `UserOpsWithEntryPoint` - the lowercase hex-encoded (no "0x" prefix) bytes of `mempool_id` + `user_operation` 

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

| Name                           | Message Type                    |
|--------------------------------|---------------------------------|
| `user_ops_with_entry_point`    | `UserOperationsWithEntryPoint`  |
| `pooled_user_ops_hashes`       | `PooledUserOperationsHashes`    |

Bundlers MUST reject (fail validation) messages containing an incorrect type, or invalid payload.

When processing incoming gossip, Bundlers MAY de-score or disconnect peers who fail to observe these constraints.

For any optional queueing, Bundlers SHOULD maintain maximum queue sizes to avoid DoS vectors.


#### Global topics

There are two primary global topics used to propagate user operations (`UserOperationWithEntryPoint`)
and aggregate user operation hashes (`NewPooledUserOperationsHashes`) to all nodes on the network.

##### `user_ops_with_entry_point`

The `user_ops_with_entry_point` topic is the concatenation of EntryPoint address and UserOperation message serialized using SSZ

The following validations MUST pass before forwarding the `user_ops_with_entry_point` on the network
- TBD
- TBD
- TBD

##### `pooled_user_ops_hashes`

The `pooled_user_ops_hashes` topic is used solely for propagating to all the connected nodes on the networks. One or more UserOps that have appeared in the network and which have not yet been included in a block are propagated to a fraction of the nodes connected to the network.

The following validations MUST pass before forwarding the `pooled_user_ops_hashes` on the network
- TBD
- TBD
- TBD

## The Req/Resp domain

### Protocol identification

Each message type is segregated into its own libp2p protocol ID, which is a case-sensitive UTF-8 string of the form:

```
/ProtocolPrefix/MessageName/SchemaVersion/Encoding
```

With:

- `ProtocolPrefix` - messages are grouped into families identified by a shared libp2p protocol name prefix.
  In this case, we use `/account_abstraction/erc4337/req`.
- `MessageName` - each request is identified by a name consisting of English alphabet, digits and underscores (`_`).
- `SchemaVersion` - an ordinal version number (e.g. 1, 2, 3…).
  Each schema is versioned to facilitate backward and forward-compatibility when possible.
- `Encoding` - while the schema defines the data types in more abstract terms,
  the encoding strategy describes a specific representation of bytes that will be transmitted over the wire.
  See the [Encodings](#Encoding-strategies) section for further details.

This protocol segregation allows libp2p `multistream-select 1.0` / `multiselect 2.0`
to handle the request type, version, and encoding negotiation before establishing the underlying streams.

### Req/Resp interaction

We use ONE stream PER request/response interaction.
Streams are closed when the interaction finishes, whether in success or in error.

Request/response messages MUST adhere to the encoding specified in the protocol name and follow this structure (relaxed BNF grammar):

```
request   ::= <encoding-dependent-header> | <encoded-payload>
response  ::= <response_chunk>*
response_chunk  ::= <result> | <encoding-dependent-header> | <encoded-payload>
result    ::= “0” | “1” | “2” | [“128” ... ”255”]
```

The encoding-dependent header may carry metadata or assertions such as the encoded payload length, for integrity and attack proofing purposes.
Because req/resp streams are single-use and stream closures implicitly delimit the boundaries, it is not strictly necessary to length-prefix payloads;
however, certain encodings like SSZ do, for added security.

A `response` is formed by zero or more `response_chunk`s.
Responses that consist of a single SSZ-list (such as `GetPooledUserOps` and `PooledUserOperations`) send each list item as a `response_chunk`.
All other response types (non-Lists) send a single `response_chunk`.

For both `request`s and `response`s, the `encoding-dependent-header` MUST be valid,
and the `encoded-payload` must be valid within the constraints of the `encoding-dependent-header`.
This includes type-specific bounds on payload size for some encoding strategies.
Regardless of these type specific bounds, a global maximum uncompressed byte size of `MAX_CHUNK_SIZE` MUST be applied to all method response chunks.

Clients MUST ensure that lengths are within these bounds; if not, they SHOULD reset the stream immediately.
Clients tracking peer reputation MAY decrement the score of the misbehaving peer under this circumstance.

#### Requesting side

Once a new stream with the protocol ID for the request type has been negotiated, the full request message SHOULD be sent immediately.
The request MUST be encoded according to the encoding strategy.

The requester MUST close the write side of the stream once it finishes writing the request message.
At this point, the stream will be half-closed.

The requester MUST wait a maximum of `TTFB_TIMEOUT` for the first response byte to arrive (time to first byte—or TTFB—timeout).
On that happening, the requester allows a further `RESP_TIMEOUT` for each subsequent `response_chunk` received.

If any of these timeouts fire, the requester SHOULD reset the stream and deem the req/resp operation to have failed.

A requester SHOULD read from the stream until either:
1. An error result is received in one of the chunks (the error payload MAY be read before stopping).
2. The responder closes the stream.
3. Any part of the `response_chunk` fails validation.
4. The maximum number of requested chunks are read.

For requests consisting of a single valid `response_chunk`,
the requester SHOULD read the chunk fully, as defined by the `encoding-dependent-header`, before closing the stream.

#### Responding side

Once a new stream with the protocol ID for the request type has been negotiated,
the responder SHOULD process the incoming request and MUST validate it before processing it.
Request processing and validation MUST be done according to the encoding strategy, until EOF (denoting stream half-closure by the requester).

The responder MUST:

1. Use the encoding strategy to read the optional header.
2. If there are any length assertions for length `N`, it should read exactly `N` bytes from the stream, at which point an EOF should arise (no more bytes).
  Should this not be the case, it should be treated as a failure.
3. Deserialize the expected type, and process the request.
4. Write the response which may consist of zero or more `response_chunk`s (result, optional header, payload).
5. Close their write side of the stream. At this point, the stream will be fully closed.

If steps (1), (2), or (3) fail due to invalid, malformed, or inconsistent data, the responder MUST respond in error.
Clients tracking peer reputation MAY record such failures, as well as unexpected events, e.g. early stream resets.

The entire request should be read in no more than `RESP_TIMEOUT`.
Upon a timeout, the responder SHOULD reset the stream.

The responder SHOULD send a `response_chunk` promptly.
Chunks start with a **single-byte** response code which determines the contents of the `response_chunk` (`result` particle in the BNF grammar above).
For multiple chunks, only the last chunk is allowed to have a non-zero error code (i.e. The chunk stream is terminated once an error occurs).

The response code can have one of the following values, encoded as a single unsigned byte:

-  0: **Success** -- a normal response follows, with contents matching the expected message schema and encoding specified in the request.
-  1: **InvalidRequest** -- the contents of the request are semantically invalid, or the payload is malformed, or could not be understood.
  The response payload adheres to the `ErrorMessage` schema (described below).
-  2: **ServerError** -- the responder encountered an error while processing the request.
  The response payload adheres to the `ErrorMessage` schema (described below).
-  3: **ResourceUnavailable** -- the responder does not have requested resource.
  The response payload adheres to the `ErrorMessage` schema (described below).
  *Note*: This response code is only valid as a response where specified.

Clients MAY use response codes above `128` to indicate alternative, erroneous request-specific responses.

The range `[4, 127]` is RESERVED for future usages, and should be treated as error if not recognized expressly.

The `ErrorMessage` schema is:

```
(
  error_message: List[byte, 256]
)
```

*Note*: By convention, the `error_message` is a sequence of bytes that MAY be interpreted as a UTF-8 string (for debugging purposes).
Clients MUST treat as valid any byte sequences.

### Encoding strategies

The token of the negotiated protocol ID specifies the type of encoding to be used for the req/resp interaction.
Only one value is possible at this time:

-  `ssz_snappy`: The contents are first [SSZ-encoded](../../ssz/simple-serialize.md)
  and then compressed with [Snappy](https://github.com/google/snappy) frames compression.
  For objects containing a single field, only the field is SSZ-encoded not a container with a single field.
  This encoding type MUST be supported by all clients.

#### SSZ-snappy encoding strategy

The [SimpleSerialize (SSZ) specification](../../ssz/simple-serialize.md) outlines how objects are SSZ-encoded.

To achieve snappy encoding on top of SSZ, we feed the serialized form of the object to the Snappy compressor on encoding.
The inverse happens on decoding.

Snappy has two formats: "block" and "frames" (streaming).
To support large requests and response chunks, snappy-framing is used.

Since snappy frame contents [have a maximum size of `65536` bytes](https://github.com/google/snappy/blob/master/framing_format.txt#L104)
and frame headers are just `identifier (1) + checksum (4)` bytes, the expected buffering of a single frame is acceptable.

**Encoding-dependent header:** Req/Resp protocols using the `ssz_snappy` encoding strategy MUST encode the length of the raw SSZ bytes,
encoded as an unsigned [protobuf varint](https://developers.google.com/protocol-buffers/docs/encoding#varints).

*Writing*: By first computing and writing the SSZ byte length, the SSZ encoder can then directly write the chunk contents to the stream.
When Snappy is applied, it can be passed through a buffered Snappy writer to compress frame by frame.

*Reading*: After reading the expected SSZ byte length, the SSZ decoder can directly read the contents from the stream.
When snappy is applied, it can be passed through a buffered Snappy reader to decompress frame by frame.

Before reading the payload, the header MUST be validated:
- The unsigned protobuf varint used for the length-prefix MUST not be longer than 10 bytes, which is sufficient for any `uint64`.
- The length-prefix is within the expected [size bounds derived from the payload SSZ type](#what-are-ssz-type-size-bounds).

After reading a valid header, the payload MAY be read, while maintaining the size constraints from the header.

A reader SHOULD NOT read more than `max_encoded_len(n)` bytes after reading the SSZ length-prefix `n` from the header.
- For `ssz_snappy` this is: `32 + n + n // 6`.
  This is considered the [worst-case compression result](https://github.com/google/snappy/blob/537f4ad6240e586970fe554614542e9717df7902/snappy.cc#L98) by Snappy.

A reader SHOULD consider the following cases as invalid input:
- Any remaining bytes, after having read the `n` SSZ bytes. An EOF is expected if more bytes are read than required.
- An early EOF, before fully reading the declared length-prefix worth of SSZ bytes.

In case of an invalid input (header or payload), a reader MUST:
- From requests: send back an error message, response code `InvalidRequest`. The request itself is ignored.
- From responses: ignore the response, the response MUST be considered bad server behavior.

All messages that contain only a single field MUST be encoded directly as the type of that field and MUST NOT be encoded as an SSZ container.

Responses that are SSZ-lists (for example `List[UserOperationsWithEntryPoint, ...]`) send their
constituents individually as `response_chunk`s. For example, the
`List[UserOperationsWithEntryPoint, ...]` response type sends zero or more `response_chunk`s.
Each _successful_ `response_chunk` contains a single `UserOperationsWithEntryPoint` payload.

### Messages

#### Status

**Protocol ID:** `/account_abstraction/erc4337/req/status/1/`

Request, Response Content:
```
(
  supported_mempools - ?
  ???
)
```
The fields are, as seen by the client at the time of sending the message:

- supported_mempools - List of supported mempools.
- 
- 
- 
- 

The dialing client MUST send a `Status` request upon connection.

The request/response MUST be encoded as an SSZ-container.

The response MUST consist of a single `response_chunk`.

Clients SHOULD immediately disconnect from one another following the handshake above under the following conditions:

1. If the supported_mempools does not match with the client's own list of supported mempools.
2. 

#### Goodbye

**Protocol ID:** `/account_abstraction/erc4337/req/goodbye/1/`

Request, Response Content:
```
(
  uint64
)
```
Client MAY send goodbye messages upon disconnection. The reason field MAY be one of the following values:

- 1: Client shut down.
- 2: Irrelevant network.
- 3: Fault/error.

Clients MAY use reason codes above `128` to indicate alternative, erroneous request-specific responses.

The range `[4, 127]` is RESERVED for future usage.

The request/response MUST be encoded as a single SSZ-field.

The response MUST consist of a single `response_chunk`.

#### Ping

**Protocol ID:** `/account_abstraction/erc4337/req/ping/1/`

Request Content:

```
(
  uint64
)
```

Response Content:

```
(
  uint64
)
```

Sent intermittently, the `Ping` protocol checks liveness of connected peers.
Peers request and respond with their local metadata sequence number (`MetaData.seq_number`).

If the peer does not respond to the `Ping` request, the client MAY disconnect from the peer.

A client can then determine if their local record of a peer's MetaData is up to date
and MAY request an updated version via the `MetaData` RPC method if not.

The request MUST be encoded as an SSZ-field.

The response MUST consist of a single `response_chunk`.

#### GetMetaData

**Protocol ID:** `/account_abstraction/erc4337/req/metadata/1/`

No Request Content.

Response Content:

```
(
  MetaData
)
```

Requests the MetaData of a peer.
The request opens and negotiates the stream without sending any request content.
Once established the receiving peer responds with
it's local most up-to-date MetaData.

The response MUST be encoded as an SSZ-container.

The response MUST consist of a single `response_chunk`.

##### `PooledUserOpsByHashes`

**Protocol ID:** `/account_abstraction/erc4337/req/pooled_user_ops_by_hashes/1/`

Request Content:

```
(
  hashes: List[bytes32, MAX_OPS_PER_REQUEST]
)
```

Response Content: 
```
(
  List[UserOp, MAX_OPS_PER_REQUEST]
)
```

The `pooled_user_ops_by_hashes` requests UserOps from the recipients UserOp mempool for a given EntryPoint contract address. The recommended soft limit for PooledUserOpsByHashes requests is MAX_OPS_PER_REQUEST hashes. The recipient may enforce an arbitrary limit on the response (size or serving time), which must not be considered a protocol violation.
 
## The discovery domain: discv5

Discovery Version 5 ([discv5](https://github.com/ethereum/devp2p/blob/master/discv5/discv5.md)) (Protocol version v5.1) is used for peer discovery.

`discv5` is a standalone protocol, running on UDP on a dedicated port, meant for peer discovery only.
`discv5` supports self-certified, flexible peer records (ENRs) and topic-based advertisement, both of which are (or will be) requirements in this context.

### Integration into libp2p stacks

`discv5` SHOULD be integrated into the bundler’s libp2p stack by implementing an adaptor
to make it conform to the [service discovery](https://github.com/libp2p/go-libp2p-core/blob/master/discovery/discovery.go)
and [peer routing](https://github.com/libp2p/go-libp2p-core/blob/master/routing/routing.go#L36-L44) abstractions and interfaces (go-libp2p links provided).

Inputs to operations include peer IDs (when locating a specific peer) or capabilities (when searching for peers with a specific capability),
and the outputs will be multiaddrs converted from the ENR records returned by the discv5 backend.

This integration enables the libp2p stack to subsequently form connections and streams with discovered peers.

### ENR structure

The Ethereum Node Record (ENR) for a bundler client MUST contain the following entries
(exclusive of the sequence number and signature, which MUST be present in an ENR):

-  The compressed secp256k1 publickey, 33 bytes (`secp256k1` field).

The ENR MAY contain the following entries:

-  An IPv4 address (`ip` field) and/or IPv6 address (`ip6` field).
-  A TCP port (`tcp` field) representing the local libp2p listening port.
-  A UDP port (`udp` field) representing the local discv5 listening port.

Specifications of these parameters can be found in the [ENR Specification](http://eips.ethereum.org/EIPS/eip-778).

#### Mempools bitfield

The ENR `mempools` entry signifies the mempools subnet bitfield with the following form
to more easily discover peers participating in particular mempool id gossip subnets.

| Key          | Value                                            |
|:-------------|:-------------------------------------------------|
| `mempools`    | SSZ `Bitvector[MEMPOOL_ID_SUBNET_COUNT]`        |

If a node's `MetaData.mempools` has any non-zero bit, the ENR MUST include the `mempools` entry with the same value as `MetaData.mempools`.

If a node's `MetaData.mempools` is composed of all zeros, the ENR MAY optionally include the `mempools` entry or leave it out entirely.

#### `account_abstraction` field

ENRs MUST carry a generic `account_abstraction` key with an 16-byte value of the node's current state(??) to ensure connections are made with peers on the intended Ethereum network.

| Key                         | Value                              |
|:----------------------------|:-----------------------------------|
| `account_abstraction`       | SSZ `ENRAccountAbstraction`        |

Specifically, the value of the `account_abstraction` key MUST be the following SSZ encoded object (`ENRAccountAbstraction`)

```
(
    ???
)
```

where the fields of `ENRAccountAbstraction` are defined as

* 
* 
* 


Clients SHOULD connect to peers with ``, ``, and `` that match local values.

Clients MAY connect to peers with the same `` but a different ``/``.
Unless `ENRAccountAbstraction` is manually updated to matching prior to the earlier `` of the two clients,
these connecting clients will be unable to successfully interact starting at the earlier ``.

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
class UserOperationsWithEntryPoint(Container):
    entry_point_contract: Address
    verified_at_block_hash: uint256
    chain_id: uint256
    user_operations: List[UserOp, MAX_OPS_PER_REQUEST]
```

#### `PooledUserOps`

```python
class PooledUserOps(Container):
    mempool_id: bytes32
    user_operations: List[UserOp, MAX_OPS_PER_REQUEST]
```