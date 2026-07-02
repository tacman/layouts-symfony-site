# Netgen Layouts – Symfony Site

Skeleton Symfony 8.1 project bootstrapped with [Netgen Layouts](https://docs.netgen.io/projects/layouts/) 2.0.

## Requirements

- PHP >= 8.4
- Composer
- Docker (for the bundled Postgres + Mailpit services), or your own PostgreSQL 16+ instance

## Setup

1. Install dependencies:

   ```bash
   composer install
   ```

2. Start the local services (Postgres + Mailpit):

   ```bash
   docker compose up -d
   ```

   The `database` service doesn't publish a fixed host port, so Docker assigns a random one. Find it with:

   ```bash
   docker compose port database 5432
   ```

   Put the resulting URL in `.env.local`, e.g.:

   ```
   DATABASE_URL="postgresql://app:!ChangeMe!@127.0.0.1:<PORT>/app?serverVersion=16&charset=utf8"
   ```

   (If you use the [Symfony CLI](https://symfony.com/download), `symfony server:start` / `symfony console` detect the Docker Compose services automatically and you can skip this step.)

3. Create the database and run the app's own migrations:

   ```bash
   php bin/console doctrine:database:create --if-not-exists
   php bin/console doctrine:migrations:migrate --no-interaction
   ```

4. Load the Netgen Layouts Core schema.

   Netgen Layouts Core ships its schema as a raw SQL file rather than a Doctrine migration for PostgreSQL (the migrations bundled in `vendor/netgen/layouts-core/migrations` are legacy, MySQL-only upgrade paths from old `ngbm_*` installs, so they aren't usable for a fresh Postgres setup). Load it directly:

   ```bash
   docker compose exec -T database psql -U app -d app -f - < vendor/netgen/layouts-core/resources/data/schema.pgsql.sql
   ```

   This creates the `nglayouts_*` tables and seeds the root rule group.

   Note: `vendor/netgen/layouts-core/migrations` also ships a Doctrine Migrations config, runnable with:

   ```bash
   php bin/console doctrine:migrations:migrate --configuration=vendor/netgen/layouts-core/migrations/doctrine.yaml
   ```

   These migrations only cover legacy upgrades (versions 0.7.0–1.3.0, old `ngbm_*` tables) and `abortIf` on anything but MySQL, so on this Postgres-based setup they won't run — use the schema file above instead.

5. Start the app:

   ```bash
   symfony server:start
   # or
   php -S 127.0.0.1:8000 -t public
   ```

## Using it

- Homepage: `http://127.0.0.1:8000/`. On a fresh install this renders an empty layout — there are no Layouts or Rules yet, only the root rule group created by the schema above.
- Admin panel: `http://127.0.0.1:8000/nglayouts/admin`, protected by HTTP Basic auth. Default credentials are `admin` / `admin` (see `NETGEN_LAYOUTS_ADMIN_PASSWORD` in `.env`).

  Use the admin panel to create a Layout, then add a Rule (under Mappings) that maps a path/route to that Layout — that's what makes content show up on the frontend.

- Layout editing app (drag-and-drop block/zone editor for a specific layout, opened from the admin panel's Layouts list): `http://127.0.0.1:8000/nglayouts/app`, or with the Symfony CLI:

  ```bash
  symfony open:local --path=/nglayouts/app
  ```

- Mailpit (catches outgoing mail): find its web UI port with `docker compose port mailer 8025`, then open `http://127.0.0.1:<PORT>`.

## Notes

- `config/packages/doctrine.yaml` sets a `schema_filter` that excludes `nglayouts_*` tables (managed separately, see above) as well as any TimescaleDB extension schemas/sequences (`_timescaledb*`, `timescaledb_*`, `toolkit_experimental`), in case you point `DATABASE_URL` at a shared Postgres cluster that has the TimescaleDB extension enabled — without this, `doctrine:migrations:diff` tries to manage the extension's internal objects and fails.
