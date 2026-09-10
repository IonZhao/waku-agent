# Hosted Waku — system design

Status: DRAFT. Nothing in this document is built. It is the design doc that
`CONTRIBUTING.md` asks for before a new package exists (precedent:
`agent-graphs-design.md`), written so the work can be split into PRs that each
stand alone.

Scope: serve hundreds of people from one deployment — each with their own
memory, history, keys and spend — on a path that grows from one machine to
several, **without changing what `pip install waku-agent` is.**

## 1. What we're adding and why

waku is single-user by construction, and says so: one `.waku/` home, one SQLite
file, one agent behind one lock in the dashboard, configuration read from the
process environment. That is the right shape for "your assistant on your
laptop". It is the wrong shape for a service, which needs:

- an identity → tenant mapping, so a Telegram id and a web login are one person
- per-tenant memory, history, configuration and spend, with nothing shared
- turns serialized *within* a tenant and parallel *across* tenants
- two key modes: a platform key with a free quota, or the tenant's own key
- authentication on the web surface
- a scaling path that does not start with a rewrite

What we deliberately do **not** do is turn waku into a multi-tenant framework.
The loop, memory, tools and evals stay exactly what they are. The harness that
serves many people is a **separate package (`waku-hosted`) that imports waku as
a library**, plus five small changes to waku itself that are useful on their own
(§8).

## 2. Decisions

### 2.1 The tenant unit is a directory, not a row

