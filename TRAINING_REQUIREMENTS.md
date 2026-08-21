# BE ORE — Category × Rec Type Training Requirements

> Reference doc for what each recommendation type needs per product category:
> which RDM fields drive it, what the hard gates are, what signals score it,
> and what failure modes to guard against.
>
> **RDM field availability reality check (active products):**
> - `complementary_items` / `match_sets` — rings only (33–34%), zero for all other categories
> - `average_width` — rings only (100%), zero elsewhere
> - `component` — 65–88% across all categories (best gem/metal source)
> - `collection` — 98–100% everywhere (reliable)
> - `interlinking` — 52–70% everywhere (variant family grouping)
> - `exclude_from_stacking` — rings only, sparse (2%) but high-signal when present
> - `gem_not_apply` — 9–33% (flags plain metal / no-gem products)

---

## How to read this doc

Each section is **Category → Rec Type**. Under each:

- **Primary RDM signals** — fields that feed the DNA profile or scoring directly
- **Hard gates** — boolean must/must-not rules; a candidate that fails is excluded before scoring
- **Soft signals** — fields that nudge scores up or down but don't exclude
- **Training shape** — what a good positive example looks like
- **Failure modes** — known wrong-rec patterns this rec type must not produce

---

## RINGS

### Rings → Similar Items

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_component` | RDM | Exact gem type, shape, color, clarity, setting, ct. wt. — most accurate DNA source |
| `custrecord_rdm_collection` | RDM | Collection membership — rings in the same collection are strong similar candidates |
| `custrecord_rdm_interlinking` | RDM | Defines the design family (same base design, different metals/styles) — must be treated as variants, not recs |
| `udf_mh_setting_style_filter_tag` | RDM UDF | Site filter value for setting style — directly maps to shopper-visible filters |
| `udf_mh_setting_type_filter_tag` | RDM UDF | Site filter value for setting type (solitaire, halo, pave, etc.) |
| `custrecord_rdm_average_width` | RDM | Band width in mm — similar width = more similar ring |
| `custrecord_rdm_gem_not_apply` | RDM | True = plain metal ring — must only be similar to other plain metal rings |
| `custrecord_rdm_marketing_product_type` | RDM | Leaf subtype (Solitaire, Three Stone, Eternity, Signet, etc.) — hard subtype gate |
| DNA: `gem_family` | Profile | Lab vs natural, gem species |
| DNA: `ring_style` / `ring_intent` | Profile | Engagement / wedding / fashion / signet intent |
| DNA: `aesthetic_tier` | Profile | Luxury register alignment |
| Visual: `visual_era`, `design_character` | Knowledge card | Style epoch and feel — prevents matching vintage to modern |

**Hard gates**
- `ring_intent = wedding_band` → never recommend to `ring_intent = engagement` (and vice versa)
- `gem_not_apply = T` (plain metal) → only recommend other plain metal rings
- `ring_style = signet` → only recommend other signets
- Products in same `interlinking` group → treat as variants, collapse to one pill, do not show as separate recs
- Same `base_sku` (strip metal suffix) → deduplicate, never show as a rec

**Soft signals**
- Same `collection` → +0.15 boost
- Same `udf_mh_setting_type_filter_tag` → +0.12
- Same `average_width` band (±0.5mm) → +0.08
- Same `component` gem shape → +0.10
- Same `visual_era` → +0.10
- Same `customer_archetype_visual` → +0.08

**Training shape (positive example)**
Anchor: BE1D1234 — Pavé engagement ring, round diamond, scalloped pave, 14KW, modern era, romantic archetype
Rec: BE1D5916 — Similar pavé engagement ring, same metal family, same setting type, same visual era
Label: `similar=1, confidence=0.87`

**Failure modes to guard against**
- Zodiac signets recommended for gemstone engagement rings (style gate failure)
- Diamond eternity bands recommended as similar to solitaire engagement rings (intent mismatch)
- Same design in different metal showing as two separate recs instead of one variant pill

---

### Rings → Pairs With

Two subtypes — engine must determine which applies based on anchor ring intent:

**Subtype A: Stack Together** (anchor is engagement/fashion ring, candidate is wedding band)
**Subtype B: Wear Together** (both fashion rings worn on different fingers)

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_complementary_items` | RDM | Merchandiser-curated pairings — treat as ground truth, highest signal |
| `custrecord_rdm_match_sets` | RDM | Linked match sets — `merchandised_set: true` = explicitly intended pairing |
| `custrecord_rdm_exclude_from_stacking` | RDM | `T` = hard exclusion from stacking recs |
| `custrecord_rdm_average_width` | RDM | Band width — wide bands (>3mm) should not stack with solitaires that specify narrow-only |
| `custrecord_rdm_component` | RDM | Gem/setting compatibility check — avoid mismatched stone quality levels |
| DNA: `band_pairing_requirements.avoid` | Profile | Visual enrichment avoid list — wide bands, yellow gold if white anchor, etc. |
| DNA: `ring_intent` | Profile | Determines subtype A vs B |
| DNA: `metal_family` | Profile | Metal must match (white/yellow/rose) unless anchor explicitly allows mixed metal |

