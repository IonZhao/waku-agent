# Hosted Waku — design

Status: DRAFT design doc. It records decisions and the shape of the system;
the spec and the implementation plan are generated from it separately.

This revision replaces the first one (commit `ee3251a`), which designed a single
process serving every tenant. Narrowing the MVP to "the dashboard, unmodified"
changed that answer; §4.4 explains why, with measurements.

## 1. Summary

One operator runs one Linux VM. Hundreds of people sign in, and each gets their
own waku: the same dashboard as local waku, their own memory, history, settings
and spend. Each active person's waku is a **stock waku process in its own
container**. A small gateway in front handles login, routing, and metering of
model calls. waku itself is not modified, apart from making the dashboard's bind
address configurable.

| Decision | Choice |
|---|---|
| Unit of tenancy | one stock waku process per active tenant, in its own container |
| Tenant data | one directory per tenant on the VM's persistent disk |
| Dashboard | stock and unmodified: all 11 views |
| Login | Supabase Auth as identity provider, then a gateway session cookie |
| Storage systems to operate | one: the filesystem. The control plane is a local SQLite file |
| Platform model key | never enters a tenant container; calls go through a metering proxy |
| Deploy target | one Linux VM with a persistent disk, any provider, one install script |
| MVP scope | the dashboard only; all channels deferred |

## 2. Goals and non-goals

Goals:

- hundreds of registered users on one VM, isolated in data, config and spend
- the dashboard exactly as it runs locally
- two key modes: a platform key with a free tier, or the user's own key (BYOK)
- one person can install, upgrade, back up and move the whole system with scripts
- waku releases flow through without maintaining a fork

Non-goals for the MVP:

- channels: Telegram, Slack, Discord, WhatsApp (§12 explains why they get cheaper later)
- more than one VM (the path is described, not built)
- nested tenants: a tenant is one person, not a business with its own customers
- host-bound features: voice, Apple tools, editor reveal, pi delegation, `gh`
- a new frontend

## 3. What in waku is single-user today

| Assumption | Where |
|---|---|
| one agent per process behind one lock | `waku/ops/browser_agent.py`: module globals `_agent`, `agent_lock` |
| configuration is the process environment; the Connections, Models and Settings pages write `.env` and set `os.environ` | `integrations._write_updates`, `settings_api.apply_settings` |
| no authentication; bound to loopback | `waku/ops/dashboard.py:1159`, `ThreadingHTTPServer(("127.0.0.1", port))` |
| process-wide caches | `integrations._HEALTH`, the model catalog cache (keyed by URL, not key), the Whisper model, the Notion store |
| `.env` is found by walking **up** from the working directory | `waku/config.py`, `find_dotenv(usecwd=True)` |
| 16 references to those globals in `dashboard.py` alone; direct SQL in 14 files | grep over `waku/` |

Every one of these is correct in a process that serves one person. That
observation drives the central decision in §4.4.

## 4. Decisions

### 4.1 A tenant is a directory

Everything waku persists already lives under `WAKU_HOME`: `state.db`, `SOUL.md`,
`skills/`, `traces/`, `usage.jsonl`, `models.json`, `calendar.ics`, `outbox/`.
One directory per tenant is physical isolation, and it is also the unit of
backup, restore and migration.

Rejected: a `tenant_id` column in shared tables. Direct SQL lives in 14 files;
one missing `WHERE` leaks data silently.

### 4.2 One storage system: the filesystem

- **Tenant data**: the tenant's directory, as above.
- **Control plane**: `control.db`, a SQLite file on the same disk: tenants,
  login sessions, proxy tokens (hashed), the spend ledger, plans.
- **Supabase**: Auth only, used as an identity provider rather than as a
  database we operate. User records live in Supabase; we store the user id (`sub`).

Why: one backup, one restore, one migration ("copy `/srv/waku`"); no network hop
on the hot path (the quota check runs before every model call); standard library only.

Rejected:

- the control plane in Supabase Postgres: a second system to operate, for data
  that fits in one file at this scale.
- memory in Supabase: semantic memory already has a swappable backend
  (`FactStore`, with `SupabaseFactStore` written), but `chat_log`,
  `calendar_events`, `SOUL.md` and `skills/` have no abstraction. The disk stays
  mandatory, so this would add a system without removing one.

When this changes: at more than one VM, the tenant-to-VM map and login sessions
must be shared. That is when a shared Postgres earns its place, and it can be
Supabase then.

