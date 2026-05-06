# Introduction to Knex

Full slide content in Markdown format. Each slide is separated by `---`.

---

## Slide 01 — Title

# Introduction to Knex

**The chainable SQL query builder for Node.js**

PostgreSQL · MySQL · SQLite · MSSQL · Oracle

> One chainable JavaScript API, one connection pool, parameterised SQL across five dialects — plus migrations, seeds and transactions in the same toolbox.

Build · Schema · Migrate · Seed · Query · Transact

---

## Slide 02 — Topics

### Foundations
- What Knex is — origins, scope, query builder vs ORM
- Why Knex — abstraction, dialect portability, trade-offs
- Installation, clients, the first query
- `knexfile.js`, environments, pool configuration

### Querying
- `SELECT`, `WHERE`, ordering, limit
- Joins — inner, left, right, cross, self
- Aggregates & `GROUP BY` / `HAVING`
- `INSERT`, `UPDATE`, `DELETE` with `RETURNING`
- Sub-queries, CTEs, `UNION`
- `knex.raw` & safe bindings

### Schema & data
- Schema builder — `createTable`, `alterTable`
- Migrations — `up`/`down`, ordering, drift
- Seeds & fixtures — idempotent populate
- Transactions, savepoints, isolation
- Streams, `batchInsert`, pagination

### Production
- TypeScript with Knex — typed tables & results
- Express integration patterns
- Testing — in-memory SQLite, transaction rollback
- Performance, `.toSQL()`, pool tuning, gotchas
- Knex vs Prisma, Drizzle, Kysely, TypeORM, raw
- Migration playbook and observability

---

## Slide 03 — What Is Knex?

Knex is a **SQL query builder** for Node.js — created by Tim Griesser in 2013 and now maintained by the knex.js community. It generates parameterised SQL for PostgreSQL, MySQL/MariaDB, SQLite, MSSQL, Oracle and CockroachDB from a chainable JavaScript API.

### What it owns
- **Query building** — chainable, composable, parameterised
- **Schema builder** — DDL across dialects in JS
- **Migrations** — versioned, ordered, reversible
- **Seeds** — repeatable data population
- **Connection pool** — via `tarn`
- **Transactions** — including savepoints
- Streams & batching for large result sets

### What it deliberately is not
- Not an ORM — no models, no identity map, no lazy relations
- No relationship objects — you write the JOIN
- No code generation from the schema

### Where it sits in the stack

| Layer         | Tool                                         | You write          |
|---------------|----------------------------------------------|--------------------|
| Driver        | `pg` · `mysql2` · `better-sqlite3`           | Raw SQL strings    |
| Query builder | **Knex · Kysely**                            | **Chainable JS**   |
| ORM           | Prisma · TypeORM · Sequelize                 | Models & relations |
| ODM           | Mongoose                                     | Documents          |

### Who uses it
- Used internally by **Bookshelf.js**, **Objection.js** and **Strapi v3**
- Powers query layers in countless Express / Koa / Fastify apps
- Still active — maintenance releases through 2024–2025; v3 in development

---

## Slide 04 — Why Knex? — Pain Points and Trade-offs

Raw drivers force string concatenation; ORMs force a model layer. Knex sits between them — **SQL stays SQL**, but composition becomes a function call rather than a template string.

### What you gain
- **Composability** — conditional `.where()` / `.join()` without ad-hoc string splicing
- **Parameterisation by default** — bindings always go through the driver, no SQL-injection footgun
- **Dialect portability** — the same code targets PG, MySQL, SQLite, MSSQL, Oracle
- **Migrations & seeds in the box** — no separate flyway / sqitch / db-migrate
- One **connection pool** shared across the app

### Composition example

```javascript
function listUsers({ role, search, page = 1 }) {
  let q = knex('users').select('id', 'name', 'email');
  if (role)   q = q.where('role', role);
  if (search) q = q.whereILike('name', `%${search}%`);
  return q.orderBy('created_at', 'desc')
          .limit(20).offset((page - 1) * 20);
}
```

### What you give up
- **No model layer** — no `user.posts`, no cascade deletes, no identity map
- **No code-gen types** — rows are `any` unless you bring TS declarations or Zod
- **Dialect parity is partial** — window functions, JSON, `RETURNING`, upserts differ between databases
- **Maintenance velocity** is slower than Prisma / Drizzle — v2 is mature; v3 is in development

### When Knex is the right tool
- You think in SQL and want SQL on the wire
- Reporting / analytics queries that an ORM mangles
- Multi-dialect codebase — SQLite for tests, PG in prod
- Migrating a legacy schema without re-modelling it
- Building *another* abstraction (Objection.js, Strapi-style) on top

---

## Slide 05 — Installation & First Query

Install Knex itself plus **one driver per dialect** — Knex is dialect-agnostic and only loads the driver you ask for.

### Install

```bash
npm install knex

# pick the driver(s) you need
npm install pg                  # PostgreSQL / CockroachDB
npm install mysql2              # MySQL / MariaDB
npm install better-sqlite3      # SQLite (sync)
npm install sqlite3             # SQLite (async, legacy)
npm install tedious             # MSSQL
npm install oracledb            # Oracle

# CLI for migrations / seeds
npm install --save-dev knex
npx knex --version
```

