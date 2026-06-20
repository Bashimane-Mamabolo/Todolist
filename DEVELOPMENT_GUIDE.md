# Developer Workflow Guide

A practical, **visual**, hands-on guide to planning, implementing, testing, and
debugging code effectively — using the terminal, Python, and Django.

> **How to read this guide:** Each section has (1) a diagram or visual model,
> (2) an explanation, (3) concrete commands, and (4) a **Practice** box. Do the
> practice before moving on. Depth beats breadth.

> **Stack assumptions:** Python + Django, Git, Windows (PowerShell) or any shell.
> Windows-specific differences are flagged with 🪟.

---

## Table of contents

```
0.  The mental model: the core loop
1.  Knowing your shells
2.  UNDERSTAND — explore before you change
3.  IMPLEMENT — small, consistent, reversible
4.  IMPLEMENTING A NEW FEATURE — full workflow
5.  VERIFY — the testing pyramid
6.  DEBUG — a decision tree
7.  SEARCH & NAVIGATE faster
8.  DJANGO-SPECIFIC lifesavers
9.  GIT as a safety net
10. REPRODUCIBILITY & ENVIRONMENT
11. HABITS THAT COMPOUND
12. A 2-week practice plan
13. Cheat sheet (printable)
```

---

## 0. The mental model: the core loop

Almost all good engineering is the **same small loop**, repeated thousands of times:

```
        ┌───────────────────────────────────────────────────┐
        │                                                     │
        ▼                                                     │
  ┌───────────┐     ┌────────────┐     ┌──────────┐          │
  │ UNDERSTAND │ ──▶ │   CHANGE   │ ──▶ │  VERIFY  │ ─────────┘
  │  the code  │     │  (small!)  │     │  (prove) │   repeat until done
  └───────────┘     └────────────┘     └──────────┘
       │                  │                  │
   read, grep,       one concern,      run something
   trace, ask        reversible        that prints a result
```

### The three rules that make it work

| # | Rule | Why |
|---|------|-----|
| 1 | **Never edit code you haven't read** | Most bugs come from changing something you didn't understand. |
| 2 | **Change one thing at a time, then verify** | Batching 5 changes = debugging a mess with 5 suspects. |
| 3 | **Prove it, don't hope it** | "It should work" is not done. Output, or it didn't happen. |

### Cheap checks first (the cost ladder)

Run the **fastest** verification that can catch the problem, before the slow one:

```
COST →  cheap ........................................ expensive
        │           │            │            │            │
     read the   python -c    one unit     full test    open browser /
     code       snippet      test         suite        real inbox / deploy
        │           │            │            │            │
     seconds    seconds      seconds      minutes       minutes+
```

> 🧠 **Principle:** *Fail fast where failure is cheap.* A bug caught by reading
> code costs seconds; the same bug caught in production costs hours and trust.

> 📦 **Practice:** Re-read the loop above. For your next task, say out loud which
> phase you're in at every moment ("I'm understanding now", "I'm verifying now").
> Awareness of the phase is half the skill.

---

## 1. Knowing your shells

People say "the shell" for several different things. Know which you're using:

```
┌──────────────────────────┬───────────────────────┬─────────┬──────────────────────────┐
│ Command                  │ What it is            │ Stays   │ Use it for               │
│                          │                       │ open?   │                          │
├──────────────────────────┼───────────────────────┼─────────┼──────────────────────────┤
│ python -c "code"         │ One-off snippet, exits│  NO     │ Scripted/repeatable check│
│ python                   │ Interactive REPL      │  YES    │ Exploring by hand        │
│ python manage.py shell   │ Django REPL (loaded)  │  YES    │ Real queries/models      │
│ python script.py         │ Runs a saved file     │  NO     │ A check worth keeping    │
└──────────────────────────┴───────────────────────┴─────────┴──────────────────────────┘
```

### When to use which (decision flow)

```
Do you need Django models/settings loaded?
   │
   ├── YES ──▶ Are you exploring by hand?
   │            ├── YES ──▶  python manage.py shell   (or shell_plus)
   │            └── NO  ──▶  a script, or python -c with django.setup()
   │
   └── NO  ──▶ Will you run this check again later?
                ├── YES ──▶  save it as a script.py
                └── NO  ──▶  python -c "quick snippet"
```

- `manage.py shell` saves the Django boilerplate — **prefer it** over plain
  `python` for anything touching your models or settings.
- 💡 Install `django-extensions` → `python manage.py shell_plus` auto-imports all
  your models so you can query immediately.