### 4.3 Deploy target: one VM with a persistent disk

The storage model requires **one long-lived writer per tenant directory, on a
local disk**. SQLite is unsafe over network filesystems because their file
locking is unreliable.

- **Fits**: any provider's VM with an attached disk: GCP Compute Engine with a
  persistent disk, AWS EC2 with EBS, Hetzner, DigitalOcean, Lightsail.
- **Does not fit**: Cloud Run, ECS on Fargate, Kubernetes Deployments. They are
  built to replace containers freely, and their persistent storage options are
  network filesystems (EFS, Filestore, GCS FUSE).
- **ECS on EC2** can be forced to work by pinning the task to one instance with
  a host bind mount. That is a VM with an orchestrator working against you. Not
  recommended.

GCP is fine; Cloud Run was the specific mismatch. The provider does not matter;
the shape (a VM and a disk) does.

### 4.4 One stock waku process per active tenant

Considered:

- **A.** One process serves every tenant through an in-process pool (the first
  revision of this doc).
- **B.** Each active tenant gets a stock waku process in its own container.

**Chosen: B.** Measured on waku 0.1.6 in this repository:

- **The dashboard is where waku keeps most of its per-process state (§3).** In A,
  every global has to become tenant-aware: 16 references in `dashboard.py`, plus
  `browser_agent`, `settings_api`, `catalog` and `integrations`. That is a deep
  change upstream is unlikely to take, so it becomes a fork that has to absorb
  every future dashboard change. In B, all of that state is already correct, and
  the dashboard is literally unmodified.
- **Per-process isolation works as expected.** Two stock dashboards were run with
  different `WAKU_HOME` and working directories. A settings write to one landed
  only in that tenant's `.env`. The other tenant's `.env` and the platform's
  `.env` were untouched, and each had its own `state.db`.
- **Cost.** About 60 MB resident per dashboard process, idle and after a data
  refresh. About 58 MB for a process that built an agent and ran one turn with a
  scripted client. Budget about 100 MB with real provider SDK clients. Cold start
  is 1 to 2 seconds until the port is listening. Sixty concurrently active
  tenants is about 6 GB; with idle stop (§6), hundreds of registered users means
  tens active.
- **Isolation becomes OS-enforced.** A container's mount namespace contains only
  that tenant's directories. In A, any code-execution path in waku, present or
  future, reaches every tenant. In B it reaches one container. For example, the
  SQL console runs a recursive query without bound (tested: killed after 8
  seconds, never returned). In A that ties up a thread and a CPU core in the
  shared server; in B a per-container CPU limit confines it.
- **Upgrades.** The tenant image pins a waku release from PyPI, so upgrading is a
  tag bump. No fork is needed for the MVP.

A still wins on density: thousands of mostly idle tenants, and exact per-turn
control inside one process. If memory ever becomes the binding constraint, A is
the upgrade path. Its full sketch, a `TenantPool` that compiles against waku,
is in commit `ee3251a`.

### 4.5 Login: Supabase Auth, then a gateway cookie

1. The gateway serves `/login`, a small page using Supabase's client library
   (email magic link or OAuth).
2. The page sends the Supabase access token to `POST /auth/session`.
3. The gateway verifies the token against the project's public signing key (no
   call to Supabase per request), creates a session row in `control.db`, and sets
   an `HttpOnly; Secure; SameSite=Lax` cookie.
4. Every later request: cookie, then session (cached in memory), then tenant id.

Why a cookie rather than a bearer token: the stock dashboard's requests are
same-origin, so the browser attaches the cookie to them automatically, and the
dashboard needs no changes. `SameSite=Lax` plus JSON-only POSTs covers CSRF for
the MVP.

The gateway owns only `/login`, `/auth/*` and `/account`, and never shadows the
dashboard's `/`, `/static/*` or `/api/*`.

Nothing depends on Supabase specifically: any provider issuing a JWT with a
stable `sub` works, for example Cognito on AWS.

Pre-warm: `POST /auth/session` also starts the tenant's container, so it boots
while the dashboard page loads.

### 4.6 Model calls go through a metering proxy

For a platform-key tenant, the tenant's `.env` points waku at the gateway:

```
WAKU_BASE_URL=http://<proxy address>
WAKU_API_KEY=<per-tenant proxy token>
```

For each call, the proxy validates the token, checks the quota (failing closed
before the call goes upstream), checks the model against the free tier's
allowlist, swaps in the platform key, forwards the request, reads usage from the
response, records spend, and holds a global limit on concurrent upstream calls.