### Driver matrix

| Dialect        | Client            | Driver                      |
|----------------|-------------------|-----------------------------|
| PostgreSQL     | `pg`              | node-postgres               |
| MySQL/MariaDB  | `mysql2`          | sidorares/node-mysql2       |
| SQLite         | `better-sqlite3`  | WiseLibs/better-sqlite3     |
| MSSQL          | `mssql`           | tediousjs/tedious           |
| Oracle         | `oracledb`        | oracle/node-oracledb        |
| CockroachDB    | `cockroachdb`     | via `pg`                    |

### The thirty-second tour

```javascript
// db.js
const knex = require('knex')({
  client: 'pg',
  connection: process.env.DATABASE_URL,
  pool: { min: 2, max: 10 }
});

const users = await knex('users')
  .select('id', 'name', 'email')
  .where('active', true)
  .orderBy('created_at', 'desc')
  .limit(20);

console.log(users);
await knex.destroy();   // close the pool on shutdown
```

### What happens behind the scenes
1. Knex compiles the chain into dialect-specific SQL
2. Bindings stay separate — the driver parameterises them
3. Pool checks out a connection (or queues)
4. Driver sends prepared statement; rows return as a Promise
5. Connection returns to the pool

---

## Slide 06 — knexfile.js & Environments

The **`knexfile.js`** is the canonical config — consumed by the CLI for migrations and seeds, and re-used by app code via `require('./knexfile')`. One file, one config per environment.

```javascript
// knexfile.js
require('dotenv').config();

module.exports = {
  development: {
    client: 'pg',
    connection: process.env.DATABASE_URL ||
                'postgres://dev:dev@localhost:5432/myapp',
    pool: { min: 2, max: 10 },
    migrations: { directory: './db/migrations' },
    seeds:      { directory: './db/seeds/development' }
  },

  test: {
    client: 'better-sqlite3',
    connection: { filename: ':memory:' },
    useNullAsDefault: true,
    migrations: { directory: './db/migrations' },
    seeds:      { directory: './db/seeds/test' }
  },

  production: {
    client: 'pg',
    connection: {
      connectionString: process.env.DATABASE_URL,
      ssl: { rejectUnauthorized: false }
    },
    pool: { min: 2, max: 20, idleTimeoutMillis: 30_000 },
    migrations: { directory: './db/migrations',
                  tableName: 'knex_migrations' }
  }
};
```

```bash
# CLI uses NODE_ENV, defaulting to development
NODE_ENV=production npx knex migrate:latest
NODE_ENV=test        npx knex seed:run

# explicit:
npx knex migrate:latest --env production
npx knex --knexfile ./infra/knexfile.js migrate:status
```

```javascript
// db.js — single shared instance
const config = require('../knexfile');
const knex = require('knex')(
  config[process.env.NODE_ENV || 'development']
);
module.exports = knex;
```

### Tips
- One Knex instance per process — never `require('knex')(...)` per request
- Co-locate `migrations/` and `seeds/` with the app code
- Store secrets in env vars, not in the knexfile
- Same migrations directory across dev/test/prod — only the connection differs
- Use `useNullAsDefault: true` for SQLite to silence a warning

---

## Slide 07 — The Connection Pool

Knex pools connections via **tarn.js**. The pool is shared across every query the Knex instance runs. Understanding it is the difference between a fast app and a stalled one.

```javascript
const knex = require('knex')({
  client: 'pg',
  connection: process.env.DATABASE_URL,
  pool: {
    min: 2,                       // keep this many warm
    max: 20,                      // hard cap on live conns
    acquireTimeoutMillis: 30_000, // queued, then throws
    createTimeoutMillis: 5_000,   // TCP handshake budget
    idleTimeoutMillis: 30_000,    // close idle conns
    reapIntervalMillis: 1_000,    // sweep frequency
    afterCreate: (conn, done) => {
      conn.query("SET TIME ZONE 'UTC'", err => done(err, conn));
    }
  }
});
```

### Anti-patterns
- Creating a Knex instance per request — exhausts DB connections in seconds
- Forgetting `knex.destroy()` in test teardown — leaks conns, hangs Jest
- Setting `max` = thousands — the DB is the bottleneck, not Knex

### Sizing rules of thumb
- PG default `max_connections` = 100 — budget across *all* services
- With *N* app instances: `max` ≈ `(server_max - reserved) / N`
- Saturating CPU on the DB > opening more connections
- For high concurrency, front the DB with **PgBouncer** in transaction-pooling mode

### Health visibility

```javascript
app.get('/healthz/db', (_req, res) => {
  const p = knex.client.pool;
  res.json({
    used:    p.numUsed(),
    free:    p.numFree(),
    pending: p.numPendingAcquires(),
    queued:  p.numPendingCreates()
  });
});
```

### Symptoms → cause
- `numPendingAcquires` climbs → pool too small or queries too slow
- `TimeoutError: Knex: Timeout acquiring a connection` → deadlock, leak, or unbounded queue
- RSS keeps climbing → transactions held without commit/rollback

---

## Slide 08 — SELECT — Where, Order, Limit

