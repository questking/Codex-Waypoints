# Codex Waypoints — Shared School Directory

The single source of truth for "which schools exist and where do their
APIs live". Consumed by every Codex mobile app — currently **Teacher
Codex**, planned: **Parent Codex** and any future role-specific app.

> **This folder is staging.** The artifacts here will be migrated to a
> standalone GitHub repo (`codex-waypoints`) hosted on Cloudflare Pages
> at `directory.codex.app` (or whatever final hostname is chosen — see
> [Hostname](#hostname-decision-deferred) below). Once the standalone
> repo is live, this folder can be deleted from the main monorepo.

---

## Why a separate repo / a single shared directory?

- **One file, many apps.** Both Teacher Codex and Parent Codex (and any
  future Codex app) need to answer the same first-screen question — *"which
  school is this user enrolling at, and what's the API base + cert pin?"*
  Duplicating the directory per-app means double maintenance and a real
  risk of the two drifting (one app sees a school the other doesn't).
- **Free, fast, signable.** Cloudflare Pages serves the file from the
  edge with HTTPS + caching for free. The cross-check against each
  school's `/.well-known/teacher-codex.json` (or `parent-codex.json`)
  makes the directory tamper-evident even without signing in v1.
- **Operations-aligned.** Adding a new school is a one-PR change to a
  small repo a non-engineer can review; redeploys happen on merge.

---

## What's in this folder

| File | Purpose |
|---|---|
| `schools.json` | The directory file every Codex app fetches. One entry per deployed school. |
| `README.md` (this file) | Schema + the operator runbook for creating the standalone repo + Cloudflare Pages site. |

---

## Operator runbook — bring up the standalone repo

1. **Create the GitHub repo** `<org>/codex-waypoints` (public — the schools
   list is not sensitive; it just maps display names → API hostnames +
   cert pins).
2. **Migrate the artifacts** — copy `schools.json` and a slimmed-down
   version of this README into the new repo's root.
3. **Provision Cloudflare Pages** (per the prerequisites task **P.10**):
   - Connect the new repo to a Pages project named `codex-waypoints`.
   - Custom domain: pick one (see below).
   - Build command: none (static).
   - Output directory: `/` (the repo root).
4. **Verify** with `curl https://<chosen-host>/schools.json`.
5. **Adding a new school** (recurring): one PR to the directory repo
   updates `schools.json`; Cloudflare Pages deploys automatically on
   merge.
6. **After verification**: delete the staging copy at
   `school_codex/codex_waypoints/` from this monorepo, since the
   standalone repo is now authoritative.

### Hostname decision (deferred)

D4 in [teacher_codex_app/decisions.md](../teacher_codex_app/decisions.md)
originally pinned the hostname to `directory.teachercodex.app`. With
Parent Codex coming, the hostname should be **app-neutral**. Pick one
when standing up the standalone repo:

- `directory.codex.app` — short, neutral, matches the umbrella brand.
- `waypoints.codex.app` — matches the repo name.
- `directory.codexapps.io` — if `codex.app` isn't owned.

Whatever you pick, every Codex app's hardcoded directory URL must match
it. **Lock this in before the first mobile-app build ships** — changing
the hostname after release means a forced app update.

---

## `schools.json` schema (v1)

```json
{
  "version": 1,
  "generated_at": "ISO 8601 UTC timestamp — when this file was last edited",
  "schools": [
    {
      "school_id": "stable slug, must match the school's SCHOOL_ID setting",
      "name": "human-readable display name",
      "code": "search alias (often same as school_id)",
      "api_base": "https://<school-host>/api/v1/",
      "well_known": "https://<school-host>/.well-known/teacher-codex.json",
      "cert_pins": ["sha256/<current-leaf>", "sha256/<next-leaf>"],
      "logo_url": "https://<school-host>/static/logo.png",
      "min_app_version": "1.0.0"
    }
  ]
}
```

### Field rules

- `school_id` is **load-bearing**. It must match the school's `SCHOOL_ID`
  Django setting (which appears in the JWT `iss` claim) AND the
  `school_id` field in that school's `/.well-known/teacher-codex.json`.
  Drift between the three sources is what cross-checking detects.
- `well_known` currently points at `teacher-codex.json` because Teacher
  Codex ships first. **When Parent Codex lands**, schools will publish
  a sibling `/.well-known/parent-codex.json` with a parallel payload, and
  this field becomes per-app:
  ```json
  "well_known": {
    "teacher": "https://<host>/.well-known/teacher-codex.json",
    "parent":  "https://<host>/.well-known/parent-codex.json"
  }
  ```
  Plan the schema bump as `version: 2` and add a migration note before
  the parent app's first beta.
- `cert_pins` should always carry **both `current` and `next`** pins
  during rotation windows (per **D6** — 60-day cadence, leaf certs, both
  pins live during the rotation window). The mobile app accepts a
  connection if the observed leaf matches **any** pin in the list.
- `min_app_version` is currently a single string. When Parent Codex
  joins, this becomes per-app too:
  ```json
  "min_app_version": {"teacher": "1.0.0", "parent": "1.0.0"}
  ```

### Cross-check semantics (the integrity guarantee for v1)

The mobile app:
1. Downloads `schools.json`.
2. Lets the user pick a school.
3. Fetches `<chosen.well_known>` and verifies its `school_id` + `api_base`
   match the directory entry.
4. **Refuses onboarding** if either is mismatched.

This catches both directory tampering (an attacker would have to also
control the school's domain to forge a matching `.well-known`) and
typo'd directory PRs.

### v2 — optional Ed25519 signing

Not in scope right now. When v2 lands, an additional `signature` field
holds a detached signature over the `schools` array, signed by an offline
Ed25519 key. Each Codex app gains a hard-coded public key and refuses
any directory whose signature doesn't verify.

---

## Maintenance checklist

- [ ] **Cert rotation** (every 60 days, per **D6**): when a school renews
      its TLS cert, open a directory PR with the *new* `next` pin alongside
      the existing `current` pin. After the rotation window closes, drop
      the old pin.
- [ ] **New school onboarding**: one PR adds an entry to `schools` array.
- [ ] **Bumping `min_app_version`**: after force-upgrade is needed (e.g.
      breaking API change), bump the per-school `min_app_version` field.
- [ ] **Adding a new Codex app** (Parent Codex etc.): bump schema to
      `version: 2`, fan out `well_known` and `min_app_version` into per-app
      objects, ship the new app's first beta against the v2 directory.