```python
# Inside `python manage.py shell` — explore REAL data interactively:
>>> from core.models import BusinessProfile
>>> b = BusinessProfile.objects.first()
>>> b.name, b.portal_url           # poke at fields
>>> BusinessProfile.objects.count()
```

> 📦 **Practice:** Open `python manage.py shell`, fetch one object
> (`MyModel.objects.first()`), print two of its fields, then `exit()`.

---

## 2. UNDERSTAND — explore before you change

**Goal:** answer three questions *from the code*, not from memory:

```
   ┌─────────────────────────────────────────────────┐
   │  Before changing anything, know:                 │
   │                                                  │
   │   1. WHERE is this used / called?                │
   │   2. WHAT does it depend on (inputs/outputs)?    │
   │   3. WHAT will I break if I change it?           │
   └─────────────────────────────────────────────────┘
```

### How to answer each

**1. Where is X used / called?**
```bash
grep -rn "function_name" .      # recursive, with line numbers
rg "function_name"              # ripgrep: faster, respects .gitignore
git grep "function_name"        # only tracked files, instant
```

**2. What does it receive / return?** Open it and **trace the data flow**:

```
   caller ──▶ builds inputs ──▶ [ your function ] ──▶ returns output ──▶ consumer
      │                              │                                      │
   (who calls it?)         (what does it do?)                  (who depends on the result?)
```

**3. What are the real field/attribute names?** Look them up — **never guess**:
```bash
grep -n "field_name" path/to/models.py
```

### The golden rule

> 🧠 **Verify names and call sites from the source.** A wrong assumption about a
> field name or who calls a function is the *most common self-inflicted bug.*
> (Example: assuming `business_phone` when the model actually has `phone_number`.)

### When to ask vs. assume

```
   Is the answer findable in the code/docs in < 2 min?
        ├── YES ──▶ find it yourself
        └── NO  ──▶ Would guessing wrong cost > 30 min of rework?
                     ├── YES ──▶ ASK (a 10-sec question beats an hour of rework)
                     └── NO  ──▶ pick the sensible default, note it, proceed
```

> 📦 **Practice:** Pick any function in this repo. *Without running it*, write down
> (a) where it's called from, (b) what arguments it needs, (c) what it returns.
> Then verify each by reading the code. Score yourself.

---

## 3. IMPLEMENT — small, consistent, reversible

### The four principles

```
┌────────────────────────────────────────────────────────────────────┐
│ 1. MIRROR existing patterns                                          │
│    └─ Find a similar working thing, copy its shape.                  │
│       Consistency > cleverness. Easier to review and maintain.       │
│                                                                      │
│ 2. BACKWARD-COMPATIBLE by default                                    │
│    └─ Add optional params (new_param=None) or a new code path        │
│       beside the old one. Existing callers/tests keep working.       │
│                                                                      │
│ 3. SMALL & FOCUSED edits                                             │
│    └─ One concern per change = easy to verify, easy to undo.         │
│                                                                      │
│ 4. MATCH the surrounding style                                       │
│    └─ Same naming, indentation, comment density. Your change         │
│       should be hard to spot as "the new code".                      │
└────────────────────────────────────────────────────────────────────┘
```

### Backward-compatible change — visual

```
BEFORE:   def send_email(user, business):            ← existing callers
                ...                                     pass 2 args

AFTER:    def send_email(user, business,             ← existing callers
                         temp_password=None,           STILL pass 2 args
                         login_url=None):               (defaults fill the rest)
                ...                                    ← new callers can pass more
```
✅ Nothing breaks. New behaviour is *additive*.

> 📦 **Practice:** Add an optional parameter to an existing function with a
> sensible default, use it inside the function, and confirm every existing caller
> still works unchanged (grep for the callers first!).

---

## 4. IMPLEMENTING A NEW FEATURE — full workflow

A new feature is just the core loop applied in **thin slices**, with a plan up
front and a test at the end.

```
 Step 0      Step 1        Step 2       Step 3            Step 4         Step 5     Step 6      Step 7
┌───────┐  ┌─────────┐  ┌─────────┐  ┌────────────┐  ┌──────────┐  ┌────────┐  ┌────────┐  ┌────────┐
│ STATE │─▶│ EXPLORE │─▶│  PLAN   │─▶│ BUILD in   │─▶│  VERIFY  │─▶│  TEST  │─▶│ SELF-  │─▶│ COMMIT │
│ goal  │  │ (under- │  │ the     │  │ thin       │  │  each    │  │ (auto- │  │ REVIEW │  │ logical│
│ in 1  │  │ stand)  │  │ change  │  │ slices     │  │  slice   │  │ mate)  │  │ (diff) │  │ unit   │
│ line  │  │         │  │         │  │            │  │          │  │        │  │        │  │        │
└───────┘  └─────────┘  └─────────┘  └─────┬──────┘  └────┬─────┘  └────────┘  └────────┘  └────────┘
                                           │              │
                                           └──── loop ◀───┘  (slice → verify → slice → verify)
```