**Verified.** A stock waku turn, configured by only those two variables, sent
both of its model calls to the proxy with the tenant token as `x-api-key`. The
proxy saw **two** calls: the retrieval gate on the small model and the loop on
the main model. waku's own `usage.jsonl` recorded **one**. The gate, and
consolidation, which follows the same pattern, bypass waku's ledger. So metering
must happen at the proxy. (The undercount is also an upstream bug worth fixing
on its own; see §13.)

What this gets us:

- the platform key is never inside a tenant container
- the quota is checked on every call, not every turn
- spend is exact
- rate limits live in one place
- revoking a token cuts a tenant off immediately
- the free tier's model list is enforced centrally

**BYOK needs no special handling.** A tenant who enters their own provider key
in the stock Connections page makes their waku talk to the provider directly.
The platform pays nothing, so there is nothing to enforce. "Back to the free
tier" is a gateway action that rewrites the two lines and restarts the container.

**Write those lines into `.env`, not the container environment.** An injected
variable takes precedence over `.env` (`load_dotenv` does not override existing
variables), so it would silently undo a tenant's switch to their own key on the
next restart.

**Streaming.** The dashboard streams replies. The proxy must pass the event
stream through unbuffered and read usage from its final events.

### 4.7 A separate project running stock waku

`waku-hosted` contains the gateway, the proxy, the spawner, and the deploy
scripts. The gateway proxies HTTP and does not import waku. The tenant image is
stock waku from PyPI.

**Required waku change: one.** The dashboard's bind address must be
configurable, for example `WAKU_DASHBOARD_HOST`, defaulting to `127.0.0.1`. The
dashboard hard-codes loopback, which cannot be reached from outside its
container. Until that lands upstream, the image carries a one-line patch or a
loopback forwarder.

## 5. Architecture

```
 browser ──HTTPS──> Caddy (TLS, :443)
                      └──> gateway            login · session cookie · route policy · streaming proxy
                             ├──> control.db  tenants · sessions · proxy tokens · spend · plans
                             ├──> spawner     start / stop / list tenant containers
                             │                (the only holder of the Docker socket)
                             └──> tenant container <id>   stock waku dashboard
                                    /data <─ /srv/waku/tenants/<id>/home   (WAKU_HOME)
                                    /work <─ /srv/waku/tenants/<id>/env    (working dir, .env)
                                    │
                                    ├── model calls ──> LLM proxy ── platform key ──> provider
                                    └── BYOK and search ──> internet

 Supabase Auth <── /login page ; the gateway verifies tokens with the project's public key
```

Suggested network shape (the spec pins the rules): the gateway and proxy run on
the host network. Tenant containers sit on a bridge with inter-container traffic
disabled, so they can reach the proxy and the internet but not each other.

## 6. Tenant lifecycle

- **Provision**, on the first successful login, if signup rules allow it: create
  the tenant row, the directory tree, a `.env` pre-filled with the platform-mode
  lines, and a proxy token.
- **Start**, on the first request or at login: the spawner creates the container
  from a fixed template (image tag, the two mounts, limits, network, minimal
  environment), and the gateway waits for it to answer.
- **Active**: an open dashboard tab keeps the container alive, since the page
  polls every 450 ms and every 5 s.
- **Idle stop**: after N minutes without a proxied request, stop the container.
  This is safe because each turn commits to SQLite when it ends, and nothing runs
  in the background in the MVP (channels are off).
- **Config change** (key mode, plan): restart the container.
- **Delete**: stop the container, archive the directory, revoke the token.

## 7. Storage layout

```
/srv/waku/
  config/          platform secrets: Supabase settings, platform model key, domain,
                   backup target. root-only, never mounted into a tenant.
  control/
    control.db     tenants · sessions · proxy tokens (hashed) · spend · plans
  tenants/<id>/
    home/          mounted at /data  — WAKU_HOME: state.db, SOUL.md, skills/, traces/, ...
    env/.env       mounted at /work  — the process's working directory
```

**Rule: every tenant's working directory gets its own `.env` before first start,
and platform config never sits in a parent directory of a tenant's working
directory.** Verified: a tenant process with no `.env` of its own walked up the
tree, loaded the platform's `.env`, and exposed the platform secret in its
environment. Had it then saved a setting, the write would have gone into the
platform's file. Containers make the parent directory invisible anyway; the
pre-created file is the second layer, and it matters for any non-container setup.

