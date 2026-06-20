# Becoming a Great Backend & Infrastructure Engineer

A structured, **visual**, hands-on study plan. The goal: own a system end to end —
model the data, build the API, ship it, run it, and fix it when it breaks.

> **How to use this:** 12 milestones, in order. Each has a diagram, an explanation,
> and a **Hands-on (this repo)** box. Don't move on until you've done the hands-on.
> Many exercises reference *this* codebase — a Django funeral-insurance backend on
> AWS (DRF, background jobs, SMS/WhatsApp, payment gateways like Pay@/EasyPay).

---

## The big picture: what a request actually touches

Before the milestones, hold this whole-system map in your head. Backend + infra is
about understanding **every box** and **what happens when each one fails**.

```
                         INTERNET
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │  DNS  →  TLS/HTTPS  →  Load Balancer  →  (CDN for static assets)      │   INFRA
   └─────────────────────────────────────────────────────────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │  App servers (Gunicorn/ASGI) running Django ── horizontally scaled    │   INFRA
   └─────────────────────────────────────────────────────────────────────┘
                            │
        ┌───────────────────┼─────────────────────┬──────────────────────┐
        ▼                   ▼                     ▼                       ▼
   ┌──────────┐      ┌─────────────┐      ┌──────────────┐       ┌───────────────┐
   │ URL→View │      │  Database   │      │  Cache /     │       │ Task queue    │   BACKEND
   │ Serializer│ ───▶ │ (Postgres/  │      │  Redis       │       │ (Celery) +    │
   │ Service  │      │  RDS)       │      │              │       │ workers       │
   └──────────┘      └─────────────┘      └──────────────┘       └───────┬───────┘
                                                                          │
                                                            ┌─────────────┴──────────────┐
                                                            ▼                            ▼
                                                   ┌──────────────────┐      ┌──────────────────┐
                                                   │ External APIs:   │      │ Email / SMS /    │   FAIL POINTS
                                                   │ payment gateways │      │ WhatsApp         │
                                                   └──────────────────┘      └──────────────────┘
                            │
                            ▼
   ┌─────────────────────────────────────────────────────────────────────┐
   │  Observability: logs + metrics + traces + alerts  (watches it ALL)   │   INFRA/SRE
   └─────────────────────────────────────────────────────────────────────┘
```

> 🧠 **The core skill:** for *every arrow* above, ask **"what happens when this
> fails, is slow, or runs twice?"** Great engineers design for the arrows, not
> just the boxes.

---

## Roadmap & dependency order

```
   PART 1: BACKEND                          PART 2: INFRASTRUCTURE
   ┌────────────────────────┐               ┌───────────────────────────┐
   │ M1 Data layer  ★START  │               │ M7  Linux & networking    │
   │ M2 API design          │               │ M8  Containers (Docker)   │
   │ M3 Concurrency/failure │               │ M9  Cloud (AWS)           │
   │ M4 Background jobs      │               │ M10 Infra as Code         │
   │ M5 Security (ALWAYS)    │               │ M11 CI/CD                 │
   │ M6 Performance/observ.  │               │ M12 Observability/SRE     │
   └────────────────────────┘               └───────────────────────────┘

   Suggested path:
   M1 ─▶ M3 ─▶ M4 ─▶ M8 ─▶ M9 ─▶ M10 ─▶ M11 ─▶ M12
        └─ M2, M5, M6 woven throughout ─┘
   (M5 Security is not a phase — it applies to every milestone.)
```

---

## The mindset

The great engineers don't separate "backend" from "infrastructure." They write
code knowing how it deploys, scales, is monitored, and **how it fails in
production**. The bar moves from *"it works on my machine"* to *"it works
reliably at 3am, under load, when a dependency is down."*

Two questions for everything you build:
- **What happens when this fails?** (the network call, the DB, the third party)
- **How will I know it's broken before a customer tells me?**

---

# PART 1 — BACKEND ENGINEERING

## ★ Milestone 1: Master the data layer (highest leverage — start here)

