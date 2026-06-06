# Impact vs Effort Matrix — IRCTC Design Sprint

## The Matrix

|                   | **Low Effort** | **High Effort** |
|-------------------|---|---|
| **High Impact**   | **Spec 4: Session Countdown** | **Spec 1: Tatkal Queue**<br>**Spec 6: Mobile Payment Errors** |
| **Low Impact**    | *None* | **Spec 2: Filter State**<br>**Spec 3: Seat Persistence**<br>**Spec 5: Refund Dashboard** |

---

## Scoring Methodology

### Impact Scoring (Scale: 1-5)
I scored Impact based on:
1. **Number of affected users (from Part A)**
2. **Severity for affected users** (can they book at all? Do they lose money?)
3. **Frequency of occurrence** (daily vs occasional)
4. **Cost to business** (support tickets, refunds, reputation)

**Scoring Guide:**
- **5 = Critical:** >5M users/day affected; blocks core flow; 100% reproducible
- **4 = High:** 2-5M users/day affected; adds friction; frequent
- **3 = Medium:** <2M affected; tangential to core flow; occasional
- **2 = Low:** <1M affected; edge case
- **1 = Minimal:** Negligible impact

### Effort Scoring (Scale: 1-5)
I scored Effort based on:
1. **Number of system components touched** (frontend, backend, database, third-party integrations)
2. **Infrastructure required** (new services, new dependencies)
3. **Risk of breaking existing flows** (regression testing overhead)
4. **Railway API dependencies** (unknown unknowns; changing APIs)

**Scoring Guide:**
- **5 = Very High:** 4+ system components; new infrastructure; major regression risk; Railway API changes
- **4 = High:** 3-4 components; moderate regression risk
- **3 = Medium:** 2-3 components; manageable risk
- **2 = Low:** 1-2 components; minimal regression risk
- **1 = Minimal:** Isolated change; no dependencies; existing patterns

---

## Placement Justifications

### Spec 1: Tatkal Virtual Queue System — **HIGH IMPACT, HIGH EFFORT**

**Impact Reasoning:**
- **15-20 million users/day affected** during peak Tatkal moments (10 AM opening)
- **100% reproducible failure** (crashes at exact moment every day)
- **Affects core business** (Tatkal bookings generate revenue; crashes cause negative PR)
- **Severe consequence** (users unable to book, lose money on alternative transport)
- **Score: 5/5** — This is a critical business blocker

**Effort Reasoning:**
- **Components:** Backend queue service (new), Redis/RabbitMQ (new infra), WebSocket layer (new), frontend TatkalQueueScreen (new), session management (modification), load balancer config (modification)
- **Regression risk:** HIGH — Queue system affects Tatkal booking flow which is core revenue stream
- **Railway API dependency:** Medium — Need to verify Tatkal quota API behavior under load
- **Total system components:** 6+ (new and modified)
- **Score: 5/5** — Major infrastructure addition

**Quadrant Placement Justification:**
Despite high effort, this must be done first. It's the #1 pain point (appears in all user research, Twitter, app reviews). High effort is acceptable because impact justifies it. Execute in parallel with Spec 4 (low-effort quick win) to maintain team morale.

---

### Spec 2: Robust Filter State Management — **MEDIUM IMPACT, MEDIUM EFFORT**

**Impact Reasoning:**
- **5-8 million searches/day affected** (35-45% failure rate on mobile, 15-20% on desktop)
- **Affects user satisfaction** but doesn't prevent booking (workaround: manual scroll through all results)
- **Reduces conversion** by ~15% (users give up on filtered search, abandon booking)
- **Secondary pain point** (not mentioned in every support ticket, but 20% of filtering attempts fail)
- **Score: 3.5/5** — Significant but not critical; users can workaround

**Effort Reasoning:**
- **Components:** Frontend state management (redesign), backend search API (parameter validation), database query optimization, caching layer (modify)
- **Regression risk:** Medium — Search results display is core flow; any changes could break existing bookings
- **Infrastructure:** No new external services; works with existing database
- **Total components:** 4 (all modifications to existing components)
- **Score: 3/5** — Manageable with existing patterns (React Router already in use)

**Quadrant Placement Justification:**
Medium impact (significant but not blocking), medium effort (standard web state management). Execute in Wave 2 (after Spec 1, 4, 6). Lower priority because workarounds exist.

---

### Spec 3: Persistent Seat Selection Across Navigation — **MEDIUM IMPACT, MEDIUM EFFORT**

**Impact Reasoning:**
- **2-3 million affected/day** (12-18% desktop, 55-65% mobile)
- **Disproportionate impact on mobile** (65% of traffic, 55-65% fail rate)
- **High frustration** (users lose their selection after clicking "Proceed"; must redo)
- **Not revenue-blocking** (users retry, eventually complete booking)
- **Score: 3.5/5** — High friction but users can recover

