# Indian CSR/Funder-Intelligence Data Sources — Accessibility Report

> Research agent report, 7 Aug 2026. Part of the FFRH funder-intelligence landscape scan — see [README.md](README.md).

## 1. MCA National CSR Portal (csr.gov.in) — **Scrape-only (India IP required); partial bulk via mirrors**
- Exposes company-, project-, state-, district-, and sector-wise CSR spend from annual MCA21 filings, searchable by FY ([taxguru summary](https://taxguru.in/company-law/csr-spending-details-data-access.html)).
- **Latest FY: 2023-24** (₹34,909 cr, 27,188 companies, 59,634 projects) ([India CSR](https://indiacsr.in/csr-spend-jumps-from-rs-10065-cr-to-rs-34908-cr-2024/)); expect ~12–18 month lag. MCA's CDM mirror ([mcacdm.nic.in/csr-data](https://www.mcacdm.nic.in/csr-data)) lags further (FY 2022-23 "as on 31 Mar 2024") and has a broken TLS cert.
- **No public API or documented CSV export.** Verified the site returns **HTTP 403 to non-Indian/automated clients** (both WebFetch and curl with browser UA) — it is geo/bot-blocked, so scraping must run from an Indian IP. HTML tables are scrapeable; an old scraper exists ([spfrantz/MCA-CSR-Scraper](https://github.com/spfrantz/MCA-CSR-Scraper), targets the pre-redesign `master_search.php`, FY2014-17 only — needs rewrite).

## 2. MCA CSR-1 registry — **Effectively closed**
- Since 1 Apr 2021 NGOs must file Form CSR-1 to receive CSR funds; MCA issues a CSR Registration Number and maintains the implementing-agency database ([ClearTax](https://cleartax.in/s/form-csr-1)).
- **No public bulk list found.** CRNs are verifiable one-at-a-time; individual CSR-1 filings are viewable via MCA21's paid "View Public Documents." The [National CSR Exchange](https://www.csrxchange.gov.in/) lists registered IAs but is a matchmaking portal (login), not open data. Verdict: query-only / no download.

## 3. NGO Darpan (ngodarpan.gov.in) — **Scrape-only (feasible)**
- Public [state-wise directory](https://ngodarpan.gov.in/index.php/home/statewise) with name, Darpan ID, registration details, FCRA no., sectors ("field of work"), state/district filters. Site is publicly reachable (200 OK, verified).
- **No official public API.** Third parties scrape its internal AJAX/JSON endpoints ([pallavmahamana/ngosearch](https://github.com/pallavmahamana/ngosearch) wraps it in a Flask REST API); commercial verification APIs exist ([Surepass Darpan ID API](https://surepass.io/darpan-id-verification-api/)).
- Quality issues: self-reported, stale/expired registrations, uneven sector tagging. Pre-compiled state-wise dumps exist on [Dataful (collection 637)](https://dataful.in/collections/637/) (paid).

## 4. BRSR filings (SEBI via NSE/BSE) — **Scrape-only, per-company XBRL; no open bulk dump**
- Top-1000 listed companies file BRSR as PDF + XBRL to exchanges; XBRL mandatory from FY 2023 data ([XBRL.org analysis](https://www.xbrl.org/unlocking-the-potential-of-esg-disclosures-in-india-a-must-read-analysis/) — 1,000+ NSE XBRL reports, ~1,600 datapoints each).
- NSE lists filings with per-company XBRL download at [corporate-filings-annual-reports-xbrl](https://www.nseindia.com/companies-listing/corporate-filings-annual-reports-xbrl); BSE equivalent via its Listing Centre / [XBRL page](https://www.bseindia.com/static/about/xbrl_info.aspx). NSE is notoriously bot-hostile (cookie/header handshake) but scriptable.
- **No official bulk endpoint.** No dedicated open-source BRSR parser found on GitHub; generic XBRL parsers apply, and XBRL International showed xBRL-JSON conversion is straightforward ([report PDF](https://www.xbrl.org/wp-content/uploads/2024/02/Unearthing-Insights-from-Indias-ESG-Disclosures.pdf)). Structured BRSR data is sold by [Dataful (ESG collection 736)](https://dataful.in/collections/736/).

## 5. NITI Aayog Champions of Change — **Public dashboard, no documented export**
- [championsofchange.gov.in/dashboard](https://championsofchange.gov.in/dashboard) is public (200 OK): delta rankings and 49 KPIs across 5 themes for 112 aspirational districts (NE districts included); the Aspirational Blocks variant sits at [abp.championsofchange.gov.in](https://abp.championsofchange.gov.in/abp_login) with granular data behind login ([NITI ADP page](https://www.niti.gov.in/aspirational-districts-programme)).
- No CSV export documented; it's a JS dashboard, so the underlying JSON API is the practical extraction route. NITI publishes analytical PDFs (e.g., [Deep Dive report](https://niti.gov.in/sites/default/files/2023-03/DEEP-DIVE-Insights-from-Champions-of-Change-The-Aspirational-Districts-Dashboard.pdf)). Verdict: scrape-only (undocumented API).

## 6. NEC / NE state CSR cells — **Fragmented, effectively closed online**
- NEC ([necouncil.gov.in](https://necouncil.gov.in/)) offers project/UC status-check portals ([services.india.gov.in](https://services.india.gov.in/service/detail/check-the-status-of-various-schemes-and-projects-funded-by-north-eastern-council-nec)) but no open dataset; however **NEC project-wise sanction lists are on data.gov.in** ([Assam 2021-23](https://www.data.gov.in/resource/project-wise-details-projects-sanctioned-north-eastern-council-nec-state-assam-2021-22), [2023-24](https://www.data.gov.in/resource/project-wise-details-project-sanctioned-under-north-eastern-council-nec-during-2023-24)).
- Assam has a **CSR Authority of Assam (CSRAA)** (single-window body under AIDC; [notice](https://aidcltd.assam.gov.in/sites/default/files/swf_utility_folder/departments/aidc_industry_uneecopscloud_com_oid_20/menu/document/notice-logo-contest.pdf)) and a [state CSR policy PDF](https://finance.assam.gov.in/sites/default/files/swf_utility_folder/departments/agriculture_com_oid_2/menu/document/csr_policy.pdf), but **no functioning open CSR data portal was found**. [northeastcsr.in](http://northeastcsr.in/index.php?p=home) is a private news/info site, not a data source.

## Reusable existing datasets/projects
1. **data.gov.in OGD CSR datasets** (free, official REST API with key): [state/UT-wise 2018-19→2022-23](https://www.data.gov.in/resource/stateut-wise-details-corporate-social-responsibility-csr-expenditure-2018-19-2022-23-0), [sector-wise](https://www.data.gov.in/resource/development-sector-wise-details-corporate-social-responsibility-csr-expenditure-2018-19-0), [company year-wise](https://www.data.gov.in/resource/year-wise-csr-corporate-social-responsibility-spent-companies-mca-ministry-corporate), plus the [csr keyword index](https://www.data.gov.in/keywords/csr). Aggregates only — not project-level.
2. **Dataful CSR Master (dataset 1612)** — 705k rows, FY2014-15→2024-25, company+project level, CSV/XLSX/Parquet, **paid** ([link](https://dataful.in/datasets/1612/)); the cheapest shortcut to a full csr.gov.in mirror.
3. **GitHub**: [spfrantz/MCA-CSR-Scraper](https://github.com/spfrantz/MCA-CSR-Scraper) (stale but a template), [pallavmahamana/ngosearch](https://github.com/pallavmahamana/ngosearch) (Darpan). No open BRSR-specific parser found.
4. **CivicDataLab / Open Budgets India** ([civicdatalab.in](https://civicdatalab.in/)) — public-finance pipelines, no CSR dataset, but useful tooling patterns.

**Bottom line:** nothing offers an open API at project level. The practical stack is data.gov.in aggregates + an India-hosted csr.gov.in scraper (or Dataful license) + Darpan AJAX scraping + per-company NSE XBRL pulls; CSR-1 and NE state cells are the genuine gaps.
