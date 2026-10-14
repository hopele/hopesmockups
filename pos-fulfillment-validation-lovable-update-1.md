# Lovable Prototype Update — In-POS Fulfillment Validation
## Update 1: Revised Interaction Model

This document describes changes to make to the existing prototype. Apply these on top of what was already built — do not start over.

Reference the existing mockup for visual context:
https://gistcdn.githack.com/hopele/91f2a707d01f860f19f1575b5d191e6a/raw/pos-fulfillment-scan-modal-mockup.html

---

## Summary of What Changed

The original prototype had scanning happen inline inside the cart in a persistent "Validate Mode." That approach has been replaced. Scanning now happens inside a modal overlay that opens when the budtender taps "Fulfill & Checkout." The cart itself is not locked during this flow — it simply shows visual status badges on each row after the order is sent to fulfillment.

---

## Part 1 — Remove These Elements Entirely

**From Screen 1 (Build Mode):**
- Rename the "Request Items" button to **"Send to Fulfillment"** everywhere it appears.

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
- The all-validated state is handled differently — see Part 3 below.

**Remove Screen 8 (Reset Validation modal):**
- No reset flow in the new design.

**Remove "Manager Override" link from below the Checkout button:**
- It moves into the scan modal footer — see Part 3 below.

---

## Part 2 — Update Screen 2 (Confirmation Modal)

Replace the existing "Request Items" confirmation modal with the updated "Send to Fulfillment" modal.

**Title:** "Send to Fulfillment"

**Body text:** "This will save the sale and send the order to the fulfillment dashboard. Items must be scanned at the register before checkout."

**Add a checkbox below the body text:**
- Label: "Print packing receipt"
- Checked by default
- Style: standard checkbox with label text in the same size as body text

