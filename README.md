# Optional browser dispatch console
Standalone Node.js 20+ service; existing FiveM resource remains intact. Uses the same Spillman SQL tables for active calls, live units, assignments and audit log.

## Network safety
The example environment binds HOST=0.0.0.0 so the app can accept connections on the container network, and sets PUBLIC_BASE_URL=http://193.31.31.158:25978 for the requested public page links. Confirm with RocketNode that port 25978 is available for web traffic and that inbound access is allowed. Direct HTTP is unencrypted; for production remote dispatch, use HTTPS via a trusted reverse proxy or VPN. Never expose MySQL publicly. DB_HOST=127.0.0.1 is intentionally unchanged when MySQL is on the same container.

## Setup
1. Install Node.js 20+ on a host where this service can run.
2. Copy `.env.example` to `.env`; fill in database settings and secrets.
3. `npm install`
4. Generate a bcrypt password hash with `node tools/hash-password.js "a-unique-password-at-least-14-chars"` and set `DISPATCH_PASSWORD_HASH`.
5. Use a dedicated MySQL account limited to required CAD tables; no admin/GRANT/DROP privileges.
6. Ensure Spillman SQL and `upgrade-0.1.7-units-gps.sql` are imported.
7. `npm start`. Test locally first. Only publish behind HTTPS/VPN after the hosting provider confirms the web port.
8. Do not share `.env` or database credentials.

Features: login, active call list, create calls, status updates, live unit list, unit assignment, audit records. This first browser version does not yet contain person/arrest/admin screens or a calibrated tile map. Concurrent call-number allocation needs hardening before production use; do not treat as production-ready until reviewed and tested.


## Copperhead RP Admin on the same port
The admin console is part of the same Express application as browser dispatch. Open `/admin` on the same host and port; no second server or port is needed. It shares the session store and CAD database, but uses separate administrator credentials.

1. Set `ADMIN_USERNAME` in `.env`.
2. The example config sets `ADMIN_PASSWORD=2026` because that was requested. This is a very weak password and should only be used temporarily; change it before public access. For stronger security, generate a bcrypt hash with `node tools/hash-password.js "your-long-unique-admin-password"`, set `ADMIN_PASSWORD_HASH`, and remove `ADMIN_PASSWORD`.
3. Restart the Node app and open `http(s)://your-host:25978/admin` (only if RocketNode confirms that port is available for web traffic).
4. Admin currently provides system/table overview and recent audit records. User/role editing and department/response-profile editors are not implemented yet.

Do not expose plain HTTP logins publicly. Use a supported HTTPS reverse proxy or VPN and restrict the database account to the needed CAD tables. The Admin API rejects requests without an administrator session.

## SOAP EMS Patient Care Reports

1. Back up the database and import `sql/soap-pcr.sql` into the configured CAD database.
2. Restart the Node dispatch web application.
3. Sign in to the existing dispatch web app and open `/pcr` on the same host and port (for example `http://HOST:25978/pcr`).
4. Create a PCR, complete Subjective, Objective, Assessment, Plan, vitals, treatments, narrative, and disposition. Save draft or submit for review.
5. Administrator sessions can approve, return for correction, or reject submitted reports. Review actions are written to `cad_audit_log`.

The PCR form stores SOAP sections and timestamped vital-sign sets as JSON columns. This implementation is roleplay CAD documentation, not a certified ePCR/EMS clinical record system. Review access is currently administrator-only; department-level supervisor permissions are not yet implemented. Use HTTPS/VPN for remote access and restrict database credentials to required tables.


### Sprint unit SOAP/PCR workflow
Sprint units may author and submit SOAP Patient Care Reports as paramedic resources. The PCR form includes an Operational Role field; select `Sprint — Primary: Paramedic` when documenting patient care. `Sprint — Secondary: EMS Supervisor` is available when documenting a supervisory role. A Sprint unit's supervisor designation does not prevent it from documenting as the primary paramedic resource. This is roleplay CAD workflow configuration and does not itself grant real-world clinical authority.


## Law-enforcement Use of Force reports (added in v0.3.2)

Import `sql/use-of-force.sql` into the same MySQL database configured for `dispatch-web`. Then open `/uof` on the same browser CAD host and sign in with a dispatch account to create drafts and submit reports. Administrator sessions can approve, return, or reject submitted reports. Reports are linked to the originating CAD incident and use `UOF-<CAD incident number>` as the report number (a suffix is added when multiple UOF reports reference the same incident). The form is based on the supplied two-page Use of Force form and is available to all listed law-enforcement agencies. This is a roleplay CAD workflow; adapt policy labels and review requirements to your server rules.


## Startup and connection diagnostics (v0.3.5)

When the Node application starts, its console prints `dispatch-web LOADED`, then reports whether the MySQL connection succeeded and whether the core CAD tables exist. It rechecks the database every 60 seconds. The read-only `/health` endpoint returns HTTP 200 when the database responds and HTTP 503 when the database check fails; it never returns credentials.

The FiveM resource separately prints its resource name, dependency states, MySQL `SELECT 1` result, and whether the core CAD tables are present in the **FiveM server console**. A successful `SELECT 1` means the connection works; missing table warnings mean the SQL migrations still need to be imported.

## Startup console diagnostics (v0.3.7)

When the Node service successfully binds its configured port, the console prints a startup report with the actual bound address and port, a listening/port status, and the Dispatch CAD, Admin, PCR, Use of Force, and health URLs. It also lists the principal API handler groups. If the port is already in use or binding is denied, the console prints an explicit `LISTENING: NO` error.

`PUBLIC_BASE_URL` controls the page links printed at startup. The example is configured as `http://193.31.31.158:25978`. Replace it with your secured HTTPS hostname if you configure a reverse proxy. `HOST=0.0.0.0` binds on all container interfaces; it does not by itself prove firewall, port-forwarding, or public internet reachability. Keep `DB_HOST=127.0.0.1` if MySQL runs in the same container; do not replace the database host with the public IP.


## Automatic startup under RocketNode
The bundled parent resource includes `../start-dispatch-web.sh`, a Linux supervisor for this Node app. Configure RocketNode's custom/pre-start hook to launch it in the background before the existing FiveM startup command. See `../README.md` for the full instructions. FiveM itself cannot launch a Node process using an fxmanifest dependency.
