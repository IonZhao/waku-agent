# Hosted Waku — design

Status: DRAFT design doc. It records the decisions and the shape of the system;
the spec and the implementation plan are generated from it in a separate step.
Appendix A is the decision log: what changed along the way and what is now
superseded, so that step does not bring it back.

This revision replaces the first one (commit `ee3251a` in `IonZhao/waku-agent`,
PR #1), which
designed a single process serving every tenant. Narrowing the MVP to "the
dashboard, unmodified" changed that answer; §4.4 explains why, with measurements.

## 1. Summary

One operator runs one Linux VM. Hundreds of people sign in, and each gets their
own waku: the same dashboard as local waku, their own memory, history, settings
and spend. Each active person's waku is a **stock waku process in its own
container**. A small gateway in front handles login, routing, and metering of
model calls. waku needs two small patches, neither of which changes anything for
a local user (§13).

| Decision | Choice |
|---|---|
| Who it serves | individuals; a tenant is one person |
| Unit of tenancy | one stock waku process per active tenant, in its own container |
| Tenant data | one directory per tenant on the VM's persistent disk |
| Dashboard | stock, all 12 views; the Compare view's races are off in the MVP |
| Login | Supabase Auth (email magic link) as the identity provider, then a gateway session cookie |
| Storage systems to operate | one: the filesystem. The control plane is a local SQLite file |
| Platform model key | never enters a tenant container; model calls go through a metering proxy |
| Deploy target | one Linux VM with a persistent disk, any provider, one install script |
| MVP scope | the dashboard only; all channels deferred |

## 2. Goals and non-goals

Goals, as the operator set them:

- one person deploys it; hundreds of people use it, isolated in data, config and spend
- the dashboard exactly as it runs locally, with every view kept
- a platform key with a free tier, or the user's own key (BYOK)
- one storage system to operate; Supabase only where it removes work (login)
- install on any provider's VM with one script; upgrade, back up and move it with scripts
- waku releases flow through without maintaining a fork

Non-goals for the MVP:

- channels: Telegram, Slack, Discord, WhatsApp. Their hard part is integration
  and account linking; §12 explains how this design removes the linking problem
- more than one VM (the path is described, not built)
- nested tenants: a tenant is one person, not a business with its own customers
- host-bound features: voice, Apple tools, editor reveal, pi delegation, `gh`
- a new frontend

## 3. What in waku is single-user today

Paths are in the `waku-agent` source, version 0.1.6.

| Assumption | Where |
|---|---|
| one agent per process behind one lock | `waku/ops/browser_agent.py`: module globals `_agent`, `agent_lock` |
| configuration is the process environment; the Connections, Models and Settings pages write `.env` and set `os.environ` | `integrations._write_updates`, `settings_api.apply_settings` |
| no authentication; bound to loopback | `waku/ops/dashboard.py:1159`, `ThreadingHTTPServer(("127.0.0.1", port))` |
| process-wide caches | `integrations._HEALTH`, the model catalog cache (keyed by URL, not key), the Whisper model, the Notion store |
| `.env` is found by walking **up** from the working directory | `waku/config.py`, `find_dotenv(usecwd=True)` |
| "now" is the process's local time, which the agent uses to resolve "tomorrow at 9" | `waku/runtime/session.py:68`, `datetime.now().astimezone()` |
| 16 call sites in `dashboard.py` alone read this state (`load_settings()`, `get_agent()`, `agent_lock` and similar); direct SQL in 14 files | grep over `waku/` |

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

Principle: use Supabase only where it removes work, and nowhere else.

- **Tenant data**: the tenant's directory, as above.
- **Control plane**: `control.db`, a SQLite file on the same disk: tenants,
  login sessions, proxy tokens (hashed), the spend ledger, plans.
- **Supabase**: Auth only, used as an identity provider rather than as a
  database we operate. User records live in Supabase; `control.db` keeps the
  user id (`sub`) and, for display, the email.

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

**Chosen: B.** Measured on waku 0.1.6:

- **The dashboard is where waku keeps most of its per-process state (§3).** In A,
  every global has to become tenant-aware: 16 references in `dashboard.py`, plus
  `browser_agent`, `settings_api`, `catalog` and `integrations`. That is a deep
  change upstream is unlikely to take, so it becomes a fork that has to absorb
  every future dashboard change. In B, all of that state is already correct, and
  the dashboard's views and routes run as shipped.
- **Per-process isolation works as expected.** Two stock dashboards were run with
  different `WAKU_HOME` and working directories. A settings write to one landed
  only in that tenant's `.env`. The other tenant's `.env` and the platform's
  `.env` were untouched, and each had its own `state.db`.
- **Cost.** About 60 MB resident per dashboard process, idle and after a data
  refresh. About 58 MB for a process that built an agent and ran one turn with a
  scripted client. Budget about 100 MB with real provider SDK clients. The process
  listens within about 2 seconds with a cold disk cache and under 1 second warm;
  creating the container adds time on top, still to be measured. Sixty concurrently active
  tenants is about 6 GB; with idle stop (§6), hundreds of registered users means
  tens active.
- **Isolation becomes OS-enforced.** A container's mount namespace contains only
  that tenant's directories. In A, any code-execution path in waku, present or
  future, reaches every tenant. In B it reaches one container. For example, the
  SQL console runs a recursive query without bound (tested: killed after 8
  seconds, never returned). In A that ties up a thread and a CPU core in the
  shared server; in B a per-container CPU limit confines it.
- **Upgrades.** The tenant image pins a waku release from PyPI plus the two
  patches in §13, so upgrading is a tag bump. No fork of the codebase is needed.

A still wins on density: thousands of mostly idle tenants, and exact per-turn
control inside one process. If memory ever becomes the binding constraint, A is
the upgrade path. Its full sketch, a `TenantPool` that compiles against waku,
is in commit `ee3251a` of `IonZhao/waku-agent`.

### 4.5 Login: Supabase Auth, then a gateway cookie

1. The gateway serves `/login`, a small page using Supabase's client library to
   sign in by email magic link. It also reads the browser's time zone
   (`Intl.DateTimeFormat().resolvedOptions().timeZone`).
2. The page sends the Supabase access token and the time zone to `POST /auth/session`.
3. The gateway verifies the token against the project's public signing key (no
   call to Supabase per request), creates a session row in `control.db`, records
   the time zone on first login, and sets an `HttpOnly; Secure; SameSite=Lax` cookie.
