# Hotel Booking Plugin Security Remediation Plan

## Scope
- Plugin scanned: `wp-content/plugins/hotel-booking`
- Coverage: static scan across PHP/JS/CSS (`82` PHP files, `133` JS files, `26` CSS files) plus focused manual review on auth/AJAX/REST/output/file-write paths.
- Dependency audit:
  - `composer audit --locked`: no advisories
  - `npm audit --omit=dev`: no advisories

## Verified Risks (Prioritized)

### Critical 1: Unauthenticated data export endpoints expose customer PII
- Files:
  - `wp-content/plugins/hotel-booking/inc/HotelAjax.php:22`
  - `wp-content/plugins/hotel-booking/inc/HotelAjax.php:23`
  - `wp-content/plugins/hotel-booking/inc/HotelAjax.php:24`
  - `wp-content/plugins/hotel-booking/inc/HotelAjax.php:25`
  - `wp-content/plugins/hotel-booking/inc/HotelAjax.php:669`
  - `wp-content/plugins/hotel-booking/inc/HotelAjax.php:733`
- Problem:
  - `wp_ajax_nopriv_*` is enabled for CSV/ICS export.
  - No nonce or capability check inside handlers.
  - Endpoint accepts client-provided order IDs and exports billing name/email/total.
  - Output files are written to public uploads paths.
- Impact:
  - Unauthenticated users can trigger export and exfiltrate order/customer data.

### High 2: Reflected XSS via unsanitized query parameters in booking form
- File:
  - `wp-content/plugins/hotel-booking/templates/frontend/single-hotel/booking.php:157`
  - `wp-content/plugins/hotel-booking/templates/frontend/single-hotel/booking.php:158`
- Problem:
  - Raw `$_GET` values are concatenated into HTML attributes without escaping.
- Impact:
  - Crafted URL can inject script in frontend context.

### High 3: Reflected XSS via unsanitized query-string rehydration in ordering form
- File:
  - `wp-content/plugins/hotel-booking/templates/frontend/loop/ordering.php:7`
  - `wp-content/plugins/hotel-booking/templates/frontend/loop/ordering.php:44`
- Problem:
  - Raw `$_SERVER['QUERY_STRING']` and parsed params are output to hidden inputs without `esc_attr`.
- Impact:
  - Crafted query string can inject into rendered HTML.

### High 4: Insecure review image upload (unvalidated base64 write)
- File:
  - `wp-content/plugins/hotel-booking/inc/class-hotel-review-room.php:447`
  - `wp-content/plugins/hotel-booking/inc/class-hotel-review-room.php:455`
  - `wp-content/plugins/hotel-booking/inc/class-hotel-review-room.php:467`
- Problem:
  - Arbitrary base64 content written with `file_put_contents`.
  - Relies on user-controlled file name/type.
  - No strict image validation (`wp_check_filetype_and_ext`, `wp_get_image_mime`, size limits server-side).
- Impact:
  - Malicious file upload risk, storage abuse, possible execution risk on weak server configs.

### Medium 5: REST route allows all callers at route layer
- File:
  - `wp-content/plugins/hotel-booking/inc/class-hotel-review-room.php:380`
- Problem:
  - `permission_callback => '__return_true'`.
  - Business/auth checks are deferred to callback logic.
- Impact:
  - Weaker defense-in-depth; easier to misuse endpoint behavior.

### Medium 6: Public AJAX endpoint without nonce/rate control
- File:
  - `wp-content/plugins/hotel-booking/inc/external-plugin/elementor/function-hotels-widget.php:2`
  - `wp-content/plugins/hotel-booking/inc/external-plugin/elementor/function-hotels-widget.php:3`
  - `wp-content/plugins/hotel-booking/inc/external-plugin/elementor/function-hotels-widget.php:10`
- Problem:
  - Endpoint exposed to unauthenticated callers and trusts `$_POST`.
  - Missing nonce and request throttling.
- Impact:
  - Abuse/DoS risk and larger attack surface.

## Remediation Plan (Before Implementation)

## Phase 1: Immediate Hotfixes (same release)
1. Lock down export endpoints in `HotelAjax`.
2. Remove `wp_ajax_nopriv_hotel_export_calendar` and `wp_ajax_nopriv_hotel_export_csv`.
3. Add `check_ajax_referer` and `current_user_can( 'manage_woocommerce' )` (or stricter capability).
4. Validate and sanitize all export inputs (`room_id`, `dates_booked[*]`, `order`).
5. Escape and sanitize export content; prevent CSV formula injection (`=`, `+`, `-`, `@` prefix hardening).
6. Replace raw `$_GET` outputs in booking/ordering templates with `sanitize_text_field( wp_unslash(...) )` + `esc_attr`.

## Phase 2: Upload/REST Hardening
1. In review upload flow:
2. Replace manual file write with WordPress upload APIs (`wp_handle_sideload`/`media_handle_sideload`).
3. Enforce whitelist MIME + extension, max file count, max bytes server-side.
4. Generate safe filename with `sanitize_file_name`.
5. Reject invalid base64 and non-image content.
6. For REST route:
7. Replace `__return_true` with strict `permission_callback` that checks login and intended capability/context.
8. Validate `product_id` belongs to expected product type before `wp_insert_comment`.

## Phase 3: Defense-in-Depth + Quality Gates
1. Add centralized request helper for nonce/capability/sanitize/validate patterns.
2. Add unit/integration tests:
3. Unauthenticated export call returns `403`.
4. Invalid nonce/capability blocked.
5. XSS payload in query args rendered inert.
6. Upload rejects invalid mime/oversize payload.
7. Add CI checks (PHPStan/PHPCS with WordPress security sniffs once standards are installed).

## Implementation Order
1. `inc/HotelAjax.php` (Critical)
2. `templates/frontend/single-hotel/booking.php` and `templates/frontend/loop/ordering.php` (High)
3. `inc/class-hotel-review-room.php` (High/Medium)
4. `inc/external-plugin/elementor/function-hotels-widget.php` (Medium)
5. Tests and regression pass

## Acceptance Criteria
- Export endpoints are inaccessible to unauthenticated users.
- Export endpoints require valid nonce and capability.
- No frontend reflected XSS from query args in booking/ordering templates.
- Review uploads only accept validated image files with server-side limits.
- REST permission callback enforces auth at route level.
- No regression in booking, review submission, and calendar export behavior for authorized admins.

