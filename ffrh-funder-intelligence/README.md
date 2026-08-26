# FFRH Funder-Intelligence Landscape Research

Research conducted 7 Aug 2026 via six parallel research agents, to inform the build-vs-partner
decision for the Fractional Fund Raising Hub (FFRH / Node/SeSTA) funder-intelligence proposal —
an open data + query layer plus a protected persona layer for North East India CSOs.

## Contents

| File | Thread |
|---|---|
| [01-daanveda-sattva.md](01-daanveda-sattva.md) | Indian competitors: DaanVeda, Sattva FundAssist |
| [02-candid-benevity-grant-guardian.md](02-candid-benevity-grant-guardian.md) | International platforms: Candid, Benevity, Grant Guardian, Claude for Nonprofits |
| [03-indian-statutory-data.md](03-indian-statutory-data.md) | Statutory data accessibility: csr.gov.in, CSR-1, NGO Darpan, BRSR, NITI Aayog, NEC |
| [04-indian-aggregators.md](04-indian-aggregators.md) | Aggregators: CSRBOX/NGOBox, Samhita, GiveIndia/Give Grants, IPN, GuideStar India |
| [05-ai-grant-tools-prior-art.md](05-ai-grant-tools-prior-art.md) | Global prior art: AI grant writing, funder matching, narrative intelligence, open source |
| [06-ne-india-context.md](06-ne-india-context.md) | NE India context: CSR spend verification, funder landscape, Node/SeSTA/FFRH, intermediaries |

## Synthesis

### TL;DR

The proposal's core bet holds up: **no one combines funder–CSO matching + RFP discovery +
proposal drafting + narrative intelligence for India, and nothing at all specializes in the
North East.** But the proposal's "What Already Exists" scan misses several important players
(GuideStar India, NGOBox, GrantSense AI, National CSR Exchange), understates how leverageable
Candid/Benevity actually are, and its headline CSR statistic is out of date — the disparity is
now *worse* than claimed. The data layer is buildable but harder than the note implies: nothing
statutory offers an open project-level API; everything is scrape-or-buy.

### What's genuinely available to leverage (open/free)

- **Candid's PCS taxonomy is CC BY 4.0** — a free, openly licensed classification system
  (subject, population, support strategy) that can be adopted wholesale for tagging NE CSOs and
  grants. Candid also ingests Indian FCRA data and its Grants API can query US-foundation grants
  to Indian recipients — more India-relevant than the proposal's table suggests.
- **Benevity's Claude MCP connector is free with a paid Claude plan, its connector code is open
  source, and its 2.4M-org database includes FCRA-compliant Indian nonprofits** — usable today
  for org discovery.
- **data.gov.in has official free REST APIs for CSR spend** (state-wise, sector-wise, company
  year-wise, FY2018-19→2022-23) — aggregates only, but a clean starting layer.