This is where backend engineers live or die.

```
   ORM call                 generated SQL              database work
   ─────────                ─────────────              ─────────────
   Policy.objects           SELECT * FROM policy       ┌───────────┐
     .filter(active=True)   WHERE active = true        │  INDEX?   │ ← fast (seek)
     .select_related(       JOIN customer ON ...        │  no index │ ← slow (full scan)
        'customer')                                     └───────────┘
        │                          │                          │
   you write this  ──▶  this actually runs  ──▶  this is what costs time
```

### The N+1 problem (the #1 backend performance bug)
```
   BAD  (N+1 queries):                 GOOD (2 queries):
   ───────────────────                 ─────────────────
   policies = Policy.objects.all()     policies = Policy.objects.select_related(
   for p in policies:                                  'customer').all()
       print(p.customer.name)  ◀── 1   for p in policies:
            extra query EACH loop!         print(p.customer.name)  ◀── no extra query
   → 1 + N queries (100 rows = 101!)   → 1 query for policies + joined customer
```

**Learn:** SQL deeply (not just the ORM), indexes & their write cost, transactions,
isolation levels, what ACID guarantees, normalization vs. denormalization,
migrations that don't lock a production table.

**Hands-on (this repo):**
1. In `manage.py shell`: `qs = SomeModel.objects.all(); print(qs.query)` — read the SQL.
2. Find a list view with related data. Detect N+1 (Django Debug Toolbar or SQL
   logging). Fix with `select_related`/`prefetch_related`; confirm query count drops.
3. Pick one model (a policy or payment). List its indexes and which queries they
   serve. Find one query that scans without an index.
4. `manage.py sqlmigrate <app> <number>` — read the SQL. Would it lock a big table?

> ✅ **Done when:** you can look at any view and predict the SQL it runs.

---

## Milestone 2: Design good APIs — the contract is the product

```
   A good endpoint:
   ┌────────────────────────────────────────────────────────────┐
   │ • Right HTTP verb + status code (200/201/400/404/409/...)   │
   │ • Validates input at the boundary (serializer)              │
   │ • Consistent error shape across the whole API               │
   │ • Paginated + filterable for lists                          │
   │ • Idempotent where retries can happen (payments!)           │
   │ • Versioned so you can evolve without breaking clients      │
   └────────────────────────────────────────────────────────────┘
```

### Idempotency — why it matters for payments
```
   Client sends "charge R500"  ──▶  [network times out]  ──▶  client RETRIES
                                                                    │
   WITHOUT idempotency: charged TWICE  ✗                            ▼
   WITH idempotency key: server sees same key → returns first result, charges ONCE ✓
```

**Hands-on (this repo):**
1. Audit one DRF endpoint: correct status codes? Input validated in the serializer?
   Error shape consistent with others?
2. Find a create-with-side-effects endpoint (payment, policy). If the client
   retries after a timeout, does it duplicate? Sketch how an idempotency key fixes it.
3. Add/verify pagination on a list endpoint that could grow large.

> ✅ **Done when:** you can design an endpoint that's clear, validated, hard to misuse.

---

## Milestone 3: Correctness under concurrency & failure

This separates senior from junior.

### The race condition (two requests, one row)
```
   Time ──▶
   Request A:  read balance=100 ─────────────── write balance=100-30=70
   Request B:       read balance=100 ── write balance=100-50=50
                                                          │
                    Both read 100. Final = 50 (or 70). One update LOST! ✗

   FIX:  with transaction.atomic():
             row = Account.objects.select_for_update().get(...)   ← locks the row
             row.balance -= amount                                  others wait
             row.save()
```

### Assume external calls fail
```
   call payment gateway ──▶ ┌─ success ─▶ done
                            ├─ timeout ─▶ retry w/ backoff (idempotent!)
                            ├─ error   ─▶ fallback or queue for later
                            └─ DUPLICATE webhook ─▶ detect & ignore (idempotent)
```

**Hands-on (this repo):**
1. Find a webhook/payment callback handler. If the same callback arrives twice,
   what happens? Is it idempotent?
