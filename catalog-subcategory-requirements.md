# Org-Specific Subcategory Display Names — Product Requirements

**Status:** In progress
**Clients driving initial need:** Story, Salve
**Mockup:** https://hopele.github.io/hopesmockups/manage-subcategory-mockup.html

---

## Background

Treez maintains a global set of canonical categories and subcategories. These are Treez-owned values that drive purchase limits, tax rules, compliance reporting (Metrc), and Sell Treez subtype resolution. They cannot be freely renamed or removed without downstream consequences.

Dispensaries increasingly need catalog names that reflect how they merchandise — not how Treez internally classifies products. A dispensary selling "Rosin" doesn't want their product dropdown to say "Concentrate — Rosin"; they want "Rosin" or whatever their brand language calls it. Today there is no mechanism to do this without breaking compliance logic.

This feature introduces org-specific custom subcategories: display aliases that sit on top of the global canonical layer. Products retain their canonical subcategory assignment (and all the compliance metadata that comes with it), while the org sees the name they've chosen.

---

## Core Concepts

### Global Canonical Subcategories
Treez-owned subcategories. These drive:
- Purchase limits
- Tax rule application
- Metrc compliance reporting
- Sell Treez subtype resolution

Global subcategories cannot be deleted or renamed. They are seeded and maintained by Treez. Orgs can only control whether a global subcategory is included or excluded from the product create/edit dropdown.

