# Butt2 feed format

Butt2 is a new binary feed format for [SSB]. The format is designed to
simple to implement in languages other than JavaScript, be easy to
integrate into existing implementations, be backwards compatible with
existing applications and lastly to be performant.

With the introduction of [meta feeds] it becomes possible to have
multiple feed formats. This allows us to have an upgrade path from
existing single feed, to potentially multiple feeds in a new format.

Butt2 uses [bipf] for encoding as that is a good foundation for an
append-only-log system (write once, many reads). It uses [ssb-bfe] for
binary encodings of things like feed and messages. Borrows a lot of
ideas from [bamboo] and supports bulk signatures for faster
validation.

## Format

The encoding of a message can be seen as a multiple layers. On the
first layer is an array of value+signature and
bipf-encoded-content. On the second layer (value+signature) is an
array of the encoded value and a dictionary of signatures. The final
layer is the encoded value.

### Encoded value

An bipf encoded value is a list of:

 - [ssb-bfe] encoded ed25519 author
 - sequence number of the message in the feed
 - the timestamp of the message
 - [ssb-bfe] encoded backlink
 - a byte with extensible tag information (`0xOO` means content is
   bipf encoded SSB, `0xO1` means end of feed).
 - the length of the content in bytes
 - [ssb-bfe] encoded [blake3] hash of content

### Signatures

Signatures is a dictionary where the key specifies the starting
sequence for the signature. It MUST contain at least 1 key, the
sequence of the current message and sign the encoded value. It can
contain a signature for N previous messages by signing the
concatenated message keys of these N messages. This allows for
substantial reduction in validation time.

## Performance

A benchmark of an initial prototype shows the time is takes to
validate and to convert to database format to be reduced in half for
single message validation. While bulk validation with signatures for
25 messages to take 1/7 the time of existing format. To put these
numbers into perspective, on an Intel i7-10510U it takes 3 minutes and
20 seconds to validate and convert 1 million messages, while butt2
takes 28 seconds.

## Design choices

FIXME: explain why this doesn't have lipmaa links


[SSB]: https://ssbc.github.io/scuttlebutt-protocol-guide/
[meta feeds]: https://github.com/ssb-ngi-pointer/ssb-meta-feeds-spec
[bipf]: https://github.com/ssbc/bipf
[bamboo]: https://github.com/AljoschaMeyer/bamboo/
[ssb-bfe]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec
[blake3]: https://github.com/BLAKE3-team/BLAKE3
