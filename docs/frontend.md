# Front end concept

Status: **proposal for discussion.** Implemented later in `web` (Laravel Blade).
A clickable prototype on synthetic data lives in `prototype/index.html`.

## 1. Who uses the interface

| Role | Typical person | Main jobs |
|---|---|---|
| Data steward | Fund data / operations team at the asset manager | Check today's data, resolve exceptions, enter corrections, explain a value |
| Viewer | Product, sales, reporting, compliance | Look up a fund, see the trusted value and where it came from |
| Client admin | Head of data at the asset manager | Priority overrides, users, API keys, exports |
| FinFiles operator | FinFiles support | Onboard clients and licences, monitor ingestion across clients (one client at a time) |
| API consumer | Fundscreener, client systems | No screens: delivery API only |

## 2. Design principles

1. **Every value shows its source.** Golden value, source badge, effective date and
   received date are always visible together. One click opens the lineage: all
   candidate values, their rank and why one was chosen.
2. **Exceptions first.** The steward's day starts with what needs a decision, not with
   a data dump.
3. **Time travel is a first-class control.** A "Known at" control in the top bar replays
   the platform as it was at a past moment (bitemporal model). The whole UI respects it.
4. **Nothing is overwritten.** Corrections, overrides and resolutions add entries; the UI
   never offers "edit value" on provider data, only "add manual correction" with a reason.
5. **One client per session.** The client name is always shown in the top bar. Operators
   switch client explicitly; there is no cross-client view.
6. **Licences are visible.** A provider the client is not licensed for appears as
   "not licensed", never with values.
7. **Standard names.** Data point labels show the canonical name; the EMT/EPT/openfunds
   reference is shown in the catalogue and on hover.

## 3. Visual design and navigation

The interface follows the **FinFiles Design System v1.0** (`finfiles_designsystem.md`):
dark-blue topbar `#004890` with the white FinFiles logo and the product name
"Data Management", Inter, white cards on `#F5F7FA`, the FinFiles palette with fixed
meanings (green = OK, red = blocking, orange = attention, grey = secondary).

The design system forbids a sidebar with many items, so navigation is three topbar
items with tabs inside each area:

```
Topbar:  [FinFiles logo / Data Management]  Universe | Data quality (badge) | Settings   [Known at ▾] [client] [user]
         Data quality tabs: Overview · Exceptions
         Settings tabs:     Priority rules · Sources · Delivery · Audit log · Catalogue
Footer:  FinFiles logo (partner logo only once a data partner is live, per co-branding rules)
```

No Morningstar co-branding yet: it applies only once Morningstar data is actually shown
(A-25), using the approved assets from `morningstar-brand-guidelines-finfiles.md`.

## 4. Screens

| Screen | Content | Main actions |
|---|---|---|
| Universe | Funds and their share classes the client follows, grouped by fund: ISIN, origin (provider data / client-created), ongoing charges, latest NAV, status | Search, filter, open detail, **add share classes**, **create fund**, **remove / restore** |
| Data quality · Overview | Golden copy coverage, open exceptions by severity, last ingestion per provider, NAV freshness | Jump to exceptions, to a failed run |
| Share class detail | Tabs: Characteristics & fees, NAV, History. Each row: golden value, source, effective date, received date | Expand lineage, add manual correction |
| Exceptions | Queue with severity, check, data point, age; detail panel with candidate values | Accept a source value, correct manually (reason required), ignore with comment |
| Priority rules | Cascade: global default (read-only, changed via config PR) → client override → data point overrides | Reorder sources, add data point override; every save audited |
| Sources | Providers, licence period, last runs, payload counts, mapping version | Re-run ingestion (operator), view raw payload metadata |
| Delivery | API clients (e.g. Fundscreener) with scopes and expiry; scheduled exports and last runs | Create/revoke API key, run export |
| Audit log | Who did what, when, on which object | Filter by user, action, date |
| Catalogue | Data points, type, unit, standard reference, quality checks | Read-only (changed via config PR) |

## 5. Key flows

1. **Morning check:** Overview → red tile "2 blocking" → Exceptions → NAV jump on a share
   class → compare Provider A vs B → accept Provider B → golden value recomputed, audit entry.
2. **Explain a value:** Funds → share class → ongoing charges → lineage shows Provider A
   1.45 % (rank 1, chosen), Provider B 1.52 % (rank 2) → client can answer an auditor.
3. **Correct a value:** Exception "missing SRI" → manual correction 4, reason
   "KID dated 2026-10-01" → stored as source MANUAL, provider data untouched.
4. **Replay the past:** Known at = 2026-10-01 → ongoing charges shows the value delivered
   that day, before the provider update of 2026-10-15.
5. **Override priority for one data point:** Priority rules → NAV → put Provider B first
   for this client → takes effect from now; past golden values unchanged.

6. **Add share classes to the universe:** Universe → Add share classes → list of what the
   licensed providers cover but the client does not follow yet → select → added; values
   arrive with the next provider run. Share classes only an unlicensed provider covers are
   shown but cannot be selected. An exception "not in universe" (a provider file mentions an
   unknown ISIN) offers "Add to universe" directly.
7. **Create a fund (e.g. a new mutual fund launch):** Universe → Create fund → fund name,
   legal form (UCITS mutual fund FGR, NV, SICAV, AIF), domicile, SFDR; one or more share
   classes with ISIN (check digit validated), name, currency, distribution policy, inception
   date, ongoing charges, management fee. Values are stored as source **Client entry** with
   user and time. When a provider later covers the fund, provider values take priority
   (Client entry ranks last by default, A-30).
8. **Remove from the universe:** inline confirmation, then collection and delivery stop
   (including to Fundscreener). Nothing is deleted: the share class appears under
   "Removed" and can be restored; "Known at" still shows it for earlier dates.

9. **Custom data points (AAIS):** Universe → Data points lists every data point the
   Fundscreener shows, whether it is *standard* (catalogue definition, Morningstar leads) or
   *custom* (AAIS manages it), and who delivers it. "Make custom" makes AAIS the leading
   source from that moment. Only the first customisation level (value override) is shown;
   the levels are pending (A-37); the data point page then shows one row per
   share class with the golden value, the Morningstar value and an editable AAIS value.
   Saved values are source "AAIS", audited, and marked as changed.
10. **Create a house fund:** same flow as 7, values stored as source "AAIS".
11. **Go to Fundscreener:** yellow button in the topbar opens a preview in the AAIS
    Fundscreener look (list, fund page with Caractéristiques / Performances / Documents),
    fed by the golden copy. Fields changed in Data Management carry a "Modifié" tag.
    A link opens the real test Fundscreener, which will show the same once it reads the
    delivery API.

## 6. Open questions

- F-01: Is the web interface for asset manager staff only, or also for FinFiles sales demos with Fundscreener side by side?
- F-02: Language of the interface: English only, or also Dutch/French?
- F-03: ~~House style~~ Resolved 2026-11-10: FinFiles Design System v1.0 applied.
- F-05: Who may change the universe: data steward, client admin, or both? Approval needed for removals that affect Fundscreener?
- F-06: Can clients create share classes in an existing provider-covered fund (e.g. a new share class not yet delivered by providers)?
- F-04: Four-eyes approval for manual corrections (A-06)?