### Step 0 — State the goal in ONE sentence
> *"When X happens, the system should do Y, so that Z."*
> If you can't write this, you don't understand the task yet. Clarify first.

### Step 1 — Explore (UNDERSTAND)
- Find the closest existing feature and read it **end to end**.
- List every layer your feature touches. For a Django app that's usually:

```
   HTTP request
       │
       ▼
   URL (urls.py)
       │
       ▼
   View / ViewSet  ──────────────┐
       │                         │
       ▼                         ▼
   Serializer (validation)   Permissions
       │
       ▼
   Service / business logic
       │
       ├──▶ Model / ORM ──▶ Database
       │
       ├──▶ Signal ──▶ Background task (email/SMS)
       │
       ▼
   Template / Response
```
- Note the patterns to reuse and the things you might break.

### Step 2 — Plan the change
Write a short plan (even just bullets):
- Which files change, and what each change does.
- Data model changes? New migration? Is it backward-compatible?
- How you'll test it.
- Get sign-off if it's non-trivial or someone else owns the code.

### Step 3 — Build in thin slices
A safe order — **keep the app runnable at every step**:

```
   data model  ──▶  business logic  ──▶  API / view  ──▶  presentation
   (migration)      (service func)       (serializer)      (template)
       │                 │                    │                 │
    verify ✓          verify ✓             verify ✓          verify ✓
```

### Step 4 — Verify each slice (cheapest tool that proves it)

```
   What did you change?            How to verify cheaply
   ─────────────────────          ──────────────────────────────────────
   Pure logic            ──▶       python -c  or  manage.py shell
   Model / migration     ──▶       makemigrations --dry-run, sqlmigrate
   API endpoint          ──▶       curl / httpie / DRF test client
   Template / UI         ──▶       render_to_string, then look at it
```

### Step 5 — Write a test
Turn your manual check into a **permanent** one. Future regressions now fail loudly.

### Step 6 — Self-review
```bash
git diff            # read EVERY line you changed
python manage.py check
pytest -k yourfeature
```
Remove debug prints, stray TODOs, typos.

### Step 7 — Commit a logical unit
Message says **why**, not just **what**.

> 📦 **Practice:** Take a tiny feature ("add a `last_seen` timestamp to users").
> Write the Step 0 sentence and the Step 2 plan *before any code*. Then build it
> in slices, verifying each.

---

## 5. VERIFY — the testing pyramid

Run the lowest-cost check that can catch the problem first.

```
                         ▲  slower, fewer, more realistic
                        ╱ ╲
                       ╱   ╲      (g) MANUAL / VISUAL
                      ╱─────╲         browser, real inbox, click-through
                     ╱       ╲        → for design & integration, LAST
                    ╱─────────╲
                   ╱           ╲   (f) AUTOMATED TESTS
                  ╱             ╲      pytest / manage.py test
                 ╱───────────────╲     → the durable version of below
                ╱                 ╲
               ╱   (a)-(e) QUICK   ╲ CHECKS
              ╱  python -c, shell,  ╲
             ╱  render, override,    ╲
            ╱   inspect data          ╲
           ╱___________________________╲
              faster, more, cheaper  ▼
```

### (a) Run a snippet of logic (no DB needed)
```bash
python -c "from mymodule import calc; print(calc(2,3))"
```
🪟 On Windows set env vars separately: `$env:VAR='x'; python -c "..."`

### (b) Probe real objects interactively
```bash
python manage.py shell
>>> from app.services import Thing
>>> Thing.do_something(...)
```

### (c) Render a template with fake data (Django)
```python
from django.template.loader import render_to_string
html = render_to_string("emails/welcome.html", {"name": "Test"})
print("OK" if "Test" in html else "MISSING")
```

### (d) Exercise a real code path WITHOUT side effects
```python
from django.test import override_settings
# Swap a setting JUST for this test — send email to the console, not for real:
with override_settings(EMAIL_BACKEND="django.core.mail.backends.console.EmailBackend"):
    send_my_email(...)        # prints the email to the terminal
```

### (e) Inspect data instead of eyeballing it
```python
# Need an exact value? MEASURE it. (e.g. exact colours from an image)
from PIL import Image
from collections import Counter
im = Image.open("logo.png").convert("RGBA")
c = Counter((r,g,b) for r,g,b,a in im.get_flattened_data() if a > 180)
print(["#%02x%02x%02x" % px for px,_ in c.most_common(3)])
```

