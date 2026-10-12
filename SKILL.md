---
name: bucket-pricing
description: Fact bucket for HOW MUCH, line by line and period by period - the order-line table (one row per product per period: dates, quantity, list / net unit price, discount, extended price), currency, printed totals and subtotals, whether totals are shown or suppressed, tiered / volume price bands, in-term price escalation, rate cards for services roles and one-time-discount wording. Joins to Products & Entitlements on product_key. Feeds TCV, currency, BOM match, phasing, uplift, uneven / front-end / back-end discount and per-line date checks.
metadata:
  bucket_id: pricing
  schema_version: 2.0.0
  output_model: pricing_models.py (PricingOutput)
  brd_refs: Sec 3.1 (Net Total Display), 3.2, 6.2, 6.3, 6.4.1, 7.2 step 6, 8.2.1
  tracker_rows: 11, 12 (line dates), 13, 14, 32, 33, 34, 36, 59
---

# Bucket: Pricing

This skill is self-contained: Part A is how to extract (shared by every bucket), Part B is what
this bucket extracts, Part C is the exact output structure.

# Part A - How to extract

You extract facts for **one bucket** (Pricing) from **one document**. Part B says which facts,
Part C gives the exact output structure, and the output is validated by `pricing_models.py`
(`PricingOutput`). Your output feeds three kinds of downstream checks:

| Output section | Downstream consumer | Your job |
|---|---|---|
| `values` | Fact rules in code (exact match, tolerance, date difference) | Transcribe the single value exactly + light parse |
| `records` | Calculation rules in code (sums, joins, period comparisons) | Transcribe every row of a repeating structure, completely, with stable keys |
| `evidence` | Contextual agent (policy overrides, deviations, judgements) | Capture verbatim clauses and signals; give non-binding hints only |

**Golden rule: you transcribe; code and the contextual agent decide.**

## A1. Input
- An OCR reconstruction of the PDF in **markdown**. Pages are separated by page markers
  `<<<PAGE n>>>`, where `n` is the 1-based physical page of the PDF.
