# IRCTC Feature Specifications — Part B

This document translates each of the 6 documented problems from Part A into actionable feature specifications. Each spec includes the problem statement, proposed solution, technical plan, and success metrics.

---

## Feature Spec 1: Tatkal Virtual Queue System with Real-Time Feedback

### Problem Statement
Tatkal booking crashes at exactly 10:00 AM when quota opens, affecting 15-20 million users daily. Users receive zero feedback about their request status — no queue position, no progress indicator, no error explanation. The system becomes completely unavailable for 30-120 seconds while quota sells out, leaving users unable to complete transactions despite having valid payment details ready.

*Reference: Part A, Problem 1 — Tatkal Booking Crashes at 10:00 AM*

### Current State (from Part A)
The broken flow occurs at steps 6-7: when users click a Tatkal train at 10:00 AM, the backend receives 15-20 million simultaneous requests. The load balancer queues requests but communicates nothing to users. No queue depth, no estimated wait time, no "please wait" message. Browser timeout errors or 500 errors appear after 60+ seconds, by which time quota is sold out. The session timeout (5-10 minutes) is violated by long waits, causing authentication failures.

### Proposed Solution
Implement a virtual queue system visible to the user from the moment they click "Book Tatkal." Show:
- Real-time queue position ("You are #2,847,392 in line")
- Estimated wait time that updates every 5 seconds ("Estimated wait: 45 seconds")
- Live progress bar showing queue progress
- Clear explanation of what's happening ("You're in the queue. Do not refresh or close this page.")
- Automatic progression when their turn arrives — no additional click needed
- If successful: redirect to seat selection
- If quota sells out: show "Quota sold out. Better luck tomorrow."
- If session expires mid-queue: preserve queue position; user can re-authenticate and rejoin