### (f) Automated tests (durable)
```bash
pytest path/to/test_file.py -k keyword -v     # pytest + pytest-django
python manage.py test app.tests               # Django's built-in runner
```
Helpful libs: `pytest-django`, `factory_boy` (build test objects), `coverage`.

### (g) Visual / full-app — LAST
Browser, click-through, real inbox. For **design & integration**, not logic bugs.

> 📦 **Practice:** Take something you'd normally check by clicking around, verify
> it instead with (a)–(d), then write it up as one `pytest` test.

---

## 6. DEBUG — a decision tree

When something breaks, don't panic-edit. Work the tree:

```
   Something broke
        │
        ▼
   Is there an error/traceback?
        │
        ├── YES ──▶ READ IT BOTTOM-UP
        │            • last line  = WHAT broke
        │            • lines above = WHERE (the call chain)
        │            • read it literally before changing anything
        │                 │
        │                 ▼
        │            Can you reproduce it small?
        │                 ├── NO  ──▶ add logging / narrow the input until you can
        │                 └── YES ──▶ put breakpoint() at the spot, inspect vars
        │
        └── NO (wrong output, no error) ──▶ Bisect the logic:
                 • print/inspect the value at the halfway point
                 • is it correct there? → narrow to the half that's wrong
                 • repeat until you find the exact line
```

### Using the debugger (`breakpoint()`)

```python
def my_function(data):
    result = transform(data)
    breakpoint()          # ← execution PAUSES here, drops you into a prompt
    return result
```
At the `(Pdb)` prompt:
```
   n   next line            p var   print a variable
   s   step INTO a call     l       list code around here
   c   continue             q       quit
```

### Print vs. debugger vs. logging

```
   One-off, throwaway check        ──▶  print()  (delete after)
   Need to step through logic       ──▶  breakpoint()
   Something you'll keep / prod      ──▶  logging  (levels, no secrets)
```

> 🧠 **Golden rule:** change ONE thing, then re-test. Shotgunning 5 fixes means
> you won't know which one worked.

> 📦 **Practice:** Add `breakpoint()` inside a function, run the code that calls
> it, inspect a couple of local variables interactively, then remove it.

---

## 7. SEARCH & NAVIGATE faster

```
   Goal                        Tool / command
   ─────────────────────       ─────────────────────────────────
   Find text in code           rg "pattern"            (ripgrep, fast)
   Find text, only py files    rg -n "pattern" --type py
   Find tracked files only     git grep "pattern"
   Find a file by name         fd welcome              (or: rg --files | rg welcome)
   Jump to a definition        editor: "Go to definition" (F12)
   Find all references         editor: "Find all references" (Shift+F12)
```

> 🧠 Learning your editor's *go-to-definition* and *find-all-references* is the
> GUI version of grep — and it's faster. Invest 30 min learning the shortcuts.

---

## 8. DJANGO-SPECIFIC lifesavers

```
   Command                                   What it does
   ───────────────────────────────────────   ─────────────────────────────────────
   manage.py check                           Static sanity check of the project
   manage.py makemigrations --dry-run        Preview migrations (don't create them)
   manage.py sqlmigrate app 0012             Print the ACTUAL SQL a migration runs
   manage.py shell  /  shell_plus            Interactive playground (code loaded)
   manage.py dbshell                         Drop into your database's own CLI
   manage.py showmigrations                  Which migrations are applied?
   manage.py runserver                       Local dev server
```

### Email backends for testing WITHOUT sending
```
   console  ─▶  django.core.mail.backends.console.EmailBackend     (prints to terminal)
   file     ─▶  django.core.mail.backends.filebased.EmailBackend   (writes .eml files)
   real     ─▶  your SMTP/SES backend                              (actually sends — careful!)
```

### Custom management commands
Wrap any repeatable dev/admin task as `python manage.py your_command` instead of
leaving throwaway scripts lying around. Structure:
```
   your_app/
     management/
       commands/
         your_command.py     ← class Command(BaseCommand): def handle(self, ...)
```

> 📦 **Practice:** Run `manage.py sqlmigrate <app> <number>` on a real migration
> and read the SQL. Ask: would this lock a large table in production?

---

## 9. GIT as a safety net

