# Prompt to production: reference

## State file

`.prompt-to-production/state.json`, at the repo root, committed with the code so
the log survives across sessions and machines.

```json
{
  "admin": {
    "userId": "user_2abc123",
    "label": "Owner account",
    "currentUserExpr": "session.user.id"
  },
  "gate": {
    "helper": "src/lib/preview.ts",
    "env": "PREVIEW_ADMIN_USER_ID"
  },
  "features": [
    {
      "id": "new-homepage",
      "title": "New homepage",
      "status": "preview",
      "created": "2026-01-15",
      "released": null,
      "gates": ["src/app/page.tsx", "src/app/api/hero/route.ts"],
      "migrations": {
        "applied": ["0007_add_hero_variants.sql"],
        "onRelease": ["0008_drop_legacy_hero_copy.sql"]
      },
      "cleanup": ["src/components/Homepage.tsx", "src/lib/legacy-hero.ts"],
      "notes": "Hero copy now comes from hero_variants. Legacy copy column is unused by the new path."
    }
  ]
}
```

| Field | Meaning |
|---|---|
| `admin.userId` | The one user id that passes the gate. Mirrors the env var. |
| `admin.currentUserExpr` | How this codebase reads the current user id, so every gate is written the same way. |
| `gate.helper` | Path of the `isPreview` helper. |
| `gate.env` | Name of the env var the helper reads. |
| `features[].id` | Kebab-case id used as the literal in every `isPreview` call. |
| `features[].status` | `preview`, `released`, or `discarded`. |
| `features[].gates` | Files that contain an `isPreview('<id>')` call. |
| `features[].migrations.applied` | Additive migrations already run in production. |
| `features[].migrations.onRelease` | Migrations written but held until release, in run order. |
| `features[].cleanup` | Old-path files and code to delete at release. |
| `features[].notes` | Anything the next session needs: decisions, leftovers, caveats. |

If the user does not want the admin id in git, leave `admin.userId` out and keep
only `gate.env`. The helper never reads the state file at runtime.

Keep released and discarded entries. They are the history of what shipped, they
are what the status view lists under "Live for everyone", and they stop an id
from being reused for a different feature.

## Worked example

**Prompt:** "Prompt to production: new onboarding screen with a goals step."

1. Add `new-onboarding-screen` to the state file with `status: "preview"`.
2. Create `OnboardingV2` beside the current `Onboarding`. Gate at the route:

   ```tsx
   export default async function OnboardingPage() {
     const user = await getUser()
     if (isPreview('new-onboarding-screen', user?.id)) return <OnboardingV2 />
     return <Onboarding />
   }
   ```

3. The goals step needs storage. Write `0012_add_user_goals.sql` (new table,
   nothing existing changes), apply it, list it under `migrations.applied`.
4. The new save endpoint checks the gate itself:

   ```ts
   export async function POST(req: Request) {
     const user = await getUser()
     if (!isPreview('new-onboarding-screen', user?.id)) {
       return new Response(null, { status: 404 })
     }
     // ...save goals
   }
   ```

5. The old flow wrote `users.onboarding_step`. The new flow does not need it, so
   write `0013_drop_onboarding_step.sql` and list it under
   `migrations.onRelease`. Do not run it yet: the old path still reads that
   column for every other user.
6. Deploy. Tell the user to sign in as the admin and open `/onboarding`.

**Later:** "New onboarding screen is ok to push to production."

1. Grep `isPreview('new-onboarding-screen'` and confirm it matches `gates`.
2. Route renders `<OnboardingV2 />` unconditionally. Endpoint loses its gate
   check. Delete `Onboarding` and its helpers from `cleanup`.
3. Show `0013_drop_onboarding_step.sql` to the user, apply it.
4. Typecheck, test, build. Grep returns nothing.
5. `status: "released"`, set `released`, commit, deploy.

## Where the gate goes

**Server-rendered or API-backed apps.** Gate on the server where the user is
already resolved (route handler, server component, loader, controller). The
client receives only the path it is allowed to see.

**Client-only web apps.** The bundle ships both paths to everyone, so the gate
only hides the feature. That is fine for layout and copy. Anything that reads or
writes data needs the check repeated in the API it calls. Expose the admin id to
the client through the bundler's public env prefix (for example
`VITE_PREVIEW_ADMIN_USER_ID`), and treat it as public.

**Mobile apps.** Same as client-only, with one difference: the gate ships in a
binary. A release build with the gate removed has to go out through the store or
an over-the-air update before "released" is true for users. Record which one in
`notes`.

**Background jobs and webhooks.** There is no current user. Key the gate on the
user the job is acting for, and skip the new behaviour when that is not the
admin.

## When this pattern is the wrong tool

- The change cannot be additive (a rewrite of a shared table, a breaking API
  change for existing clients). Use a staging environment or a real migration
  plan.
- The feature needs feedback from several people or a percentage rollout. Use a
  feature flag service. This skill is a single-user gate by design.
- The feature only makes sense for logged-out visitors. There is no user id to
  gate on. Use a preview deployment instead.
