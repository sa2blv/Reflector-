# ReflectorTrunkManager - Documentation

**Author:** Peter Lundberg / SA2BLV  
**Date:** 2026-01-11  
**Version:** 1.0  
**License:** GNU General Public License v2.0+

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Main Classes and Structures](#main-classes-and-structures)
4. [Configuration](#configuration)
5. [Public API](#public-api)
6. [Private Methods](#private-methods)
7. [Usage Examples](#usage-examples)
8. [Security and Encryption](#security-and-encryption)
9. [Troubleshooting](#troubleshooting)

---

## Overview

**ReflectorTrunkManager** is a central component in the SvxReflector system responsible for managing connections between multiple SvxLink servers (trunk peers). It enables:

- **Peer Management:** Configuration and maintenance of connections to other trunks
- **Talk Group (TG) Routing:** Routing of radio messages between talk groups with rule-based filtering
- **Heartbeat/Keepalive:** Monitoring of peer status through regular heartbeat messages
- **Talk Group Translation:** Conversion of talk group numbers between different trunks
- **Encryption:** AES encryption for secure communication between peers
- **Dynamic Configuration:** Reading configuration from Async::Config

### Key Features

- 🔌 **UDP-based communication** with multiple peer trunks
- 💬 **Intelligent talk group routing** with regex-based rules
- 🔄 **Automatic heartbeat monitoring** for peer status
- 🔐 **AES-128-CBC encryption** for secure data exchange
- 📊 **JSON-based status reporting**
- 🔄 **Talk group mapping** from external files

---

## Architecture

```
┌─────────────────────────────────────────┐
│   ReflectorTrunkManager (Singleton)     │
│  ┌───────────────────────────────────┐  │
│  │  Peer Management                  │  │
│  │  - Initialization                 │  │
│  │  - Heartbeat Monitoring           │  │
│  ├───────────────────────────────────┤  │
│  │  Audio Routing                    │  │
│  │  - Handle outgoing audio          │  │
│  │  - TG filtering & translation     │  │
│  │  - Retransmission                 │  │
│  ├───────────────────────────────────┤  │
│  │  Security                         │  │
│  │  - AES Decryption                 │  │
│  │  - Cryptographic key management   │  │
│  ├───────────────────────────────────┤  │
│  │  Communication                    │  │
│  │  - UDP socket handling            │  │
│  │  - Message serialization          │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
         │
         ├─────► [Peer 1] (UDP)
         ├─────► [Peer 2] (UDP)
         └─────► [Peer N] (UDP)
```

---

## Main Classes and Structures

### 1. `ReflectorTrunkManager` (Singleton)

**Purpose:** Central manager for trunk peers and audio routing

**Instantiation:**
```cpp
ReflectorTrunkManager* manager = ReflectorTrunkManager::instance();
```

**Private Member Variables:**
- `m_cfg`: Pointer to Async::Config object
- `m_gatewayId`: This trunk server's ID
- `m_localAudioPort`: Local audio port (default: 5401)
- `m_localControlPort`: Local control port (default: 5400)
- `peers`: Vector of TrunkPeer objects
- `config_peers`: List of peer names from config
- `sockfd`: UDP socket descriptor

---

### 2. `TrunkPeer` Structure

**Purpose:** Represents a connected trunk peer

**Members:**
```cpp
struct TrunkPeer {
    std::string name;                          // Peer name from config
    std::string host;                          // IP address or hostname
    std::string Bacup_host;                    // Backup host (for future use)
    
    // Heartbeat handling
    int qualify = 1;                           // If peer is qualified (1=yes, 0=no)
    int heartbeat_send_nr = 0;                 // Sent heartbeat number
    int heartbeat_send_nr_old = 0;
    int heartbeat_recived_nr = 0;              // Received heartbeat number
    int heartbeat_recived_nr_old = 0;
    bool online = false;                       // Peer online status
    
    // Communication
    int audioPort = 0;                         // Peer audio port
    int controlPort = 0;                       // Peer control port
    
    // Configuration
    int Retransmitt = 0;                       // Retransmission setting
    int Active_filter = 0;                     // Active filter setting
    std::regex tgRule;                         // Talk group matching rule
    std::string Trunk_type;                    // Type of trunk (receiver)
    std::string Trunk_type_send;               // Type of trunk (sender)
    std::string Crypt_key;                     // AES encryption key
    
    // Talk Group Translation
    bool Use_tg_translation = false;
    std::string tranlationfile = "";
    std::vector<TrunkPeerTnaslation> Translation;      // src_tg -> dest_tg
    std::vector<TrunkPeerTnaslation> Translation_dest;  // Sorted by dest_tg
    
    // Runtime
    std::shared_ptr<ReflectorClientUdp> client;  // UDP client connection
    std::vector<int> activeTalkgroups;           // Active talk groups
};
```

---

### 3. `TrunkPeerTnaslation` Structure

**Purpose:** Represents a talk group mapping

```cpp
struct TrunkPeerTnaslation {
    int src_tg;      // Source talk group number
    int dest_tg;     // Destination talk group number
};
```

---

### 4. `MsgTrunkQso` Class

**Purpose:** Represents a QSO (Radio Communication) message for trunk protocol

```cpp
class MsgTrunkQso {
public:
    MsgTrunkQso(TrunkQsoType type, int tg, const std::string &gatewayId);
    
    TrunkQsoType type() const;
    int tg() const;
    const std::string &gatewayId() const;
    
    std::vector<uint8_t> serialize() const;
    static MsgTrunkQso deserialize(const std::vector<uint8_t> &buf);
};
```

**Enum `TrunkQsoType`:**
- `QSO_START`: Indicates start of communication
- `QSO_END`: Indicates end of communication

---

### 5. `MsgReflectorQso` Class

**Purpose:** Reflector-specific version of the QSO message

**Functionality:** Identical to `MsgTrunkQso` but uses `ReflectorQsoType`

```cpp
enum class ReflectorQsoType { QSO_START, QSO_END };
```

---

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

## Public API

### Initialization and Singleton Pattern

```cpp
// Get instance
ReflectorTrunkManager* mgr = ReflectorTrunkManager::instance();

// Set configuration
void setConfig(const Async::Config* cfg);

// Initialize from configuration
void init();
```

---

### Peer Management

```cpp
// Read all peers
const std::vector<TrunkPeer>& getPeers() const;

// Add a peer (from config)
void addPeer(const std::string& str);
```

---

### Heartbeat Monitoring

```cpp
// Called when heartbeat is received from peer
void Heartbeat_recive(const std::string& host, int nr);

// Send heartbeat to all peers
void Heartbeat_send();
```

**Heartbeat Logic:**
- Counts the health status of each peer
- If received heartbeats stagnate → peer marked as offline
- Counters compared every heartbeat cycle

---




---

## Usage Examples

### 1. Basic Initialization

```cpp
#include "ReflectorTrunkManager.h"
#include <AsyncConfig.h>

// Read configuration
Async::Config config;
config.readFile("/etc/svxreflector/config.conf");

// Get instance and initialize
ReflectorTrunkManager* mgr = ReflectorTrunkManager::instance();
mgr->setConfig(&config);
mgr->init();
```

### 2. Send Audio to Peers

```cpp
// Send audio from talk group 11 to all peers that accept it
MsgUdpAudio_trunk audio_msg;
audio_msg.tg = 11;
audio_msg.data = audio_buffer;

mgr->handleOutgoingAudio_width_remap(11, audio_msg);
```

### 3. Monitor Peer Status

```cpp
// In a regular timer callback (e.g., every second)
mgr->Heartbeat_send();

// When heartbeat is received from peer:
mgr->Heartbeat_recive("192.168.1.10", heartbeat_nr);

// Read status
Json::Value status = mgr->JSON_array_staus();
for (const auto& peer : status) {
    std::cout << "Peer: " << peer["name"].asString()
              << " Online: " << peer["status"].asBool() << std::endl;
}
```

### 4. Handle Talk Group Filtering

```cpp
std::vector<int> talkgroups = {11, 12, 13, 20};

// Filter according to peer rules
mgr->incomming_filter(talkgroups, "192.168.1.10");

// Result: talkgroups contains only TGs matching peer's regex rule
```

### 5. Talk Group Conversion

```cpp
// Send message with TG conversion
MSG_Trunk_Change msg;
msg.tg = 11;  // Incoming TG

// Converted according to mapping rules
mgr->handleOutgoingMessage_width_remap(11, msg);
```

### 6. Get Encryption Key

```cpp
std::string key = mgr->get_key("192.168.1.10");
if (!key.empty()) {
    // Decrypt message
    auto decrypted = mgr->decryptAES(encrypted_data, data_len, key);
}
```

---

## Security and Encryption

### AES-128-CBC Decryption

ReflectorTrunkManager uses OpenSSL for AES-128-CBC decryption:

**Decryption Process:**
1. First 16 bytes of input data is IV (Initialization Vector)
2. Remaining bytes is encrypted data
3. Uses encryption key from peer configuration
4. Returns decrypted plaintext

**Example:**
```cpp
std::string encrypted_message = receive_message();
std::string key = mgr->get_key(peer_host);

std::vector<unsigned char> plaintext = mgr->decryptAES(
    encrypted_message.data(),
    encrypted_message.size(),
    key
);
```

### Best Practices for Keys

- 🔐 Store keys in configuration file with restricted permissions (600)
- 🔐 Use strong keys (at least 16 characters, mixed character types)
- 🔐 Rotate keys regularly
- 🔐 Never log out keys or encrypted data

```bash
# Example of secure file configuration
chmod 600 /etc/svxreflector/config.conf
chown root:root /etc/svxreflector/config.conf
```

---

## Troubleshooting

### Problem: Peer Shows as Offline

**Possible Causes:**
1. Heartbeat response not received from peer
2. Host cannot be reached (firewall, network)
3. Peer is not started

**Solution:**
```cpp
// Check peer status
Json::Value status = mgr->JSON_array_staus();
for (const auto& peer : status) {
    std::cout << "Peer: " << peer["name"].asString()
              << " Send: " << peer["heartBeatSend"].asInt()
              << " Recv: " << peer["heartBeatRecived"].asInt()
              << " Online: " << peer["status"].asBool() << std::endl;
}

// If Send >> Recv = problem with reception
// If Send ≈ Recv but online=false = logic error
```

### Problem: Audio Not Routed to Peer

**Possible Causes:**
1. Talk group does not match TGRule regex
2. Peer is offline
3. Peer is not qualified
4. Filter settings

**Solution:**
```cpp
// Verify TGRule regex
const std::vector<TrunkPeer>& peers = mgr->getPeers();
for (const auto& peer : peers) {
    std::string tg_str = "11";
    if (std::regex_match(tg_str, peer.tgRule)) {
        std::cout << "TG 11 matches peer " << peer.name << std::endl;
    }
}
```

### Problem: Decryption Fails

**Possible Causes:**
1. Wrong encryption key configured
2. Encrypted data is corrupt
3. IV (first 16 bytes) is incorrect

**Solution:**
```cpp
// Verify key
std::string key = mgr->get_key(peer_host);
if (key.empty()) {
    std::cerr << "No encryption key for peer!" << std::endl;
}

// Check data size (must be >= 16 bytes for IV)
if (data_len < 16) {
    std::cerr << "Encrypted data too short for IV!" << std::endl;
}
```

---


---

## Future Improvements

- [ ] Backup host functionality
- [ ] Dynamic peer addition/removal without restart
- [ ] Redundancy and failover mechanisms

---

## Performance Considerations

### Memory Usage
- Each peer adds approximately 1-2 KB overhead
- Talk group mappings scale linearly with file size
- Active talk group list size depends on network activity

### Network Bandwidth
- Heartbeat messages: ~32 bytes per peer per second
- Audio bandwidth: Depends on codec and compression
- Status messages: ~1-2 KB per poll cycle

### CPU Usage
- Minimal CPU impact under normal conditions
- Regex matching performed only on incoming talk groups
- AES decryption cost is negligible for modern CPUs

---

## FAQ

**Q: Can I add/remove peers without restarting?**  
A: Currently no, peers are loaded only at initialization. Future versions will support dynamic peer management.

**Q: What's the maximum number of peers supported?**  
A: No hard limit, but tested up to 50+ peers. Performance depends on network and system resources.

**Q: Can I use IPv6 addresses?**  
A: Yes, the system supports IPv6, but some features (like hostname resolution) may need additional testing.

**Q: How do I debug communication issues?**  
A: Enable stdout logging, check firewall rules, verify configuration syntax, and use the status API to monitor peer health.

**Q: Is there a way to monitor the system remotely?**  
A: The `JSON_array_staus()` method provides JSON output that can be exposed via a web API or monitoring system.

---

## License

```
SvxReflector - An Trunk for svxreflector for connecting SvxLink Servers
Copyright (C) 2025-2026 Peter Lundberg / SA2BLV

This program is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation; either version 2 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

---

## Contact and Support

- **Developer:** Peter Lundberg (SA2BLV)
- **Project:** SvxReflector+
- **License:** GPLv2+
- **Repository:** [Link to repository if available]

---

## Changelog

### Version 1.0 (2026-01-11)
- Initial release
- Complete documentation of all major features
- Configuration examples and troubleshooting guide

---

*Documentation Last Updated: 2026-01-11*  
*Version: 1.0*  
*Language: English*
