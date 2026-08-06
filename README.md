# Peer — P2P Messaging Between Aura Friday Instances

**Direct, encrypted, hole-punched communication between your machines — no middleman.**

[![License](https://img.shields.io/badge/license-Proprietary-blue.svg)](LICENSE)
[![Python](https://img.shields.io/badge/python-3.9%2B-blue.svg)](https://www.python.org/)
[![Platform](https://img.shields.io/badge/platform-Windows%20%7C%20Linux%20%7C%20macOS-lightgrey.svg)](https://github.com/AuraFriday/mcp-link-server)

---

## What This Means For You

### 1. Direct Machine-to-Machine Communication

The `peer` tool is the **data plane** underlying Den, Tunnel, and cross-device capabilities. It binds an iroh QUIC endpoint on your machine, giving it a stable cryptographic identity (64-hex EndpointId) that persists across restarts. Other Aura Friday instances connect to you by your ticket — a single string carrying your identity, relay hint, and direct addresses.

### 2. Hole-Punched When Possible, Relayed When Not

iroh's QUIC transport punches through most NATs automatically. When direct connectivity exists (same LAN, or cooperative NATs), traffic flows peer-to-peer with no relay in the path. When NAT traversal fails, traffic routes through our self-hosted relays (`relay-us.aurafriday.com` and `relay-eu.aurafriday.com`) — still encrypted end-to-end, with the relay unable to read message contents.

### 3. The Foundation for Den and Tunnel

While you CAN use `peer` directly for raw JSON messaging (useful for custom protocols, debugging, and inter-agent coordination), its primary role is providing the iroh endpoint that `den` (tool sharing) and `tunnel` (account mesh) build on top of.

---

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│ This Machine                                                    │
│                                                                 │
│  ┌──────────────┐                                               │
│  │ peer tool    │ ← MCP tool handlers (start/connect/send/recv) │
│  └──────┬───────┘                                               │
│         │ posts coroutines to the iroh loop                     │
│  ┌──────▼───────────────────────────┐                           │
│  │ Background asyncio loop (daemon) │                           │
│  │  ┌─────────────────────────┐     │                           │
│  │  │    iroh Endpoint        │     │                           │
│  │  │  • ALPN: af/mcp/1       │     │ ← mesh messaging          │
│  │  │  • ALPN: af/mcp-session │     │ ← den tool-sharing        │
│  │  │  • ALPN: af/den-coord   │     │ ← coordinator wake        │
│  │  └─────────────────────────┘     │                           │
│  └──────────────────────────────────┘                           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                    │ QUIC (UDP)
                    ▼
    ┌─────────────────────────────┐
    │  Direct (hole-punched)      │  ← preferred (LAN or cooperative NAT)
    │  — or —                     │
    │  Relayed (relay-us/eu)      │  ← fallback (restrictive NAT)
    └─────────────────────────────┘
```

One background daemon thread owns a private asyncio event loop; the iroh Endpoint lives on that loop. Tool operations post coroutines to it with `run_coroutine_threadsafe` and wait for the result. Received messages land in a bounded thread-safe inbox.

---

## Operations

| Operation | Purpose |
|-----------|---------|
| `readme` | Documentation + unlock token |
| `start` | Bind the iroh endpoint (stable identity persisted to key file). Returns EndpointId + ticket. |
| `status` | EndpointId, ticket, relay state, per-peer DIRECT/RELAY path info (RTT, tx/rx bytes) |
| `connect` | Dial a peer using its ticket string. Returns the peer_id once connected. |
| `send` | Deliver a JSON message to one peer (by EndpointId) or `"all"` |
| `recv` | Collect queued incoming messages (optionally block up to `wait_seconds`) |
| `disconnect` | Drop one peer or all peers |
| `stop` | Shut the entire endpoint down |

---

## Typical Session

```
1. Start:
   {"input": {"operation": "start", "tool_unlock_token": "..."}}
   → Returns: endpoint_id, ticket, relay_ready

2. Share your ticket with a peer (or use theirs to connect):
   {"input": {"operation": "connect", "ticket": "<peer's ticket>", "tool_unlock_token": "..."}}
   → Returns: peer_id, paths (DIRECT or RELAY)

3. Send a message:
   {"input": {"operation": "send", "peer_id": "all", "message": {"hello": "world"}, "tool_unlock_token": "..."}}

4. Receive messages:
   {"input": {"operation": "recv", "wait_seconds": 10, "tool_unlock_token": "..."}}
   → Returns: messages with from_peer_id and timestamp
```

---

## Relay Configuration

| Choice | Behavior |
|--------|----------|
| `tunnel.af` / `default` | Both production relays (US + EU); iroh picks the nearest |
| `legacy` | The old single relay.tunnel.af (vmO11) |
| `n0` | iroh's public relays (for A/B testing) |
| `disabled` | Direct/LAN only (no relay fallback) |
| Any `https://...` URL | Custom relay |

The default uses **two production relays** (`relay-us.aurafriday.com` in San Jose, `relay-eu.aurafriday.com` in Cork). iroh selects the nearest as its home relay and fails over automatically.

---

## Security

- **Stable Cryptographic Identity**: a 32-byte secret key (persisted at `<user_data>/peer_iroh_identity.key`, mode 0600) gives this endpoint a stable EndpointId. Allowlist it once on the relay for cross-NAT access.
- **Default-Deny Mesh Inbound**: unsolicited mesh connections from unknown peers are dropped before reading any application bytes (doc 77 ruling). Only peers you've dialed (or pinned coordinator identities) are accepted.
- **ALPN Routing**: multiple protocols share one endpoint. Connections are routed by their negotiated ALPN: `af/mcp/1` for mesh messaging, `af/mcp-session/1` for den tool-sharing sessions. An unknown ALPN is refused.
- **Relay Paywall**: the production relays require each EndpointId to be allowlisted (deployed via `tunnel.af/relay-access/allowlist.map`). Without it, direct/LAN dials via ticket still work but cross-NAT will not.

---

## Key Facts

- `start` is **idempotent**: if already started, it returns current status.
- The inbox holds the newest **1000 messages**; `recv` drains it.
- Requires the `iroh` package (bundled from v1.2.87; otherwise `pip install iroh==1.0.0`).
- **WSL1 is NOT supported** (broken netlink emulation) — run on native Windows instead.
- Messages are framed as **4-byte big-endian length + UTF-8 JSON** (max 1 MiB per message).
- Incoming connections are accepted automatically once started; mesh connections land in the inbox, den-ALPN connections are routed to the `den` tool.

---

## Relationship to Other Tools

| Tool | Role |
|------|------|
| **peer** | Owns the iroh endpoint; provides raw JSON messaging and the transport layer |
| **den** | Builds on peer's endpoint to share MCP tools between machines |
| **tunnel** | Builds on peer's endpoint to connect this device to an Aura Friday account |

The `peer` tool ALWAYS owns the single iroh endpoint. Den and tunnel never bind their own — they drive peer's endpoint via internal APIs.

---

> 🧠 **For AI agents:** You rarely need to call `peer` directly. The `den` tool (for cross-device tool access) and `tunnel` tool (for account enrollment) handle the peer endpoint for you. Use `peer` when you need raw P2P messaging, debugging connectivity, or checking DIRECT vs RELAY path status.
