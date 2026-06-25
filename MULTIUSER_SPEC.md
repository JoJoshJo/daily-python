# MULTIUSER_SPEC.md — add multi-user support to "Python, One Day at a Time"

> Hand this to Claude Code in the same repo, after BUILD_SPEC.md has already been implemented.
> Instruction to give it:
> **"Read MULTIUSER_SPEC.md and apply it to the existing index.html. Keep every screen, lesson, and behavior from BUILD_SPEC.md exactly as is — this only changes how progress is stored and identified. Commit when done."**

## 1. The problem

The app currently stores one learner's progress under a single localStorage key. If a second person opens the same browser and types their name, they'd see (or silently overwrite) the first person's progress. We need multiple people to share one device, each identified by the name they type, each with their own separate, persistent progress.

This stays name-only — no password. That's an intentional, acceptable tradeoff for a small group of family/friends sharing a device, not a security boundary. Don't add auth beyond this.

## 2. New storage shape

Replace the single `pyjourney_v1` key with three keys:

```js
// One entry per person, keyed by normalized name.
// localStorage key: "pyjourney_users_v1"
{
  "josh": { displayName: "Josh", startDate: "2026-06-24", completed: [1,2,3], streak: 3, lastDate: "2026-06-24" },
  "joe":  { displayName: "Joe",  startDate: "2026-06-25", completed: [],      streak: 0, lastDate: null }
}

// Which profile is currently active in this browser, so it doesn't ask every visit.
// localStorage key: "pyjourney_current_v1"
"josh"   // a single string, the normalized key, or absent if no one is active

// Every name that's ever been used on this device, for the suggestion chips in step 4.
// Can be derived from Object.keys(pyjourney_users_v1) directly — no separate key needed.
```

**Normalizing a name:** trim whitespace, collapse internal multiple spaces to one, lowercase the result. That normalized string is the lookup key. Keep the original (trimmed, not lowercased) typed casing as `displayName` for greetings — e.g. typing "joe " or "Joe" or "JOE" all resolve to the same profile, but each saves "Joe" as typed the first time and doesn't change `displayName` on later logins unless you want it to (don't bother updating it — first-write wins, keeps it simple).

**One-time migration:** if the old `pyjourney_v1` key exists and `pyjourney_users_v1` does not, migrate it on load: take its `name`, normalize it, create one entry in `pyjourney_users_v1` from its fields, set `pyjourney_current_v1` to that key, then leave the old key alone (don't delete — harmless either way, but no need to clean up). This preserves any progress already made during testing.

## 3. Screen flow changes

**On page load:**
- Run the migration check above first.
- If `pyjourney_current_v1` is set and that key exists in `pyjourney_users_v1` → skip the name prompt, go straight to Home for that profile.
- Otherwise → show the Welcome / name screen.

**Welcome / name screen (now doubles as "log in or create"):**
- Same input and copy as before: "What should I call you?"
- Add small suggestion chips below the input, one per existing name in `pyjourney_users_v1` (display each profile's `displayName`), only if there's at least one. Tapping a chip fills the input with that name (don't auto-submit — let them confirm by pressing the button, in case of a near-duplicate they didn't mean to pick).
- On submit: normalize the typed name.
  - If it matches an existing profile → load it, set `pyjourney_current_v1` to that key, go to Home. Show a one-line "Welcome back, {displayName}!" transition instead of treating it as a fresh start.
  - If it's new → create a profile (`displayName` = trimmed input as typed, `startDate` = today, `completed: []`, `streak: 0`, `lastDate: null`), save it, set as current, go to Home as a first-time visit (same as today's existing behavior).
- Empty/whitespace-only input: keep existing validation, don't submit.

**Home screen:**
- Next to the `>>> Hey {name}` greeting, add a small, low-emphasis **"Switch user"** link/button.
- Tapping it clears `pyjourney_current_v1` only (never touches `pyjourney_users_v1`) and returns to the Welcome/name screen, now showing the suggestion chips so the next person can pick their name or a new person can type theirs.

**Reset progress:**
- Now scoped to the current profile only. Update the confirm dialog text to name them, e.g. "This clears {displayName}'s progress on this device. Continue?"
- On confirm: delete that one key from `pyjourney_users_v1` (leave every other profile untouched), clear `pyjourney_current_v1`, return to Welcome.

## 4. Everything else stays the same

All 30 lessons, the predict/run/both practice logic, Pyodide loading and the `input()` bridge, the streak math (§6 of BUILD_SPEC.md, unchanged — it now just reads/writes within the *current* profile's object instead of the single old object), the 30-day journey strip, animations, and design system are untouched. Every place the old code read or wrote the single profile object now reads/writes `pyjourney_users_v1[currentKey]` instead, saving the whole `pyjourney_users_v1` object back to localStorage on every change (same try/catch-with-in-memory-fallback pattern as before).

## 5. Definition of done

1. Fresh browser (no storage): Welcome screen, no suggestion chips, typing a new name creates a profile and proceeds normally.
2. Typing an existing name (any case/spacing) loads that exact profile's progress — completed days, streak, and dots all match what that person left off at. Does not reset anything.
3. Two different names on the same browser have fully independent progress; completing a lesson as one never changes the other's data.
4. Refreshing the page keeps you logged in as the last active profile (no re-prompt).
5. "Switch user" returns to the name screen, shows chips for everyone used on this device, and switching to another name loads their progress correctly. Switching back to the first name still has their progress intact.
6. "Reset progress" only clears the current profile; other profiles on the same device are unaffected.
7. If an old single-profile `pyjourney_v1` key is present with no `pyjourney_users_v1` yet, it's migrated once into the new shape with no data loss, and the app proceeds straight to that person's Home screen.
8. No regressions to any item in BUILD_SPEC.md's own Definition of done (§13).