2. Find a read-then-write on the same row (balance, counter, status). Could
   concurrency corrupt it? How would `atomic()` + `select_for_update()` fix it?
3. Find an external call (SMS/WhatsApp/gateway). What happens on timeout? Retry?
   Fallback? (You already have an SMS→fallback path — study it.)

> ✅ **Done when:** for any feature you can answer "what if this fails or runs twice?"

---

## Milestone 4: Background & asynchronous work

Keep slow/unreliable work **off the request path**.

```
   SYNCHRONOUS (bad for slow work):
   request ─▶ do work + send email + call gateway ─▶ response   ← user waits for ALL of it

   ASYNCHRONOUS (good):
   request ─▶ enqueue task ─▶ response (fast!)
                  │
                  ▼
            ┌──────────┐   picks up task   ┌─────────┐
            │  queue   │ ────────────────▶ │ worker  │ ─▶ send email, retry on fail
            │ (Redis)  │                   │(Celery) │     idempotently
            └──────────┘                   └─────────┘
```

**What belongs async:** email, SMS, report generation, reconciliation — anything
slow or flaky.

**Hands-on (this repo):**
1. List everything done *inside* a request that could move to a background task
   (e.g. sending email/SMS on a signal). Note the latency it adds today.
2. Design (on paper) how one becomes a retried, idempotent background task. What if
   it runs twice?

> ✅ **Done when:** you instinctively ask "should this be synchronous?" for slow work.

---

## Milestone 5: Security (non-negotiable — money + PII)

Applies to **every** milestone, not just this one.

```
   OWASP-style checklist for each endpoint:
   ┌──────────────────────────────────────────────────────────────┐
   │ [ ] AuthN: is the caller who they say they are?               │
   │ [ ] AuthZ: are they allowed to touch THIS object? (IDOR)      │
   │ [ ] Input validated & escaped (injection, XSS)?               │
   │ [ ] CSRF protection where needed?                             │
   │ [ ] Secrets in env/secret manager, NEVER in code?             │
   │ [ ] PII encrypted in transit (TLS) and at rest?               │
   │ [ ] Least privilege (DB user, IAM role, API scope)?           │
   └──────────────────────────────────────────────────────────────┘
```

### IDOR — the classic access-control bug
```
   GET /api/policies/123/     ← user A's policy
   GET /api/policies/124/     ← user B's policy

   If changing the ID lets user A see B's data → IDOR vulnerability.
   FIX: scope the queryset to the caller:
        Policy.objects.filter(customer__owner=request.user)
```

**Hands-on (this repo):**
1. Pick one endpoint: can user A read user B's data by changing an ID? Verify the
   permission/queryset scoping blocks it.
2. Grep for hardcoded secrets/keys → move any to env/secret manager.
3. List what PII this system stores and where. Who can read it? Protected at rest?
   (POPIA applies in South Africa.)

> ✅ **Done when:** you treat every input as hostile and every secret as sacred.

---

## Milestone 6: Performance & observability

```
   The optimization loop (NEVER skip step 1):
   ┌─────────┐   ┌──────────┐   ┌──────────┐   ┌─────────┐
   │ MEASURE │─▶ │ find the │─▶ │ fix the  │─▶ │ MEASURE │─▶ repeat
   │ first!  │   │bottleneck│   │ one thing│   │ again   │
   └─────────┘   └──────────┘   └──────────┘   └─────────┘
        ▲                                            │
        └──────────── don't guess; profile ─────────┘
```

**Learn:** profiling, query counts/timing, caching (Redis, HTTP, cached queries),
connection pooling, structured logging, metrics, tracing.

**Hands-on (this repo):**
1. Find the slowest endpoint you know. Measure *why* (queries? external calls?)
   before changing anything.
2. Identify one expensive, rarely-changing computation that could be cached.
3. Review logging: on a production failure, is there enough context to diagnose
   *without* reproducing?

> ✅ **Done when:** "it's slow" becomes a measurement, not a guess.

---