## 8. The hosted dashboard

The dashboard is unmodified. The gateway applies a per-route policy:

| Routes | Gateway | Why |
|---|---|---|
| `/`, `/static/*`, `/api/data`, `/api/events`, `/api/session`, `/api/memory`, `/api/chat`, `/api/chat/stream`, `/api/pin`, `/api/models`, `/api/query`, `/api/compare/history`, `/api/graph/stream` | pass | per-tenant by construction: own process, own home. `/gather`'s GitHub step finds no `gh` in the container and falls back to its empty result (`_safe` in `gather.py`) |
| `/api/providers` | pass | a tenant setting their own key is BYOK; billing is enforced at the proxy |
| `/api/settings` | filter: drop `experimental` | otherwise a tenant can enable delegation tools (verified: the write succeeds). pi is also absent from the image, so this is two layers |
| `/api/connections`, `/api/connections/test` | filter: allow providers, Tavily, Notion; reject channel tokens (MVP), host-bound integrations (Apple, Google Calendar), hosted memory backends | channels are deferred; host-bound integrations cannot work in a container |
| `/api/compare/*`, `/api/memory-arena/*` | block in the MVP | compare races several models (several times the spend); the arena needs hosted memory keys |
| `/api/voice` | block in the MVP | every container would load its own Whisper model |
| `/api/reveal` | block | opens files in a local editor; harmless in a container, but a dead button |

Routes not in the table pass by default. Isolation does not depend on this table,
because the container provides it. A new upstream route cannot break isolation;
at worst it spends that tenant's own quota. On each waku upgrade, diff the pinned
route list in `evals/deterministic/test_dashboard_routes.py` and review what changed.

What differs in the frontend: the gateway serves the login page and `/account`
(logout, free-tier status, "back to free tier"). The stock static files are
untouched. Two gaps follow from that:

- The dashboard has no logout or account link, so `/account` is reached by URL
  in the MVP.
- When a session expires, full page loads redirect to `/login`, but the page's
  background requests just fail until the user reloads.

A small upstream option to show an external account link would fix both.

## 9. Keys, quotas, spend

- Free tier: a dollar cap per period plus turns per hour, enforced at the proxy.
- Per call: token, then tenant, then quota, then model allowlist, then forward,
  then meter.
- Global: a cap on concurrent upstream calls, because the provider's rate limit
  is per key, not per tenant.
- BYOK traffic never passes the platform's meter.
- The spend ledger lives in `control.db`. The tenant's own `usage.jsonl` is still
  what their dashboard shows, and it **underreports** (it misses gate and
  consolidation calls) until the upstream fix lands.

## 10. Security model

**The container is the boundary.** Assume a tenant may get code execution inside
their own container; the boundary must hold anyway. This posture also survives
future waku features nobody has audited.

Controls:

1. **Mounts**: only the tenant's two directories. No Docker socket and no other
   host paths.
2. **Secrets**: no platform secret in a tenant's environment or files. The
   platform model key exists only in the proxy.
3. **Network**: tenant containers cannot reach each other. The stock dashboard has
   no authentication, so network position is its only protection. Containers
   reach only the proxy and the internet. **Block the cloud metadata address
   (`169.254.169.254`)** from tenant containers; otherwise a container can read
   the VM's cloud credentials.
4. **Limits** per container: memory, CPU, process count. These confine a runaway
   query or a stuck turn to one tenant.
5. **Docker access**: the internet-facing gateway does not hold the Docker socket.
   A local spawner with a fixed container template does (or use rootless Podman).
6. **Image**: runs as a non-root user, with a read-only root filesystem apart
   from the mounts and a tmpfs `/tmp`.
7. **Tenant ids** come only from the verified session, never from request data,
   and are validated against `^[a-z0-9][a-z0-9_-]{0,63}$` before they touch a path.

Residual risks:

- a compromised container can still send outbound traffic (add egress rate
  limits later)
- compromise of the VM itself exposes all data, as for any single-VM system

## 11. Deployment and portability

Three layers, and only the first depends on the provider:

| Layer | What | Provider-specific |
|---|---|---|
| Provision | a VM, a disk, a firewall, DNS: Terraform or OpenTofu modules per provider, or by hand | yes, and optional |
| Install | `install.sh` on a fresh Ubuntu or Debian VM: install Docker; create the `/srv/waku` tree with permissions; write config from flags; start Caddy, gateway and spawner with Compose; add the tenant network and the metadata block; install the backup timer. Caddy obtains TLS certificates automatically | no |
| Operate | `upgrade` (bump image tags), `backup`, `restore`, `migrate` | no |