The query builder mirrors SQL clause names. Every chain is **thenable** — you can `await` it directly — and every chain stays composable until awaited.

### Basic SELECT

```javascript
await knex('users').select('id', 'name');
await knex('users').select(); // all columns

await knex('users').where({ active: true, role: 'admin' });
await knex('users').where('created_at', '>', '2025-01-01');

await knex('users')
  .where('role', 'admin')
  .orWhere(b => b.where('role', 'editor').andWhere('verified', true));

await knex('users').whereIn('id', [1, 2, 3]);
await knex('users').whereNotIn('role', ['guest']);
await knex('users').whereNull('deleted_at');
await knex('users').whereILike('email', '%@dev.io');
```

### Ordering, limit, distinct

```javascript
await knex('posts')
  .select('id', 'title')
  .orderBy([
    { column: 'pinned', order: 'desc' },
    { column: 'created_at', order: 'desc' }
  ])
  .limit(20).offset(40);

await knex('posts').distinct('author_id');
await knex('users').count('* as total').first();
await knex('users').pluck('email');
await knex('users').first('id', 'name');
```

### Inspect what is sent

```javascript
const q = knex('users').where('id', 1).toSQL();
// q.sql      — "select * from users where id = ?"
// q.bindings — [1]
console.log(q.toNative()); // dialect-native (e.g. $1 for pg)
```

---

## Slide 09 — Joins

Knex exposes every join SQL has, plus a **callback form** for compound `ON` clauses. There are no relationship objects — you write the join you want, and the planner sees the SQL you wrote.

```javascript
// inner join
await knex('posts as p')
  .join('users as u', 'u.id', 'p.author_id')
  .select('p.id', 'p.title', 'u.name as author');

// left join — users with optional posts
await knex('users as u')
  .leftJoin('posts as p', 'p.author_id', 'u.id')
  .select('u.id', 'u.name', 'p.title');

// compound ON
await knex('orders as o')
  .join('payments as pm', function () {
    this.on('pm.order_id', '=', 'o.id')
        .andOn('pm.status', '=', knex.raw('?', ['captured']));
  });

// self-join — reports-to manager
await knex('employees as e')
  .leftJoin('employees as m', 'e.manager_id', 'm.id')
  .select('e.name', 'm.name as manager');
```

### Aggregating across a join

```javascript
await knex('users as u')
  .leftJoin('posts as p', 'p.author_id', 'u.id')
  .select('u.id', 'u.name')
  .count('p.id as post_count')
  .groupBy('u.id', 'u.name')
  .havingRaw('count(p.id) > ?', [5])
  .orderBy('post_count', 'desc');
```

### Pitfalls
- Forgetting table aliases on ambiguous columns → runtime SQL error
- Using `.where()` instead of `.on()` inside the join callback
- Cross-join by accident: missing `ON` in the callback form

---

## Slide 10 — Aggregates & Grouping

Knex covers the standard aggregate functions and a few PG-specific helpers. For anything truly bespoke, drop down to `knex.raw`; the rest of the chain still works around it.

```javascript
await knex('orders').count('* as total').first();
await knex('orders').countDistinct('user_id as buyers');

await knex('orders')
  .select('user_id')
  .sum('total as revenue')
  .min('created_at as first_order')
  .max('created_at as last_order')
  .groupBy('user_id');

await knex('orders')
  .select('user_id')
  .sum('total as revenue')
  .groupBy('user_id')
  .havingRaw('SUM(total) > ?', [1000]);
```

### Window functions (raw)

```javascript
await knex('posts')
  .select('id', 'title', 'author_id')
  .select(knex.raw(
    'row_number() over (partition by author_id ' +
    'order by created_at desc) as rn'
  ));
```

### PG upserts

```javascript
await knex('users')
  .insert({ email: 'a@x', name: 'Alice' })
  .onConflict('email').merge({ name: 'Alice' });

await knex('users')
  .insert({ email: 'a@x', name: 'Alice', source: 'csv' })
  .onConflict('email').merge(['name']);
```

### String & date helpers
- `knex.fn.now()` — `CURRENT_TIMESTAMP` across dialects
- `knex.fn.uuid()` — `gen_random_uuid()` on PG, `UUID()` on MySQL 8
- `knex.ref('users.id')` — safe identifier reference

---

## Slide 11 — INSERT, UPDATE, DELETE — with RETURNING

Mutating queries follow the same chainable shape. `returning()` is the single most useful flag — it skips a round-trip to read back the new row.

### INSERT

```javascript
const [user] = await knex('users')
  .insert({ name: 'Alice', email: 'a@dev.io' })
  .returning(['id', 'created_at']);

await knex('users').insert([
  { name: 'Bob',   email: 'b@dev.io' },
  { name: 'Carol', email: 'c@dev.io' }
]);

await knex('users')
  .insert({ email: 'a@dev.io', name: 'Alice 2' })
  .onConflict('email').merge();

await knex('audit_log')
  .insert(rows).onConflict('event_id').ignore();
```

> **MySQL caveat.** MySQL < 8 doesn't support `RETURNING`. Knex uses `LAST_INSERT_ID()` for the auto-increment column only — for the full row, do a follow-up `SELECT`.

### UPDATE