The alternative — a `tenant_id` column on every table — was rejected. Direct SQL
against `conn` lives in 14 files (`chat_log`, `calendar_events`, notes, the
dashboard's reads); only `facts` and `episodes` sit behind a backend protocol.
One missing `WHERE` is a cross-tenant leak, and nothing in CI would catch it.

Instead: `Settings(home=/data/tenants/<id>)`. Everything waku persists already
lives under `home` — `state.db`, `SOUL.md`, `skills/`, `traces/`, `usage.jsonl`,
`calendar.ics`, `outbox/`, `models.json`, `token.json`. A directory per tenant
is physical isolation (there is no shared table to leak through), and the
directory is also the unit of backup, migration and sharding. `DISCORD_HOME` in
`waku/gateway/discord.py` is the existing precedent for "a separate home for a
separate audience".

### 2.2 One Waku per tenant, not one per gateway

Today each gateway builds its own `Waku` with its own `Session` (`terminal`,
`telegram`, `discord`, `dashboard-<date>`). Everything is written to one
`chat_log`, but working memory — the last N turns that enter the prompt — is per
gateway, and cross-channel continuity comes only from consolidation (every six
exchanges, summarized). The Telegram runner also starts with empty working
memory after every restart (`runner.py` sets the session id but never calls
`switch()`).

Hosted: one runner per tenant; every channel calls it. Working memory is loaded
per thread through `Session.switch()`, which reads `chat_log` — so history is
synchronized by construction and survives restarts. `chat_log.source` still
records which channel each message came in through.

### 2.3 A separate package, not a mode

`architecture.md` says "not production". `CONTRIBUTING.md` declines "anything
that costs every user context" and "reading `.env` or secrets". A hosted mode
inside waku would fight both. `waku-hosted` depends on `waku-agent`; nothing
under `waku/` imports it. The upstream changes in §8 each default to today's
behaviour and carry a deterministic eval.

### 2.4 Threads, not asyncio

The same decision as `agent-graphs-design.md` §7.2: waku is synchronous and owns
SQLite connections. `GatewayAgentRunner` already gives each agent one owner
thread; a tenant pool is a dict of those. Hundreds of threads blocked on HTTP is
fine — the work is I/O-bound, the GIL is irrelevant.

### 2.5 The process environment is the platform's, never a tenant's

`os.environ` and `.env` hold platform settings only: the bot token, the Postgres
URL, the platform API key, the OTel endpoint. Tenant settings are
**constructed**, not read — `Settings(provider=…, api_key=…, model=…)`.
`integrations.apply_*` and `settings_api.apply_settings`, which write `.env` and
mutate `os.environ`, are never called in hosted mode.

### 2.6 Two key modes

| mode | who pays | model choice | stops |
|---|---|---|---|
| platform | the operator | a fixed default (small/mid tier) | free tier in dollars, plus turns per hour |
| byok | the tenant | any provider in `PROVIDERS` | turns per hour only — an abuse stop, not a spend stop |

BYOK keys are encrypted at rest in the control plane, decrypted just in time
into a `TenantConfig`, and never written under the tenant's home. The retrieval
gate and consolidation use the same client as the loop
(`Memory(conn, settings, client)`), so a BYOK tenant's key pays for those calls
too; the UI says so rather than hiding it.

## 3. Layers

```
L0  Edge            TLS · connection limits · Telegram webhook ingress
L1  Identity        (channel, external_id) -> tenant_id: Telegram user.id, Discord author.id, web JWT
L2  Control plane   Postgres: tenants · identities · plans and quotas · keys (encrypted) · spend
L3  Tenant runtime  waku-hosted: TenantPool  tenant_id -> GatewayAgentRunner(Waku(Settings(home, api_key, …)))
                    per tenant: one thread · one TurnBudget · one SpendMeter · LRU eviction
                    per process: one semaphore in front of the model provider
L4  waku core       unchanged: app.py -> loop -> memory (gate, consolidation) -> tools -> tracer
L5  Tenant storage  /data/tenants/<id>/  — the unit of isolation, backup, migration, sharding
L6  Ops             OTel collector with a tenant attribute · per-directory backups ·
                    WAKU_HOME=/data/tenants/<id> waku dashboard for operator debugging
```

| layer | reused from waku | new |
|---|---|---|
| L1 | Telegram and Discord numeric ids are stable | web auth; `/link` account linking |
| L2 | `usage.jsonl` per tenant as the audit trail; `TurnBudget` | schema, admin API, key encryption |
| L3 | the `Waku(settings=, client=, conn=)` seam; `GatewayAgentRunner`; `Session.switch()`; `respond(stream=True)` | `TenantPool`, `SpendMeter`, the provider semaphore |
| L4 | everything | five small changes (§8) |
| L5 | `Settings.home` | directory layout, backups |
| L6 | JSONL traces, OTel export, the dashboard itself | the tenant attribute |

## 4. Tenant runtime — `TenantPool`

The skeleton below is the design. The shape is final; the bodies are the
minimum that makes the shape run. It uses only waku's public seams and the
standard library. It lives in `waku-hosted`, and nothing under `waku/` imports
it. Two lines depend on waku change §8.2 and are marked.

```python
"""waku_hosted/pool.py — one Waku per tenant, on waku's own seams."""

from __future__ import annotations

import re
import sqlite3
import threading
from collections import OrderedDict
from collections.abc import Callable
from dataclasses import dataclass
from datetime import UTC, datetime, timedelta
from pathlib import Path

from waku.app import Waku
from waku.config import Settings
from waku.gateway.discord import TurnBudget      # discord itself is imported lazily; no extra needed
from waku.gateway.runner import GatewayAgentRunner
from waku.loop.agent import LoopResult
from waku.ops.pricing import price_for, usage_summary

# A tenant id becomes a directory name. Validate it once, here, so no path a
# tenant chose can ever contain a separator or a dot-dot.
TENANT_ID = re.compile(r"^[a-z0-9][a-z0-9_-]{0,63}$")


class QuotaExceeded(Exception):
    """Raised BEFORE a turn runs, on the tenant's worker thread. The gateway
    turns it into one reply and no model call happens: the free tier fails
    closed. str(exc) is safe to show the user."""


@dataclass(frozen=True)
class TenantConfig:
    """What the runtime needs to know about one tenant.

    Resolved from the control plane, never from os.environ — the environment
    is the platform's. `api_key` is the platform key or the tenant's own,
    decrypted just in time; it lives in this object and in the Waku built from
    it, and is never written under the tenant's home."""

    tenant_id: str
    provider: str
    model: str
    small_model: str
    api_key: str
    key_mode: str                 # "platform" | "byok"
    turns_per_hour: int = 30
    dollar_cap: float = 0.0       # platform mode: the free tier. 0 = no cap (byok)


def tenant_settings(root: Path, cfg: TenantConfig) -> Settings:
    """A Settings that reaches nothing on the host machine.

    Every other field keeps its default, which Settings reads from the
    PLATFORM environment — acceptable for knobs like history_turns, wrong for
    anything that identifies a person or a key. Those are set here."""
    if not TENANT_ID.match(cfg.tenant_id):
        raise ValueError(f"bad tenant id {cfg.tenant_id!r}")
    return Settings(
        home=root / cfg.tenant_id,
        provider=cfg.provider,
        api_key=cfg.api_key,
        model=cfg.model,
        small_model=cfg.small_model,
        # hosted posture: nothing that shells out, opens a browser or reads
        # the operator's own accounts
        apple_calendar=False,
        apple_tools=False,
        google_calendar=False,
        gh_tool=False,
        experimental=False,
    )


class SpendMeter:
    """Dollars one tenant has spent, counted from the loop's own `llm` events
    as they happen. Seeded ONCE from usage.jsonl when the tenant is built —
    that file is the audit trail, this is the live counter, and re-reading the
    ledger before every turn is what made a per-turn check too slow."""

    def __init__(self, home: Path, cap: float) -> None:
        self.cap = cap
        self.spent = float(usage_summary(home)["total_cost"]) if cap else 0.0

    def observer(self, provider: str, model: str) -> Callable[[str, dict], None]:
        def on_event(kind: str, ev: dict) -> None:
            if kind != "llm":
                return
            usage = ev.get("usage") or {}
            price_in, price_out = price_for(provider, ev.get("model") or model)
            self.spent += usage.get("in", 0) / 1e6 * price_in + usage.get("out", 0) / 1e6 * price_out
            # TODO: flush (tenant_id, self.spent) to the control plane
        return on_event

    def check(self) -> None:
        if self.cap and self.spent >= self.cap:
            raise QuotaExceeded("Your free tier is used up. Add your own API key to continue.")


class _Gated:
    """Wraps a Waku so every turn first passes the budget checks and takes a
    provider slot — on the runner's worker thread, so a full queue blocks THIS
    tenant and never the event loop or another tenant. Everything else the
    runner touches (session, conn, close) passes straight through."""

    def __init__(self, agent: Waku, slots: threading.BoundedSemaphore,
                 budget: TurnBudget, meter: SpendMeter) -> None:
        self._agent, self._slots, self._budget, self._meter = agent, slots, budget, meter

    def respond(self, *args, **kwargs) -> LoopResult:
        self._meter.check()
        if not self._budget.allow():
            raise QuotaExceeded("Too many messages this hour. Try again in a little while.")
        with self._slots:
            return self._agent.respond(*args, **kwargs)

    def __getattr__(self, name):
        return getattr(self._agent, name)


def current_thread(home: Path, idle_minutes: int = 60) -> str:
    """The thread a new message continues: the most recent one on ANY channel
    if it is still fresh, else a new dated one. This is browser_agent's
    resume_or_new_session with the source filter removed — which is exactly
    how a Telegram message picks up where the web left off."""
    row = None
    db = home / "state.db"
    if db.exists():
        conn = sqlite3.connect(f"file:{db}?mode=ro", uri=True)
        try:
            row = conn.execute(
                "SELECT session_id, MAX(created_at) FROM chat_log "
                "GROUP BY session_id ORDER BY 2 DESC LIMIT 1"
            ).fetchone()
        except sqlite3.OperationalError:   # brand-new tenant, no table yet
            row = None
        finally:
            conn.close()
    if row and row[1]:
        last = datetime.strptime(row[1], "%Y-%m-%d %H:%M:%S").replace(tzinfo=UTC)
        if datetime.now(UTC) - last <= timedelta(minutes=idle_minutes):
            return row[0]
    return datetime.now(UTC).strftime("t-%Y%m%d-%H%M%S")


@dataclass
class _Live:
    runner: GatewayAgentRunner
    meter: SpendMeter
    cfg: TenantConfig
    home: Path


class TenantPool:
    """tenant_id -> live runner, bounded.

    A live tenant is one worker thread, one SQLite connection, one Waku. When
    the pool is full the least recently used tenant is closed (connection and
    all); its next message rebuilds it from its directory. Nothing is lost on
    eviction because nothing lives only in memory except the working-memory
    window, and Session.switch() reloads that from chat_log."""

    def __init__(self, root: Path, resolve: Callable[[str], TenantConfig], *,
                 client_factory: Callable[[TenantConfig], object] | None = None,
                 max_live: int = 500, provider_concurrency: int = 32) -> None:
        self.root = root
        self._resolve = resolve
        # Evals inject a ScriptedClient here, the same way make_waku() does.
        self._client_factory = client_factory
        self._max_live = max_live
        self._live: OrderedDict[str, _Live] = OrderedDict()
        self._lock = threading.Lock()
        # One gate per PROCESS: the provider's rate limit is per key, not per
        # tenant, so a hundred tenants must not become a hundred parallel calls.
        self._slots = threading.BoundedSemaphore(provider_concurrency)

    def _build(self, cfg: TenantConfig) -> _Live:
        settings = tenant_settings(self.root, cfg)
        settings.ensure_home()
        meter = SpendMeter(settings.home, cfg.dollar_cap)
        budget = TurnBudget(cfg.turns_per_hour)

        def factory() -> _Gated:
            # Runs on the runner's worker thread, so the SQLite connection is
            # created, used and closed on one thread — runner.py's whole point.
            client = self._client_factory(cfg) if self._client_factory else None
            return _Gated(Waku(settings=settings, client=client), self._slots, budget, meter)

        runner = GatewayAgentRunner(factory, session_id="hosted", source="hosted",
                                    observer=meter.observer(cfg.provider, cfg.model))
        return _Live(runner=runner, meter=meter, cfg=cfg, home=settings.home)

    def get(self, tenant_id: str) -> _Live:
        evicted: list[_Live] = []
        with self._lock:
            live = self._live.get(tenant_id)
            if live is None:
                live = self._build(self._resolve(tenant_id))
                self._live[tenant_id] = live
            self._live.move_to_end(tenant_id)
            while len(self._live) > self._max_live:
                _, old = self._live.popitem(last=False)
                evicted.append(old)
        for old in evicted:
            old.runner.close()        # waits for an in-flight turn; kept outside the lock
        return live

    def evict(self, tenant_id: str) -> None:
        """Forget a tenant's live runner — after a key change, a plan change or
        a reset. The next message rebuilds it from resolve(); state is on disk."""
        with self._lock:
            live = self._live.pop(tenant_id, None)
        if live is not None:
            live.runner.close()

    async def turn(self, tenant_id: str, text: str, *, thread_id: str | None = None,
                   source: str = "web") -> LoopResult:
        """One message for one tenant, from any channel.

        thread_id=None continues the tenant's current thread, whichever channel
        it started on — that is the history sync. A web client that lists
        threads (memory.list_sessions()) passes an explicit id instead."""
        live = self.get(tenant_id)
        thread = thread_id or current_thread(live.home)
        # Needs waku change §8.2: respond() takes session_id and source per
        # turn, and switches the Session on the worker thread.
        return await live.runner.respond(text, session_id=thread, source=source)

    def close(self) -> None:
        with self._lock:
            live, self._live = list(self._live.values()), OrderedDict()
        for item in live:
            item.runner.close()
```

A gateway is then a few lines. Telegram, with identity resolved by the control
plane:

```python
async def handle(update, context):
    tenant = identify("telegram", str(update.effective_user.id))   # control plane lookup
    if tenant is None:
        await update.message.reply_text("Link this chat to your account first: /link <code>")
        return
    try:
        result = await pool.turn(tenant, update.message.text, source="telegram")
    except QuotaExceeded as exc:
        await update.message.reply_text(str(exc))
        return
    await update.message.reply_text(result.reply)
```

And the first deterministic eval, offline, in the style of
`evals/deterministic/test_gateway_runner.py` — the script is one gate decision
then one reply per turn:

```python
def test_two_tenants_never_share_a_directory(tmp_path):
    def resolve(tenant_id):
        return TenantConfig(tenant_id, "anthropic", "main", "small", "k", "platform")

    def client(cfg):
        return ScriptedClient([
            response([text_block('{"retrieve": false, "query": "", "reason": "t"}')]),
            response([text_block(f"hello {cfg.tenant_id}")]),
        ])

    pool = TenantPool(tmp_path, resolve, client_factory=client)
    assert asyncio.run(pool.turn("alice", "hi")).reply == "hello alice"
    assert asyncio.run(pool.turn("bob", "hi")).reply == "hello bob"
    assert (tmp_path / "alice" / "state.db").exists()
    assert (tmp_path / "bob" / "state.db").exists()
    alice = sqlite3.connect(tmp_path / "alice" / "state.db")
    assert alice.execute("SELECT COUNT(*) FROM chat_log").fetchone()[0] == 2
```

### 4.1 Turns and threads — how history stays in sync

A turn is `(tenant_id, thread_id, text, source)`.

- The runner switches the `Session` when `thread_id` differs from the current
  one. `Session.switch()` reloads the last `history_turns` exchanges of that
  thread from `chat_log`, including the `[tools used: …]` records, so the model
  remembers what it already did.
- Telegram passes no thread: it continues the tenant's current thread, whichever
  channel it started on (`current_thread()`, with the dashboard's 60-minute idle
  rule). A phone continues what the laptop started.
- The web client lists threads with `memory.list_sessions()` and passes one
  explicitly; "New chat" starts a fresh id.
- Long-term memory (`facts`, `episodes`, `SOUL.md`, skills) was already shared
  across every thread; nothing changes there.

### 4.2 Budgets

Two stops per tenant and one per process:

- `TurnBudget(per_hour)` — the abuse stop, reused from `waku/gateway/discord.py`
  (it goes silent at the limit rather than replying; hosted replies once, then
  goes silent).
- `SpendMeter(dollar_cap)` — platform-key tenants only. It counts every `llm`
  event the loop emits, priced with `pricing.price_for`, seeded once from
  `usage.jsonl` at build time and flushed to the control plane.
- `BoundedSemaphore(provider_concurrency)` — one per process, taken on the
  tenant's worker thread. The model provider rate-limits per key; without this
  a busy hour turns into a wall of 429s for everyone.

### 4.3 Key changes and rebuilds

Free tier exhausted → the tenant enters a key on the web settings page → the
control plane stores it encrypted → `pool.evict(tenant_id)`. The next message
rebuilds the tenant from `resolve()` with the new `TenantConfig`. This is
`browser_agent.rebuild()` without the module global: build fresh, close old.

## 5. Identity and account linking

- **Telegram / Discord**: the numeric id is the identity. First contact with an
  unknown id either creates a platform-mode tenant or asks the person to link
  an existing account — a product decision (§11.3), not a runtime one.
- **Web**: an OIDC provider (Supabase Auth is the obvious choice if the control
  plane is Supabase) issues a JWT; its `sub` is the identity.
- **Linking**: `/link <code>` in the bot, with a one-time code minted on the web
  settings page (or the Telegram Login Widget, which proves the id to the web
  side directly).

Without linking, a Telegram user and a web user are two tenants with two
directories. "History is synchronized across channels" is true of the runtime
and depends entirely on this step.

## 6. Web surface — which dashboard views survive

The dashboard front end (`waku/ops/static/`, about 4,200 lines, no build step)
has eleven views registered in one dict (`js/views.js`, `VIEWS`). The backend
reads are functions of a `home`. Hosted serves the **same API contract**, scoped
to the authenticated tenant, and hides what does not belong to a tenant.

| view | hosted | why |
|---|---|---|
| chat dock | keep | the product. The per-turn cards (gate decision, tools, latency, model) already come from `chat_log.meta` |
| memory | keep | facts, episodes, SOUL, skills — "your memory is yours" is the pitch, and editing it is the feature |
| gateway | keep | the cross-channel inbox is the history-sync story, rendered |
| overview | keep, trimmed | the tenant's own cost, latency and gate split; drop the platform eval verdict |
| loop, tools | behind an "advanced" toggle | transparency for the curious; hide the MCP connector list |
| models | replace | the picker's save path writes `.env` (`/api/providers`); pins in `home/models.json` are per tenant and survive |
| settings | replace | becomes key mode, BYOK entry, usage |
| database | operator only | a read-only SQL console over the tenant's own db is safe (`mode=ro`, SELECT only), but it is a developer tool |
| graph, ops, connections, compare, memory arena | hide | platform-level, or they spend tokens |

**Reuse cost.** Sixteen call sites in `dashboard.py` reach `load_settings()`,
`get_agent()` or `agent_lock`; each becomes "the current tenant's home or
runner". The read paths (`collect`, `_thread_history`, `run_query`,
`events_since`) already take or derive a home. On the front end, hiding views is
a flag on the `VIEWS` dict. Serving the existing UI under a `hosted` flag with
the tenant-scoped API is two to three days. A product-grade UI is a front-end
rewrite against the same API — a separate decision (§11.2), kept honest by the
route contract pinned in `evals/deterministic/test_dashboard_routes.py`.

