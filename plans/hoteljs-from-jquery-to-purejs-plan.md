### Hotel Booking Frontend Migration Plan (jQuery -> Pure JS + Flatpickr, full parity)

### Summary
- Create new file `hotel-booking.js` in `wp-content/plugins/hotel-booking/assets/js/frontend/`.
- Migrate `hotel.js` to a vanilla class/manager architecture modeled after the single-tour analysis pattern (central state, delegated events, explicit picker instances) to new file.
- Keep the same functionality as `hotel.js`.
- Keep the `hotel.js` file as backup, do not delete it.
- Keep backend booking contract unchanged (`?hotel-ajax=add_room_to_cart` and `HotelAjax::add_room_to_cart()`).
- Replace jQuery UI datepicker with Flatpickr for both search flow and single-room booking flow, preserving disabled dates, min-date constraints, and selected-range highlighting.

### Key Changes
1. Runtime architecture
- Refactor [hotel.js](c:/laragon/www/travelwp/wp-content/plugins/hotel-booking/assets/js/frontend/hotel.js) into `HotelBookingManager` with:
  - `elements` map (forms, inputs, spinner, extras, totals).
  - `state` map (date format, disable lists, schedule data, nights booked, qty limits, old selected dates).
  - `pickers` map (`searchCheckIn`, `searchCheckOut`, `bookingCheckIn`, `bookingCheckOut`).
- Keep existing calculation behavior (`calculate_qty_can_book`, nightly schedule pricing, extras/deposit price effects) functionally identical.

2. Flatpickr migration
- Replace all `datepicker(...)` calls with Flatpickr instances.
- Implement date-format mapping from localized PHP format to Flatpickr format (`Y/m/d`, `d-m-Y`, etc.).
- Recreate `beforeShowDay` behavior via:
  - `disable` callbacks for blocked dates.
  - `onDayCreate` for visual classes (`date-picked`, `start-date`, `end-date`, `unavailable`).
- Keep booking rules:
  - Check-in minimum = first available date.
  - Check-out minimum = check-in + 1 day.
  - Check-out maximum = day before next disabled date.
- Remove Safari-only parsing branch by normalizing date parsing/format conversion in one utility path.

3. Event model and submit flow
- Replace direct jQuery bindings with delegated listeners:
  - Form-level `input/change` for qty + extras recalc.
  - Document/form delegated click for price-details toggle and booking submit.
- Replace `$.ajax` with `fetch` + `FormData`.
- Preserve payload keys exactly:
  - Required: `nonce`, `room_id`, `room_date_check_in`, `room_date_check_out`, `qty_room`.
  - Optional: `extras_room` (JSON string), `room_deposit`.
- Keep redirect/error UX parity (spinner state, disable button, `.error-message` behavior).

4. Integration and compatibility
- Update frontend enqueue in [hotel-booking-phys.php](c:/laragon/www/travelwp/wp-content/plugins/hotel-booking/hotel-booking-phys.php):
  - Add Flatpickr CSS loading path (from bundle output or explicit enqueue).
  - Remove `jquery-ui-datepicker` dependency from hotel frontend handle after parity validation.
  - Keep `hotel_booking_phys_js_frontend` localization object stable.
- Add `flatpickr` dependency in `package.json` and keep webpack entry `frontend/hotel` unchanged.
- No API contract change in [HotelAjax.php](c:/laragon/www/travelwp/wp-content/plugins/hotel-booking/inc/HotelAjax.php).

### Test Plan
- Search form
  - Check-in/out pickers initialize and enforce min-date sequence.
  - Date changes trigger recalculation hooks without JS errors.
- Single booking form
  - Disabled dates blocked correctly.
  - Range highlighting appears on both calendars.
  - Qty max updates correctly from booked nights and disabled dates.
  - Price updates match legacy for: base nights, schedule overrides, extras qty, extras checkbox.
- AJAX booking
  - Success path redirects to `page_redirect`.
  - Invalid date/qty/server errors render visible message and re-enable button.
  - Payload field names and formats match current backend expectations.
- Woo cart integration regression
  - Added cart item contains hotel booking meta (`room_date_check_in/out`, extras, deposit flag).
  - Cart totals still include schedule pricing + extras + deposit behavior.

### Assumptions
- Scope is full file parity (search + booking + submit), not booking-only.
- Backend `HotelAjax::add_room_to_cart()` stays unchanged except optional hardening in a later pass.
- Existing template hidden fields and selectors remain stable during this migration.
