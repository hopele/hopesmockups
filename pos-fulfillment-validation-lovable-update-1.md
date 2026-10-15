# Lovable Prototype Update — In-POS Fulfillment Validation
## Update 1: Revised Interaction Model

This document describes changes to make to the existing prototype. Apply these on top of what was already built — do not start over.

Reference the existing mockup for visual context:
https://gistcdn.githack.com/hopele/91f2a707d01f860f19f1575b5d191e6a/raw/pos-fulfillment-scan-modal-mockup.html

---

## Summary of What Changed

The original prototype had scanning happen inline inside the cart in a persistent "Validate Mode." That approach has been replaced. Scanning now happens inside a modal overlay that opens when the budtender taps Checkout — triggered automatically based on the store's config and the current fulfillment status of items on the order. The cart is not locked during this flow.

"Send to Fulfillment" has been retired as a distinct button. All saved orders already appear on the fulfillment dashboard — saving the sale is how FF is notified. The bottom bar save actions have been updated to reflect this clearly.

---

## Part 1 — Remove These Elements Entirely

**Remove the "Request Items" button:**
- Remove it from the bottom bar entirely. It is replaced by updated save buttons — see Part 2.

**Remove Screen 2 (Request Items confirmation modal):**
- No longer needed. See Part 2 for the replacement.

**Remove Screen 3 (Validate Mode) completely:**
- Remove the amber mode banner ("Fulfillment Validation — Scan items to confirm before checkout")
- Remove the scan input field at the top of the cart panel ("Scan item barcode to validate...")
- Remove the disabled state on qty controls (− / +) in the cart
- Remove the disabled/grayed trash icons in the cart
- Remove the "Reset" text link
- Remove the "Request More Items" text link
- Remove the "Save & Exit" relabeling of "Save Sale"

**Remove Screen 6 (All Items Validated banner state):**
- Remove the green banner variant ("All items validated — ready for checkout")

**Remove Screen 8 (Reset Validation modal):**
- No reset flow in the new design.

**Remove "Manager Override" link from below the Checkout button:**
- It moves into the scan modal footer — see Part 4.

---

## Part 2 — Update the Bottom Bar Save Actions

Replace the existing "Save Sale" button with two explicit save buttons. Remove "Request Items" entirely — it is no longer needed.

**Bottom bar button changes:**

Remove:
- "Save Sale" (single button)
- "Request Items" button

Add in their place (in this order, between Print Label and Print Receipt):
- **"Save & Exit"** — ghost/outlined style, same as existing ghost buttons. Saves the sale and exits to the customer queue. This is the default save behavior.
- **"Save & Stay"** — ghost/outlined style. Saves the sale and keeps the budtender on the active sale screen.

**After either save action:**
- Toast appears: "Sale saved."
- The fulfillment status pill appears near the Receipt # label showing "⏳ Awaiting Processing" (gray pill) — FF has received the order.
- Cart rows update to show FF item status — see Part 3.

**Current Sale dropdown (top bar pill):**

The "↗ Current Sale" pill in the top bar opens a dropdown. Add save options at the top of this dropdown:
- **Save & Exit** — saves and exits the sale
- **Save & Stay on Sale** — saves and stays
- Divider, then any existing sale-level actions below (View Sale Details, Add Order Notes, Void Sale, etc.)

---

## Part 3 — Cart Row FF Status and Checkout Behavior

### Cart Row FF Status

After the ticket is saved, cart rows show fulfillment verification status based on what FF has actually confirmed. Do not disable qty controls or the trash icon — the cart remains editable.