# PART 2 — INFRASTRUCTURE

## Milestone 7: Linux & networking fundamentals

Everything runs on these.

### What happens between a browser and your app
```
   browser
     │ 1. DNS: name → IP
     ▼
   [DNS resolver] ── "fbt-api.prestigetech.cloud → 13.245.x.x"
     │ 2. TCP connect to IP:443
     ▼
   [TLS handshake] ── certificate verified, encrypted channel established
     │ 3. HTTPS request
     ▼
   [Load balancer] ── picks a healthy app server
     │
     ▼
   [App server: Gunicorn] ── WSGI ──▶ Django ──▶ view ──▶ DB
     │ 4. HTTPS response travels back up the chain
     ▼
   browser renders
```

**Learn:** shell fluency, processes, permissions, systemd, file descriptors,
signals; DNS, TCP/IP, HTTP(S), TLS/certs, ports, load balancers, firewalls.

**Hands-on:**
1. Trace a request to this app end to end and name every hop (use the diagram).
2. Inspect a domain's TLS cert (`openssl s_client -connect host:443` or browser
   devtools). When does it expire? Who renews it?

> ✅ **Done when:** you can explain the full path of a request and where it breaks.

---

## Milestone 8: Containers (Docker)

```
   "Works on my machine" problem        Containers solve it
   ─────────────────────────────        ───────────────────
   your laptop:  Python 3.12, libX 2.0   ┌─────────────────────────┐
   server:       Python 3.9,  libX 1.4   │  IMAGE (a recipe):      │
        → subtle bugs, "but it worked!"  │   base OS + Python +    │
                                         │   deps + your code      │
   A container packages the app AND its  │  ───────────────────    │
   exact environment, so it runs the     │  runs IDENTICALLY on    │
   same everywhere.                      │  laptop, CI, and prod   │
                                         └─────────────────────────┘
```

**Learn:** images, layers, Dockerfiles, volumes, networks, `docker-compose` for
local dev. Kubernetes **later**, only when scale demands it (it's heavy — don't
reach for it early).

**Hands-on (this repo):**
1. Write/review a Dockerfile that runs this Django app.
2. Use `docker-compose` to run app + Postgres + Redis locally with one command.

> ✅ **Done when:** you can containerize and run this app reproducibly anywhere.

---

## Milestone 9: Cloud (AWS — this system's platform)

```
   Core AWS building blocks (map them to your system):
   ┌──────────────┬──────────────────────────────────────────────┐
   │ Compute      │ EC2 / ECS / Lambda  ─ runs your code          │
   │ Storage      │ S3                  ─ files, static, backups  │
   │ Database     │ RDS (Postgres)      ─ your data               │
   │ Networking   │ VPC, subnets, security groups ─ the walls     │
   │ Edge         │ API Gateway, CloudFront ─ the front door      │
   │ IAM ★        │ who/what can do what ─ MOST breaches are here │
   └──────────────┴──────────────────────────────────────────────┘
```

**Learn IAM properly** — most cloud breaches are misconfigured permissions, not
clever hacks.

**Hands-on:**
1. Map this system's AWS footprint (you have API Gateway, Lambda, a DB, region
   `af-south-1`). Draw the architecture diagram.
2. Review one IAM role/policy. Least privilege, or wildcard `*`?

> ✅ **Done when:** you can draw the production architecture from memory.

---

## Milestone 10: Infrastructure as Code (IaC)

```
   Clicking in the console            Infrastructure as Code
   ───────────────────────            ──────────────────────
   ✗ not reproducible                 ✓ versioned in git
   ✗ no review                        ✓ peer-reviewed like code
   ✗ "who changed this?!"             ✓ history + blame
   ✗ drift between environments       ✓ identical dev/staging/prod

   workflow:   write .tf  ─▶  terraform plan (PREVIEW)  ─▶  terraform apply
                                      │
                          the infra equivalent of `git diff`
```

**Learn:** Terraform (or CloudFormation/CDK). Never click around the console for
production.