**Hard gates**
- `exclude_from_stacking = T` → never show in Pairs With
- Metal mismatch (white vs yellow vs rose) → exclude unless anchor has `jset_mix_metal = T`
- `ring_intent = wedding_band` anchor → Stack Together only shows other bands (not engagement rings)
- `ring_intent = engagement` anchor → Stack Together shows wedding bands only; Wear Together shows fashion rings only
- Same `interlinking` group as anchor → exclude (variant, not a pairing)
- Same `base_sku` → exclude

**Soft signals**
- `complementary_items` match → score override to 0.90+ (merchandiser ground truth)
- `match_sets` with `merchandised_set: true` → score override to 0.85+
- Same `collection` as anchor → +0.20
- `average_width` compatible with anchor avoid list → +0.10
- Same `component` gem quality tier → +0.08

**Training shape (positive example)**
Anchor: BE101 — 2mm solitaire engagement ring, 14KW
Rec: BE201 — Matching slim wedding band, 14KW (appears in anchor's `complementary_items`)
Pairing type: `Stack Together`
Label: `pairs=1, type=stack_together, confidence=0.92, rdm_source=complementary_items`

**Failure modes to guard against**
- Diamond engagement rings pairing with plain metal bands of wrong width
- Wedding band anchor recommending engagement rings back
- Same design in different metal showing as a pairing instead of a variant

---

### Rings → Complete the Look

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_collection` | RDM | Cross-category collection match — if anchor earrings are in same collection, boost strongly |
| `custrecord_rdm_component` | RDM | Gem type/quality → informs what necklace/earring gem tier to match |
| DNA: `metal_family` | Profile | Metal match across categories |
| DNA: `occasion_range` | Profile | Occasion gate — everyday ring → everyday earrings, not cocktail |
| Visual: `luxury_register`, `identity_signal` | Knowledge card | Aesthetic alignment across categories |

**Hard gates**
- Metal mismatch → exclude (white gold ring → no yellow gold earrings unless `versatile` occasion)
- Occasion gap ≥ 2 tiers (everyday ↔ black tie) → exclude

**Soft signals**
- Same `collection` across categories → +0.25 (strongest cross-category signal)
- Same `luxury_register` → +0.15
- Same `visual_era` → +0.12
- Same `identity_signal` → +0.10

**Failure modes**
- Bold statement earrings paired with minimalist solitaire ring
- Mismatched metal across the look
- Occasion mismatch (casual ring + formal necklace)

---

### Rings → Search / Chatbot

**All RDM fields in scope.** The chatbot must be able to answer any product question from:

| Data source | Fields |
|---|---|
| RDM component | Exact gem specs: type, shape, color, clarity, ct. wt., setting type, quantity |
| RDM product name/desc | Natural language product description |
| RDM MPT | Full taxonomy hierarchy for category navigation |
| RDM collection | Collection context and story |
| RDM `udf_mh_default_shape/metal` | Default configuration shown on site |
| DNA profile | All 40+ structured attributes |
| Knowledge card | Visual synthesis, design character, occasion, identity signal |

**Training shape**
Query: "Show me a vintage-inspired engagement ring with a cushion diamond under $5,000"
Required fields to answer: `component.shape`, `visual_era`, `price`, `ring_intent`
Good response: surfaces rings where `component.shape = Cushion`, `visual_era = Vintage/Art Deco`, `price ≤ 5000`

**Failure modes**
- Recommending CYO settings as if they are complete rings (no center stone)
- Confusing lab vs natural origin when customer specifies
- Mixing ring intents in search results (wedding bands in engagement results)

---

## NECKLACES

> **Note:** Necklaces is a new category being split from pendants. Unified profiles don't exist yet — enrichment is Priority 1. Training requirements defined here drive what the enrichment pipeline should extract.

### Necklaces → Similar Items

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_marketing_product_type` | RDM | Leaf subtype is critical: Chain / Pendant / Tennis / Medallion / Station / Collar |
| `custrecord_rdm_component` | RDM | Gem type, chain style, ct. wt. |
| `custrecord_rdm_interlinking` | RDM | Design family (same pendant different chain, same chain different length) |
| `custrecord_rdm_collection` | RDM | Collection membership |
| `custrecord_rdm_gem_not_apply` | RDM | Plain chain — must only match other plain chains |
| DNA: `necklace_subtype` | Profile (to build) | Chain / Pendant / Tennis / Medallion — primary gate |
| DNA: `chain_style` | Profile (to build) | Cable / Paperclip / Rope / Cuban / Bead Station |
| DNA: `length_inches` | Profile (to build) | 16 / 18 / 20 / 24 — similar lengths preferred |
| Visual: `luxury_register`, `design_character` | Knowledge card | Aesthetic alignment |

**Hard gates**
- `necklace_subtype = chain` → never recommend pendant-style necklaces (and vice versa)
- `necklace_subtype = tennis` → only recommend other tennis necklaces
- `gem_not_apply = T` → plain chain only matches other plain chains
- Same `interlinking` group → collapse to variants

**Soft signals**
- Same `chain_style` → +0.15
- Same `length_inches` (±2 inches) → +0.10
- Same `collection` → +0.15
- Same `component` gem type → +0.12

**Failure modes**
- Plain cable chains recommended as similar to diamond pendant necklaces
- Tennis necklace recommended as similar to a layering chain
- Letter/initial charms recommended as similar to solitaire pendants

---

### Necklaces → Pairs With

**Logic:** Necklaces pair with earrings. The pairing type is always **Coordinate**.

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_component` | RDM | Gem type match to earring gems |
| `custrecord_rdm_collection` | RDM | Same collection earrings are best pairing |
| DNA: `necklace_subtype` | Profile | Determines what earring styles are compatible |
| DNA: `metal_family` | Profile | Metal match |
| Visual: `design_character` | Knowledge card | Delicate necklace → delicate earrings, statement → statement |

**Hard gates**
- Statement pendant + statement earrings → exclude (too much visual competition)
- Metal mismatch → exclude
- Earring category only — no rings, no bracelets in Pairs With for necklaces

**Soft signals**
- Same `collection` as anchor → +0.25
- Same `component` gem → +0.15
- Complementary scale (delicate pendant + small stud, chunky chain + hoop) → +0.10

**Failure modes**
- Showing other necklaces in Pairs With (same category pairing makes no sense)
- Diamond tennis necklace paired with oversized statement earrings

---

### Necklaces → Complete the Look

Target categories: rings, earrings, bracelets (not other necklaces)

**Hard gates**
- Metal mismatch → exclude
- Occasion gap ≥ 2 → exclude

**Soft signals**
- Same `collection` → +0.20
- Same `luxury_register` → +0.15
- Complementary earring scale (pendant necklace → small earrings, not competing) → +0.10

---

### Necklaces → Search / Chatbot

**Key fields the chatbot must use:**
- `component`: chain material, gem type/shape/ct.wt.
- MPT leaf: Chain style (Paperclip / Cuban / Rope / Cable / Bead Station)
- `length` from product name parsing (16in / 18in / 20in)
- `interlinking` for "same necklace different length" queries
- `collection` for "show me the full Fairmined collection"

**Failure modes**
- Treating CYO pendant settings as complete necklaces
- Confusing chain length variants as different products
- Missing plain gold chain requests because gem fields are empty

---

## BRACELETS

### Bracelets → Similar Items

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_marketing_product_type` | RDM | Leaf subtype: Tennis / Bangle / Chain / Cuff / Station / Pearl |
| `custrecord_rdm_component` | RDM | Gem type, ct. wt., setting — tennis bracelet with pave vs bezel are different |
| `custrecord_rdm_interlinking` | RDM | Same bracelet, different metal or length |
| `custrecord_rdm_collection` | RDM | Collection membership |
| `custrecord_rdm_gem_not_apply` | RDM | Plain metal / chain only |
| DNA: `bracelet_style` | Profile | Tennis / Bangle / Chain / Cuff — primary gate |
| DNA: `stackable` | Profile | Determines whether it's a wrist-stack candidate |
| Visual: `design_character` | Knowledge card | Delicate vs bold |

**Hard gates**
- `bracelet_style = tennis` → only recommend other tennis bracelets in Similar
- `bracelet_style = pearl` → only recommend other pearl bracelets
- Same `interlinking` group → variants, not recs

**Soft signals**
- Same `collection` → +0.15
- Same `component` gem type → +0.12
- Same clasp type (from component) → +0.08

**Failure modes**
- Pearl bracelet recommended as similar to diamond tennis bracelet
- Watch (collab item) recommended as similar to a chain bracelet

---

### Bracelets → Pairs With

**Logic:** Bracelets pair with other bracelets for wrist stacking — **contrasting styles only**.

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_exclude_from_stacking` | RDM | Hard exclusion |
| `custrecord_rdm_component` | RDM | Width/thickness estimate for stack balance |
| DNA: `bracelet_style` | Profile | Determines which contrasting styles are valid |

**Contrasting style logic**
| Anchor | Valid stack partners |
|---|---|
| Tennis | Bangle, chain, cuff — NOT another tennis |
| Bangle | Tennis, chain — NOT another bangle |
| Chain | Tennis, bangle, cuff |
| Cuff | Chain, bangle — NOT another cuff |
| Pearl | Chain only — statement pearl + nothing else bold |

**Hard gates**
- `exclude_from_stacking = T` → never appear in Pairs With
- Same style + same category → exclude (no tennis + tennis)
- Metal mismatch → exclude (unless layered mixed metal is the explicit look)

**Failure modes**
- Two tennis bracelets shown as a stack (visual overkill)
- Pearl bracelet paired with bold cuff

---

### Bracelets → Complete the Look

Target: rings, earrings, necklaces

**Hard gates:** Metal mismatch, occasion gap ≥ 2

**Soft signals**
- Same `collection` → +0.20
- Same `luxury_register` → +0.15
- Complementary scale to ring anchor → +0.10

---

### Bracelets → Search / Chatbot

**Key fields:**
- `component`: gem type, ct. wt., clasp, setting
- MPT leaf: Tennis / Bangle / Chain / Station / Cuff / Pearl
- `interlinking`: length variants (6in / 6.5in / 7in / 7.5in)
- `collection`: "show me the Malibu Blue collection"

**Failure modes**
- Confusing length variants as separate products
- Treating watches as bracelets in search results (watch = separate subcategory)

---

## EARRINGS

### Earrings → Similar Items

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_marketing_product_type` | RDM | Leaf subtype: Stud / Hoop / Drop / Huggie / Climber / Jacket |
| `custrecord_rdm_component` | RDM | Gem type, shape, ct. wt., setting — studs can be bezel vs prong |
| `custrecord_rdm_interlinking` | RDM | Same earring different metal |
| `custrecord_rdm_collection` | RDM | Collection membership |
| `custrecord_rdm_gem_not_apply` | RDM | Plain metal hoops / huggies |
| DNA: `earring_style` | Profile | Stud / Hoop / Drop / Huggie — primary gate |
| DNA: `backing_type` | Profile | Post / Lever-back / Threader — lifestyle signal |
| Visual: `design_character` | Knowledge card | Delicate vs statement |

**Hard gates**
- `earring_style = stud` → only recommend other studs in Similar
- `earring_style = hoop` → only recommend hoops
- `gem_not_apply = T` → plain metal only
- Same `interlinking` group → variants

**Soft signals**
- Same `component` gem shape → +0.15
- Same `component` ct. wt. tier → +0.10
- Same `collection` → +0.15
- Same `design_character` → +0.10

**Failure modes**
- Diamond studs (6 ct. tw.) recommended as similar to small diamond huggies (wrong scale)
- Huggie hoops recommended as similar to large statement hoops

---

### Earrings → Pairs With

**Logic:** Earrings pair with necklaces/pendants only. Never with other earrings.

**Primary RDM signals**
| Field | Source | Role |
|---|---|---|
| `custrecord_rdm_component` | RDM | Gem type to match with pendant |
| `custrecord_rdm_collection` | RDM | Same collection necklace/pendant |
| DNA: `earring_style` | Profile | Statement vs delicate — determines necklace scale |
| DNA: `metal_family` | Profile | Metal match |

**Hard gates**
- Target category must be necklaces or pendants — never other earrings
- Statement earring + statement necklace → exclude (too competitive)
- Metal mismatch → exclude

**Soft signals**
- Same `collection` → +0.25
- Same `component` gem → +0.15
- Complementary scale (statement earring → simple chain, delicate stud → pendant necklace) → +0.12

**Pairing type label:** `Coordinate Necklace`

**Failure modes**
- Showing other earrings in Pairs With (confirmed wrong — people don't wear two pairs)
- Diamond stud earrings paired with chunky statement necklace (scale mismatch)

---

### Earrings → Complete the Look

Target: rings, necklaces, bracelets

**Hard gates:** Metal mismatch, occasion gap ≥ 2

**Soft signals**
- Same `collection` across categories → +0.20
- Same `luxury_register` → +0.15
- Complementary scale (statement earrings → simpler ring, not competing) → +0.10

---

### Earrings → Search / Chatbot

**Key fields:**
- `component`: gem type, shape, ct. wt., setting, quantity (stud pairs vs singles)
- MPT leaf: Stud / Hoop / Drop / Huggie / Climber
- `interlinking`: metal variants
- `collection`: collection queries

**Failure modes**
- Recommending a single earring (odd item) instead of a pair
- Confusing huggie hoops with small hoops in size queries

---

## CROSS-CUTTING TRAINING RULES

These apply to all categories and rec types:

### Variant deduplication (all rec types)
1. Products sharing the same `interlinking` group = same design, different options → collapse to one entry with metal/style pill selector
2. Products sharing the same `base_sku` (strip metal suffix) → always collapse
3. Only show one design per `interlink_group` ID — pick highest-scoring variant as the representative

### RDM merchandiser signals as ground truth
When `complementary_items` or `match_sets` (with `merchandised_set: true`) link two products:
- That pairing should score ≥ 0.85 in Pairs With regardless of visual similarity
- These are explicit human-curated pairings and override algorithmic scoring
- Currently only rings have this data — treat as labeled training pairs

### Component-driven DNA accuracy
When `custrecord_rdm_component` is available (65–88% of products), it supersedes any inferred gem/metal values in the DNA profile. Component data is the authoritative source for:
- Gem type and origin (lab vs natural)
- Gem shape and ct. wt.
- Setting type
- Metal (from component rows, not just product name)

### Collection signal
`custrecord_rdm_collection` (98–100% populated) is the strongest cross-product grouping signal available. Same collection = designed to be worn together. Boost across all rec types when anchor and candidate share a collection.

### Occasion integrity
`occasion_range` from the knowledge card gates Complete the Look across all categories. The tiers are:
- `everyday` → `versatile` → `occasion` → `special_occasion`
A gap of ≥ 2 tiers is a hard exclusion. A gap of 1 is a soft penalty (-0.15).

---

## WHAT'S MISSING UNTIL PROD RDM ACCESS

| Signal | Current state | Impact |
|---|---|---|
| `complementary_items` for necklaces/earrings/bracelets | 0% — not in sandbox | No merchandiser ground truth for Pairs With outside rings |
| `match_sets` for non-ring categories | 0% | Same gap |
| Certified diamond flag | 0% in sandbox | Can't distinguish certified vs non-certified in recs |
| `jewelry_set` linking | 0% | Can't surface explicitly linked jewelry sets |
| Full variant pricing | Sandbox only | Can't use price tier in scoring reliably |

Once prod RDM access is confirmed, the first pull should focus on these five fields across all categories.