- **NGOBox (CSRBOX's sister site) is the de-facto free live RFP/EOI feed for India** — actively
  updated, freely crawlable, the single best target for the "proximal RFPs" use case.
- **Claude for Nonprofits**: up to 75% off (Team seats from $8/user/mo) plus free AI-fluency
  training; India eligibility unverified but not excluded — worth applying.
- **Existing scrapers to reuse as templates**: spfrantz/MCA-CSR-Scraper (csr.gov.in, stale,
  needs rewrite) and pallavmahamana/ngosearch (working NGO Darpan AJAX wrapper).

### What exists but is closed — partner candidates

- **DaanVeda** has already built most of the proposed stack (40–52K CSR company profiles, 195K+
  RFPs, AI matching, proposal drafting, free tier, ₹1–2L/yr paid). But it's ~27 people with
  under $150K raised, closed data, no API, credit-metered — partner-plausible precisely
  *because* it's small and hungry, but fragile. A cheap pilot partnership is worth testing
  before building the funder-database portion from scratch.
- **Sattva** is really three things: FundAssist (curated funder data + advisory, no AI),
  **India Partner Network (a Sattva initiative** — 4,000+ NGOs, free, "Donor Connects"
  matchmaking), and India Data Insights (open data portal). Sattva is the strongest *strategic*
  partner in the landscape; FundAssist alone doesn't substitute for a build.
- **GuideStar India** — the biggest omission from the proposal's scan. Candid's India analogue:
  10,000+ NGOs searchable free, ~1,750 certified on a four-tier transparency ladder
  (Transparency Key → Silver → Gold → Platinum). No API; the path is a data-sharing agreement.
  Natural source for nonprofit-side credibility signals.
- **Give Grants** is corporate-side only — nonprofits get evaluated through it, not intelligence
  from it. A distribution channel later, not a data source now.
- **Grant Guardian is free-to-use but NOT open source** (gated behind foundation verification).
  Its published playbook — grantmaker-defined indicators, LLM extraction with citations, human
  override, multi-currency — is a copyable blueprint for a diligence tool reading Indian audited
  statements, but there's no code to fork.

### The statutory data layer: harder than the note implies

Nothing offers an open project-level API. Accessibility verdicts:

| Source | Verdict |
|---|---|
| MCA CSR Portal (csr.gov.in) | Scrape-only, **geo-blocked (403 to non-India IPs)** — scraper must run from India; latest data FY2023-24, ~12–18 month lag |
| MCA CSR-1 registry | Effectively closed — no bulk list; one-at-a-time verification or paid MCA21 document views |
| NGO Darpan | Scrape-only but feasible (public AJAX endpoints); self-reported, stale data |
| BRSR (NSE/BSE) | Per-company XBRL downloads, no bulk endpoint, NSE is bot-hostile; no open parser exists |
| NITI Aayog dashboards | Public JS dashboard, undocumented JSON API is the extraction route |
| NEC / state CSR cells | Fragmented; Assam has a CSR Authority but no functioning open data portal — a genuine gap, though NEC sanction lists are on data.gov.in |

Paid shortcut: **Dataful sells a company+project-level CSR master dataset (705K rows,
FY2014-15→2024-25)** — likely the cheapest way to bootstrap the funder data layer versus months
of scraping; worth pricing out in month one. Also newly relevant: the government's own
**National CSR Exchange (csrxchange.gov.in)** — a login-gated matchmaking portal for registered
implementing agencies, signalling government movement into this space.

### What genuinely doesn't exist (the defensible gaps)

1. **Narrative intelligence for institutional funders** — nothing anywhere systematically mines
   funder LinkedIn/media/sector discourse to infer shifting priorities. Closest analogues
   (Hatch.ai, Overton/Altmetric) target different problems. Strongest differentiator — with one
   execution risk to acknowledge: LinkedIn's anti-scraping ToS.
2. **NE India specialization** — zero coverage from any platform, Indian or global.
3. **An integrated stack at CSO-affordable pricing** — the pieces exist separately (FreeWill's
   Grant Assistant is closest on RFP-to-draft) but at US enterprise prices; emerging Indian
   entrants (GrantSense AI, ImpactX Bridge, GrantMatch AI) are shallow and none combine all four
   capabilities.
4. **Open-source anything in this space for India** — the open data/query layer positioning is
   genuinely unoccupied.

### Corrections the proposal should make

- **The CSR statistic is stale and understated.** FY2024-25 (per Parliament, Aug 2026): the
  eight NE states received ~₹855 crore of ₹40,794 crore ≈ **2.1%**, while Maharashtra is now
  **~21%** (₹8,631 cr), not 17%. Assam alone is 71% of the NE total — seven states are nearly
  invisible to CSR. Caveat: state-wise figures exclude "PAN-India" allocations, so phrase
  carefully.
- **A timing argument the note doesn't use**: the May 2025 Rising Northeast Investors Summit
  drew ₹4.3 lakh crore in commitments (Reliance ₹75K cr, Adani ₹50K cr, Vedanta ₹30K+ cr). As
  those localize, Section 135 CSR obligations follow into NE states — making CSO
  funder-readiness time-critical.
- **Azim Premji Foundation explicitly prioritizes NE India** in its early-stage CSO grants
  program — a named engaged funder with a live, matching program.
- **Foundation for Social Transformation (Guwahati)** — indigenous NE grantmaker + capacity
  builder — is the closest local analogue and is absent from the note; likely partner or at
  minimum someone not to blindside.