```
   ┌──────────────┐   git add    ┌──────────────┐   git commit   ┌──────────────┐
   │  WORKING dir │ ───────────▶ │   STAGING    │ ─────────────▶ │   HISTORY    │
   │ (your edits) │              │ (next commit)│                │  (snapshots) │
   └──────────────┘              └──────────────┘                └──────────────┘
         │                                                              │
     git diff  ◀── read before staging                       git log --oneline
     git stash ◀── shelve to do later                        git bisect (find the bad commit)
```

```
   Command                  Use it for
   ──────────────────       ────────────────────────────────────────────
   git status               What's changed (run it constantly)
   git diff                 READ your own changes before committing ★
   git stash / stash pop    Shelve changes to try something else
   git log --oneline -10    Recent history
   git bisect               "Worked last week" → binary-search the bad commit
```

> 🧠 ★ `git diff` before every commit catches debug lines, typos (like a
> mistyped colour value), and accidental edits. Read it like a reviewer would.

> 📦 **Practice:** Make a small change, run `git diff`, read every line out loud,
> then commit with a one-line message explaining the *reason*.

---

## 10. REPRODUCIBILITY & ENVIRONMENT

```
   ┌─────────────────────────────────────────────────────────────┐
   │  "Works on my machine" → make EVERY machine the same machine │
   └─────────────────────────────────────────────────────────────┘
     virtualenv ─────▶ isolated deps (python -m venv .venv)
     requirements ───▶ pinned versions (pip freeze > requirements.txt)
     .env / env vars ▶ secrets & config OUT of the code, never hardcoded
     DJANGO_SETTINGS_MODULE ▶ tells Django which settings to load
```

- Always work inside a **virtual environment**. Never install project deps globally.
- Keep secrets in env vars / `.env`, **never** committed.
- 🪟 PowerShell: `$env:DJANGO_SETTINGS_MODULE='project.settings'` (not `export`).

---

## 11. HABITS THAT COMPOUND

```
   ① One change in flight at a time
   ② Cheapest verification first; fail fast where it's cheap
   ③ Prove every change with output or a test
   ④ Read tracebacks bottom-up; read your own diffs before committing
   ⑤ Mirror existing patterns; keep changes backward-compatible
   ⑥ Turn any repeated check into a script / command / test
   ⑦ When genuinely unsure between two paths, ASK instead of guessing
```

---

## 12. A 2-week practice plan

```
  Week 1
  ┌─────────┬───────────────────────────────────────────────────────────┐
  │ Day 1-2 │ Live in `manage.py shell`. Query models, call your funcs.  │
  │ Day 3-4 │ Replace one "click to check" with a python -c / shell check│
  │ Day 5-6 │ Learn breakpoint(). Debug one real issue with it.          │
  │ Day 7   │ Rest / review notes.                                       │
  └─────────┴───────────────────────────────────────────────────────────┘
  Week 2
  ┌──────────┬──────────────────────────────────────────────────────────┐
  │ Day 8-9  │ Write your first pytest test for existing behaviour.       │
  │ Day 10-11│ Build a tiny feature using the Section 4 workflow.         │
  │ Day 12-13│ Get fluent with git diff before every commit; try stash.   │
  │ Day 14   │ Wrap a repeated check as a custom manage.py command.       │
  └──────────┴──────────────────────────────────────────────────────────┘
```

---

## 13. Cheat sheet (printable)

```
 UNDERSTAND ───────────────────────────────────────────────────────────
   grep -rn "name" .          rg "name"          git grep "name"
   read the function; trace inputs→outputs; verify field names in models

 RUN / EXPLORE ────────────────────────────────────────────────────────
   python manage.py shell        # interactive, Django loaded
   python -c "snippet"           # one-off check
   render_to_string(tpl, ctx)    # test a template

 VERIFY WITHOUT SIDE EFFECTS ──────────────────────────────────────────
   with override_settings(EMAIL_BACKEND="...console..."): send(...)
   pytest -k name -v             python manage.py check

 DEBUG ────────────────────────────────────────────────────────────────
   read traceback BOTTOM-UP      breakpoint()  → n / s / c / p var / q

 DJANGO ───────────────────────────────────────────────────────────────
   makemigrations --dry-run      sqlmigrate app 0012     showmigrations

 GIT ──────────────────────────────────────────────────────────────────
   git status    git diff (read it!)    git stash    git log --oneline

 THE LOOP ──────────────────────────────────────────────────────────────
   UNDERSTAND → CHANGE small → VERIFY → repeat.   Prove it, don't hope it.
```

> Keep this file open while you work. The goal isn't to memorize commands — it's
> to internalize the loop: **Understand → Change small → Verify → repeat.**
>
> See also: `BACKEND_INFRA_GUIDE.md` (technical depth) and
> `BECOMING_A_10X_DEV.md` (mindset & leverage).
