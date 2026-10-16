# Codex Waypoints — Shared School Directory

The single source of truth for "which schools exist and where do their APIs
live". Consumed by every Codex mobile app — **Teacher Codex** today,
**Parent Codex** next, and any future role-specific app.

Served at **<https://waypoints.schoolcodex.com/schools.json>** from Cloudflare
Pages. Every Codex app has that URL hardcoded.

**Schema: v2** (2026-10-16), additive and backward compatible.

---

## Why one shared directory

- **One file, many apps.** Every Codex app has the same first-screen question:
  *which school is this user enrolling at, and what is its API base and
  certificate pin?* A directory per app means double maintenance and a real
  risk of the two drifting, so that one app sees a school the other does not.
- **Cheap and fast.** A static file on an edge CDN over HTTPS.
- **Operations-aligned.** Adding a school is a one-PR change to a small repo
  that a non-engineer can review. Cloudflare Pages deploys on merge.

---

## What's here

| File | Purpose |
|---|---|
| `schools.json` | The directory every Codex app fetches. One entry per deployed school. |
| `README.md` | This file: the schema and the operator checklist. |

---

## `schools.json` schema (v2 — current)

**v2 is additive and backward compatible, deliberately.** `well_known` and
`min_app_version` keep their v1 **types** — plain strings — and their teacher
values. Per-app values live in two new sibling maps that older parsers ignore.

```json
{
  "version": 2,
  "generated_at": "ISO 8601 UTC timestamp — when this file was last edited",
  "schools": [
    {
      "school_id": "stable slug, must match the school's own SCHOOL_ID",
      "name": "human-readable display name",
      "code": "search alias (often same as school_id)",
      "api_base": "https://<school-host>/",

      "well_known": "https://<host>/.well-known/teacher-codex.json",
      "well_known_by_app": {
        "teacher": "https://<host>/.well-known/teacher-codex.json",
        "parent":  "https://<host>/.well-known/parent-codex.json"
      },

      "min_app_version": "1.0.0",
      "min_app_version_by_app": {"teacher": "1.0.0", "parent": "1.0.0"},

      "apps": ["teacher", "parent"],
      "cert_pins": ["sha256/<current-leaf>", "sha256/<next-leaf>"],
      "logo_url": "https://<host>/static/logo.png"
    }
  ]
}
```

### ⚠️ Keep the flat fields as strings

`well_known` and `min_app_version` **must stay plain strings**. Do not turn
either into a per-app object, however tidy that looks — a shipped app parses
them as strings and will not survive the change. Per-app values go in the
`_by_app` maps, which is what they are for.

Enforced on the client side: the Teacher Codex test suite parses the entry
below with the shipped parser class and fails if either type changes.

### How a client reads it

Read your own key out of the `_by_app` map and fall back to the flat field.
Two lines, and it works against v1, v2, and a rollback to v1.

```dart
final byApp = json['well_known_by_app'] as Map<String, dynamic>?;
final wellKnown = (byApp?['parent'] as String?) ?? json['well_known'] as String;
```

`version` is informational. No shipped client reads it; it is a signal to
operators and future parsers, not a gate.

### Field rules

- **`school_id` is load-bearing.** It must match the school's own configured
  id and the `school_id` in that school's `/.well-known/<app>-codex.json`.
  Drift between the two is exactly what the cross-check below detects.
- **`api_base` and `well_known` must be HTTPS and the same host.** A client
  refuses the entry otherwise.
- **`cert_pins` carries both the current and the next pin during a rotation
  window** (60-day cadence, leaf certificates). A client accepts a connection
  whose observed leaf matches any pin in the list. Drop the old pin once the
  window closes.
- **`min_app_version` is the directory's opinion only.** Each school's server
  enforces its own minimum from the app's version header and answers `426`.
  That is the gate that actually stops an old build.
- **`apps` is advisory.** A client may hide a school it is not listed for, but
  the school's own `/.well-known/<app>-codex.json` is the authority — a school
  that does not serve a parent document cannot be onboarded by Parent Codex
  whatever this array says.

### Cross-check semantics — the integrity guarantee

A Codex app:

1. Downloads `schools.json`.
2. Lets the user pick a school.
3. Fetches that school's `well_known` document and verifies its `school_id`,
   `api_base` and `cert_pins` match the directory entry.
4. **Refuses onboarding** on any mismatch.

So tampering with a directory entry cannot point an app at someone else's API:
an attacker would have to control the school's own domain as well. It catches
typo'd directory PRs for the same reason.

### Not yet signed

v1 and v2 carry no signature; the cross-check above is the whole integrity
story. Signing is planned as its own change — a detached Ed25519 signature over
the `schools` array from an offline key, with each app carrying the public key
and refusing a directory that does not verify. It replaces the cross-check
rather than supplementing it, which is why it is being done deliberately rather
than quickly.

---

## `schools.json` schema (v1 — historical)

v1 is still a valid document and a valid rollback target. It is v2 without the
two `_by_app` maps and the `apps` array.

```json
{
  "version": 1,
  "generated_at": "ISO 8601 UTC timestamp",
  "schools": [
    {
      "school_id": "stable slug",
      "name": "human-readable display name",
      "code": "search alias",
      "api_base": "https://<school-host>/",
      "well_known": "https://<school-host>/.well-known/teacher-codex.json",
      "cert_pins": ["sha256/<current-leaf>", "sha256/<next-leaf>"],
      "logo_url": "https://<school-host>/static/logo.png",
      "min_app_version": "1.0.0"
    }
  ]
}
```

A client reading v1 sees no `well_known_by_app`, so an app other than Teacher
Codex should treat that school as **not yet available** rather than as an
error.

---

## Operator checklist

- [ ] **Adding a school.** One PR adding one entry to the `schools` array.
      Confirm the school actually serves its `/.well-known/<app>-codex.json`
      before listing that app in `apps`.
- [ ] **Certificate rotation**, every 60 days. Open a PR adding the *new* pin
      alongside the existing one. Remove the old pin after the window closes.
      Never leave the array empty for a school in production.
- [ ] **Raising `min_app_version`** after a breaking API change. Remember the
      school's own server is the enforcing gate; this field is advisory.
- [x] **Adding a new Codex app** — done for Parent Codex (v2, 2026-10-16). A
      third app needs no schema change: give it a key in `well_known_by_app`
      and `min_app_version_by_app`, and add it to each school's `apps` array.
- [ ] **Changing the hostname.** Don't. Every app has
      `waypoints.schoolcodex.com` compiled in, so changing it forces an app
      update through both stores. It is treated as load-bearing.
