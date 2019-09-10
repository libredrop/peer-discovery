This document describes the networking protocol to discover peers on LAN which
is used by `libredrop`. Although, the protocol is general purpose an can be
easilly reused for peer discovery on LAN.

The repo also has some protocol implementations. Feel free to make a PR with an
implementation for different languages.

## The protocol

Peer discovery works by periodically broadcasting beacon messages on LAN.

## Flow

The peer that wants to be discovered sends [discovery message](### Message format)
over LAN. The multicast address `255.255.255.255` is used to transfer peer discover
messages to all devices connected to the same LAN. The UDP port 5330 is used. 533 stands
for SEE in [leet language](https://en.wikipedia.org/wiki/Leet) and 0 is just a suffix
to make port be higher than 1024 (privileged ports).

```text
     +-------+        discovery_msg              +--------+
     | Peer1 | -------------------------->       | Router |
     +-------+   to=255.255.255.255:5330/UDP     +--------+
                                                     |
                                                     |   forwards to
                                                     | all peers on LAN
                                                     |
                                        +------------+----------+
                                        |                       |
                                        |                       |
                                        V                       V
                                    +-------+               +-------+
                                    | Peer2 |               | Peer3 |
                                    +-------+               +-------+
```

Each of those peers participating in some service discovery keep resending
discovery every 3 seconds - not too small number to avoid network congestion
and not too big to have relatively low latency of new peer discovery.

### Message format

Peer discovery message is a set of arbitrary key-value pairs:
```C
struct DiscoveryMsg {
    version: u8,
    items: HashMap<String, Bytes>,
}
```

Key is UTF-8 encoded string and value is an array of bytes.
Also, one byte version field is appended to allow format changes in the future.
Version is an integer number: 1, 2, 3, etc.

### Message binary format

The serialized message has to fit into 65000 bytes - maximum size of UDP packet.
The big-endian byte order is used to encode numeric values.

```text
      1B         2B          2B                2B    key1_len
  +---------+------------+----------+-----+----------+------+-----+-------+
  | version | item_count | key1_len | ... |key_n_len | key1 | ... | key_n |
  +---------+------------+----------+-----+----------+------+-----+-------+
        2B                2B       value1_len     value_x_len
  +------------+-----+-------------+--------+-----+---------+
  | value1_len | ... | value_x_len | value1 | ... | value_x |
  +------------+-----+-------------+--------+-----+---------+
```

* `version` (1 byte) -  protocol version number.
* `item_count` (2 bytes) - the number of key-value pairs.
* `key_x_len` (2 bytes) - defines how long is the key string. There is one `key_x_len`
    record for each item.
* `key_x` (key_x_len bytes) - UTF-8 encoded key content. There is one `key_x`
    record for each item.
* `value_x_len` (2 bytes) - defines how long is the bytes array of an item value.
   There is one `value_x_len` record for each item.
* `value_x` (2 bytes) - arbitrary bytes array represting item value.
   There is one `value_x` record for each item.
