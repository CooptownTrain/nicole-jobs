# Nicole Buchan Curated-Jobs Report — Engineering Handoff

**Owner of record:** Steve Cooper (FMJ Chief Architect)
**Engineering author:** Claude Opus 4.7 (1M context) acting as Chief Engineer
**Status:** Live, public mirror updated 2026-04-24 11:20 AM ET; this doc drafted 2026-04-27 for handoff to the New FMJ team
**Audience:** New FMJ engineering, product, QA — anyone building the AI-resume + AI-jobs pipeline

This document captures everything we built, every input we used, every input we wished we had, and the upgrade plan for merging this work into New FMJ.

---

## 1. Executive Summary

We shipped a single-file curated job-match report for Nicole Buchan (FAU Multimedia Journalism senior, Women's Lacrosse captain, May 2026 grad). 33 Florida + remote roles, two priority tiers, every card grounded in BLS / NACE / Jobscan / employer-published sources, every claim traceable to a working URL.

It is hosted as a public GitHub Pages site (`cooptowntrain.github.io/nicole-jobs/`) that mirrors a local source HTML. There is **no backend, no build step, no framework, no API**. Everything is hand-authored markup driven by the Claude agent inside Claude Code, with three days of iteration captured in the conversation transcript and four governance docs in `docs/product/`.

The next phase replaces the manual authoring with a Python + Claude pipeline that pulls live job postings from APIs, scores fit against a parsed resume, and writes the same card structure programmatically. That pipeline is what merges into New FMJ.

---

## 2. What Shipped

| Artifact | Location | Purpose |
|---|---|---|
| Source HTML | `/private/tmp/nicole_jobs_full.html` (8,360 lines) | Single source of truth; all edits land here first |
| Public mirror | `https://cooptowntrain.github.io/nicole-jobs/` | The link Nicole received |
| GitHub repo | `github.com/CooptownTrain/nicole-jobs` | Pages-backing repo |
| Local working clone | `/tmp/gh-work/nicole-jobs/` | `gh repo clone` workspace; `cp` source → `index.html` on every push |
| Card content standards | `docs/product/CARD_CONTENT_STANDARDS.md` | Binds AI generator + reviewers; §2.05 word-count targets |
| Card production guide | `docs/product/CARD_CONTENT_PRODUCTION_GUIDE.md` | 5-stage pipeline; subagent briefing templates |
| AI behavior governance | `docs/governance/AI_BEHAVIOR.md` | Em-dash ban, no-fabrication law, voice rules |

**Scale of the artifact:**
- 33 role cards (14 High Priority + 19 Medium Priority; Low Priority tier removed)
- 1,687 `<div>` opens / 1,687 closes (validated after every edit)
- ~250 outbound citation URLs across BLS, NACE, Jobscan, employer career portals, GitHub, LinkedIn search seed URLs
- Six per-card sections (header, meta-row, fit, gaps, next-steps tabs, sources, actions) plus three modal-only sections (jd, company, news) and a full-bleed sticky filter bar

---

## 3. Architecture & Code Scope

### 3.1 File anatomy (single-file HTML, no framework)

```
/private/tmp/nicole_jobs_full.html              (8,360 lines)
├── lines 1–6        <head> + <meta viewport>
├── lines 7–1166     <style>  — all CSS, ~1,160 lines
├── lines 1168–7780  <body>
│   ├── 1170–1207     <div class="sticky-nav">      sticky header + filter bar
│   ├── 1208+         <div class="container">       carousel-mode card stack
│   ├── 6595–7780     <div id="card-modal-store">   hidden modal-only content
│   ├── 7781–7791     <div class="modal-overlay">   modal scaffold
├── lines 7793–8357  <script>                       all JS, ~560 lines
└── line 8359        </body></html>
```

There is no bundler, no transpilation, no NPM. The file works on any static host.

### 3.2 CSS architecture

Naming is BEM-adjacent but not strict. Major class families:

| Family | Purpose |
|---|---|
| `.sticky-nav`, `.filter-bar`, `.filter-dropdown(-menu/-trigger)`, `.filter-chip`, `.dd-caret` | Top-level sticky filter UI |
| `.container`, `.card`, `.card-header`, `.fit-badge`, `.fav-btn`, `.meta-row`, `.meta-item` | Card outer chrome |
| `.section`, `.section-fit`, `.section-gaps`, `.section-next`, `.section-sources`, `.section-jd`, `.section-company`, `.section-news`, `.section-body` | Per-card accordion sections |
| `.fit-summary`, `.bullet-list`, `.cite` | Fit section internals |
| `.gap-card`, `.gap-card-head`, `.gap-number`, `.gap-title`, `.gap-section`, `.gap-section-label`, `.gap-section-body`, `.gap-action`, `.gap-action-head`, `.gap-action-title`, `.gap-action-bullet`, `.gap-action-directive`, `.gap-copy-btn`, `.gap-ref` | Gaps section internals (4-part Stern-style structure) |
| `.tabs`, `.tab-bar`, `.tab-label`, `.tab-content`, `.tab-lead`, `.resume-lead`, `.ats-explainer`, `.ats-hint` | Next-Steps 5-tab system |
| `.sources-list`, `.src-note` | Sources footer |
| `.actions`, `.btn`, `.btn-primary`, `.btn-secondary` | Apply / research CTA row |
| `.modal-overlay`, `.modal-card`, `.modal-header`, `.modal-body`, `.modal-close` | Modal popup for jd/company/news |
| `.empty-state`, `.kbd-hint`, `.carousel-nav`, `.recover-tag`, `.favorites-tag` | Carousel chrome |

**Tier color coding** (CSS attribute selectors):
```css
.card[data-tier="high"]   .fit-badge { background: #22c55e; color: #fff; }     /* green */
.card[data-tier="medium"] .fit-badge { background: #eab308; color: #422006; }  /* amber */
.card[data-tier="low"]    .fit-badge { background: #ef4444; color: #fff; }     /* red — currently unused */
```

**Mobile responsive block** (≤640px): single `@media (max-width: 640px)` rule that compacts sticky-nav, drops the apply button stack from a right rail to an inline row, halves card padding, shrinks meta-row icons, tightens the gap-card border, and reduces sources-list font.

### 3.3 JavaScript catalog

All inline at the bottom of `<body>`. No external scripts, no module system. Functions in execution order:

| Function | Line | What it does |
|---|---|---|
| `openModal(cardId, type)` | 7833 | Opens jd/company/news modal; pulls content from `#card-modal-store [data-card]` first, falls back to in-card section |
| `closeModal()`, `closeModalBackdrop(e)` | 7860/7864 | Modal teardown + backdrop click handling |
| `copyGapText(btn)` | 7867 | Copy-to-clipboard for resume-action bullets; falls back to `execCommand('copy')` |
| `switchTab(cardId, tabName)` | 7895 | Next-Steps tab switcher (Resume / Network / Research / Position / Interview) |
| `getFavs() / saveFavs() / toggleFav() / updateFavCount() / toggleFavFilter()` | 7906–7938 | localStorage favorites under `FAV_KEY = 'nicole-job-favorites'` |
| `getRemoved() / saveRemoved() / updateRemovedCount() / toggleDeletedView() / exitDeletedView() / renderDeletedList() / showDeletedCard() / removeJob() / recoverJob() / updateCardActionButton()` | 7942–8064 | localStorage removed-jobs under `REMOVED_KEY = 'nicole-job-removed'`; lets Nicole hide cards and recover them |
| `getActiveValues(attr) / cardMatchesFilters(card) / getVisibleCardIds() / resetFilters() / toggleDropdown(name, ev) / updateTriggerLabels()` | 8066–8140 | Filter logic; cards survive filter when (a) tier matches active tier chips OR no tier active, AND (b) type matches, AND (c) location matches, AND (d) favorites toggle OK, AND (e) not in removed-list |
| `showCard(idx) / nextCard() / prevCard() / refreshVisible()` | 8144–8176 | Carousel navigation; `currentIdx` is the cursor across the filter-survivor list |

**State that persists per Nicole's browser** (no server):
- `localStorage['nicole-job-favorites']` — JSON array of card IDs she starred
- `localStorage['nicole-job-removed']` — JSON array of card IDs she dismissed

### 3.4 Per-card structure (anatomical map)

Each card is a `<div class="card" id="<slug>" data-tier="..." data-type="..." data-location="...">` containing six rendered sections + three hidden-store sections + an actions row:

```
<div class="card" id="..." data-tier="high|medium" data-type="full-time|part-time|internship" data-location="...">
  <div class="card-header">              fit-badge, favorite star, h2 title, .company line
  <div class="meta-row">                 4 emoji-prefixed factual chips (location / pay / education / experience)

  <details class="section section-fit">     summary paragraph + <ul class="bullet-list">
  <details class="section section-gaps">    one or more <div class="gap-card"> blocks, each with:
                                              gap-card-head (number + title)
                                              gap-section × 3 (What screening for / Adjacent experience / The reframe)
                                              gap-action (resume-action bullet + 📋 copy button)
  <details class="section section-next">    <div class="tabs"> with 5 tab-content panels:
                                              -resume   (ats-explainer + resume-lead + numbered ATS-hinted ol)
                                              -network  (tab-lead + numbered ol of outreach targets)
                                              -research (tab-lead + ul of authoritative reading)
                                              -position (tab-lead + positioning sentence + don't-lead-with line)
                                              -interview(tab-lead + STAR-prompt ol)
  <details class="section section-sources"> <ol class="sources-list"> with src-note per item

  <div class="actions">                  Apply CTA + research-the-company secondary
</div>
```

**Modal-only sections** live entirely outside the card, in `<div id="card-modal-store">` near the bottom of the body. Each child `<div data-card="<slug>">` holds the card's `section-jd`, `section-company`, `section-news`. JS pulls these on demand.

This separation was a deliberate fix: rendering jd/company/news inside the card made the carousel too tall and broke mobile. Moving them to the store kept the card scannable and let the modal own the deep content.

### 3.5 Hosting & publish workflow

Documented in `reference_nicole_jobs_publish.md` (memory). Every edit:

1. Update header timestamp in `/private/tmp/nicole_jobs_full.html` (`Updated <date> · <time> ET`)
2. Body `<div>` open/close balance check — must be exactly zero diff
3. `cp /private/tmp/nicole_jobs_full.html /tmp/gh-work/nicole-jobs/index.html`
4. Update `README.md` timestamp + role count in the local clone
5. `git add index.html README.md && git commit -m "..." && git push origin main`
6. GitHub Pages auto-deploys in 30–60 s

---

## 4. Card Content System

The card content rules are governed by `docs/product/CARD_CONTENT_STANDARDS.md`. The relevant sections for any team rebuilding this:

| Section | Rule |
|---|---|
| §0.1 No-Fabrication | Every factual claim grounded in injected context or a working public URL; otherwise hedge or drop |
| §0.2 Sources Footer | Every card ends with a numbered `<ol class="sources-list">` matching inline `[n]` citations |
| §1 Why You're a Good Fit | 75–110 word summary + 3–5 arrow bullets; ≥1 bullet must cite |
| §2 Gaps to Close | 4-part block per gap: What they're screening for / Your adjacent experience / The reframe / Resume Action |
| §2.05 Tightness | Word-count maxes (post-research calibration — see §6 below) |
| §2.1 Resume Bullet Rigor | 15–25 words, XYZ formula (Accomplished X measured by Y by doing Z), ATS Keywords Met footer |
| §3 Next Steps | 5 tabs × 120–200 words each |
| §3.1 ATS Explainer | Required block on every Resume tab |
| §5 Sources Footer Format | One numbered entry per source, working URL, src-note |
| §7 QA Checklist | Run before publishing |
| §8 Enforcement | AI generator must hard-refuse if standards aren't met |

**Tightness targets (§2.05):**

| Element | Max | Target |
|---|---:|---:|
| Fit summary paragraph | 60 words | 40–50 |
| Fit arrow bullet | 25 words | 15–20 |
| Gap title | 12 words | — |
| Gap What/Experience/Reframe (each) | 25–30 words | 1 sentence preferred |
| Next-Steps lead | 50 words | 30–40 |
| Next-Steps bullet | 40 words | 25–35 |

These targets came from Brysbaert (2019) reading-speed meta-analysis and Nielsen Norman "How Little Do Users Read" (2008): anxious job-seekers triage at ~20% word intake, attention decays past 110 words per block. Total visible words per card after tightening: **300–400** for the Fit + Gaps combined block.

---

## 5. Inputs Used to Analyze the Jobs

This is the most important section for the New FMJ team — the success of the Phase-2 pipeline depends on getting the **input contract** right.

### 5.1 What we knew about Nicole (treated as ground truth)

Provided directly by Steve via conversation, no resume PDF parse:

| Datum | Value | Confidence |
|---|---|---|
| Name | Nicole Buchan | Verified |
| Education | Florida Atlantic University, BA Multimedia Journalism, Sports Minor | Verified |
| Graduation | May 2026 | Verified |
| Athletics | FAU Women's Lacrosse, 4-year team, **Senior Captain** | Verified |
| Work history | Orangetheory Fitness — Sales Associate, 3+ years (concurrent w/ school) | Verified |
| Volunteer | Team Israel Lacrosse, **Ambassador**, since Sept 2025 | Verified |
| Geography | Plantation, FL (45 min from Miami) | Verified |
| Languages | English (native); no Hebrew or Spanish stated | Inferred-default |
| GPA | **Not provided** | Missing — material gap, see §5.4 |
| Portfolio | **Not provided** | Missing |
| Authorization | **Not provided** (assumed US person) | Inferred |
| Salary floor | **Not provided** | Missing |
| Relocation appetite | **Stated FL + remote only** | Verified |

### 5.2 What Steve told us directly (binding directives)

These are the user-facing "rules of engagement" that shaped the entire report. Recording them so the New FMJ team can recreate them as system prompts:

1. **Florida + remote only** — no out-of-state.
2. **Three role types**: Full Time, Part Time, Internship (Low Priority tier removed mid-build).
3. **Match the NYU Stern card depth** — no shortcuts on Fit, Gaps, or Next Steps richness.
4. **Everything must be true.** No fabrication. (This was said as a correction after early drafts speculated.) → Codified as §0.1 No-Fabrication Rule.
5. **No em dashes, no AI-tells.** → Codified in `docs/governance/AI_BEHAVIOR.md`.
6. **XYZ resume bullet rigor** — match New FMJ's `ResumeBulletSchema`.
7. **300–400 visible words per card after tightening** — anxious job-seekers can't sustain dense prose.
8. **Mobile-friendly** — header was eating 1/3 of the mobile screen.
9. **Public GitHub Pages mirror** — link Nicole already received must always reflect latest.
10. **One bundled PR per change set** — no churn from many tiny pushes.

### 5.3 What we inferred (calibrated assumptions, flagged in source notes)

| Inferred | Basis |
|---|---|
| Bilingual content production capability | Team Israel Ambassador role implies cross-cultural audience reach; coded as transferable skill, not a Spanish/Hebrew claim |
| Comfort with social-media production tools | Multimedia Journalism degree + Team Israel content output |
| Captain → calendar discipline + deadline endurance | Standard team-sport translation; cited in fit bullets |
| Customer-facing ease | 3 yr Orangetheory sales role |
| Workday / Greenhouse familiarity | Implicit in any FAU Workday-routed application — flagged in ATS explainer not on resume |
| Comfort with cold outreach | Same Orangetheory inference; appears in network-tab leads |

Each inference appears in cards as language ("captaincy demonstrates calendar discipline") rather than asserted fact ("Nicole has 4 years of project management experience"). The boundary is enforced by §0.1.

### 5.4 What was missing — the precision gap

**Without these inputs, the report ran wider than necessary.** Documenting them so New FMJ can require them upfront:

| Missing input | Effect on report | What we did instead |
|---|---|---|
| **GPA** | Could not auto-pass/fail 3.0+ minimum filters (e.g. TelevisaUnivision 3.0 hard cut) | Made GPA-add a Resume Action gap on every card with a published GPA floor |
| **Resume PDF / structured resume** | Could not score job-title keyword match per Jobscan's 10.6× rule precisely | Used proxy: degree-name match + role-name match + verbal evidence |
| **Portfolio URL** | Could not anchor "show, don't tell" advice in real assets | Recommended building a 2-piece Notion portfolio as a Gap action |
| **Salary floor** | Couldn't filter out unpaid or below-floor roles | Surfaced compensation in meta-row + flagged unpaid in card body, no auto-cull |
| **Visa / authorization** | Couldn't hard-screen sponsorship-required roles | Assumed US person; added inference flag |
| **Schedule constraints** (lacrosse season, finals) | Couldn't time application advice | Used generic "apply this week / next 2 weeks" cadence |
| **Existing applications & contacts** | Couldn't dedupe or warm-route | Treated all 33 as cold |
| **Preferred work format** (in-office vs remote vs hybrid) | Couldn't prioritize remote roles | Used `data-location` filter only |
| **Languages** beyond English | Couldn't score bilingual-preferred postings (Univision, Faena) | Hedged ("Spanish preferred but not required on this req") |
| **Specific career arc preference** (sports vs PR vs broadcast vs corp comms) | Built a generalist mix; couldn't depth-bias | Distributed across sectors |
| **Driver's license / car** | Couldn't score commute-eligibility (FL is car-dependent) | Assumed yes for all FL roles |
| **References** | Couldn't pre-warm any introductions | Suggested faculty / Career Center as default |

The Phase-2 pipeline must collect these as a structured intake form before any AI scoring runs. Data missing here = noise in every downstream card.

---

## 6. Research & Authoring Pipeline (what we actually did)

This is the manual process Phase-2 will replace. Documented so the replacement preserves what worked.

### 6.1 Five stages

```
INTAKE (Steve provides candidate facts + role list)
   ↓
RESEARCH (parallel subagents grounded in BLS / NACE / Jobscan / employer docs)
   ↓
WRITING (main agent assembles Fit / Gaps / Next Steps / Sources from research briefs)
   ↓
QA (no-fab check, em-dash strip, div-balance, tightness §2.05)
   ↓
INSERT (write to /private/tmp/nicole_jobs_full.html, cp to mirror, push)
```

### 6.2 Parallel subagent batches

We ran subagents 3 cards per agent, 3 agents in parallel per batch, grouped by sector (sports, hospitality, university, corporate). Each subagent had a strict no-fabrication briefing:

> "Find authoritative URL-backed evidence for: (a) the employing company's basic facts (size, HQ, recent strategy moves), (b) the role's BLS occupational match (median wage + growth), (c) any employer-published HR policy / fiscal-year detail / pay plan, (d) Glassdoor interview difficulty rating if present, (e) ATS platform if discoverable. Return as structured JSON with one URL per claim. If a claim cannot be sourced, return `null` and a `reason` field. Do not invent."

The main agent then composed the card from the JSON, never adding facts the subagent didn't return.

### 6.3 Citation hierarchy (from §0.1)

1. BLS OOH / OES — wage, growth, occupation
2. O*NET — skills, related roles
3. NACE — class-year hiring outlook, starting salary
4. Employer's own published documentation — fiscal calendars, HR policy, pay plans, career portals
5. Jobscan / peer-reviewed research — ATS stats
6. SEC filings / IR pages — financials, strategy

### 6.4 Tightening pass (calibrated separately)

After all 33 cards were written at full length, we ran a separate research pass on reading-comprehension capacity (Brysbaert 2019, Nielsen Norman 2008) and used the findings to set §2.05 word-count maxes. We then tightened all 14 upgraded cards (Enterprise as the exemplar + 14 in queue, completed 2026-04-24) one card at a time, preserving every citation but cutting filler ("in order to," "the fact that," "essentially").

### 6.5 QA invariants (run after every edit)

| Check | How |
|---|---|
| Body `<div>` balance | `python3` regex count of `<div\b` vs `</div>` between `<body>` and `</body>` — must be 0 diff |
| Em-dash absence | grep for ` — ` and `—` in body copy → must be 0 in body, allowed only in source titles |
| Sources match inline cites | Every `<sup><a class="cite" href="#x-src-N">[N]</a></sup>` must have an `<li id="x-src-N">` |
| All URLs reachable | Manual spot check on changed cards |
| Card count parity | `grep -c '<div class="card" id='` must equal the number in `<small>` header |
| Mobile render check | Viewport meta + 640px media query; no overflow at 375px width |

---

## 7. Anti-Patterns We Hit (don't repeat in the v2 pipeline)

| Bug | Cause | Fix | Permanent guard |
|---|---|---|---|
| 523 em dashes in body copy | Default LLM punctuation | Python global replace `" — "` → `", "` | §AI_BEHAVIOR governance + post-write linter |
| Fabricated claims ("FAU hires students first," "Specialist I = junior ladder") | Plausible-sounding inference treated as fact | Subagent research + §0.1 No-Fabrication Rule | Schema requires `source_url` per claim |
| Python deletion script chopped 3 cards | Pre-computed offsets walked into mutated HTML during sequential `rfind`s | Reconstruct deletions one at a time, verify balance after each | Don't mutate offsets across deletions; structural edits go through DOM, not text positions |
| Duplicate `<div class="tab-content" id="c6-interview">` collapsed entire card layout | Earlier copy-paste left an orphan | div-depth tracer found imbalance | Always run div-balance after structural edits |
| Two-level fit-bullet renderers looked dense | Per-bullet `<div class="fit-bullet-head"> + <div class="fit-bullet-why">` doubled visual weight | Flattened to single-line `<li><strong>Title.</strong> Why</li>` matching `.bullet-list` | One bullet rendering pattern across all cards |
| Cards rendered two-at-a-time in carousel | Broken `display: none / block` toggle on `.card.active` after a structural fix | Restored carousel CSS + corrected nesting | Snapshot test for carousel visual state |

---

## 8. Upgrade Roadmap — Job Finding & AI Evaluation

This is the merge plan with New FMJ. The manual process above becomes a six-stage pipeline. Each stage names the right tool (Python orchestrator vs. Claude AI agent vs. Anthropic API with structured output) and the contract between stages.

### 8.1 Stage A — Candidate Intake

**Tool:** New FMJ resume-parsing AI (in build) + intake form
**Owner:** New FMJ frontend
**Output contract** (`CandidateProfile` JSON):

```ts
{
  identity: { name, email, phone, geography, willing_to_relocate: bool, work_authorization },
  education: [{ school, degree, major, minor?, gpa?, expected_graduation, coursework: string[] }],
  experience: [{ employer, title, start, end, bullets: string[], xyz_normalized: ResumeBullet[] }],
  athletics_leadership: [{ org, role, years, achievements: string[] }],
  volunteer: [{ org, role, start, end, bullets: string[] }],
  skills: { tools: string[], languages: string[], certifications: string[] },
  preferences: {
    role_types: ('full-time'|'part-time'|'internship')[],
    sectors: string[],
    salary_floor?: number,
    schedule_constraints: string[],
    sponsorship_needed: bool,
    car_available: bool,
    referral_pool: { name, employer, relationship }[],
  },
  portfolio_urls: string[],
}
```

**Why this contract matters:** §5.4 lists every precision gap from the manual run. Forcing this schema at intake closes them all. Reject incomplete profiles with a structured "you're missing X" UI rather than letting the AI guess.

### 8.2 Stage B — Job Ingestion (the part Steve flagged for upgrade)

**Tool:** Python orchestrator (FastAPI service or Cloud Function)
**Sources to integrate** (in priority order — all have stable APIs):

| Source | API | Coverage | Notes |
|---|---|---|---|
| LinkedIn Jobs | Official Talent Solutions API (paid) or Phantombuster / Coresignal (third-party) | Largest US source | Watch ToS — use official where viable |
| Indeed Publisher API | Public, key-gated | Massive aggregator | Rate-limited |
| Adzuna | Public REST | Strong aggregator | Free tier OK for prototype |
| Jobs API (Greenhouse) | Per-employer endpoints | High-quality structured data | Requires per-employer integration; covers Inter Miami, IMCF tier orgs |
| Workday Jobs | Per-tenant endpoints (e.g. `fau.wd1.myworkdayjobs.com`) | University + enterprise | Already in Nicole's report |
| iCIMS / Lever / Ashby | Per-tenant | Mid-market | Reachable via headless fetch + structure parser |
| Handshake | OAuth, university-gated | Student-only roles | Requires university partnership; high-fidelity |
| GovJobs / FederalJobs (USAJobs.gov) | Public | Government | Free; non-trivial schema |
| TeamWork Online | Scraping (or affiliate) | Sports industry | High value for Nicole-class candidates |

**Output contract** (`JobPosting`):

```ts
{
  source: string,           // 'linkedin' | 'workday' | 'greenhouse' | ...
  source_id: string,         // canonical posting id
  url: string,               // apply URL
  fetched_at: ISO8601,
  employer: { name, hq_city, hq_state, size_band },
  title: string,
  jd_text: string,           // full JD for downstream AI
  jd_structured: {            // best-effort parse
    requirements: string[],
    nice_to_have: string[],
    education_min: string,
    gpa_min?: number,
    visa_sponsorship: bool|null,
    salary?: { min, max, currency, basis: 'hourly'|'annual' },
  },
  location: { city, state, remote_eligible: bool, hybrid: bool },
  type: 'full-time'|'part-time'|'internship'|'contract',
  posted_at?: ISO8601,
  ats_platform?: string,
}
```

**Dedup key:** `(employer.name, title, location.city)` over a 14-day rolling window. The same role gets posted to LinkedIn + Indeed + the employer ATS; we want one card.

### 8.3 Stage C — Hard Pre-Filter (Python, deterministic)

Before any AI runs, drop postings the candidate cannot legally take. This saves ~80% of LLM spend.

```python
def passes_hard_filters(posting: JobPosting, profile: CandidateProfile) -> bool:
    # Geography
    if posting.location.state not in profile.preferences.geo_states \
       and not posting.location.remote_eligible:
        return False
    # Type
    if posting.type not in profile.preferences.role_types:
        return False
    # GPA floor
    if posting.jd_structured.gpa_min and profile.education[0].gpa:
        if profile.education[0].gpa < posting.jd_structured.gpa_min:
            return False
    # Sponsorship
    if posting.jd_structured.visa_sponsorship is False and profile.preferences.sponsorship_needed:
        return False
    # Salary floor
    if posting.jd_structured.salary and profile.preferences.salary_floor:
        if posting.jd_structured.salary.max < profile.preferences.salary_floor:
            return False
    # Graduation timing for new-grad roles
    if 'recent grad' in posting.jd_structured.requirements:
        if not eligible_for_grad_year(profile, posting):
            return False
    return True
```

### 8.4 Stage D — AI Fit-Scoring (Anthropic API, structured output)

**Tool:** `claude-sonnet-4-6` (cost-optimal for scoring) + Anthropic structured-output mode
**Input:** `CandidateProfile` + one `JobPosting`
**Prompt anchor:** the §0.1 no-fabrication rule + the §1/§2 Fit/Gaps structure
**Output contract** (`MatchScore`):

```ts
{
  posting_id: string,
  fit_tier: 'high' | 'medium' | 'low',
  fit_confidence: number,            // 0..1
  rationale: string,                  // 2-3 sentence WHY this tier
  matched_signals: [                  // for the Fit section bullets
    { claim: string, evidence_from_resume: string, source_url?: string }
  ],
  gap_signals: [                      // for the Gaps section
    { gap: string, severity: 'blocker'|'tweakable'|'nice-to-have', what_screening_for: string,
      adjacent_experience: string, reframe: string,
      resume_action: { directive: string, bullet_xyz: ResumeBullet, ats_keywords: string[] } }
  ],
  ats_platform_inferred?: string,
  flagged_unverifiable_claims: string[],  // anything the model wanted to say but couldn't ground
}
```

**Why structured output:** lets us reject any model response missing a `source_url` on a non-inferred claim. No-fab becomes a schema validator, not a hope.

### 8.5 Stage E — Card Generation (Anthropic API, structured output → HTML template)

**Tool:** `claude-opus-4-7` (rich card prose) + Jinja2 template
**Input:** `CandidateProfile` + `JobPosting` + `MatchScore`
**Output:** card HTML matching the §3.4 anatomy
**Constraint:** every `<sup><a class="cite">` index must come from `MatchScore.matched_signals[].source_url` or `gap_signals[].*.source_url`. The template fails the build if a citation is dangling.

The Jinja template is built once, lives in `templates/match_card.html.j2`, and is the canonical card renderer. It also generates the `card-modal-store` entry for the same card.

### 8.6 Stage F — Auto-Tighten Pass

**Tool:** Python word-counter + Claude rewrite call
**Logic:** for each card section, count visible words. If over §2.05 maxes, send the section back to Claude with a tighten prompt ("Compress to N words. Preserve every citation."). Loop until under cap. This replaces the manual tightening pass of 14 cards.

### 8.7 Stage G — Render & Publish

**Tool:** Python build script
**Output:** single static HTML (or per-tenant subroute in New FMJ)
**Publish:** for prototype keep GitHub Pages; for production push to the New FMJ Next.js app under `/match/<candidate-id>` so it inherits auth + theming.

---

## 9. Build vs. Buy: Where Python Helps, Where AI Helps

The split that worked for Nicole, generalized:

| Concern | Python | Claude | Why |
|---|---|---|---|
| API ingestion + rate-limit handling | ✅ | ❌ | Deterministic, cheap, easy to retry |
| Dedup, hard filters, salary math | ✅ | ❌ | No model needed |
| Resume parsing | ❌ | ✅ | Already New FMJ's domain |
| JD requirements extraction | partial | ✅ | LLM beats regex on unstructured "must / nice / preferred" |
| Fit/Gaps narrative writing | ❌ | ✅ | This is the model's edge |
| Citation grounding | ✅ | ✅ | Python validates the URL responds 200; Claude generates the claim |
| Tightening pass | ✅ counting | ✅ rewriting | Python decides over budget; Claude rewrites |
| Render template | ✅ | ❌ | Jinja is enough |
| Mirror publish | ✅ | ❌ | Plain `git push` |

**Don't have Claude orchestrate the pipeline.** Use it as a worker behind a Python coordinator. Two reasons: (1) determinism, retries, and cost controls live naturally in code, not in a chat loop; (2) when something breaks in the pipeline, you want a stack trace, not a "the agent decided to stop."

---

## 10. Integration with New FMJ Resume Piece

The merge plan in three increments:

### Increment 1 — Share the resume parser

New FMJ is building an AI-resume tool with `ResumeBulletSchema` and XYZ enforcement. **Reuse that exact schema** as `CandidateProfile.experience[].xyz_normalized` (§8.1). One Zod source of truth, two consumers (resume builder + match generator).

### Increment 2 — Plug the matcher into the resume builder UI

In the New FMJ app, after a candidate finishes their resume, surface a "See your matches" CTA. Clicking it:
1. Hits the Stage-B ingestion endpoint
2. Runs Stages C–G server-side
3. Renders the card stack at `/match/<candidate-id>`

This is the "wow" moment — they finish a resume and immediately see 33 cards built around it. The Nicole report is the static prototype of this.

### Increment 3 — Feedback loop into the resume builder

When a candidate stars or removes a card, write that signal back. Use it to:
- Down-weight similar postings on next refresh (Stage B re-rank)
- Flag XYZ bullets that consistently surface as gaps and prompt the resume builder to suggest rewrites

The localStorage state in §3.3 (`nicole-job-favorites`, `nicole-job-removed`) is the prototype of this signal stream. In production it persists in the candidate row in Postgres.

---

## 11. Open Risks

1. **Source URL rot.** A card built today citing an FAU FY budget URL may 404 in 6 months. Mitigation: cache the cited content into S3 + watermark with fetch date; show a stale-cite banner if the URL fails a periodic HEAD check.
2. **Jobscan / NACE paywall drift.** Public reports today might move behind a wall. Mitigation: maintain a curated list of fallback equivalent stats and let the matcher pick the freshest accessible one.
3. **LinkedIn ToS exposure.** Scraping LI is the easiest ingestion shortcut and the riskiest legally. Use the official Talent Solutions API or a licensed third-party (Coresignal, Bright Data with consent) for production.
4. **Hallucination regression at scale.** §0.1 caught it in 33 cards by hand. At 33,000 cards/day the only protector is the structured-output schema + URL liveness check from Stage E. Don't ship without both.
5. **Tightness-vs-credibility tradeoff.** Going below §2.05 thresholds saves words but kills the trust signal that rich, sourced prose buys. Don't auto-tighten past target; cap at min.
6. **Tier inflation.** "High Priority" loses meaning if everything is high. Calibrate the tier thresholds against historical interview-conversion data once we have it.
7. **Mobile-first regressions when rendering inside Next.js.** The single-file Nicole report is mobile-checked at 375px; porting into the FMJ shell may re-introduce overflow at the filter bar. Test before merge.

---

## 12. Appendix A — Source Material Catalog

Authoritative sources used across the 33 cards (deduped):

| Source | Used for |
|---|---|
| BLS OOH (Public Relations Specialists 27-3031, Editors 27-3041, Film & Video Editors 27-4032, Sales Reps Services 41-3091, Meeting/Event/Convention Planners 13-1121, Public Relations Managers 11-2032) | Median wage, projected growth, occupational baselines |
| BLS OES Florida & Miami MSA tables | Local wage adjustments |
| O*NET | Skills, related roles, hot technologies |
| NACE Job Outlook 2026 (Spring update) + Class of 2024 starting-salary | Class hiring projection +5.6%, communications major averages |
| Jobscan Fortune 500 ATS Report 2024 | 98.4% F500 use ATS, 76.4% recruiters filter by skills, 10.6× match-title interview rate |
| Workday recruiter docs | ATS keyword mechanics |
| Glassdoor company / interview pages | Difficulty ratings, time-to-hire |
| Per-employer career portals + financial planning + HR pay-plan documents | Fiscal year, role classifications, pay bands |
| SEC 10-Q / IR pages (TelevisaUnivision, etc.) | Revenue, segment performance |
| DoD SkillBridge program page | Eligibility for SkillBridge-flagged postings |
| FAU financial-planning calendar | July 1 fiscal year (used for grad-timing fit) |

## 13. Appendix B — Prompt Templates (canonical)

**B.1 Research subagent (per role, parallel)**

> You are researching one job posting for a candidate match card. Find URL-backed evidence for: employer basic facts, BLS occupational match (median wage + 2024–34 growth), employer-published HR / fiscal / pay-plan details, Glassdoor difficulty if present, ATS platform if discoverable. Return JSON: `{employer:{...}, role:{...}, bls:{wage, growth, soc}, employer_docs:[{url, what_we_used_it_for}], glassdoor:{difficulty, time_to_hire}, ats:{platform, evidence_url}}`. For any field you cannot ground with a working URL, return `null` and a `reason` string. Do not invent numbers, programs, or policies.

**B.2 Card-writer agent (per role)**

> You are writing one match card. Inputs: `CandidateProfile`, `JobPosting`, `MatchScore`, `ResearchBrief`. Produce JSON matching the `MatchCard` schema. Every numeric or specific factual claim must reference a `source_id` from `ResearchBrief`. Tightness targets (§2.05 of CARD_CONTENT_STANDARDS): summary 40–50 words, bullets 15–20, gap blurbs ≤ 25–30 each. No em dashes. No "Not just X, but Y." No "essentially," "in order to," "the fact that." Resume actions must follow XYZ (Accomplished X measured by Y by doing Z), 15–25 words.

**B.3 Tightener agent (per oversize section)**

> Compress this section to ≤ N words. Preserve every `<sup><a class="cite">` and every URL. Do not remove citations. Prefer verbs. Cut filler. Return only the compressed HTML for this section.

---

## 14. Appendix C — Open Questions for New FMJ Team

1. Does New FMJ already have a `ResumeBulletSchema` Zod definition we can import, or do we recreate it in the matcher service?
2. What's the current architectural decision on multi-tenant isolation — one Postgres per tenant or schema-per-tenant? Affects where `CandidateProfile` lives.
3. Is the Anthropic key budget centralized? Stage D + Stage E will be the single largest LLM line item.
4. Is there an existing job-ingestion contract being built for any of LinkedIn / Indeed / Workday already? Don't want to double-build.
5. Should `/match/<candidate-id>` be public-by-link (like Nicole's GitHub Pages mirror) or auth-required? The link Nicole has is public and SEO-indexable; for production candidates that's probably wrong.
6. Where does the feedback loop (§10 Increment 3) write to? A `candidate_signal_event` table seems right.
7. Who owns the per-source URL liveness monitor (§11.1)? Probably the matcher service, but worth confirming.

---

**End of handoff.** Steve to forward to the New FMJ engineering lead. Source artifact, governance docs, memory files, and conversation transcript all referenced above are accessible from the `fmj-search-compiler-v1` workspace.
