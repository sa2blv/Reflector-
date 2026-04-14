# ReflectorTrunkManager - Documentation

**Author:** Peter Lundberg / SA2BLV  
**Date:** 2026-04-11  
**Version:** 1.0  
**License:** GNU General Public License v2.0+

---

## Overview
This system implements a remote configuration pipeline where a client periodically fetches a full JSON configuration snapshot and applies only incremental changes using a diff-based approach.

---

## Architecture

- **ExternalConfigFetcher**
  - Fetches JSON config via HTTP (libcurl)
  - Runs in a background thread
  - Sends updates via callback

- **Reflector**
  - Processes incoming config
  - Applies only changes (diff-based)
  - Executes transient commands
  - Persists stable config locally

---

## Expected JSON Structure

```json
{
  "USERS": {
    "SM0ABC": "password123",
    "SA2XYZ": "password456"
  },
  "TALKGROUPS": {
    "2400": "enabled",
    "2401": "enabled"
  },
  "SETTINGS": {
    "timeout": "30",
    "mode": "auto"
  },
  "PTY": [
    "COMMAND1",
    "COMMAND2"
  ]
}
```

---

## Section Types

### Persistent Sections
- JSON objects
- Used for diff comparison and persistence

Examples:
- USERS
- TALKGROUPS
- SETTINGS
- PTY

---

### PTY (Ephemeral Commands)
- JSON array of strings
- Executed immediately
- Not stored
- Not diffed

---

## Initialization

```cpp
ExternaConfigFetcher::initialize(url, api_key, node_id, 30);

ExternaConfigFetcher::instance()->setCallback(
    [this](const Json::Value& config) {
        this->Process_New_Config_update(config);
    }
);

ExternaConfigFetcher::instance()->start();
```

Load previous config:

```cpp
std::ifstream file("config.json");
Json::parseFromStream(readerBuilder, file, &config, &errs);
Process_New_Config_update(config);
```

---

## Diff-Based Update Logic

### Modified Values
Only update when changed:

```cpp
if (newSection[key] != oldSection[key])
```

---

### Removed Keys

- USERS → set to "Removed"
- TALKGROUPS → set to ""

---

### Added Keys

Detected when key exists in new but not old config.

---

## Avoiding Unnecessary Updates

Core logic:

```cpp
persistentConfig.removeMember("PTY");

if (persistentConfig == m_lastConfig)
{
    return;
}
```

### Effect
- Prevents redundant writes
- Avoids unnecessary processing
- Ignores transient PTY differences

---

## Persistence

Saved to `config.json`:

```json
{
  "USERS": { ... },
  "TALKGROUPS": { ... },
  "SETTINGS": { ... }
}
```

PTY is excluded.

---

## Threading Model

- Fetcher runs in background thread
- Callback executes in same thread
- Mutex protects shared config

---

## Design Principles

- Full snapshot from server
- Local diff computation
- Idempotent updates
- Separation of transient vs persistent data

---

## Summary

The system ensures efficient configuration handling by:
- Applying only real changes
- Ignoring transient commands in persistence
- Preventing unnecessary updates

This results in stable, efficient, and predictable runtime behavior.