**Hands-on:**
1. Express one piece of infra as code (an S3 bucket, a security group).
2. Practice `plan` (preview) before `apply`.

> ✅ **Done when:** you'd be uncomfortable changing prod infra by hand.

---

## Milestone 11: CI/CD

```
   git push
      │
      ▼
   ┌──────┐   ┌───────┐   ┌───────┐   ┌─────────┐   ┌──────────┐
   │ LINT │─▶ │ TESTS │─▶ │ BUILD │─▶ │ DEPLOY  │─▶ │ MIGRATE  │
   │      │   │       │   │ image │   │(staging→│   │ safely   │
   │      │   │       │   │       │   │  prod)  │   │ on live  │
   └──────┘   └───────┘   └───────┘   └─────────┘   └──────────┘
      │           │                        │
   fail fast → stop the pipeline        zero-downtime + ROLLBACK ready
```

### Safe migrations on a live DB (expand/contract)
```
   Renaming/removing a column while old code still runs = breakage.
   Do it in steps instead:
     1. ADD new nullable column        (old + new code both work)
     2. BACKFILL data                  (background)
     3. Switch code to new column
     4. DROP old column                (after old code is gone)
```

**Hands-on (this repo):**
1. Sketch the pipeline: what runs on every push?
2. Find a migration that would be unsafe during a deploy and rewrite the plan to
   be safe (expand/contract above).

> ✅ **Done when:** shipping is boring, repeatable, and reversible.

---

## Milestone 12: Observability & reliability (SRE mindset)

```
   The three pillars of observability:
   ┌────────────┬──────────────────────────────────────────────┐
   │ LOGS       │ what happened (events, errors, context)       │
   │ METRICS    │ how much / how fast (rates, latency, counts)  │
   │ TRACES     │ the path of one request across services       │
   └────────────┴──────────────────────────────────────────────┘
            │            │             │
            └────────────┴─────────────┴──▶ ALERTS ──▶ on-call ──▶ fix ──▶ postmortem
                                              │
                            "find out from monitoring, not from customers"
```

```
   Backups reality check:
   backup exists  ✓   ──but──  has a RESTORE ever been tested?  ✗
   → An untested backup is NOT a backup. Test the restore.
```

**Learn:** monitoring (Prometheus/Grafana, CloudWatch), centralized logs, alerting,
on-call, incident response, blameless postmortems; SLOs, error budgets, blast
radius, graceful degradation.

**Hands-on:**
1. Define one SLO (e.g. "99.5% of payment callbacks processed within 30s"). How
   would you measure it?
2. Find where backups live and whether a restore has ever been tested.
3. Write a mini runbook for one failure mode ("the SMS provider is down").

> ✅ **Done when:** you find out about problems from monitoring, not customers.

---

## How it all connects (the whole loop)

```
   ┌────────┐   ┌────────┐   ┌─────────┐   ┌────────┐
   │ BUILD  │─▶ │  SHIP  │─▶ │ OPERATE │─▶ │  FIX   │─▶ (learn) ─┐
   │backend │   │ CI/CD  │   │ observe │   │incident│           │
   │+ tests │   │ + IaC  │   │ + alert │   │+postmrt│           │
   └────────┘   └────────┘   └─────────┘   └────────┘           │
        ▲                                                        │
        └───────────────── feeds back into design ──────────────┘
```

Great engineers run this whole loop, not just the "build" box.

---

## Books that punch above their weight
- **Designing Data-Intensive Applications** — Martin Kleppmann (backend/data depth).
- **Google's SRE Book** (free online) — the operations mindset.
- **The Phoenix Project** — why flow, automation, and feedback loops matter.

## The fastest way to learn all of it
Take ownership of **one real system end to end**: model the data, build the API,
containerize it, deploy with IaC, add monitoring, and be the person who responds
when it breaks. One full loop of **build → ship → operate → fix** teaches more
than any course.

> See also: `DEVELOPMENT_GUIDE.md` (daily workflow) and
> `BECOMING_A_10X_DEV.md` (mindset & leverage).
