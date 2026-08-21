 # APLA Connect Data — Manual vs. Automated Reference

**Purpose:** answer "what's manual, what's automated, what's the source, and why"
for any field in the product launch pipeline, without re-deriving it from scratch
in a meeting. This is the accumulated result of the flow-mapping work, the
Smartsheet automation audit, and the RDM field-ownership analysis.

**How to read the "Why manual" column:**
- **No structured source** — nothing upstream has this value in any system; a
  human is the only place it exists
- **Judgment call** — a person is making a real decision, not just relaying data
- **Cross-team handoff** — data exists but lives with a different team/tool and
  has to be manually carried across
- **Re-keying** — the value is derivable/constant but a person re-enters it anyway
  (a good automation candidate)
- **Unconfirmed** — inferred, not verified with the actual person who does it

---

## 1. Trigger and pre-RDM pipeline (NetSuite / Smartsheet side)

| Step | Manual/Automated | Source | Why |
|---|---|---|---|
| Photograph field flips to "To Be Photographed" | **Automated** | NetSuite | Event-driven NS script, fires on Round 2 QA save. One-shot, not polled — if the human step upstream doesn't happen, nothing fires (no safety net) |
| NS → Smartsheet sync (new SKU into New PD Launches) | **Manual** | NetSuite CSV export | Someone exports and uploads a CSV into an unconfirmed tool (platform never confirmed — Mickey Avila is the contact) |
| Naming and description | **Manual** | Smartsheet (New PD Launches sheet directly, NOT a separate sheet) | Judgment call — copywriter (Ecco) writes original content. "Ecco View" is a filtered Report on New PD Launches, not an independent source (confirmed via 100% value match on real export cross-check) |
| Tracking fields (Photograph status, Assignee, dates) | **Mostly automated** | Smartsheet, ~90 confirmed automation rules (owned almost entirely by Mickey Avila) | Most status fields cascade automatically off one trigger change (e.g., Render Status change auto-updates 3-5 other fields, records dates, clears checkboxes). Only the *initial* status-setting action is human |
| "Request CMS Creation" checkbox | **Automated** | Smartsheet automation | Confirmed rule: fires when Render Status changes to "Clipping Panda" (second-to-last render review stage). Not a person checking a box. **Caveat added 2026-08-13:** no live row currently carries the "Clipping Panda" status — the values in use are CAD Approved (287), blank (163), Uploaded to Web (14), Assigned to Contractor (12). Either the status is transient and rarely caught mid-flight, or the pipeline no longer passes through it. Worth confirming with Mickey |
| Ecomm creates CMS record | **Manual** | Person, triggered by above | Someone has to actually go create the Oscar/RDM record once flagged |
| Render via FTP (Mickey) | **Manual** | FileZilla upload | Physical file transfer step. Feeds into a longer internal review pipeline (1S/1S → Adam Review → Mykola → Christina Review → Clipping Panda → Uploaded to Web) — deliberately not fully diagrammed at that depth per Christine's guidance, but confirmed real via automation trigger names |
| "Help Produce Notes" / How Produced | **Manual, no structured source** | NetSuite field, also visible directly in RDM's Gemstones tab | Free-text notes a person writes and later has to translate into structured fields. Real example seen: vendor casting instructions dated back to 2020 |
| Gemstone SKU decode (e.g. `D1.2RDFGSI1`) | **Manual today, but parseable** | Free-text "Melee/Side Stones/Gems" notes field | The SKU pattern itself is decodable (validated 43/43 against real data), but it lives inside hand-typed prose mixed with quantities and asides — not a clean structured field |
| FJ Merch Team pricing upload | **Manual, cross-team** | NetSuite item master | Triggered by "Anticipated Launch Date," ~2+ weeks lead time. Weekly Mon EOD export / Tue AM "New PD Pricing Needed" NetSuite search email tells Merch what needs pricing |
| "Pricing complete" / "Merch Completed" tracking fields | **Split verdict — see note** | Smartsheet | **Refined 2026-08-13.** `Merch Completed` really is unused: 0% checked everywhere, including on the Merch Specs sheet that owns it. But there is a second column one letter apart, **`Merch Complete`**, which is live — 10.1% in progress, 40.8% on launched, 63.1% on Merch Specs. And `Pricing complete` is not dead either; it is maintained at 19.1% on **Merch Specs on New Product**, just not on the launch sheets. So the original reading was measuring the wrong columns. The real finding is duplicated field names across sheets with nothing reconciling them |

---

## 2. RDM-side manual editing (from real change-log analysis)

This section reflects your own field-ownership audit — who actually touches which
RDM field, based on genuine edits (with form-save noise removed; ~43% of raw log
entries were noise, not real edits).

### Cherian Chacko — 2,429 of 3,163 products touched

