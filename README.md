# multigram

[![](https://img.shields.io/badge/made%20by-Protocol%20Labs-blue.svg?style=flat-square)](http://ipn.io)
[![](https://img.shields.io/badge/project-multiformats-blue.svg?style=flat-square)](http://github.com/multiformats/multiformats)
[![](https://img.shields.io/badge/freenode-%23ipfs-blue.svg?style=flat-square)](http://webchat.freenode.net/?channels=%23ipfs)

> Protocol negotiation and multiplexing over datagrams

This document describes:

- Multigram, a self-describing packet format for multiplexing different protocols on the same datagram connection.
- Multigram-Setup, a protocol for negotiating a shared table of protocol identifiers for use with Multigram.

Multigram is part of the [Multiformats family][multiformats].
So far Multigram is being implemented in Golang ([go-multigram][go-multigram])
and JavaScript ([js-multigram][js-multigram]). More implementations are welcome.

- [Introduction](#introduction)
- [Packet Layout](#packet-layout)
- [Identifier Table](#identifier-table)
- [Multigram-Setup](#multigram-setup)
- [Maintainers](#maintainers)
- [Contribute](#contribute)
- [License](#license)

Note: this document makes use of the [Multiaddr format][multiaddr] whenever it mentions network addresses,
and the [Multicodec format][multicodec] when encoding values.

[multiformats]: https://github.com/multiformats
[go-multigram]: https://github.com/multiformats/go-multigram
[js-multigram]: https://github.com/multiformats/multigram/issues/2
[multiaddr]: https://github.com/multiformats/multiaddr
[multicodec]: https://github.com/multiformats/multicodec


## Introduction

Multigram operates on datagrams, which can be UDP packets, Ethernet frames, etc. and which are unreliable and unordered.
All it does is prepend a field to the packet, which signifies the protocol of this packet.
The endpoints of the connection can then use different packet handlers per protocol.

If you're looking for similar functionality on top of reliable streams, check out the [Multistream format][multistream].

[multistream]: https://github.com/multistream


## Packet layout

For multiplexing different protocols on the same datagram connection, multigram prepends a 1-byte header to every packet. This header represents an index in a table of protocols shared between both endpoints. This protocol table is negotiated by exchanging the intersection of the endpoint's supported protocols. The protocol table's size of 256 tuples can be increased by nesting multiple multigram headers.

```
                  1               2               3               4
   0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
0 |  Table Index  |                                               |
  +-+-+-+-+-+-+-+-+                Packet Payload                 +
4 |                                                               |
  +
```

- **Table Index:**
  - Type: `varint`
  - The numerical identifier signifying the protocol of this packet, as per the protocol table.
- **Packet Payload:**
  - Type: `[]byte`
  - The raw data of this packet. This is what the packet handler gets to see.


## Identifier table

Multigram assumes an independent protocol table for each remote address.
For example, datagrams from/to `/ip4/1.2.3.4/udp/4737` will build up their own protocol table
independent from datagrams from/to `/ip4/5.6.7.8/udp/4737`.

The protocol table MUST be append-only and immutable. It MUST initially contain exactly one tuple:

```
0x00,/multigram-setup/0.1.0
```

The `/multigram-setup` protocol is used for appending to the shared protocol table.

```
                  1               2               3               4
   0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7 0 1 2 3 4 5 6 7
  +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
0 |     0x00      |                                               |
  +-+-+-+-+-+-+-+-+           Operation as Multicodec             +
4 |                                                               |
  +
```

- Note how the Table Index field is set to `0x00`, selecting the `/multigram-setup` protocol.
- The Operation field MUST support at least the `/cbor/` multicodec format. It SHOULD support the `/protobuf/` and `/json/` formats.
- In the future, an `/ipfs/` format can be used to resolve code and specification for supporting the format.

## Multigram-Setup

Either endpoint can send `append` proposals, and the other endpoint will reply with the result, based on their own supported protocols.

(1) Endpoint A sends a proposal to Endpoint B. A doesn't commit this proposal to its own view of the protocol table. It dismisses it right away and waits for B's reply.
```
0x00
/json/
{"0x01":"/foo/1.0.0","0x02":"/bar/1.0.0","0x03":"/baz/1.0.0"}
```

(2) Endpoint B forms the intersection of this proposal and its own supported protocols, appends to its own view of the protocol table, and replies.
```
0x00
/json/
{"0x01":"/foo/1.0.0","0x02":"/bar/1.0.0"}
```

(3) We can list the table by sending the other endpoint an empty proposal.
```
0x00
/json/
{}
```

(4) The protocol table now contains `0x00`, `0x01`, and `0x02`.
```
0x00
/json/
{"0x01":"/foo/1.0.0","0x02":"/bar/1.0.0"}
```

Any field in the operation data starting in `0x` is considered for being appended.
Any other field names can be used e.g. for checksums, detecting packet loss, or communicating errors.
This is subject to updates of the multigram-setup protocol.

### Nested multigrams

The protocol tables of nested multigrams can be set up within one packet.
This works because any trailing data will be processed after the table operation.

```
0x00
/json/
{"0x01":"/multigram/0.1.0"}
0x0100
/json/
{"0x01":"/multigram/0.1.0"}
0x010100
/json
{"0x01":"/ipfs/identify/1.0.0","0x02":"/fc00/iptunnel/0.1.0","0x03":"/fc00/pathfinder/0.1.0"}
```

Packets starting in `0x010103` would belong to `/fc00/pathfinder`.


## Maintainers

Captain: [@lgierth](https://github.com/lgierth).


## Contribute

Contributions welcome. Please check out [the issues](https://github.com/multiformats/multigram/issues).

Check out our [contributing document](https://github.com/multiformats/multiformats/blob/master/contributing.md) for more information on how we work, and about contributing in general. Please be aware that all interactions related to multiformats are subject to the IPFS [Code of Conduct](https://github.com/ipfs/community/blob/master/code-of-conduct.md).


## License

[MIT](LICENSE) © Protocol Labs Inc.
