# migration-guide

How to take a VietRocHack hackathon project off of a teammate's personal
Firebase/GCP project and put it on the team's own infrastructure, live
forever at `<app>.vietrochack.com`. Written up after doing this for
[RocMap](https://github.com/VietRocHack/RocMap), which is the reference
example throughout.

This is a playbook, not a script. Read it, adapt it, don't blindly copy
commands, especially resource names and project IDs.

## Before you start: find what's actually broken

Hackathon projects almost always have hardcoded references to whoever's
personal cloud project they were demoed from. Find these before touching
anything:

- Frontend fetch calls to an absolute `https://*.cloudfunctions.net/...` or
  `https://*.firebaseapp.com/...` URL
- Backend code that pulls data from a teammate's personal GitHub fork or
  branch instead of the data already committed in the repo
- Image/asset URLs pointing at `firebasestorage.googleapis.com/v0/b/<personal
  project>...`
- A root-level `package.json` or config file left over from a different
  branch/experiment that isn't actually used by the real app

Grep for `cloudfunctions.net`, `firebaseapp.com`, `firebasestorage`,
`raw.githubusercontent.com`, and any hardcoded project ID that isn't the
team's. Read the actual code paths, don't assume from the README.

## Naming and siloing in `vietrochack-lab`

`vietrochack-lab` hosts more than one project. Every app needs to be
scoped so it doesn't collide with, or get confused with, whatever else is
in there.

- **Cloud Functions**: prefix with the app name. `rocmap-findDirection`, not
  `findDirection`.
- **Firebase Hosting**: give every app its own **named Hosting site**, never
  the project's default site. Site IDs are globally unique across *all* of
  Firebase (not just this project), so `<app>` alone will often be taken;
  fall back to something like `vietrochack-<app>`. Map it to a Hosting
  **target** in `.firebaserc` so `firebase deploy --only hosting:<app>` works
  cleanly.
- **Database, if the app needs one**: a dedicated named Firestore database
  per app (`gcloud firestore databases create --database=<app>`), not the
  project's shared `(default)` database.
- **Storage, if the app needs one**: a dedicated bucket per app
  (`vietrochack-lab-<app>`), not a shared bucket with path prefixes.

## Deploy topology: one origin, no CORS

Put the frontend and backend behind the **same domain** using a Firebase
Hosting rewrite, instead of a separate `api.<app>.vietrochack.com`
subdomain:

```json
{
  "hosting": {
    "target": "<app>",
    "public": "<frontend build dir>",
    "rewrites": [
      {
        "source": "/api/**",
        "function": { "functionId": "<app>-<function>", "region": "us-central1" }
      },
      { "source": "**", "destination": "/index.html" }
    ]
  }
}
```

The frontend then calls a relative `/api/...` path instead of an absolute
Cloud Function URL. This removes CORS entirely and means there's only one
DNS record to manage, not two.

**Cloud Functions (gen2) vs Cloud Run**: if the backend is already a single
HTTP function (Python's `functions_framework`, Node's `onRequest`, etc.),
deploy it as a Cloud Function directly, don't rewrite it into a long-running
Cloud Run service for no reason. Cloud Run is the right call when the
backend is a real persistent server (Express app, etc.) or needs something
Functions can't do.

Images and other static, rarely-changing assets that a teammate previously
hosted on Firebase Storage: just ship them as static files through the same
Hosting deploy instead of provisioning a Storage bucket. One less resource
to secure.

**Backend data**: if the backend fetches data over the network from a
teammate's personal fork on every request, bundle that data with the deploy
instead (copy it alongside the function's source, read it from disk). Don't
leave a live app's correctness dependent on someone else's GitHub repo
staying up and unchanged forever.

## Step by step

Assumes `gcloud`, `firebase-tools`, and `gh` are installed and authenticated,
and the app's GCP project already exists under `vietrochack-lab` (or is
`vietrochack-lab` itself, if apps aren't split into separate projects).

```bash
# 1. Enable Firebase on the project (safe, non-destructive, same project ID)
firebase projects:addfirebase vietrochack-lab

# 2. Enable the APIs the deploy needs
gcloud services enable \
  cloudfunctions.googleapis.com run.googleapis.com cloudbuild.googleapis.com \
  artifactregistry.googleapis.com eventarc.googleapis.com \
  billingbudgets.googleapis.com \
  --project=vietrochack-lab

# 3. Create the app's own Hosting site + target
firebase hosting:sites:create vietrochack-<app> --project vietrochack-lab
# then write .firebaserc by hand (firebase target:apply needs an existing
# firebase.json, which you're about to write anyway):
#   "targets": { "vietrochack-lab": { "hosting": { "<app>": ["vietrochack-<app>"] } } }

# 4. Deploy the backend function
gcloud functions deploy <app>-<function> \
  --gen2 --runtime=<runtime> --region=us-central1 \
  --source=<backend dir> --entry-point=<entry point> \
  --trigger-http --allow-unauthenticated \
  --project=vietrochack-lab

# 5. Build and deploy the frontend
cd <frontend dir> && npm run build && cd -
firebase deploy --only hosting:<app> --project vietrochack-lab
```

## Custom domain

`firebase-tools` has no CLI command for custom domains. Use the REST API
directly:

```bash
TOKEN=$(gcloud auth print-access-token)

# Request the domain
curl -s -X POST \
  "https://firebasehosting.googleapis.com/v1beta1/projects/vietrochack-lab/sites/vietrochack-<app>/customDomains?customDomainId=<app>.vietrochack.com" \
  -H "Authorization: Bearer $TOKEN" -H "X-Goog-User-Project: vietrochack-lab" \
  -H "Content-Type: application/json" -d '{}'

# Poll for the exact DNS records it wants
curl -s \
  "https://firebasehosting.googleapis.com/v1beta1/projects/vietrochack-lab/sites/vietrochack-<app>/customDomains/<app>.vietrochack.com" \
  -H "Authorization: Bearer $TOKEN" -H "X-Goog-User-Project: vietrochack-lab"
```

This returns a CNAME (`<app>.vietrochack.com` -> `vietrochack-<app>.web.app`)
and a TXT record (`_acme-challenge.<app>.vietrochack.com`) for cert
validation. Hand those exact values to whoever manages DNS for
`vietrochack.com` (Namecheap, as of this writing) - you likely won't have
access to add them yourself. Poll the same GET afterward:
`hostState` goes `HOST_UNHOSTED` -> `HOST_ACTIVE`, `ownershipState` goes
`OWNERSHIP_MISSING` -> `OWNERSHIP_ACTIVE`. Usually resolves well under 24h
after the records are added.

## Cost safety net

This is a permanent deploy, not a hackathon demo that gets torn down after
judging. Set these up once, not as an afterthought:

```bash
# Budget alert, scoped to just this GCP project (billing accounts are often
# shared across many personal/team projects - don't alert on their combined spend)
gcloud billing budgets create \
  --billing-account=<billing account id> \
  --display-name="<app> budget" \
  --budget-amount=10USD \
  --calendar-period=month \
  --filter-projects=projects/vietrochack-lab \
  --threshold-rule=percent=0.5 --threshold-rule=percent=0.9 --threshold-rule=percent=1.0 \
  --project=vietrochack-lab
```

```bash
# Artifact Registry cleanup policy - every Cloud Functions deploy (especially
# from CI on every push) builds a new container image that is NOT auto-deleted.
# Left alone this slowly accumulates real storage cost.
cat > /tmp/cleanup-policy.json <<'EOF'
[
  { "name": "keep-minimum-versions", "action": {"type": "Keep"}, "mostRecentVersions": {"keepCount": 3} },
  { "name": "delete-untagged", "action": {"type": "Delete"}, "condition": {"tagState": "UNTAGGED", "olderThan": "86400s"} },
  { "name": "delete-old-tagged", "action": {"type": "Delete"}, "condition": {"tagState": "ANY", "olderThan": "7776000s"} }
]
EOF
gcloud artifacts repositories set-cleanup-policies gcf-artifacts \
  --project=vietrochack-lab --location=us-central1 \
  --policy=/tmp/cleanup-policy.json --no-dry-run
```

## CI/CD: GitHub Actions with Workload Identity Federation

Don't put a GCP service-account key in a GitHub secret. Use Workload
Identity Federation instead, so GitHub proves its identity per run and gets
a short-lived credential, no key ever stored anywhere.

```bash
PROJECT_ID="vietrochack-lab"
REPO="VietRocHack/<repo>"
PROJECT_NUMBER=$(gcloud projects describe "$PROJECT_ID" --format='value(projectNumber)')
SA_EMAIL="gh-actions-deploy-<app>@${PROJECT_ID}.iam.gserviceaccount.com"

gcloud iam service-accounts create gh-actions-deploy-<app> \
  --project "$PROJECT_ID" --display-name "GitHub Actions deploy (<app>)"

for ROLE in roles/cloudfunctions.developer roles/run.admin roles/iam.serviceAccountUser \
            roles/cloudbuild.builds.editor roles/artifactregistry.admin \
            roles/storage.admin roles/firebasehosting.admin; do
  gcloud projects add-iam-policy-binding "$PROJECT_ID" \
    --member "serviceAccount:${SA_EMAIL}" --role "$ROLE"
done

gcloud iam workload-identity-pools create "github" \
  --project="$PROJECT_ID" --location="global" --display-name="GitHub Actions Pool"

# One pool ("github") can be shared across every VietRocHack app's project;
# give each REPO its own provider + attribute-condition inside it
gcloud iam workload-identity-pools providers create-oidc "<app>" \
  --project="$PROJECT_ID" --location="global" --workload-identity-pool="github" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository == '${REPO}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"

gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
  --project="$PROJECT_ID" --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/attribute.repository/${REPO}"

# Store as GitHub repo VARIABLES, not secrets - these identifiers aren't
# sensitive, the security boundary is the attribute-condition above
gh variable set WORKLOAD_IDENTITY_PROVIDER --repo "$REPO" \
  --body "projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/providers/<app>"
gh variable set GCP_SERVICE_ACCOUNT --repo "$REPO" --body "$SA_EMAIL"
```

`.github/workflows/deploy.yml` then just needs
`google-github-actions/auth@v2` with `workload_identity_provider` and
`service_account` pulled from `vars.*`, `permissions: id-token: write` at
the job level, and no secrets at all. See RocMap's for a full working
example.

## Agentic dev-env scaffolding

Every migrated app should end up with this structure, so both humans and AI
agents working on it later have the context they need without rediscovering
it:

- **`CLAUDE.md`** at repo root: repo map, a pointer to read `docs/adr/`
  before architectural changes, the progress-logging convention, and the
  actual local dev / deploy commands for *this* app
- **`docs/adr/`**: one file per real decision (Status/Context/Decision/
  Consequences), numbered in order. Record deploy topology, any bundled-data
  or storage-vs-hosting tradeoffs, and the resource-naming convention this
  app follows.
- **`docs/product/`**: the original pitch/Devpost writeup, so "why does this
  exist" survives independent of whoever built it
- **`docs/progress/YYYYMMDD.md`**: one file per calendar day, a `##` section
  per work session, written as it happens
- **`docs/runbook.md`**: the one-time setup checklist (APIs enabled, Hosting
  site created, DNS records, budget alert, WIF setup) plus anything ongoing
- **`docs/backlog.md`**: known non-blocking issues and cleanup, checked off
  as they land
- **`scripts/deploy.sh`** and **`.github/workflows/deploy.yml`**: the manual
  and automatic deploy paths, doing the same two steps in the same order

## Branding

Every app under `*.vietrochack.com` should carry the same minimal VietRocHack
identity:

- Use [`icon.svg`](https://github.com/VietRocHack/home/blob/main/public/icon.svg)
  from the `home` repo as the actual favicon (`<link rel="icon" ...
  type="image/svg+xml">`), not a generic CRA/Vite default icon
- A footer with `© <current year> VietRocHack`, linking to
  [vietrochack.com](https://vietrochack.com), plus a link to the project's
  Devpost page if it has one
- After deploying, add the app to the **Projects** section of the
  [`home`](https://github.com/VietRocHack/home) repo so it's discoverable
  from the team site

## Frontend polish checklist

Hackathon frontends are usually tuned for one screen size (often mobile,
since that's what gets tested during a sleepless 24 hours) and never
touched again. Before calling a migration done:

1. **Actually load the app at multiple real widths** (~375px, ~768px,
   ~1280px+) and look at it. Don't assume the existing CSS "should" work at
   a given breakpoint just because it looks reasonable in the source.
2. **Grep the CSS for classes that only exist inside one media query block.**
   This is the single most common cause of "looks fine on my phone, broken
   on a real monitor" - a section styled *only* inside `@media
   (max-width: ...)` has zero styling outside that range.
3. **Drive the actual user flow**, not just the happy path: submit with
   fields missing, change a selection after making a different one, trigger
   a network failure. Hackathon code rarely guards against any of this.
4. **Keep the app's actual visual identity.** Polishing means fixing
   layout/bugs/spacing, not redesigning colors, fonts, or the overall vibe
   unless asked.

## Verification

Don't call a migration or a polish pass done from reading the diff alone.
Actually run it:

- Local dev server for iterating (add a `proxy` field, or equivalent, so
  local dev can hit the live backend/assets instead of needing everything
  running locally)
- Drive the real UI through a browser automation tool: type into fields,
  click through dropdowns, submit, check the network tab for the requests
  that actually fired and their status codes
- After shipping, load the actual production URL (not just localhost) and
  repeat the core flow once more
