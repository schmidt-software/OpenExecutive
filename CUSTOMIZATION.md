# Customizing Open Executive for Your Company — Worked Example: "Nohwere Inc."

This tutorial walks a system administrator with basic IT and AI knowledge through
adapting Open Executive to a specific company. It uses a fictional company,
**Nohwere Inc.**, with a five-level management hierarchy:

```
Chef (CEO)
  └── Bereichsleiter        (one per division / "Bereich")
        └── Abteilungsleiter (department head within a division)
              └── Gruppe    (a group led by a lead)
                    └── Teamleiter (team leader)
```

No code changes are required for anything in this guide — everything is done
through the web UI (primary path) or a handful of `curl` calls (faster for bulk
data). Read **Section 0** first: it explains exactly how Open Executive's data
model maps onto a 5-level hierarchy, because the mapping is not 1:1.

---

## 0. Understand the mapping before you start

Open Executive is intentionally **not** an org-chart app. It has two data
objects for structure, and both are flat:

| Nohwere Inc. role | Open Executive object | Notes |
|---|---|---|
| Chef (CEO) | **Company Profile** (`company/profile.yaml`) + one **Person** with `is_principal: true` | The Chef is "the principal" — the fallback approver for everything, and the person the Executive ultimately answers to. |
| Bereichsleiter (division head) | One **Department** per Bereich, with `head_person_id` pointing to that Bereichsleiter's **Person** record | Departments are a **flat list** — there is no parent/child nesting. A "Bereich" *is* a Department in this system. |
| Abteilungsleiter (department head under a Bereich) | A **Person** record with `department_slugs: [<bereich-slug>]` and `reports_to_person_id: <bereichsleiter's id>` | Represented as data on the People roster, not as a separate object. |
| Gruppe / Teamleiter | Further **Person** records, same pattern, `reports_to_person_id` chained up to the Abteilungsleiter | The manager chain is what encodes "Gruppe" and "Team" as levels — there's no separate "Gruppe" object. |

**Two things this system deliberately does NOT do**, so you don't design around
a capability that isn't there:

1. **The user never talks to "the Bereichsleiter" or "the Abteilungsleiter" as a
   separate chat persona.** Every conversation goes through one coherent
   "Executive" voice (see `CLAUDE.md` → *Architecture*). Bereichsleiter,
   Abteilungsleiter, Gruppen, and Teamleiter exist so the system knows **who
   owns what, who approves what, and who to notify** — not so each of them gets
   their own chatbot. Only the Executive itself (i.e., effectively "the Chef's
   AI") can be given a distinct voice/persona (Section 6).
2. A Department's `head_persona_slug` field exists in the schema but is
   currently a reserved placeholder (no prompt/voice is actually generated from
   it yet). Don't spend time trying to make a Bereichsleiter "speak"
   differently — assign `head_person_id` for routing/notifications and stop
   there.

With that understood, here is the concrete plan for Nohwere Inc.:

- **8 Bereiche**, one per the system's 8 built-in specialist domains (you can
  rename the titles to whatever Nohwere actually calls them — the underlying
  `slug` and `specialist_key` stay fixed to route to the right specialist):

  | Slug | Specialist | Suggested Nohwere Bereich title |
  |---|---|---|
  | `strategy` | CSO | Bereich Strategie & Wettbewerb |
  | `finance` | CFO | Bereich Finanzen |
  | `hr` | CHRO | Bereich Personal |
  | `legal` | GC | Bereich Recht & Compliance |
  | `operations` | COO | Bereich Betrieb & Produktion |
  | `marketing` | CMO | Bereich Marketing & Vertrieb |
  | `product` | CPO | Bereich Produkt & Entwicklung |
  | `board_comms` | Board Comms | Bereich Board & Investor Relations |

  If Nohwere has fewer or more real-world divisions than 8, that's fine — a
  Department can also be **informational only** (`specialist_key: null`, e.g. a
  "Facility Services" Bereich with no dedicated AI specialist); see Section 4.