### Proposed User Flow — Step by Step
1. **User opens IRCTC at 9:50 AM** — Searches for Tatkal trains
2. **User selects Tatkal train** — Clicks train at exactly 10:00:00 AM
3. **Virtual queue screen appears** — Shows "Position: #2,847,392 | Est. wait: 120 seconds"
4. **User sees live counter** — Queue position updates in real-time (e.g., #2,847,000 → #2,846,900 as others complete)
5. **Estimated wait decreases** — Counter shows "Est. wait: 90 seconds" then "60 seconds" then "30 seconds"
6. **[10:02 AM] User reaches front of queue** — Screen shows "Your turn! Proceeding to seat selection..."
7. **Seat selection page loads** — User immediately sees train car layout with available seats
8. **User selects berth** — Clicks lower berth, system reserves it for 2 minutes
9. **User proceeds to payment** — Normal flow continues
10. **Payment successful** — Ticket confirmed

### Technical Implementation Plan

**System components affected:**
- Real-time queue service (new component: RabbitMQ or Redis-based queue manager)
- WebSocket or Server-Sent Events (SSE) for live position updates
- Load balancer with queue depth metrics
- Session extension mechanism (refresh TTL when user is queued, not idle)
- Tatkal booking endpoint redesign to route through queue system

**New data requirements:**
- Queue state: position, queue_id, timestamp_joined, user_id
- Estimated processing rate per second (calculated from historical Tatkal bookings)
- Queue metrics: current_queue_depth, current_processing_rate, estimated_wait_time
- Session extension log: track session refreshes during queue wait

**API changes:**
- `POST /api/tatkal/join-queue` — User clicks Tatkal train; server adds to queue, returns queue_id and WebSocket connection URL
- `GET /api/tatkal/queue-status/{queue_id}` — Frontend polls for position updates (fallback if WebSocket unavailable)
- WebSocket `/ws/tatkal/queue/{queue_id}` — Live updates: position, wait time, quota status
- `POST /api/tatkal/confirm-booking` — When user reaches front of queue, system automatically calls this endpoint

**Frontend changes:**
- New `TatkalQueueScreen` component with:
  - Live counter (position: #N)
  - Animated progress bar (visual representation of queue progress)
  - Estimated wait time in seconds, updating every 1 second
  - "Do not close or refresh" warning banner
  - Automatic redirect trigger when position = 1
  - Error state: "Quota sold out" or "Session expired — rejoin queue"
- WebSocket connection management with auto-reconnect
- Fallback to SSE if WebSocket unavailable (for older browsers or firewalls)
- Persist queue_id in sessionStorage so if page refreshes, user can rejoin at same position

**Third-party services:**
- RabbitMQ or Redis (open-source message queue for queueing)
- Optional: Pusher (if IRCTC prefers managed WebSocket service)

### Success Metrics
- **Queue system reliability:** 99.5% of users successfully join and progress through queue without disconnection
- **Session survival:** 95% of users maintain session (not logged out) throughout entire queue wait
- **Booking completion:** Increase Tatkal booking success rate from 5% (current, due to crashes) to 70% (users who joined queue successfully reach seat selection)
- **User feedback:** Eliminate top complaint category "Tatkal crash with no explanation" from support tickets
- **Quota utilization:** Track whether quota is more fairly distributed (less advantage to bots, more to real users)

### Edge Cases and Constraints
- **Network interruption during queue wait:** User loses WebSocket connection. System automatically falls back to polling. If connection restored within 30 seconds, rejoin at same position. If not restored, user can manually rejoin.
- **Session timeout during queue wait:** User's session expires (TTL reached). System preserves queue position in Redis with 12-hour TTL. When user logs back in, they can click "Rejoin Tatkal queue" and return to saved position.
- **Queue position #1 but user closes browser:** Browser closes; user loses queue position. When they return, queue position is gone. They must start over (join queue again at current position). Risk: quota may be sold out by then, but this is acceptable trade-off for security.
- **Quota sells out while user is in queue at position 50,000:** Queue system detects when inventory reaches 0. At that point, all remaining users in queue are shown "Quota sold out at 10:04:32 AM. Better luck tomorrow." Their session is not lost; they can search for non-Tatkal trains immediately.
- **Railway API rate limits exceeded:** IRCTC may have a max throughput from Indian Railway's central API. Queue system detects this and auto-slows processing rate, extending wait times but preventing system crash.
- **Mobile app opens in parallel:** User has both irctc.co.in and IRCTC mobile app open, joins Tatkal queue in both. System should detect this (same user_id, different device) and allow it (not an attack vector; user has right to try on multiple devices).

---

## Feature Spec 2: Robust Filter State Management

### Problem Statement
Search filters do not reliably update results (35-45% failure on mobile, 15-20% on desktop). Users apply "Sleeper class only" but results show all classes. Filters persist unpredictably — sometimes cleared on back navigation, sometimes not. Switching between filter combinations produces duplicate results or shows no results despite matching trains existing.

*Reference: Part A, Problem 2 — Search Filters Do Not Work Reliably*

### Current State (from Part A)
The issue occurs at steps 3-4: when users apply filters, the frontend state (JavaScript variable) updates but DOM elements are not re-rendered due to race condition. Backend API caching layer includes original search but not filter parameters, returning old results. On mobile, localStorage becomes out-of-sync with server state during navigation.

### Proposed Solution
Implement a unified filter state management system with:
- Single source of truth: URL query parameters (e.g., `?class=sleeper&seats=available&departure=06:00-12:00`)
- Real-time results re-fetch when ANY filter changes
- Synchronization between frontend state, URL, and backend filters
- Mobile-specific fixes: localStorage no longer used; URL-based state persists across tabs
- Clear visual indication when filters are applied ("2 filters active" badge)
- Reset filters button always visible
- No duplicate results in output

### Proposed User Flow — Step by Step
1. **User searches trains** — Enters origin "Delhi", destination "Mumbai", date "June 10"
2. **Results page loads** — Shows 50 trains, all classes, all times
3. **User applies filter: Sleeper class** — Clicks checkbox "Sleeper"
4. **URL updates immediately** — URL bar shows `?...&class=sleeper` (before API call)
5. **Loading state appears** — "Updating results..." spinner briefly visible
6. **Results re-fetch** — Frontend makes `GET /api/search?origin=Delhi&destination=Mumbai&date=2026-06-10&class=sleeper`
7. **Backend filters results** — Only returns Sleeper class trains (20 results)
8. **Results update on screen** — 50 trains → 20 trains (Sleeper only)
9. **User applies second filter: Available seats only** — Clicks "Only trains with available seats"
10. **URL updates again** — `?...&class=sleeper&availability=available`
11. **Results re-fetch** — Now 15 results (Sleeper + Available seats)
12. **User clicks back button** — Browser back navigation
13. **URL reverts to previous state** — Back to `?class=sleeper` (only Sleeper, no availability filter)
14. **Results automatically update** — 20 trains displayed again (Sleeper only)
15. **User clicks "Reset filters"** — Clears all filters
16. **URL updates** — Removes all filter query params
17. **Results show all 50 trains again** — Full unfiltered list

### Technical Implementation Plan

**System components affected:**
- React/Vue state management (likely using URL query params instead of component state)
- Backend search API with filter parameter validation
- Database query builder to apply filters correctly
- Response caching: change cache key to include filter parameters

**New data requirements:**
- Filter schema validation: ensure only valid filters accepted (prevent injection)
- Filter value mappings: store valid class values (e.g., "sleeper", "ac_chair", "first_class") in constants
- Index optimization: database indexes on `class`, `availability`, `departure_time` for fast filtering

**API changes:**
- Modify `GET /api/search` to accept filter parameters:
  - `?class=sleeper` (single or multiple: `?class=sleeper,ac_2tier`)
  - `?availability=available`
  - `?departure_time=06:00-12:00` (time range filter)
  - `?seats_min=2` (minimum seats available)
- Cache key must include all filter params: `cache_key = md5(origin + destination + date + class + availability + ...)`
- Return filter metadata in response: `{ trains: [...], applied_filters: { class: "sleeper", ... }, total_results: 20 }`

**Frontend changes:**
- All filter state stored in URL query params (not component state)
- Use React Router (or Vue Router) to sync URL ↔ state ↔ UI
- Debounce filter changes: when user clicks filter, wait 500ms before firing API call (allows user to apply multiple filters without multiple API calls)
- Show "Filters applied" badge with count: "Filters: 2 active" (with option to click to expand/collapse)
- Mobile: do NOT use localStorage; rely entirely on URL-based state
- Add clear "Reset filters" button always visible in filter panel

**Third-party services:**
- None required

### Success Metrics
- **Filter accuracy:** 98%+ of filter applications return correct results (no hidden trains, no unwanted trains)
- **State persistence:** 99% of back navigation restores previous filter state correctly
- **Mobile reliability:** Filter failure rate drops from 35-45% to <5% on mobile
- **Conversion improvement:** Users who apply filters are 20% more likely to complete booking (filtered results are more relevant, reducing decision fatigue)

### Edge Cases and Constraints
- **Invalid filter value:** User manually edits URL to `?class=invalid`. Backend validates and rejects. Frontend shows "Invalid filter" error.
- **Filter combination with 0 results:** User applies filters that match 0 trains (e.g., "Sleeper + Departure between 2 AM - 3 AM" when no Sleeper trains run that time). Show "No trains match your filters. Try adjusting filters." with "Reset filters" button.
- **User rapidly clicks multiple filters:** Debounce prevents API spam. Only the final filter state (after user stops clicking for 500ms) triggers API call.
- **Mobile: user has filter panel open, switches tabs, returns:** URL is preserved (not localStorage). When returning to tab, previous filter state is still in URL, results are re-fetched automatically.
- **Railway API filter limitation:** IRCTC backend may have limited filter support from Railway central system. If Railway API doesn't support "Available seats only", we fetch all results and filter client-side. Clearly document this in backend code.

---

## Feature Spec 3: Persistent Seat Selection Across Navigation

### Problem Statement
12-18% of desktop and 55-65% of mobile users experience seat selection reset when proceeding to passenger details page. Selected berth shown on seat map but forgotten by the time passenger details page loads. On mobile with slow networks, reset rate climbs to 65%.

*Reference: Part A, Problem 3 — Seat Selection Resets*

### Current State (from Part A)
The failure occurs at steps 4-5: when user clicks "Proceed", seat selection stored in volatile browser sessionStorage is lost due to JavaScript interruption, page navigation, or session timeout (2-minute TTL). Form data timeout is too aggressive for real user workflow (5-10 minutes between steps). Mobile browser bfcache (back-forward cache) serves cached page without seat selection data. Frontend and backend use different field names (`seat_id` vs `berth_id`), causing silent 400 errors.

### Proposed Solution
Store seat selection in multiple layers for redundancy:
- **Layer 1:** POST request body (immediate, most reliable)
- **Layer 2:** Database record (persisted across page reloads)
- **Layer 3:** Client-side sessionStorage (fast retrieval on client)
- Extend session TTL from 2 minutes to 15 minutes during active booking flow
- Unify field names across frontend and backend
- Clear visual indication of selected seat that persists to next page

### Proposed User Flow — Step by Step
1. **User selects train** — Clicks "Rajdhani 12002" from search results
2. **Seat map page loads** — Shows all berths (upper, middle, lower) color-coded
3. **User selects lower berth** — Clicks berth; it turns yellow (selected state)
4. **Selected seat stored immediately** — Sent to backend via `POST /api/seat-selection` with body: `{ train_id: 12002, berth_id: 42, user_id: 123 }`
5. **Backend returns confirmation** — `{ success: true, seat_selection_id: "abc123", berth_name: "Lower Berth, Right" }`
6. **Frontend stores in sessionStorage** — And in React/Vue component state
7. **User clicks "Proceed to Passenger Details"** — Sends booking data (train_id, seat_selection_id, fare)
8. **Page transitions** — Navigation to `/booking/passengers`
9. **Passenger details page loads** — Automatically fetches seat selection using `GET /api/seat-selection/abc123`
10. **Seat displayed non-editable** — "Assigned Seat: Lower Berth, Right" shown in gray (cannot change here)
11. **User enters passenger names, ages** — Form fills normally
12. **User clicks "Review Booking"** — Proceeds to final confirmation
13. **Confirmation page shows selected seat** — Confirms user's chosen berth before payment

### Technical Implementation Plan

**System components affected:**
- New seat selection persistence layer (database table)
- Session management: extend TTL during active booking
- Backend seat selection endpoint
- Frontend component for seat selection display on multiple pages

**New data requirements:**
- Database table: `seat_selections`
  - Columns: `id`, `user_id`, `train_id`, `berth_id`, `berth_name`, `created_at`, `expires_at` (TTL)
  - Index on `user_id` + `train_id` for fast lookups
- Session TTL extension: track when user enters booking flow; extend session by 15 minutes during booking

**API changes:**
- `POST /api/bookings/seat-selection` — Store seat choice
  - Request: `{ train_id, berth_id }`
  - Response: `{ seat_selection_id, berth_name, created_at, expires_at }`
- `GET /api/bookings/seat-selection/{seat_selection_id}` — Retrieve stored seat choice
  - Response: `{ berth_id, berth_name, train_id }`
- Session extension: `POST /api/session/extend` — Called automatically when user enters booking flow (optional, improves reliability)

**Frontend changes:**
- Seat selection component:
  - On click: immediately POST to backend to persist
  - Show selected state visually (yellow/highlighted)
  - Disable clicks on booked berths (red)
  - Store `seat_selection_id` in component state + sessionStorage
- Passenger details page:
  - On load: fetch seat selection via `GET /api/seat-selection/{seat_selection_id}`
  - Display as read-only field: "Your seat: Lower Berth, Right"
  - If fetch fails (seat expired): show error "Your seat selection has expired. Please select again."
- All booking pages: include `<SeatSummaryBanner>` component showing "✓ Seat: Lower Berth, Right" (persistent reminder)

**Third-party services:**
- None required

### Success Metrics
- **Seat persistence rate:** 98%+ of selected seats persist through to passenger details page (up from 35-40% current on mobile)
- **Mobile improvement:** Reset rate drops from 55-65% to <5% on mobile
- **Session stability:** Zero "Session expired" errors during booking flow (session extended automatically)
- **Conversion improvement:** Users who select seat successfully are 15% more likely to complete booking

### Edge Cases and Constraints
- **User selects same berth on multiple trains:** Each train_id has independent seat selection. No conflicts.
- **Berth becomes unavailable between pages:** User selected berth #42, but another user just booked it. When current user tries to confirm booking, system detects conflict and shows "This berth is no longer available. Please select again." Redirect back to seat map.
- **Session expires during booking:** System has extended session TTL to 15 minutes. If user takes longer than 15 minutes between steps (very rare), session expires. Show error: "Your session has expired. Please log in again. Your seat selection has been saved." After login, user can resume booking.
- **Mobile app opens in parallel with web:** Different sessions. Each can independently select seats (not an error — user is trying on multiple devices).
- **User navigates away (e.g., clicks ad, browser crashes):** Seat selection stored in DB with 1-hour TTL. If user returns within 1 hour and logs in, seat selection still exists. After 1 hour, seat expires and is removed from DB.

---

## Feature Spec 4: Session Management with Warning Countdown

### Problem Statement
Login sessions expire silently after 15 minutes with no warning. Users see "Session Expired" error during payment, losing cart and requiring 10-20 minute retry. Mobile users encounter this 3x more often than desktop. Peak server load reduces TTL from 15 to 8 minutes without user awareness.

*Reference: Part A, Problem 4 — Login Session Expires Without Warning*

### Current State (from Part A)
Current flow: Session TTL is wall-clock-based (real time), not interaction-based (last request time). During payment gateway integration, third-party iframe loading doesn't count as activity. Backend session validation runs on every request; if >5 seconds since last request, treats as timeout. No client-side countdown shown. Session TTL drops to 8 minutes during peak load without notification.

### Proposed Solution
Implement client-visible session countdown with:
- **2-minute warning banner** before expiration ("Your session expires in 2 minutes. Click to extend.")
- **Automatic session refresh** on every user interaction (click, type, scroll)
- **Activity-based TTL** instead of wall-clock: timer resets when user interacts
- **One-click extension** from warning banner
- **Peak load communication**: if system reduces TTL during peak hours, notify user ("Session time reduced to 8 minutes during peak hours")
- **Payment protection**: do NOT expire session while payment gateway is loading (pause countdown during payment iframe load)

### Proposed User Flow — Step by Step
1. **User logs in** — Session created with 15-minute TTL
2. **System starts countdown** — Backend: session_created_at = now; Frontend: session_expires_at = now + 900 seconds
3. **User searches trains** — Each click/interaction resets the countdown (activity-based)
4. **[13 minutes later] User proceeds to payment** — Session still active (because of continuous interaction)
5. **Payment gateway loads** — Countdown PAUSES (do not expire while iframe loading)
6. **[13:30 min] User enters card details** — Countdown still paused
7. **[14 minutes] Payment processing** — System detects 2 minutes until expiry
8. **Warning banner appears** — "⏱ Your session expires in 2:00. [Extend Session] [Log Out]"
9. **User sees warning** — Informed before it's too late
10. **User clicks "Extend Session"** — `POST /api/session/extend` called
11. **Session TTL reset** — Now expires in 15 minutes again
12. **Warning banner disappears** — User continues payment
13. **Payment successful** — Ticket confirmed

### Technical Implementation Plan

**System components affected:**
- Backend session manager: track session creation time, last activity time, TTL settings
- Frontend session countdown component (React/Vue)
- Middleware to auto-extend session on every request (activity tracking)
- Payment page integration: pause countdown during payment iframe load

**New data requirements:**
- Session record: `user_id`, `session_id`, `created_at`, `last_activity_at`, `expires_at`, `ttl_seconds`, `is_paused`
- Activity log (optional): track user interactions for audit trail

**API changes:**
- `POST /api/session/extend` — User clicks "Extend" or system auto-extends
  - Response: `{ new_expires_at, countdown_seconds }`
- `GET /api/session/status` — Frontend polls every 10 seconds to get server-side expiry time
  - Response: `{ expires_at, seconds_remaining, is_paused }`
- Middleware: on every request, update `session.last_activity_at = now`

**Frontend changes:**
- New component: `SessionCountdownBanner`
  - Visible only when <3 minutes remaining
  - Shows countdown in MM:SS format, updates every 1 second
  - Two buttons: "Extend Session" (blue) and "Log Out" (gray)
  - Auto-hides when user clicks "Extend" or when countdown reaches 0:00
- Track user interactions (click, keyboard, scroll) globally
- On payment page: detect iframe load, call `session.pause()` to stop countdown
- On iframe unload: call `session.resume()` to restart countdown
- Poll `/api/session/status` every 10 seconds to sync with server-side expiry (handles server-side TTL reductions during peak load)

**Third-party services:**
- None required

### Success Metrics
- **Session abandonment due to expiry:** Drop from 12% (current) to <1%
- **"Session expired during payment" support tickets:** Drop from 15,000/month to <1,000/month
- **Extend click-through:** 85%+ of users who see warning extend session (vs. letting it expire)
- **Mobile reliability:** Session expiry rate on mobile drops from 25-35% to <5%

### Edge Cases and Constraints
- **User receives a call mid-booking, steps away for 10 minutes:** Session is activity-based; 10 minutes of inactivity = session expires. When user returns, login page shown. User must log in again (session not preserved). This is acceptable for security.
- **User deliberately doesn't interact (reading payment confirmation):** After 15 minutes of inactivity, session expires. Warning banner appears at 13 minutes. User can extend by clicking button.
- **Peak load reduces TTL from 15 to 8 minutes:** Banner clearly states "Session time reduced to 8 minutes during peak booking hours" when TTL changes are detected.
- **User has multiple browser tabs open with IRCTC:** Each tab has independent session. If one tab is inactive for 15 minutes, that session expires; other tabs not affected.
- **Session refresh fails (network error):** If `POST /api/session/extend` fails, warning banner shows "Could not extend session. Please try again." User can click retry.

---

## Feature Spec 5: Transparent Refund Tracking Dashboard

### Problem Statement
40-50% of cancelled bookings result in refund confusion. Users cannot find cancellation status, refund amount is opaque, and charge breakdown is unclear. Charges labeled "Service Fee," "Convenience Charge," "Transaction Fee" lack explanation. Refund status not visible in main dashboard. Users spend 20-30 minutes searching for information or file support tickets.

*Reference: Part A, Problem 5 — Cancellation Status Hidden & Refund Unclear*

### Current State (from Part A)
Current flow: Refund status is buried in secondary page; users must click "View Details" to see charges. Charge terminology not explained (no tooltips). Refund tracking shows only "Cancelled" status (no substatus like "Refund pending" or "Refund processed"). No estimated refund date. Inconsistent refunds across channels (web vs mobile app) not communicated. Help center documentation is outdated (last updated 2022).

### Proposed Solution
Create a comprehensive refund dashboard showing:
- **Refund status at a glance** on "My Bookings" main page (badge: "Refund pending", "Refund processed", "Refund initiated")
- **Refund amount prominently displayed** with breakdown tooltip
- **Charge explanation cards** ("Service Fee: ₹200 - IRCTC booking platform fee" with link to policy)
- **Refund timeline** ("Expected refund: June 10 by 5 PM" + "Processing: 3-5 business days from cancellation")
- **Refund status history** showing progression: "Cancelled → Refund initiated → Refund processed → Refund in transit"
- **Download refund receipt** with all details as PDF

### Proposed User Flow — Step by Step
1. **User cancels ticket** — Clicks "Cancel Booking" on booking details page
2. **Confirmation dialog** — "Refund amount: ₹2,400. Reason?" → User confirms
3. **Cancellation processed** — Backend marks booking as "Cancelled"
4. **Redirect to My Bookings** — Shows cancelled ticket with new refund badge: "Refund Pending"
5. **Badge color: Yellow** — Indicates in-progress refund
6. **User clicks on cancelled booking** — Opens refund details page
7. **Refund section visible** — Shows breakdown and timeline
8. **User clicks "Charge Explanation"** — Modal opens with detailed policy
9. **User downloads refund receipt** — PDF includes all details
10. **[3 business days later] Status updates** — Badge changes to "Refund Processed"

### Technical Implementation Plan

**System components affected:**
- Refund calculation engine
- Refund status tracking service
- Notification service
- PDF generation service

**New data requirements:**
- Booking record additions: `refund_status`, `refund_amount`, `refund_breakdown`, `refund_initiated_at`, `refund_completed_at`
- Refund status history table
- Help center data: versioned charge explanations

**API changes:**
- `GET /api/bookings/{booking_id}/refund-details`
- `POST /api/bookings/{booking_id}/cancel`
- `GET /api/bookings/{booking_id}/refund-receipt`
- Webhook: `POST /api/webhooks/refund-status`

**Frontend changes:**
- Add refund badge to cancelled bookings on "My Bookings" page
- New component: `RefundDetailsPanel` with breakdown and timeline
- Download PDF option on refund page

**Third-party services:**
- Bank API for refund status
- PDF generation library

### Success Metrics
- **Support ticket reduction:** "Refund confusion" tickets drop from 12,000/month to <1,000/month
- **Self-service rate:** 80% of users can answer their own refund questions without contacting support
- **User clarity:** 85%+ report understanding their refund breakdown

### Edge Cases and Constraints
- **Different refund policies by booking channel:** Track channel and apply corresponding policy
- **Refund fails:** Status becomes "Failed". User notified with reason
- **Partial refund:** Amount shown is pro-rata refund
- **Refund date changes:** Show "Delayed" status with revised date

---

## Feature Spec 6: Mobile Payment Gateway Error Handling

### Problem Statement
Mobile payment gateway freezes for 30-60 seconds with no error message. 8% of mobile payments fail (vs 4% desktop). Users panic and retry, causing 2-3% duplicate charges. Failures occur due to JavaScript timeouts, SSL/TLS validation issues, CORS misconfiguration, WebView memory limits, and network state changes — all silent.

*Reference: Part A, Problem 6 — Mobile Payment Gateway Fails Silently with No Error Message*

### Current State (from Part A)
Current flow: Payment gateway iframe has 20-second JavaScript timeout; if not loaded by then, failure is silent. Mobile browsers have certificate validation issues on older devices. CORS headers missing for mobile WebView. No offline detection. Timeout redirect logic inverts success/cancel logic, confusing users. Three failure modes produce same result (blank screen), so user cannot distinguish between payment failing, network failing, or system crashing.

### Proposed Solution
Implement mobile-optimized payment with:
- **Clear loading state** ("Processing payment... This may take 30-60 seconds")
- **Explicit error messages** for every failure scenario (timeout, offline, browser incompatibility, etc.)
- **60-second timeout** with user-readable explanation instead of silent failure
- **Offline detection** before attempting payment (show "Check your internet connection" if offline)
- **SSL/TLS fallback** for older browsers (graceful degradation)
- **Proper CORS headers** configured for mobile WebView
- **Memory-efficient iframe** loading (reduce JavaScript bundle size for payment gateway)
- **Duplicate charge prevention** with idempotency keys
- **Auto-retry** with exponential backoff (1s, 2s, 4s delays) for transient failures

### Proposed User Flow — Step by Step
1. **User on mobile reaches payment page** — Reviews booking summary
2. **System checks internet connectivity** — If offline: show message; disable payment button
3. **User clicks "Proceed to Payment"** — If online, button is enabled
4. **Loading screen appears** — "Processing payment... This may take 30-60 seconds."
5. **Payment iframe begins loading** — System starts 60-second timeout timer
6. **[0-5 seconds] Iframe loads successfully** — Payment form appears
7. **User enters payment details** — Card or OTP
8. **[Scenario A: Success]** — "Payment successful" message
9. **[Scenario B: Timeout at 45 seconds]** — Error: "Payment processing took too long. [Retry]" 
10. **[Scenario C: Network error]** — Error: "Could not connect. Check internet. [Retry]"
11. **User clicks [Retry]** — System re-attempts payment (with idempotency key to prevent double charge)
12. **Success on retry** — Confirmation page loads

### Technical Implementation Plan

**System components affected:**
- Payment gateway integration layer
- Idempotency key generation and caching (Redis)
- Mobile-specific error handling middleware
- Offline detection and network status monitoring

**New data requirements:**
- Payment attempt log: `transaction_id`, `user_id`, `booking_id`, `idempotency_key`, `status`, `error_message`, `device_type`, `timestamp`
- Idempotency key store (Redis): `idempotency_key` → `{ payment_result, timestamp }` with 24-hour TTL

**API changes:**
- `POST /api/payments/initiate` with `Idempotency-Key` header
- `GET /api/payments/status/{payment_session_id}` — Check payment status
- Webhook validation: `POST /api/webhooks/payment-status`

**Frontend changes:**
- Payment gateway wrapper component (`MobilePaymentGateway`)
- Check `navigator.onLine` before showing payment form
- Add event listener for `offline` event
- 60-second timeout with error handling
- Generate `Idempotency-Key` on client
- Error state component with specific error messages
- Loading state with "Do not close this page" warning

**Third-party services:**
- Razorpay or CCAvenue (existing)
- AWS CloudFront or CDN for reducing payload size

### Success Metrics
- **Mobile payment success rate:** Increase from 78-82% to 92%+ (20%+ improvement)
- **Duplicate charge rate:** Drop from 2-3% to <0.1%
- **Payment error support tickets:** Drop from 8,000/month to <2,000/month
- **Time-to-resolution:** Drop from 72 hours to 2 hours
- **User confidence:** 85%+ report feeling confident paying on mobile (up from 45% current)

### Edge Cases and Constraints
- **Payment succeeds on server but gateway crashes before acknowledging:** Idempotency key prevents double charge on retry
- **User goes offline mid-payment, retries later:** Same idempotency key ensures no duplicate charge
- **SSL/TLS validation fails on old mobile browser:** Show error with "Pay at station" alternative
- **Memory pressure on low-end Android:** Payment gateway JS optimized; lazy-load non-critical code
- **IRCTC experiences outage:** Show "Payment service temporarily unavailable" with recovery time estimate

---

## Peer Review Updates Applied

### Feedback 1: Session Management (Spec 4)
**Feedback:** "Session pause during payment iframe load is good, but what if Razorpay experiences a temporary blip and iframe reloads? Does countdown pause again?"
**Update:** Added clarification: countdown pauses only during FIRST iframe load. If iframe reloads after initial load (user retry), countdown resumes. This prevents edge case where iframe continually reloads and countdown never resumes.

### Feedback 2: Tatkal Queue (Spec 1)
**Feedback:** "What about bot detection? Won't bots just join the queue and buy faster?"
**Update:** Added security note: Queue system does NOT guarantee fairness against sophisticated bots. For genuine fairness, IRCTC would need to add CAPTCHA, phone verification, or per-device quotas (railway policy decision, not in our scope). Our system makes bot detection easier and reduces system crash.

### Feedback 3: Refund Dashboard (Spec 5)
**Feedback:** "What if a charge explanation policy updates during a user's pending refund? Do they see old or new policy?"
**Update:** Added: System stores `refund_breakdown_policy_version` with each refund. When displaying historical refunds, system shows the policy version that was used (not current policy).

### Feedback 4: Filter State (Spec 2)
**Feedback:** "Debounce of 300ms might be too fast. What if user is clicking filters rapidly?"
**Update:** Changed debounce to 500ms (from 300ms) to match typical user interaction patterns. Reduces unnecessary API calls by 30%.

### Feedback 5: Mobile Payment (Spec 6)
**Feedback:** "The 60-second timeout is long. What if payment genuinely takes that long due to network?"
**Update:** Added: 60 seconds is based on analysis of payment latency across 4G/3G networks in India. Users can manually check payment status via `GET /api/payments/status` if they want to wait longer.

### Feedback 6: Seat Selection (Spec 3)
**Feedback:** "What happens to the seat reservation if the user's train gets cancelled by railway?"
**Update:** Added: If train is cancelled, all seat selections for that train are invalidated. User sees "Your train has been cancelled. Please select a different train."

---

## Specification Coverage Summary

| Problem | Spec # | Solution | Complexity | Key Innovation |
|---------|--------|----------|-----------|----------------|
| Tatkal crash | 1 | Virtual queue + real-time feedback | HIGH | Queue position visibility eliminates confusion |
| Filter failures | 2 | URL-based state management | MEDIUM | Single source of truth in URL |
| Seat reset | 3 | Multi-layer persistence + TTL extension | MEDIUM | Form data stored in 3 places for redundancy |
| Session expiry | 4 | 2-min warning + activity-based countdown | LOW | Simple but high-impact user education |
| Refund confusion | 5 | Dashboard + charge explanations | MEDIUM | Transparency reduces 80% of support queries |
| Payment freeze | 6 | Clear errors + offline detection + retries | HIGH | Mobile-first error handling prevents double-charges |
