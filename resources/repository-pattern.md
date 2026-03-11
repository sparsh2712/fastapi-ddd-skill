# Repository Pattern

---

## Purpose

The Repository provides an **illusion of an in-memory collection** of domain objects. It hides all database access. The service layer treats it as a plain Python collection — no knowledge of SQL, connections, or transactions.

---

## Abstract Repository (the interface)

```python
# infrastructure/repositories/ordering/batch_repository.py
import abc
from domain.ordering.model import Batch

class AbstractBatchRepository(abc.ABC):
    def __init__(self) -> None:
        self.seen: set[Batch] = set()

    async def get(self, reference: str) -> Batch | None:
        batch = await self._get(reference)
        if batch:
            self.seen.add(batch)
        return batch

    async def get_by_sku(self, sku: str) -> list[Batch]:
        batches = await self._get_by_sku(sku)
        self.seen.update(batches)
        return batches

    def add(self, batch: Batch) -> None:
        self.seen.add(batch)
        self._add(batch)

    async def save(self, batch: Batch) -> None:
        self.seen.add(batch)
        await self._save(batch)

    @abc.abstractmethod
    async def _get(self, reference: str) -> Batch | None: ...

    @abc.abstractmethod
    async def _get_by_sku(self, sku: str) -> list[Batch]: ...

    @abc.abstractmethod
    def _add(self, batch: Batch) -> None: ...

    @abc.abstractmethod
    async def _save(self, batch: Batch) -> None: ...
```

The `seen` set tracks every aggregate the repository has touched during a unit of work. It enables the UoW to reach in and inspect aggregates after work completes.

---

## Concrete Implementation

Write SQL. Construct domain objects manually from rows. The `_to_domain` method is the explicit translation boundary between the database schema and the domain model.

```python
from sqlalchemy.ext.asyncio import AsyncConnection
from sqlalchemy import text
from domain.ordering.model import Batch, OrderLine

class PostgresBatchRepository(AbstractBatchRepository):
    def __init__(self, conn: AsyncConnection) -> None:
        super().__init__()
        self.conn = conn

    async def _get(self, reference: str) -> Batch | None:
        result = await self.conn.execute(
            text("""
                SELECT reference, sku, purchased_quantity, eta
                FROM ordering.batches
                WHERE reference = :reference
            """),
            {"reference": reference},
        )
        row = result.mappings().one_or_none()
        if row is None:
            return None
        allocations = await self._fetch_allocations(reference)
        return self._to_domain(row, allocations)

    async def _fetch_allocations(self, reference: str) -> set[OrderLine]:
        result = await self.conn.execute(
            text("""
                SELECT ol.order_id, ol.sku, ol.quantity
                FROM ordering.order_lines ol
                JOIN ordering.allocations a ON a.orderline_id = ol.id
                JOIN ordering.batches b ON b.id = a.batch_id
                WHERE b.reference = :reference
            """),
            {"reference": reference},
        )
        return {
            OrderLine(order_id=r["order_id"], sku=r["sku"], quantity=r["quantity"])
            for r in result.mappings()
        }

    def _to_domain(self, row, allocations: set[OrderLine]) -> Batch:
        batch = Batch(
            reference=row["reference"],
            sku=row["sku"],
            purchased_quantity=row["purchased_quantity"],
            eta=row["eta"],
        )
        batch._allocations = allocations
        return batch

    def _add(self, batch: Batch) -> None:
        self.conn.execute(
            text("""
                INSERT INTO ordering.batches (reference, sku, purchased_quantity, eta)
                VALUES (:reference, :sku, :purchased_quantity, :eta)
            """),
            {
                "reference": batch.reference,
                "sku": batch.sku,
                "purchased_quantity": batch._purchased_quantity,
                "eta": batch.eta,
            },
        )

    async def _save(self, batch: Batch) -> None:
        await self.conn.execute(
            text("""
                UPDATE ordering.batches
                SET purchased_quantity = :qty, eta = :eta
                WHERE reference = :reference
            """),
            {"qty": batch._purchased_quantity, "eta": batch.eta, "reference": batch.reference},
        )

    async def _get_by_sku(self, sku: str) -> list[Batch]:
        result = await self.conn.execute(
            text("""
                SELECT reference, sku, purchased_quantity, eta
                FROM ordering.batches
                WHERE sku = :sku
                ORDER BY eta NULLS FIRST
            """),
            {"sku": sku},
        )
        batches = []
        for row in result.mappings():
            allocations = await self._fetch_allocations(row["reference"])
            batches.append(self._to_domain(row, allocations))
        return batches
```

---

## Fake Repository (for Unit Tests)

```python
class FakeBatchRepository(AbstractBatchRepository):
    def __init__(self, batches: list[Batch] | None = None) -> None:
        super().__init__()
        self._store: dict[str, Batch] = {b.reference: b for b in (batches or [])}

    async def _get(self, reference: str) -> Batch | None:
        return self._store.get(reference)

    async def _get_by_sku(self, sku: str) -> list[Batch]:
        return [b for b in self._store.values() if b.sku == sku]

    def _add(self, batch: Batch) -> None:
        self._store[batch.reference] = batch

    async def _save(self, batch: Batch) -> None:
        self._store[batch.reference] = batch
```

No database. No migrations. Unit tests run instantly.

---

## Repository Rules

**Never call `commit()` or `rollback()`.** That belongs to the Unit of Work exclusively.

**One repository per Aggregate root.** An `OrderLineRepository` is a design smell — `OrderLine` is internal to the `Batch` aggregate. External code accesses `OrderLine` only through `Batch`.

**Methods use domain language.** Not `find_by_id` everywhere — use `get(reference)`, `get_by_sku(sku)`, `list_available_for(sku, quantity)`. The interface reads like domain expert vocabulary.

**Return domain objects.** Never raw dicts, never database row proxies. The `_to_domain` method is the explicit, deliberate translation between the database schema and the domain model.

---

## CQRS Read Queries

For read-heavy endpoints that cross aggregate boundaries or serve reporting needs, bypass the domain model entirely with a direct SQL query returning plain dataclasses. These are not domain repositories — they are read model queries.

```python
# infrastructure/repositories/ordering/read/batch_availability.py
from sqlalchemy.ext.asyncio import AsyncConnection
from sqlalchemy import text
from dataclasses import dataclass

@dataclass
class BatchAvailabilityRow:
    reference: str
    sku: str
    available_quantity: int

async def get_batch_availability(conn: AsyncConnection, sku: str) -> list[BatchAvailabilityRow]:
    result = await conn.execute(
        text("""
            SELECT
                b.reference,
                b.sku,
                b.purchased_quantity - COALESCE(SUM(ol.quantity), 0) AS available_quantity
            FROM ordering.batches b
            LEFT JOIN ordering.allocations a ON a.batch_id = b.id
            LEFT JOIN ordering.order_lines ol ON ol.id = a.orderline_id
            WHERE b.sku = :sku
            GROUP BY b.reference, b.sku, b.purchased_quantity
            ORDER BY b.eta NULLS FIRST
        """),
        {"sku": sku},
    )
    return [BatchAvailabilityRow(**row) for row in result.mappings()]
```

Read functions receive a connection directly — not via UoW. They do not pass through the domain model.
