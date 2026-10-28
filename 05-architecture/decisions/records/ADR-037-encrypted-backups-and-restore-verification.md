# ADR-037 — Encrypted backups with an automated restore test

- **ID:** ADR-037
- **Date:** 2026-10-28
- **Status:** Proposed
- **Authors:** FaceAttend EDU Team
- **Related:** ADR-016, ADR-026, ADR-031 (release gate); `FULL` PR #303 and #304.

---

## Context

There was no evidence of encrypted backups of PostgreSQL and MongoDB nor of a restore ever being tested (finding OPS-01, high). The data is already encrypted field by field, but the backup file itself was not, and a backup that has never been restored is not a backup.

---

## Decision

**We decided:**
1. `database/scripts/backup.sh` and `restore.sh` take the container names as parameters and **encrypt the backup with a passphrase** (`BACKUP_PASSPHRASE`, OpenSSL) when it is set.
2. `database/scripts/verify-restore.sh` proves the cycle with throwaway containers (`fae-vr-*`, no published ports): seed data, encrypted backup with no plaintext inside, restore into empty containers, compare the data, and check that a wrong passphrase is rejected. It waits for the real end of MongoDB's init (`init process complete`), not just a ping.

**Status is Proposed** because the decision is only half applied: encryption is optional and backups are not yet scheduled nor stored off the server.

**Proposed to accept once:**
- encryption is mandatory in production (the script refuses to run without `BACKUP_PASSPHRASE`);
- backups run on a schedule and are copied off the EC2 host;
- a restore is rehearsed on the production server and timed (RTO), and the backup interval sets the RPO.

---

## Evaluated alternatives

| Alternative | Pros | Cons | Reason for discarding |
|---|---|---|---|
| Scripts with OpenSSL encryption and a restore test — chosen | No new service; testable anywhere with Docker | Passphrase to manage; scheduling still manual | — (chosen) |
| Managed backups only (Neon point-in-time restore) | Nothing to run | Does not cover MongoDB or MinIO | Partial |
| Volume snapshots of the EC2 disk | Simple | Not application-consistent; restore is all-or-nothing | Complement, not a replacement |

---

## Consequences

**Positive:** a restore is proven in about a minute on any machine with Docker; backups no longer leak data if the file is copied.

**Negative / Trade-offs:** losing the passphrase makes every backup unreadable, so it must be kept apart from the backups (like the encryption keys of ADR-016).

---

## Risks

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Backups not run in practice | Medium | High | Schedule them and alert on a missing backup |
| Passphrase lost | Low | High | Store it in the secrets manager, separate from the backups |

---

## References

- Audit finding OPS-01; `FULL` PR #303 (encryption and restore test) and #304 (MongoDB init wait), both run on two machines.
- Related to: ADR-016, ADR-026, ADR-031.
