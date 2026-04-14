## Configuration

GeuReflector uses the same `svxreflector.conf` format as SvxReflector with two
additions.

### 1. Declare this reflector's TG prefix

Add `LOCAL_PREFIX` to the `[GLOBAL]` section. A comma-separated list is accepted
when a single reflector owns multiple prefix groups:

```ini
[GLOBAL]
LISTEN_PORT=5300
LOCAL_PREFIX=1              # owns TGs 1, 10, 100, 1000, ...
# LOCAL_PREFIX=11,12,13     # multiple prefixes on one instance
# TRUNK_LISTEN_PORT=5302    # trunk server port (default 5302)
```

### 2. Cluster TGs (optional)

Cluster TGs are broadcast to **all** trunk peers regardless of prefix ownership,
like BrandMeister's nationwide talk groups. Any reflector can originate a
transmission on a cluster TG and all other reflectors in the mesh will hear it.

```ini
[GLOBAL]
CLUSTER_TGS=222,2221,91    # comma-separated list of cluster TG numbers
```

Each reflector owner chooses which cluster TGs to subscribe to. A reflector only
sends and accepts traffic for cluster TGs listed in its own `CLUSTER_TGS`. If
reflector A subscribes to TG 222 but reflector B does not, A will send TG 222
traffic to B, but B will ignore it — this is normal operation, not a
misconfiguration. Only reflectors that both subscribe to a given cluster TG will
exchange audio for it.

Cluster TG numbers must not overlap with any `LOCAL_PREFIX` or `REMOTE_PREFIX`.

**Note:** Satellite links are unaffected by cluster TG configuration — they
forward all TGs unconditionally (see [Satellite reflectors](#satellite-reflectors)).

### 3. Trunk debug logging (optional)

Enable verbose trunk logging to diagnose connection issues:

```ini
[GLOBAL]
TRUNK_DEBUG=1
```

When enabled, the reflector logs detailed information about every trunk
connection state change, including:

- Connection and disconnection events with full state (both directions'
  hello/heartbeat counters, whether the other direction is up)
- Heartbeat countdown warnings as the RX counter approaches timeout
- Every non-heartbeat frame received, with direction (IB/OB), type, and size
- Send path decisions (outbound, fallback to inbound, or dropped)
- Trunk server peer matching: which sections matched or mismatched on HMAC
  secret and prefix, making configuration errors immediately visible

**Warning:** `TRUNK_DEBUG` logs **every audio frame** received over trunk. During
active transmissions this means dozens of lines per second per TG per trunk link
(e.g. ~50 lines/sec with 20ms Opus framing). On a busy reflector with multiple
simultaneous talkers across several trunk peers, this can produce hundreds of
log lines per second and rapidly fill disk. **Do not leave `TRUNK_DEBUG=1`
enabled in production.** Use it only to diagnose a specific issue, then disable
it immediately.

### 4. Add a trunk section per peer

```ini
[TRUNK_AB]
HOST=reflector-b.example.com
PORT=5302             # trunk port (default 5302, separate from client port 5300)
SECRET=shared_secret
REMOTE_PREFIX=2       # peer owns TGs 2, 20, 200, 2000, ...
# REMOTE_PREFIX=11,12,13  # comma-separated list accepted here too
```

Repeat for each peer (`[TRUNK_AC]`, `[TRUNK_AD]`, …).

**Important:** The `[TRUNK_x]` section name must be **identical on both sides** of
the link. Both sysops must agree on a shared section name before configuring the
link. For example, if reflectors A and B want to link, both configs must use the
same name (e.g. `[TRUNK_AB]`). A connection from a peer whose section name does
not match any local section will be rejected.


### Full-mesh example — three reflectors

**Reflector A** — owns prefix "1":
```ini
[GLOBAL]
LOCAL_PREFIX=1

[TRUNK_AB]
HOST=reflector-b.example.com
SECRET=secret_ab
REMOTE_PREFIX=2

[TRUNK_AC]
HOST=reflector-c.example.com
SECRET=secret_ac
REMOTE_PREFIX=3
```

**Reflector B** — owns prefix "2":
```ini
[GLOBAL]
LOCAL_PREFIX=2

[TRUNK_AB]
HOST=reflector-a.example.com
SECRET=secret_ab
REMOTE_PREFIX=1

[TRUNK_BC]
HOST=reflector-c.example.com
SECRET=secret_bc
REMOTE_PREFIX=3
```

**Reflector C** — owns prefix "3":
```ini
[GLOBAL]
LOCAL_PREFIX=3

[TRUNK_AC]
HOST=reflector-a.example.com
SECRET=secret_ac
REMOTE_PREFIX=1

[TRUNK_BC]
HOST=reflector-b.example.com
SECRET=secret_bc
REMOTE_PREFIX=2
```

Both sides of each trunk link must use the **same section name** and share the
same `SECRET`. The sysops of both reflectors agree on the section name and
secret when setting up the link.

### Satellite reflectors

A satellite is a lightweight reflector that connects to a parent reflector
instead of joining the trunk mesh. Clients connect to the satellite normally;
it relays everything to the parent. The parent is unchanged — it still
participates in the trunk mesh as before.

```
          ┌──── Full mesh (unchanged) ────┐
          │                               │
    ┌─────┴─────┐                   ┌─────┴─────┐
    │ Refl. 01  │◄── trunk ──────►│ Refl. 02  │
    └─────┬─────┘                   └───────────┘
     ┌────┼────┐
     ▼    ▼    ▼
   Sat A Sat B Sat C     ← satellites of Reflector 01
```

**Parent side** — add a `[SATELLITE]` section to accept inbound satellites:

```ini
[SATELLITE]
LISTEN_PORT=5303
SECRET=regional_satellite_secret
```

**Satellite side** — set `SATELLITE_OF` in `[GLOBAL]` instead of
`LOCAL_PREFIX` and `[TRUNK_x]` sections:

```ini
[GLOBAL]
SATELLITE_OF=reflector-01.example.com
SATELLITE_PORT=5303
SATELLITE_SECRET=regional_satellite_secret
SATELLITE_ID=my-satellite
```

A satellite does not set `LOCAL_PREFIX`, `REMOTE_PREFIX`, `CLUSTER_TGS`, or any
`[TRUNK_x]` sections. It inherits its parent's identity. Remote reflectors in
the mesh see satellite clients as if they were connected directly to the parent.

**No TG filtering:** Unlike trunk links (which filter by prefix and cluster TG),
satellite links forward **all** audio and talker signaling in both directions
without any TG filtering. The satellite is a transparent relay — every TG active
on the parent is heard on the satellite and vice versa, regardless of
`CLUSTER_TGS` or prefix configuration.

Port `5303` is the default satellite port (separate from client port `5300` and
trunk port `5302`).