- This guide fully details two Bereiche end-to-end — **Bereich Finanzen** and
  **Bereich Betrieb & Produktion** — down to Teamleiter level, with a template
  to repeat for the other six.

---

## 1. Prerequisites

1. Clone the repo and confirm you can start it locally:
   ```bash
   git clone https://github.com/SenteLabsAI/OpenExecutive.git
   cd OpenExecutive
   cp .env.example .env
   ```
2. Edit `.env` and set your Anthropic API key:
   ```
   ANTHROPIC_API_KEY=sk-ant-...
   ```
3. Start the app:
   ```bash
   make dev
   ```
   The API runs on `http://localhost:8000`, the UI on `http://localhost:3000`.
   First run takes a few minutes (downloads an embedding model); later runs are
   fast.
4. Open `http://localhost:3000` in a browser. You should see the onboarding
   flow (if no `company/profile.yaml` exists yet) or the main app.

> **Note on `curl` steps below:** if you set `BACKEND_SHARED_SECRET` in `.env`,
> every API request needs an `x-api-key: <that value>` header. If you left it
> blank (the default for local dev), omit the header.

---

## 2. Set up the Company Profile (the Chef's company)

### Option A — Guided wizard (recommended for a sysadmin)

Open `http://localhost:3000/onboard`. This is a 12-question, conversational
wizard. Answer as follows for Nohwere Inc. (adjust the numbers to the real
company, obviously):

| Step | Question | Example answer for Nohwere Inc. |
|---|---|---|
| 0 | Company name | `Nohwere Inc.` |
| 1 | Industry & stage | `Industrial logistics & contract manufacturing, privately held, 15 years in business` |
| 2 | Team size & founding | `340 employees, founded 2011` |
| 3 | Business model | `We manufacture and ship custom industrial components for B2B clients across Europe; annual revenue is approximately €48M` |
| 4 | Competitors & advantage | `Regional competitors X and Y; our advantage is same-week turnaround and in-house tooling` |
| 5 | Strategic priorities & north star | `Expand into the Nordics, cut order lead time to 5 days, launch a second production line; north star is on-time delivery rate` |
| 6 | Culture & values *(optional)* | `Reliability first, safety over speed, direct communication` |
| 7 | Financials *(optional)* | `Monthly burn is not applicable (profitable); skip or describe cash position` |
| 8 | Mission & vision *(optional)* | `Mission: be the manufacturing partner clients never have to double-check. Vision: the most trusted mid-size contract manufacturer in Central Europe.` |
| 9 | **Your identity** | `<Chef's real name>, CEO` — this is what makes the Chef the **principal** |
| 10 | Team members *(optional)* | See Section 5 — you can skip this here and add the full roster afterward for more control |
| 11 | Fractional executives *(optional)* | Skip unless Nohwere genuinely uses fractional/part-time executives |

Steps 9–11 write directly into the **People** roster, not the profile. For a
5-level hierarchy you'll get more precise control doing People setup
separately (Section 5), so it's fine to answer step 9 (it creates the Chef as
principal) and skip 10–11.

Submitting the wizard writes `packages/core/company/profile.yaml` and creates
the Chef's `Person` row.

### Option B — Edit the profile directly afterward

Once a profile exists, `http://localhost:3000/company-profile` lets you view
and edit every field (industry, business model, competitive landscape,
strategic priorities, culture, financials) without redoing the wizard. Use
this to fine-tune wording after the initial wizard pass.

### Option C — Hand-edit `profile.yaml` (advanced)

The file lives at `packages/core/company/profile.yaml` (gitignored — never
commit it) and follows this schema (`org_structure` here is informational
context text only, **not** the mechanism that creates Departments/People —
those are separate objects, see Sections 4–5):

