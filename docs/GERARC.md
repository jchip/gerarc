# Relational Design Foundations: GER + ARC

## Overview

This document outlines a set of **idiomatic relational design patterns and best practices** centered around two core building blocks:

* **GER (Global Entity Registry)**: a normalized identity and supertype pattern
* **ARC (Association / Relationship / Connection tables)**: normalized association tables for modeling relationships

Together, **GER + ARC** provide a clean, flexible foundation for building complex relational systems that need to support many entity types, shared resources, and evolving relationships, while remaining fully relational and operationally sound.

This document also describes the surrounding best practices and secondary patterns that typically layer on top of GER + ARC to form a robust production schema.

---

## Core Structural Patterns

### 1. Global Entity Registry (GER)

**Purpose**
GER centralizes identity across heterogeneous entity types and eliminates unsafe polymorphic `type/id` pairs.

**Pattern**

```sql
entities (
  id BIGINT PRIMARY KEY,
  kind_id SMALLINT NOT NULL,
  native_id BIGINT NOT NULL,
  UNIQUE (kind_id, native_id)
)
```

**What GER provides**

* A single, normalized identity layer
* Referential integrity across entity types
* A unified relational identity key across entity tables
* A clean replacement for `entity_type + entity_id`

**Textbook mapping**

* Supertype table / Surrogate identity / Class Table Inheritance (CTI)
* Entities and Aggregate Roots (objects defined by identity, not attributes)

GER answers the question:
**"What is a thing, independent of how it is used?"**

---

### 2. ARC Tables (Association / Relationship / Connection)

**Purpose**
ARC tables model relationships between entities or between entities and shared resources.

**Pattern**

```sql
entity_resource_links (
  entity_id BIGINT REFERENCES entities(id),
  resource_id BIGINT REFERENCES resources(id),
  is_primary BOOLEAN,
  is_verified BOOLEAN,
  UNIQUE (entity_id, resource_id)
)
```

**What ARC provides**

* Explicit relationship modeling
* Support for many-to-many and optional relationships
* Attributes on relationships
* Elimination of embedded ownership fields

**Textbook mapping**

* Association table / Junction table with payload
* Cross-Aggregate references and Aggregate-to-Value Object associations

ARC answers the question:
**"How do things relate to each other?"**

---

## GER + ARC Together

GER and ARC form the **two most idiomatic structural patterns in relational design**:

* GER defines **identity**—what things are (Entities, Aggregate Roots)
* ARC defines **relationships**—how things connect (Aggregate boundaries, associations)

Everything else in a relational system builds on these two ideas.

This provides a relationship-centric model that stays entirely within SQL, constraints, and joins. It also cleanly separates Entities (identity-based) from Value Objects (attribute-based, shareable), and makes Aggregate boundaries explicit through relationship tables rather than embedded FKs.

---

## Attribute Tracking and Validation Patterns (GERARC)

When verification/validation metadata must be tracked at the **attribute level**, GERARC extends the base GER + ARC model with an attribute-centric pattern:

* Attribute definitions identify what is being attested.
* Evidence records store proof events (who/what/when/how).
* Attribute ARCs link subject entity + attribute + evidence.

Conceptually, three attribute classes are handled differently.

### 1) Idiomatic literal value attribute

Example: `accounts.display_name` stored as a literal column.

**Approach**

* Keep canonical value in the existing literal column.
* Track value-change time with a singleton, mutable `value_modification` record per `(entity, attribute)`.
* Track verification/validation as immutable append-only ARC events.
* Detect staleness by comparing latest verification time vs latest value-modification time.

**Why this pattern**

* Avoids forced normalization migrations for legacy literals.
* Preserves full audit trail for verification events.
* Supports efficient staleness checks without scanning full history.

### 2) FK attribute value

Example: `orders.customer_id` referencing `customers.id`.

**Approach**

* Treat the FK column as the attribute being attested.
* In attribute ARCs, store the referenced GER entity for the FK target.
* Keep verification/validation evidence immutable and append-only.
* Use latest-event query semantics for current trust state.

**Why this pattern**

* Reuses existing relational FK structure.
* Makes it explicit which referenced entity was verified/validated.
* Keeps trust metadata decoupled from base table schema changes.

### 3) Full ARC linked attribute (clean GERARC)