| State | When | Left border color | Badge |
|---|---|---|---|
| No badge | Ticket not yet saved, or FF has not yet confirmed this item | None | — |
| Verified by fulfillment | FF has confirmed this item (fulfilledQuantity = quantity) | Green (#8BC34A) | "✓ Verified by fulfillment" |

There are no invented intermediate states ("Awaiting Scan," "Needs Scan"). The badge only appears when FF has positively confirmed an item. Everything else shows no badge.

### Checkout Button Behavior

The Checkout button label does **not** change. It always reads **"Checkout"**.

When the budtender taps Checkout:
- **If all items are verified by FF** (or config does not require scanning): proceed to checkout normally.
- **If scanning is required** (based on store config and unverified items): the Scan Modal opens instead — see Part 4. A small indicator below the Checkout button reads: *"Items require scanning before checkout"* while any items remain unverified.

---

## Part 4 — Add Scan Modal (Replaces Validate Mode)

This modal opens when Checkout is tapped and scanning is required. It is a centered overlay on top of the POS, not a full-screen mode change.

**Important behavior:** Scan state is persisted on the order for the session. If the cashier partially or fully scans items, closes the modal, and reopens it (e.g., after returning from the payment screen), items that were already scanned remain verified. Scan state resets only if items are added or quantities change on the order.

**Why no backend persistence is needed:** The POS payment screen is not a separate page or route — it is a Redux tab toggle (`paymentTabShowing`) within the same SPA. Navigating to payment and back does not destroy the Redux store. Storing scan state in Redux (keyed by ticket ID) is sufficient for the core use case. `sessionStorage` keyed by ticket ID can serve as a lightweight fallback for edge cases like browser refresh or device switch, but no new DB column or API endpoint is required for MVP.

### Modal Structure

**Header:**
- Title: "Scan Items to Verify"
- Subtitle: "Scan each item to confirm before checkout" — updates to "X of Y items verified" as items are scanned
- X close button (top right) — closes modal, discards all scan progress, returns to cart

**Progress bar:**
- Full-width bar below the header
- Fills green left-to-right as items are scanned
- Shows fraction label: "0 / 3 verified" updating with each scan

**Filter tabs:**
- Three tabs: **All** | **Scanned** | **Waiting**
- Each tab shows a live count in parentheses, e.g., "All (3)" / "Scanned (1)" / "Waiting (2)"
- Tabs filter the item list below

**Item list:**
Each item row contains:
- Left: circle status indicator (28px)
  - Pending: empty circle, gray border (#d0d0d0)
  - Verified: filled green circle (#8BC34A) with white checkmark
- Center: product name (bold), package/lot number below in gray
- Right: qty badge ("1 unit", "2 units", etc.)

For qty > 1 items: show scan progress inline below the package number, e.g., "1 of 2 scanned" in small amber text. The circle shows as an amber partial ring while partially scanned, then flips to green checkmark when fully scanned.

**Simulate scan controls (prototype only):**
- A bar of buttons for testing: "Simulate: Correct Scan" | "Simulate: Wrong Product" | "Simulate: Wrong Package"
- Style these clearly as prototype controls (light gray background, small text) so they read as separate from the real UI
- "Simulate: Correct Scan" marks the next pending item as verified
- "Simulate: Wrong Product" triggers the Wrong Product modal (Screen 4 from original — keep as-is)
- "Simulate: Wrong Package" triggers the Package Mismatch modal (Screen 5 from original — keep as-is)

**Footer:**
- Left: **"Manager Override"** — small text link. Opens the Manager Override modal (see Part 5).
- Right: **"Verified — Checkout"** button — solid green (#558B2F), disabled (gray, #BDBDBD) until all items are verified. When all verified, becomes active green and tapping it completes checkout.

### All Items Verified State

When every item reaches Verified:
- Progress bar fills completely
- Subtitle updates to "All items verified — ready for checkout"
- "Verified — Checkout" button activates (green, clickable)
- Cart rows all show green "✓ Verified" badges

### Closing the Modal Without Completing

Tapping X closes the modal. Scan progress is **preserved** — items already verified remain verified. Cart rows update to show "✓ Verified" for any items scanned so far. The next tap of Checkout reopens the modal showing current state, with remaining unverified items still pending.

---

## Part 5 — Update Manager Override (Screen 7)

Keep the existing override modal content. The only change is the trigger location:

**Old trigger:** Text link below the grayed-out Checkout button on the main cart screen.

**New trigger:** "Manager Override" text link in the scan modal footer (left side).

The override modal itself stays the same:
- Title: "Manager Override — Skip Validation"
- Body: "A manager PIN is required to bypass fulfillment validation. This action will be logged."
- PIN input (4-digit, masked)
- "Cancel" — dismiss, return to scan modal
- "Override & Checkout" — solid red (#E53935); simulate success, close both the override modal and the scan modal, proceed to checkout, show toast "Override logged — proceeding to checkout"

Post-override: show a small red warning label below the checkout summary area: "Validation overridden by manager"

---

## Part 6 — Update Print Receipt Button

The "Print Receipt" button in the bottom bar now opens a modal instead of printing immediately.

**Modal title:** "Print Receipt"

**Body text:** "Select which receipts to print."

**Two checkbox options:**
- **Customer Receipt** — checked by default. Subtitle: "Itemized receipt for the customer."
- **Packing Receipt** — unchecked by default. Subtitle: "Back-of-house receipt for fulfillment."

**Actions:**
- **"Cancel"** — outlined button, dismisses modal, no print action taken.
- **"Print"** — solid green (#8BC34A). Prints whichever options are checked. Shows toast confirming what was printed:
  - Both checked: "Customer receipt and packing receipt printed."
  - Customer only: "Customer receipt printed."
  - Packing only: "Packing receipt printed."

The Print Receipt button is always enabled (not gated on fulfillment state).

---

## Part 7 — Keep These Screens Unchanged

**Screen 4 — Wrong Product Scanned modal:** Keep exactly as built. Triggered by "Simulate: Wrong Product" in the scan modal controls instead of inline scan input.

**Screen 5 — Package Mismatch modal:** Keep exactly as built. Triggered by "Simulate: Wrong Package" in the scan modal controls.

**Fulfillment status pill** (near Receipt # in the bottom bar): Keep as built. Still shows Awaiting Processing / In Process / Packed & Ready.

---

## Updated Interaction Wiring

| Trigger | Action |
|---|---|
| Tap "Save & Exit" | Save sale, show toast "Sale saved.", exit to customer queue; FF status pill appears |
| Tap "Save & Stay" | Save sale, show toast "Sale saved.", remain on active sale screen; FF status pill appears |
| FF confirms an item | That cart row gains green "✓ Verified by fulfillment" badge |
| All items verified by FF → tap Checkout | Proceed to checkout normally, no scan modal |
| Some items not verified by FF → tap Checkout | Scan modal opens; all items start as pending regardless of FF status |
| Tap X in scan modal | Close modal, preserve scan progress; verified items stay verified on cart rows |
| "Simulate: Correct Scan" | Mark next pending item as Verified, update progress bar and filter counts |
| "Simulate: Wrong Product" | Show Wrong Product modal |
| "Simulate: Wrong Package" | Show Package Mismatch modal |
| Tap "Add to Order" in Wrong Product modal | Add item as new Verified line, dismiss modal |
| Tap "Update Package" in Package Mismatch modal | Update package label on item, mark as Verified, dismiss modal |
| All items Verified | Activate "Verified — Checkout" button |
| Tap "Verified — Checkout" | Complete checkout |
| Tap "Manager Override" in scan modal footer | Open override modal |
| Tap "Override & Checkout" | Close scan modal, proceed to checkout, show red warning label |
| Tap "Print Receipt" | Open Print Receipt modal with Customer Receipt (checked) and Packing Receipt (unchecked) options |
| Tap "Print" in Print Receipt modal | Print selected receipts, show toast confirming what was printed, dismiss modal |

---

## Part 8 — Fallback Mode: New Item Added to a Previously Packed Order

This is a distinct scenario prototyped as a separate entry point. It applies only to Fallback mode config and represents the most common real-world case for that mode.

### What the scenario represents

The order was already processed by the fulfillment team and reached Packed & Ready — fulfillment confirmed all original items. A budtender then adds a new item at POS, which pushes the order back from Packed & Ready to In Process. The Fallback block fires, "Fulfill & Checkout" appears, and the cashier only needs to scan the newly added item. The original items were confirmed by fulfillment and appear pre-verified in the scan modal.

### Trigger button — location and style

Add a **"⚡ Fallback Scenario"** button to the **bottom bar of the cart** (same row as Save & Exit and Save & Stay), not inside the scan modal's simulate controls.

- Style: ghost/outlined button with amber border (#FFB300) and orange text (#E65100), slightly smaller font than the primary buttons
- Include a tooltip or title attribute: "Simulate: order was Packed & Ready, new item added at POS"
- The fallback button disables itself after being tapped — the scenario is one-time setup only.

### What happens when the fallback button is tapped (cart state)

1. Original cart rows flip to green "✓ Verified by fulfillment" badges — reflecting FF-confirmed status.
2. A new cart row appears at the bottom with no badge — it has not been confirmed by FF.
3. The fulfillment status pill switches to amber "🔄 In Process."
4. Show toast: "Fallback scenario: order pushed back from Packed & Ready."
5. The fallback button disables itself (grayed out).
6. The cashier saves the updated order (Save & Exit or Save & Stay) to sync the new item to FF before proceeding to checkout.

### Scan modal — fallback pre-populated state

When Checkout is tapped after the fallback scenario is set up, the scan modal opens in a different initial state from the standard flow:

**Original items (pre-validated by FF):**
- Green filled checkmark circle — same circle-done style as a scanned item
- Row background: light green tint (#f9fdf4)
- Below the package number: small green text label — **"Confirmed by fulfillment"**
- Qty badge: green-tinted (same as a completed scan)
- No cashier action required on these items

**New item:**
- Empty gray pending circle — standard pending state
- No label below the package number
- Must be scanned by the cashier

**Progress bar and subtitle:**
- Progress bar pre-filled to reflect confirmed items, e.g., 3/4 if there are 3 original items and 1 new item
- Subtitle text: **"1 new item needs scanning — 3 confirmed by fulfillment"** (adjust numbers to match actual item counts)

**Filter tabs:**
- Scanned count: matches number of pre-validated items
- Waiting count: 1 (the new item)

**"Simulate: Correct Scan"** targets only the pending new item — pre-validated items are already done and are skipped automatically.

### Closing and reopening the modal in fallback mode

Tapping X closes the modal. When Checkout is tapped again:
- Original items still show as pre-validated (green circles + "Confirmed by fulfillment") — comes from FF data, never resets
- The new item retains whatever POS scan progress was made — same persistence behavior as the standard flow

### Interaction wiring for this scenario

| Trigger | Action |
|---|---|
| Tap "⚡ Fallback Scenario" button in bottom bar | Original cart rows → green "✓ Verified by fulfillment" badges; new cart row → no badge; FF pill → In Process (amber); toast shown; fallback button disables itself |
| Tap Save & Exit or Save & Stay after fallback setup | Sale saved, new item synced to FF |
| Tap Checkout in fallback state | Scan modal opens with original items pre-validated (green circles + "Confirmed by fulfillment" label); new item pending; progress pre-filled; subtitle shows pending count |
| "Simulate: Correct Scan" in fallback modal | Marks the pending new item as verified; progress bar completes; subtitle → "All items verified — ready for checkout"; "Verified — Checkout" activates |
| Tap X in fallback scan modal | Close modal; original items remain pre-validated on reopen; new item retains its POS scan progress |

---

## Visual Design Notes

- No new design patterns — match the existing POS component style throughout
- The scan modal is centered, ~600px wide, white background, border radius 8px, subtle drop shadow
- Modal overlay background: semi-transparent dark (#000 at 40% opacity)
- Toast notifications: bottom-center, green background, white text, auto-dismiss after 3 seconds
- Simulate scan controls: visually distinct from real UI — use a light gray bar with small muted text