**Effort Reasoning:**
- **Components:** Frontend seat component (modify), backend seat persistence layer (new table), session management (extend TTL), database schema (add seat_selections table)
- **Regression risk:** Medium — Seat selection is core flow; database schema change needs careful migration
- **Infrastructure:** No new external services; uses existing database
- **Total components:** 4
- **Score: 3/5** — Moderate complexity; database schema change is medium-risk

**Quadrant Placement Justification:**
Medium impact (significant mobile issue), medium effort. Execute in Wave 2. Priority lower than Spec 1 because only affects seat flow, not the entire booking flow.

---

### Spec 4: Session Management with Warning Countdown — **HIGH IMPACT, LOW EFFORT**

**Impact Reasoning:**
- **3-5 million sessions/day affected** (8-12% desktop, 25-35% mobile)
- **Affects payment stage** (most critical moment; user loses cart)
- **High frustration** (requires 10-20 minute retry)
- **Most mentioned support issue** (15,000+ tickets/month)
- **Easy to fix** (just add a countdown timer + user notification)
- **Score: 4/5** — High-volume pain point that's trivially simple to solve

**Effort Reasoning:**
- **Components:** Frontend SessionCountdownBanner (new component), backend session manager (modify to track TTL), API for session extension (simple POST endpoint)
- **Regression risk:** LOW — No database schema changes; no core flow changes
- **Infrastructure:** No new services
- **Total components:** 2-3
- **Implementation pattern:** Existing in many web applications (login timeout warnings); well-established UX pattern
- **Score: 1.5/5** — Trivial; similar to Gmail's "Session expiring soon" banner

**Quadrant Placement Justification:**
**QUICK WIN.** High impact, low effort, minimal risk. Execute FIRST (parallel with Spec 1). This should take 2-3 days. Delivers massive value (eliminates 12% of bookings being abandoned at payment stage). Builds team momentum.

---

### Spec 5: Transparent Refund Tracking Dashboard — **MEDIUM IMPACT, MEDIUM EFFORT**

**Impact Reasoning:**
- **2-3 million cancellations/month** (40-50% of users confused about refund)
- **Creates support burden** (12,000 refund confusion tickets/month)
- **Not payment-blocking** (refund confusion happens post-booking)
- **Improves NPS** (transparent refunds increase customer trust)
- **Score: 3/5** — High support overhead but doesn't prevent bookings

**Effort Reasoning:**
- **Components:** Frontend RefundDetailsPanel (new), backend refund calculation engine (new logic), database fields (add refund_breakdown, refund_status, timestamps), notification system (modify), PDF generation (new or third-party)
- **Regression risk:** Medium — Adding fields to booking record; must careful migration
- **Infrastructure:** Optional PDF service (existing library sufficient)
- **Total components:** 5
- **Score: 3/5** — Moderate complexity; new refund calculation logic needed

**Quadrant Placement Justification:**
Medium impact (high support overhead), medium effort. Execute in Wave 2. Lower priority because doesn't affect booking success — just reduces support tickets. High ROI on time (eliminates 80% of refund support inquiries).

---

### Spec 6: Mobile Payment Gateway Error Handling — **HIGH IMPACT, HIGH EFFORT**

**Impact Reasoning:**
- **5-8 million mobile payment attempts/day** (8% failure rate = 400-640K failures/day)
- **Affects revenue** (payment failures = lost bookings = lost revenue)
- **Causes duplicate charges** (2-3% of failures result in users retrying, causing double-charge)
- **High support overhead** (8,000 "payment failed" tickets/month)
- **Core monetization flow** (booking is incomplete without payment)
- **Score: 5/5** — Critical business impact

**Effort Reasoning:**
- **Components:** Frontend payment error handling (redesign), idempotency key system (new), offline detection (new), mobile WebView optimization (new), Razorpay integration (modification), retry logic (new), timeout handling (new)
- **Regression risk:** HIGH — Payment flow is critical; any changes could accidentally break existing successful payments
- **Infrastructure:** Redis for idempotency key store (may already exist)
- **Third-party dependency:** Razorpay; need to coordinate with them on CORS, SSL certificate issues
- **Total components:** 6+
- **Score: 4.5/5** — Complex payment flow; high regression risk

**Quadrant Placement Justification:**
High impact (core revenue stream), high effort (complex payment integration). Execute in Wave 1 (parallel with Spec 1, after Spec 4). Execute second after Spec 4 (quick win). Payment issues directly affect revenue.

---

## Recommended Sprint Order (Execution Sequence)

### Wave 1: Quick Wins + Core Blockers (Weeks 1-2)

**1. Spec 4: Session Management Countdown** ⭐ **START HERE**
- **Duration:** 2-3 days
- **Why first:** Highest ROI on time. Eliminates 12% of abandoned bookings. Low risk. Builds momentum.
- **Deliverable:** Session countdown banner, extension button, warning at 2-min mark
- **Owner:** Frontend Lead + Backend (session TTL extension)