```javascript
const [updated] = await knex('users')
  .where('id', 42)
  .update({
    name: 'Alice Smith',
    last_login: knex.fn.now()
  })
  .returning('*');

await knex('counters').where('id', 1).increment('hits', 1);
await knex('inventory').where('product_id', 99).decrement('stock', 1);

const rowCount = await knex('users')
  .where('verified', false)
  .where('created_at', '<', knex.raw("now() - interval '30 days'"))
  .update({ status: 'expired' });
```

### DELETE

```javascript
await knex('sessions').where('expires_at', '<', knex.fn.now()).del();
await knex('users').where('id', 42).update({ deleted_at: knex.fn.now() });
await knex('temp_uploads').truncate();
```

---

## Slide 12 — Sub-queries, CTEs & UNION

Composition pays off here: any builder can be passed as a sub-query, and CTEs are written with `.with()`. The same Knex chain that produces a row set can also produce a `FROM` clause.

### Sub-queries inline

```javascript
await knex('users').where('id', '=',
  knex('orders').max('user_id').where('total', '>', 1000)
);

await knex('users').whereExists(
  knex('orders').whereRaw('orders.user_id = users.id')
);

await knex('users').whereIn('id',
  knex('orders').select('user_id').where('total', '>', 100)
);

const top = knex('orders')
  .select('user_id').sum('total as revenue')
  .groupBy('user_id').as('t');

await knex('users as u')
  .innerJoin(top, 'u.id', 't.user_id')
  .select('u.name', 't.revenue')
  .orderBy('t.revenue', 'desc').limit(10);
```

### CTEs — `WITH ...`

```javascript
await knex
  .with('recent_orders',
    knex('orders').where('created_at', '>',
      knex.raw("now() - interval '7 days'")))
  .select('user_id').count('* as orders_7d')
  .from('recent_orders').groupBy('user_id');

// recursive
await knex.withRecursive('chain',
  knex('employees').select('id', 'manager_id', 'name')
       .where('id', 42)
       .unionAll(qb => qb.select('e.id','e.manager_id','e.name')
                       .from('employees as e')
                       .innerJoin('chain as c', 'c.manager_id', 'e.id'))
).select().from('chain');
```

### UNION / INTERSECT / EXCEPT

```javascript
await knex('archived_users').select('id','email')
  .union(knex('users').select('id','email').where('active', false));

await q1.unionAll([q2, q3]);
```

This is where Knex shines and most ORMs stumble — window functions, recursive CTEs, derived tables and complex aggregations all stay first-class. You don't drop into raw strings; you compose builders.

---

## Slide 13 — `knex.raw` & Safe Bindings

Sometimes the chain doesn't go far enough — database-specific functions, vendor extensions, full-text search. `knex.raw` is the escape hatch — with **positional**, **named** and **identifier** bindings to keep parameterisation intact.

### Three binding styles

```javascript
// ?  — positional value binding
knex.raw('select * from users where id = ?', [42]);

// :name  — named binding
knex.raw('select * from users where role = :role', { role: 'admin' });

// ??  — identifier binding (table / column)
knex.raw('select ?? from ??', ['email', 'users']);

// mixed
knex.raw('update ?? set ?? = ? where ?? = :id',
         ['users', 'name', 'Alice', 'id', { id: 42 }]);
```

### Never do this

```javascript
// 🚨 SQL injection — user input concatenated
knex.raw(`select * from users where email = '${input}'`);

// ✅ parameterised
knex.raw('select * from users where email = ?', [input]);
```

### Raw inside a chain

```javascript
await knex('events')
  .select('user_id', knex.raw('count(*) filter (where kind = ?) as logins',
                              ['login']))
  .groupBy('user_id');

await knex('users')
  .whereRaw('lower(email) = ?', [email.toLowerCase()]);

await knex('users').orderByRaw('last_login desc nulls last');
```

### Useful raw companions
- `knex.ref('users.email')` — quoted identifier; reusable
- `knex.fn.now()`, `knex.fn.uuid()` — portable functions
- `.toSQL()`, `.toString()` — render SQL for logging or tests

### Full-text example (PG)

```javascript
await knex('articles')
  .select('id', 'title')
  .whereRaw("to_tsvector('english', body) @@ plainto_tsquery(?)", [query])
  .orderByRaw("ts_rank(to_tsvector('english', body), " +
              "plainto_tsquery(?)) desc", [query]);
```

---

## Slide 14 — Schema Builder

The same JS API that builds queries also builds **DDL**. Knex translates a column-by-column description into the right `CREATE TABLE` for each dialect, including index syntax and FK clauses.

```javascript
await knex.schema.createTable('users', (t) => {
  t.increments('id').primary();
  t.string('email', 320).notNullable().unique();
  t.string('name').notNullable();
  t.specificType('role', 'user_role').notNullable().defaultTo('user');
  t.boolean('active').notNullable().defaultTo(true);
  t.jsonb('metadata').defaultTo('{}');
  t.timestamp('last_login_at');
  t.timestamps(true, true);            // created_at, updated_at

  t.index(['active', 'role']);
});

await knex.schema.createTable('posts', (t) => {
  t.uuid('id').primary().defaultTo(knex.fn.uuid());
  t.text('title').notNullable();
  t.text('body');
  t.integer('author_id').unsigned().notNullable()
   .references('id').inTable('users').onDelete('CASCADE');
  t.timestamps(true, true);
});
```

