# Couchbase Eventing Patterns

## Pattern 1: Real-Time Data Enrichment

Enrich documents with data from other collections as they are created/updated:

```javascript
// Source:      orders collection
// Binding:     users (read-only → users collection)
//              products (read-only → products collection)
//              enriched_orders (read-write → enriched_orders collection)

function OnUpdate(doc, meta) {
    if (doc.type !== "order") return;
    if (doc.status !== "placed") return;

    // Fetch related data
    var user    = users["user::" + doc.userId];
    var product = products["product::" + doc.lineItems[0].productId];

    var enriched = {
        type:         "enriched_order",
        orderId:      meta.id,
        orderStatus:  doc.status,
        total:        doc.total,
        customerName: user  ? user.name  : "Unknown",
        customerTier: user  ? user.tier  : "standard",
        primaryItem:  product ? product.name : "Unknown",
        enrichedAt:   new Date().toISOString()
    };

    enriched_orders[meta.id] = enriched;
}

function OnDelete(meta, options) {
    delete enriched_orders[meta.id];
}
```

## Pattern 2: Change Notification Webhook

Fire HTTP webhooks on document mutations:

```javascript
// URL binding: "webhook" → https://api.example.com/events
//   Method: POST; Allowed headers: Content-Type

function OnUpdate(doc, meta) {
    // Only notify on status transitions
    if (doc.type !== "order") return;
    if (!doc.status) return;

    var payload = {
        event:    "order.updated",
        orderId:  meta.id,
        status:   doc.status,
        userId:   doc.userId,
        total:    doc.total,
        ts:       new Date().toISOString()
    };

    var request = {
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify(payload)
    };

    try {
        var response = curl("POST", webhook, request);
        if (response.status < 200 || response.status >= 300) {
            log("Webhook error", meta.id, response.status, response.body);
        }
    } catch (e) {
        log("Webhook exception", meta.id, e.message);
    }
}
```

## Pattern 3: Aggregation / Counter Rollup

Maintain counters or aggregates in a summary document:

```javascript
// Source:      events collection
// Binding:     counters (read-write → counters collection)

function OnUpdate(doc, meta) {
    if (doc.type !== "page_view") return;

    var key = "daily::" + doc.date + "::" + doc.page;

    // Use couchbase.mutateIn for atomic increments
    try {
        couchbase.mutateIn("counters", key, [
            couchbase.MutateInSpec.increment("views", 1, { createPath: true }),
            couchbase.MutateInSpec.increment("uniqueUsers", doc.isNewSession ? 1 : 0, { createPath: true }),
            couchbase.MutateInSpec.upsert("lastViewedAt", new Date().toISOString())
        ]);
    } catch (e) {
        // Doc doesn't exist yet — insert it
        if (e.message && e.message.includes("KEY_ENOENT")) {
            counters[key] = {
                type: "daily_counter",
                date: doc.date,
                page: doc.page,
                views: 1,
                uniqueUsers: doc.isNewSession ? 1 : 0,
                lastViewedAt: new Date().toISOString()
            };
        }
    }
}
```

## Pattern 4: Data Archival / TTL Cascade

Move documents to an archive collection before expiry:

```javascript
// Source:      sessions collection (documents have TTL)
// Binding:     archive (read-write → session_archive collection)

function OnDelete(meta, options) {
    // Only archive docs that expired (not explicitly deleted)
    if (!options.is_expiry) return;

    // Note: document is already deleted — only meta is available
    // Store a tombstone in the archive
    archive["archived::" + meta.id] = {
        type:      "archived_session",
        sessionId: meta.id,
        archivedAt: new Date().toISOString(),
        reason:    "expired"
    };
}
```

For archiving with the full document, process before expiry using timers (see Pattern 6).

## Pattern 5: Cross-Collection Sync

Mirror documents from one collection to another (e.g., replication to a read-replica or denormalized view):

```javascript
// Source:      products collection
// Binding:     search_ready (read-write → products_search collection)

function OnUpdate(doc, meta) {
    if (doc.type !== "product") return;
    if (doc.status === "draft") return;  // don't sync drafts

    // Create a flattened document optimized for search indexing
    search_ready[meta.id] = {
        type:        "product_search",
        id:          meta.id,
        name:        doc.name,
        description: doc.description,
        brand:       doc.brand,
        category:    doc.category,
        price:       doc.price,
        tags:        doc.tags || [],
        searchText:  [doc.name, doc.description, doc.brand, doc.category].join(" "),
        updatedAt:   new Date().toISOString()
    };
}

function OnDelete(meta, options) {
    delete search_ready[meta.id];
}
```

