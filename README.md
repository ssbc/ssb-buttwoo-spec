# ssb-bipf-format

- Based on [bipf] as that is a good foundation for an append-only-log system (write once, many reads)
- Similar to [bendy-butt] uses simple encoding and [ssb-bfe] for binary encodings
- Similar to [bamboo] signs the hash only
- Should include aggregate hashes (maybe input is `list<hash>` + `list<signature>`). This should be extensible, with something like every 25 as a good default for better performance([ssb-performance-notes])

[bipf]: https://github.com/ssbc/bipf
[bamboo]: https://github.com/AljoschaMeyer/bamboo/
[bendy-butt]: https://github.com/ssb-ngi-pointer/bendy-butt-spec
[ssb-bfe]: https://github.com/ssb-ngi-pointer/ssb-bfe-spec
[ssb-performance-notes]: https://github.com/arj03/ssb-performance-notes
