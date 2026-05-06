# 🪢 Introduction to Knex

An interactive Reveal.js presentation covering **Knex** — the chainable SQL query builder for Node.js. One JavaScript API across PostgreSQL, MySQL, SQLite, MSSQL and Oracle — plus migrations, seeds, transactions, streams and a connection pool.

## ▶ [Open the Presentation](https://brendanjameslynskey.github.io/Introduction_to_Knex/)

## 📄 [Markdown Version](presentation.md)

## 🗄 [Companion deck — Introduction to Node.js & Databases](https://brendanjameslynskey.github.io/Introduction_to_Node_Databases/)

---

## Contents

| # | Topic | Description |
|---|-------|-------------|
| 01 | Title | Pitch and the build → schema → migrate → seed → query → transact flow |
| 02 | Topics | Map of foundations, querying, schema & data, production |
| 03 | What Is Knex? | Origins, scope, query-builder vs ORM, who uses it |
| 04 | Why Knex? | Pain points solved, composition, honest trade-offs |
| 05 | Installation | Knex + per-dialect drivers, the thirty-second tour |
| 06 | knexfile.js | Environments, CLI vs app-side instantiation |
| 07 | Connection pool | tarn config, sizing, health visibility, anti-patterns |
| 08 | SELECT | Where, ordering, limit, distinct, `.toSQL()` |
| 09 | Joins | Inner, left, right, cross, self, callback `ON` clauses |
| 10 | Aggregates | Count / sum / window functions / upserts / helpers |
| 11 | INSERT / UPDATE / DELETE | `RETURNING`, `onConflict`, increments, soft delete |
| 12 | Sub-queries & CTEs | Inline, derived tables, `with`, `withRecursive`, UNION |
| 13 | knex.raw | `?`, `:name`, `??` bindings, raw inside a chain, FTS |
| 14 | Schema builder | Create / alter / drop, FK clauses, type cheat sheet |
| 15 | Migrations | Workflow, immutability, transactional DDL gotchas |
| 16 | Seeds & fixtures | Idempotency, faker, ordering, dev vs test |
| 17 | Transactions | Callback / manual, savepoints, isolation levels |
| 18 | Streams & batching | `.stream()`, `batchInsert`, keyset pagination |
| 19 | TypeScript | `Knex.Tables` declaration merging, Zod boundaries, limits |
| 20 | Testing | In-memory SQLite, transaction rollback, Testcontainers |
| 21 | Performance & gotchas | `.toSQL()`, N+1, pool starvation, indexing wins |
| 22 | Knex vs alternatives | Capability matrix vs Prisma, Drizzle, Kysely, TypeORM, raw |
| 23 | Production patterns | Repository pattern, Express, retries, observability |
| 24 | Summary | Take-aways, next steps, further reading |

---

## Slide Controls

| Action | Key |
|--------|-----|
| Next / Previous | `→` `←` or swipe |
| Overview | `Esc` |
| Fullscreen | `F` |
| Speaker notes | `S` |
| Export to PDF | Append `?print-pdf` to URL, then print |

## Technology

[Reveal.js 4.6](https://revealjs.com) · [highlight.js](https://highlightjs.org) · Playfair Display + DM Sans + JetBrains Mono

Single self-contained `index.html` — no build step, no npm, no dependencies to install.

## References

Knex documentation — knexjs.org/guide · Knex source — github.com/knex/knex · node-postgres — node-postgres.com · mysql2 — sidorares.github.io/node-mysql2 · better-sqlite3 — github.com/WiseLibs/better-sqlite3 · Kysely — kysely.dev · PostgreSQL docs — postgresql.org/docs

## License

Educational use. Code examples provided as-is.
