# International Nonprofit-Data Platforms: Candid, Benevity, Grant Guardian, Claude for Nonprofits

> Research agent report, 7 Aug 2026. Part of the FFRH funder-intelligence landscape scan — see [README.md](README.md).

## 1. Candid (Foundation Center + GuideStar)

**APIs.** Candid's developer portal ([developer.candid.org](https://developer.candid.org/)) offers Essentials (search), Premier (full profiles), Charity Check (compliance), Grants, News, Demographics, and Taxonomy APIs ([overview](https://candid.org/data/explore-apis/)). These are **open APIs in the "documented, key-based" sense, but paid/sales-mediated** — API access sits in the custom-priced Enterprise tier; no public self-serve free API tier could be verified as of 2026.

**Pricing** ([candid.org/pricing](https://candid.org/pricing/)): Free plan for verified nonprofits (limited search, claim/update own profile); Premium $219/mo or $1,199/yr; Ultimate $1,699/yr (adds compliance status, Landscapes); Enterprise custom (includes APIs). Nonprofits under $1M revenue get Premium free by earning a Gold Seal of Transparency ([NonProfit PRO](https://www.nonprofitpro.com/article/candid-launches-unified-search-for-nonprofit-and-grant-data/)).

**Claude connector.** Launched Dec 2025 with Claude for Nonprofits ([Candid press](https://candid.org/press/candid-partners-with-anthropic-to-bring-trusted-data-to-claude-for-nonprofits/), [tutorial](https://claude.com/resources/tutorials/using-the-candid-connector-in-claude)). Beta MCP connector; covers 1.9M+ orgs, org search, taxonomy lookup, knowledge base. Requires only a **free Candid account** and a paid Claude plan; pulls public data only ([Candid blog](https://candid.org/blogs/claude-for-nonprofits-candid-mcp-connector-access-nonprofit-data-ai-assistant/)).

**India data: yes, limited but real.** Candid explicitly ingests **Indian Ministry of Home Affairs FCRA foreign-contribution data** on Indian nonprofits ([data sources](https://candid.org/about/our-data/data-sources/)), and the Grants API supports "recipient country if not USA" — so US foundation grants to Indian recipients are queryable ([Grants API FAQ](https://developer.candid.org/reference/faqs-grants-api)). Core depth remains US-centric.

**Taxonomy (PCS): genuinely open.** The Philanthropy Classification System is licensed **CC BY 4.0**, downloadable as Excel, and vendors may embed it as their default taxonomy with attribution ([taxonomy.candid.org/resources/downloads](https://taxonomy.candid.org/resources/downloads), [Candid blog](https://candid.org/blogs/why-we-made-candids-philanthropy-classification-system-available-to-use/)).

## 2. Benevity

**Claude connector.** The Benevity General Nonprofit MCP Server (launched Giving Tuesday 2025) exposes their Search Engine API to Claude — search by cause/location/keyword, returning profiles with mission and geographic focus. Free with any paid Claude plan ([Benevity MCP docs](https://causeshelp.benevity.org/hc/en-us/articles/43364091494164-Benevity-nonprofit-MCP-server), [claude.com/connectors/benevity](https://claude.com/connectors/benevity)).

**Database & India.** ~2.4–2.5M vetted nonprofits across ~200 countries, built from government registers plus vetting partners TechSoup Global, LexisNexis, Charity Navigator, Moody's ([due diligence](https://benevity.com/solutions/nonprofit-due-diligence)). **Indian nonprofits are included**: Benevity handles FCRA-compliant disbursements, and all orgs on its India platform are FCRA-compliant, able to receive foreign and local funds ([money movement](https://benevity.com/solutions/money-movement)). Indian NGOs self-register free via the Causes Portal.

**API access.** [developer.benevity.org](https://developer.benevity.org/) documents APIs, but they are **for Benevity's corporate/enterprise clients — not an open public API**. The MCP connector is the only no-contract access path.

## 3. Grant Guardian (PJMF)

**What it is.** AI financial due-diligence tool: grantmakers define indicators/thresholds/weights; nonprofits upload existing financial documents; Claude (3.5 → 3.7/4.0 in production) extracts data and computes ratios with source citations and human override ([product page](https://www.mcgovern.org/our-work/data-solutions/grant-guardian/), [Claude customer story](https://claude.com/customers/pjmf)).

**Licensing.** **Free to use for verified philanthropic institutions — NOT open source.** Access is gated (AWS Cognito SSO/MFA, contact product@mcgovern.org). No code repository exists publicly.

**Documents.** IRS 990s, balance sheets, audited and unaudited income statements (PDF uploads). Notably, PJMF **built multi-currency conversion specifically to support international organizations** ([Medium post](https://medium.com/patrick-j-mcgovern-foundation/social-responsibility-comes-first-product-development-lessons-from-grant-guardian-3c5fe6916019)) — so Indian audited statements are plausibly processable, but **FCRA returns (Form FC-4) support is unverified**; no India-specific claim exists.

**Architecture (published).** Claude + custom prompt engineering, AWS hosting, encryption in transit/at rest, human-in-the-loop at decision points, bias testing on 121 orgs, no funding recommendations made by the AI.

## 4. Claude for Nonprofits (Anthropic)

Launched Dec 2, 2025 with GivingTuesday ([announcement](https://www.anthropic.com/news/claude-for-nonprofits)): **up to 75% off** (Team seats from $8/user/mo; Enterprise custom), described as global ("organizations across the world") though country eligibility isn't itemized — India eligibility unverified but not excluded. Includes the three **open-source MCP connectors** (Candid, Benevity, Blackbaud — Anthropic states the connector code is on GitHub and customizable), free "AI Fluency for Nonprofits" course, and implementation partners (Bridgespan, Vera Solutions, etc.) ([help center](https://support.claude.com/en/articles/12893767-getting-started-with-claude-for-nonprofits), [Future of Good](https://futureofgood.co/anthropic-partners-with-benevity-blackbaud-and-candid-to-widen-non-profit-ai-access/)).

## Reusable for India NE vs. architecture-reference only

**Directly leverageable:**
- **Candid PCS taxonomy (CC BY 4.0)** — the biggest win: a free, openly licensed classification system to tag India NE nonprofits/grants by subject, population, support strategy.
- **Benevity MCP connector** — Indian FCRA-compliant nonprofits are in the database; usable today for org discovery with any paid Claude plan; the connector code is open source and adaptable.
- **Candid's FCRA-derived data + Grants API (recipient country)** — evidence US foundation grants to Indian orgs are queryable; worth a sales conversation, though core India NE depth will be thin.
- **Claude for Nonprofits discount ($8 seats) + free AI Fluency course** — apply for eligibility; likely usable by an Indian nonprofit tech project.

**Architecture references only:**
- **Grant Guardian** — closed source, foundation-gated; but its published playbook (grantmaker-defined indicators, LLM extraction with citations, human override, multi-currency, bias testing) is a directly copyable blueprint for a Claude-based tool reading Indian audited statements and FCRA FC-4 returns.
- **Candid Premium/Enterprise data products** — pricing and US-centricity make them a model (unified search, transparency seals, compliance check) rather than a source for India NE.

**Unverified:** Candid API free-tier existence; Grant Guardian's handling of FCRA returns; formal India eligibility for Anthropic's nonprofit discount.