4. Every later request: cookie, then session (cached in memory), then tenant id.

Why a cookie rather than a bearer token: the stock dashboard's requests are
same-origin, so the browser attaches the cookie to them automatically, and the
dashboard needs no changes. `SameSite=Lax` plus JSON-only POSTs covers CSRF for
the MVP.

Invite-only needs no code: Supabase Auth can turn off open signup and invite
users by email.

The gateway owns only `/login`, `/auth/*` and `/account`, and never shadows the
dashboard's `/`, `/static/*` or `/api/*`.

Nothing depends on Supabase specifically: any provider issuing a JWT with a
stable `sub` works, for example Cognito on AWS.

Pre-warm: `POST /auth/session` also starts the tenant's container, so it boots
while the dashboard page loads.

### 4.6 Model calls go through a metering proxy

The MVP proxy fronts the Anthropic API. For a platform-key tenant, the tenant's
`.env` points waku at it:

```
WAKU_BASE_URL=http://<proxy address>
WAKU_API_KEY=<per-tenant proxy token>
```

For each model call, the proxy:

1. validates the token
2. checks the quota, failing closed before the call goes upstream
3. checks the model against the free tier's allowlist
4. swaps in the platform key and forwards the request
5. reads usage from the response and records spend
6. holds a global limit on concurrent upstream calls

`GET /v1/models` answers with the allowlist and is not metered, so the stock
Models page shows exactly what the free tier offers.

**Verified: calls.** A stock waku turn, configured by only those two variables,
sent both of its model calls to the proxy with the tenant token as `x-api-key`.
The proxy saw **two** calls: the retrieval gate on the small model and the loop
on the main model. waku's own `usage.jsonl` recorded **one**. The gate, and
consolidation, which follows the same pattern, bypass waku's ledger. So
metering must happen at the proxy.

**Verified: model listing needs a patch.** For the `anthropic` provider, waku
lists models from a hard-coded public catalog URL even when a custom base URL is
set. Unpatched, a platform tenant's Models page sends the proxy token to
`api.anthropic.com` and fails. Saving provider settings fails its check too,
because the check runs the same listing. Patched so that a custom base URL lists
from that endpoint, the request reached the proxy and returned the allowlist (§13).

What this gets us:

- the platform key is never inside a tenant container
- the quota is checked on every call, not every turn
- spend is exact
- rate limits live in one place
- revoking a token cuts a tenant off immediately
- the free tier's model list is enforced centrally

