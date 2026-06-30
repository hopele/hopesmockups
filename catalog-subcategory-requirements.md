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

Global subcategories cannot have their display names changed directly. They are seeded and maintained by Treez.

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

The starting point is a category selector. After choosing a category (e.g., Concentrate), the page shows two panels:

**Include / Exclude panel**
- All global subcategories for the selected category are listed, split into Excluded and Included columns.
- Clicking a subcategory moves it between columns.
- The Global badge distinguishes canonical subcategories from org-created customs.
- Helper text explains the two-layer model upfront: including a subcategory makes it available in product dropdowns; creating custom subcategories further controls which names appear.

**Included Subcategories panel**
- Lists all included global subcategories for the selected category.
- Each global row can have zero or more custom subcategories nested under it.
- Kebab menu on each row (global and custom) offers: Edit Name, Delete, Reassign Products (fast follow — see below).

**Custom subcategory behavior**
- When at least one custom subcategory exists under a global, the global name is hidden from the product assignment dropdown. Products are assigned to one of the custom names instead. The global still appears in the UI with a "Not shown in product dropdown" indicator — it remains the compliance anchor.
- If no customs exist under a global, the global name appears directly in the product dropdown.

**Add custom subcategory**
- Inline form under the global row. Name is required.
- No reassignment modal when creating the first custom for a global — products assigned to the global stay assigned to it until a future reassign action.

**Edit custom subcategory name**
- Inline edit. Name change only; the canonical mapping does not change.
- MVP does not auto-reassign products in Sell Treez when a name is edited (fast follow).

**Delete custom subcategory**
- Only available when zero products are assigned to the custom subcategory. If products are assigned, the Delete option does not appear in the kebab.
- Deleting a custom with zero products is safe and has no downstream impact.

**Exclude a subcategory with products assigned (MVP block)**
- In MVP, excluding a subcategory that has products assigned to it is blocked. The user must first reassign those products before the exclusion can proceed. The exclusion is not allowed to be skipped.
- The reassign modal is shown when the user attempts to exclude — but the full reassign flow is a fast follow (see below).

**Impact on product editing**
- Excluded subcategories do not appear in the subcategory dropdown on the product edit form.
- If a product is already assigned to a subcategory that is later excluded, that assignment persists but the subcategory is no longer selectable for new edits. Auto-reassignment is a fast follow.

---

## Fast Follow — Product Reassignment

When excluding a subcategory with products, or deleting a custom subcategory with products, a reassign modal is shown. The flow mirrors the existing brand reassignment UX.

**Reassign modal**
- Dropdown lists valid reassignment targets: other custom subcategories under included globals, or globals with no customs.
- Reassignment is required — it cannot be skipped.
- On confirm: products are moved to the new subcategory, exclusion/deletion proceeds, a success toast is shown.

**Collection count warning**
- When the reassign modal opens, a count of automated collections referencing the source subcategory is fetched from the product-collections service.
- If n > 0, the modal shows: "{n} automated collection(s) reference this subcategory and will be updated to the new subcategory."
- If the fetch fails or is still loading, the modal remains fully functional — the action is not blocked.

---

## Collections Impact

### How Collection Rules Store Subcategories

Automated collections store filter rules as a JSONB object on the collection record. The subcategory filter uses the key `subCategory` and stores an array of canonical subcategory UUIDs — not names. Example:

```json
{
  "category": ["<categoryId>"],
  "subCategory": ["<subcategoryId-A>", "<subcategoryId-B>"]
}
```

Because rules store IDs, display name changes have no effect on rule validity. However, reassigning products (which changes their canonical `productSubCategoryId`) does change which products match a collection's rule.

### Impact Matrix

| Operation | Collection rule breaks? | Collection membership changes? | Action required |
|---|---|---|---|
| Rename custom subcategory display name | No | No | None. ID unchanged. |
| Delete custom subcategory (0 products) | No | No | None. Canonical ID unchanged. |
| Exclude a global subcategory | No — rule remains syntactically valid | No for existing products | The excluded subcategory should not appear as an option in the collection rule builder for this org. Collections MFE must filter rule options to the org's included subcategories. |
| Include a global subcategory | No | No | Newly included subcategory becomes available in the rule builder. |
| Reassign products from A to B | No — rule is syntactically valid | Yes — products leave A's membership, enter B's | Collection rules referencing A lose those products; rules referencing B gain them. Auto-rewrite rule A→B on reassignment. Show count warning in modal. |

