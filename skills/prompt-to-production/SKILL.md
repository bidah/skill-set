---
name: prompt-to-production
description: Ship features straight to production behind a gate that only the admin test user passes, track each one in a local feature log, and release it to everyone on approval by removing the gate and applying pending migrations. Use when the user says "prompt to production", wants to try a feature in production before anyone else sees it, asks for an admin-only preview or a single-user A/B test, asks what is in preview or what is live, or says a previewed feature is "ok to push to production", "ship it to everyone", or should be discarded.
---

# Prompt to production

Every new feature goes to production immediately, but only the **admin user**
sees it. Everyone else keeps the current behaviour. When the admin approves, the
gate comes out and the feature is live for all users.

Three pieces make this work:

1. **State file** `.prompt-to-production/state.json`: who the admin is, where the
   gate lives, and one entry per feature. This is your memory. Read it at the
   start of every session before touching a gated feature.
2. **Gate helper**: one function, `isPreview(featureId, userId)`.
3. **Feature id**: a kebab-case literal such as `new-homepage` or
   `new-onboarding-screen`. It appears in the state file and in every gate, so one
   grep finds all the code a feature touches.

## Setup (once per project)

Run this when `.prompt-to-production/state.json` does not exist.

1. Ask the user which account is the admin. Record the stable user id, not an
   email or display name, and the expression the app uses for the current user id
   (for example `session.user.id`).
2. Add the gate helper in the project's language. The admin id comes from an env
   var so each environment can point at its own account:

   ```ts
   // src/lib/preview.ts
   const ADMIN_ID = process.env.PREVIEW_ADMIN_USER_ID

   export function isPreview(featureId: string, userId?: string | null): boolean {
     void featureId // kept so every gate names its feature and stays greppable
     return Boolean(ADMIN_ID) && userId === ADMIN_ID
   }
   ```

3. Set the env var in production through the project's normal env tooling. Ask
   before changing production configuration.
4. Write the state file and commit it. [REFERENCE.md](REFERENCE.md) has the full
   schema.

## Build a feature in preview

1. Pick a feature id. If the user gave a name ("new homepage"), kebab-case it. If
   the id already exists in the state file, continue that feature.
2. Add the entry to the state file with `status: "preview"` **before** writing
   code, then keep `gates`, `migrations`, and `cleanup` current as you work.
3. Build the new path next to the old one and branch at the highest sensible
   point (a route, a screen, a handler), not deep inside shared components:

   ```tsx
   if (isPreview('new-homepage', user?.id)) return <NewHomepage />
   return <Homepage />
   ```

4. Follow the gate rules below, then ship through the project's normal deploy
   path and tell the user what to check while signed in as the admin.

### Gate rules

- Always call `isPreview` with a **string literal** id. Never compare user ids
  inline and never build the id dynamically.
- No user, or any other user, gets the old path. The old path stays untouched and
  working until release.
- Enforce on the server. Hiding UI on the client is not a gate: any endpoint,
  action, or query the feature adds must check `isPreview` itself.
- Migrations during preview are **additive only**: new tables, new nullable
  columns, new indexes. Apply them when the preview ships and list them under
  `migrations.applied`. Anything the old path could not survive (drops, renames,
  `NOT NULL`, backfills that change old behaviour) goes under
  `migrations.onRelease` and waits.

## Release ("X is ok to push to production")

The user's approval covers the feature they named, nothing else in preview.

1. Find the entry in the state file. If the name is ambiguous, list the features
   in preview and ask which one.
2. `grep -rn "isPreview('<id>'"` across the repo. Compare the hits with `gates`;
   investigate any difference before editing.
3. At each gate, make the new path unconditional and delete the old path. Then
   remove everything in `cleanup` and any code that is now unreachable.
4. Show the user the `migrations.onRelease` list, then apply it with the
   project's migration tooling in the order recorded. Stop if one fails; do not
   deploy code that depends on a migration that did not run.
5. Run the project's typecheck, tests, and build. The grep from step 2 must now
   return nothing.
6. Set `status: "released"` and the `released` date, commit, and deploy.

## Status ("what's in preview?", "what's live?")

Read the state file and reply in plain sentences with two short markdown lists,
newest first. Never answer from memory of the session: the file is the source
of truth. No tables and no code block; write it the way you would say it:

> Two features are in preview right now. Only you can see them:
>
> - **new-homepage** has been in preview since 2026-01-15. It has 1 gate and 1
>   migration waiting for release.
> - **new-onboarding-screen** has been in preview since 2026-01-12. It has 2
>   gates and nothing waiting, so it is ready to release.
>
> Two features are live for everyone:
>
> - **pricing-table**, released 2026-01-10.
> - **weekly-digest-email**, released 2026-01-03.

- For each preview entry give the gate count and either how many
  `migrations.onRelease` are still waiting or that it is ready to release.
- Before replying, grep each preview id and say so if the hits no longer match
  `gates`.
- Mention discarded features only when the user asks for history.

## Other requests

- **"Drop X"** Remove the new path and its gates, keep the old path, set
  `status: "discarded"`. Leave additive migrations in place unless the user asks
  to revert them, and record what was left behind in `notes`.
- **Admin changed** Update the state file and the env var together.

See [REFERENCE.md](REFERENCE.md) for the state file schema, a worked example, and
gate placement for server, client-only, and mobile apps.
