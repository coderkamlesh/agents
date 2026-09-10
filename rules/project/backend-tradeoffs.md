---
alwaysApply: true
---

# Trade-off protocol

The user wants code that is **scalable, readable, and performant**. When a decision forces a
real trade-off between any of these, stop and ask instead of picking silently.

## Ask, do not guess

Present the choice before writing code when any of these come up:

- JPA versus JDBC / NamedParameterJdbcTemplate
- Synchronous versus asynchronous execution
- Strong consistency versus availability or latency
- Normalised versus denormalised schema
- Adding a cache, and what invalidation strategy follows
- Batching versus streaming
- Adding a new dependency or abstraction layer

## How to present it

For each option state: what it buys, what it costs, and a recommendation. Then ask.

Ask **one question at a time** - never a wall of questions in a single message.

## Do not

- Do not silently pick the simpler option to save time.
- Do not implement first and mention the trade-off afterwards.
- Do not ask about trivial choices that are easily reversible.
