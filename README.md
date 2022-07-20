# Buttwoo feed format

Status: **In review**

Buttwoo is a new binary feed format for [SSB]. It draws inspiration
from [bamboo] and [meta feeds].

The format is designed to be simple to implement, be easy to integrate
into existing implementations, be backwards compatible with existing
applications and lastly to be performant.

Buttwoo uses [bipf] for encoding since its a good foundation for an
append-only-log system (write once, many reads). It uses [ssb-bfe] for
binary encodings of feed IDs and message IDs.

## Format

A buttwoo message consists of a bipf-encoded array of 3 fields:

- `metadata`
- `signature`
- `content`

The message ID is the BFE encoding of the [blake3] hash of the
concatenation of `metadata` bytes with `signature` bytes.

### Metadata

The `metadata` field is a bipf-encoded array with 8 fields:

 - `author`: [ssb-bfe]-encoded buttwoo feed ID, an ed25519 public key
 - `parent`: [ssb-bfe]-encoded buttwoo message ID used for subfeeds. 
   For the top feed this must be BFE nil.
 - `sequence`: integer representing the position for this message in
   the feed. Starts from 1.
 - `timestamp`: integer representing the UNIX epoch timestamp of 
   message creation
 - `previous`: [ssb-bfe]-encoded message ID for the previous message 
   in the feed. For the first message this must be BFE nil.
 - `tag`: a byte with extensible tag information (the value `0x00` 
   means a standard message, `0x01` means subfeed, `0x02` means 
   end-of-feed). One can use other tags to mean something else. This 
   could be used to carry for example files as content.
 - `contentLength`: the length of the bipf-encoded `content` in bytes
 - `hash`: concatenation of `0x00` with the [blake3] hash of the 
    bipf-encoded `content` bytes
    
### Signature

The `signature` uses the same HMAC signing capability (`sodium.crypto_auth`) 
and `sodium.crypto_sign_detached` as in the classic SSB format (ed25519).

It is important to note that one author can have multiple feeds, each
feed defined as author + parent. `sequence` and `previous` relates to
the feed. Also note that unless parent is used, this behaves exactly
like an ordinary classic SSB feed.

### Content

The `content` is a free form field. When unencrypted, it SHOULD be a 
bipf-encoded object. If encrypted, `content` MUST be an 
[ssb-bfe encrypted data format].

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
 - Metadata must be an bipf encoded array of 8 elements:
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
 - Signature MUST sign the the encoded metadata using the authors key. It
   MUST be 64 bytes.

Content, if available MUST conform to the following rules: 
 - The byte length must match the content size in value
 - Content hashed with blake3 must match the content hash in values

## Integration with existing SSB stack

### EBT

Data sent over the wire should be bipf encoded as:

```
transport:  [metadata, signature, content]
metadata:   [author, parent, sequence, timestamp, previous, tag, contentLen, contentHash]
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
metadata part, bipf is a relatively simple format. The JavaScript
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