### Org-Specific Custom Subcategories
A custom subcategory is a display alias created by an org, mapped 1:1 to a global canonical subcategory. Key rules:
- Every custom subcategory points to exactly one canonical parent.
- One canonical can have many custom children (one-to-many from canonical's side).
- A custom subcategory inherits all compliance metadata from its canonical parent — purchase limits, tax classification, Metrc type — without the org needing to configure anything.
- The canonical `productSubCategoryId` remains the source of truth on the product record. A nullable `customSubCategoryId` field is added alongside it.

### Include / Exclude
Orgs control which global subcategories are active for their catalog on a per-category basis. An excluded subcategory:
- Does not appear in the product editing dropdown for that org.
- Does not appear as a rule option in the automated collections builder for that org.
- Does not prevent products already assigned to it from existing — existing assignments are not auto-migrated in MVP (reassignment is a fast follow).

---

## MVP Scope

### 1. Manage > Subcategories Page

The starting point is a category selector. After choosing a category (e.g., Concentrate), the page shows two panels.

**Include / Exclude**
- All global subcategories for the selected category are listed, split into Excluded and Included columns. Clicking a subcategory moves it between columns.
- Excluding is always allowed — no block based on product count.
- Excluded subcategories are removed from the product create/edit dropdown and do not appear as subcategory filter options in the POS product menu. Existing product assignments are unaffected.
- Reassigning products off an excluded subcategory is available in the fast follow via the "Reassign Products" kebab option.
- The Global badge distinguishes canonical subcategories from org-created customs.

**Included Subcategories panel**
- Lists all included global subcategories. Each global row can have zero or more custom subcategories nested under it.
- Global rows have no delete or rename action — globals cannot be deleted or renamed by orgs.

**Custom subcategories**
- When at least one custom exists under a global, the global is hidden from the product create/edit dropdown — new products must be assigned to one of the customs. Existing products on the global remain there until explicitly reassigned (fast follow). The global still appears in the UI with a "Not shown in product dropdown" indicator — it remains the compliance anchor.
- If no customs exist under a global, the global name appears directly in the product dropdown.
- **Add:** Inline form under the global row. Name is required. Custom is added immediately — no prompts or modals in MVP.
- **Delete:** Only available when zero products are assigned. If products are assigned, Delete does not appear in the kebab.

### 2. Catalog — Product List & Product Card

**Product list grid**
- The subcategory column shows the custom subcategory name when assigned to a custom, otherwise the global name.

**Product card — create / edit**
- The subcategory dropdown shows custom subcategory names where they exist under an included global, otherwise the global name.

**Product list filters**
- Subcategory filter options in the catalog product list continue to display all subcategories (global and custom) in MVP — they are not yet restricted to the org's included set.
- Filtering by included/excluded state will be revisited after the reassignment flow is in place, once it is safe to hide options without stranding products in unfiltered states.

### 3. Sell Treez

Custom subcategory display names surface in three ST areas at MVP. Where no custom exists, the global name is used.
**POS product menu — subcategory filter**
- The subcategory filter shows custom subcategory names for orgs that have them.

**Receipts**
- All receipt types (print, email, SMS) display the custom subcategory display name where subcategory is shown.

**Product Groups (legacy ST discount management)**
- The subcategory filter shows custom subcategory names. Rules can be built against any active subcategory.

### 4. Collections

Collection rules store subcategory UUIDs — custom or canonical — depending on what the operator targets. Products can be in four assignment states: on an included global with no customs (standard case), on a custom subcategory, on an included global whose customs exist but the product hasn't been reassigned yet, or on an excluded global (exclusion does not migrate assignments). Operators may need to select both a global and its customs to build a complete collection rule.

**No rule impact from include/exclude or custom subcategory creation**
- Excluding, including, or creating a custom subcategory does not affect existing collection rules. Rules remain syntactically valid and collection membership is unchanged.
- Deleting a custom subcategory with zero products assigned leaves any referencing rule entry inert (the deleted UUID returns no products) but does not break the collection. Users can clean up stale rule entries manually.

**API reassignment changes collection membership**
- Product reassignment is available via the catalog API in MVP. Reassigning a product changes its subcategory ID, which directly changes which automated collections it belongs to.
- Collection rules are **not** auto-rewritten in MVP. Any collection rule referencing the source subcategory UUID will silently lose those products after reassignment. Auto-rewrite of collection rules ships as fast follow.
- Orgs using the API reassignment path in MVP should audit their automated collections manually after reassigning.

**Collection rule builder**
- The rule builder must expose custom subcategory names, included globals (even those with customs, since products may still be assigned there), and excluded globals (since products may still be assigned there too). Showing the full option set with custom names ships as fast follow.

---

## Fast Follow

The following capabilities ship after MVP, ordered by priority. Items 1–3 are tightly coupled — rename and reassignment share the same PMRS fan-out infrastructure and the collection warning/auto-rewrite depend on the same reassignment event. They are scoped and built together.

**1. Rename + Reassignment fan-out** _(highest priority — coupled infrastructure)_

Rename and reassignment both use the same fan-out: PMS queries all affected products in the org and emits a `ProductCore v1x3x0` event per product with updated subcategory fields; PMRS updates `subtype` per product via existing event processing.

_Rename (Edit Name):_
- Edit Name added to the custom subcategory kebab menu.
- Name change only — canonical mapping does not change, no products move.
- On save: PMS fans out `ProductCore v1x3x0` events for all products with that `customSubCategoryId`, updating `customSubCategoryDisplayName`. PMRS updates `subtype` per product. ST (POS menu, reporting, filter options) reflects the new name after fan-out completes.
- Rename does not ship without the fan-out — ST showing the old name is not acceptable.

_Reassign Products (custom subcategory kebab):_
- "Reassign Products" added to the custom subcategory kebab when products are assigned (Delete remains hidden in this state).
- Title: "Reassign [Custom Subcategory Name]"
- Body: "{n} products will be reassigned and the subcategory will be deleted."
- "This action cannot be undone." warning shown in red.
- Confirm disabled until a target is selected. Reassignment cannot be skipped.
- Dropdown lists valid targets: other custom subcategories under included globals, or globals with no customs. Source excluded from the target list.
- On confirm: products move to target, source custom subcategory is deleted. Does not remain as an empty shell.
- Success toast: "[Name] deleted. {n} products reassigned to "[target]"."

**2. Collection count warning in reassign modal** _(ships with #1 — same modal surface)_
- When the reassign modal opens, a count of automated collections referencing the source subcategory is fetched from the product-collections service.
- If n > 0, the modal shows: "{n} automated collection(s) reference this subcategory and will be updated to the new subcategory."
- If the fetch fails or is still loading, the modal remains fully functional — the action is not blocked.

**3. Auto-rewrite collection rules on reassignment** _(ships with or immediately after #1 — consumes the same event)_
- When reassignment is confirmed, PMS emits a `SUBCATEGORY_REASSIGNED` event. The product-collections service finds all collections in the org whose `subCategory` rule array contains `fromSubCategoryId` and replaces it with `toSubCategoryId` using set semantics.
- Idempotent. Republishes a Collection Updated event for each rewritten collection.
- Standalone excludes or deletes without a replacement target do NOT trigger auto-rewrite — those collections are left for the user to clean up manually.
- See Collections Impact section for full detail.

**4. First-custom reassign prompt** _(small addition once reassignment exists)_
- When creating the **first** custom subcategory under a global that has products assigned, a prompt offers to reassign those products to the new custom now.
- Applies only on first custom creation — subsequent customs don't change existing assignment state.
- Skipping is allowed; products can be reassigned later via the "Reassign Products" kebab option.

**5. Collections MFE — filter rule options to org-included subcategories** _(independent work, important for collection accuracy)_
- Today the collection rule builder fetches all global subcategories. Post include/exclude, it must fetch only the org's included subcategories (and their custom subcategories) as rule options.
- Prevents users from accidentally building rules on excluded subcategories.

**6. Import service — accept custom subcategory by name or ID** _(self-service path; API available in MVP)_
- File-based bulk imports will accept custom subcategory display name or ID — non-technical self-service path.
- Bulk re-tagging via the catalog API is available in MVP; import adds the self-service path for non-technical users.

**7. Catalog-to-ST reconciliation — seed & migrate** _(separate work stream, gated on client comms)_
- Class 1 canonical renames, Class 2 seeded default customs, Class 3 missing canonicals. See Catalog Reconciliation section for class definitions.
- Requires client communication before running — orgs need to know product subcategory assignments may change.
- PMRS hardcoded type map removed after seed + migrate is verified.

---

## Collections Impact

### How Collection Rules Store Subcategories

Automated collections store filter rules as a JSONB object on the collection record. The subcategory filter uses the key `subCategory` and stores an array of subcategory UUIDs — either custom subcategory UUIDs or canonical subcategory UUIDs, depending on what the operator selects.

```json
{
  "category": ["<categoryId>"],
  "subCategory": ["<customSubCategoryId-A>", "<canonicalSubCategoryId-B>"]
}
```

Product assignment is not always clean-cut. Products can be assigned to:
- An **included global canonical UUID (no customs)** — the standard case; the global is the only option and products are assigned directly to it
- A **custom subcategory UUID** — when the org has customs and the product was explicitly assigned to one
- An **included global canonical UUID (customs exist)** — the global is hidden from the product dropdown but products assigned before customs were created remain there until explicitly reassigned
- An **excluded global canonical UUID** — exclusion does not migrate existing assignments

Because of this, the rule builder must expose all three: custom subcategory options, included globals with customs (since products may still live there), and excluded globals (since products may still be assigned there too). An operator targeting all products in a given subcategory family may need to select both the global and its customs to get full coverage.

Because rules store IDs (not display names), renaming a custom subcategory has no effect on rule validity. Reassigning products (which changes their `customSubCategoryId`) does change which products match a collection's rule.

### Impact Matrix

| Operation | Collection rule breaks? | Collection membership changes? | Action required |
|---|---|---|---|
| Rename custom subcategory display name | No | No | Collection rules unaffected (custom UUID unchanged). PMRS must fan out the new name to assigned products so ST stays in sync — rename is therefore a fast follow. |
| Delete custom subcategory (0 products) | Rule references a stale UUID | No — 0 products were assigned, so collection membership was already empty for that entry | Rule entry becomes inert until manually edited. Safe to allow in MVP since membership impact is zero. |
| Exclude a global subcategory | No — rule remains syntactically valid | No for existing products | Excluded subcategory should not appear as an option in the collection rule builder. Collections MFE must filter rule options to org-included subcategories (fast follow). |
| Include a global subcategory | No | No | Newly included subcategory becomes available in the rule builder. |
| Reassign products from custom A to custom B | No — rule is syntactically valid | Yes — products leave A's membership, enter B's | Rules referencing custom A's UUID lose those products; rules referencing custom B gain them. Auto-rewrite replaces custom A UUID → custom B UUID on reassignment. Show count warning in modal. |

### Reassignment: Auto-Rewrite Collection Rules

When a subcategory reassignment is confirmed, PMS emits a `SUBCATEGORY_REASSIGNED` event. The product-collections service consumes it and rewrites every affected collection's rule:
- Find all collections in the org whose `subCategory` rule array contains `fromSubCategoryId` (custom UUID if applicable, otherwise canonical UUID).
- Replace `fromSubCategoryId` with `toSubCategoryId` using set semantics (dedupe if target already present).
- Republish a Collection Updated event for each modified collection so Sell Treez and the bridge recompute membership.
- Handler is idempotent.

Standalone excludes or deletes without a replacement target do NOT trigger auto-rewrite. Those collections are left stale until the user edits them manually.

**Note:** The collections MFE currently has a hardcoded name-based exclusion list (`EXCLUDED_SUB_CATEGORY_BY_CATEGORY`) that filters certain subcategory names from the rule builder. This is explicitly marked as a temporary workaround for catalog/ST mix issues and can be removed once catalog reconciliation is complete.

### Collections MFE: Filter by Org-Included Subcategories

Today the collection rule builder fetches all global subcategories. Once include/exclude is live, it must fetch only the org's included subcategories (and their custom subcategories) as rule options. This prevents users from accidentally building rules on excluded subcategories.

---

## Technical Debt — Catalog-to-ST Reconciliation

The current catalog and Sell Treez subtype data have diverged over time. Three classes of reconciliation work are needed before the custom subcategory feature is fully clean. Reconciliation does not block the user-defined custom subcategory feature — the PMRS hardcoded map stays as a fallback and keeps working throughout. The **audit is MVP**; the **seed + migrate is fast follow**, gated on client communication.


**Class 1 — Name mismatch (rename in place)**
A subcategory exists in both catalog and ST but the names don't match. Resolution: rename the catalog subcategory to match ST (or the canonical standard). No migration of product assignments needed.

**Class 2 — Catalog-only subcategory**
A subcategory exists in the catalog but has no matching ST subtype. These are legacy values with no compliance anchor. Resolution: seed a default custom subcategory for each org that uses these values. The products remain on the same name but it becomes an org-specific custom backed by the nearest canonical. The seeded default custom subcategory can be deleted by the org if no products are assigned to it.

**Class 3 — ST subtype missing in catalog**
A subtype exists in ST but is absent from the global catalog. Resolution: seed the canonical in the catalog so the full chain is consistent.

**PMRS passthrough**
PMRS currently maintains a hardcoded `typeToCategorySubTypeMap` that maps catalog types to Sell Treez subtypes. After reconciliation, this mapping is eliminated and Sell Treez reads subtype directly from the canonical subcategory ID on the product — a passthrough with no translation layer.

**Client communication required before reconciliation runs**
The Class 2 migration (seeding default customs and re-tagging existing products across all orgs) is the highest blast-radius action in this effort. Client communication is required before it runs — orgs need to know their product subcategory assignments may change. This gates `product-management-service#1570`.

---

## Event Model

**ProductCore v1x3x0** (additive fields on existing product events):
- `customSubCategoryId` — nullable UUID of the org-specific custom subcategory
- `customSubCategoryDisplayName` — the custom display name string
- `customSubCategoryUpdatedAt` — timestamp for ordering/idempotency

**SUBCATEGORY_REASSIGNED** (new PMS event, consumed by product-collections):
- `fromSubCategoryId` — UUID being reassigned from (custom subcategory UUID if one exists, otherwise canonical UUID)
- `toSubCategoryId` — UUID being reassigned to (custom subcategory UUID if one exists, otherwise canonical UUID)
- `organizationId`

**Note on rename:** No dedicated rename event is needed. When a custom subcategory is renamed, PMS fans out `ProductCore v1x3x0` events for all affected products (same mechanism as reassignment). PMRS updates `subtype` per product via existing event processing. Rename and reassignment share this fan-out infrastructure and are built together in the fast follow.

---

## Feature Flag

Feature flag: **`Catalog Management - Custom Subcategory Management`** (managed in organization-management service). Enabled for Story first.

**Gated by the flag (management surface):**
- The Manage > Subcategories page
- Include/exclude org activations
- Org-initiated creation, editing, and deletion of custom subcategories
- Product assignment to a custom subcategory via the UI/API

**Not gated — runs for all orgs (MVP):**
- The `customSubCategory` table and `customSubCategoryId` FK on product (schema is global)
- `ProductCore v1x3x0` additive event fields

**Not gated — runs for all orgs (fast follow, when reconciliation ships):**
- Class 1 canonical renames, Class 2 seeded default customs, Class 3 missing canonical seeds
- PMRS fan-out for display name edits on seeded defaults

Orgs without the flag still get the schema changes and event field updates in MVP. Seeded defaults reach all orgs when the reconciliation fast follow ships — not at MVP launch.

---

## Phasing

### MVP
- Manage > Subcategories page: include/exclude panel + custom subcategory CRUD (behind feature flag)
- Excluding a subcategory is freely allowed — no block, no reassignment required. Existing product assignments are unaffected; the subcategory is simply removed from the product dropdown.
- No delete option in kebab if products assigned to a custom subcategory
- No rename option on custom subcategories (Edit Name is fast follow — requires PMRS fan-out)
- No automatic collection rule rewrites
- Catalog-to-ST reconciliation audit (understand scope of mismatches across all three classes)
- Catalog: Product list grid displays custom subcategory display name
- Catalog: Product card subcategory dropdown shows custom names; excluded subcategories hidden
- Catalog: Product list filters show all subcategories in MVP — scoped to included only after reassign flow ships
- Collections: Include/exclude and custom creation have no effect on collection rules
- Collections: API reassignment changes collection membership — rules are not auto-rewritten until fast follow
- Collections: Rule builder shows all global subcategories in MVP — org-filtered view is fast follow
- ST: POS product menu subcategory filter shows custom names
- ST: All receipts display custom subcategory display name
- ST: Product Groups subcategory filter supports custom names

### Fast Follow
Ordered by priority:
1. **Rename + Reassignment fan-out** — coupled work; same PMRS `ProductCore` fan-out. Rename = Edit Name in kebab. Reassignment = modal (reassign + delete source custom).
2. **Collection count warning** — ships with reassignment, same modal surface.
3. **Auto-rewrite collection rules** — consumes `SUBCATEGORY_REASSIGNED` event emitted by reassignment; ships with or immediately after #1.
4. **First-custom reassign prompt** — small addition once reassignment exists.
5. **Collections MFE filter** — filter rule builder to org-included subcategories; independent work.
6. **Import service** — file-based bulk import accepts custom subcategory by name or ID; API path available in MVP.
7. **Catalog-to-ST reconciliation seed + migrate** — Class 1/2/3 migrations; separate work stream gated on client comms.

### Q3/Q4 — Custom Categories
- Org-specific category display names (mapped to global canonical categories)
- Same include/exclude and custom alias model, one layer up

---

## Open Questions

- **Subcategory filter in product search / FOH:** Today the subcategory filter in product search shows all options. Post-include/exclude, should it filter to only subcategories with products assigned in the org's catalog? Likely yes, but may depend on having auto-reassignment in place first to avoid showing empty filters.
- **Collections MFE rule builder for custom subcategories:** When an org has custom subcategories, should the rule builder show the custom name, the canonical name, or both? The rule stores the canonical ID — the display is a UX decision.
- **Org-specific display name resolution in the collection product grid:** The collection detail grid currently resolves subcategory name from the canonical `productSubCategory` table. For org-specific names to show in that grid, PMRS/FOH resolution needs to route through the custom subcategory table. Scope TBD.
- **Seeded default custom subcategories (Class 2 reconciliation):** When seeded defaults are created for an org, should they pre-populate as custom subcategories on the Manage page, or should they appear as globals that the org can optionally customize? Needs design alignment.
