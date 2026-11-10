# Reference case: Fundscreener for AAIS

Observed on 2026-11-10 in the test environment (`test-fundscreener.finfiles.nl`,
screener for ABN AMRO Investment Solutions, 136 share classes). Purpose: learn which data
Fundscreener actually shows, so the platform's catalogue and delivery API cover it.
No data was copied; this file lists fields and observations only.

## 1. What Fundscreener shows

**List view** (cards or table, 10 per page, sortable, search by name or ISIN)
- Name, ISIN, SFDR classification, NAV + currency, NAV date
- Return for a selectable period: YTD, 1 y, 3 y, 5 y, since inception
- Morningstar rating (stars)
- Documents dropdown per share class
- Filters: asset class, SFDR classification, currency, ESG score

**Fund page**, three tabs, plus a "other share classes" switcher (same fund)
- Header: name, NAV + date, ISIN, SFDR, Morningstar category
- *Characteristics*: investment objective (long text), SRI 1–7 (PRIIPs) with explanation,
  list of main risks, costs (entry, exit, management and other administrative costs,
  performance fee, swing pricing yes/no, transaction costs), general information
  (fund name, reference currency, share class currency, ISIN, share class launch date,
  domicile, legal form, management company, investment manager, type UCITS/AIF,
  minimum investment, depositary, order cut-off time, benchmark name, fund duration,
  recommended holding period, NAV frequency, master-feeder structure, oldest share
  class launch date), legal disclaimer, data source credits (MSCI, Morningstar, ABN AMRO)
- *Performance*: NAV vs benchmark chart since inception with periods 1M–60M and a
  range selector; annualised, cumulative and calendar-year returns for fund and benchmark,
  with an as-of date
- *Documents*: commercial (factsheet) and regulatory (shareholder notices, PRIIPs KID,
  prospectus, annual report, semi-annual report), each with language and date

Branding is the client's (AAIS dark teal), not FinFiles': Fundscreener is white-labelled.

## 2. Data quality issues visible today

These are exactly the problems a golden copy with lineage should catch:

| # | Observation | What the platform would do |
|---|---|---|
| Q-1 | Same share class shows SFDR **Article 6** in the table view but **"-"** on its fund page | One golden SFDR value per share class, served to both views; `required` check raises an exception when missing |
| Q-2 | 1-year return differs between list (as of 6 Nov) and fund page (as of 5 Nov) | Every value carries its effective date; the API returns one consistent as-of date per request |
| Q-3 | "Avis aux actionnaires" (shareholder notice) listed twice on the documents tab | Documents as data points with type, language, date and source; duplicate check |
| Q-4 | Chart period labels shown as raw keys (`performance.INCEPTION`, `performance.1M`…) | Front-end translation bug in Fundscreener, not a data issue; reported for the Fundscreener team |

## 3. Consequences for the platform

1. **Identifiers.** Fundscreener addresses share classes by the Morningstar SecId
   (`/instrument/F00000…`). The delivery API must accept and return **both ISIN and
   provider ids** (from `entity_identifier`), or Fundscreener needs a migration (new A-33).
2. **Catalogue scope is too narrow.** The first scope (static, fees, NAV) covers about a
   third of what Fundscreener shows. Missing groups to add to the catalogue:
   - Texts: investment objective, risk list, disclaimer (multi-language: FR, NL, EN, DE)
   - Performance: returns per period (YTD, 1/3/5 y, inception; annualised, cumulative,
     calendar) for fund and benchmark, benchmark name and series
   - Ratings and classifications: Morningstar rating and category, ESG score, asset class
   - Operational: depositary, cut-off time, minimum investment, NAV frequency,
     holding period, master-feeder, swing pricing
   - Documents: type, language, date, URL/file, source
3. **Derived values.** Returns can be taken from a provider or calculated from the golden
   NAV series. Decision needed: provider value, own calculation, or both with a check (A-34).
4. **Licensed data and credits.** Morningstar rating, category and returns are Morningstar IP:
   shown only with a licence and with the source credit, as Fundscreener already does.
   The licence model (A-05, A-17) must cover redistribution to the client's Fundscreener.
5. **One client = one Fundscreener.** The AAIS screener id in the URL maps naturally to
   one client schema and one client-scoped API key.

## 4. Open questions

- A-33: Keep Morningstar SecId as Fundscreener's key, or switch Fundscreener to ISIN?
- A-34: Performance figures: provider-supplied, calculated by the platform, or both?
- F-07: Should the AAIS Fundscreener be the first live consumer of the platform (pilot)?
