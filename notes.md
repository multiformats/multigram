latest thoughts
- [varint protocol][varint length][bytes payload]
  - payload is length prefixed, so one datagram can carry multiple multigrams
  - no nested multigrams though
- use varints
  - how to read a varint from a packet byte[]
    - encoding/binary.ReadUvarint()
    - io.ByteReader
    - bytes.Reader.Size() minus Len()
  - $protocol+$length should always be padded to 4 or 8 bytes => cpu word length
- protocol 0x00 is multigram-select
  - every multigram-select packet sent includes the last known remote state
  - send errorish packet if received packet is of unknown protocol
  - send errorish packet not more than once per $errorInterval
  - maybe able to append an error multigram to a regular multigram
  - remote can ask to have a higher $errorInterval
  - remote can ask to reset local table
  - remote can ask to redefine 0x00 as a different version of multigram-select
    - a multigram-select channel can upgrade itself!

---

- discussion with whyrusleeping
  - setup packets and data packets are always separate (proto =0 vs. proto >0)
  - just start sending data packets
    - the other end will respond with error packets
    - the other end MAY buffer packets with a proto it doesn't know yet
    - until you get an ack, send a setup packet for every packet of the given protocol
  - dont do table exchange, setup protocols as needed?

---

- previous discussion: https://github.com/ipfs/specs/pull/123
- discussion with jbenet
  - include checksum
    - so we can work on raw ip
    - udp checksums suck
    - would be great to fit in 3/7/11/15 bytes, to fit 4-byte word length
    - out of scope, should be another format: multisum
      - we'll have multiple mutligrams per packet, one checksum for each is a MUST NOT
      - the packet's integrity is protected by crypto anyhow most of the time
- todo
- decide int8 vs. varint
  - nested multigrams experiment
  - int8 vs. varint benchmarks
- decide on /multigram-setup multicodec formats
- research prior art on table replication
- success responses
- error responses
- ability to reset the table?


- discussion with kubuxu
  - monotonic time: golang/go#12914
