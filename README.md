# omos-nats-plugin

**OneMind OS — NATS fabric operator console for CloudTAK**

A CloudTAK plugin that brings the full NATS operations suite directly into the map UI.
This is the Vue 3 / CloudTAK-native replacement for the omos-ui NATS, Wire, and Security pages.

---

## What's Inside

**Operator Console** (`home-menu → NATS`) — 8 tabs:

| Tab | What it does |
|---|---|
| **Wire** | Live NATS subject tap — subscribe to any subject pattern, see real-time frames with badge classification (HB, PROMPT, CHUNK, TEL, CMD, EVT, DET, ENT, SVC, SYS, MSG). Filter, pause, clear. Expand any frame for full payload with JSON pretty-print. |
| **Topology** | Server cards from nats-mon (`/varz` + `/routez`). Shows primary node, routes, connections, in/out msgs, leaf nodes. |
| **Connections** | `$SYS.REQ.SERVER.PING` CONNZ table — every client connection with name, IP, account, sub count, in/out msgs, data volume, lang, RTT. Click to expand and see all subscriptions. |
| **Streams** | JetStream browser — all streams with name, retention policy, msg count, bytes, consumer count. Expand to see subjects, config, and live consumer state. Purge with confirmation. |
| **Key-Value** | KV bucket browser — list all buckets, open any bucket to browse keys, view values (JSON pretty-print), set new keys, delete keys. |
| **Publish** | Publish or request-reply with full header support. Mode toggle, subject autocomplete, JSON payload editor with byte counter. |
| **Security** | `$SYS` audit log — auth errors, client connects, client disconnects. Color-coded event cards. |
| **Settings** | Multi-profile NATS connection manager — add, remove, edit, reorder profiles. Set active profile. Server URL, auth mode, credentials, JetStream enabled flag. |

**Bottom Bar** — live NATS connection status dot + RTT.

---

## Architecture

```
omos-nats-plugin/
├── index.ts                      ← Plugin entry (routes, menu, bottom bar)
├── package.json                  ← nats.ws is bundled as a dependency
├── lib/
│   ├── stores/
│   │   └── nats.store.ts         ← Shared Pinia store: connection, status, RTT, profiles
│   ├── composables/
│   │   └── useNatsTap.ts         ← Reactive subscription composable (Wire engine)
│   ├── NatsConsole/
│   │   ├── NatsConsole.vue       ← 8-tab container
│   │   ├── WireTab.vue           ← Live tap
│   │   ├── WireFrame.vue         ← Individual frame row (expandable)
│   │   ├── wire.types.ts         ← Badge classification
│   │   ├── TopologyTab.vue       ← Server topology cards
│   │   ├── ConnectionsTab.vue    ← Live client connections table
│   │   ├── StreamsTab.vue        ← JetStream streams + consumers + purge
│   │   ├── KVTab.vue             ← KV bucket browser + key/value editor
│   │   ├── PublishTab.vue        ← Publish + request-reply
│   │   ├── SecurityTab.vue       ← $SYS security audit log
│   │   └── NatsSettings.vue      ← Connection profiles manager
│   ├── MenuTemplate.vue          ← CloudTAK menu wrapper
│   ├── NatsBottomBar.vue         ← Status dot in bottom bar
│   └── IconNats.vue              ← Plugin icon
```

---

## Relationship to omos-ui

| omos-ui page | omos-nats-plugin equivalent |
|---|---|
| `/bus` | Topology + Connections + Streams + KV tabs |
| `/wire` | Wire tab |
| `/security` | Security tab |
| `/settings` | Settings tab |
| `/nats` | All of the above |

**This plugin replaces omos-ui's NATS operations surface entirely.** Once deployed, omos-ui has no operational advantage and can be archived.

---

## Deploying

Add to `WEB_PLUGINS` in the OMOS fork Dockerfile, alongside `omos-ai-plugin`:

```dockerfile
ARG WEB_PLUGINS="\
https://github.com/onemindos/omos-ai-plugin.git,\
https://github.com/onemindos/omos-nats-plugin.git"
```

On next CloudTAK build, the NATS console appears in the main menu under **NATS**.

---

## Developing

```bash
npm install nats.ws lucide-vue-next
# Edit src files — no build step needed; the plugin is compiled by CloudTAK's own Vite build
# Commit to main → WEB_PLUGINS pulls latest at next Docker build
```
