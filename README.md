# Bendy butt 2 feed format

**Do not implement, currently being redesigned**

Bendy butt 2 is a new binary feed format for [SSB]. It is meant as a
successor to both the [classic] SSB feed format and [bendy butt]. In
contrast with [bendy butt] subfeeds are identified by the message key
of the parent feed, thus retaining the same key. It also borrows
heavily from [bamboo].

The format is designed to be simple to implement, be easy to integrate
into existing implementations, be backwards compatible with existing
applications and lastly to be performant.

Bendy butt 2 uses [bipf] for encoding since its a good foundation for
an append-only-log system (write once, many reads). It uses [ssb-bfe]
for binary encodings of feed and messages.

## Format

A bendy butt 2 message consists of 8 fields encoded as an array and a
signature. The message key is the hash of the encoded values
concatenated with the signature bytes.

### Value

A bipf encoded value is an array of:

 - [ssb-bfe] encoded ed25519 author
 - [ssb-bfe] encoded parent message id. For the top feed this must be
   BFE nil.
 - sequence number of the message in the feed
 - the timestamp of the message
 - [ssb-bfe] encoded previous message id. For the first message this
   must be BFE nil.
 - a byte with extensible tag information (`0x00` means standard
   message, `0x01` means sub-feed, `0x02` means end-of-feed).
 - the length of the content in bytes
 - [ssb-bfe] encoded [blake3] hash of content

## Meta feeds

Meta feeds are identified by the tag. Content can include extra
information what is contained in the sub feed such as the feed
purpose. For messages in a subfeed parent must be filled with the
message id in the parent feed.

### Content

If not encrypted, content should be bipf encoded. If encrypted, a
[ssb-bfe encrypted data format] MUST be used.

## Performance

A benchmark of a prototype shows the time it takes to validate and
convert for storing in a database to be reduced in half for single
message validation. Similar to classic it is possible for lite clients
to queue up messages for validation and only check the signature of a
random or the latest message. This can improve the bulk validation
substantially in onboarding situations.

## Size

There is roughly a 20% size reduction in network traffic compared to
classic format.

## Validation

A butt2 message MUST conform to the following rules:
 - Value must be an bipf encoded array of 8 elements:
   - a [ssb-bfe] encoded author
   - a [ssb-bfe] encoded parent message id
   - a sequence that starts with 1 and increases by 1 for each message
   - a timestamp representing the UNIX epoch timestamp of message
     creation
   - a [ssb-bfe] encoded previous messages key
   - a byte representating a tag of either: `0xOO` or `0xO1`
   - the content length in bytes. This number must not exceed 16384.
   - a [ssb-bfe] encoded [blake3] hash of the content bytes
 - Signature must be a [ssb-bfe] encoded signature and sign the
   encoded value array.

Content, if available MUST conform to the following rules: 
 - it must be valid bipf
 - The byte length must match the content size in value

## Integration with existing stack

### EBT

Data sent over the wire should be bipf encoded as:

```
transport:       [value, signature, contentBipf]
value:           [author, parent, sequence, timestamp, previous, tag, contentLen, contentHash]
```

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
[bendy butt]: https://github.com/ssb-ngi-pointer/bendy-butt-spec
[meta feeds]: https://github.com/ssb-ngi-pointer/ssb-meta-feeds-spec
[bipf]: https://github.com/ssbc/bipf
[bamboo]: https://github.com/AljoschaMeyer/bamboo/
[ssb-bfe]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec
[ssb-bfe encrypted data format]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec#5-encrypted-data-formats
[blake3]: https://github.com/BLAKE3-team/BLAKE3
[EBT]: https://github.com/ssbc/ssb-ebt
[ssb-db2]: https://github.com/ssb-ngi-pointer/ssb-db2
[Why it is ok to sign hashes instead of full messages]: https://crypto.stackexchange.com/questions/6335/is-signing-a-hash-instead-of-the-full-data-considered-secure
