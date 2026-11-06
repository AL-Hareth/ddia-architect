# Data models

Book: Ch 2, "Data Models and Query Languages" (sections: Relational Model vs. Document Model; Many-to-one and many-to-many relationships; Relational vs. document databases today; Graph-like Data Models)

## Core ideas

- The data model shapes how the problem is thought about and which queries are easy. Any model can emulate the others, but the result is usually awkward.
- **Relationships are the deciding factor.**
  - *One-to-many, self-contained*: a résumé and its positions, an order and its line items, a post and its comments. This forms a tree. If the whole tree is usually loaded at once, a document model fits well and gives storage locality.
  - *Many-to-one*: many profiles reference the same city or employer. The shared entity should be stored once and referenced by ID, so its human-meaningful details live in one place and change in one place. That requires joins.
  - *Many-to-many*: needs joins or graph traversal. Document databases with weak join support push those joins into application code.
  - Data tends to become more interconnected as features are added: free-text fields turn into entities, and entities start referencing each other. Plan for where the model is heading, not only the first release.
- **Relational**: joins, constraints, and declarative queries that an optimizer can adapt as the data changes; schema enforced on write. A mature, flexible default. Modern relational databases also store and index JSON.
- **Document**: schema flexibility, locality for whole-aggregate reads, a shape closer to application objects. Costs: weak or missing joins, denormalized copies that must be kept in sync, and whole-document rewrites on update, so keep documents small and never let them grow without bound.
- **Schema-on-read vs schema-on-write**: "schemaless" means the schema is implicit in the code that reads the data, not absent. It helps when records are genuinely heterogeneous or their structure is dictated by external systems. Otherwise it moves migration work into every reader.
- **Graph** (property graphs queried with Cypher-style languages, triple stores with SPARQL, Datalog): for data where anything can relate to anything, many-to-many is the norm, and queries traverse a variable number of hops (social networks, fraud rings, recommendations, knowledge graphs, network topology). Recursive SQL can traverse graphs but becomes clumsy quickly.
- **Declarative queries** (SQL, Cypher) let the system choose and change execution strategies and parallelize; imperative access ties the application to one strategy.
- **An event log as the model of record** (store what happened, derive current state) is another option. See `derived-data.md`.

## Questions to ask

- Sketch the main entities. For each relationship: a contained child, a reference to a shared entity, or many-to-many?
- Is a typical read "load one aggregate in full", "join across entities", or "follow relationships several hops"?
- Which entities are shared and mutable (names, prices, organizations)? Where would duplicated copies need updating?
- Will other teams, analysts, or reporting tools query this data directly?
- Are records structurally uniform, or many different shapes (user-defined fields, data from external sources)?
- Does history matter: who changed what and when, or what the state was on a past date?

## Decision guide

| Data shape and access | Lean toward |
|---|---|
| Uniform entities with references, joins, integrity constraints, ad-hoc queries | Relational (the default) |
| Self-contained aggregates read and written as a unit, few cross-references | A document store, or JSON columns in a relational database |
| Mostly relational with some flexible or heterogeneous attributes | Relational plus JSON columns |
| Densely connected data where variable-depth traversals are core queries | A graph database (or recursive SQL if traversals are modest and rare) |
| Audit trail, temporal queries, several read models over the same facts | An append-only event log as system of record, plus derived views |

When unsure, start relational: it handles every relationship type acceptably and is the easiest to evolve. Normalize deliberately: store shared facts once and reference them by ID. Denormalize only for a measured read-performance need, and name the mechanism that keeps the copies in sync.

## Red flags

- A document database chosen "because it's schemaless" while the data is clearly relational.
- Application-side joins on hot paths (N+1 query patterns) to work around a missing join.
- Copies of shared, mutable data embedded in many documents with no update mechanism.
- Large documents updated one field at a time, or documents that grow without bound (an ever-growing array of events inside a single record).
- A graph database for data that is really a handful of foreign keys.
- Several stores using different models with no statement of which is the source of truth.

## What to record

- The model chosen for each store and why, tied to the relationships and the top queries.
- An entity-relationship sketch with the top queries mapped onto it.
- Normalization and denormalization decisions, and how denormalized copies are maintained.
