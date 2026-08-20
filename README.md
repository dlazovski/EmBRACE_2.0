# EmBRACE 2.0 — CompanyWall lead generation

An n8n workflow that builds a lead list for the **EmBRACE** EU grant program (micro/small
enterprises in eligible Croatian counties) by scraping [companywall.hr](https://www.companywall.hr)
through [ScrapingBee](https://www.scrapingbee.com) and writing one complete row per company
into Google Sheets.

**Workflow file:** [`n8n/companywall-embrace-lead-gen.json`](n8n/companywall-embrace-lead-gen.json)

---

## What it does

One run = **one county + one size category**. There is deliberately no loop over counties;
you edit the **Config** node by hand and run the workflow again for the next combination.

```
Manual Trigger → Config → Init Pagination
     ↓
 ┌── Build Search URL → Wait 3s → ScrapingBee (search page)
 │        ↓
 │   empty page 1? → retry once with render_js=true
 │        ↓
 │   no results → END      failed page → log + skip to next page
 │        ↓
 │   Split Out Companies → Add Company Fields
 │        ↓
 │   Loop Over Companies (batch size 1)
 │        ↓
 │     Wait 3s → ScrapingBee (company profile) → Add Profile Fields
 │        ↓
 │     Lookup OIB in "Raw Leads" → duplicate? → skip + log
 │        ↓
 │     Append Lead Row → log
 │        ↓
 └── Next Page (page += 1, stop after page 2 when test_mode)
```

Search-page data and profile-page data are gathered in a single pass and land on the **same
item**, so every appended row is complete. There is no Merge node — the profile fields are
simply added onto the item that already carries the search-page fields.

---

## Setup

### 1. Import

n8n → **Workflows** → **Import from File** → pick `n8n/companywall-embrace-lead-gen.json`.

### 2. ScrapingBee credential

Both HTTP Request nodes expect a credential of type **Header Auth → Query Auth**
(`HTTP Query Auth`) named **`ScrapingBee`**. If it doesn't exist yet:

**Credentials → New → Query Auth**

| Field | Value |
| --- | --- |
| Name (of the credential) | `ScrapingBee` |
| Parameter Name | `api_key` |
| Parameter Value | *your ScrapingBee API key* |

Then open each of the three ScrapingBee nodes (`Fetch Search Page`, `Retry Page 1 With JS`,
`Fetch Company Profile`) and select that credential — the exported JSON carries a placeholder
credential id, so n8n will ask you to pick it once.

### 3. Google Sheets credential

Both `Lookup OIB in Sheet` and `Append Lead Row` are set to **Authentication: OAuth2** and expect a
`Google Sheets OAuth2 API` credential. Select your existing credential on both nodes.

If your existing credential is a **Service Account** instead, switch the **Authentication** dropdown
on both nodes to *Service Account* first — the credential picker only lists credentials matching the
selected authentication type.

The Google account behind the credential needs **edit** access to the spreadsheet (for a service
account, share the sheet with its `client_email`).

The target spreadsheet is hard-coded:

- **Spreadsheet ID:** `1NHEAPRSMdIECCZ06QmtVZZIhtocWK-omXAY1ObNQjdw`
- **Tab:** `Raw Leads` — header row must already exist in row 1; the workflow only appends below it.
- **Tab:** `Selected` — never touched by the workflow. Copy the rows you want there by hand.

Expected `Raw Leads` headers, in this order:

```
Name | Profile URL | Status | OIB | Employees | Revenue | County | Size Category | Owner/Director | Email | Founding Date | NKD Code | MBS
```

---

## Configuring a run

Edit the **Config** node before every execution.

| Field | Type | Notes |
| --- | --- | --- |
| `county_code` | number | See table below — pick exactly one |
| `county_name` | string | Must match `county_code`; written into the Sheet as plain text |
| `employees_from` / `employees_to` | number | Micro: 1–10 · Malo: 11–49 |
| `revenue_from` / `revenue_to` | number | EUR. Micro: 0–2,000,000 · Malo: 2,000,000–10,000,000 |
| `size_category` | string | `Micro` or `Malo` |
| `test_mode` | boolean | `true` → stop after page 2 (pages 1 and 2 are fully processed and appended first) |

### EmBRACE-eligible counties

| Code | County | | Code | County |
| ---: | --- | --- | ---: | --- |
| 1 | Zagrebačka | | 12 | Brodsko-posavska |
| 3 | Sisačko-moslavačka | | 13 | Zadarska |
| 4 | Karlovačka | | 15 | Šibensko-kninska |
| 7 | Bjelovarsko-bilogorska | | 16 | Vukovarsko-srijemska |
| 9 | Ličko-senjska | | 17 | Splitsko-dalmatinska |
| 11 | Požeško-slavonska | | 19 | Dubrovačko-neretvanska |

Leave `test_mode = true` for the first run against a new county, confirm the rows in the Sheet
look right, then set it to `false` and run the full county.

---

## ⚠️ Profile-page fields: how they are extracted

The search-page selectors are the ones supplied with the spec and work as-is (they fill
columns A–H). The **profile page** is different: `companywall.hr` is blocked by the network
egress policy of the environment this workflow was built in, so it was never possible to fetch
a sample profile and read its markup.

Rather than guess at CSS classes, `Fetch Company Profile` requests the **raw HTML** (no
`extract_rules`) and `Add Profile Fields` parses it. Every tag boundary becomes a line break,
so `<span>MBS</span><span>080123456</span>` becomes two consecutive lines, and each field is
found by matching its Croatian label and taking the value beside it. This depends on nothing
but the page's own label wording. Verified against sibling-`<span>`, `<table>`, `<dl>`,
inner-tag-wrapped (`<b>`/`<small>`) and same-line `Label: value` layouts, including
entity-encoded labels (`&Scaron;ifra djelatnosti`).

Email is handled separately: a `mailto:` link wins, because the value next to an `E-mail`
label is often a link whose text is *Pošalji*, not the address.

**If a column is still blank**, the node prints a diagnostic to the execution log for the first
few companies:

```
--- PROFILE PAGE TEXT SAMPLE (NEKA TVRTKA d.o.o.) ---
missing fields: nkd_code, mbs
Osnovni podaci | OIB | 12345678901 | Matični broj | 080123456 | ...
--- end sample ---
```

That shows the real label wording. Add it to the `LABELS` map at the top of the
`Add Profile Fields` code node — no selectors involved:

```js
const LABELS = {
  mbs: ['MBS', 'Matični broj subjekta', 'Matični broj'],   // <- add yours here
  ...
};
```

Because the profile response is full HTML rather than a small JSON payload, execution data is
larger (roughly 200–500 KB per company). This is fine for a normal county, but on a very large
run consider setting the workflow's *Save successful production executions* to **Do not save**.

## Reliability and cost

**Failure handling.** Every ScrapingBee and Google Sheets node has *Retry On Fail* (2 retries)
plus *Continue On Fail*, so nothing kills a whole run:

- A **search page** that fails after retries is logged (county + page number) and **skipped** —
  pagination continues with the next page.
- A **profile page** that fails after retries still produces a row, written from the search-page
  data with `Owner/Director`, `Email`, `Founding Date`, `NKD Code` and `MBS` left blank. The
  lead is never lost.
- An **empty page 1** is retried once with `render_js=true` before concluding there are no
  results — this is the guard against the site's reCAPTCHA / session-cookie gate silently
  returning an empty page. An empty **page 2+** just means pagination is done.

**Progress logging.** Watch the n8n execution log (or the browser console in the editor):

```
--- Page 3 -> https://www.companywall.hr/pretraga?...
Page 3: 15 companies in search results.
Written: NEKA TVRTKA d.o.o. (OIB 12345678901)
Skipped duplicate: 98765432109 (DRUGA TVRTKA d.o.o.)
Page 3 done: 14 companies written, 1 duplicates skipped, 0 profile fetches failed (run totals: 41 written / 4 skipped)
```

`End Run` prints a final summary including the full list of skipped pages and failed profiles,
and returns it as JSON, so nothing disappears silently.

**Deduplication** is per-company, on **OIB**, immediately before writing. A company already in
`Raw Leads` from any earlier run — a different county/size combination, or a re-run of the same
one — is skipped. This costs one extra Sheets read per company, which is the intended tradeoff.

*Caveat:* if the Sheets lookup itself errors out (after its retries), the company is treated as
new and the row is appended — better a rare duplicate than a lost lead. Duplicates like this are
easy to spot by sorting on OIB.

**Numbers stay text.** `Employees` and `Revenue` are written exactly as the site returns them
(`"1.568.788,47"`, `"3"`). Nothing in the workflow parses or converts them, and the append uses
`cellFormat: RAW` so Google Sheets can't reinterpret Croatian decimal formatting either. Clean
them up manually afterwards.

**ScrapingBee credits.** Each company costs **2 calls**. Calls run with `render_js=false`
(1 credit) instead of ScrapingBee's default of `true` (5 credits); only the empty-page-1 retry
uses `render_js=true`. If *every* page comes back empty, flip `RENDER_JS` to `true` at the top
of the `Build Search URL` code node.

**Runtime.** 3 s before every call, so ~6 s per company plus request time. A county with 300
companies takes roughly 45–60 minutes. Keep n8n's execution timeout high enough, or work through
the county in `test_mode` batches.

---

## Known n8n caveats

- **`Loop Over Companies` reset.** Its *Reset* option is the expression
  `{{ $prevNode.name === 'Add Company Fields' }}`. This is required: without it the Split In
  Batches node keeps the finished batch context from the previous page and reports "done"
  immediately on page 2 onwards. If you rename `Add Company Fields`, update this expression too,
  or pagination will silently process only page 1.
- **Wait nodes.** The 3-second waits keep the execution in memory (n8n only offloads waits
  longer than 65 s), so a long run holds one execution slot for its whole duration.
