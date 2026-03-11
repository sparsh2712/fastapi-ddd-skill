# Domain Discovery — Interview Protocol

---

## When to Use This Protocol

Before writing any domain code, run this protocol when:

- A new system or major feature is being designed
- It is unclear whether concepts belong in one Bounded Context or multiple
- The user describes business capabilities without clear boundaries
- Domain language is ambiguous or overloaded (same term, different meanings)

**Rule: If uncertain whether domains are separate or connected, always interview the user. Never assume.**

---

## Phase 1 — Identify Business Capabilities

Ask the user to describe what the system does in business terms, not technical terms. Extract a flat list of capabilities.

**Interview questions:**

1. "What are the distinct business capabilities this system provides?"
2. "Who are the different actors or user roles that interact with the system?"
3. "If you had to explain this system to a new hire, what are the major areas you would describe separately?"

**Output:** A capability list.

```text
Example:
- Place orders
- Manage product catalog
- Process payments
- Fulfill and ship orders
- Handle returns and refunds
- Manage customer accounts
```

---

## Phase 2 — Classify Subdomains

For each capability, determine its subdomain type.

**Interview questions:**

1. "Which of these capabilities gives your business a competitive advantage?" → **Core Domain**
2. "Which capabilities are necessary but not differentiating — you just need them to work?" → **Supporting Subdomain**
3. "Which capabilities are commodity — could be replaced by an off-the-shelf solution?" → **Generic Subdomain**

**Output:** Subdomain classification table.

| Capability | Subdomain Type | Rationale |
|---|---|---|
| Pricing engine | Core Domain | Differentiates from competitors |
| Order management | Core Domain | Central business logic |
| User authentication | Generic Subdomain | Use a library or service |
| Email notifications | Generic Subdomain | Commodity infrastructure |
| Inventory tracking | Supporting Subdomain | Needed but not differentiating |

**Design investment rule:**

| Subdomain Type | Investment Level |
|---|---|
| Core Domain | Full DDD — Aggregates, domain events, TDD, rich behavior |
| Supporting Subdomain | Moderate — lighter domain model, some CRUD acceptable |
| Generic Subdomain | Minimal — use a library, buy a service, simple CRUD |

---

## Phase 3 — Identify Bounded Context Boundaries

Determine where to draw boundaries. Each Bounded Context has its own domain model, its own Ubiquitous Language, and its own module structure.

**Interview questions to detect separate contexts:**

1. "Does the same term mean different things in different parts of the business?"
   - Yes → likely separate Bounded Contexts
   - Example: "Order" in Sales (a purchase request) vs "Order" in Fulfillment (a pick-and-pack instruction)

2. "Do these concepts always change together in a single transaction?"
   - Yes → likely the same Aggregate or Bounded Context
   - No → likely separate Bounded Contexts communicating through events

3. "Would different teams own these parts of the system?"
   - Yes → strong signal for separate Bounded Contexts
   - No → may still be separate if the models diverge

4. "If one of these parts had a bug or went down, would the others still make sense on their own?"
   - Yes → separate Bounded Contexts
   - No → likely the same context or tightly coupled contexts that need rethinking

5. "Do these parts share the same lifecycle? Are they deployed together, versioned together?"
   - Yes → possibly the same context
   - No → separate contexts

**Signals that domains are CONNECTED (same Bounded Context):**

- They share transactional consistency — must succeed or fail together
- They use the same vocabulary with the same meaning
- They are owned by the same team
- One cannot exist without the other
- Changes to one always require changes to the other

**Signals that domains are SEPARATE (different Bounded Contexts):**

- The same word means different things in each area
- They have independent lifecycles
- Different teams own them or would naturally own them
- They communicate through well-defined contracts, not shared state
- They can evolve independently without breaking each other

**Output:** Bounded Context catalog.

| Bounded Context | Responsibility | Key Aggregates |
|---|---|---|
| Ordering | Order placement and lifecycle | Order, OrderLine |
| Catalog | Product data and pricing | Product, Category |
| Fulfillment | Picking, packing, shipping | Shipment, PickList |
| Billing | Invoicing and payment processing | Invoice, Payment |