- Markdown formatting in the input (`#` headings, `**bold**`, `_italic_`, `|` table cells, `-` bullets,
  `>` quotes, backticks, `<br>`, `\` escapes) is layout produced by the OCR step, not document text.

Use nothing else: not Salesforce, not Anaplan, not other documents, not general knowledge about the
customer. If a fact is not in this document, it is `not_found`.

## A2. Applicability
Read Part B section 1 and the document.
- Bucket does not apply -> `applicable: false`, a one-sentence `applicability_reason`, and
  `values`, `records`, `evidence` set to `null`. `observations` and `notes` are `[]`.
- Bucket applies -> `applicable: true` and fill **every** field Part B defines.

"Applies" means the document is the kind that could contain these facts, not that it does.
Silence in an applicable document is evidence (no opt-out clause, no budget cap), so it is reported
as `not_found`, never skipped.

## A3. Citations
Every `found`, `inferred` or `unreadable` fact carries `sources`, a list of
`{"page_number": int, "quote": [string, ...]}`. The quotes are used to **highlight the fact in the
original PDF**, so they must match the OCR text exactly.

**Rules for each string in `quote`:**
1. It is a verbatim span from **one single line** of the page. Never join two lines in one string.
2. It contains **plain text only**: no markdown syntax (`#`, `**`, `__`, backticks, `>`, leading
   `-` / `*` / `+` bullets, `<br>`, HTML tags, `\` escapes) and **never a `|` character**. Drop the
   formatting characters and keep the words between them: `**Order Schedule Number:** ACM-1`
   is quoted as `Order Schedule Number: ACM-1`.
3. Copy the characters exactly as the OCR text has them, including OCR typos, casing, punctuation,
   currency symbols and number formatting. No ellipses, no paraphrase, no translation.
4. Make it long enough to identify the fact unambiguously on the page (label + value, or the
   sentence that states it), but no longer than the line.

**When a fact spans several lines or cells, use one string per line or cell, in reading order:**
- Address block on 5 lines -> 5 strings.
- Label on one line, value on the next -> 2 strings.
- Table row -> one string per relevant cell (`| Solution Architect | 200 |` ->
  `["Solution Architect", "200"]`).
- A printed value that itself contains `|` (e.g. DocuSign `10-Apr-26 | 5:35:23 PM EDT`) -> split at
  the pipe: `["10-Apr-26", "5:35:23 PM EDT"]`. The `raw` field still keeps the full printed value.
- A fact on two pages -> two source objects, one per page.

**By status:**
- `inferred`: quote the text you inferred from. Never invent text for something that is absent.
- `unreadable`: quote the partially legible text as the OCR gives it; if nothing is legible, use
  `["[illegible]"]` with the page number.
- `not_found` / `not_applicable`: `sources` is `[]`.

## A4. Status and confidence
| Status | Use when | raw / parsed |
|---|---|---|
| `found` | Stated in the document | raw = as printed (formatting removed); parsed = light parse |
| `inferred` | Not stated directly but follows from specific text | raw = the text relied on; parsed = your reading; confidence at most `medium` |
| `not_found` | Bucket applies, document is silent | both null, no sources, no candidates |
| `not_applicable` | This field cannot exist in this document type | both null, no sources, no candidates |
| `unreadable` | Text is there but illegible | raw = what is legible; parsed = null |

`confidence`: `high` = explicit and unambiguous; `medium` = explicit but wording or formatting
leaves some doubt, or inferred; `low` = conflicting or barely legible.

## A5. Light parsing
`raw` is the truth (as printed, markdown removed); `parsed` is a convenience that code re-checks.
Put `parsed: null` rather than guessing.

| Type in Part C | parsed format | Rules |
|---|---|---|
| Text | string | Trim whitespace and line breaks; do not rewrite. |
| Number | number | Remove thousands separators; respect decimal commas (`1.234,56` -> 1234.56). Percentages in percent units (`99.5%` -> 99.5). |
| Boolean | true / false | Only from explicit wording; otherwise null. |
| Date | `YYYY-MM-DD` | Only if unambiguous. `03/04/2026` with no locale evidence -> null + `normalization` note. |
| Time | `HH:MM:SS` (24 h) | Seconds `00` if not printed. |
| Timestamp | `YYYY-MM-DDTHH:MM:SS` | Local time as printed; **no conversion, no offset**. |
| UtcOffset | `+HH:MM` / `-HH:MM` | Only when printed as an offset or an unambiguous zone listed in Part B. |
| Money | `{"amount": number, "currency_iso": "USD"}` | ISO 4217. `$` alone is ambiguous -> `currency_iso: null` unless the document names the currency. |
| CountryCode | ISO 3166-1 alpha-2 (`US`, `GB`) | |
| SubdivisionCode | ISO 3166-2 (`US-IL`, `CA-ON`) | |
| LanguageCode | ISO 639-1 (`en`, `fr`) | |
| CurrencyCode | ISO 4217 (`USD`, `EUR`) | Only when the document names the currency or uses an unambiguous symbol (`€`, `£`). |
| enum (`"a" \| "b"`) | one listed value | If nothing fits, use `other` where listed and keep the wording in raw. Never force a fit. |

**Never compute.** No totals, sums, differences, term lengths, execution dates, discounts, FX or
time-zone conversions. If the document prints a total, transcribe it. Code does the arithmetic.

## A6. Conflicts -> candidates
If the document states the same fact differently in two places, put the reading you judge primary
in the value, every other reading in `candidates` (each with its own sources), set
`confidence: low`, and add a `conflict` note. Do not reconcile.

## A7. Records (repeating structures)
- One item per row / entry **as printed**. Do not merge, split, de-duplicate or re-order.
- `key`: follow the key rule in Part B. Keys must be unique within the collection.
- Leaves are `{"raw", "parsed"}`; `status` and `sources` sit on the item (quote the row's line or cells).
- A column that is blank for a row -> `{"raw": null, "parsed": null}`.
- `completeness`: `items_extracted` = number of items you output; `items_declared` = a count the
  document itself prints, else null; `continues_beyond_extraction` = true if the table visibly
  continues somewhere not in this document; `status` = `complete` / `partial` / `unknown`.
- `declared_totals`: every total / subtotal printed for the collection, transcribed.
- Empty collection: `items: []`, `items_extracted: 0`, status `complete`.

## A8. Evidence (for the contextual agent)
**Clauses**: capture each clause matching one of Part B's clause categories.
- `clause_id`: `C1`, `C2`, … in document order.
- `text_verbatim`: the clause text with markdown formatting removed and line breaks kept.
- Length: full text for categories marked ★ (full-verbatim). Otherwise up to 1,500 characters; if
  cut, set `text_truncated: true` and stop at a sentence end.
- `sources`: one string per line of the clause (and one source object per page).
- `overrides_hint`: fill it when the clause says it modifies, replaces, supplements or conflicts with
  another document or clause; otherwise null.

**Signals**: short factual pointers explaining why something may matter.
- `signal_id`: `S1`, `S2`, … Link `clause_id` when the signal comes from a captured clause.
- Record weak signals too. `description` states what is printed, not a conclusion.

**Hints**: one entry for every hint Part B defines, every time the bucket applies.
- `likely` must cite at least one `signal_id`; `unlikely` / `unclear` may have an empty list.
- Hints are leanings, not verdicts.

## A9. Observations and notes
- `observations`: useful facts this bucket does not define but that matter to contract review
  (cited). Do not repeat facts already in a field.
- `notes`: `conflict`, `normalization`, `missing` (expected but absent), `multi_document`,
  `truncation`, `other`, with `field_path` when about a specific field.

## A10. Fact paths
Downstream checks, agent findings and human overrides address facts by path:
`pricing.values.<field>`, `pricing.records.<collection>[<key>].<leaf>`,
`pricing.evidence.clauses[<clause_id>]`, `pricing.evidence.signals[<signal_id>]`.

## A11. What the validator checks
`pricing_models.py` rejects the output if any of these fail; it is retried once with the errors,
then routed to a human.
1. Structure, field names, types and enum values match Part C exactly (no extra fields).
2. Every quote string is plain text (no `|`, no markdown syntax, no line break) and is found on a
   single line or table cell of its page.
3. Status rules: `found` / `inferred` / `unreadable` have sources; `not_found` / `not_applicable` have
   none and null raw / parsed; `inferred` is never `high` confidence.
4. `values` / `records` / `evidence` are null exactly when `applicable` is false.
5. `items_extracted` equals the number of items; record keys are unique; every item has a source.
6. Clause and signal IDs are unique and every reference to them exists.
7. Every hint appears exactly once; `likely` hints cite a signal; ★ clauses are not truncated.

# Part B - Pricing

**Core rule: transcribe the order-line table row by row; compute nothing.** TCV, discounts,
phasing, uplift and Year-1 ACV are all calculated by code from these rows. List prices normally come
from the Salesforce quote lines (an Anaplan OS does not print them), so missing list prices are
expected, not an error.

## 1. Applicability
**Applies** to any document that prints prices, amounts or rates: OS, SOW, change order, amendment,
quote, marketplace private offer, PO with amounts, partner agreement with financial terms.

**Does not apply** to documents with no amounts or price terms (e.g. an SSA with no pricing, a DPA,
an SLA exhibit, a Deal Desk email).

## 2. Ownership boundaries
| Fact | Owner bucket | Join |
|---|---|---|
| Product identity, type, SKU, bundles, entitlements | Products & Entitlements | `product_key` |
| Contract term start / end | Term & Termination | line dates are here |
| Invoice schedule, payment days, billing frequency, PO amount requirements | Billing, Payment & Tax | |
| Renewal price caps, future discounts, MFN, options at a fixed price | Special Terms | in-term escalation is here |
| Services budget caps, milestone amounts | Services Delivery | services role **rates** are here (`rate_card`) |

## 3. product_key (shared with Products & Entitlements)
- SKU printed -> the SKU upper-cased with spaces removed: `ANP-EPU`.
- No SKU -> the product name lower-cased, every run of non-alphanumeric characters replaced by `_`,
  trimmed: `Anaplan Enterprise Planner User` -> `anaplan_enterprise_planner_user`.

## 4. values (fact rules)
| Field | Type | Extraction notes |
|---|---|---|
| `currency` | CurrencyCode | From the currency label, column header or amounts. Several currencies -> primary in the value, others in `candidates`, `conflict` note. |
| `grand_total` | Money | The total net amount of the order as printed ("Total Net Fees", "Grand Total", "Total Contract Value"). Never summed by the model. Several totals (e.g. per year plus overall) -> the overall one here; per-year totals go in `lines.declared_totals`. |
| `net_total_display` | enum `shown`, `included_suppressed`, `not_printed` | `included_suppressed` when the total or line amounts read "Included" or are blanked out (BRD 3.1 Net Total Display). |
| `list_prices_printed` | Boolean | true if any list price column or list amount is printed; false (found) if the table has no list prices (quote the header row cells). |
| `price_escalation_pct` | Number | Stated in-term or renewal increase, percent units. |
| `price_escalation_basis` | enum `in_term`, `on_renewal`, `cpi_linked`, `other` | When the increase applies. |

## 5. records (calculation rules)

### lines: one item per printed row of the order-line table
| Leaf | Type | Notes |
|---|---|---|
| `product_key` | Text | See section 3. |
| `product_name` | Text | As printed on the row. |
| `line_number` | Text | Row number as printed, if any. |
| `period_label_raw` | Text | "Year 1", "Ramp 2", "Initial Term", if printed. |
| `period_start` / `period_end` | Date | Row start and end dates as printed. |
| `quantity` | Number | |
| `unit_list_price` | Money | Only if printed. |
| `unit_net_price` | Money | Net / sales price per unit as printed. |
| `discount_pct` | Number | Only if printed; never computed. |
| `extended_net_price` | Money | Row total as printed. "Included" -> raw "Included", parsed null. |

Key: `<product_key>@<period_start>` (`ANP-EPU@2026-06-01`); `na` when the row has no start date;
`:<n>` for repeats.

Rules:
- One item per printed row, in table order, including zero-price and "Included" rows.
- Amounts in cells without a currency take the order currency **only if** the document names it
  (header or label); otherwise `currency_iso: null`.
- Every printed subtotal and total for the table ("Year 1 Total", "Total Net Fees") goes in
  `lines.declared_totals`.
- A table continuing onto another page: one source object per page for the rows on each page.

### price_tiers: one item per printed price band
Key `<product_key>:<n>`. Leaves: `product_key`, `band_from`, `band_to` (null for open-ended), `unit_price`.

### rate_card: one item per services role rate
Key `role_key` (role name lower-case, punctuation removed, spaces as `_`; same as Services Delivery
`resourcing`). Leaves: `role_name`, `rate` (Money), `rate_unit` (`hour`, `day`, `week`, `month`, `other`).

## 6. evidence (contextual agent)
**Clause categories** (★ = full-verbatim): `pricing_terms`, `price_escalation` ★,
`tiered_or_volume_pricing` ★, `one_time_discount` (e.g. "one-time discount, not applicable to future
Order Schedules"), `total_suppression`, `fx_terms`, `rate_card_terms`, `other_pricing`.

**Signal types**:
| Type | Record when |
|---|---|
| `uplift_language` | Prices increase by a stated % or amount during the term or on renewal |
| `tiered_pricing` | Price bands by quantity |
| `volume_discount_language` | Discounts tied to volume or commitments |
| `one_time_discount_language` | Discount stated as one-time / not applicable to future orders |
| `total_suppressed` | Totals or line amounts shown as "Included" or blank |
| `zero_price_line` | A row priced at 0 or "no charge" |
| `list_price_absent` | No list prices printed (normal on an OS; record once) |
| `totals_mismatch_printed` | Two printed totals for the same thing disagree |
| `other` | Anything else about pricing a reviewer should see |

**Hints** (each exactly once):
| Hint | Lean on… | Feeds |
|---|---|---|
| `uplift_present` | Unit prices for the same product rise during the term | #36 Uplifted Annual Price |
| `volume_or_tiered_discount` | Pricing depends on volume bands or volume-based discounts | #33 Front End / Back End / Volume Discount (volume part) |

## 7. Code-side derivations (never done by the model)
- `sum_lines` = Σ `extended_net_price`; compared with `grand_total` and `declared_totals`.
- Per product: quantity and unit net price by period (phasing, uplift).
- Discount per line = 1 - net / list, using SF quote-line list prices when the OS prints none.
- Year-1 ACV and total NNACV for Rule 8.2.1; TCV (Reporting) with SYS02 FX rates.
- `min_line_start` / `max_line_end` across rows.

## 8. Validations fed (rule registry)
| Rule | Type | Check | Fact paths |
|---|---|---|---|
| Gate G1 BOM match | calc | Every OS row matches a primary-quote line (product, dates, quantity, net amount) and vice versa; G1 failure blocks SKU-flag checks | `records.lines` |
| #13 Currency | fact | `currency` = SF `CurrencyIsoCode`; every row in that currency | `values.currency`, `records.lines[*].extended_net_price` |
| #14 TCV (LC) | calc | `sum_lines` vs SF TCV (tolerance 1.00); `grand_total` vs `sum_lines`; suppressed total -> use `sum_lines`, mark "Total suppressed" | `values.grand_total`, `values.net_total_display`, `records.lines` |
| #11 / #12 Line dates | calc | SF start = `min_line_start`, SF end = `max_line_end` | `records.lines[*].period_start`, `.period_end` |
| #34 Phased Deal (Rule 6.4.1) | calc | Same product with different quantity across periods -> TRUE; also price phasing; Year-1 ACV % (Rule 8.2.1) | `records.lines` grouped by `product_key` |
| #36 Uplifted Annual Price | calc | Unit net price of a product rises > 0.5% period over period (not caused by quantity) or a stated uplift | `records.lines`, `values.price_escalation_pct`, hint `uplift_present` |
| #59 Uneven Discount | calc | Per-line discount deviates >= 0.05 pts from the product average (list from SF) | `records.lines` + SF list prices |
| #33 FE / BE / Volume | calc | Discount higher in earlier (front-end) or later (back-end) periods by >= 0.05 pts; tiered bands -> volume | `records.lines`, `records.price_tiers` |
| #32 / Rule 6.3.1 | context | One-time-discount boilerplate -> no future discount; current-deal discount for the material-right comparison | `evidence.clauses[one_time_discount]` |
| BRD 7.2 s6 Rate card | calc | Every Services Delivery `resourcing` role has a `rate_card` row | `records.rate_card` ⋈ resourcing on `role_key` |
| BRD 3.1 Net Total Display | fact | Suppressed totals recorded for the reviewer | `values.net_total_display` |

# Part C - Output structure

Return **one JSON object** exactly in this shape. Every key shown is required (use `null` /
`[]` as the rules allow); no other keys are allowed.

## C1. Shared shapes

```jsonc
Source        = { "page_number": int >= 1, "quote": [string, ...] }        // >= 1 plain single-line string
Value<T>      = { "raw": string | null, "parsed": T | null,
                  "status": "found" | "not_found" | "not_applicable" | "inferred" | "unreadable",
                  "confidence": "high" | "medium" | "low",
                  "sources": [Source], "candidates": [Candidate] }
Candidate     = { "raw": string, "parsed": string | number | boolean | Money | null, "sources": [Source] }
Leaf<T>       = { "raw": string | null, "parsed": T | null }
Money         = { "amount": number | null, "currency_iso": string | null }   // ISO 4217, 3 letters
Item          = { "key": string, "status": "found" | "inferred" | "unreadable", "sources": [Source] (>= 1), ...leaves }
Collection<I> = { "completeness": Completeness, "declared_totals": [DeclaredTotal], "items": [I] }
Completeness  = { "items_extracted": int, "items_declared": int | null,
                  "continues_beyond_extraction": boolean, "status": "complete" | "partial" | "unknown" }
DeclaredTotal = { "label_raw": string, "value": Leaf<Money>, "sources": [Source] }
Clause        = { "clause_id": string, "clause_number_raw": string | null, "heading_raw": string | null,
                  "text_verbatim": string, "text_truncated": boolean, "category_hints": [category],
                  "overrides_hint": OverridesHint | null, "sources": [Source] }
OverridesHint = { "target_document_raw": string | null, "target_clause_raw": string | null,
                  "relationship": "modifies" | "replaces" | "supplements" | "conflicts" | "unclear" }
Signal        = { "signal_id": string, "signal_type": signal_type, "description": string, "entity_raw": string | null,
                  "strength": "strong" | "moderate" | "weak", "clause_id": string | null, "sources": [Source] }
Hint          = { "hint": hint_name, "value": "likely" | "unlikely" | "unclear", "signal_ids": [string] }
Observation   = { "label": string, "value_raw": string, "sources": [Source] }
Note          = { "type": "conflict" | "normalization" | "missing" | "multi_document" | "truncation" | "other",
                  "field_path": string | null, "message": string }

Scalar types: Text = string, Number = number, Boolean = boolean, Date = "YYYY-MM-DD",
Time = "HH:MM:SS", Timestamp = "YYYY-MM-DDTHH:MM:SS", UtcOffset = "+HH:MM" | "-HH:MM",
CountryCode = "US", SubdivisionCode = "CA-ON", LanguageCode = "en", CurrencyCode = "USD", Money as above.
```

## C2. `pricing` output

```jsonc
{
  "bucket": "pricing",
  "schema_version": "2.0.0",
  "applicable": true,                // false -> values, records, evidence are null
  "applicability_reason": string,

  "values": {
    "currency": Value<CurrencyCode>,
    "grand_total": Value<Money>,
    "net_total_display": Value<"shown" | "included_suppressed" | "not_printed">,
    "list_prices_printed": Value<Boolean>,
    "price_escalation_pct": Value<Number>,
    "price_escalation_basis": Value<"in_term" | "on_renewal" | "cpi_linked" | "other">
  },

  "records": {
    "lines": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "product_key": Leaf<Text>,
      "product_name": Leaf<Text>,
      "line_number": Leaf<Text>,
      "period_label_raw": Leaf<Text>,
      "period_start": Leaf<Date>,
      "period_end": Leaf<Date>,
      "quantity": Leaf<Number>,
      "unit_list_price": Leaf<Money>,
      "unit_net_price": Leaf<Money>,
      "discount_pct": Leaf<Number>,
      "extended_net_price": Leaf<Money>
    }>,
    "price_tiers": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "product_key": Leaf<Text>,
      "band_from": Leaf<Number>,
      "band_to": Leaf<Number>,
      "unit_price": Leaf<Money>
    }>,
    "rate_card": Collection<{
      "key": string, "status": ItemStatus, "sources": [Source],
      "role_name": Leaf<Text>,
      "rate": Leaf<Money>,
      "rate_unit": Leaf<"hour" | "day" | "week" | "month" | "other">
    }>
  },

  "evidence": {
    "clauses": [Clause],
    "signals": [Signal],
    "hints": [Hint]
  },

  "observations": [Observation],
  "notes": [Note]
}
```

Allowed lists for this bucket:

- `category` (clause `category_hints`): `pricing_terms`, `price_escalation` ★, `tiered_or_volume_pricing` ★, `one_time_discount`, `total_suppression`, `fx_terms`, `rate_card_terms`, `other_pricing`
  (★ = full-verbatim, never truncated)
- `signal_type`: `uplift_language`, `tiered_pricing`, `volume_discount_language`, `one_time_discount_language`, `total_suppressed`, `zero_price_line`, `list_price_absent`, `totals_mismatch_printed`, `other`
- `hint_name` (each exactly once): `uplift_present`, `volume_or_tiered_discount`