**Rejections must read well and must not be retried.** The SDK automatically
retries rate-limit and server errors. So a quota rejection uses a status it does
not retry (403), with a message in the provider's error format: "Free tier used
up. Add your own key in Connections." The stock dashboard shows that text. A
genuine upstream rate limit stays a 429, where retrying is right.

**BYOK needs no special handling.** A tenant who enters their own provider key
in the stock Connections page makes their waku talk to the provider directly.
The platform pays nothing, so there is nothing to enforce. "Back to the free
tier" is a gateway action that rewrites the two lines and restarts the container.

These two lines belong in `.env`, not in the container environment (§7).

**Streaming.** The dashboard streams replies. The proxy must pass the event
stream through unbuffered and read usage from its final events.

### 4.7 A separate project running stock waku

`waku-hosted` contains the gateway, the proxy, the spawner, and the deploy
scripts. The gateway proxies HTTP and does not import waku. The tenant image is
waku from PyPI plus two small patches (§13), carried in the image until upstream
takes them:

1. **Configurable bind address**, for example `WAKU_DASHBOARD_HOST`, defaulting to
   `127.0.0.1`. The dashboard hard-codes loopback, which cannot be reached from
   outside its container.
2. **Model listing follows a custom base URL**, as verified in §4.6.

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
                                    ├── model calls ──> LLM proxy ── platform key ──> Anthropic
                                    └── BYOK and search ──> internet

 Supabase Auth <── /login page ; the gateway verifies tokens with the project's public key
```

Network shape: Caddy, the gateway and the proxy run on the host network. Tenant
containers sit on a bridge with inter-container traffic disabled, so they can
reach the proxy and the internet but not each other, and they cannot reach the
cloud metadata address (§10). The exact firewall rules are implementation.

## 6. Tenant lifecycle

- **Provision**, on the first successful login:
  - create the tenant row, the directory tree, and a proxy token
  - write a `.env` pre-filled with the platform-mode lines
  - write a `SOUL.md` suited to hosting (waku's default says it runs on the
    user's laptop)
  - record the time zone captured at login
- **Start**, on the first request or at login: the spawner creates the container
  from a fixed template (image tag, the two mounts, limits, network, the
  platform-controlled environment of §7), and the gateway waits for it to answer.
- **Active**: an open dashboard tab keeps the container alive, since the page
  polls every 450 ms and every 5 s.
- **Idle stop**: a container is idle when no request is in flight and none has
  arrived for N minutes; then it is stopped. This is safe because each turn
  commits to SQLite when it ends, and nothing runs in the background in the MVP
  (channels are off).
- **Capacity**: a cap on running containers, sized to memory. At the cap, the
  longest-idle container is stopped; if none is idle, the gateway shows "at
  capacity, try again shortly".
- **Config change**: returning to the free tier or changing the time zone restarts
  the container. Plan changes apply at the gateway and proxy, without a restart.
- **Delete**: stop the container, archive the directory, revoke the token.

## 7. Storage layout and configuration

```
/srv/waku/
  config/          platform secrets: Supabase settings, the platform model key,
                   domain, backup target. root-only, never mounted into a tenant.
  control/
    control.db     tenants · sessions · proxy tokens (hashed) · spend · plans
  tenants/<id>/
    home/          mounted at /data: WAKU_HOME (state.db, SOUL.md, skills/, traces/, ...)
    env/           mounted at /work: the process's working directory, holding its .env
```

**Where each setting lives.** Environment variables injected into a container
take precedence over `.env`, because `load_dotenv` does not override variables
that already exist. That gives a clean split:

- **Platform-controlled settings go in the container environment**, where the
  tenant cannot change them: `WAKU_HOME=/data`, the bind address and port, and
  `TZ=<the tenant's time zone>`.
- **Tenant-owned settings go in `.env`**, which the stock pages edit: provider,
  keys, models, toggles, and the platform-mode lines of §4.6. Injecting those
  instead would silently undo a tenant's switch to their own key on the next restart.

**Every tenant's working directory gets its own `.env` before first start, and
platform config never sits in a parent directory of a tenant's working
directory.** Verified: a tenant process with no `.env` of its own walked up the
tree, loaded the platform's `.env`, and exposed the platform secret in its
environment. Had it then saved a setting, the write would have gone into the
platform's file. Containers make the parent directory invisible anyway; the
pre-created file is the second layer, and it matters for any non-container setup.

**Time zone.** Verified: waku's "now" follows the process time zone. At the same
moment it read Tuesday 19:09 under UTC and Wednesday 03:09 under Asia/Shanghai.
A container defaults to UTC, so without `TZ` every non-UTC tenant's "tomorrow at
9" resolves a day or hours off. The image must include the system time zone
database (the `tzdata` package).

