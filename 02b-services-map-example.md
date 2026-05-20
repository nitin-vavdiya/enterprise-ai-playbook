# SERVICES.md — worked example

> A fully filled-in example using a fictional "Acme" e-commerce platform. Use this to see what a healthy `SERVICES.md` looks like at ~6 services. The blank scaffold lives in `02a-services-map-template.md`. Do not copy this file directly into your own `platform-docs` — start from the template.

---

# Acme Platform — Services Map

> Canonical source: `github.com/acme/platform-docs`. Do not edit copies in service repos.
> Last updated: 2026-05-18 · Owner: Platform team (#platform-chat on Slack)

## How to read this file

Every service has the same sections:

- **Purpose** — one sentence, what it owns
- **Repo** — where the code lives
- **Owns** — domain entities/data this service is the source of truth for
- **Sync calls out** — services this one calls (HTTP/gRPC)
- **Async publishes** — events/messages this service emits
- **Consumes** — events/messages this service subscribes to
- **Called by** — known upstream consumers (sync)
- **Contracts** — paths to API specs / schemas

If you are an AI agent: **before making any change that touches an API, an event payload, or a shared entity, read this file end-to-end and identify every service in the blast radius.**

---

## System overview

```
                  ┌─────────────┐
                  │   web-app   │
                  └──────┬──────┘
                         │ HTTP
                  ┌──────▼──────┐
                  │ api-gateway │
                  └──────┬──────┘
              ┌──────────┼──────────┐
              │ HTTP     │ HTTP     │ HTTP
       ┌──────▼─────┐ ┌──▼────────┐ ┌─▼──────────┐
       │ order-svc  │ │ user-svc  │ │ catalog-svc│
       └──────┬─────┘ └───────────┘ └────────────┘
              │ gRPC
       ┌──────▼─────┐
       │ payment-svc│
       └────────────┘

   Kafka events flow between order-svc, billing-svc, notification-svc, analytics-svc.
   MQTT used by warehouse-svc to talk to physical scanners.
```

---

## Services

### order-service

- **Purpose:** Order lifecycle — create, cancel, fulfill. Owns Order and LineItem.
- **Repo:** `github.com/acme/order-service`
- **Owns:** `Order`, `LineItem`, `OrderStatus`
- **Sync calls out:**
  - `payment-service` (gRPC) — to authorize and capture payments
  - `catalog-service` (HTTP/REST) — to validate SKUs and pricing
  - `user-service` (HTTP/REST) — to verify customer
- **Async publishes (Kafka):**
  - `order.created` (v2, Avro) — emitted on successful order placement
  - `order.cancelled` (v1, Avro) — emitted on cancellation
  - `order.fulfilled` (v1, Avro) — emitted when warehouse confirms shipment
- **Consumes (Kafka):**
  - `payment.captured` — to advance order to PAID state
  - `warehouse.shipment.dispatched` — to advance order to FULFILLED
- **Called by:** `api-gateway`, `admin-console`, `mobile-bff`
- **Contracts:**
  - REST API: `contracts/order-service/openapi.yaml`
  - Event schemas: `contracts/events/order.*.avsc`

---

### payment-service

- **Purpose:** Process payments via Stripe and internal wallet. Owns Payment.
- **Repo:** `github.com/acme/payment-service`
- **Owns:** `Payment`, `PaymentMethod`, `Refund`
- **Sync calls out:**
  - Stripe API (external HTTPS)
  - `user-service` (HTTP/REST) — to fetch billing address
- **Async publishes (Kafka):**
  - `payment.captured` (v1, Avro)
  - `payment.failed` (v1, Avro)
  - `payment.refunded` (v1, Avro)
- **Consumes (Kafka):** none
- **Called by:** `order-service` (gRPC, sole sync caller)
- **Contracts:**
  - gRPC: `contracts/payment-service/payment.proto`
  - Event schemas: `contracts/events/payment.*.avsc`
- **⚠️ Notes:** PCI scope. Do not log card numbers. Schema changes require security review.

---

### user-service

- **Purpose:** User accounts, authentication, profile. Owns User.
- **Repo:** `github.com/acme/user-service`
- **Owns:** `User`, `Address`, `AuthToken`
- **Sync calls out:** none (leaf service)
- **Async publishes (Kafka):**
  - `user.created` (v1, Avro)
  - `user.updated` (v2, Avro)
  - `user.deleted` (v1, Avro)
- **Consumes (Kafka):** none
- **Called by:** `order-service`, `payment-service`, `notification-service`, `api-gateway`
- **Contracts:**
  - REST API: `contracts/user-service/openapi.yaml`
  - Event schemas: `contracts/events/user.*.avsc`

---

### catalog-service

- **Purpose:** Product catalog, pricing, inventory snapshots. Owns Product.
- **Repo:** `github.com/acme/catalog-service`
- **Owns:** `Product`, `SKU`, `Price`
- **Sync calls out:**
  - `warehouse-service` (HTTP/REST) — for inventory checks
- **Async publishes (Kafka):**
  - `product.created`, `product.updated`, `product.discontinued` (all v1, Avro)
- **Consumes:** none
- **Called by:** `order-service`, `api-gateway`, `search-indexer`
- **Contracts:**
  - REST API: `contracts/catalog-service/openapi.yaml`

---

### warehouse-service

- **Purpose:** Physical inventory, picking, shipping. Owns Inventory.
- **Repo:** `github.com/acme/warehouse-service`
- **Owns:** `InventoryItem`, `Shipment`, `PickList`
- **Sync calls out:** none
- **Async publishes (Kafka):**
  - `warehouse.shipment.dispatched` (v1, Avro)
  - `warehouse.inventory.low` (v1, Avro)
- **MQTT publishes:**
  - `warehouse/+/scanner/+/scan` — scan events from handheld devices
  - `warehouse/+/printer/+/status` — label printer health
- **MQTT subscribes:**
  - `warehouse/+/scanner/+/command` — commands sent to scanners
- **Consumes (Kafka):**
  - `order.created` — to add to pick queue
  - `order.cancelled` — to remove from pick queue
- **Called by:** `catalog-service` (HTTP), `admin-console` (HTTP)
- **Contracts:**
  - REST API: `contracts/warehouse-service/openapi.yaml`
  - MQTT topics: `contracts/warehouse-service/mqtt-topics.md`
  - Event schemas: `contracts/events/warehouse.*.avsc`
- **⚠️ Notes:** MQTT broker is Mosquitto on-prem. Scanners use QoS 1.

---

### notification-service

- **Purpose:** Send transactional email, SMS, push notifications.
- **Repo:** `github.com/acme/notification-service`
- **Owns:** `NotificationTemplate`, `DeliveryLog`
- **Sync calls out:**
  - SendGrid (external HTTPS), Twilio (external HTTPS)
  - `user-service` (HTTP/REST) — to fetch contact preferences
- **Async publishes:** none
- **Consumes (Kafka):**
  - `order.created`, `order.cancelled`, `order.fulfilled`
  - `payment.failed`
  - `user.created`
- **Called by:** none (pure consumer of events)
- **Contracts:** internal only — no public API

---

## Event topology (Kafka)

Who emits what, who consumes what. **Update this table when adding or changing any event.**

| Event | Producer | Consumers | Schema version |
|-------|----------|-----------|----------------|
| `user.created` | user-service | notification-service, analytics-svc | v1 |
| `user.updated` | user-service | analytics-svc | v2 |
| `user.deleted` | user-service | order-service*, analytics-svc | v1 |
| `order.created` | order-service | warehouse-svc, notification-svc, analytics-svc | v2 |
| `order.cancelled` | order-service | warehouse-svc, notification-svc, analytics-svc | v1 |
| `order.fulfilled` | order-service | notification-svc, analytics-svc | v1 |
| `payment.captured` | payment-service | order-service, analytics-svc | v1 |
| `payment.failed` | payment-service | order-service, notification-svc | v1 |
| `payment.refunded` | payment-service | analytics-svc | v1 |
| `warehouse.shipment.dispatched` | warehouse-svc | order-service, notification-svc | v1 |
| `warehouse.inventory.low` | warehouse-svc | catalog-service | v1 |

*order-service consumes `user.deleted` to anonymize historical orders.

---

## Change-impact cheatsheet

Before changing any of the following, check who's affected:

| If you change… | Re-check these services |
|----------------|--------------------------|
| `User` schema or `/users` REST API | order-service, payment-service, notification-service, api-gateway |
| `Order` schema or `/orders` REST API | api-gateway, mobile-bff, admin-console |
| `payment.proto` (gRPC) | order-service (sole caller), and bump version per gRPC rules |
| Any `order.*` event schema | warehouse-svc, notification-svc, analytics-svc |
| Any `user.*` event schema | notification-svc, analytics-svc, order-service |
| Any MQTT topic under `warehouse/` | warehouse-service code, scanner firmware, on-prem broker config |

---

## Conventions

- **REST APIs:** versioned via URL path (`/v1/...`). Breaking changes require new version, 6-month deprecation.
- **gRPC:** versioned per [official guidelines](https://protobuf.dev/programming-guides/proto3/#updating). Never reuse field numbers. Mark deprecated fields, don't delete.
- **Kafka events:** Avro schemas in the registry. Forward + backward compatible only. Breaking changes = new event name + version suffix.
- **MQTT topics:** structure is `<facility>/<area>/<device-type>/<device-id>/<action>`. Don't change topic structure without coordinating with on-prem ops.

---

## Updating this file

1. Edit in `platform-docs` repo, PR with reviewer from affected service team(s).
2. On merge, CI syncs to every service repo's root.
3. If you add a service: add a section here AND update the event topology table AND the change-impact cheatsheet.
4. If you delete a service: leave a tombstone entry (`### old-service-name [REMOVED 2026-04]`) for 6 months so the agent doesn't try to call it.
