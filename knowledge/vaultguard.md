# VaultGuard

## What It Is

VaultGuard is the household secrets store. It is the single location for all actual credentials, account numbers, passwords, seed phrases, and sensitive financial identifiers.

VaultGuard is separately scoped from this system and has its own build plan. This document covers only how the agent system interacts with it.

## The Pointer Convention

`agent-shared/life/` references VaultGuard entries by name only. It never contains the secret itself.

**Correct:**
```markdown
Vanguard rollover IRA — see VaultGuard entry `vanguard_ira_rollover`
```

**Never:**
```markdown
Vanguard rollover IRA — account number: 12345678
```

This applies to every file in `life/` without exception. The life KB is git-tracked in a private repo; VaultGuard is encrypted at rest. The two systems have different security boundaries and must stay separated.

## Risk

Accidentally committing a secret to `life/` is classified as **high impact** (R7 in the risk register), even though `life/` is a private repo. Mitigations:

- Pointers-only convention (this document).
- Pre-commit hook scanning for account-number patterns (planned).

## Scope Boundary

VaultGuard provides secret storage. This system provides knowledge storage. They are not substitutes for each other and must not be conflated.

Agents read `life/` for facts and follow pointer references when they need to surface a VaultGuard entry name to the user — they do not retrieve or handle secrets directly.
