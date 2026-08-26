# Prior Art: AI Grant-Writing, Funder Matching, and Narrative Intelligence (as of Aug 2026)

> Research agent report, 7 Aug 2026. Part of the FFRH funder-intelligence landscape scan — see [README.md](README.md).

## 1. AI grant-writing tools (crowded, US-centric)

- **Grantable** ([grantable.co](https://grantable.co/)) — AI workspace for the full grant lifecycle: AI drafting that "remembers your organization," funder discovery from IRS 990 data, pipeline management. Free tier, ~$50/mo Starter, ~$150/mo Pro; $25/mo for small nonprofits. 27,000+ users. US 990-data dependent.
- **Grant Assistant (FreeWill)** ([grantassistant.ai](https://www.grantassistant.ai/), [product page](https://www.nonprofits.freewill.com/products/grant-assistant)) — closest to the proposal's RFP-matching + drafting: "Discover" uses semantic matching for RFPs; "Respond" drafts full proposals, converts complex RFPs into 2-page briefs, and runs compliance-matrix gap checks. Trained on 7,000+ winning proposals; built by ex-USAID leaders, so it does handle international-development RFPs. Enterprise custom pricing — out of reach for most Global South CSOs.
- **Instrumentl** ([instrumentl.com](https://www.instrumentl.com/)) — market leader; 450k+ funder profiles, 36k+ active RFPs, automated match alerts. $179–499/mo. Explicitly poor ROI for international nonprofits: US-foundation-centric database ([Grantsights review](https://grantsights.com/blog/instrumentl-review-2026)).
- **Granted AI** ([grantedai.com](https://grantedai.com/for/nonprofits)) — 85k grants/144 sources, RFP analysis, AI section drafting; from $29/mo; skews toward research-council funding.
- **GrantStation** ([grantstation.com](https://grantstation.com/find-grants/global)) — 15k+ curated funder profiles including a genuine international/global section (~1,255 global opportunities); list $699/yr but commonly $100–200/yr. Curated database, no AI drafting.
- **OpenGrants** ([opengrants.io](https://opengrants.io/introducing-ai-powered-grant-matching/)) — notable for publicly documenting an **embedding-based** matching approach (profile and grant text embedded, cosine-similarity matching, thumbs-up/down feedback loop). ~5,000 grants, US government focus. No "Streetlight" product surfaced; it appears not to be an established entrant.

## 2. Funder matching / prospect research — technical approaches

- **Taxonomy-based**: Candid's Foundation Directory matches via its Philanthropy Classification System (5 facets: subject, population, org type, transaction, support strategy), with an ML "autocoder" that tags grants/RFPs/missions to PCS terms, now exposed via an MCP connector ([taxonomy.candid.org](https://taxonomy.candid.org/), [AI at Candid](https://blog.candid.org/post/ai-at-candid-powering-technology-social-sector-success), [MCP connector](https://learning.candid.org/getting-started-with-the-candid-mcp-connector/375441)). So: ML-assisted classification into a fixed taxonomy, not free-text semantic matching.
- **Embedding/LLM-based**: OpenGrants (above) and Grant Assistant's semantic matching are the clearest embedding-first systems. Instrumentl claims AI "trained on winning proposals + 990 data" (hybrid).
- **Predictive-scoring (individual donors, not institutional funders)**: DonorSearch AI ([donorsearch.net/donorsearch-ai](https://www.donorsearch.net/donorsearch-ai/)) and iWave/Kindsight ([kindsight.io/iwave](https://kindsight.io/iwave/)) layer ML on wealth/990/real-estate/SEC/political-giving records to score giving propensity. This is US wealth-screening — a different problem from funder–CSO matching and irrelevant data infrastructure for India.

## 3. Narrative / signal intelligence — the differentiator claim largely holds

No product was found that systematically monitors funder **LinkedIn posts, media coverage, program-officer discourse, or sector narratives** to infer shifting funding priorities. What exists is adjacent:

- **Hatch.ai** ([hatch.ai](https://hatch.ai/)) and Momentum gather social/lifestyle/career signals — but for **individual major-donor** prospecting, not institutional funders.
- **Pivot-RP (Clarivate)** monitors 20k global funders but via published opportunities, not discourse ([clarivate.com](https://clarivate.com/academia-government/scientific-and-academic-research/research-funding-analytics/pivot-rp-funding/)).
- **Overton/Altmetric** track research citations in policy documents and news — the closest methodological analogue (mining unstructured public discourse for institutional-intent signals), but pointed at research impact, not funder intent ([overton.io](https://www.overton.io/data-collaborations/), [evaluation study](https://www.ncbi.nlm.nih.gov/pmc/articles/PMC12780511/)).
- Sector commentary confirms AI is used to "summarize funder priorities" ad hoc, not as a continuous signal pipeline ([NPQ](https://nonprofitquarterly.org/using-ai-for-fundraising-still-requires-human-strategy/)).

**Verdict: "narrative intelligence" for institutional funders appears genuinely novel as a product** — though it's a workflow sophisticated fundraisers do manually, and LinkedIn's anti-scraping ToS is a real execution risk worth acknowledging in the proposal.

## 4. Open-source assets worth evaluating

- **Grantmakers.io** ([github.com/grantmakers/grantmakers.github.io](https://github.com/grantmakers/grantmakers.github.io)) — mature, free profiles of ~150k US foundations from IRS 990-PF. Best-engineered asset, but US-only data; reusable as an architecture pattern.
- **openrfps-scrapers** ([github.com/dobtco/openrfps-scrapers](https://github.com/dobtco/openrfps-scrapers)) — standardized government-RFP scraping framework (old, US states) — pattern reusable for Indian govt/CSR portals.
- **GrantAIScraper** ([github.com/zaina-saif/GrantAIScraper](https://github.com/zaina-saif/GrantAIScraper)) — small AI grant-scraping demo; reference only.
- **Omdena AI-NGO fundraising project** ([omdena.com/projects/ai-ngo-fundraising](https://www.omdena.com/projects/ai-ngo-fundraising)) — open collaborative grant-matchmaking build; check for reusable code/collaborators.
- Nothing directly forkable found from Code for America, DataKind, or Tech To The Rescue on grant matching specifically; Fast Forward's [AI for Humanity playbook](https://www.ffwd.org/ai-for-humanity) is ecosystem context, not code.

## What's genuinely missing (positioning for the proposal)

1. **India/Global South coverage**: every major platform is US-990-data-centric. Indian entrants exist but are early: **GrantSense AI** (grant discovery + proposal automation, scrapes 100+ govt/CSR portals — [grantsense.com](https://grantsense.com/)), **GrantMatch AI** ([grantmatcher.fund](https://grantmatcher.fund/)), **ImpactX Bridge** (CSR–NGO matchmaking — [impactxbridge.com](https://www.impactxbridge.com/)), **Daanveda** ([daanveda.com](https://www.daanveda.com/)). Cite these as emerging but shallow prior art — none combine matching + drafting + signal intelligence.
2. **Narrative intelligence**: no direct competitor globally — strongest differentiator; frame it as "Overton/Altmetric methods applied to funder intent."
3. **Integrated stack at CSO-affordable pricing**: matching + RFP discovery + drafting each exist separately (Grant Assistant comes closest) but at US/enterprise prices; no one combines all four for FCRA/CSR/Global South contexts.
