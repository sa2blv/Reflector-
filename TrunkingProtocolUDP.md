#Trunk UDP Message System

![Status](https://img.shields.io/badge/status-active-green)
![Protocol](https://img.shields.io/badge/protocol-UDP-blue)
![C++](https://img.shields.io/badge/language-C++-brightgreen)

**Author:** Peter Lundberg / SA2BLV  
**Year:** 2026  

---

## – Table of Contents

- [Overview](#overview)
- [MsgUdpAudio_trunk](#msgudpaudio_trunk)
- [MSG_Trunk_Change](#msg_trunk_change)
- [MSG_Trunk_tg_subsribe](#msg_trunk_tg_subsribe)
- [MSG_Trunk_tg_heart_beat](#msg_trunk_tg_heart_beat)
- [Serialization Notes](#serialization-notes)
- [Bandwidth Summary](#bandwidth-summary)

---

## Overview

This document describes the UDP message types used in the trunk reflector system.

Key features:
- Lightweight UDP messaging
- Talkgroup routing control
- Audio streaming support
- Heartbeat monitoring
- Efficient serialization via `ASYNC_MSG_MEMBERS`

---

#  MsgUdpAudio_trunk

## Code
```cpp
class MsgUdpAudio_trunk : public ReflectorUdpMsgBase<101>
{
public:
    MsgUdpAudio_trunk(void) {}
    MsgUdpAudio_trunk(const std::vector<uint8_t>& audio_data)
      : m_audio_data(audio_data) {}

    MsgUdpAudio_trunk(const void *buf, int count)
    {
        if (count > 0)
        {
            const uint8_t *bbuf = reinterpret_cast<const uint8_t*>(buf);
            m_audio_data.assign(bbuf, bbuf + count);
        }
    }

    std::vector<uint8_t>& audioData() { return m_audio_data; }
    const std::vector<uint8_t>& audioData() const { return m_audio_data; }

    ASYNC_MSG_MEMBERS(m_audio_data, tg, Talker)

    int tg = 0;
    std::string Talker;
    int ttl = 10;

private:
    std::vector<uint8_t> m_audio_data;
};
```

## Payload Size
- Audio: variable (typically 160â€“320 bytes)
- Metadata overhead: small
- **Total:** ~180â€“360 bytes

---

# ðŸ”„ MSG_Trunk_Change

## Code
```cpp
class MSG_Trunk_Change : public ReflectorMsgBase<140>
{
public:
    int talker_status = 0;
    int qsy = 0;
    int tg = 0;
    int new_tg = 0;
    std::string talker;
    int ttl = 10;

    ASYNC_MSG_MEMBERS(talker_status, qsy, tg, new_tg, talker);
};
```

## Payload Size
- Integers: 16 bytes
- String: variable
- **Total:** ~30â€“50 bytes

---

# MSG_Trunk_tg_subsribe

## Code
```cpp
class MSG_Trunk_tg_subsribe : public ReflectorMsgBase<131>
{
public:
    std::string trunkid;
    std::vector<int> Talkgroups;
    int ttl = 10;

    ASYNC_MSG_MEMBERS(trunkid, Talkgroups);
};
```

## Payload Size
- trunkid: ~15â€“40 bytes
- Talkgroups: 4 bytes per TG
- **Total:** ~40â€“100 bytes

---

#  MSG_Trunk_tg_Heart_beat

## Code
```cpp
class MSG_Trunk_tg_Heart_beat : public ReflectorMsgBase<132>
{
public:
    std::string trunkid;
    int status = 0;
    int nr = 0;
    int ttl = 10;

    ASYNC_MSG_MEMBERS(trunkid, status, nr);
};
```

## Payload Size
- trunkid: ~15â€“40 bytes
- ints: 8 bytes
- **Total:** ~25â€“50 bytes

---

#  Serialization Notes

- Only fields listed in `ASYNC_MSG_MEMBERS` are transmitted
- `ttl` is NOT serialized
- `std::string` = length + data
- `std::vector` = length + elements

---

#  Bandwidth Summary

| Message Type      | Size (Approx) |
|------------------|--------------|
| Audio            | 180â€“360 B    |
| Change           | 30â€“50 B      |
| Subscribe        | 40â€“100 B     |
| Heartbeat        | 25â€“50 B      |

---


---

#  Architecture
           UDP Packet
                ↓
       Security / Decrypt
                ↓
         Header Decode
                ↓
        Type Dispatcher
     ┌──────┬──────┬──────┐
     ↓      ↓      ↓      ↓
  Audio  Control  Heart  Nodes
     ↓      ↓      ↓      ↓
 Routing Engine + TG Mapping
                ↓
      Trunk Forwarding Layer
---


---
|  Type      | Message            |
|------------------|--------------|
| 101        | 	Audio             |
| 131        | TG Subscribe       |
| 132        | 	Heartbeat         |
| 133        | Node Broadcast     |
| 140        | Talker Control     |
---

	





##  Design Goals

- Low-latency UDP transport
- Minimal control overhead
- Efficient talkgroup distribution
- Scalable reflector architecture