### Altering, dropping

```javascript
await knex.schema.alterTable('users', (t) => {
  t.string('handle', 60).unique();
  t.dropColumn('legacy_token');
  t.renameColumn('name', 'display_name');
  t.string('email', 320).notNullable().alter();
});

await knex.schema.renameTable('users', 'accounts');
await knex.schema.dropTableIfExists('temp_uploads');
```

### Column type cheat sheet

| Knex                    | PG               | MySQL                     |
|-------------------------|------------------|---------------------------|
| `increments()`          | `serial`         | `int auto_increment`      |
| `bigIncrements()`       | `bigserial`      | `bigint auto_increment`   |
| `uuid()`                | `uuid`           | `char(36)`                |
| `text()`                | `text`           | `longtext`                |
| `jsonb()`               | `jsonb`          | `json`                    |
| `timestamps(true,true)` | `timestamptz NOT NULL` | `timestamp NOT NULL` |

> **Don't fight your dialect.** Use `specificType()` for vendor types (PG enums, `tsvector`, `citext`) — portability already broke the moment you used them, so make it explicit.

---

## Slide 15 — Migrations

Migrations are **version control for your schema**. Knex tracks which have been applied in a `knex_migrations` table; migrations run in **timestamp order**, in batches, and each is wrapped in a transaction (where the dialect allows).

### Workflow

```bash
npx knex migrate:make add_posts_table
# → db/migrations/20260506T120000_add_posts_table.js

npx knex migrate:latest    # apply pending
npx knex migrate:rollback  # undo last batch
npx knex migrate:status    # what is applied / pending
npx knex migrate:list
npx knex migrate:up        # one forward
npx knex migrate:down      # one back
npx knex migrate:unlock    # if a migration crashed mid-run
```

### The migration file

```javascript
// 20260506T120000_add_posts_table.js
exports.up = async function (knex) {
  await knex.schema.createTable('posts', (t) => {
    t.increments('id').primary();
    t.string('title').notNullable();
    t.text('body');
    t.integer('author_id').unsigned()
     .references('id').inTable('users').onDelete('CASCADE');
    t.timestamps(true, true);
    t.index(['author_id', 'created_at']);
  });
};

exports.down = async function (knex) {
  await knex.schema.dropTable('posts');
};

// optional — isolates this migration's transaction
exports.config = { transaction: false };
```

### Rules of the road
- Filenames sort lexicographically — the timestamp prefix is load-bearing
- Once a migration is committed and shipped, it is **immutable** — any change goes in a new file
- `up` and `down` are mirror images — if `down` can't unwind `up`, document why
- Avoid mixing schema changes and bulk data updates in one migration — the lock is held for the whole transaction
- For long backfills: **expand → migrate → contract** across deploys

### Production gotchas
- PG: certain DDL (e.g. `CREATE INDEX CONCURRENTLY`) **cannot** run inside a transaction — set `config.transaction = false`
- MySQL: implicit commit on DDL means rollback is partial — design your `down` accordingly
- `knex_migrations_lock.is_locked = 1` after a crash — `migrate:unlock` clears it
- Adding a NOT NULL column to a large table without a default rewrites the whole table — **avoid in prod**

### CI / CD
Run `migrate:latest` as a release-time job (not at app startup), gate it behind a health check, and fail the deploy if `migrate:status` shows drift.

---

## Slide 16 — Seeds & Fixtures

Seeds populate a database with **development** or **test** data. They run unordered by filename, idempotently — clearing and repopulating — and never depend on production data.

```bash
npx knex seed:make 01_users
npx knex seed:make 02_posts
npx knex seed:run
npx knex seed:run --specific=01_users.js
```

### A seed file

```javascript
// db/seeds/development/01_users.js
exports.seed = async function (knex) {
  await knex('posts').del();
  await knex('users').del();

  await knex('users').insert([
    { id: 1, name: 'Alice', email: 'alice@dev.io', role: 'admin' },
    { id: 2, name: 'Bob',   email: 'bob@dev.io',   role: 'user'  },
    { id: 3, name: 'Carol', email: 'carol@dev.io', role: 'user'  },
  ]);
};
```

### Faker for volume

```javascript
const { faker } = require('@faker-js/faker');

exports.seed = async function (knex) {
  await knex('users').del();
  const rows = Array.from({ length: 1_000 }, () => ({
    name:  faker.person.fullName(),
    email: faker.internet.email().toLowerCase(),
    role:  faker.helpers.arrayElement(['user', 'admin'])
  }));
  await knex.batchInsert('users', rows, 200);
};
```

### Best practice
- **Idempotent** — safe to run twice
- **Deterministic** — `faker.seed(42)` for reproducible test data
- Separate `seeds/development/` from `seeds/test/` via two configs in the knexfile
- Never seed in production — if you need reference data there, use a migration
- Honour FK constraints — delete children before parents, insert parents first

> **Pitfall.** Seed files run in *alphabetical* order, not creation order. Use a numeric prefix — `01_users.js`, `02_posts.js` — or you'll hit FK violations on the first run after rename.