| Field | Manual/Automated | Why |
|---|---|---|
| `custrecord_rdm_interlink_group` | **Manual, judgment call — curated by Ecomm and Merch, not by Cherian alone** | 97% his share of edits. ~~Created 365/368 interlink sets~~ — **corrected 2026-08-13.** He is the `owner` on 365 of 368, but Christine clarified that reflects a bulk backfill import; the team hand-builds interlinks in the tool he wrote. She last-modified 156 of them. ~~~40% derivable by shared SKU prefix~~ — **corrected 2026-08-19.** That figure counted sets whose members merely *share a prefix*, which is a weaker test than reproducing the set. A proposer built and scored against all 364 real sets reproduces **19% exactly**, another 20% partially, and **misses 61% entirely**. Worse, precision is **8%**: it generates 867 candidate groups to find 69 real ones, because SKU structure groups many products nobody chose to interlink. Only the narrow `lab_pair` rule (`BE4D7933` / `BE4D7933LC`, natural vs lab) is reliable enough to suggest on its own. The rest group by Total Carat Weight, **Style** (largest unresolved category, 40 sets), Gemstone, or Band Width — "same style" has no confirmed rule |
| `custrecord_rdm_sync_status` | **Manual re-trigger** | His single most frequent action (6,980 changes). Whether this is a workaround for a failure or a normal publish step — unconfirmed, needs asking |
| `udf_hp_notes`, `udf_pd_po_link` | **Automated — sourced from NetSuite item master** | ~~No structured source~~ — **corrected 2026-08-13.** Both match the item master at 100%, and `rdm_recon_mr` logs writing them from it (`hp_notes` 340 entries, `pd_po_link` 19), with reason `"update RDM from NetSuite source item"`. The Smartsheet search was looking in the wrong system. Cherian's hand edits are corrections on top of the sync, not original entry |
| `custrecord_rdm_status` | **Manual, judgment call** | Downstream effect of setting this is unconfirmed |

### Ashley Tamblyn — 878 products touched (not previously known to be involved)

| Field | Manual/Automated | Why |
|---|---|---|
| `custrecord_rdm_match_sets` | **Manual, judgment call** | Ring-to-band pairing decision — mechanism differs from Cherian's interlinking, rule (if any) unconfirmed |
| `custrecord_rdm_push_schedule` | **Manual, likely re-keying** | Evidence of batch behavior: 252 products set to one identical date, 179 to another — looks like a bulk operation being done one record at a time. If confirmed, a per-product automation button won't fit this workflow; may need a batch tool instead |
| `custrecord_rdm_component`, `custrecord_rdm_gap` | **Manual, no structured source** | She owns these outright (70-73% share); nothing in Smartsheet supplies them |
| `upper_tolerance` / `lower_tolerance` | **Manual, overlaps with automation** | Also written by the `rdm_recon_mr` automated recon job (owned by Ray Zhu) — unconfirmed whether Ashley is correcting recon's output or filling gaps recon leaves |

### Christine Werner — 1,411 products touched

| Field | Manual/Automated | Why |
|---|---|---|
| `metaonly_status`, `custrecord_shipping_category` | **Neither — logging artifacts** | ~~Both 100% human-set, both exactly 1,236 changes~~ — **corrected 2026-08-13.** Every one of those 1,236 entries, on both fields, is identical: `None -> {'id': '{}', 'name': 'Unknown'}`. An empty id with the name "Unknown" is a form placeholder, not a value anyone chose. Genuine edits: **zero**. The matching counts were the tell. Christine does not set these; nobody does. Worth reporting to Cherian as a change-log bug |
| Interlink set corrections | **Manual, judgment call** | Last-modified 156 of Cherian's sets — whatever she corrects is exactly where his implicit rule breaks down in practice |
| Women's CYO category | **Disproportionately manual** | Median 30 field-changes across 17 sittings per product, vs. 3-6 edits / 2-5 sittings for every other category. Best pilot candidate for automation — most to gain |

### Mandy DeBoer — 505 products touched

| Field | Manual/Automated | Why |
|---|---|---|
| `engravable`, `personalize` | **Per-product decision, NOT re-keying** | ~~100% populated on bands/pendants/bracelets~~ — **corrected 2026-08-13.** That figure counted a `False` checkbox as populated. Counting only true values: `engravable` is 59% on womens bands, 89% on mens bands, 43% on fashion rings, 4% on pendants, 2% on bracelets. It genuinely varies within a category, so it is a real decision and not a default being re-typed. `personalize` is true on 0-4% everywhere. **Mandy confirmed directly: "Engravable and Personalize should not be 100% populated."** This is not the cheap automation win it appeared to be |
| `udf_type_of_chain`, necklace/earring style + type filter tags | **Manual, no structured source** | No Smartsheet source exists for any of these |
| `custrecord_rdm_image_status` | **Mostly automated, manually overridden** | RESTlet (automated) writes this 710 times vs. her 89 manual edits — unconfirmed what she's correcting when she steps in |

