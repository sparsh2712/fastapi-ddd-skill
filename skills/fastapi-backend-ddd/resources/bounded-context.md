# Bounded Context & Ubiquitous Language

---

## Bounded Context

A Bounded Context is an **explicit boundary within which a domain model is consistent and valid**. The same term can mean different things in different bounded contexts — and that is fine, as long as each context is explicit about its own model.

Example: "Customer" across contexts:
- **Sales context**: a prospect with a credit limit and account manager
- **Shipping context**: a delivery address and contact name
- **Support context**: a ticket history and SLA tier

These are different models. They share a name but must not share a class. Each context defines its own version.

In practice, each bounded context maps to a module with its own:
- `domain/` — its own Entities, Value Objects, Aggregates
- `service/` — its own use cases
- `infrastructure/repositories/` — its own repositories
- Optionally its own database schema

---

## Ubiquitous Language

Ubiquitous Language is the **shared vocabulary between domain experts and developers**, used consistently in conversation, documentation, and code. It is not a separate glossary — it IS the code.

**The test**: Can a domain expert read your class and method names and understand what the system does without reading the method bodies?

```python
# Bad — technical language, no domain meaning
class DataProcessor:
    def process_entity(self, entity_id: str, data: dict): ...
    def update_record(self, record_id: str, delta: dict): ...

# Good — domain language, self-describing
class Batch:
    def allocate(self, line: OrderLine) -> None: ...
    def deallocate(self, line: OrderLine) -> None: ...
    def can_allocate(self, line: OrderLine) -> bool: ...
```

**Naming conventions:**
- Class names: nouns from the domain (`Batch`, `Order`, `Shipment`, `Invoice`)
- Method names: verbs from the domain (`allocate`, `confirm`, `ship`, `cancel`)
- Parameter names: domain concepts (`order_line`, `sku`, `reference`) — not `data`, `payload`, `obj`
- Exceptions: the business rule that was violated (`OutOfStockError`, `OrderAlreadyConfirmedError`)
- Module/package names: the bounded context name (`ordering`, `shipping`, `billing`)
- File names: describe the use case group (`allocation.py`, `fulfillment.py`)

---

## No Cross-Context Model Sharing

```python
# Wrong — importing a domain class from another context
from domain.ordering.model import Batch  # inside the warehouse service

# Right — each context owns its own model
# domain/warehouse/model.py defines PickList, StorageLocation in warehouse vocabulary
# Each context defines its own representation of shared concepts
```

---

## Anti-Corruption Layer

When integrating with an external system or a legacy model whose vocabulary doesn't match yours, build a translation layer. Never let their model bleed into your domain.

```python
# infrastructure/acl/legacy_inventory_adapter.py
from domain.ordering.model import Batch
from datetime import datetime

class LegacyInventoryAdapter:
    def __init__(self, legacy_client) -> None:
        self._client = legacy_client

    async def get_batches_for_sku(self, sku: str) -> list[Batch]:
        raw = await self._client.query_stock(product_code=sku)
        return [self._to_domain(r) for r in raw["STOCK_RECORDS"]]

    def _to_domain(self, raw: dict) -> Batch:
        return Batch(
            reference=raw["STOCK_REF"],
            sku=raw["PRODUCT_CODE"],
            purchased_quantity=raw["QTY"],
            eta=datetime.fromisoformat(raw["ETA"]) if raw.get("ETA") else None,
        )
```

The domain never sees the external format.

---

## Signals That You Need a New Bounded Context

- The same term means different things in different domain expert conversations
- Different teams own distinct parts of the business
- A set of concepts has strong internal consistency but loose coupling to the rest of the system

## Signals That Contexts Are Leaking

- Domain models have many optional fields that only apply in certain "modes"
- A class has if-branches based on what role it's currently playing
- One context directly imports domain classes from another context

---

## Subdomain Classification

| Subdomain Type | Description | Investment |
|---|---|---|
| **Core Domain** | The unique capability that gives competitive advantage | Maximum — rich domain model, Aggregates, TDD |
| **Supporting Subdomain** | Necessary but not differentiating | Moderate — lighter DDD, some CRUD acceptable |
| **Generic Subdomain** | Commodity (auth, email, payments) | Minimal — use a library or buy a service |

Apply full DDD tactical patterns where they matter — the core domain. Do not apply them uniformly to every part of the system.