Example: account linked to a normalized contact method through an association ARC.

**Approach**

* Model value as its own entity (or value object table) and link via ARC.
* Use the linked entity/association timestamps as canonical modification signal.
* Store verification/validation as immutable evidence-backed ARC events.
* Evaluate freshness against the linked entity or association lifecycle timestamps.

**Why this pattern**

* Most normalized and composable long-term model.
* Natural fit for multi-use, shared, or evolving value objects.
* Simplifies consistency and future relationship-centric querying.

### Cross-cutting validation rules

Across all three patterns:

* `value_modification` metadata is mutable and singleton per tracked attribute (efficiency).
* Verification/validation records are immutable and append-only (audit integrity).
* Current status is derived from latest relevant evidence, not from destructive updates.

---

## Surrounding Best Practices and Supporting Patterns

The following patterns are not alternatives to GER + ARC. They **layer on top of them** to address lifecycle, performance, history, and operational needs.

---

### 3. Base Entity Tables

Domain-specific tables that hold intrinsic attributes.

Entities (identity matters):
* `accounts`, `organizations`

Value Objects (attributes matter, shareable):
* `postal_addresses`, `contact_numbers`, `email_addresses`

Best practices:

* Third Normal Form or better
* No polymorphic ownership
* Stable primary keys
* Value Objects should be immutable or versioned

GER does not replace these tables—it provides a unified identity layer above them.

---

### 4. Lookup and Domain Tables

Used to constrain and normalize categorical values.

Examples:

* `entity_kinds`
* `roles`
* `statuses`
* `countries`

Benefits:

* Enforced domains
* Smaller rows
* Easier evolution than enums at scale

---

### 5. Subtype and Inheritance Patterns

Used when entities share identity but differ in attributes.

Common options:

* Class Table Inheritance (preferred with GER)
* Single Table Inheritance (use cautiously)
* Concrete inheritance

GER aligns naturally with CTI.

---

### 6. Constraint-Driven Design

Relational integrity should live in the database whenever possible.

Includes:

* Foreign keys
* UNIQUE constraints
* CHECK constraints
* Partial UNIQUE constraints

GER + ARC maximize the ability to express invariants declaratively.

---

### 7. Lifecycle and State Modeling

Used when entity state changes matter.

Patterns:

* Current state column
* State transition tables
* Event-style append tables

These are behavioral layers on top of core identity and relationships.

---

### 8. Temporal and Historical Patterns

Used to preserve history and time correctness.

Examples:

* `valid_from / valid_to`
* `effective_at`
* Slowly Changing Dimensions (Type 2)

Often combined with ARC tables for time-bound relationships.

---

### 9. Canonical vs Snapshot Data

Separates shared truth from historical context.

Examples:

* Canonical: `resources`
* Snapshot: `event_resource_snapshot`

Prevents denormalization while preserving auditability.

---

### 10. Soft Delete and Retention Patterns

Used when data cannot be physically deleted.

Patterns:

* `deleted_at`
* Partial indexes
* Archive tables

Works naturally with GER and ARC via cascades and constraints.

---

### 11. Denormalized Read Models

Normalized write models do not preclude optimized reads.

Examples:

* Materialized views
* Read replicas
* Cached aggregates

Key principle:
**Normalize first, optimize reads second.**

---

## Relational Positioning

GER + ARC represent a **relational middle ground** between:

* Ad-hoc polymorphic schemas, and
* Overly rigid one-to-one ownership models

This positioning:

* Improves data quality today
* Keeps relationship modeling explicit and auditable
* Avoids premature infrastructure complexity

---

## Design Principles Summary

* Prefer **identity tables** over polymorphic references
* Model **relationships explicitly** as Aggregate boundaries
* Distinguish Entities (identity-primary) from Value Objects (attribute-primary)
* Use constraints to enforce invariants
* Normalize canonical data; keep Value Objects shareable
* Preserve history deliberately
* Optimize reads without corrupting the model
* Keep patterns composable and incremental

---

## Conclusion

GER and ARC are not new inventions. They are **core relational modeling patterns** that bring schemas back in line with first principles:

* Identity is explicit
* Relationships are data
* Constraints enforce reality

When combined with established best practices around lifecycle, history, and performance, **GER + ARC provide a clean, idiomatic foundation for building and evolving complex relational systems**.
