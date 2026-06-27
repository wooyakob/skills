---
name: couchbase-eventing
description: Create and deploy Couchbase Eventing functions to react to document mutations in real time. Use when implementing event-driven workflows, data enrichment pipelines, change notifications, cross-collection data propagation, aggregation rollups, or scheduled/timer-based processing.
---

# Couchbase Eventing

The Eventing Service runs JavaScript functions that fire on document mutations (create, update, delete) or on timers. Functions run inside the Couchbase cluster — no external infrastructure needed.

## Core Concepts

| Concept | Description |
|---------|-------------|
| **Source bucket/collection** | Where mutations are observed |
| **Metadata bucket** | Internal state store (dedicate a separate bucket; do not use for app data) |
| **Binding** | Named reference to a bucket/collection accessible inside the function as a JS object |
| **Handler** | `OnUpdate(doc, meta)` or `OnDelete(meta, options)` |
| **Timer** | Schedule a callback at a future time |

## Function Structure

```javascript
// Eventing function (JavaScript)
// Bindings configured in the UI:
//   - "src"  → source collection (read-only alias)
//   - "dst"  → destination collection (read-write alias)

function OnUpdate(doc, meta) {
    // Fires on every create or update in the source collection
    // meta.id    = document key
    // meta.expiration = TTL in seconds
    
    // Guard: only process specific document types
    if (doc.type !== "order") return;
    if (doc.status !== "placed")  return;
    
    // Enrich and write to destination collection
    var enriched = {
        type:       "order_summary",
        orderId:    meta.id,
        userId:     doc.userId,
        total:      doc.total,
        itemCount:  doc.lineItems ? doc.lineItems.length : 0,
        processedAt: new Date().toISOString()
    };
    dst["summary::" + meta.id] = enriched;
    
    log("Processed order", meta.id);
}

function OnDelete(meta, options) {
    // Fires on document delete or expiry
    // options.is_expiry = true if triggered by TTL expiry
    delete dst["summary::" + meta.id];
}
```

## Configuring a Function

1. **Couchbase UI** → Eventing → Add Function
2. Set **Source Bucket/Scope/Collection** (where to listen)
3. Set **Metadata Bucket** (dedicated bucket, e.g., `eventing-meta`)
4. Add **Bindings**:
   - Bucket binding: give a JS alias name + choose the target collection + access level (read/read-write)
   - URL binding: for outbound HTTP calls (webhooks, REST APIs)
5. Paste your function code
6. **Deploy** → choose "Feed boundary": `From now` (new mutations only) or `From beginning` (replay all)

## Accessing Other Collections (Bindings)

```javascript
// Bucket binding "users" → bucket.scope.users collection (read-only)
// Bucket binding "dst"   → bucket.scope.processed collection (read-write)

function OnUpdate(doc, meta) {
    if (doc.type !== "order") return;
    
    // Read from another collection via binding
    var user = users["user::" + doc.userId];
    if (!user) {
        log("User not found:", doc.userId);
        return;
    }
    
    // Write enriched doc
    dst[meta.id] = {
        type:         "enriched_order",
        orderId:      meta.id,
        userName:     user.name,
        userEmail:    user.email,
        total:        doc.total,
        processedAt:  new Date().toISOString()
    };
}
```

## Timers — Schedule Future Work

```javascript
function OnUpdate(doc, meta) {
    if (doc.type !== "subscription" || !doc.renewsAt) return;
    
    // Schedule a reminder 3 days before renewal
    var renewDate = new Date(doc.renewsAt);
    var reminderDate = new Date(renewDate.getTime() - 3 * 24 * 60 * 60 * 1000);
    
    createTimer(sendRenewalReminder, reminderDate, meta.id, { userId: doc.userId });
}

function sendRenewalReminder(context) {
    // This fires at the scheduled time
    // context.userId is available from the payload
    var notification = {
        type:    "notification",
        userId:  context.userId,
        message: "Your subscription renews in 3 days",
        sentAt:  new Date().toISOString()
    };
    notifications["notif::" + context.userId + "::" + Date.now()] = notification;
}
```

## Outbound HTTP (URL Bindings)

```javascript
// URL binding "webhookEndpoint" → https://api.example.com/webhooks
// configured in the UI with method allow-list and optional auth headers

function OnUpdate(doc, meta) {
    if (doc.type !== "alert" || doc.severity !== "critical") return;
    
    var request = {
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({ alert: meta.id, message: doc.message })
    };
    
    var response = curl("POST", webhookEndpoint, request);
    if (response.status !== 200) {
        log("Webhook failed:", response.status, response.body);
    }
}
```

## Deployment Lifecycle

| Action | Description |
|--------|-------------|
| **Deploy** | Start processing mutations; choose feed boundary |
| **Pause** | Stop processing; resume later from where it left off |
| **Undeploy** | Stop and reset; next deploy starts from feed boundary again |
| **Delete** | Remove function entirely |

- Functions are **stateless** — all state goes in bucket bindings
- One function can write to another function's source (chaining)
- Check **Eventing Stats** in the UI for mutation rates, errors, and timer counts

For patterns (data enrichment, aggregation rollup, archival, audit log, cross-collection sync), see `references/eventing-patterns.md`.
