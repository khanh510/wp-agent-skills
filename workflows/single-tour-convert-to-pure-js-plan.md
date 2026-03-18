### Frontend Booking Migration Spec (jQuery `booking.js` -> `single-tour-booking.js`, full parity)

### Summary
- Produce decision-complete migration documentation in `.agent/workflow` (docs/spec only), using `travel-booking-plugin-analysis.md` as architectural context.
- Treat `assets/js/frontend/single-tour-booking.js` as canonical baseline and map all missing behavior from `assets/js/frontend/booking.js`.
- Scope is full parity, including booking flow + side widgets (notify polling, orderby, price slider, rating init).

### Implementation Changes
- Create `/.agent/workflow/frontend-booking-vs-single-tour-analysis.md` with:
  - Current-state behavior map: initialization, pricing pipeline, submit payload contract, variation handling, discount/deposit behavior.
  - Delegated-event analysis in `single-tour-booking.js` (`form input/change`, document click submit) vs direct jQuery bindings in `booking.js`.
  - Date system comparison: jQuery UI datepicker + daterangepicker in legacy vs Flatpickr in single manager.
  - Explicit parity gaps already found:
    - Combined range input `.tour-datepicker-range-checkin-checkout` path exists in legacy/template but is not implemented in `single-tour-booking.js`.
    - Legacy side features (`load_notify_products_of_order`, `tour_orderby`, `filter_tour_price_widget`, `barrating`) are absent from `single-tour-booking.js`.
    - Group-discount UI label rendering is present in legacy but not in single manager.
    - Localized price key mismatch risk: PHP localizes `thousand_separator`/`decimal_separator`, while single manager reads `price_thousands_separator`/`price_decimal_separator`.
- Create `/.agent/workflow/frontend-booking-to-single-tour-migration-spec.md` with:
  - Target architecture: keep booking core in `single-tour-booking.js`; move parity-only side features into companion module (imported by same frontend entry) to avoid bloating booking core.
  - Behavior-preserving migration mapping:
    - Date flows: fixed-date, check-in/out, and combined range mode (calendar type 1) using Flatpickr-compatible implementation.
    - Variation required-validation parity (`errors` class + timeout feedback) while preserving delegated recalculation model.
    - Payload parity for `add_tour_to_cart_phys`: `date_booking`, `date_check_in/out`, `number_ticket`, `number_children`, `tour_variations`, deposit fields, custom fields.
    - Group discount UI parity and subtotal formatting parity.
    - Side widgets parity: notify polling, orderby submit, noUiSlider sync, bar rating init.
  - Integration notes for implementers:
    - Keep handle `tour-booking-js-frontend` as single entrypoint output.
    - Ensure webpack entry/import chain includes booking core + parity companion module.
    - Define deprecation path for legacy `assets/js/frontend/booking.js` after parity acceptance.
  - Interface/API compatibility checklist:
    - `travel_booking` localized object keys required by migrated JS.
    - DOM selectors/data attributes consumed by booking form and variation blocks.
    - AJAX endpoint contract and expected response schema (`status`, `message`).

### Test Plan
- Booking core parity tests:
  - Fixed-date booking with disabled dates and duration highlighting.
  - Check-in/check-out flow with min/max constraints and range highlighting.
  - Combined range input mode (`tour_datepicker_range_checkin_checkout`) end-to-end.
  - Price-of-dates override accumulation across nights for adult/child/variation prices.
  - Required variation validation for quantity/select/checkbox/radio.
  - Group discount calculation and label rendering (percent/flat).
  - Deposit calculation paths (webtomizer + AWCDP + official plan/fixed/percent where present).
  - Submit payload parity including custom fields and redirect/cookie behavior.
- Side feature parity tests:
  - Notification polling refresh flow.
  - Archive orderby auto-submit.
  - Price slider input sync.
  - Rating widget initialization without JS errors.
- Compatibility checks:
  - `travel_booking` localization keys consumed by JS match localized payload.
  - No simultaneous behavior conflicts between migrated modules and legacy script.

### Assumptions
- Use `single-tour-booking.js` as the base implementation and port missing behavior from legacy/new drafts.
- Deliverables for this step are docs/spec artifacts only, saved under `.agent/workflow`.
- Full parity means no user-visible regressions relative to `frontend/booking.js` on product single, archive, and checkout contexts.
