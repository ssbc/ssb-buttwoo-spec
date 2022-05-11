# Butt2 feed format

Butt2 is a new binary feed format for [SSB]. The format is designed to
simple to implement in languages other than JavaScript, be easy to
integrate into existing implementations, be backwards compatible with
existing applications and lastly to be performant.

With the introduction of [meta feeds] it becomes possible to have
multiple feed formats. This allows us to have an upgrade path from
existing single feed, to potentially multiple feeds in new formats.

Butt2 uses [bipf] for encoding as that is a good foundation for an
append-only-log system (write once, many reads). It uses [ssb-bfe] for
binary encodings of things like feed and messages. Borrows a lot of
ideas from [bamboo] and supports bulk signatures for faster
validation.

## Format

The encoding of a message can be seen as a multiple layers. The first
layer is the value layer. The encoded value is used for
signatures. The second layer consists of an array of encoded value and
the signatures dictionary. The encoded of this is the base for the
message key. Finally the transport layer is an encoded array of the
second layer together with the bipf encoded content. Note this
transport layer is only for backwards compatibility with existing
replication such as [EBT] and thus not strictly part of the feed
format.

Visually the format can be viewed as:

```
  Transport:       [butt2, contentBipf]
  butt2:           [value, signatures]
  value:           [author, sequence, timestamp, backlink, tag, contentLen, contentHash]
  signatures:      { sequence: signature, ... }
```

### Value

A bipf encoded value is a list of:

 - [ssb-bfe] encoded ed25519 author
 - sequence number of the message in the feed
 - the timestamp of the message
 - [ssb-bfe] encoded backlink
 - a byte with extensible tag information (`0xOO` means content is
   bipf encoded SSB, `0xO1` means end of feed).
 - the length of the content in bytes
 - [ssb-bfe] encoded [blake3] hash of content

### Content

If not encrypted, content should be bipf encoded. If encrypted, a
[ssb-bfe encrypted data format] MUST be used.

### Signatures

Signatures is a dictionary where the key specifies the starting
sequence for the signature. It MUST contain at least 1 key, the
sequence of the current message and sign the encoded value. It can
contain one or more signatures for the N previous messages by signing
the concatenated message keys of these N messages. This allows for
substantial reduction in validation time. [Why it is ok to sign hashes
instead of full messages]. Signatures MUST be sorted by sequence in
ascending order. This is to ensure canonical encoding.

## Performance

A benchmark of a prototype shows the time it takes to validate and
convert for storing in a database to be reduced in half for single
message validation. While bulk validation with a signature for every
25 messages takes 1/7 the time of existing format. To put these
numbers into perspective, on an Intel i7-10510U it takes 3 minutes and
20 seconds to validate and convert 1 million messages, while butt2
takes 28 seconds.

## Size

There is roughly a 20% size reduction in network traffic compared to
classic format.

## Validation

A butt2 message MUST conform to the following rules:
 - be an bipf encoded array of two elements: value, signatures
 - Value must be an bipf encoded array of 7 elements:
   - a [ssb-bfe] encoded author
   - a sequence that starts with 1 and increases by 1 for each message
   - a timestamp representing the UNIX epoch timestamp of message
     creation
   - a [ssb-bfe] encoded backlink of the previous messages key
   - a byte representating a tag of either: `0xOO` or `0xO1`
   - the content length in bytes. This number must not exceed 16384.
   - a [ssb-bfe] encoded [blake3] hash of the content bytes
 - Signatures must be an bipf encoded dictionary mapping sequences to
   [ssb-bfe] encoded signatures
   - must be sorted by sequence in ascending order
   - if sequence is the same as for the message, then signature is the
     bytes of the bipf encoded `value` signed using the authors key
   - any other value must have a sequence lower than the message. For
     these the signature signs the concatenated bytes of the message
     keys for the messages starting from `sequence` until but not
     including the `sequence` of this message.

Content, if available MUST conform to the following rules: 
 - it must be valid bipf
 - The byte length must match the content size in value

## Design choices

### Keeping timestamps

One difference between butt2 and bamboo is that bamboo does not have
timestamps in the format, instead leaving those to be part of the
content. This is important for private messages. This choice was
mostly formed from an backwards compatible perspective. It should be
noted that with meta feeds it becomes possible to store the messages
of a private group in a feed that is only exchanged with members of
the group, thus leaving the potential metadata leak problem void.

### Lipmaa links

Another difference between butt2 and bamboo is that lipmaa links are
not included. Lipmaa links allows partial replication in cases where
the specific subset of messages are not important, only that they form
a valid chain back to the root message. This comes at the cost that
validation is now more expensive because for roughly every second
message an additional link needs to be checked.

Furthermore with meta feeds we can now partition the data of a feed
into subsets (such as the friends graph and about messages in separate
feeds). This leaves public messages where lipmaa links based partial
replication could be useful. Also note for this to really work, the
friend graph needs to include the root hash besides the feed,
otherwise an adversary (given the private key) could still create a
fake feed. 

Lastly we already have another mechanism for doing partial
replication, namely tangles where messages from multiple feeds are
linked together and form their own chain.

To sum up, the advantages does not outweight the disadvantages for
SSB.

### Sign hash of the content

Similar to bamboo the signature is over the hash of the content and
not the actual content. This allows validation of a log without the
actual content. It should be noted that deletion is a whole topic in
itself, this is just to note that this format also supports this case.

### Bipf encoding

While many encodings could be used for encoding of especially the
value part, bipf is a relatively simple format. The JavaScript
implementation is roughly 250 lines for encode and decode. Bipf allows
the content to be reused when encoding for the database in [ssb-db2]
resultating in roughly half the time used compared to existing feed
format.

[SSB]: https://ssbc.github.io/scuttlebutt-protocol-guide/
[meta feeds]: https://github.com/ssb-ngi-pointer/ssb-meta-feeds-spec
[bipf]: https://github.com/ssbc/bipf
[bamboo]: https://github.com/AljoschaMeyer/bamboo/
[ssb-bfe]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec
[ssb-bfe encrypted data format]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec#5-encrypted-data-formats
[blake3]: https://github.com/BLAKE3-team/BLAKE3
[EBT]: https://github.com/ssbc/ssb-ebt
[ssb-db2]: https://github.com/ssb-ngi-pointer/ssb-db2
[Why it is ok to sign hashes instead of full messages]: https://crypto.stackexchange.com/questions/6335/is-signing-a-hash-instead-of-the-full-data-considered-secure