---

## Slide 17 — Transactions, Savepoints & Isolation

Knex transactions are **scope-bound**: pass the `trx` handle wherever you'd pass `knex`. The promise's resolution commits; a thrown error rolls back. Nested transactions become **savepoints** automatically.

### Callback-style (auto-commit)

```javascript
await knex.transaction(async (trx) => {
  const [order] = await trx('orders')
    .insert({ user_id: 1, total: 99.99 })
    .returning('*');

  await trx('inventory')
    .where('product_id', 42)
    .decrement('stock', 1);

  await trx('payments').insert({
    order_id: order.id, amount: 99.99
  });
});  // throws => ROLLBACK · resolves => COMMIT
```

### Manual control

```javascript
const trx = await knex.transaction();
try {
  await trx('users').insert({ ... });
  await trx('audit_log').insert({ ... });
  await trx.commit();
} catch (e) {
  await trx.rollback();
  throw e;
}
```

### Savepoints

```javascript
await knex.transaction(async (trx) => {
  await trx('users').insert(user);

  // nested => SAVEPOINT
  try {
    await trx.transaction(async (sp) => {
      await sp('subscriptions').insert(plan);
      throw new Error('plan invalid');
    });
  } catch { /* savepoint rolled back */ }

  await trx('events').insert({ kind: 'user_created' });
});
```

### Isolation level

```javascript
await knex.transaction(
  async (trx) => { /* ... */ },
  { isolationLevel: 'serializable' } // PG, MySQL, MSSQL
);
```

- `read uncommitted` · `read committed` · `repeatable read` · `serializable`
- PG/MSSQL default: **read committed**. MySQL InnoDB: **repeatable read**
- Use `serializable` for money & quotas; expect `40001` retries

### Pitfalls
- Mixing `knex(...)` and `trx(...)` in one transaction — the bare call uses the pool, escapes the txn, and deadlocks
- Long-running transactions — one slow HTTP call inside `BEGIN ... COMMIT` can block writers for seconds

---

## Slide 18 — Streams & Batching

For million-row exports or imports, materialising the result set blows up memory. Knex offers **cursor-style streams** for reads and **`batchInsert`** for writes — both pool-aware.

### Streaming reads

```javascript
// pg: server-side cursor; mysql2: row-by-row
const stream = knex('events')
  .select()
  .where('created_at', '>', cutoff)
  .stream();

for await (const row of stream) {
  await processOne(row);     // back-pressure honoured
}

// pipe to a CSV stream
const fastCsv = require('fast-csv');
knex('events').stream()
  .pipe(fastCsv.format({ headers: true }))
  .pipe(fs.createWriteStream('events.csv'));
```

### Pagination patterns
- **Offset/limit** — simple, but slows for deep pages (DB scans rows it then discards)
- **Keyset (seek)** — use the last row's sort key as `WHERE`; constant-time per page

```javascript
await knex('posts')
  .where('created_at', '<', lastSeenCreatedAt)
  .orderBy('created_at', 'desc').limit(50);
```

### Batched writes

```javascript
await knex.batchInsert('events', rows, 500).returning('id');

await knex.transaction(async (trx) => {
  await knex.batchInsert('events', rows, 500).transacting(trx);
});
```

### Bulk update / chunked delete

```javascript
// PG: bulk update via VALUES + JOIN
await knex.raw(`
  UPDATE users AS u
     SET name = v.name
    FROM (VALUES ${placeholders}) AS v(id, name)
   WHERE v.id = u.id
`, flatBindings);

let from = 0; const chunk = 10_000;
while (true) {
  const n = await knex('logs')
    .whereBetween('id', [from, from + chunk]).del();
  if (n === 0) break;
  from += chunk;
}
```

> **Avoid** calling `.then()` on a giant un-paged `.select()` — the result set lands in Node's heap before you process row one.

---

## Slide 19 — TypeScript with Knex

Knex ships official types. The big move is **declaration merging** on `Knex.Tables` — once your tables are typed, every chain inherits column-level safety on inputs and result rows.

### Declare your tables

```typescript
// db/types.ts
import { Knex } from 'knex';

interface User {
  id: number;
  email: string;
  name: string;
  role: 'user' | 'admin';
  active: boolean;
  created_at: Date;
}

declare module 'knex/types/tables' {
  interface Tables {
    users: User;
    users_composite: Knex.CompositeTableType<
      User,                                 // result
      Pick<User, 'email' | 'name'>,         // insert
      Partial<Pick<User, 'name' | 'role'>>  // update
    >;
  }
}
```

### Use it

```typescript
// rows is User[] — no any
const rows = await knex('users').where({ active: true });

await knex('users_composite').insert({
  email: 'a@x', name: 'A'
});

await knex('users_composite').where({ id: 1 })
  .update({ name: 'A2' });
```

### Validators at the boundary

```typescript
import { z } from 'zod';

const UserSchema = z.object({
  id: z.number(),
  email: z.string().email(),
  name: z.string(),
  role: z.enum(['user', 'admin']),
});

const rows = await knex('users').where({ id });
const safe = z.array(UserSchema).parse(rows);
```

