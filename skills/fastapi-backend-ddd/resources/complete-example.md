# Complete Example — Stock Allocation System

Full end-to-end implementation across all four layers.

---

## 1. Domain Model

```python
# domain/ordering/model.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
from domain.ordering.events import Allocated, Deallocated
from domain.ordering.exceptions import OutOfStockError

@dataclass(frozen=True)
class OrderLine:
    order_id: str
    sku: str
    quantity: int

class Batch:
    def __init__(
        self,
        reference: str,
        sku: str,
        purchased_quantity: int,
        eta: Optional[datetime],
    ) -> None:
        self.reference = reference
        self.sku = sku
        self._purchased_quantity = purchased_quantity
        self.eta = eta
        self._allocations: set[OrderLine] = set()
        self.events: list = []

    @property
    def available_quantity(self) -> int:
        return self._purchased_quantity - sum(line.quantity for line in self._allocations)

    def can_allocate(self, line: OrderLine) -> bool:
        return self.sku == line.sku and self.available_quantity >= line.quantity

    def allocate(self, line: OrderLine) -> None:
        if not self.can_allocate(line):
            raise OutOfStockError(f"Cannot allocate {line.quantity} of {line.sku}")
        self._allocations.add(line)
        self.events.append(Allocated(
            order_id=line.order_id,
            sku=line.sku,
            quantity=line.quantity,
            batchref=self.reference,
        ))

    def deallocate(self, line: OrderLine) -> None:
        if line in self._allocations:
            self._allocations.discard(line)
            self.events.append(Deallocated(
                order_id=line.order_id,
                sku=line.sku,
                quantity=line.quantity,
            ))

    def __eq__(self, other: object) -> bool:
        if not isinstance(other, Batch):
            return False
        return self.reference == other.reference

    def __hash__(self) -> int:
        return hash(self.reference)

    def __gt__(self, other: "Batch") -> bool:
        if self.eta is None:
            return False
        if other.eta is None:
            return True
        return self.eta > other.eta
```

---

## 2. Domain Events & Exceptions

```python
# domain/ordering/events.py
from dataclasses import dataclass

@dataclass(frozen=True)
class Allocated:
    order_id: str
    sku: str
    quantity: int
    batchref: str

@dataclass(frozen=True)
class Deallocated:
    order_id: str
    sku: str
    quantity: int
```

```python
# domain/ordering/exceptions.py
class OutOfStockError(Exception):
    pass

class InvalidSkuError(Exception):
    pass

class BatchNotFoundError(Exception):
    pass
```

---

## 3. Repository

```python
# infrastructure/repositories/ordering/batch_repository.py
import abc
from sqlalchemy.ext.asyncio import AsyncConnection
from sqlalchemy import text
from domain.ordering.model import Batch, OrderLine

class AbstractBatchRepository(abc.ABC):
    def __init__(self) -> None:
        self.seen: set[Batch] = set()

    async def get(self, reference: str) -> Batch | None:
        result = await self._get(reference)
        if result:
            self.seen.add(result)
        return result

    async def get_by_sku(self, sku: str) -> list[Batch]:
        results = await self._get_by_sku(sku)
        self.seen.update(results)
        return results

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

---

## 4. Unit of Work

```python
# unit_of_work/postgres.py
from sqlalchemy.ext.asyncio import AsyncEngine, AsyncConnection
from infrastructure.repositories.ordering.batch_repository import PostgresBatchRepository
from unit_of_work.abstract import AbstractUnitOfWork

class PostgresUnitOfWork(AbstractUnitOfWork):
    def __init__(self, engine: AsyncEngine) -> None:
        self._engine = engine

    async def __aenter__(self) -> "PostgresUnitOfWork":
        self._conn: AsyncConnection = await self._engine.connect()
        await self._conn.begin()
        self.batches = PostgresBatchRepository(self._conn)
        return await super().__aenter__()

    async def __aexit__(self, *args) -> None:
        await super().__aexit__(*args)
        await self._conn.close()

    async def commit(self) -> None:
        await self._conn.commit()

    async def rollback(self) -> None:
        await self._conn.rollback()
```

```python
# unit_of_work/fake.py
from infrastructure.repositories.ordering.batch_repository import FakeBatchRepository
from unit_of_work.abstract import AbstractUnitOfWork

class FakeUnitOfWork(AbstractUnitOfWork):
    def __init__(self) -> None:
        self.batches = FakeBatchRepository()
        self.committed = False

    async def __aenter__(self) -> "FakeUnitOfWork":
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
        if exc_type:
            await self.rollback()

    async def commit(self) -> None:
        self.committed = True

    async def rollback(self) -> None:
        pass
```

---

## 5. Service Layer

```python
# service/ordering/allocation.py
from dataclasses import dataclass
from datetime import datetime
from typing import Optional
from unit_of_work.abstract import AbstractUnitOfWork
from domain.ordering.model import Batch, OrderLine
from domain.ordering.services import allocate_to_earliest_batch
from domain.ordering.exceptions import InvalidSkuError, BatchNotFoundError

