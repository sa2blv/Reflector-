```
  _____       __ _           _                    
 |  __ \     / _| |         | |              _    
 | |__) |___| |_| | ___  ___| |_ ___  _ __ _| |_  
 |  _  // _ \  _| |/ _ \/ __| __/ _ \| '__|_   _| 
 | | \ \  __/ | | |  __/ (__| || (_) | |    |_|   
 |_|  \_\___|_| |_|\___|\___|\__\___/|_|          
```
                                          
# Reflector+ Is a trunking reflector with support for multiple trunk types.

Reflector+ is a fork of [SvxReflector](https://github.com/sm0svx/svxlink)
extended with a **server-to-server trunkoing protocols** that lets multiple reflector
instances share talk groups (TGs) as independent parallel voice channels —
analogous to telephone trunk lines between telephone exchanges.
```
Original SvxReflector by Tobias Blomberg / SM0SVX.
GEU Trunk extension by IW1GEU.
UDPtrunk extension by SA2BLV.
ExternalConfig extension by SA2BLV.
RoutingTable extension by SA2BLV.
```
---

## Why trunking?

A standard SvxReflector is a single centralized server. Without trunking, a
multi-site national deployment has two unsatisfying options:

- **One national server** — only one TG can be active at a time across the
  entire network; one outage takes everyone down.
- **Independent regional servers bridged by SvxLink instances** — technically
  possible, but each bridge instance handles only one TG at a time and one
  instance is needed per pair of servers. For 20 regions in full mesh that is
  190 SvxLink bridge processes for a single shared TG, multiplied by the number
  of TGs to share. Unmanageable at scale.

GeuReflector solves this: each region runs its own independent reflector
(resilience, local autonomy) and the trunk links connect them so they can share
TGs selectively. All regional TGs can carry simultaneous independent QSOs while
a national TG remains available to all. Talker arbitration is handled
automatically.

---



## What it adds over SvxReflector

## BLV trunking

- **Talkgroup Management** - Handles talkgroups per client, Auto QSY, monitored TGs
- **Trunk Linking** - Connections between multiple peer reflectors with node list exchange
- **Routing & Node Management** - Dynamic routing table with nodes (callsign, TG, trunk)
- **MQTT Messaging** - Message handling 
- **Remote Configuration Management** - Asynchronous configuration and file I/O

## GEU Trunking

- **Prefix-based TG ownership** — each reflector is authoritative for all TGs
  whose decimal representation starts with a configured digit string (e.g.
  `LOCAL_PREFIX=1` → owns TGs 1, 10, 100, 1000, 12345 …). Both `LOCAL_PREFIX`
  and `REMOTE_PREFIX` accept comma-separated lists for reflectors that own
  multiple prefix groups (e.g. `LOCAL_PREFIX=11,12,13`)
- **Server-to-server trunk links** — persistent TCP connections between pairs of
  reflectors carry talker state and audio for shared TGs
- **Full-mesh topology** — any number of reflectors can be trunked together; each
  pair has a direct link so no multi-hop routing is needed
- **Unlimited concurrent conversations** — the trunk TCP connection multiplexes
  all active TGs simultaneously; the only per-TG rule is one talker at a time
- **Independent talker arbitration** — each reflector arbitrates its own clients;
  the trunk adds a tie-breaking layer (random priority nonce) for simultaneous
  claims
- **Cluster TGs** — BrandMeister-style nationwide talk groups that are broadcast
  to all trunk peers regardless of prefix ownership
- **Satellite reflectors** — lightweight relay instances that connect to a parent
  reflector instead of joining the full mesh, reducing configuration overhead for
  large deployments
- **HTTP `/status` endpoint** — JSON status with trunk state,
  active talkers, satellite connections, and static configuration


---

## How TG ownership works

TG numbers work like telephone numbers. The reflector that owns a TG prefix is
the authoritative home for every TG in that prefix group:

```
Reflector 1  (LOCAL_PREFIX=1)   →  TGs  1, 10, 11, 100, 1000, 12345, …
Reflector 2  (LOCAL_PREFIX=2)   →  TGs  2, 20, 21, 200, 2000, 25000, …
Reflector 3  (LOCAL_PREFIX=3)   →  TGs  3, 30,  …
```

When a client on Reflector 1 joins TG 25, Reflector 1 forwards the conversation
over its trunk to Reflector 2 (which owns the "2" prefix). Clients on both
reflectors hear the same audio on TG 25 with no extra configuration.

---

## Build

Requires the same dependencies as SvxReflector: libsigc++, OpenSSL, libjsoncpp,
libpopt, libmosquitto. Optional: libopus, libgsm, libspeex. libpaho-mqttpp-dev

```bash
cd geureflector
cmake -S src -B build -DLOCAL_STATE_DIR=/var
cmake --build build
# binary at build/bin/svxreflector
```

---

## Testing

Integration tests spin up a 3-reflector Docker mesh and verify trunk routing,
audio delivery, and protocol behavior:

```bash
cd tests && bash run_tests.sh
```

Requires Docker and Python 3.7+. The script builds the images, starts the mesh,
runs 18 automated tests, then drops into an interactive prompt where you can
enter any TG number and see which reflector it routes to (verified via container
logs).

The mesh topology is defined in `tests/topology.py` — edit prefixes, cluster
TGs, or add reflectors there, and `run_tests.sh` regenerates all configs
automatically.

---

### Network requirements

**Trunk links** require **mutual reachability**: each reflector both listens for
inbound trunk connections and connects outbound to every peer.  Both sides
attempt to connect simultaneously; outbound and inbound connections operate
independently and can both be active at the same time.  When sending, the
outbound connection is preferred with inbound as fallback.  All reflectors in
the mesh need a static IP (or
stable DNS name) and the trunk port (default 5302, configurable via
`TRUNK_LISTEN_PORT`) open for inbound TCP connections.

**Satellite links** are **one-way**: the satellite connects outbound to the
parent. Only the parent needs a static IP and its satellite port (default 5303)
open. The satellite itself does not need a static IP or any open ports — it can
run behind NAT, just like a regular SvxLink client node.

| Port | Protocol | Direction | Required on |
|------|----------|-----------|-------------|
| 5300 | TCP+UDP | Inbound | All reflectors (client connections) |
| 5302 | TCP | Inbound | All trunk mesh reflectors (peer connections) |
| 5303 | TCP | Inbound | Parent reflectors accepting satellites |
| 5500 | UDP | Inbound | udp trunk mesh reflectors |
| 5500 | UDP | Outbound | udp trunk mesh reflectors |
| — | TCP | Outbound | Satellites (no inbound ports needed) |





## HTTP Status

Enable with `HTTP_SRV_PORT=8080` in `[GLOBAL]`.

### `GET /status_geu`

Returns live state (nodes, trunk connections, active talkers, satellites) and
static configuration in a single response:

```json
{
  "version": "1.0.0",
  "mode": "reflector",
  "local_prefix": ["1"],
  "listen_port": "5300",
  "http_port": "8080",
  "nodes": { ... },
  "cluster_tgs": [222, 2221, 91],
  "trunks": {
    "TRUNK_2": {
      "host": "reflector-b.example.com",
      "port": 5302,
      "connected": true,
      "local_prefix": ["1"],
      "remote_prefix": ["2"],
      "active_talkers": {
        "222": "IW1GEU",
        "25": "SM0ABC"
      }
    }
  },
  "satellites": {
    "my-satellite": {
      "id": "my-satellite",
      "authenticated": true,
      "active_tgs": [1, 100]
    }
  },
  "satellite_server": {
    "listen_port": "5303",
    "connected_count": 2
  }
}
```

`active_talkers` lists TGs with an active remote talker at query time (both
prefix-based and cluster TGs). `satellites` and `satellite_server` appear only
when applicable.


---

## Documentation

- [`docs/INSTALL.md`](docs/INSTALL.md) — how to build and install GeuReflector
  as a drop-in replacement for an existing SvxReflector installation
- [`docs/DOCKER.md`](docs/DOCKER.md) — running GeuReflector in a Docker container
  as a drop-in replacement for an existing SvxReflector installation
- [`docs/TRUNK_PROTOCOL.md`](docs/TRUNK_PROTOCOL.md) — GEU full wire protocol
  specification: message format, handshake sequence, talker arbitration
  tie-breaking, heartbeat, and the complete message type table
- [`docs/DESIGN_SATELLITE_AND_CLUSTER.md`](docs/DESIGN_SATELLITE_AND_CLUSTER.md) — design
  rationale for satellite reflectors and cluster TGs

- [`docs/Config-example.md`](docs/Config-example.md) — Configuration example
  
- [`docks/ExternalConfig.md`](docs/ExternalConfig.md) — Remote configuration poller.

- [`docks/SetupUDPTrunk.md`](docs/SetupUDPTrunk.md) — Udp Trunk setup guide.

- [`docks/ReflectorTrunkManager.md`](docs/ReflectorTrunkManager.md) — ReflectorTrunkManager documentation

- [`docks/ReflectorTrunkManager.md`](docs/GeuTrunk.md) — Geu trunk documentation

- [`tests/TESTS.md`](tests/TESTS.md) — integration test suite documentation:
  topology, test cases, harness components, and how to run

---

## License

GNU General Public License v2 or later — same as SvxLink upstream.

---
