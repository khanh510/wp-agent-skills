---
title: "single-tour-booking.js analysis (delegation + flatpickr)"
source:
  - "wp-content/plugins/travel-booking/assets/js/frontend/single-tour-booking.js"
  - "wp-content/plugins/travel-booking/assets/js/admin/booking.js"
  - ".agent/workflows/travel-booking-plugin-analysis.md"
created_at: "2026-03-18"
---

# 1) Scope

This analysis focuses on:
- delegated event architecture in `single-tour-booking.js`
- Flatpickr integration and date-state flow
- migration notes from legacy jQuery admin script (`admin/booking.js`)

It follows the same plugin-level model from `travel-booking-plugin-analysis.md`: UI inputs produce normalized date + variation payloads, then PHP price engine handles final authority (`TravelPhysCalculate`, `TravelPhysAjax`).

# 2) Delegated Event Architecture (`single-tour-booking.js`)

## 2.1 Form-level delegated recalculation

`initEventListeners()` uses form-level delegation (single listeners):
- `form.addEventListener('input', ...)`
- `form.addEventListener('change', ...)`

Gate function `shouldRecalculate(target)` allows recalculation for:
- `.field-travel-booking`
- all `input[type=number]`
- all `select`
- all `input[type=radio|checkbox]`

Impact:
- async-safe for dynamically inserted variation fields
- fewer listeners than per-input binding
- keeps `updateTotalPrice()` as a single pricing orchestration entrypoint

## 2.2 Document-level delegated submit

`setupFormSubmission()` binds one delegated click listener on `document`:
- resolves `.btn-booking` via `closest()`
- prevents default
- calls `submitBooking(btn)`

Impact:
- submit button still works if injected late by theme/template hooks
- avoids race conditions with DOM-ready timing

## 2.3 Delegation quality vs legacy jQuery

Compared with `admin/booking.js` (`.on(...)`, `$(document).delegate(...)`):
- modern file uses explicit `closest()` and target guards
- fewer implicit global selectors
- clearer ownership around one manager instance

# 3) Flatpickr Integration (`single-tour-booking.js`)

## 3.1 Picker modes

The file uses 3 picker instances:
- `pickers.fixed` for fixed-date booking (`date_book`)
- `pickers.checkIn` and `pickers.checkOut` for range booking

## 3.2 Config normalization

Important normalized state:
- date format map from localized config (`travel_booking.tour_date_format`)
- locale first day (`travel_booking.flatpickr_first_day_of_week`)
- max year cap from hidden input (`phys_tour_max_year_enable`)
- disabled dates from hidden JSON (`phys_tour_dates_disable`)

This mirrors plugin architecture: frontend can represent rules, backend remains source of truth when cart item is validated/calculated.

## 3.3 Fixed-date behavior

`setupFixedDatePicker(minDate)`:
- computes `firstAvailableDate` (today if valid)
- marks disabled dates in `onDayCreate`
- paints duration range preview (start/end/date-picked classes)
- stores computed end date in `data-end-date`
- recalculates price on close

## 3.4 Check-in/out behavior

`setupCheckInOutPickers(minDate)`:
- default check-in = minDate, default check-out = +1 day (skip disabled)
- check-in close updates check-out minDate and dynamic maxDate (stop before first blocked date)
- both calendars paint selected range via `tourDatesChoose`
- `showRangeDatesChoose()` recalculates and redraws both instances

## 3.5 Integration strengths

- clear split between UI date handling and pricing math
- redraw-based visual sync for cross-calendar highlights
- event order intentionally prevents invalid end date selection

# 4) Gaps / Risks Observed

- `new Date('YYYY/MM/DD')` parsing depends on browser/date parser behavior.
- many operations rely on hidden JSON fields being valid; malformed JSON silently degrades to null.
- no debouncing on `input` recalculation (can be chatty on large forms).

# 5) Migration Mapping from `admin/booking.js` (jQuery -> vanilla)

Legacy patterns found:
- container delegation: `el.on('click', '.selector', handler)`
- document delegation: `$(document).delegate('.selector', 'change keyup', handler)`
- jQuery UI datepicker for disable/multiple/range date pricing

Vanilla mapping used in conversion artifact:
- `container.addEventListener('click'|'input'|'change', e => e.target.closest(...))`
- Flatpickr replacement for datepicker instances
- hidden field sync kept identical (`phys_dates_disable`, `phys_price_of_dates_option` JSON)

# 6) Output Artifact

Converted file created at:
- `.agent/workflow/booking-vanilla-flatpickr.js`

Coverage in conversion:
- delegated events for tour variations, group discount, personal info
- Flatpickr integrations for disable dates, multiple dates, date ranges
- JSON save format preserved to stay compatible with existing PHP handlers
- intentionally excludes sortable migration (requires SortableJS or drag-drop implementation)