@dataclass
class AllocationResult:
    batchref: str
    sku: str

async def create_batch(
    reference: str,
    sku: str,
    quantity: int,
    eta: Optional[datetime],
    uow: AbstractUnitOfWork,
) -> str:
    async with uow:
        uow.batches.add(Batch(reference=reference, sku=sku, purchased_quantity=quantity, eta=eta))
        return reference

async def allocate(
    order_id: str,
    sku: str,
    quantity: int,
    uow: AbstractUnitOfWork,
) -> AllocationResult:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        if not batches:
            raise InvalidSkuError(f"Invalid sku {sku}")
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        batchref = allocate_to_earliest_batch(batches, line)
        return AllocationResult(batchref=batchref, sku=sku)

async def deallocate(
    order_id: str,
    sku: str,
    quantity: int,
    uow: AbstractUnitOfWork,
) -> str:
    async with uow:
        batches = await uow.batches.get_by_sku(sku)
        line = OrderLine(order_id=order_id, sku=sku, quantity=quantity)
        batch = next((b for b in batches if line in b._allocations), None)
        if batch is None:
            raise BatchNotFoundError(f"No allocation for order {order_id} sku {sku}")
        batch.deallocate(line)
        await uow.batches.save(batch)
        return batch.reference
```

---

## 6. Unit Tests

```python
# tests/unit/service/test_allocation.py
import pytest
from datetime import datetime, timedelta
from service.ordering.allocation import allocate, create_batch, deallocate
from unit_of_work.fake import FakeUnitOfWork
from domain.ordering.exceptions import InvalidSkuError, OutOfStockError

async def test_allocates_to_earliest_available_batch():
    uow = FakeUnitOfWork()
    today = datetime.now()
    later = today + timedelta(days=10)
    await create_batch("speedy-batch", "SMALL-TABLE", 100, today, uow)
    await create_batch("slow-batch", "SMALL-TABLE", 100, later, uow)

    result = await allocate("order-001", "SMALL-TABLE", 10, uow)

    assert result.batchref == "speedy-batch"

async def test_raises_for_invalid_sku():
    uow = FakeUnitOfWork()
    await create_batch("b1", "REAL-SKU", 100, None, uow)

    with pytest.raises(InvalidSkuError):
        await allocate("o1", "NONEXISTENT", 10, uow)

async def test_raises_when_out_of_stock():
    uow = FakeUnitOfWork()
    await create_batch("b1", "POPULAR-CHAIR", 10, None, uow)

    with pytest.raises(OutOfStockError):
        await allocate("o1", "POPULAR-CHAIR", 20, uow)

async def test_commits_on_successful_allocation():
    uow = FakeUnitOfWork()
    await create_batch("b1", "POPULAR-CHAIR", 100, None, uow)

    await allocate("o1", "POPULAR-CHAIR", 10, uow)

    assert uow.committed is True
```

---

## 7. Router

```python
# entrypoints/routers/allocation.py
from fastapi import APIRouter, Depends, status
from pydantic import BaseModel
from datetime import datetime
from typing import Optional
from entrypoints.dependencies import get_uow
from unit_of_work.abstract import AbstractUnitOfWork
from service.ordering import allocation as allocation_service
from domain.ordering.exceptions import InvalidSkuError, OutOfStockError, BatchNotFoundError

router = APIRouter(prefix="/allocations", tags=["allocations"])

class BatchCreateRequest(BaseModel):
    reference: str
    sku: str
    quantity: int
    eta: Optional[datetime] = None
    model_config = {"extra": "forbid"}

class AllocateRequest(BaseModel):
    order_id: str
    sku: str
    quantity: int
    model_config = {"extra": "forbid"}

class AllocateResponse(BaseModel):
    batchref: str

@router.post("/batches", status_code=status.HTTP_201_CREATED)
async def add_batch(
    body: BatchCreateRequest,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> dict:
    ref = await allocation_service.create_batch(
        reference=body.reference,
        sku=body.sku,
        quantity=body.quantity,
        eta=body.eta,
        uow=uow,
    )
    return {"reference": ref}

@router.post("", status_code=status.HTTP_201_CREATED)
async def allocate(
    body: AllocateRequest,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> AllocateResponse:
    result = await allocation_service.allocate(
        order_id=body.order_id,
        sku=body.sku,
        quantity=body.quantity,
        uow=uow,
    )
    return AllocateResponse(batchref=result.batchref)

@router.delete("/{order_id}", status_code=status.HTTP_204_NO_CONTENT)
async def deallocate(
    order_id: str,
    sku: str,
    quantity: int,
    uow: AbstractUnitOfWork = Depends(get_uow),
) -> None:
    await allocation_service.deallocate(
        order_id=order_id, sku=sku, quantity=quantity, uow=uow
    )
```
