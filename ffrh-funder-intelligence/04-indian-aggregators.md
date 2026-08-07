# India CSR/Nonprofit Aggregator Landscape — Research Report

> Research agent report, 7 Aug 2026. Part of the FFRH funder-intelligence landscape scan — see [README.md](README.md).

## 1. CSRBOX (csrbox.org)
**What it does:** India's largest CSR knowledge/media platform: corporate CSR profiles, CSR project database, RFP/grant/job listings (7,500+ opportunities/yr via sister site NGOBox), and paid reports. It's a matchmaking/media platform, not management software ([csrbox.org](https://csrbox.org/home/), [Transunifyy comparison](https://www.transunifyy.com/blog/top-10-csr-management-software-india-2026)).
**Openness:** Browsing RFPs, project pages, and company profiles is largely free. NGO memberships are paid annual tiers — Rs 10k / 27k / 59k + GST — buying visibility, proposal forwarding, and promotion; memberships were "temporarily closed due to website revamp" at check time ([Price-List](https://csrbox.org/Price-List)). **No public API.** Detailed CSR-spend intelligence comes as sold reports.
**NE relevance:** Real but thin: a paid *North East India CSR Report* (85 companies, Rs 342.6 Cr spend, 287 projects in FY17-18), NE CSR Forum events in Guwahati, and profiles of NE NGOs (NEN, SeSTA) ([NE report](https://csrbox.org/India_CSR_report_North-East-India-CSR-Report-2019_55)). Also note the niche site [northeastcsr.in](http://northeastcsr.in/index.php?p=home).

## 2. Samhita / GoodCSR (samhita.org, goodcsr.in)
**What it does:** Consulting-first. GoodCSR is its corporate-facing platform (project discovery, GoodCSR Direct project management, tracking/reporting); originally Gates Foundation/Tata Trusts-backed, with a ~5,000+ NGO database shared historically across GoodCSR/SociallyGood/Goodera ([YourStory](https://yourstory.com/socialstory/2019/03/india-csr-goodera-sociallygood-goodcsr-lgc4dot6f4), [goodcsr.in](https://www.goodcsr.in/)).
**Openness:** Publishes thought-leadership reports openly; the NGO database and matchmaking are behind corporate engagements. No API, no nonprofit-facing funder intelligence. Low value as a data source; possible services competitor/partner.

## 3. GiveIndia / Give Grants (give.do, givegrants.in)
**What it does:** Give.do is India's largest donation platform — 2M+ donors, ~2,500-3,000+ **verified** nonprofits after strong due diligence ([giveindia.org](https://www.giveindia.org/), [give.do/aboutus](https://give.do/aboutus)). **Give Grants** (built on the acquired Goodera India CSR business, 50+ corporate clients; Give has 200+ corporate partners) is a corporate grant-lifecycle stack: nonprofit discovery/onboarding with a "comprehensive diligence framework," fund-utilization tracking, financial audit, and impact measurement ([acquisition post](https://give.do/blog/give-acquires-gooderas-leading-india-csr-grant-management-business/), [givegrants.in](https://givegrants.in/) — site was unreachable at check time).
**Openness:** Corporate-side SaaS/service; **nonprofits do not get funder intelligence through it** — they get onboarded/evaluated. No public API found, but Give has a partnership-oriented posture and also runs *Give Discover* (open insights portal, [give.do/discover](https://give.do/discover/)).
**NE relevance:** Pan-India verified-NGO pool; no NE-specific product.

## 4. India Partner Network (IPN)
**What it is:** A **Sattva Consulting initiative** — free capacity-building platform for small/mid nonprofits: 4,000+ registered nonprofits across 36 states/UTs, 6,000+ users, 350+ multilingual resources, a new NGO **Directory**, and a **Donor Connects** matchmaking feature disseminating funding/training opportunities ([indiapartnernetwork.org](https://www.indiapartnernetwork.org/), [about](https://blog.indiapartnernetwork.org/about), [Donor Connects](https://www.indiapartnernetwork.org/donor-connects)).
**Openness:** Free membership; data depth per NGO is modest (self-registered profiles). Sattva itself is a serious data player (Sattva Knowledge Institute, India Data Insights) — the partnership conversation is really with Sattva.

## 5. GuideStar India — the Candid analogue (flagged)
**What it is:** Run by CSIS India (an Indian trust) with GuideStar International UK + TechSoup lineage; India's largest verified-NGO information repository — 10,000+ NGOs fully searchable free at guidestarindia.org, ~1,750+ certified ([LinkedIn](https://in.linkedin.com/company/guidestar-india), [portal](https://guidestarindia.org/)).
**Certification ladder** (transparency/governance/compliance rigor increases per tier): **Transparency Key → Silver (KYC/Transparency) → Gold (Advanced — governance norms) → Platinum (Champion — full accountability + impact reporting)**; certified-NGO lists downloadable as PDFs ([resources.guidestarindia.org](https://resources.guidestarindia.org/), [certification brochure](https://guidestarindia.org/SiteImages/Certifications/HelpGuideCertification.pdf)).
**Caveats:** No public API; portal tech is dated (ASP.NET pages); data is self-reported-but-substantiated. Contrast: NGO Darpan (NITI Aayog) has 573k+ registrations but no verification depth. NE coverage exists but certified-NGO density in NE states is likely low — worth quantifying directly.

## 6. Other tools worth knowing (2026)
- **NGOBox** (CSRBOX group) — the de-facto free RFP/EOI/grants/tenders feed for India; actively updated with 2025-26 deadlines ([rfp_eoi_listing](https://ngobox.org/rfp_eoi_listing.php)). Best free scraping/aggregation target for live RFPs.
- **fundsforNGOs** — global; dedicated India category and premium donor-search database (paid, ~modest fee); useful for international funders into India, weak on CSR ([India tag](https://www.fundsforngos.org/category/developing-countries-2/india/), [premium](https://home.fundsforngospremium.com/)).
- **DevNetJobsIndia** — jobs-first with tender/consultancy postings; marginal for grants.
- **Credibility Alliance** — accreditation norms body (Basic/Desirable norms), complements GuideStar certification; not a discovery tool.
- **GivingTuesday India / DaanUtsav** — campaign infrastructure, not funder data.
- **AI grant tools:** the 2026 crop (Instrumentl, Granted AI, Grant Assistant, FundRobin) is US/global-funder oriented — **none has Indian CSR/RFP coverage** ([Grant Assistant roundup](https://www.grantassistant.ai/resources/articles/the-best-ai-grant-writing-tools-for-nonprofits-in-2026)). This is the gap this project would fill.

## Top partnership / data-source candidates
1. **GuideStar India** — the only verification-graded, open-searchable NGO database; natural data partner for nonprofit-side credibility signals (mirrors Candid MCP access on the US side). No API, so a data-sharing agreement is the path.
2. **NGOBox/CSRBOX** — richest live RFP/CSR-profile feed; freely crawlable today, and a commercial listing/report partnership is cheap (Rs 10k-59k tiers). Only player with NE-specific CSR reports.
3. **IPN/Sattva** — free, growing NGO network with Donor Connects; Sattva's data arm makes them the strongest strategic (vs. data) partner. Give Grants is corporate-side only — a distribution channel later, not a data source now.