The operator keeps the unmodified dashboard:
`WAKU_HOME=/data/tenants/<id> waku dashboard` on the box, bound to loopback, to
look at any one tenant exactly the way a single user would.

## 7. Storage, scaling and operations

**Single node first.** One process, every tenant, an LRU of a few hundred live
runners. Turns are serialized per tenant and parallel across tenants; the
provider semaphore is the real ceiling, not Python.

**Scale-out is sharding, not a new database.** Tenants hash to a process or
node; a directory is opened by exactly one process at a time. The web edge
routes by the JWT's tenant. Telegram must move from long polling to webhooks —
one poller per token is a hard Telegram rule (`telegram.py` already detects
the `Conflict`) — with the edge routing each update by `user.id`.

**SQLite stays on local disk, never NFS.** Backups are per directory: Litestream
to object storage, or nightly snapshots. Restore is copying a directory;
migrating a tenant to another node is the same operation.

**Postgres for tenant data is not on the path.** Only `facts` and `episodes`
have a backend protocol; `chat_log` and `calendar_events` are direct SQL. If a
hosted memory backend is ever wanted, Mem0 and Zep already scope by `user_id`
(`MEM0_USER_ID`, `ZEP_USER_ID`) and only need the per-tenant override in §8.1.

**Traces.** JSONL per tenant under its own `traces/`, plus OTel to one collector
with a `tenant.id` attribute (§8.4). Operator dashboards group by that.

