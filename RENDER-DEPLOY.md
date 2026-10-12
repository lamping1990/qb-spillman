# Spillman CAD — Render deployment package

This is the standalone browser dispatch web app extracted from the qb-spillman v0.4.3 package and adjusted for Render. It is not the FiveM resource; keep `qb-spillman` installed on RocketNode for in-game integration.

## Render settings
- Root Directory: leave blank (package.json is in the repository root)
- Runtime: Node
- Build command: `yarn install`
- Start command: `yarn start`
- Health check path: `/health`

If deploying this folder from a monorepo instead of uploading its contents to the repository root, set Root Directory to the folder containing this package.json.

## Required environment variables
Set these in Render Dashboard > your service > Environment. Do not commit a real `.env` file.
- `HOST=0.0.0.0`
- `TRUST_PROXY=true`
- `SESSION_SECRET`: a unique random secret of at least 32 characters (Render can generate one)
- `DISPATCH_USERNAME`: dispatcher login name
- `DISPATCH_PASSWORD_HASH`: bcrypt hash for dispatcher password
- `ADMIN_USERNAME`: admin login name
- `ADMIN_PASSWORD`: unique strong password; preferably use `ADMIN_PASSWORD_HASH` instead and remove `ADMIN_PASSWORD`
- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`: RocketNode MySQL details
- `PUBLIC_BASE_URL`: your `https://<service>.onrender.com` URL after Render creates it

`PORT` is assigned by Render automatically. Do not set it to the RocketNode port. Render serves the site on its own `onrender.com` hostname; it will not use `193.31.31.158:25978` directly.

## Before deployment
1. Verify RocketNode permits remote MySQL connections from Render. If the DB is not reachable from outside RocketNode, deployment can start but CAD database features and `/health` will fail.
2. Rotate any database password previously exposed in chat/logs and use the new value only in Render environment variables.
3. Create a bcrypt password hash for `DISPATCH_PASSWORD_HASH`. You can generate it locally after `yarn install` with `node tools/hash-password.js "a-unique-long-password"`. Do not commit the hash if you prefer to keep credentials out of Git history; set it directly in Render.
4. Ensure the database has the core Spillman tables and import the SQL migrations for PCR and UOF if those features are needed (`sql/soap-pcr.sql`, `sql/use-of-force.sql`). Back up the database before importing migrations.
5. Deploy and open `/health`. A 200 response indicates MySQL connectivity and core table checks passed; 503 means database/schema needs attention.

## Known hosting limitation
Render's free service may spin down after inactivity and the local SQLite session store may be ephemeral across restarts/redeploys. Users may need to sign in again. This package does not copy or host the FiveM resource itself.
