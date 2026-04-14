## Configuration

### Configuration Structure

ReflectorTrunkManager reads its configuration from an `Async::Config` object. The configuration is divided into several sections:

#### `[ReflectorTrunk]` - Main Section

```ini
[ReflectorTrunk]
GatewayId = MyReflector1
Port = 5401
Peers = peer1,peer2,peer3
```

**Parameters:**
- `GatewayId`: This trunk server's identifier
- `Port`: Local port for audio and control
- `Peers`: Comma-separated list of peer identifiers

#### `[TrunkPeer#<n>]` - Peer-Specific Section

For each peer, a section must be defined:

```ini
[TrunkPeer#peer1]
Host = 192.168.1.10
Port = 5401
Qualify = 1
Retransmit = 0
ActiveFilter = 0
TGRule = ^(11|12|13)[0-9]*$
TgMapFile = /etc/svxreflector/tg_mapping_peer1.txt
Key = MySecretKey123456
```

**Parameters:**

| Parameter | Type | Example | Description |
|-----------|------|---------|-------------|
| `Host` | String | `192.168.1.10` | IP address or hostname for peer |
| `Port` | Integer | `5401` | UDP port for peer communication |
| `Qualify` | Integer (0/1) | `1` | Whether peer should be activated |
| `Retransmit` | Integer | `0` | Retransmission setting |
| `ActiveFilter` | Integer | `0` | Filter setting for active TGs |
| `TGRule` | Regex | `^(11\|12\|13)` | Regex to filter allowed talk groups |
| `TgMapFile` | String | `/etc/tg_map.txt` | Path to TG mapping file |
| `Key` | String | `MyKey123` | AES encryption key (optional) |

#### Talk Group Mapping File

Mapping file format (use `;` as separator):

```
11;111
12;112
13;113
100;1100
```

**Format:** `src_tg;dest_tg`
- `src_tg`: Incoming talk group number
- `dest_tg`: Outgoing talk group number for this peer

---
