# Address Validation Pipeline: Multi-Source AI-Orchestrated Triage

**Status:** Live in production since March 2026 (v5)
**Role:** Sole engineer
**Employer:** Triquetra Health

> This is a case study describing proprietary work. Production code is not published. For code demonstrating the underlying pattern, see the [multi-source-triage-engine](../multi-source-triage-engine) sample repository.

## The Problem

Roughly 20 orders per day were being flagged by ShipHero's address hold system. Each hold blocked the order from shipping until a CX agent could review the address, contact the customer if needed, and either release the hold or cancel and refund.

The cost was three-fold:

1. **CX team hours:** each hold took 5 to 15 minutes to resolve, plus the context-switch cost of interrupting other work
2. **Customer experience:** delayed shipments and repeat "please confirm your address" emails
3. **Missed edge cases:** repeat customers whose addresses had shipped successfully many times were still being held every single order, because the hold logic did not know they were repeat customers

The initial "fix" (a Google Apps Script that pulled ShipHero holds into a Sheet and let CX agents work through them faster) was a productivity band-aid. It did not reduce the number of holds. It just made processing them slightly less painful.

## The Approach

I rebuilt the workflow as a multi-source triage pipeline that decides autonomously how to route each flagged address, and only escalates to human review when the decision is genuinely ambiguous.

The pipeline runs three checks in order:

1. **Repeat-customer heuristics (fast path).** If the customer has shipped to this exact address before, or has an order history that indicates a trusted-customer pattern, release the hold immediately. Approximately 45 percent of flagged orders resolve here without any external API call.
2. **USPS DPV validation (primary).** For US addresses, call the USPS Address Information API. USPS returns Delivery Point Validation (DPV) codes that indicate whether the address is deliverable, undeliverable, or ambiguous. Deliverable addresses release the hold. Undeliverable addresses trigger a customer email.
3. **Google Maps geocoding (fallback).** For USPS-ambiguous results, non-US addresses, or edge cases where USPS returns unexpected responses, fall back to Google Maps Geocoding API. If Google Maps returns a high-confidence geocode with a matching country and postal code, release the hold. Otherwise, escalate to human review.

Every decision writes to a structured audit log with the source, confidence level, response payload, and final action.

## Design Decisions Worth Naming

**Iterated through five versions before landing.** The first three versions were single-source (USPS only, then Google Maps only, then USPS-primary-with-manual-review). Version 4 attempted USPS-primary-with-Google-fallback but had a fundamental architecture flaw: on ambiguous USPS results, it queued a customer email while simultaneously calling Google Maps, so customers received "please confirm your address" emails for addresses that Google Maps was about to confirm as valid. I caught this in staging by running production replay data through it. Version 5 introduced strict sequential dependency: no customer-facing action fires until the full triage chain resolves.

**Repeat-customer heuristics were added after production data showed the pattern.** The first three versions treated every order as a first-time order. Looking at three weeks of production logs, I saw that ~45 percent of holds were on customers with clean shipping history. The heuristics layer was a direct response to that data, not a preplanned design.

**Human-in-the-loop for genuine ambiguity.** The pipeline is opinionated: if it is not confident, it does not act. Ambiguous results go to a dedicated Slack channel with the full context payload, so a CX agent can make the call with all the data visible.

**Complete audit trail on every decision.** Each decision writes a row to a Sheets tab with 12 fields: timestamp, order ID, customer ID, source used, USPS response, Google Maps response, heuristics matched, final action, actor (system vs human), CX agent (if applicable), resolution time, notes. This is the primary artifact when anything needs to be debugged.

## Tech Stack

- **Runtime:** Google Apps Script (V8 engine)
- **APIs:** USPS Address Information API v3, Google Maps Geocoding API, ShipHero GraphQL (for hold release), Slack Web API (for escalation notifications)
- **Storage:** Google Sheets (audit log)
- **Trigger:** Scheduled every 15 minutes; also on ShipHero hold-created webhook
- **Auth:** OAuth 2.0 (USPS, Google Maps), Bearer token (ShipHero), Slack Bot Token

## Impact

- **20+ flagged orders per day** processed autonomously
- **~45 percent** of flagged orders resolve at the heuristics layer with zero external API calls
- **~40 percent** resolve through USPS DPV validation
- **~10 percent** require Google Maps geocoding
- **~5 percent** escalate to human review
- **Estimated 5 to 8 hours per week** of CX team time reclaimed
- **Zero duplicate customer emails** since v5 deployment (the v4 architecture flaw would have caused ~4 per day)

## Lessons Learned

**1. Production data changes the design.**
The first three versions were designed on assumptions about hold patterns. The heuristics layer was born from three weeks of production logs. Design decisions should always be revisited against real data, not proxy data.

**2. Sequential dependency matters when actions have real-world consequences.**
Parallel execution is a tempting optimization, but for anything that touches a customer (an email, a charge, an order modification), the safest default is strict sequential execution with a full "should we act" decision before the "act" step. The v4 architecture flaw would have been embarrassing in production.

**3. Confidence thresholds are policy, not code.**
USPS returns DPV codes as strings (Y, N, S, D). Google Maps returns location_type as strings (ROOFTOP, RANGE_INTERPOLATED, GEOMETRIC_CENTER, APPROXIMATE). Which of these count as "confident enough to auto-release" is a policy decision, not a technical one. That policy was written down explicitly in the run book and agreed with the CX team before v5 shipped.

**4. Audit logs are the primary artifact.**
When something goes wrong (a customer complains, an address gets held incorrectly, a shipment goes to the wrong place), the audit log is where the debugging starts. Building the log first, and populating it on every decision including automated ones, was the single most impactful design choice.

## What's Next

- Add international address validation (currently the fallback path handles non-US addresses in a general way; a country-specific validation layer would improve confidence)
- Add ML classification on the audit log to identify emerging hold patterns before they become significant
- Extend the pattern to fraud-signal triage (already partially implemented in the [Fraud Detection Automation](../fraud-detection-automation))

## References

- USPS Address Information API v3: [https://developer.usps.com/apis](https://developer.usps.com/apis)
- Google Maps Geocoding API: [https://developers.google.com/maps/documentation/geocoding](https://developers.google.com/maps/documentation/geocoding)
- ShipHero GraphQL API: [https://developer.shiphero.com](https://developer.shiphero.com)