```yaml
company:
  name: Nohwere Inc.
  industry: Industrial logistics & contract manufacturing
  stage: Privately held, profitable
  founding_year: 2011
  headcount: 340
  annual_revenue_arr: 48000000
  mission: Be the manufacturing partner clients never have to double-check.
  vision: The most trusted mid-size contract manufacturer in Central Europe.
  target_customer:
    profile: B2B industrial clients needing custom components, Central/Northern Europe
    pain_points:
      - Long lead times from larger competitors
      - Inconsistent quality from low-cost overseas suppliers
  competitive_landscape:
    primary_competitors:
      - Competitor X — larger, slower turnaround
      - Competitor Y — cheaper, weaker QA
    competitive_advantages:
      - Same-week turnaround
      - In-house tooling and QA
  org_structure:
    departments:
      - strategy
      - finance
      - hr
      - legal
      - operations
      - marketing
      - product
      - board_comms
    leadership_team:
      - "<Chef name>, CEO — principal, final call on all cross-Bereich decisions"
  strategic_priorities:
    current_year:
      - Expand into the Nordics
      - Cut order lead time to 5 days
      - Launch a second production line
    north_star_metric: On-time delivery rate
  culture:
    values:
      - Reliability first
      - Safety over speed
      - Direct communication
    operating_principles:
      - Every commitment to a client is checked against current production capacity
  financials:
    burn_rate_monthly: 0
    runway_months: null
    key_metrics:
      on_time_delivery_pct: 91
      order_lead_time_days: 8
```

Restart the API (`make dev`, or just the backend) after a manual edit so it
re-reads the file.

---

## 3. Give the Chef final authority (the "principal")

If you used the wizard's step 9, the Chef already exists as a `Person` with
`is_principal: true` and an implicit `wildcard` authority scope (the principal
approves everything by default — that's what makes them the ultimate
escalation point). Confirm this on `http://localhost:3000/people`: the Chef
should appear with a "Principal" badge.

If you skipped it, create the Chef manually now (Section 5 shows the People
UI in detail) with:
- Full name: the Chef's real name
- Role: `CEO`
- **Is principal: yes**
- Preferred channel + contact info (email, Slack, Discord, or Telegram —
  whichever Nohwere actually uses; see Section 8)

There is only ever **one** principal. Do not mark any Bereichsleiter as
principal.

---

## 4. Create the Bereiche (Departments)

Go to `http://localhost:3000/departments`.

For **each** of the 8 Bereiche listed in Section 0:

1. Click **New Department** (or whatever the create action is labeled).
2. Enter:
   - **Title**: e.g. `Bereich Finanzen` (the human-readable name shown
     everywhere in the UI — customize freely per Bereich)
   - **Mission**: one sentence, e.g. *"Own Nohwere's cash position, production
     cost tracking, and client payment terms."*
