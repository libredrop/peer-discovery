This document describes the networking protocol to discover peers on LAN which
is used by `libredrop`. Although, the protocol is general purpose an can be
easilly reused for peer discovery on LAN by any other application.

## The protocol

Peer discovery works by periodically broadcasting UDP messages on LAN.
Each message describes the service being advertised and how to contact with
it on this particular local area network.

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
3 seconds is the default, but this number is arbitrary and can be changed
by implementers.

### Message format

Peer discovery message is a set of arbitrary key-value pairs:
```C
enum TransportProtocol {
    TCP = 0,
    UDP = 1,
}

struct DiscoveryMsg {
    // Protocol version.
    version: u8,
    // Unique peer ID.
    id: Uuid4,
    // The name of the services peer advertises so other peers would know
    // if they are interested or not.
    service_name: String,
    protocol: TransportProtocol,
    // The port that remote peer exposes its services on.
    service_port: unsigned int16,
    // A list of IP addresses to contact remote peer.
    ipv4_addrs: Vec<Ipv4Addr>,
    // Arbitrary data
    items: HashMap<String, Bytes>,
}
```

One byte `version` field specifies protocol version which allows format changes
in the future. Version is an integer number: 1, 2, 3, etc.

`id` is 128 bit [unique peer ID](https://en.wikipedia.org/wiki/Universally_unique_identifier).
It is used to differentiate the peers, identify ourselves when the router
forwards broadcast message back to us, etc.

Since `peerdiscovery` can be used for multiple different services
simulataneously we need a way to tell which services is being advertised.
Hence `service_name` field, e.g. "libredrop", "my-network-game", etc.

Then every networking service will be listening on some `port`. Since a peer
on LAN might have multiple network interfaces, it sends a list of
`ipv4_addrs`.

Finally, peer discovery message can contain arbitrary data, e.g. public key.
For this, there is a key-value mappings where key is UTF-8 encoded string and
value is an array of bytes.

### Message binary format

The serialized message has to fit into 65000 bytes - maximum size of UDP packet.
The big-endian byte order is used to encode numeric values.

```text
      1B     16B          1B        service_name_len      1B
  +---------+----+------------------+--------------+-------------------+
  | version | ID | service_name_len | service_name | transport_protocol|
  +---------+----+------------------+--------------+-------------------+

        2B           1B        4B           4B        1B
  +--------------+----------+-----+-----+------+------------+
  | service_port | ip_count | IP1 | ... | IP_N | item_count |
  +--------------+----------+-----+-----+------+------------+

      1B               1B     key1_len     key_n_len
  +----------+-----+----------+------+-----+-------+
  | key1_len | ... |key_n_len | key1 | ... | key_n |
  +----------+-----+----------+------+-----+-------+

        2B                2B       value1_len     value_x_len
  +------------+-----+-------------+--------+-----+---------+
  | value1_len | ... | value_x_len | value1 | ... | value_x |
  +------------+-----+-------------+--------+-----+---------+
```

* `version` (1 byte) -  protocol version number.
* `ID` (16 bytes) - 128bit unique identifier, big endian encoding.
* `service_name_len` (1 byte) - the length of the service name string.
   NOTE, that since `service_name` is UTF-8, this is NOT a number of symbols,
   but rather a number of bytes.
* `service_name` (service_name_len) - UTF-8 encoded service name.
* `transport_protocol` (1 bytes) - the protocol that peer exposes its service
   over. 0 - TCP, 1 - UDP.
* `service_port` (2 bytes) - TCP or UDP port of the service the peer exposes.
* `ip_count` (1 byte) - the number of IPv4 addresses the peer has.
* `IP_N` (4 bytes) - all peer IPv4 addresses each encoded with big endian.
* `item_count` (1 byte) - the number of key-value pairs.
* `key_x_len` (1 byte) - defines how long is the key string. There is one `key_x_len`
    record for each item. NOTE, that keys are UTF-8 strings, this is NOT
    a number of symbols, but rather a number of bytes.
* `key_x` (key_x_len bytes) - UTF-8 encoded key content. There is one `key_x`
    record for each item.
* `value_x_len` (2 bytes) - defines how long is the bytes array of an item value.
   There is one `value_x_len` record for each item.
* `value_x` (2 bytes) - arbitrary bytes array represting item value.
   There is one `value_x` record for each item.
