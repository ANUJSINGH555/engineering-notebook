## 2026-08-23 · Session 2 · ledger-engine

**Did:** Fixed .gitignore (node_modules was committed), renamed test/ → tests/,
got Docker Compose running Postgres 16, wrote the migration runner with
transactional apply, wrote the seed script. Full pipeline works: compose up →
migrate → seed → 9 accounts in the DB.

**Learned:**
- Docker doesn't ship with Postgres — the image comes from Docker Hub. Image =
  template, container = running instance.
- Postgres env vars (POSTGRES_DB etc.) only apply on FIRST boot. Need
  `docker compose down -v` to wipe the volume and re-init.
- `await` only pauses the function it's written in. Inside a `forEach` callback
  it pauses the callback, not the loop — so all iterations fire concurrently.
  `for...of` actually waits. Wrong tool for migrations.
- `fs.readdirSync` returns bare filenames, not paths — hence ENOENT until I
  joined with the directory.
- Migration SQL and the `schema_migrations` insert must be in the SAME
  transaction, or a crash between them leaves a migration applied but unrecorded.
- The principle underneath: Two facts that must always agree should be written together, atomically. Never separately.
- Parameterized queries: `client.query(sql, [a, b, c])` — one array, positional.
- psql needs a semicolon. Prompt shows `ledger-#` when a statement is unfinished.

**Felt:** Struggled with syntax a lot tonight and it was demotivating — 4 years
in and fumbling on basics. Naming it here because it's worth tracking. Six
unfamiliar surfaces in one session (Docker, raw pg, fs API, git edge cases,
migrations, psql) is a lot at once, and I debugged every issue I hit.

**Stuck on:** Nothing blocking.

**Next session:** POST /transfers happy path — the transfer inside a real
DB transaction, plus GET balance.