### Minor / occasional editors

- **Ryan O'Connor** (118 products) — touches `match_sets`, `shipping_category`. Unconfirmed whether this is an approval step or filling gaps.
- **Christina Lewis** (3 products) — occasional, likely photo-related.

---

## 3. Confirmed automated systems (not manual, verified with evidence)

| System | Evidence | Notes |
|---|---|---|
| Workato → RDM writes | RDM Audit Log shows edits attributed to "Workato Integration" via RESTLET/WEBSERVICES API, as recent as Aug 1, 2026 | Proves a working API access pattern already exists into RDM — use as leverage for our own access request |
| RDM ⇄ Shopify sync (Product ID, Handle writeback) | Confirmed via BE101's System tab: "RDM record updated with Shopify ids. RDM SYNC status updated to COMPLETED" | May be scoped to already-active products specifically; Christine described new-launch sync as "not truly used in today current state" — possible these are two different scenarios, not yet reconciled |
| RDM's built-in GAP/melee calculator | Seen directly in BE101's Attributes tab | We previously assumed this was an external tool — it's native to RDM. Connect Data doesn't need to replicate this math |
| RDM's built-in Image Validation dashboard | Seen directly in BE101's Images tab, tracks per shape/metal/size image load status | This IS the "imagery ready" signal — no need to build separate detection |
| `rdm_recon_mr` (NetSuite script, owned by Ray Zhu) | Confirmed to source 9 fields into RDM automatically | Owns 19 of 21 total RDM scripts. If Connect Data writes a field this job also writes, conflict resolution is unconfirmed |
| ~90 Smartsheet automation rules | **Rule list NOT obtainable via API** — corrected 2026-08-13. The `automationrules` endpoint returns only rules the caller may administer, and Mickey Avila owns them, so it returns 0 to us. If a full list was reviewed, it came from another route and should be attached here | Reconstructed 27 cascades empirically instead, from cell history: columns changing at the *same second* on the same row indicate a rule firing. Dominant rules found: `Copy Complete?` + `Naming Complete?` (38 co-changes), `Anticipated Launch Date` + `Launch Notes` (34), `Photograph` + `Photography Status` (26). Only ~16% of Smartsheet activity is automation; the other 84% is manual single-column entry. Render pipeline order per trigger names: CAD Approved → Assigned to Contractor → 1S/1S Pending Approval → 1S/1S Approved → Adam Review → Final Post Production → Christina Review → GG Review → Clipping Panda → Uploaded to Web |

---

## 4. Explicitly out of scope for this project

- Writing to Oscar directly (Aragon's separate RDM→Oscar integration owns this)
- Gemstone/loose diamond products
- Diamond inventory sync (already automated via Snowflake → Workato — different project)

---

## 5. Open questions — not yet resolved, don't state these as fact

1. What defines two products as the same "Style" for interlinking? (Cherian, Christine, Merch — Christine confirmed it is "a judgement call, curated by the Ecomm and Merch teams", so the question is whether any repeatable rule exists at all)
2. ~~Are `engravable`/`personalize` real per-product decisions or category defaults?~~ **ANSWERED 2026-08-13** — real per-product decisions. Confirmed by Mandy and by recounting with `False` excluded.
3. Is `push_schedule` a batch operation happening one record at a time? (Ashley)
4. ~~Are "Pricing complete" / "Merch Completed" dead fields?~~ **PARTLY ANSWERED 2026-08-13** — `Merch Completed` is unused; `Merch Complete` and `Pricing complete` are live but maintained on the Merch Specs sheet. Open part: should the unused duplicates be retired, and which column is authoritative? (Christine, Mandy)
5. Does the RDM⇄Shopify Workato sync apply to new launches, or only already-active products?
6. Is "Add Row to Merch Spec Sheet" the same as the "Merch Smartsheet (badges, LP recs)" we mapped, or a different sheet?
7. If Connect Data and `rdm_recon_mr` both write the same field, which wins?
8. What does setting `custrecord_rdm_status` actually trigger downstream?

---

## 6. How to use this doc when an engineer asks a pointed question

**"What's the source for field X?"** → check the table row. If it says "no
structured source," say that plainly — don't imply a source exists if one doesn't.

**"Why is this manual instead of automated?"** → use the Why column's category
(judgment call vs. re-keying vs. no source vs. cross-team handoff). Re-keying
fields are your strongest automation candidates; judgment-call fields need a
human-in-the-loop design, not full automation.

**"Are you sure about that?"** → check if the row says "unconfirmed" or
"possibly" — if so, say that directly rather than defending it as settled fact.