**Replace the two original buttons with three actions:**
- **"Cancel"** — outlined button, left-aligned. Dismisses modal, no action taken.
- **"Send & Stay on Sale"** — solid green (#8BC34A), white text. Saves the ticket, shows a toast "Sent to fulfillment. Packing receipt printed." (or "Sent to fulfillment." if checkbox is unchecked), stays on the active sale screen.
- **"Send & Exit Sale"** — solid green (#8BC34A), white text. Same as above, then exits the sale.

**After either Send action:**
- The fulfillment status pill appears near the Receipt # label showing "Awaiting Processing" (gray pill).
- Cart rows update — see Part 3.
- The Checkout button relabels to "Fulfill & Checkout" — see Part 3.

---

## Part 3 — Add New Cart States and "Fulfill & Checkout" Button

### Cart Row States After "Send to Fulfillment"

After either Send action, each cart row shows a small status badge below the product name. Do not disable qty controls or the trash icon — the cart remains editable.

| State | When | Left border color | Badge label |
|---|---|---|---|
| Awaiting Scan | Sent to fulfillment, not yet verified at POS | Amber (#FFB300) | "⏳ Awaiting scan" |
| Verified | Scanned and confirmed in the scan modal | Green (#8BC34A) | "✓ Verified" |
| Needs Scan | Added via + after fulfillment was already sent | Orange (#E65100) | "+ Needs scan" |

When an item is added via the left panel + button after the order has been sent, it appears immediately with the "Needs Scan" (orange) treatment.

### "Fulfill & Checkout" Button

After sending to fulfillment, the Checkout button label changes to **"Fulfill & Checkout"** (same green style, same position).

- If all items in the cart are already Verified, the label reverts to **"Checkout"** (normal green checkout behavior).
- Tapping "Fulfill & Checkout" opens the Scan Modal — see Part 4.

---

## Part 4 — Add Scan Modal (Replaces Validate Mode)

This modal opens when "Fulfill & Checkout" is tapped. It is a centered overlay on top of the POS, not a full-screen mode change.

**Important behavior:** Every time the modal opens, all scan state resets to zero. There is no carry-over from a previous checkout attempt. Items that were verified in a prior attempt start as pending again.

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

Tapping X closes the modal. All scan progress is discarded. Cart rows that were showing as "Verified" from this attempt revert to "Awaiting Scan." The next tap of "Fulfill & Checkout" starts fresh.

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

## Part 6 — Keep These Screens Unchanged

**Screen 4 — Wrong Product Scanned modal:** Keep exactly as built. Triggered by "Simulate: Wrong Product" in the scan modal controls instead of inline scan input.

**Screen 5 — Package Mismatch modal:** Keep exactly as built. Triggered by "Simulate: Wrong Package" in the scan modal controls.

**Fulfillment status pill** (near Receipt # in the bottom bar): Keep as built. Still shows Awaiting Processing / In Process / Packed & Ready.

---

## Updated Interaction Wiring

| Trigger | Action |
|---|---|
| Cart empty → tap "Send to Fulfillment" | Nothing (button disabled) |
| Cart has items → tap "Send to Fulfillment" | Open confirmation modal |
| Tap "Cancel" in confirmation modal | Dismiss, return to cart |
| Tap "Send & Stay on Sale" | Save ticket, show toast, update cart rows to Awaiting Scan, relabel Checkout → "Fulfill & Checkout", stay on sale |
| Tap "Send & Exit Sale" | Same as above, then exit sale session |
| All items already Verified → tap "Fulfill & Checkout" | Skip scan modal, proceed to checkout directly (label shows "Checkout") |
| Some items not Verified → tap "Fulfill & Checkout" | Open scan modal, all items start as pending |
| Tap X in scan modal | Close modal, discard all scan progress, reset Verified rows to Awaiting Scan |
| "Simulate: Correct Scan" | Mark next pending item as Verified, update progress bar and filter counts |
| "Simulate: Wrong Product" | Show Wrong Product modal |
| "Simulate: Wrong Package" | Show Package Mismatch modal |
| Tap "Add to Order" in Wrong Product modal | Add item as new Verified line, dismiss modal |
| Tap "Update Package" in Package Mismatch modal | Update package label on item, mark as Verified, dismiss modal |
| All items Verified | Activate "Verified — Checkout" button |
| Tap "Verified — Checkout" | Complete checkout |
| Tap "Manager Override" in scan modal footer | Open override modal |
| Tap "Override & Checkout" | Close scan modal, proceed to checkout, show red warning label |
| Add item via + after Send to Fulfillment | New cart row appears with "Needs Scan" (orange) badge; item added as pending to scan modal item list |

---

## Part 7 — Fallback Mode: New Item Added to a Previously Packed Order

This is a distinct scenario prototyped as a separate entry point. It applies only to Fallback mode config and represents the most common real-world case for that mode.

### What the scenario represents

The order was already processed by the fulfillment team and reached Packed & Ready — fulfillment confirmed all original items. A budtender then adds a new item at POS, which pushes the order back from Packed & Ready to In Process. The Fallback block fires, "Fulfill & Checkout" appears, and the cashier only needs to scan the newly added item. The original items were confirmed by fulfillment and appear pre-verified in the scan modal.

### Trigger button — location and style

Add a **"⚡ Fallback Scenario"** button to the **bottom bar of the cart** (same row as Send to Fulfillment), not inside the scan modal's simulate controls.

- Style: ghost/outlined button with amber border (#FFB300) and orange text (#E65100), slightly smaller font than the primary buttons
- Include a tooltip or title attribute: "Simulate: order was Packed & Ready, new item added at POS"
- The fallback button disables itself after being tapped — the scenario is one-time setup only.
- The "Send to Fulfillment" button remains visible and active after the fallback button is tapped. The cashier must still tap "Send to Fulfillment" to sync the updated order (with the new item) to the fulfillment dashboard.

### What happens when the fallback button is tapped (cart state)

1. Original cart rows flip to green "✓ Verified" badges — same visual treatment as a verified item in the normal flow.
2. A new cart row appears at the bottom with an orange "+ Needs scan" badge.
3. The fulfillment status pill switches to amber "🔄 In Process."
4. The Checkout button label shows "Fulfill & Checkout."
5. Show toast: "Fallback scenario: order pushed back from Packed & Ready."
6. The fallback button disables itself (grayed out). The "Send to Fulfillment" button remains available — the cashier still needs to tap it to sync the new item to FF.

### Scan modal — fallback pre-populated state

When "Fulfill & Checkout" is tapped after the fallback scenario is set up, the scan modal opens in a different initial state from the standard flow:

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

Tapping X closes the modal. When "Fulfill & Checkout" is tapped again:
- Original items still show as pre-validated (green circles + "Confirmed by fulfillment") — this state is NOT reset on close because it comes from fulfillment data, not POS scan progress
- The new item resets to pending — its POS scan progress is discarded on close, same as in the standard flow

### Interaction wiring for this scenario

| Trigger | Action |
|---|---|
| Tap "⚡ Fallback Scenario" button in bottom bar | Original cart rows → green Verified badges; new cart row → orange Needs Scan; FF pill → In Process (amber); Checkout → "Fulfill & Checkout"; toast shown; fallback button disables itself; Send to Fulfillment button remains visible |
| Tap "Send to Fulfillment" after fallback setup | Opens confirmation modal; on confirm, syncs updated order (with new item) to FF; new item badge updates from "Needs Scan" to "Awaiting Scan"; original items stay Verified |
| Tap "Fulfill & Checkout" in fallback state | Scan modal opens with original items pre-validated (green circles + "Confirmed by fulfillment" label); new item pending; progress pre-filled; subtitle shows pending count |
| "Simulate: Correct Scan" in fallback modal | Marks the pending new item as verified; progress bar completes; subtitle → "All items verified — ready for checkout"; "Verified — Checkout" activates |
| Tap X in fallback scan modal | Close modal; original items remain pre-validated on reopen; new item resets to pending |

---

## Visual Design Notes

- No new design patterns — match the existing POS component style throughout
- The scan modal is centered, ~600px wide, white background, border radius 8px, subtle drop shadow
- Modal overlay background: semi-transparent dark (#000 at 40% opacity)
- Toast notifications: bottom-center, green background, white text, auto-dismiss after 3 seconds
- Simulate scan controls: visually distinct from real UI — use a light gray bar with small muted text
- The "Needs Scan" orange left border on cart rows should use #E65100