## 8. The hosted dashboard

The dashboard's views and routes run as shipped. The gateway applies a per-route policy:

| Routes | Gateway | Why |
|---|---|---|
| `/`, `/static/*`, `/api/data`, `/api/events`, `/api/session`, `/api/memory`, `/api/chat`, `/api/chat/stream`, `/api/pin`, `/api/models`, `/api/query`, `/api/compare/history`, `/api/graph/stream` | pass | per-tenant by construction: own process, own home. `/api/models` relies on the listing patch (§13). `/gather`'s GitHub step finds no `gh` in the container and falls back to its empty result (`_safe` in `gather.py`) |
| `/api/providers` | pass | provider and key changes, including BYOK. Billing is enforced at the proxy, so nothing needs guarding here |
| `/api/settings` | filter: drop `experimental` | otherwise a tenant can enable delegation tools (verified: the write succeeds). pi is also absent from the image, so this is two layers |
| `/api/connections`, `/api/connections/test` | filter: allow Tavily and Notion; reject channel tokens (Telegram, Discord, WhatsApp), host-bound integrations (Apple Calendar, Apple tools, Google Calendar), hosted memory backends (Mem0, Zep, LangMem, Supabase) and OpenTelemetry | channels are deferred; host-bound integrations cannot work in a container; memory backends and telemetry are platform decisions. Providers are not on this route: waku handles them through `/api/providers` |
| `/api/compare/*`, `/api/memory-arena/*` | block in the MVP | the two races of the Compare view: the model race runs several models at once (several times the spend); the memory race needs hosted memory keys |
| `/api/voice` | block in the MVP | every container would load its own Whisper model |
| `/api/reveal` | block | opens files in a local editor; harmless in a container, but a dead button |

**Blocked and filtered requests answer in the shape the stock UI already
renders**: a JSON body with an `error` field. For streaming routes, a single
terminal `done` event carries the error. The page then shows a readable message
instead of breaking.

Routes not in the table pass by default. Isolation does not depend on this table,
because the container provides it. A new upstream route cannot break isolation;
at worst it spends that tenant's own quota. On each waku upgrade, diff the pinned
route list in waku's `evals/deterministic/test_dashboard_routes.py` and review
what changed.

What differs in the frontend: the gateway serves the login page and `/account`
(logout, free-tier status, time zone, "back to free tier"). The stock static
files are untouched. Two gaps follow from that:

- The dashboard has no logout or account link, so `/account` is reached by URL
  in the MVP.
- When a session expires, full page loads redirect to `/login`, but the page's
  background requests just fail until the user reloads. Sessions last 30 days to
  make this rare.

A small upstream option to show an external account link would fix both.

## 9. Keys, quotas, spend

- **Dollars**: the proxy enforces a per-tenant dollar cap per month on
  platform-key traffic, per call (§4.6).
- **Turns**: the gateway enforces turns per hour for every tenant, BYOK included,
  because it protects the VM's compute rather than the platform's key. A turn is
  one request to `/api/chat`, `/api/chat/stream` or `/api/graph/stream`; the
  gateway sees those, whereas the proxy only sees individual model calls.
- **Concurrency**: the proxy caps concurrent upstream calls, because the
  provider's rate limit is per key, not per tenant.
- **BYOK** traffic never passes the platform's meter.
- **Ledger**: spend lives in `control.db`. The tenant's own `usage.jsonl` is still
  what their dashboard shows, and it **underreports** (it misses gate and
  consolidation calls) until the upstream fix in §13 lands.
- **Web search**: waku's search tool uses Tavily when it has a key, otherwise
  keyless DuckDuckGo scraping. The platform's Tavily key is not injected into
  tenant containers. So free-tier search uses the keyless path, which is likely
  to be rate-limited when many tenants share one server address. Tenants can add
  their own Tavily key in Connections; a platform search proxy is phase 2.

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
   A local spawner does, and it creates containers only from its fixed template.
6. **Image**: runs as a non-root user, with a read-only root filesystem apart
   from the two mounts and a tmpfs `/tmp`. The spec confirms waku writes nowhere else.
7. **Tenant ids** come only from the verified session, never from request data,
   and are validated against `^[a-z0-9][a-z0-9_-]{0,63}$` before they touch a path.

Residual risks:

- a compromised container can still send outbound traffic (add egress rate
  limits later)