## 8. Changes to waku itself

Each is its own PR, defaults to today's behaviour, and carries a deterministic
eval. None of them mention tenants; each is a seam waku should have anyway.

| # | change | where | why |
|---|---|---|---|
| 1 | `Settings` can override the edges that read the environment directly: the Tavily key, MCP `auth_env` values, the `user_id` of mem0/zep/supabase stores | `tools/search.py:63`, `tools/mcp_client.py`, `memory/semantic/*_store.py` (`env_or`) | those are process-global today; in one process serving many people they must not be |
| 2 | `GatewayAgentRunner.respond(text, *, session_id=None, source=None)` switches the `Session` on the worker thread and tags the turn's source per call | `gateway/runner.py` | the thread model in §4.1; also fixes the empty working memory after a Telegram restart |
| 3 | `catalog.list_models(provider, api_key=…)` | `ops/catalog.py:99` | testing a BYOK key without mutating `os.environ`, which `apply_provider` does today |
| 4 | `Tracer` stamps an optional tenant attribute on JSONL records and OTel spans | `ops/tracing.py` | one collector, many tenants |
| 5 | `WAKU_DASHBOARD_HOST`, default `127.0.0.1` | `ops/dashboard.py:1159` | operator use behind a proxy; `PORT` is already honoured but the bind is loopback, which contradicts it |

