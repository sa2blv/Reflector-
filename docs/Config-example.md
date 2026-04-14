# SVXReflector Configuration (with explanations)

This file describes a configuration for **SVXReflector** including trunking, MQTT, and certificate handling.

---

## GLOBAL

```ini
[GLOBAL]

# Directory for additional configuration files
#CFG_DIR=svxreflector.d

# Timestamp format used in logs
TIMESTAMP_FORMAT="%c"

# Main listening port for the reflector
LISTEN_PORT=5300

# Database timeout (seconds)
#SQL_TIMEOUT=600
#SQL_TIMEOUT_BLOCKTIME=60

# Audio codec (OPUS recommended)
#CODECS=OPUS

# Default talkgroup for legacy (v1) clients
TG_FOR_V1_CLIENTS=999

# Random QSY range
#RANDOM_QSY_RANGE=12399:100

# Built-in HTTP server (status/monitoring)
HTTP_SRV_PORT=8085

# PTY device for control scripts
COMMAND_PTY=/dev/shm/reflector_ctrl

# Callsign validation (regex)
#ACCEPT_CALLSIGN="[A-Z0-9][A-Z]{0,2}\\d[A-Z0-9]{0,3}[A-Z](?:-[A-Z0-9]{1,3})?"
#REJECT_CALLSIGN=""

# Allowed email format for certificates
#ACCEPT_CERT_EMAIL="\\w+(?:[-._+]\\w+)*@\\w+(?:\\.\\w+)*"

# PKI (certificate infrastructure)
CERT_PKI_DIR=/home/fura/pki/
CERT_CA_BUNDLE=ca-bundle.crt
CERT_CA_KEYS_DIR=private/
CERT_CA_PENDING_CSRS_DIR=pending_csrs/
CERT_CA_CSRS_DIR=csrs/
CERT_CA_CERTS_DIR=certs/

# Script triggered on certificate events
CERT_CA_HOOK=/usr/share/svxlink/ca-hook.py

# Local network prefix (country/region identifier)
LOCAL_PREFIX=240

# Port used for trunk connections
TRUNK_LISTEN_PORT=5302

# Enable trunk debugging (disabled by default)
#TRUNK_DEBUG=1

# Cluster talkgroups (empty = not used)
CLUSTER_TGS=

# Remote configuration via API
REMOTE_CONFIG_URL=https://example.api/api.ph
REMOTE_CONFIG_KEY=API_KEY
REMOTE_CONFIG_ID=NODE_ID
```

---

## MQTT

```ini
[MQTT]

# MQTT broker (used for events, status, etc.)
Server=MQTTserver.example

# Authentication
Username=Username
Password=PASSWORD
```

---

## Reflector Trunk

```ini
[ReflectorTrunk]

# Unique gateway identifier
GatewayId=ID

# Port for trunk traffic
Port=5500

# Defined trunk peers
Peers=Trunk1,Trunk2
```

---

## Trunk Peer 1

```ini
[TrunkPeer#Trunk1]

# Peer IP address or hostname
Host=5.6.7.8

# Port
Port=5500

# Allowed talkgroups (regex, here: allow all)
TGRule=^.*$

# Retransmit traffic
Retransmit=1

# Filtering disabled (0 = off)
ActiveFilter=0

# Optional talkgroup mapping file
#TgMapFile=/etc/svxlink/tg_mapping.csv

# Encryption key
Key=aes-key

# Trunk type (A or B)
TrunkType=A

# What traffic to send to peer
TrunkTypeSend=B

# Enable peer qualification/validation
Qualify=1
```

---

## Trunk Peer 2

```ini
[TrunkPeer#Trunk2]

Host=1.2.3.4
Port=5500
TGRule=^.*$
Retransmit=1
ActiveFilter=0
#TgMapFile=/etc/svxlink/tg_mapping.csv

Key=aes-key

# This peer uses trunk type B
TrunkType=B

# Sends both A and B traffic
TrunkTypeSend=A,B

Qualify=1
```

---

## GEU Trunk (AAA/SE link)

```ini
[TRUNK_AA_SE]

# Remote server address
HOST=remote.server.example

# Port
PORT=5302

# Shared secret (MUST be changed in production)
SECRET=change_me

# Remote network prefix
REMOTE_PREFIX=240

# Peer identifier
PEER_ID=id

# Blocked talkgroups
BLACKLIST_TGS=1,2,3,4,5,6,7,8,9,99

# Trunk type
TrunkType=A

# Traffic sent to peer
TrunkTypeSend=B
```

---

---