### Limits
Knex types are *structural*, not *generated* — `knex.raw` result types are `any`, ad-hoc joins lose typing, and there's no compile-time SQL syntax check. If you need that, look at **Kysely** or **Drizzle**.

---

## Slide 20 — Testing Database Code

Tests need **isolation**, **speed** and **fidelity**. Knex's dialect-portability and migrations give you three good strategies; pick one per layer.

### Strategy A — in-memory SQLite

```javascript
const knex = require('knex')({
  client: 'better-sqlite3',
  connection: { filename: ':memory:' },
  useNullAsDefault: true
});

beforeAll(async () => {
  await knex.migrate.latest();
  await knex.seed.run();
});
afterAll(() => knex.destroy());
```

Fast (millisecond per test). Lower fidelity — SQLite differs from PG on JSON, regex, locking and FK enforcement.

### Strategy B — transaction rollback

```javascript
let trx;
beforeEach(async () => { trx = await knex.transaction(); });
afterEach (async () => { await trx.rollback(); });

test('creates a user', async () => {
  await usersRepo.create(trx, { email: 'a@x' });
  expect(await usersRepo.find(trx, 'a@x')).toBeDefined();
});
```

Real Postgres, near-zero teardown cost — but tests must thread `trx` through.

### Strategy C — Testcontainers

```javascript
const { GenericContainer } = require('testcontainers');

const pg = await new GenericContainer('postgres:16')
  .withEnvironment({ POSTGRES_PASSWORD: 'pw' })
  .withExposedPorts(5432).start();

const knex = require('knex')({
  client: 'pg',
  connection: {
    host: pg.getHost(),
    port: pg.getMappedPort(5432),
    user: 'postgres', password: 'pw', database: 'postgres'
  }
});
await knex.migrate.latest();
```

Highest fidelity — one container per worker; CI cost goes up but flake goes down.

### Tips that apply to all three
- Use `knex.migrate.latest()`, not raw SQL fixtures — tests catch migration drift
- Always `knex.destroy()` in `afterAll` — otherwise Jest hangs
- Factory functions, not shared mutable seed rows — each test owns its data
- Run unit tests against SQLite, integration tests against PG

---

## Slide 21 — Performance & Gotchas

Knex is a thin layer — the SQL it generates is the SQL the DB plans. Most "Knex is slow" reports are **missing indexes**, **N+1 loops**, or **pool starvation**.

### Inspect what's running

```javascript
const { sql, bindings } = q.toSQL().toNative();
console.log(sql, bindings);

knex.on('query', e => console.log(e.__knexUid, e.sql, e.bindings));
knex.on('query-response', (rows, e) =>
  console.log(e.__knexUid, 'rows:', rows.length));
knex.on('query-error', (err, e) => console.error(e.sql, '\n  ', err));
```

### The N+1 trap

```javascript
// BAD — 1 + N queries
const users = await knex('users');
for (const u of users) {
  u.posts = await knex('posts').where('author_id', u.id);
}

// GOOD — one query, group in JS
const ids = users.map(u => u.id);
const posts = await knex('posts').whereIn('author_id', ids);
const byAuthor = Object.groupBy(posts, p => p.author_id);
for (const u of users) u.posts = byAuthor[u.id] ?? [];
```

### Quick wins
- Index every FK column — especially the child side of one-to-many
- Composite indexes for filter+sort pairs — `(status, created_at desc)`
- `EXPLAIN ANALYZE` the slowest 10 endpoints; act on Seq Scans on big tables
- Use `.first()` when expecting one row
- Set `statement_timeout` on the PG side to kill runaways

### Common pitfalls
- Returning huge result sets — paginate or stream
- Holding transactions across HTTP calls — one slow downstream stalls writers
- Mismatched pool size — `max` > database `max_connections / N` stalls under load
- `orderBy('random()')` — full sort; use `tablesample` on PG
- String comparison on indexed JSONB — cast or use `jsonb_path_ops`

### Observability
Wire `knex.on('query', ...)` to your logger with `req_id`; export pool gauges (used / free / pending) to Prometheus; alert on `numPendingAcquires` > threshold for > 30s.

---

## Slide 22 — Knex vs Prisma, Drizzle, Kysely, TypeORM, raw

The Node.js DB landscape grew up around Knex; today there are stronger alternatives *per axis*. The right choice depends on what you're optimising for.

| Criterion        | Raw (pg)        | Knex                          | Kysely         | Drizzle          | Prisma                | TypeORM             |
|------------------|-----------------|-------------------------------|----------------|------------------|-----------------------|---------------------|
| Abstraction      | None            | Query builder                 | Query builder  | Query builder + ORM | ORM                | ORM                 |
| Type safety      | None            | Manual decls                  | Excellent      | Excellent        | Excellent (codegen)   | Decorator-based     |
| SQL fidelity     | Full            | High                          | Full           | Full             | Medium                | Medium              |
| Migrations       | Manual          | Built-in                      | Built-in       | Built-in         | Built-in (best)       | Built-in            |
| Dialects         | One driver      | PG/MySQL/SQLite/MSSQL/Oracle  | PG/MySQL/SQLite| PG/MySQL/SQLite  | PG/MySQL/SQLite/Mongo | ~10 dialects        |
| Bundle size      | Tiny            | Small                         | Small          | Small            | Heavy (Rust engine)   | Medium              |
| Maturity         | Battle-hardened | Mature (2013)                 | Active         | Active           | Active                | Stagnant            |
| Best for         | Hot paths       | SQL-first, multi-dialect      | SQL-first + types | Edge / Cloudflare | CRUD, TS-first apps | Legacy AR-style apps |