## 9. What hosted mode disables

Voice (microphone, `say`), Apple tools and Apple Calendar (`osascript`),
`reveal` (opens the host's editor), the `gh` tool (the host's credentials),
experimental tools (pi runs as a subprocess with the host environment), MCP
stdio servers (subprocesses), Google Calendar OAuth (`run_local_server` assumes
a local browser) and MCP OAuth (loopback redirect on 41765). Remote MCP servers
over HTTP are allowed per tenant once §8.1 lands.

## 10. Work plan

Sizes are Claude-driven implementation with a human reviewing and testing:
S = an hour or two, M = half a day to a day, L = one to two days. MVP is the
smallest deployable thing: Telegram only, platform key with a free tier, BYOK
by `/key`, no web UI.

| id | task | size | depends on | MVP |
|---|---|---|---|---|
| U1 | Settings overrides for env-read edges (§8.1) + eval | M | — | yes |
| U2 | `respond(text, session_id=, source=)` + eval; fixes restart amnesia | M | — | yes |
| U3 | `list_models(provider, api_key=)` + eval | S | — | yes |
| U4 | Tracer tenant attribute + eval | S | — | — |
| U5 | `WAKU_DASHBOARD_HOST` + eval | S | — | — |
| H1 | `waku-hosted` package skeleton, `TenantPool`, `tenant_settings`, offline evals with `ScriptedClient` | L | U2 | yes |
| H2 | `SpendMeter` + `TurnBudget` wiring + provider semaphore + evals | M | H1 | yes |
| H3 | Telegram gateway in webhook mode, `/start`, `/link`, `/key`, quota replies | L | H1, H2, C1 | yes |
| H4 | Discord gateway on the pool | M | H3 | — |
| C1 | Control plane schema: tenants, identities, plans, keys, spend | M | — | yes |
| C2 | Key encryption at rest, decrypt into `TenantConfig`, key test via U3 | M | C1, U3 | yes |
| C3 | Admin CLI: list, plan, top up, disable, evict | M | C1 | yes |
| C4 | Web auth (JWT verification) and account linking codes | L | C1 | — |
| W1 | Tenant-scoped HTTP gateway: chat stream, session, memory, data (trimmed), usage, settings | L | H1, C4 | — |
| W2 | Existing static UI under a `hosted` flag: hide views per §6, replace models/settings | L | W1 | — |
| W3 | Product front end (optional, replaces W2) | XL | W1 | — |
| O1 | Dockerfile, compose (app, proxy, Postgres), volume layout | M | H3 | yes |
| O2 | Per-directory backup and a restore drill | M | O1 | yes |
| O3 | OTel collector and tenant dashboards | M | U4 | — |
| O4 | Offline load test: 300 simulated tenants through the pool with `ScriptedClient`; then a small live soak | L | H2 | yes |
| O5 | Security pass: tenant id validation, no env writes reachable, secret redaction in replies (`safe_exception`), rate limits at the edge | M | H3, W1 | yes |

Totals: about 21 tasks. MVP (U1–U3, H1–H3, C1–C3, O1, O2, O4, O5) is thirteen
tasks, roughly eight to ten working days of implementation plus review. The
full plan with the reused web UI (W1, W2) is about fifteen to twenty days; a
product front end (W3) is its own project. Human time — reading diffs, testing
the bot, running the restore drill — is a third again on top.

## 11. Open questions

1. Do platform-mode tenants pick a model, or only BYOK tenants? (Cheaper to
   fix one default; the switcher is a BYOK feature.)
2. Is the first web UI the reused dashboard under a flag (W2) or a fresh front
   end (W3)? W2 ships in days and is honest about being an operator's UI.
3. First contact on Telegram: create a platform tenant automatically (the free
   tier is then an abuse surface) or invite-only until linking exists?
4. Where does the control plane live — Supabase (Auth and Postgres in one) or
   self-hosted Postgres?