3. Save. This creates the department with a system-assigned `slug`
   (derived from the specialist it's wired to — e.g. `finance`) and a default
   `authority_level` of `propose_only`.
4. Open the newly created department and fill in the rest:
   - **Charter** — `scope` (bullet list of what this Bereich owns) and
     `out_of_scope` (what it explicitly does NOT own, to avoid overlap with
     neighboring Bereiche)
   - **Authority level** — one of:
     - `auto_execute` — the Bereich's routine proactive actions (cadence
       check-ins, scheduled follow-ups) run without asking anyone first
     - `propose_only` — actions are proposed to a matching approver
       (Section 5's `authority_scope`) before they run
     - `escalate` — actions run **and** the Chef is notified same-day
       (use this for legally/financially sensitive Bereiche, e.g. Recht &
       Compliance)
   - **Cadence** — e.g. a weekly check-in: `weekly@mon@09:00`
   - **Head person** — leave this empty for now; you'll set it once the
     Bereichsleiter's Person record exists (Section 5), then come back here

### Example: Bereich Finanzen, fully configured

```
Title:          Bereich Finanzen
Charter mission: Own Nohwere's cash position, production cost tracking, and
                 client payment terms.
Scope:
  - Monthly close and cash-flow forecasting
  - Per-order cost tracking (materials, labor, tooling amortization)
  - Client payment terms and overdue-invoice follow-up
  - Vendor contracts over €10,000
Out of scope:
  - Pricing strategy for new client segments (Bereich Strategie)
  - Payroll mechanics (Bereich Personal)
Authority level: propose_only
Cadence:         weekly@mon@09:00
```

### Example: Bereich Betrieb & Produktion, fully configured

```
Title:          Bereich Betrieb & Produktion
Charter mission: Run production reliably at the committed lead time and
                 quality bar, and flag anything that puts a client
                 commitment at risk.
Scope:
  - Production scheduling and capacity planning across both lines
  - Supplier and raw-material logistics
  - Quality control and non-conformance tracking
Out of scope:
  - Client pricing and contract terms
  - Marketing and demand generation
Authority level: auto_execute
Cadence:         daily@08:00
```

Repeat for the remaining six Bereiche using the table in Section 0 as a
starting point for titles, and Nohwere's actual responsibilities for the
mission/scope text.

### Advanced/bulk: create a Department via API

```bash
curl -X POST http://localhost:8000/departments \
  -H "Content-Type: application/json" \
  -d '{"title": "Bereich Finanzen", "mission": "Own Nohwere'\''s cash position, production cost tracking, and client payment terms."}'
```

Then `PATCH` the charter, authority level, and cadence:

```bash
curl -X PATCH http://localhost:8000/departments/finance \
  -H "Content-Type: application/json" \
  -d '{
    "authority_level": "propose_only",
    "cadences": {"check_in": "weekly@mon@09:00"},
    "charter": {
      "mission": "Own Nohwere'\''s cash position, production cost tracking, and client payment terms.",
      "scope": ["Monthly close and cash-flow forecasting", "Per-order cost tracking", "Client payment terms", "Vendor contracts over €10,000"],
      "out_of_scope": ["Pricing strategy for new client segments", "Payroll mechanics"]
    }
  }'
```

(If a department for a given specialist doesn't already exist as a built-in
default, check `GET /departments` first — the 8 defaults are usually seeded
automatically on first boot, so you'll mostly be `PATCH`ing existing rows
rather than `POST`ing new ones. `POST` is for genuinely new, informational-only
Bereiche with no specialist agent, e.g. a "Facility Services" division.)

---

## 5. Build the org roster: Bereichsleiter → Abteilungsleiter → Gruppen → Teamleiter

Go to `http://localhost:3000/people`.

For every person, you will fill in:

- **Full name**, **Role** (free text — this is where "Bereichsleiter", "Abteilungsleiter", "Teamleiter" actually shows up, since there's no dedicated hierarchy-level field)
- **Department(s)** — which Bereich slug(s) they belong to (a person can belong to more than one)
- **Reports to** — the manager's Person record (this is what encodes the hierarchy level)
- **Preferred channel** + contact handle (`email`, `slack_user_id`, `telegram_chat_id`, or `discord_user_id` — whichever integration Nohwere has configured, see Section 8)
- **Authority scope** — which approval types this person can sign off on. Available tokens: `spend_lt_2k`, `spend_lt_10k`, `spend_gt_10k`, `hiring_signoff`, `vendor_onboarding`, `customer_credit`, `legal_sign`, `board_comms`, `meeting_scheduling`, `wildcard` (wildcard = principal-only, don't assign it to anyone else)
- **Response SLA (hours)** — how long before an unanswered proposal escalates further up the chain

### Step-by-step: Bereich Finanzen roster

1. **Bereichsleiterin** — create first (no manager above her except the Chef):
   - Full name: `Jamie Chen`
   - Role: `Bereichsleiterin Finanzen`
   - Department(s): `finance`
   - Reports to: *(leave empty, or set to the Chef if you want escalations to name them explicitly)*
   - Authority scope: `spend_lt_10k`, `vendor_onboarding`
   - Preferred channel: `email`
   - Save, then **note her Person ID** (shown in the UI / API response).

2. Go back to **Bereich Finanzen** in `/departments` and set **Head person** =
   Jamie Chen. This is what makes her show up as the Bereich's owner
   everywhere (Today page, escalations, notifications).

3. **Abteilungsleiter** under her:
   - Full name: `Sam Patel`
   - Role: `Abteilungsleiter Controlling`
   - Department(s): `finance`
   - Reports to: `Jamie Chen`
   - Authority scope: `spend_lt_2k`
   - Preferred channel: `email`

4. **Gruppe / Teamleiter** under Sam:
   - Full name: `Robin Fischer`
   - Role: `Teamleiter Kreditorenbuchhaltung` (Accounts Payable)
   - Department(s): `finance`
   - Reports to: `Sam Patel`
   - Authority scope: *(usually none — routine work, escalates to Sam)*
   - Preferred channel: `email`

That's the full chain: **Chef → Jamie Chen (Bereichsleiterin) → Sam Patel
(Abteilungsleiter) → Robin Fischer (Teamleiter)**. Add as many Gruppen/Teamleiter
as Nohwere actually has by repeating step 4 with different `reports_to` values,
and as many Abteilungsleiter as Nohwere has by repeating step 3.

### Step-by-step: Bereich Betrieb & Produktion roster (second worked example)

- Bereichsleiter: `Chris Donovan` — Role `Bereichsleiter Betrieb & Produktion`,
  department `operations`, authority scope `spend_lt_2k`
- Abteilungsleiter: `Morgan Alves` — Role `Abteilungsleiterin Produktionslinie 1`,
  reports to Chris Donovan, department `operations`
- Teamleiter: `Kai Weber` — Role `Teamleiter Frühschicht`, reports to
  Morgan Alves, department `operations`
- Teamleiter: `Nadia Kowalski` — Role `Teamleiterin Spätschicht`, reports to
  Morgan Alves, department `operations`

Set `Chris Donovan` as **Head person** on the Bereich Betrieb & Produktion
department, same as step 2 above.

Repeat this whole pattern for the remaining six Bereiche.

### Advanced/bulk: create People via API

Useful once you have Person IDs and want to script a large roster instead of
clicking through the UI for every Teamleiter:

```bash
# 1. Create the Bereichsleiterin, capture her returned "id"
curl -X POST http://localhost:8000/people \
  -H "Content-Type: application/json" \
  -d '{
    "full_name": "Jamie Chen",
    "role": "Bereichsleiterin Finanzen",
    "department_slugs": ["finance"],
    "email": "jamie.chen@nohwere.example",
    "preferred_channel": "email",
    "authority_scope": ["spend_lt_10k", "vendor_onboarding"]
  }'
# → { "id": 7, ... }

# 2. Point the department at her
curl -X PATCH http://localhost:8000/departments/finance \
  -H "Content-Type: application/json" \
  -d '{"head_person_id": 7}'

# 3. Create her Abteilungsleiter, reporting to id 7
curl -X POST http://localhost:8000/people \
  -H "Content-Type: application/json" \
  -d '{
    "full_name": "Sam Patel",
    "role": "Abteilungsleiter Controlling",
    "department_slugs": ["finance"],
    "reports_to_person_id": 7,
    "email": "sam.patel@nohwere.example",
    "preferred_channel": "email",
    "authority_scope": ["spend_lt_2k"]
  }'
# → { "id": 12, ... }

# 4. Create a Teamleiter reporting to id 12
curl -X POST http://localhost:8000/people \
  -H "Content-Type: application/json" \
  -d '{
    "full_name": "Robin Fischer",
    "role": "Teamleiter Kreditorenbuchhaltung",
    "department_slugs": ["finance"],
    "reports_to_person_id": 12,
    "email": "robin.fischer@nohwere.example",
    "preferred_channel": "email"
  }'
```

Script this loop (name/role/manager-id triples in a spreadsheet →
`curl` calls) for a roster with dozens of Teamleiter — it is far faster than
the UI for bulk entry.

---

## 6. Bulk-load the whole org in one shot (fixture approach)

If you'd rather author the entire Nohwere Inc. setup as files and load it in
one command (recommended once you're comfortable with the format — it's also
the easiest way to reset back to a known-good state during testing), build a
fixture bundle following the existing examples in `fixtures/companies/`
(`halcyon_motors` is a good reference to copy from):

1. Create `fixtures/companies/nohwere_inc/`.
2. Add `profile.yaml` — same schema as Section 2, Option C.
3. Add `departments.yaml`:
   ```yaml
   departments:
     - slug: finance
       title: Bereich Finanzen
       specialist_key: cfo
       head_person_name: Jamie Chen
       authority_level: propose_only
       charter:
         mission: Own Nohwere's cash position, production cost tracking, and client payment terms.
         scope:
           - Monthly close and cash-flow forecasting
           - Per-order cost tracking
           - Client payment terms
         out_of_scope:
           - Pricing strategy for new client segments
       headcount: 14
       budget_usd: 1200000
       okrs:
         - quarter: Q3 2026
           key_result: Cut days-sales-outstanding from 48 to 35
           target: "<= 35 days"
           current: "48 days"
           status: at_risk
     # ... repeat for operations, hr, legal, marketing, product, strategy, board_comms
   ```
4. Add `people.yaml` — one entry per person, `department_slugs` +
   `authority_scope` as shown in Section 5 (note: `reports_to_person_id` isn't
   resolved by name in fixture YAML the way `head_person_name` is — set up
   manager chains via the UI/API after loading, or check the loader code if
   you need this scripted for a very large roster).
5. Add `docs/nohwere-org-chart.md`, `docs/nohwere-approval-policy.md`, etc. —
   see Section 7.
6. Add `memory.json` (can start empty):
   ```json
   {"decisions": [], "initiatives": [], "advice_given": [], "scheduled_actions": []}
   ```

Load it:

```bash
curl -X POST http://localhost:8000/fixtures/nohwere_inc/load
```

or via the UI at `http://localhost:3000/demo`. **This replaces the currently
active company profile, departments, people, and memory** — use it on a fresh
local instance or when you intend to reset.

---

## 7. Add Nohwere-specific knowledge (org chart, policies)

Two different knowledge locations exist — do not mix them up:

- `knowledge/` (git-tracked, repo-wide): the shared, generic MBA/business
  knowledge base shipped with Open Executive. **Never put Nohwere-specific
  content here** — it's shared across every deployment of this codebase.
- `packages/core/company/docs/` (gitignored, per-company): exactly where
  Nohwere-specific material belongs.

Upload via `http://localhost:3000/knowledge` (drag-and-drop) — this is the
sysadmin-friendly path and handles chunking/indexing into ChromaDB
automatically. Good candidates for Nohwere:

- `nohwere-org-chart.md` — a plain-language written description of the
  Chef → Bereichsleiter → Abteilungsleiter → Gruppen → Teamleiter structure
  (the Executive can reference this in conversation even though the People/
  Department objects don't store the full chart as a diagram)
- `nohwere-approval-policy.md` — spend limits, who signs what, escalation
  rules (reinforces what you configured as `authority_scope`/`authority_level`)
- `nohwere-production-sops.md`, client contracts, vendor agreements, etc.

---

## 8. Wire up communication channels

Each Person's `preferred_channel` only works once the corresponding
integration is actually configured in `.env`. Set what Nohwere uses:

```bash
# Slack
SLACK_BOT_TOKEN=xoxb-...
SLACK_APP_TOKEN=xapp-...

# Discord
DISCORD_BOT_TOKEN=...
DISCORD_APP_ID=...
DISCORD_GUILD_IDS=...

# Telegram
TELEGRAM_BOT_TOKEN=...
TELEGRAM_WEBHOOK_SECRET=...

# Email (used both for sending and for the inbound poller)
EXEC_EMAIL_ADDRESS=exec@nohwere.example
```

Restart the API after changing `.env`. See `docs/deployment.md` and
`docs/google_chat_setup.md` (for Google Chat) if Nohwere needs those
specifically — this is beyond a basic customization pass and closer to a
deployment task.

---

## 9. Give the Chef's Executive a distinct voice (optional)

The single "Executive" persona everyone talks to can be given Nohwere's own
tone — e.g., more formal, more terse, more safety-conscious — separately
from any of the 8 specialist prompts.

1. Create a new persona file (or use the `/settings` UI if it exposes persona
   editing; otherwise via API):

   ```bash
   curl -X POST http://localhost:8000/personas \
     -H "Content-Type: application/json" \
     -d '{
       "display_name": "Nohwere Executive Voice",
       "body": "Adopt the voice, tone, register, and communication style described below.\n\n- Formal and precise. Address the reader respectfully; avoid casual phrasing.\n- Safety-first. Any recommendation touching production or client commitments explicitly calls out the safety/quality trade-off.\n- Numbers-driven. Always tie a recommendation to lead time, cost, or on-time-delivery impact."
     }'
   ```
   This returns a `slug` (a slugified version of the display name).

2. Activate it for the Executive:
   ```bash
   curl -X PATCH http://localhost:8000/agents/executive \
     -H "Content-Type: application/json" \
     -d '{"voice_persona_slug": "nohwere-executive-voice"}'
   ```

3. Reload the chat UI and confirm the tone changed. To revert, `PATCH` again
   with `"voice_persona_slug": null`, or use the persona's reset endpoint.

Remember (from Section 0): this changes **one** voice — the Executive the
Chef and everyone else talks to. It does not create separate voices per
Bereichsleiter/Abteilungsleiter.

---

## 10. Verify the setup

1. **Restart the API** (`make dev`, or restart the backend process) and check
   the boot log for org-completeness warnings, e.g.:
   ```
   Department 'Bereich Marketing & Vertrieb' has no head person configured.
   No non-principal approver for scope 'legal_sign'.
   ```
   Fix any that appear — either set a missing `head_person_id`, or assign the
   relevant `authority_scope` to at least one non-principal person.
2. Open `http://localhost:3000/departments` — all 8 (or however many) Bereiche
   should show their head person, authority-level pill, and cadence.
3. Open `http://localhost:3000/people` — spot-check that a few Teamleiter show
   the correct "Reports to" chain up to their Bereichsleiter.
4. Open the chat UI and ask something cross-Bereich, e.g. *"We might miss a
   client delivery date next week — what should we do?"* — confirm the
   response reflects Nohwere's actual mission/values/priorities from Section 2,
   and that Operations-specific scope/out-of-scope boundaries are respected.
5. If you configured `escalate`-level Bereiche (e.g. Recht & Compliance),
   trigger a scenario that should notify the Chef and confirm the notification
   arrives on the Chef's configured channel.

---

## Quick reference: Nohwere role → Open Executive object

| Nohwere concept | Object | Key field(s) |
|---|---|---|
| Chef (CEO) | `Person` | `is_principal: true`; implicit `wildcard` authority |
| Chef's AI voice | `Persona` on the `executive` agent override | `voice_persona_slug` |
| Bereich | `Department` | `slug`, `title`, `specialist_key`, `head_person_id`, `authority_level`, `charter`, `cadences` |
| Bereichsleiter | `Person` | `department_slugs: [<bereich slug>]`, referenced by the Bereich's `head_person_id` |
| Abteilungsleiter | `Person` | `department_slugs`, `reports_to_person_id: <bereichsleiter id>` |
| Gruppe / Teamleiter | `Person` | `department_slugs`, `reports_to_person_id: <manager id>` |
| Company-wide facts | `company/profile.yaml` (`CompanyProfile`) | `name`, `industry`, `mission`, `org_structure`, `strategic_priorities`, ... |
| Nohwere-specific documents | `packages/core/company/docs/` | uploaded via `/knowledge` UI |
| Approval limits | `Person.authority_scope` + `Department.authority_level` | e.g. `spend_lt_10k`, `propose_only` |
