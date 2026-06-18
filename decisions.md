# Decision Log
**Candidate:** Ahmed Tarek | **Exercise:** CISPA Take-home Design Exercise | **Date:** 2026-06-18

---

## Anomalies Found

**Anomaly 1  T-9999: Sentinel test row (Source A)**

Row `Q-6999 / T-9999` in Source A has `traveler_name = TEST TEST`, `dest_city = TEST`, `depart_date = 2099-01-01`, and `quote_amount = 1 EUR`. This is clearly not a real trip. It doesn't exist in Source B or C at all.

Rule: Reject at schema validation using a date-range guard  any departure date beyond current year + 2 gets rejected immediately with flag `INVALID_TEST_RECORD`. It never reaches the reconciliation step.

---

**Anomaly 2  T-1078: Approved and insured trip with no booking quote (Sources B and C only)**

T-1078 (Yusuf Demir) has a full approval record in Source B and an insurance record in Source C, but there is no corresponding quote in Source A. The trip happened  it was approved and insured  but the booking agency never issued a quote for it, or the quote was lost.

Rule: Treat as a partially reconciled record. Include it in the unified view with an `incomplete_sources` flag noting Source A is absent. Route a copy to the human-review queue so a data owner can confirm whether the quote exists elsewhere or the trip was booked outside the agency.

---

**Anomaly 3  T-1078: Non-standard booking reference `BK-EXT-77` (Source C)**

The `booking_ref` for T-1078 is `BK-EXT-77`, while every other record follows the `BK-{number}` pattern. Combined with the missing Source A quote, this strongly suggests the trip was booked directly with an external provider rather than through the internal agency.

Rule: Apply a regex check (`^BK-\d+$`) at validation. Any value that fails is flagged as `BOOKING_REF_FORMAT_ANOMALY` and routed to human review. The record is not discarded  it stays in the pipeline with the flag attached.

---

**Anomaly 4  T-1091: Same approver name, two different IDs in approval chain (Source B)**

In T-1091's approval chain, both Step 1 and Step 2 are attributed to `M. Schmidt`, but with different `approver_id` values: `u-2305` and `u-2412`. This is suspicious  either there are two people named M. Schmidt in the system, or one person has a duplicate account, or it was a data entry error.

Rule: Flag the approval chain as `APPROVER_IDENTITY_CONFLICT` and route to human review. The trip itself is still treated as approved since both decisions are `approved`  but the identity issue needs to be resolved by the data owner before the record is fully trusted.

---

**Anomaly 5  T-1102: Two Source B records for the same trip with conflicting status (A-9106 and A-9107)**

Source B contains two approval records for T-1102 (Greta Vogt): `A-9106` with `status: approved` and `A-9107` with `status: delegate_recorded`. Both share the same `trip_ref`, same dates, and same employee  they clearly refer to the same trip.

Rule: Apply a status priority ranking: `approved` > `delegate_recorded` > `pending`. The `approved` record (A-9106) becomes the canonical approval. The `delegate_recorded` record (A-9107) is retained in a separate `approval_audit` table as supplementary context  not discarded, but not used as the primary record either.

---

**Anomaly 6  T-1078: Traveller name casing mismatch across sources**

Source B stores the name as `Yusuf Demir` (title case) while Source C stores it as `YUSUF DEMIR` (all caps). These refer to the same person, and the trip IDs confirm it  but a naive exact-match would fail to link them.

Rule: Normalize all traveller names to title case at ingestion, before any matching happens. Cross-source identity matching uses `trip_id` as the primary key; name matching is a secondary validation. If `trip_id` aligns but names differ by more than casing, flag as `NAME_MISMATCH` for human review.

---

## Conflict Resolution Rule

**When sources disagree on trip dates, Source B wins.**

Source A is a quote  it reflects what the travel agency proposed before any internal decision was made. Source B is the approval record  it reflects the dates the organization actually authorized the trip to happen. Source C has consistently matched Source B's dates in this dataset, which supports B being the authoritative source for trip timing.

So the rule is: for `depart_date` and `return_date`, Source B is canonical. Source A's dates are stored as `quoted_dates` in the unified record for audit purposes but are not used as the trip's official dates. If B and C ever disagree on dates, that conflict is escalated to human review rather than resolved automatically  because both represent post-approval records and neither is clearly more reliable than the other.

---

## Scaling: What Happens When Source D Arrives

**What wouldn't need to change:**

The canonical schema, conflict-resolution rules, validation logic, deduplication rules, the human-review queue, and the unified output layer are all source-agnostic. They don't know or care how many sources exist  they operate on normalized records regardless of origin. The status vocabulary, the trust hierarchy (Source B canonical for dates), and the `incomplete_sources` flagging mechanism would all apply to Source D without any modification.

**What would change:**

The only thing that needs to be written is a new source connector  a config that declares Source D's field mappings, file format, encoding, and parser settings. Beyond that, one important design question would need to be answered upfront: where does Source D sit in the business process  before or after approval? A source that captures pre-approval data (like Source A) gets a lower trust level than a post-approval source (like Source B or C). If this is declared in the connector config, the pipeline automatically assigns the right trust level without any rule changes. This is the key design principle that keeps the system extensible.