- compromise of the VM itself exposes all data, as for any single-VM system
- the operator can read every tenant's data; the terms of use must say so

## 11. Deployment and portability

Three layers, and only the first depends on the provider:

| Layer | What | Provider-specific |
|---|---|---|
| Provision | a VM, a disk, a firewall, DNS: Terraform-compatible modules per provider, or by hand | yes, and optional |
| Install | `install.sh` on a fresh Ubuntu 24.04 LTS VM: install Docker; create the `/srv/waku` tree with permissions; write config from flags; start Caddy, gateway and spawner with Compose; add the tenant network and the metadata block; install the backup timer. Caddy obtains TLS certificates automatically | no |
| Operate | `upgrade` (bump image tags), `backup`, `restore`, `migrate` | no |

**Migrate**: stop, take a final backup, restore onto the new VM, run the install
script, move DNS. Downtime is minutes, because all state is one directory tree.

**Backups**: nightly, keeping 7 daily and 4 weekly copies. Each run takes a
consistent copy of every SQLite file (tenant databases and `control.db`) with
SQLite's online backup, then sends encrypted, deduplicated snapshots with restic
to S3-compatible storage (S3, GCS, R2 or B2). A restore drill is part of the MVP,
not a later task.

**Upgrades** roll as containers restart. waku's schema migrations are additive
and idempotent, so an old tenant database opens under a newer image. Downgrades
are not supported.

**Operator access to one tenant**: run a stock `waku dashboard` on the VM with
`WAKU_HOME` pointing at that tenant's `home/`, bound to loopback, reached over
an SSH tunnel. The gateway is not involved.

## 12. Scope

**MVP (phase 1)**

- gateway: login, cookie sessions, `/account`, route policy, turn limits,
  streaming proxy
- LLM proxy: tokens, quota, model allowlist, metering, concurrency cap
- spawner: start, stop, idle stop, capacity cap, fixed template, limits, network
  isolation
- `control.db`
- tenant image: waku from PyPI with the two patches, time zone data, and a hosted `SOUL.md`
- scripts: install, upgrade, backup, restore, migrate
- verification: an isolation suite, a polling load test, a restore drill, a
  security review

**Phase 2 (deferred)**

- **Channels.** The hard part of channels is integration and account linking,
  and this design removes the linking. A tenant pastes their own bot token into
  their own Connections page, and waku's stock Telegram or Discord gateway runs
  inside their container. There is no central bot and no account to link.
  - Cost: the tenant's container must stay up (no idle stop), which needs an
    "always on" plan flag.
  - The gateway should require `TELEGRAM_ALLOWED_USER`, so strangers cannot
    spend the tenant's quota.
  - Slack needs a new waku gateway (none exists); Socket Mode fits the same model.
  - WhatsApp needs per-tenant webhook routing through the gateway.
  - History: every channel writes to the same chat log and long-term memory, and
    the dashboard's Gateway view already shows every channel in one inbox. Each
    channel keeps its own recent-conversation window, as in local waku. Making a
    phone message continue the exact dashboard thread needs an upstream runner
    change (a per-turn session id), sketched in the first revision.
- compare for BYOK tenants, voice, hosted memory backends, remote MCP servers per
  tenant, Google Calendar through a web OAuth flow, OAuth sign-in providers
- a platform search proxy; data export from `/account`
- an operator view across tenants; OpenTelemetry export

**Not doing**

- in-process multi-tenancy (§4.4), unless density forces it
- more than one VM. The path, when needed: a shared control plane holding the
  tenant-to-VM map, with tenant directories moved as whole units

**Workstreams**, a coarse cut for the plan:

1. Tenant runtime: image and patches, spawner, lifecycle, network isolation, limits
2. Gateway: auth, sessions, policy, turn limits, streaming proxy (the largest)
3. LLM proxy and control plane: tokens, quota, metering, allowlist
4. Deploy: install, upgrade, backup, restore, migrate; provision modules
5. Verification: isolation, load, restore drill, security review

**Invariants the spec must preserve**

- no request can reach another tenant's container or directory
- the platform key appears in no tenant container: environment, files or logs
- an exhausted quota is rejected before any upstream call, with a readable message
- a tenant's assistant uses that tenant's time zone
- a waku upgrade needs no gateway change unless the pinned route list changed
- the whole system can be restored onto a fresh VM from backups and the install script

## 13. Changes to waku