### Choose Knex when
- You think in SQL and want SQL on the wire
- Multi-dialect — SQLite for tests, PG in prod
- Migrations & seeds matter and you don't want a separate tool
- You're building a higher-level abstraction (Objection.js-style)

### Choose Kysely when
- You want Knex-style SQL plus full TypeScript inference
- Type-safe joins and result inference matter
- PG / MySQL / SQLite is enough — no Oracle / MSSQL

### Choose Prisma / Drizzle when
- Greenfield TypeScript project, CRUD-heavy
- You want generated types from schema
- Drizzle: edge runtimes (Cloudflare, Bun, Deno)

---

## Slide 23 — Production Patterns

Knex in a real app is one shared instance, threaded through repositories, observed at the pool level, and wrapped in a **retry-on-transient** envelope.

### Repository pattern

```javascript
// db/repositories/users.js
module.exports = (db) => ({
  byEmail: (email)  => db('users').where({ email }).first(),
  create:  (input)  => db('users').insert(input).returning('*'),
  update:  (id, p)  => db('users').where({ id }).update(p),
  destroy: (id)     => db('users').where({ id }).del(),
});

// app.js
const knex = require('./db');
const users = require('./db/repositories/users')(knex);
app.get('/users/:email',
  async (req, res) => res.json(await users.byEmail(req.params.email)));
```

### Express + Knex

```javascript
function createApp(db) {
  const app = express();
  app.locals.db = db;
  app.get('/users/:id', async (req, res, next) => {
    try {
      const u = await db('users').where('id', req.params.id).first();
      if (!u) return res.sendStatus(404);
      res.json(u);
    } catch (err) { next(err); }
  });
  return app;
}

process.on('SIGTERM', async () => {
  server.close(); await knex.destroy();
});
```

### Retries on transient errors

```javascript
async function withRetry(fn, { attempts = 3, base = 100 } = {}) {
  for (let i = 0; i < attempts; i++) {
    try { return await fn(); }
    catch (e) {
      const transient = ['40001', '40P01', '57P01'].includes(e.code) ||
        ['ECONNRESET', 'ETIMEDOUT'].includes(e.code);
      if (!transient || i === attempts - 1) throw e;
      const delay = base * 2 ** i + Math.random() * base;
      await new Promise(r => setTimeout(r, delay));
    }
  }
}

await withRetry(() => knex.transaction(transferFunds));
```

### Observability checklist
- **Pool gauges** → Prometheus: used / free / pending acquires / pending creates
- **Slow-query log** — `knex.on('query-response', ...)` with timestamps; export histogram
- **Trace context** — pass `req_id` through to every query for correlation
- **Migration drift** — CI gate on `migrate:status`
- **Health probe** — `SELECT 1` via Knex on `/healthz`; never expose pool stats publicly
- Set `statement_timeout`, `idle_in_transaction_session_timeout` at the DB

> Don't catch every DB error and swallow it — constraint violations are signals from the schema, not bugs in *your* code.

---

## Slide 24 — Summary & Next Steps

### Where Knex earns its keep
- **SQL stays SQL** — you compose it, the DB plans it
- **One toolbox** for queries, schema, migrations, seeds, transactions, streams
- **Multi-dialect** — SQLite for tests, PG/MySQL in prod, MSSQL/Oracle when forced
- Battle-tested for over a decade; stable surface; small surface area
- Excellent escape hatch via `knex.raw` — rest of the chain still composes

### Where it falls short
- Type safety is hand-rolled — Kysely / Drizzle do better
- No relational object graph — if you want `user.posts`, look at Prisma
- v3 is in progress — pace of change is slower than newer entrants

### Take-aways
- One Knex instance per process; one pool; `knex.destroy()` on shutdown
- Always parameterise — `?`, `:name`, `??` — never concat user input
- Migrations are immutable once shipped; expand → migrate → contract for big tables
- Index every FK; index the columns you sort by; `EXPLAIN ANALYZE` the top 10
- Use `.toSQL()` — understanding the SQL is half the battle

### Next steps
- Build a small CRUD service: Express + Knex + PG, with migrations & seeds
- Set up a TS project with declaration-merged `Knex.Tables`
- Add a Testcontainers integration suite alongside SQLite unit tests
- Profile with `knex.on('query-response')`; add a slow-query histogram
- Try **Objection.js** for a model layer over Knex, or **Kysely** for a fully-typed alternative

### Essential reading
- [knexjs.org/guide](https://knexjs.org/guide/) — the canonical reference
- [github.com/knex/knex](https://github.com/knex/knex) — source & changelog
- [PostgreSQL docs](https://www.postgresql.org/docs/current/index.html) — the dialect Knex was first written for
- [kysely.dev](https://kysely.dev/) — the type-safe sibling worth knowing
