# Git Conventions

> **Read this document before making your first commit on the project.**

## Branch strategy

```
main        ← Production. Merge from release only. Always stable. (current: refactor branches diverge from main until PR)
  └── dev   ← Continuous integration. Merge from features (create when first feature merges; until then main is dev)
        └── feat/[description]    ← One branch per feature/user story
        └── fix/[description]     ← One branch per bugfix
        └── docs/[description]    ← Documentation only (e.g., docs/remove-audit-bounded-context)
        └── chore/[description]   ← Infrastructure, docs, dependency changes (alternative for docs)
        └── refactor/[description]← Code/structure refactoring without behavior change (allowed variant)
        └── hotfix/[description]  ← Urgent fixes directly to main
```

**Rules:**
- Nobody commits directly to `main` or `dev`
- Every task = one branch + one Pull Request
- One branch = one task (do not mix different features)
- Branches are deleted after merge

---

## Branch naming format

```
[type]/[description-in-kebab-case]

Allowed types: feat, fix, docs, chore, refactor, hotfix

Examples:
feat/oauth2-login
fix/schedule-overlap-calculation
docs/remove-audit-bounded-context
docs/improve-00-01-context-governance
chore/update-spring-dependencies
refactor/remove-audit-domain
hotfix/null-token-expiration
```

---

## Commit format (Conventional Commits)

```
[type]([scope]): [lowercase description, imperative mood, no trailing period]

[optional body — explain WHY, not what]

[optional footer — issue/user story references]
```

**Types:**
| Type | When to use |
|------|-------------|
| `feat` | New functionality |
| `fix` | Bug fix |
| `docs` | Documentation only (preferred for 00/01/02/03) |
| `style` | Formatting, whitespace (no logic change) |
| `refactor` | Code refactoring without behavior change (allowed for domain renames, e.g., refactor/remove-audit-domain) |
| `test` | Add or modify tests |
| `chore` | Tooling, dependencies, CI |
| `perf` | Performance improvement |

**Examples:**
```
feat(iam): implement JWT login

fix(scheduling): correct schedule overlap validation
Closes #42

docs(api): update actor service OpenAPI contract

chore(deps): upgrade Spring Boot to 3.2.0
```

---

## Pull Request policy

- **Size:** maximum 400 lines of code (excluding tests). If larger, split it.
- **Reviewers:** minimum 1 approval before merging
- **Review time:** reviewer has a maximum of 24 business hours
- **Template:** use the template at `.github/pull_request_template.md`
- **Green CI:** merge only proceeds if all pipeline checks pass

---

## Merge policy

- Use **Squash and Merge** for features (keeps `dev` history clean)
- Use **Merge Commit** for releases to `main` (preserves full history)
- **Do not** use Rebase & Merge (creates confusion in shared history)

---

## Tags and versioning

Follow [SemVer](https://semver.org/): `MAJOR.MINOR.PATCH`

```bash
# When releasing to production
git tag -a v1.2.0 -m "Release v1.2.0: add reports module"
git push origin v1.2.0
```