## Pattern 6: Scheduled Reminder (Timer)

Schedule per-document future actions:

```javascript
// Source:      subscriptions collection
// Binding:     notifications (read-write → notifications collection)

function OnUpdate(doc, meta) {
    if (doc.type !== "subscription") return;

    // Cancel any existing timer for this subscription
    cancelTimer(meta.id + "_renewal_reminder");

    if (doc.status !== "active" || !doc.renewsAt) return;

    // Schedule reminder 7 days before renewal
    var renewDate   = new Date(doc.renewsAt);
    var reminderAt  = new Date(renewDate.getTime() - 7 * 24 * 60 * 60 * 1000);

    if (reminderAt > new Date()) {
        createTimer(
            sendRenewalReminder,
            reminderAt,
            meta.id + "_renewal_reminder",
            {
                userId: doc.userId,
                plan:   doc.plan,
                renewsAt: doc.renewsAt
            }
        );
    }
}

function sendRenewalReminder(context) {
    var notifKey = "notif::" + context.userId + "::" + Date.now();
    notifications[notifKey] = {
        type:    "notification",
        userId:  context.userId,
        channel: "email",
        template: "subscription_renewal_reminder",
        data: { plan: context.plan, renewsAt: context.renewsAt },
        status:  "pending",
        createdAt: new Date().toISOString()
    };
}
```

## Pattern 7: Audit Log

Create an immutable audit trail for document changes:

```javascript
// Source:      any collection (e.g., user_settings)
// Binding:     audit_log (read-write → audit_log collection)

function OnUpdate(doc, meta) {
    // Skip audit documents themselves
    if (doc.type === "audit_entry") return;

    var auditKey = "audit::" + meta.id + "::" + new Date().getTime();
    audit_log[auditKey] = {
        type:       "audit_entry",
        targetId:   meta.id,
        targetType: doc.type || "unknown",
        action:     "updated",
        snapshot:   doc,  // full document snapshot
        timestamp:  new Date().toISOString()
    };
}

function OnDelete(meta, options) {
    var auditKey = "audit::" + meta.id + "::" + new Date().getTime();
    audit_log[auditKey] = {
        type:      "audit_entry",
        targetId:  meta.id,
        action:    options.is_expiry ? "expired" : "deleted",
        timestamp: new Date().toISOString()
    };
}
```

## Eventing JavaScript Built-ins

```javascript
// Logging (visible in Eventing logs in UI)
log("message", variable1, variable2);

// Timer management
createTimer(callbackFn, dateObject, timerId, context);
cancelTimer(timerId);

// HTTP (requires URL binding named "myUrl")
var response = curl("GET",  myUrl, { headers: {} });
var response = curl("POST", myUrl, { headers: { "Content-Type": "application/json" }, body: JSON.stringify(data) });
// response.status, response.headers, response.body (string)

// Sub-document mutations on bucket bindings
couchbase.mutateIn(bindingRef, docKey, [
    couchbase.MutateInSpec.upsert("field", value),
    couchbase.MutateInSpec.increment("counter", delta, { createPath: true }),
    couchbase.MutateInSpec.arrayAppend("list", item, { createPath: true }),
    couchbase.MutateInSpec.remove("field"),
]);

// Read sub-document
var result = couchbase.lookupIn(bindingRef, docKey, [
    couchbase.LookupInSpec.get("field"),
    couchbase.LookupInSpec.exists("field"),
]);

// N1QL (use sparingly — has overhead; prefer KV via bindings)
var result = N1QL("SELECT * FROM `bucket` WHERE userId = $uid", {"uid": userId});
```

## Deployment Tips

- Test with **Feed boundary = From now** first; use **From beginning** to replay history
- Use `log()` liberally during development; check logs at Eventing → your function → Log
- **Pause** (not Undeploy) when making changes to preserve cursor position
- Each function has its own processing thread — multiple functions can run in parallel
- Avoid heavy CPU work (large loops, synchronous HTTP calls in tight loops) — this blocks the processing thread
- Write errors are NOT automatically retried; implement your own error handling or dead-letter queue