| Change | Needed for | Status |
|---|---|---|
| configurable dashboard bind address (default `127.0.0.1`) | reaching the dashboard inside a container | **required**; patch in the image until upstream |
| model listing follows a custom base URL for Anthropic-wire providers | the Models page and the provider save check for platform tenants; also for anyone behind an Anthropic-compatible gateway | **required**; verified fix, patch in the image until upstream |
| meter the retrieval gate and consolidation in `usage.jsonl` | honest cost figures in the dashboard | recommended; it is a local bug too |
| SQL console: a SQLite authorizer allowlist and a time budget via progress handler | a runaway query should fail, not hang | recommended; containers already confine it |
| optional external account link in the dashboard | logout and account from inside the dashboard | nice to have |

## 14. Risks

- **Polling fan-out.** Each open tab makes about 2.2 requests per second
  (`/api/events` every 450 ms) plus one `/api/data` every 5 s. The second one
  builds a full summary of the tenant's database. At 100 open tabs that is
  roughly 240 requests per second through the gateway. Measure it; the knob is
  the stock frontend's poll interval.
- **Memory**: about 100 MB per active tenant. Idle stop and the capacity cap are
  the mitigation.
- **Cold start**: up to about 2 seconds for the process, plus container creation,
  on the first request after idle. Pre-warming at login hides most of it.
- **Cost display**: the dashboard's cost figures underreport compared with the
  proxy's meter until the upstream ledger fix.
- **Keyless search** is likely to be rate-limited from one shared server address.
- **Patch drift**: the two image patches must be re-applied on each waku upgrade
  until upstream takes them. The pinned route list shows route changes.
- **Single VM**: one point of failure. Backups and the migrate script are the recovery path.

## 15. Defaults for the spec

Nothing below blocks the spec. Each item has a default; override it before the
spec if you disagree.

| Question | Default |
|---|---|
| signup | invite-only, using Supabase's invite by email; a free tier is an abuse target |
| free tier size | $1 per user per month, and 30 turns per hour (waku's own per-hour turn default for its Discord bot). Worst case is registered users times the cap: 300 users cost at most $300 a month |
| free tier models | one mid-tier Anthropic model, named in config at deploy time |
| idle-stop timeout | 15 minutes |
| session length | 30 days |
| first VM | whichever provider the operator already uses. Start at 4 vCPU, 16 GB RAM and a 100 GB disk: about 100 active tenants with headroom, since memory is the binding resource |
| upstream | offer the two required patches and the two recommended fixes in §13 as pull requests to `ShenSeanChen/waku-agent`; carry them in the image meanwhile |

## Appendix A. Decision log

How the design got here. Items marked superseded must not be reintroduced
without revisiting the reason.

| Topic | Earlier | Now | Why it changed |
|---|---|---|---|
| audience | open question: one person per tenant, or businesses with their own customers | one person per tenant; a personal harness that many people use | operator's decision |
| MVP scope | Telegram first | **superseded**: the dashboard only; all channels deferred | channel integration and account linking are the hard part; the operator chose to start with the dashboard |
| dashboard | trim to four customer views | **superseded**: keep every view, unmodified | the operator wants it as each user's management console, exactly as it is |
| tenancy | one process for all tenants, an in-process `TenantPool` (rev 1) | **superseded**: one stock waku process per active tenant, in a container | with an unmodified dashboard, per-process state is correct as-is; measured cost is low (§4.4) |
| storage | Supabase Postgres for the control plane and the usage ledger | **superseded**: filesystem only; Supabase for Auth | operator's principle: one storage system, Supabase only where it removes work |
| metering | count spend from waku's own `llm` events (rev 1 `SpendMeter`) | **superseded**: meter at the proxy | measured: waku's ledger misses the retrieval gate's calls |
| platform key | held by every tenant's agent inside the shared process (rev 1) | **superseded**: stays in the proxy; tenants get revocable tokens | a tenant with code execution in their container must not obtain the platform key |
| deploy target | Cloud Run and ECS came up | **superseded**: any provider's VM with a persistent disk | single-writer SQLite needs a local disk; those platforms replace containers freely |
| cross-channel history | synchronized by construction via a per-turn session id in the pool (rev 1) | **superseded**: phase 2, stock per-channel windows with shared memory (§12) | per-tenant processes run waku's stock gateways |
| waku changes | five seams in waku (rev 1) | two small image patches (§13) | the stock dashboard needs only reachability and correct model listing |
| effort | 10 to 14 days estimated for rev 1 | **superseded**: re-estimate during planning | different architecture and workstreams |
| process | — | this design doc, then a spec from a separate spec-writing skill, then a plan | operator's decision |
