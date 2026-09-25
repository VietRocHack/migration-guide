# migration-guide

How to move a VietRocHack hackathon project off a teammate's personal
Firebase/GCP project onto the team's `vietrochack-lab` project, live for
good at `<app>.vietrochack.com`. The worked examples are
[RocMap](https://github.com/VietRocHack/RocMap), SwipeAndFly and YoungHeroes.

This is a playbook, not a script. Adapt names and IDs; don't paste blindly.

## The checklist

1. [Find what's broken](#1-find-whats-broken): hardcoded personal URLs, dead repos
2. [Pick a topology](#2-pick-a-topology): one domain, no CORS
3. [Name everything per app](#3-name-everything-per-app)
4. [Deploy](#4-deploy)
5. [Custom domain](#5-custom-domain)
6. [Cost safety net](#6-cost-safety-net): budget alert and image cleanup
7. [Keys and secrets](#7-keys-and-secrets)
8. [CI/CD with Workload Identity Federation](#8-cicd-github-actions--workload-identity-federation)
9. [Docs scaffolding, branding and polish](#9-docs-scaffolding)
10. [Verify in a real browser, locally and in prod](#12-verify)

---

## 1. Find what's broken

Grep for these, then read the code paths that use them (don't trust the README):

| Search for | Usually means |
|---|---|
| `cloudfunctions.net`, `firebaseapp.com` | Frontend calling a personal backend URL |
| `firebasestorage` | Images hosted in a personal bucket |
| `raw.githubusercontent.com` | Backend fetching data from someone's fork on every request |
| Any project ID that isn't the team's | Everything else |

Also watch for:
- **Leftover config**, like a root `package.json` from an old branch that the app doesn't use.
- **Dead sibling repos.** An `<app>-backend` repo may be an abandoned draft,
  with the real logic living in the frontend repo (e.g. Vercel functions
  under `api/`). Diff the two before picking a source of truth.
- **UTF-16 files.** Windows-saved `.md` or `requirements.txt` files can look like
  garbage to naive tools. Decode them before concluding they're empty.

## 2. Pick a topology

Always put the frontend and backend on **one domain** with a Firebase
Hosting rewrite. The frontend calls a relative `/api/...` path, so there's no CORS
and only one DNS record. Pick the row that matches the app:

| The app is… | Deploy as | Hosting rewrite |
|---|---|---|
| Static frontend + a single HTTP function | Cloud Function (gen2). Don't convert it to Cloud Run. | `{"source": "/api/**", "function": {"functionId": "<app>-<fn>", "region": "us-central1"}}` |
| Static frontend + a real multi-route server | Frontend on Hosting, backend on Cloud Run | `{"source": "/api/**", "run": {"serviceId": "<app>-server", "region": "us-central1"}}` |
| One server that already serves its own frontend build | Cloud Run only; `public` points at an empty placeholder dir | `{"source": "**", "run": {"serviceId": "<app>-server", "region": "us-central1"}}` |

Always end the rewrites with `{ "source": "**", "destination": "/index.html" }`
(except in the Cloud-Run-serves-everything case).

Related rules:
- **Static assets** from Firebase Storage should ship with the Hosting deploy. No bucket needed.
- **Data fetched from someone's GitHub** should be bundled with the deploy and read from disk.
- **WebSockets don't pass through Hosting rewrites.** The upgrade gets stripped and the request 404s. Connect straight to the Cloud Run URL for those (YoungHeroes).
- **Stateful LLM APIs don't port between providers.** OpenAI's Assistants API
  keeps threads and config on OpenAI's side, and Gemini has no equivalent.
  Store the history yourself (a Firestore doc per session) and replay it on
  each request. The assistant's prompt and config live in OpenAI's dashboard,
  not the repo, so they have to be rewritten.

## 3. Name everything per app

`vietrochack-lab` hosts many apps, so scope every resource to its app:

| Resource | Convention |
|---|---|
| Cloud Function | `<app>-<function>` |
| Cloud Run service | `<app>-server` |
| Hosting site | Its own named site, never the default. IDs are globally unique, so use `vietrochack-<app>` if `<app>` is taken. Map it to a target in `.firebaserc`. |
| Firestore | A dedicated named database: `gcloud firestore databases create --database=<app>` |
| Storage | A dedicated bucket: `vietrochack-lab-<app>` |
| Artifact Registry | A dedicated repo named `<app>` (Cloud Run); `gcf-artifacts` is auto-created for Functions |
| API keys | Created under the app's own project |

## 4. Deploy

Assumes `gcloud`, `firebase-tools` and `gh` are installed and logged in.

```bash
# 1. Enable Firebase (non-destructive, same project ID)
firebase projects:addfirebase vietrochack-lab

# 2. Enable APIs
gcloud services enable \
  cloudfunctions.googleapis.com run.googleapis.com cloudbuild.googleapis.com \
  artifactregistry.googleapis.com eventarc.googleapis.com billingbudgets.googleapis.com \
  --project=vietrochack-lab

# 3. Create the Hosting site, then write .firebaserc by hand:
#    "targets": { "vietrochack-lab": { "hosting": { "<app>": ["vietrochack-<app>"] } } }
firebase hosting:sites:create vietrochack-<app> --project vietrochack-lab

# 4. Deploy the backend (Cloud Function shown; Cloud Run: gcloud run deploy)
gcloud functions deploy <app>-<function> \
  --gen2 --runtime=<runtime> --region=us-central1 \
  --source=<backend dir> --entry-point=<entry point> \
  --trigger-http --allow-unauthenticated --project=vietrochack-lab

# 5. Build and deploy the frontend
(cd <frontend dir> && npm run build)
firebase deploy --only hosting:<app> --project vietrochack-lab
```

### Cloud Run gotchas

- **Listen on `$PORT`.** Hardcoded ports build fine but then fail to start.
  Keep the old port as a fallback:
  `ENTRYPOINT ["sh", "-c", "gunicorn --bind 0.0.0.0:${PORT:-5000} ..."]`
- **`gcloud run deploy --source` can't pass `--build-arg`.** If the frontend
  needs a build-time env var, deploy in two steps instead: a `cloudbuild.yaml` running
  `docker build --build-arg KEY=$_KEY -t $_IMAGE .` via
  `gcloud builds submit --substitutions=_KEY=...`, then
  `gcloud run deploy --image=$_IMAGE`.
- **CI build "fails" but actually succeeded.** A least-privilege service account
  can't stream the logs. Add `options: { logging: CLOUD_LOGGING_ONLY }` to
  `cloudbuild.yaml` and grant `roles/logging.viewer`.
- **Floating base images drift.** An old `apt-get install` can break because a
  package was renamed upstream (e.g. `libgl1-mesa-glx` → `libgl1` + `libglib2.0-0`).
- **Pinned deps can conflict silently.** Adding a package locally may upgrade
  another pinned package in your venv, so local tests pass while the clean
  container build fails. Resolve against a clean install before pushing.

### Non-GCP dependencies

`~/.aws` credential mounts don't work on Cloud Run. Prefer a GCP equivalent
that needs no stored key: DynamoDB → a dedicated Firestore DB, S3 → a
dedicated bucket. Then grant the service's own service account
`roles/datastore.user` / `roles/storage.objectAdmin`. Use a real external secret
only when there's no GCP equivalent.

## 5. Custom domain

`firebase-tools` has no CLI command for this, so use the REST API:

```bash
TOKEN=$(gcloud auth print-access-token)
BASE="https://firebasehosting.googleapis.com/v1beta1/projects/vietrochack-lab/sites/vietrochack-<app>/customDomains"
H=(-H "Authorization: Bearer $TOKEN" -H "X-Goog-User-Project: vietrochack-lab")

curl -s -X POST "$BASE?customDomainId=<app>.vietrochack.com" "${H[@]}" \
  -H "Content-Type: application/json" -d '{}'          # request it
curl -s "$BASE/<app>.vietrochack.com" "${H[@]}"         # get the DNS records it wants
```

Send the returned CNAME (`<app>` → `vietrochack-<app>.web.app`) and TXT
(`_acme-challenge.<app>`) to whoever manages DNS for `vietrochack.com` (Namecheap).
Re-poll until `hostState: HOST_ACTIVE` and `ownershipState: OWNERSHIP_ACTIVE`.
This usually takes under 24h.

## 6. Cost safety net

This is a permanent deploy, so set these up on day one.

```bash
# Budget alert. It watches the WHOLE project's spend, not just this app's:
# per-app budgets on a shared project are redundant alarms, not per-app slices.
gcloud billing budgets create \
  --billing-account=<billing account id> --display-name="<app> budget" \
  --budget-amount=10USD --calendar-period=month \
  --filter-projects=projects/vietrochack-lab \
  --threshold-rule=percent=0.5 --threshold-rule=percent=0.9 --threshold-rule=percent=1.0 \
  --project=vietrochack-lab

# Image cleanup. Every deploy leaves a container image that's never auto-deleted.
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

**Public endpoints that call a paid API** (an LLM, etc.) also need abuse limits.
Set a spend cap in AI Studio and `--max-instances` on Cloud Run, and rate-limit
the endpoint. See YoungHeroes' `docs/adr/0006-abuse-prevention.md`.

## 7. Keys and secrets

| Key type | Where it goes | What actually protects it |
|---|---|---|
| **Client-exposed** (e.g. Maps JS key, anything in the built bundle) | Build-time env var (GitHub secret or Cloud Build substitution, just to keep it out of logs) | An **HTTP referrer restriction**: `gcloud services api-keys create --allowed-referrers=...` |
| **Server-only** (LLM keys, etc.) | Secret Manager, mounted with `--set-secrets=KEY=secret-name:latest` | Grant `roles/secretmanager.secretAccessor` to the runtime service account and the CI service account. Also restrict the key to the one API it needs. |

Gotcha: `--allowed-referrers` doesn't add up when you repeat it. The last flag wins.
Pass all referrers as one comma-separated value.

**No budget for OpenAI anymore?** If the code already uses a configurable
`base_url` (common when Groq was added as a fallback), point it at Gemini's
OpenAI-compatible endpoint
`https://generativelanguage.googleapis.com/v1beta/openai` with a Gemini key.
No SDK change is needed, and it runs on the free tier.

## 8. CI/CD: GitHub Actions + Workload Identity Federation

Don't store a service-account key in GitHub. With WIF, GitHub gets a
short-lived credential per run instead.

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

# The "github" pool is shared by all apps; skip this if it already exists.
gcloud iam workload-identity-pools create "github" \
  --project="$PROJECT_ID" --location="global" --display-name="GitHub Actions Pool"

# One provider per repo, locked to that repo.
gcloud iam workload-identity-pools providers create-oidc "<app>" \
  --project="$PROJECT_ID" --location="global" --workload-identity-pool="github" \
  --attribute-mapping="google.subject=assertion.sub,attribute.repository=assertion.repository" \
  --attribute-condition="assertion.repository == '${REPO}'" \
  --issuer-uri="https://token.actions.githubusercontent.com"

gcloud iam service-accounts add-iam-policy-binding "$SA_EMAIL" \
  --project="$PROJECT_ID" --role="roles/iam.workloadIdentityUser" \
  --member="principalSet://iam.googleapis.com/projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/attribute.repository/${REPO}"

# Repo VARIABLES, not secrets. These IDs aren't sensitive.
gh variable set WORKLOAD_IDENTITY_PROVIDER --repo "$REPO" \
  --body "projects/${PROJECT_NUMBER}/locations/global/workloadIdentityPools/github/providers/<app>"
gh variable set GCP_SERVICE_ACCOUNT --repo "$REPO" --body "$SA_EMAIL"
```

In `deploy.yml`: `permissions: id-token: write`, then
`google-github-actions/auth@v2` with `vars.WORKLOAD_IDENTITY_PROVIDER` and
`vars.GCP_SERVICE_ACCOUNT`. No secrets needed. RocMap's workflow is a full example.

## 9. Docs scaffolding

Every migrated app gets these, so the next person (or AI agent) has context:

| File | Contents |
|---|---|
| `CLAUDE.md` | Repo map, "read `docs/adr/` first", progress-log rule, local dev and deploy commands |
| `docs/adr/NNNN-*.md` | One real decision each (Status / Context / Decision / Consequences): topology, storage choices, naming |
| `docs/product/` | The original pitch / Devpost writeup |
| `docs/progress/YYYYMMDD.md` | One file per day, one `##` section per work session |
| `docs/runbook.md` | One-time setup: APIs, Hosting site, DNS, budget, WIF |
| `docs/backlog.md` | Known non-blocking issues |
| `scripts/deploy.sh` + `.github/workflows/deploy.yml` | Manual and automatic deploys, doing the same steps in the same order |

## 10. Branding

- Favicon: [`icon.svg`](https://github.com/VietRocHack/home/blob/main/public/icon.svg)
  from `home` (`<link rel="icon" type="image/svg+xml" ...>`).
- Footer: `© <year> VietRocHack` linking to [vietrochack.com](https://vietrochack.com),
  plus the Devpost link if there is one.
- Add the app to the Projects section of [`home`](https://github.com/VietRocHack/home).

## 11. Frontend polish

Hackathon UIs are usually tuned for one screen size. Before calling it done:

1. Load the app at ~375px, ~768px and ~1280px+ and actually look at it.
2. Grep the CSS for classes styled only inside a `@media` block. That's the #1
   cause of "fine on my phone, broken on a monitor".
3. Try unhappy paths: missing fields, changed selections, network failures.
4. Fix layout and bugs, but keep the app's look (colors, fonts, vibe) unless asked.

## 12. Verify

Don't call it done from the diff. Run it:

- Locally, with a dev proxy to the backend.
- In a real browser: click through the flows and check the network tab for the
  requests that fired and their status codes.
- After shipping, repeat the core flow on the **production URL**.