### Reassignment: Auto-Rewrite Collection Rules

When a subcategory reassignment is confirmed, PMS emits a `SUBCATEGORY_REASSIGNED` event. The product-collections service consumes it and rewrites every affected collection's rule:
- Find all collections in the org whose `subCategory` rule array contains `fromSubCategoryId`.
- Replace `fromSubCategoryId` with `toSubCategoryId` using set semantics (dedupe if target already present).
- Republish a Collection Updated event for each modified collection so Sell Treez and the bridge recompute membership.
- Handler is idempotent.

Standalone excludes or deletes without a replacement target do NOT trigger auto-rewrite. Those collections are left stale until the user edits them manually.

**Note:** The collections MFE currently has a hardcoded name-based exclusion list (`EXCLUDED_SUB_CATEGORY_BY_CATEGORY`) that filters certain subcategory names from the rule builder. This is explicitly marked as a temporary workaround for catalog/ST mix issues and can be removed once catalog reconciliation is complete.

### Collections MFE: Filter by Org-Included Subcategories

Today the collection rule builder fetches all global subcategories. Once include/exclude is live, it must fetch only the org's included subcategories (and their custom subcategories) as rule options. This prevents users from accidentally building rules on excluded subcategories.

---

## Technical Debt — Catalog-to-ST Reconciliation

The current catalog and Sell Treez subtype data have diverged over time. Three classes of reconciliation work are needed before the custom subcategory feature is fully clean:

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
- `fromSubCategoryId` — canonical UUID being reassigned from
- `toSubCategoryId` — canonical UUID being reassigned to
- `organizationId`

---

## Feature Flag

Feature flag: **`Catalog Management - Custom Subcategory Management`** (managed in organization-management service). Enabled for Story first.

**Gated by the flag (management surface):**
- The Manage > Subcategories page
- Include/exclude org activations
- Org-initiated creation, editing, and deletion of custom subcategories
- Product assignment to a custom subcategory via the UI/API

**Not gated — runs for all orgs:**
- The `customSubCategory` table and `customSubCategoryId` FK on product (schema is global)
- Reconciliation: Class 1 canonical renames, Class 2 seeded default customs, Class 3 missing canonical seeds
- PMRS fan-out for display name edits on seeded defaults
- `ProductCore v1x3x0` additive event fields

Orgs without the flag still receive seeded defaults and reconciliation output — they just cannot create new custom subcategories or manage include/exclude until the flag is enabled.

---

## Phasing

### MVP
- Manage > Subcategories page: include/exclude panel + custom subcategory CRUD (behind feature flag)
- Block excluding a subcategory with products (no skip — user must reassign first)
- No delete option in kebab if products assigned to a custom subcategory
- Custom subcategory name edits do not auto-reassign products in ST
- No automatic collection rule rewrites

### Fast Follow
- Product reassignment flow from the Manage > Subcategories page
- When creating the **first** custom subcategory under a global, optionally prompt the user to reassign products currently assigned to the bare global onto the new custom (only applicable on first custom creation, since subsequent customs don't change the existing assignment state)
- Collection count warning in reassign modal
- Auto-rewrite collection rules on `SUBCATEGORY_REASSIGNED` event
- Collections MFE filters rule options to org-included subcategories
- Import service: accept custom subcategory by display name or ID in file-based bulk imports (non-technical self-service path; bulk re-tagging via catalog API is available in MVP)

### Q3/Q4 — Custom Categories
- Org-specific category display names (mapped to global canonical categories)
- Same include/exclude and custom alias model, one layer up

---

## Open Questions

- **Subcategory filter in product search / FOH:** Today the subcategory filter in product search shows all options. Post-include/exclude, should it filter to only subcategories with products assigned in the org's catalog? Likely yes, but may depend on having auto-reassignment in place first to avoid showing empty filters.
- **Collections MFE rule builder for custom subcategories:** When an org has custom subcategories, should the rule builder show the custom name, the canonical name, or both? The rule stores the canonical ID — the display is a UX decision.
- **Org-specific display name resolution in the collection product grid:** The collection detail grid currently resolves subcategory name from the canonical `productSubCategory` table. For org-specific names to show in that grid, PMRS/FOH resolution needs to route through the custom subcategory table. Scope TBD.
- **Seeded default custom subcategories (Class 2 reconciliation):** When seeded defaults are created for an org, should they pre-populate as custom subcategories on the Manage page, or should they appear as globals that the org can optionally customize? Needs design alignment.