**2. Spec 1: Tatkal Virtual Queue System**
- **Duration:** 3-4 weeks
- **Why parallel with Spec 4:** Critical business blocker. Start immediately after Spec 4 begins (can overlap).
- **Deliverable:** Queue service, WebSocket integration, position counter, estimated wait time
- **Owner:** Backend Lead (new queue service) + Frontend (TatkalQueueScreen) + DevOps (load balancer config)

**3. Spec 6: Mobile Payment Gateway Error Handling**
- **Duration:** 2-3 weeks
- **Why Wave 1:** High impact (revenue). Execute after Spec 4 stabilizes (can overlap with Spec 1).
- **Deliverable:** Error messages for all failure modes, offline detection, retry logic, idempotency keys
- **Owner:** Frontend Lead (mobile UX) + Backend (idempotency) + Payment Integration Lead

### Wave 2: Friction Reducers (Weeks 3-4)

**4. Spec 2: Robust Filter State Management**
- **Duration:** 1-2 weeks
- **Why Wave 2:** Medium impact, medium effort. Improves conversion by ~15%.
- **Deliverable:** URL-based filter state, real-time result refresh, no duplicate results
- **Owner:** Frontend Lead (state management) + Backend (filter API validation)

**5. Spec 3: Persistent Seat Selection**
- **Duration:** 1.5-2 weeks
- **Why Wave 2:** Medium impact, moderate effort. Reduces mobile friction significantly.
- **Deliverable:** Multi-layer seat persistence, seat_selections table, session TTL extension
- **Owner:** Backend Lead (seat persistence) + Frontend (UI)

**6. Spec 5: Transparent Refund Dashboard**
- **Duration:** 1.5-2 weeks
- **Why Wave 2:** Medium impact on support overhead. Doesn't affect core booking flow.
- **Deliverable:** Refund status page, charge breakdown, receipt download, timeline
- **Owner:** Frontend Lead (dashboard) + Backend (refund calculation) + Backend (PDF generation)

### Parallel Initiatives
- **AI Feature (Predictive Tatkal Queue):** Develop in parallel with Spec 1. Use Tatkal booking data to train model in Week 3-4. Deploy in Week 4 after Spec 1 is stable.
- **Documentation:** Update help center with new features as they roll out. Dedicate 1 person part-time.
- **Monitoring:** Set up dashboards for queue depth, payment failure rates, session expiry rates. Monitor daily during and after deployment.

---

## Risk-Adjusted Timeline

| Spec | Wave | Duration | Risk | Mitigation |
|------|------|----------|------|------------|
| 4 | 1 | 2-3 days | Low | None needed; standard feature |
| 1 | 1 | 3-4 weeks | High | Start infrastructure early; load test weekly |
| 6 | 1 | 2-3 weeks | High | Partner with Razorpay; stage deployments |
| 2 | 2 | 1-2 weeks | Medium | A/B test filter changes; revert if conversion drops |
| 3 | 2 | 1.5-2 weeks | Medium | Database migration in maintenance window; plan rollback |
| 5 | 2 | 1.5-2 weeks | Low | New table; no migration risk |

**Total Estimated Duration:** 6-8 weeks (Wave 1: 3-4 weeks parallel; Wave 2: 2-3 weeks sequential)

---

## Success Criteria (End-to-End)

After all 6 specs are implemented:

✅ **Tatkal success rate:** 5% → 70% (users who join queue now complete booking)  
✅ **Filter failure rate:** 15-45% → <5% (consistent experience desktop+mobile)  
✅ **Seat reset rate:** 12-65% → <5% (selections persist)  
✅ **Session expiry support tickets:** 15,000/month → 1,000/month  
✅ **Refund confusion tickets:** 12,000/month → 1,000/month  
✅ **Mobile payment success rate:** 78-82% → 92%+  
✅ **Duplicate charge incidents:** 5,000/month → <50/month  
✅ **Overall booking completion rate:** +25% improvement  
✅ **Customer satisfaction NPS:** +30 points increase  

---

## Staffing and Ownership

**Recommended team composition:**
- **Backend Lead:** 1 (infrastructure, queue system, session management)
- **Frontend Lead:** 1 (UX, component design, mobile optimization)
- **Payment Integration Lead:** 1 (Razorpay integration, idempotency, error handling)
- **DevOps/Infrastructure:** 1 (load balancer, Redis, monitoring)
- **QA Lead:** 1 (regression testing, load testing, UAT)
- **Product Manager:** 1 (prioritization, stakeholder comms)
- **Data/ML Engineer (Part-time):** 0.5 (AI feature training)

**Total FTE:** 5.5 engineers + 1 PM = 6.5 people for 6-8 weeks
