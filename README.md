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

If the backend is a persistent server that *already* serves the built
frontend itself (a Flask/Express app with the frontend's static build
copied into its own container, one process, one port - common in
hackathon projects that never split frontend/backend hosting), don't
also deploy the frontend build to Hosting separately. Just rewrite
*everything* to Cloud Run:

```json
{ "source": "**", "run": { "serviceId": "<app>-server", "region": "us-central1" } }
```

with `public` pointing at an empty placeholder directory (Hosting still
requires one to exist, even though nothing in it is ever served). One
container, one deploy artifact, no risk of the Hosting-served frontend
and the container's own copy of it drifting apart.

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

## Common Cloud Run gotchas

Things that will burn a build/deploy cycle each if you don't know to look
for them up front:

- **The container must listen on `$PORT`, not a hardcoded port.** Cloud
  Run injects `PORT=8080` and health-checks that port; a hackathon
  Dockerfile that hardcodes `--bind 0.0.0.0:5000` (or similar) will build
  fine and then fail to deploy with a "failed to start and listen on the
  port" error. Fix at the entrypoint, falling back to the old hardcoded
  port so any existing non-Cloud-Run deploy path (docker-compose, a VM)
  keeps working unchanged:
  `ENTRYPOINT ["sh", "-c", "gunicorn --bind 0.0.0.0:${PORT:-5000} ..."]`
- **`gcloud run deploy --source` can't pass a `--build-arg` through to a
  Dockerfile build.** If the frontend needs a build-time env var baked in
  (e.g. a Maps API key read via `import.meta.env` at build time, not
  runtime), `gcloud run deploy --source .` has no flag for it. Use a
  two-step deploy instead: a custom `cloudbuild.yaml` that runs
  `docker build --build-arg KEY=$_KEY -t $_IMAGE .` via
  `gcloud builds submit --substitutions=_KEY=...`, then
  `gcloud run deploy --image=$_IMAGE`. `scripts/deploy.sh` and
  `.github/workflows/deploy.yml` should both do these same two steps.
- **A scoped CI service account can build successfully and still report
  failure**, because `gcloud builds submit` waits by streaming build logs,
  and streaming from the default GCS logs bucket only recognizes the
  primitive `Viewer`/`Owner` project roles - not a least-privilege custom
  role, even one with `cloudbuild.builds.editor`. The build itself
  actually succeeds; only the CLI's log tail fails. Fix: add to
  `cloudbuild.yaml`
  ```yaml
  options:
    logging: CLOUD_LOGGING_ONLY
  ```
  and grant the CI service account `roles/logging.viewer` (in addition to
  the roles already listed in the WIF section below).
- **Old Dockerfiles drift as base images move forward.** `python:3.11-slim`
  (and similar floating tags) resolve to whatever Debian release is
  current *now*, not whatever it was when the Dockerfile was written. A
  hackathon Dockerfile's `apt-get install -y libgl1-mesa-glx` (a common
  opencv dependency) can start failing with "has no installation
  candidate" months later because that package was renamed/split upstream
  (`libgl1` + `libglib2.0-0`, in this case) on the newer Debian release.
  If an `apt-get install` step in an old Dockerfile suddenly fails, check
  whether the package was renamed before assuming anything else is wrong.

## Non-GCP dependencies (AWS, etc.)

Hackathon backends often reach out to a non-GCP service - most commonly
AWS (DynamoDB/S3), authenticated via a local credentials file
(`~/.aws`) mounted into the container. That mount trick doesn't work on
Cloud Run (no local filesystem to mount from, no way to hand it a file at
deploy time short of baking a static key into a secret and managing its
rotation forever).

Before wiring up a standing external credential as a Cloud Run secret,
check whether the dependency has a native GCP equivalent that needs no
credentials at all: DynamoDB -> a dedicated Firestore database (per the
siloing convention above), S3 -> a dedicated Cloud Storage bucket. On
Cloud Run, both authenticate automatically via the service's own service
account (grant it `roles/datastore.user` / `roles/storage.objectAdmin` as
needed) - nothing to rotate, nothing to leak. Only reach for a real
external secret when there's no GCP equivalent (e.g. a third-party API
that only exists outside GCP).

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

Note if several apps share one GCP project (the common case here): each
app's budget alert fires against that *whole project's* total spend, not
just its own app's usage - they're independent alerts on the same number,
not additive per-app slices. Multiple budgets on a shared project is
normal (per this guide's own convention above) and gives redundant alerts
rather than more precise ones; that's fine, just don't expect a budget
named `<app>` to isolate that app's actual cost.

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
gcloud artifacts repositories set-cleanup-policies <repo> \
  --project=vietrochack-lab --location=us-central1 \
  --policy=/tmp/cleanup-policy.json --no-dry-run
```

`<repo>` is `gcf-artifacts` for a Cloud Functions deploy (auto-created for
you). For Cloud Run built via `gcloud builds submit`/`gcloud run deploy
--image`, create your own named repo first
(`gcloud artifacts repositories create <app> --repository-format=docker
--location=us-central1`) and use that name instead - same reasoning as
every other resource in this doc, don't dump every app's images into one
shared, unscoped repo.

## Third-party API keys: browser-restricted vs. secret-stored

Not every key needs Secret Manager. There are two genuinely different
cases:

- **Client-exposed keys** (a Maps JavaScript key, or anything else that
  ends up inside the built frontend bundle) are not secrets - anyone can
  read them out of the shipped JS regardless of where you store them.
  The actual security boundary is an **HTTP referrer restriction**
  (`gcloud services api-keys create --allowed-referrers=...`), locking
  the key to your app's own domains. Passing it as a GitHub Actions
  secret and a Cloud Build substitution is just about keeping it out of
  workflow logs/history, not secrecy.
- **Real server-side keys** (an LLM provider key, anything called only
  from your backend) belong in Secret Manager, mounted into Cloud Run via
  `--set-secrets=KEY=secret-name:latest`, with `roles/secretmanager.secretAccessor`
  granted to the Cloud Run service's own service account (and to the CI
  deploy service account, so it can attach the binding at deploy time).

Gotcha: `--allowed-referrers` (and similar repeatable-looking flags) does
**not** accumulate across repeated uses on the same `create`/`update`
call - each invocation replaces the previous value. Pass every referrer
as one comma-separated string in a single flag, or call `update` again
with the full combined list.

Also create these keys scoped to the app's own project/service, following
the same siloing convention as everything else - a key created under some
unrelated personal or shared project mixes its quota/billing with
whatever else uses that key.

**If the backend calls a paid LLM API** (OpenAI is the common hackathon
default) and there's no budget to keep paying for it post-hackathon,
check whether the code already abstracts the call behind a configurable
`base_url` - a lot of hackathon projects that started on OpenAI and later
added Groq as a cheaper fallback already do this, since Groq's API is
OpenAI-compatible. If so, swapping to Gemini is usually just adding one
more config entry pointing at Gemini's own OpenAI-compatible endpoint
(`https://generativelanguage.googleapis.com/v1beta/openai`) with a
Gemini API key - no new SDK, no rewritten call sites, and it runs on
Gemini's free tier by default.

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
            roles/storage.admin roles/firebasehosting.admin roles/logging.viewer \
            roles/secretmanager.secretAccessor; do
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
