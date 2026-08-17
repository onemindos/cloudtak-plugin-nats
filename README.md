# omos-nats

**OneMind OS — NATS fabric operator console for CloudTAK**

A CloudTAK plugin that brings the full NATS operations suite directly into the map UI.
This is the Vue 3 / CloudTAK-native port of the NATS pages from `omos-ui`.

## What's inside

| Tab | Description | Status |
|-----|-------------|--------|
| **Wire** | Live NATS message tap — subscribe to any subject pattern, real-time frame stream with badge classification, pause/freeze, JSON pretty-print | ✅ Built |
| **Security** | `$SYS` advisory audit log — auth errors, connect/disconnect events | ✅ Built |
| **Settings** | Multi-profile NATS WebSocket connection manager | ✅ Built |
| Topology | ForceGraph2D cluster visualization | 🔄 Planned |
| Connections | `$SYS.REQ.SERVER.*.CONNZ` connection table | 🔄 Planned |
| Streams | JetStream browser | 🔄 Planned |
| Key-Value | KV bucket browser | 🔄 Planned |

Also includes:
- **Bottom bar widget** — live NATS connection status dot + label
- **Shared Pinia store** (`useNatsStore`) — single WebSocket connection shared across all tabs

## Architecture

```
index.ts              ← Plugin entry (install/enable/disable)
lib/
  stores/
    nats.store.ts     ← Shared NATS connection (nats.ws, multi-profile)
  composables/
    useNatsTap.ts     ← Reactive subscription tap composable
  NatsConsole/
    NatsConsole.vue   ← Tabbed container (7 tabs)
    WireTab.vue       ← Live tap UI
    WireFrame.vue     ← Individual frame row (expand to see payload)
    wire.types.ts     ← Badge classification (HB/PROMPT/CHUNK/TEL/CMD/EVT/DET/ENT/SVC/SYS)
    SecurityTab.vue   ← $SYS audit log
    NatsSettings.vue  ← NATS connection profile manager
  MenuTemplate.vue    ← CloudTAK panel shell
  NatsBottomBar.vue   ← Bottom bar status widget
  IconNats.vue        ← Plugin SVG icon
```

## Subject badge classification

| Badge | Subject prefix | Color |
|-------|---------------|-------|
| `HB` | `agents.hb.*` | cyan |
| `PROMPT` | `agents.prompt.*` | indigo |
| `CHUNK` | `_INBOX.*` | green |
| `TEL` | `tel.*` | amber |
| `CMD` | `cmd.*` | orange |
| `EVT` | `evt.*` | purple |
| `DET` | `det.*` | red |
| `ENT` | `ent.*` | emerald |
| `SVC` | `svc.*` | blue |
| `SYS` | `$SYS.*`, `$JS.*` | slate |
| `MSG` | everything else | gray |

## Deploy

Add to `WEB_PLUGINS` in the OMOS fork Dockerfile alongside `omos-ai`:

```dockerfile
ARG WEB_PLUGINS="https://github.com/onemindos/omos-ai.git,https://github.com/onemindos/omos-nats.git"
```

## Replaces (from omos-ui)

- `/bus` — topology, connections, streams, KV, publish
- `/wire` — live NATS tap
- `/security` — $SYS audit log
- `/settings` (connection section) — NATS profile manager
