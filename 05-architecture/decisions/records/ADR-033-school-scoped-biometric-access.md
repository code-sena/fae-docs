# ADR-033 — Biometric access and matching are scoped to the caller's school

- **ID:** ADR-033
- **Date:** 2026-10-28
- **Status:** Accepted
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-002 (schema-per-context, no cross-context FK), ADR-003, ADR-024, ADR-026; `FULL` PR #280, #288 and #294.

---

## Context

Several schools share one deployment. The biometric service authorised by role only: an Instructor of school A could read, delete or match the templates of school B (findings N-02 and N-12, high). The 1:N `/identify` scanned every active template of every school, and `/verify` accepted any `person_id`; the WebSocket channel called the same handlers.

---

## Decision

**We decided:** every biometric operation is limited to the school of the caller; only the Super Administrador crosses schools.

1. **Per-person routes** (read, history, delete, summary): the caller must have the person in scope (`ensure_staff_scope`); responses never contain the decrypted vector or the raw fingerprint sample.
2. **1:N identification:** among the candidates above the threshold, the first that belongs to the caller's school wins (`MatchScope` / `best_allowed`, up to 5 candidates); a face or fingerprint of another school answers "no match", exactly like an unknown one, so the answer does not reveal that the person exists elsewhere.
3. **1:1 verification:** a person of another school answers "no template".
4. **WebSocket:** the same scope is passed in (`match_scope` is a required argument), and broadcasts go only to the school of the event.
5. An Instructor with no academic actor (no school) cannot identify: every kiosk must use an account linked to a school.

**Justification:** school isolation is a business rule, not a data-model one (ADR-002 forbids cross-context FKs), so it is enforced in the service that owns the templates, at the point where the match is decided.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Filter candidates by school after matching — chosen | No change to the stored documents; one place to enforce | A slightly larger scan | — (chosen) |
| Store `school_id` in each template and filter in the query | Smaller scan | Duplicates a cross-context fact that changes (transfers) | Staleness risk; not needed at current volume |
| One MongoDB collection per school | Strong isolation | Operational overhead, migrations per school | Out of proportion for the MVP |
| Rely on the UI to show only own-school people | None | Trivially bypassed through the API | Not a control |

---

## Consequences

**Positive:** closes N-02 and N-12; a matched face of another school is indistinguishable from an unknown face.

**Negative / Trade-offs:** the scope is computed per request from identity/academic (cached briefly); an account without a school cannot run a kiosk.

**Impact on the system:** `06-ms-biometric` (`match_scope.py`, routers, WebSocket), `shared/auth-*` for the decoded-path checks (#280).

> **Implementation note (2026-11-08, `FULL` PR #395):** `MatchScope` and `best_allowed` moved to `application/match_selection.py` and the facial flows to `application/facial_services.py`; `match_scope.py` re-exports them, so the routers and the WebSocket channel keep the same entry points. The rule is unchanged.

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| A new biometric route forgets the scope | Medium | High | Tests per route (`test_match_scope.py`, `test_person_scope.py`); review checklist |
| A person who changes school keeps old matches in logs | Low | Low | Logs expire after 90 days (ADR-026) |

---

## References

- Audit findings N-02, N-12, S-NEW-02, S-NEW-04; `FULL` PR #280, #288, #294 (tested end to end through Kong with two schools).
- Related to: ADR-002, ADR-024, ADR-026.
