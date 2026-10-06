# Design QA

- Source visual truth: `output/design-qa/reference.png` (user-provided screenshot; Figma node `1800:8608`)
- Implementation screenshot: `output/design-qa/implementation.png`
- Viewport: 773 × 428 CSS px, device scale factor 1
- Pixel dimensions: source 773 × 428; implementation 773 × 428; no density normalization required
- State: checkout open, PayPal selected, United States selected, payment ready

## Full-view comparison

The implementation matches the source's desktop composition: gray stage, inset white checkout, dark header with yellow divider, two-column body, three payment rows, pale right summary rail, bottom agreement copy, and dark primary action.

## Focused-region comparison

- Payment methods: icon scale, row radius, selected PayPal border/background, radio treatment, save-account inset, and recommendation badge were checked at the source viewport.
- Billing and order rail: compact floating labels, neutral fields, white summary card, red total, agreement links, and bottom action alignment were checked at the source viewport.
- No additional crop was needed because both source and implementation are readable at identical pixel dimensions.

## Findings

- No actionable P0, P1, or P2 mismatch remains.
- P3: exact Figma font metadata and exported vector assets were unavailable because the Figma reader/browser could not be initialized. The supplied screenshot was used as the visual truth, and the visible payment icons were extracted directly from it.

## Interaction verification

- Payment method switching: passed for PayPal and card.
- Billing country switching: passed for United States and Canada.
- Scenario simulation drawer: passed; all prior success, 3DS, saved-card, timeout, failure, ZIP, and email scenarios remain available, and selection updates the checkout.
- Card form reveal and disabled-submit state: passed.
- Browser console errors: none (favicon-only 404 observed before final refresh; no runtime application errors).
- Responsive check: passed at 390 × 844; columns stack without horizontal overflow and the primary action remains visible.
- Production build: passed.

## Comparison history

1. Initial implementation retained external debug/help panels, reducing fidelity. Fixed by making the checkout the default full-screen experience.
2. First equal-size capture made the billing/order rail too tall and clipped the action. Fixed by compacting label/input/card spacing and matching the source column ratio.
3. Final equal-size capture shows the full action area and matches the intended hierarchy and density.

## Implementation checklist

- [x] Match desktop checkout shell and proportions
- [x] Match payment method rows and selected state
- [x] Add PayPal save-account control and Alipay HK option
- [x] Match billing/order summary presentation
- [x] Preserve card and regional postal-code interactions
- [x] Verify production build and browser console

final result: passed
