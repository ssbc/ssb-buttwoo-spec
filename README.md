# Buttwoo feed format

Status: **Ready to implement**

Buttwoo is a new binary feed format for [SSB]. It draws inspiration
from [bamboo] and [meta feeds].

The format is designed to be simple to implement, be easy to integrate
into existing implementations, be backwards compatible with existing
applications and lastly to be performant.

Buttwoo uses [bipf] for encoding since its a good foundation for an
append-only-log system (write once, many reads). It uses [ssb-bfe] for
binary encodings of feed and messages.

## Format

A buttwoo message consists of 8 fields encoded as an array and a
signature. The message key is the hash of the encoded values
concatenated with the raw signature bytes.

### Value

A bipf encoded value is an array of:

 - [ssb-bfe] encoded ed25519 author
 - [ssb-bfe] encoded parent message id used for subfeeds. For the top
   feed this must be BFE nil.
 - sequence number of the message in the feed
 - the timestamp of the message
 - [ssb-bfe] encoded previous message id. For the first message this
   must be BFE nil.
 - a byte with extensible tag information (`0x00` means standard
   message, `0x01` means subfeed, `0x02` means end-of-feed).
 - the length of the content in bytes
 - hash of content encoded as `0x00` concatenated with the [blake3]
   hash bytes

It is important to note that one author can have multiple feeds, each
feed defined as author + parent. `sequence` and `previous` relates to
the feed. Also note that unless parent is used, this behaves exactly
like an ordinary classic SSB feed.

## Subfeeds

As noted above it is possible to have multiple feeds with the same
author. To initiate a subfeed one create a special message on the
feed. This message must use the `0x01` tag and content can include
extra information about what is contained in the subfeed, such as the
feed purpose. The id of this message serves as the parent id of each
message on the subfeed.

In contrast with [bendy butt], subfeeds maintain the same feed
identitier. This makes it easier to work with in situations where,
what in classic SSB would be single feed, is split into multiple
parts. As an example, a feed could be split into: about messages, the
social graph and ordinary messages. While on the other hand [bendy
butt] had a clear separation between what are meta feeds and what are
normal feeds, allowing normal feeds to use different feed formats. In
this way, they can be seen as complementary.

### Content

If there is content and it is not encrypted, content SHOULD be a bipf
encoded object. Note this applies to the 3 tag types defined in this
document. One can use other tags to mean something else. This could be
used to carry for example files as content.

If encrypted, a [ssb-bfe encrypted data format] MUST be used.

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
     on the feed
   - a timestamp representing the UNIX epoch timestamp of message
     creation
   - a [ssb-bfe] encoded previous messages key on the feed
   - a byte representating a tag of either: `0x00`, `0x01` or `0x02`
   - the content length in bytes. This number must not exceed 16384.
   - content hash MUST start with `0x00` and be of length 33
 - Signature MUST sign the the encoded value using the authors key. It
   MUST be 64 bytes.

Content, if available MUST conform to the following rules: 
 - The byte length must match the content size in value
 - Content hashed with blake3 must match the content hash in values

## Integration with existing SSB stack

### EBT

Data sent over the wire should be bipf encoded as:

```
transport:       [value, signature, content]
value:           [author, parent, sequence, timestamp, previous, tag, contentLen, contentHash]
```

If content is not encrypted, then this value will be a bipf encoded
buffer. If encrypted, this will be a base64 encoded BFE string
representation.

The feedId should be author + parent.

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
[bendy butt]: https://github.com/ssb-ngi-pointer/bendy-butt-spec
[bipf]: https://github.com/ssbc/bipf
[bamboo]: https://github.com/AljoschaMeyer/bamboo/
[ssb-bfe]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec
[ssb-bfe encrypted data format]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec#5-encrypted-data-formats
[blake3]: https://github.com/BLAKE3-team/BLAKE3
[EBT]: https://github.com/ssbc/ssb-ebt
[ssb-db2]: https://github.com/ssb-ngi-pointer/ssb-db2
[Why it is ok to sign hashes instead of full messages]: https://crypto.stackexchange.com/questions/6335/is-signing-a-hash-instead-of-the-full-data-considered-secure