**Migrate**: stop, take a final backup, restore onto the new VM, run the install
script, move DNS. Downtime is minutes, because all state is one directory tree.

**Backups**: take a consistent copy of each SQLite file (tenant databases and
`control.db`), then send encrypted, deduplicated snapshots to S3-compatible
storage (S3, GCS, R2 or B2). A restore drill is part of the MVP, not a later task.

## 12. Scope

**MVP (phase 1)**

- gateway: login, cookie sessions, `/account`, route policy, streaming proxy
- LLM proxy: tokens, quota, model allowlist, metering, concurrency cap
- spawner: start, stop, idle stop, fixed template, limits, network isolation
- `control.db`
- tenant image: stock waku plus the bind-address flag
- scripts: install, upgrade, backup, restore, migrate
- verification: an isolation suite, a polling load test, a restore drill, a
  security review

**Phase 2 (deferred)**

- **Channels.** Telegram and Discord become cheap in this design. A tenant pastes
  their own bot token into their own Connections page, and the stock gateway runs
  inside their container. There is no central bot and no account linking. The
  cost is that the tenant's container must stay up (no idle stop). This needs an
  "always on" plan flag and should require `TELEGRAM_ALLOWED_USER`. Slack needs a
  new waku gateway (none exists; Socket Mode fits the same model). WhatsApp needs
  per-tenant webhook routing through the gateway.
- compare for BYOK tenants, voice, hosted memory backends, remote MCP servers per
  tenant, Google Calendar through a web OAuth flow
- an operator view across tenants; OpenTelemetry export

**Not doing**

- in-process multi-tenancy (§4.4), unless density forces it
- more than one VM. The path, when needed: a shared control plane holding the
  tenant-to-VM map, with tenant directories moved as whole units

**Workstreams**, a coarse cut for the plan:

1. Tenant runtime: image and bind flag, spawner, lifecycle, network isolation, limits
2. Gateway: auth, sessions, policy, streaming proxy (the largest)
3. LLM proxy and control plane: tokens, quota, metering, allowlist
4. Deploy: install, upgrade, backup, restore, migrate; provision modules
5. Verification: isolation, load, restore drill, security review

**Invariants the spec must preserve**

- no request can reach another tenant's container or directory
- the platform key appears in no tenant container: environment, files or logs
- an exhausted quota is rejected before any upstream call
- a waku upgrade needs no gateway change unless the pinned route list changed
- the whole system can be restored onto a fresh VM from backups and the install script

## 13. Upstream changes to waku

| Change | Needed for | Status |
|---|---|---|
| configurable dashboard bind address (default `127.0.0.1`) | reaching the dashboard inside a container | **required**; patch in the image until upstream |
| meter the retrieval gate and consolidation in `usage.jsonl` | honest cost tiles in the dashboard | recommended; it is a local bug too |
| SQL console: a SQLite authorizer allowlist and a time budget via progress handler | a runaway query should fail, not hang | recommended; containers already confine it |
| optional external account link in the dashboard | logout and account from inside the dashboard | nice to have |

## 14. Risks

- **Polling fan-out.** Each open tab makes about 2.2 requests per second
  (`/api/events` every 450 ms) plus one `/api/data` every 5 s. The second one
  builds a full summary of the tenant's database. At 100 open tabs that is
  roughly 240 requests per second through the gateway. Measure it; the knob is
  the stock frontend's poll interval.
- **Memory**: about 100 MB per active tenant. Idle stop is the mitigation.
- **Cold start**: 1 to 2 seconds on the first request after idle. Pre-warming at
  login hides most of it.
- **Cost display**: the dashboard's cost figures underreport compared with the
  proxy's meter until the upstream ledger fix.
- **Upgrade drift**: waku route changes. Review them through the pinned route list.
- **Single VM**: one point of failure. Backups and the migrate script are the recovery path.

## 15. Open questions for the spec

1. Signup: invite-only (recommended for the MVP, since a free tier is an abuse
   target) or open?
2. Free tier: how many dollars per month and turns per hour?
3. Free tier: which models are on the allowlist?
4. Idle-stop timeout?
5. Which provider hosts the first VM?
6. Offer the §13 changes upstream as pull requests to `ShenSeanChen/waku-agent`?