---

## Phase 4 — Map Context Relationships

For every pair of Bounded Contexts that interact, define the relationship pattern and contract ownership.

**Interview questions:**

1. "When Context A needs data from Context B, who adapts to whom?"
2. "Does one context dictate the model and the other conforms?"
3. "Do the teams collaborate closely, or does one publish a contract the other consumes?"

**Relationship patterns:**

| Pattern | Description | When to Use |
|---|---|---|
| **Partnership** | Two contexts evolve together, mutual coordination | Teams collaborate closely, shared roadmap |
| **Shared Kernel** | Small shared model owned jointly | Tightly coupled concepts that cannot diverge |
| **Customer-Supplier** | Upstream delivers what downstream needs | Clear producer-consumer relationship |
| **Conformist** | Downstream accepts upstream model as-is | Upstream has no incentive to accommodate |
| **Anti-Corruption Layer (ACL)** | Downstream translates upstream model into its own language | Protecting domain purity from external/legacy models |
| **Open Host Service** | Upstream provides a well-defined protocol for many consumers | Public API, multiple downstream consumers |
| **Published Language** | Shared interchange format (e.g., JSON schema, protobuf) | Standard data exchange across contexts |

**Output:** Context relationship map.

| Upstream | Downstream | Pattern | Contract Owner | Translation Needed |
|---|---|---|---|---|
| Catalog | Ordering | Customer-Supplier | Catalog | Yes — product snapshot at order time |
| Ordering | Fulfillment | Customer-Supplier | Ordering | Yes — order → shipment instruction |
| Payments (external) | Billing | ACL | Billing | Yes — external payment model translated |

---

## Phase 5 — Establish Ubiquitous Language

For each Bounded Context, define the canonical terms. Resolve ambiguity explicitly.

**Interview questions:**

1. "What do you call [concept] in the context of [business area]?"
2. "When you say [term], what exactly does it mean? What does it NOT mean?"
3. "Are there terms your team uses that developers might misunderstand, or vice versa?"

**Output:** Ubiquitous Language glossary per context.

| Term | Definition | Context | Anti-terms (what it does NOT mean) |
|---|---|---|---|
| Order | A confirmed purchase request with one or more lines | Ordering | Not a draft cart, not a shipment |
| Batch | A quantity of stock received from a supplier | Ordering | Not a database batch operation |
| Allocation | Reserving stock from a Batch for an OrderLine | Ordering | Not a memory allocation |
| Shipment | A physical package dispatched to a customer | Fulfillment | Not the order itself |

**Naming validation rule:** Every class, method, and module name in code must use terms from this glossary. If a developer cannot find the right term in the glossary, the glossary is incomplete — update it before writing code.

---

## Phase 6 — Document Boundary Decisions

Record why boundaries were drawn where they are, so future developers do not accidentally merge or split contexts without understanding the rationale.

**Template:**

```text
Decision: [Context A] and [Context B] are separate Bounded Contexts.
Rationale: [Why they are separate — different ownership, different models, different lifecycles]
Consequences: [How they communicate — events, ACL, direct calls]
Risks: [What could go wrong — coupling, data duplication, consistency gaps]
Revisit when: [Triggers for re-evaluating — team restructuring, feature convergence]
```

---

## Quick Reference — Decision Flowchart

```text
Start: New business capability identified
  │
  ├─ Does the same term mean different things here vs elsewhere?
  │    Yes → Separate Bounded Context
  │    No  ↓
  │
  ├─ Must these concepts change in the same transaction?
  │    Yes → Same Aggregate / Bounded Context
  │    No  ↓
  │
  ├─ Would different teams naturally own these?
  │    Yes → Separate Bounded Context
  │    No  ↓
  │
  ├─ Can these parts evolve independently?
  │    Yes → Separate Bounded Context
  │    No  → Same Bounded Context (but monitor for divergence)
```

**When still unclear after all questions → Ask the user. Never guess domain boundaries.**
