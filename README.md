# Release Manifest Builder

A GitHub Pages site that lets teams select repositories per environment, then **submit** to automatically open a Pull Request containing a release manifest JSON file — driven by a GitHub Actions pipeline.

---

## How it works (end to end)

```
Browser (GitHub Pages)                GitHub Actions (this repo)
─────────────────────                 ──────────────────────────
1. Tick repositories per tier
2. Set release name + date
3. Click "Submit & Create PR"
        │
        │  POST /repos/{owner}/{repo}/dispatches
        │  event_type: create-manifest
        │  client_payload: { release, date, manifest }
        ▼
                                      4. Workflow triggers on
                                         repository_dispatch
                                      5. Writes manifests/manifest-<rel>-<date>.json
                                      6. Opens a Pull Request
```

The page is static (GitHub Pages has no backend), so submitting fires a **`repository_dispatch`** event to the GitHub API. The workflow in `.github/workflows/create-manifest.yml` receives it, writes the manifest file, and opens a PR.

---

## The page

Two environments side by side:

| Left panel | Right panel |
|-----------|------------|
| **REL** (Release Environment) | **PROD** (Production) |

Each environment has four tiers, each with 10 repositories (`Repo1 … Repo10`):

| Tier | Colour |
|------|--------|
| Application | Purple |
| Platform | Amber |
| Data | Cyan |
| MicroFrontends | Pink |

Header fields: **Release Name** and **Release Date** (defaults to today), plus a **Day / Night theme toggle**.
Config bar: **GitHub Owner**, **Repository**, and an **Access Token** (`GH_Page_TOKEN`). Owner/repo are auto-detected from the Pages URL; the token is remembered in **this browser only** (localStorage) after you paste it once — it is never committed, and the browser cannot read the server-side repo secret.

**Each selected environment produces its own manifest file** — selecting repos in both REL and PROD yields two separate files in the same PR.

---

## One-time setup

### 1. Enable GitHub Pages
- **Settings → Pages → Source:** `Deploy from a branch`, branch `main`, folder `/` (root).
- Site is served at `https://<owner>.github.io/<repo>/`.

### 2. Allow Actions to open PRs — pick ONE

**Option A — repo setting (simplest):**
- **Settings → Actions → General → Workflow permissions:**
  - Select **Read and write permissions**.
  - Tick **Allow GitHub Actions to create and approve pull requests** → **Save**.

> If that checkbox is greyed out, an **organization policy** has locked it (common in enterprise orgs). Use Option B instead.

**Option B — PAT secret (works even when the org locks Option A):**
- Create a **fine-grained PAT** with `Contents: Read and write` **and** `Pull requests: Read and write` on this repo.
- Add it as a repository secret named **`MANIFEST_PAT`**: **Settings → Secrets and variables → Actions → New repository secret**.
- The workflow automatically uses it (`token: ${{ secrets.MANIFEST_PAT || github.token }}`). The PR is opened as your user, which is not subject to the Actions restriction.

### 3. Create an access token (per user)
Create a **fine-grained Personal Access Token** scoped to this repository:
- **Repository access:** only this repo.
- **Permissions:** `Contents: Read and write` (this covers the `dispatches` endpoint).

The token is pasted into the page's **Access Token** field. It is held in browser memory for the session only — never stored in `localStorage`, never committed.

---

## Using it

1. Open the Pages site.
2. Confirm **Owner** / **Repository** (auto-filled from the URL).
3. Paste your **access token**.
4. Set **Release Name** (e.g. `v2.4.0`) and **Release Date**.
5. Tick repositories in **REL** and/or **PROD**. Use **Select All** / **Clear** per tier.
6. Click **🚀 Submit & Create PR**.
7. A `204` response confirms dispatch; the status line updates and a PR appears under the repo's **Pull Requests** tab within seconds.

> **Download JSON** (secondary button) saves the manifest locally without opening a PR — handy for testing or offline use.

---

## Manifest JSON format

**One file per selected environment.** Selecting repos in both environments commits both files:

- `manifests/manifest-rel-<release>-<date>.json`
- `manifests/manifest-prod-<release>-<date>.json`

An environment with no repos selected produces no file. Each file looks like:

```json
{
  "release": "v1.0.0",
  "date": "2026-08-17",
  "generatedAt": "2026-08-17T10:30:00.000Z",
  "environment": "rel",
  "tiers": {
    "application":    ["Repo1", "Repo3"],
    "platform":       ["Repo2"],
    "data":           [],
    "microfrontends": ["Repo5", "Repo8"]
  }
}
```

---

## The pipeline

`.github/workflows/create-manifest.yml`:

- **Trigger:** `repository_dispatch` with type `create-manifest`.
- **Permissions:** `contents: write`, `pull-requests: write`.
- Writes the payload manifest to `manifests/` (payload passed via env vars to avoid shell injection).
- Uses [`peter-evans/create-pull-request`](https://github.com/peter-evans/create-pull-request) to open (or update) a PR on branch `manifest/<release>-<date>`.

---

## Customising repositories

In `index.html`, edit the `TIERS` array and `REPO_COUNT` near the top of the `<script>` block:

```js
const TIERS = [
  { id: 'application',    label: 'Application',    color: '#8b5cf6' },
  { id: 'platform',       label: 'Platform',       color: '#f59e0b' },
  { id: 'data',           label: 'Data',           color: '#06b6d4' },
  { id: 'microfrontends', label: 'MicroFrontends', color: '#ec4899' },
];
const REPO_COUNT = 10;
```

Replace the generated `Repo1 … RepoN` labels with real names via the `repos` array in `buildTierCard()`:

```js
const repos = ['payments-api', 'auth-service', 'order-processor', /* ... */];
```

---

## Troubleshooting

| Symptom | Cause / fix |
|--------|-------------|
| `Dispatch failed: HTTP 404` | Wrong owner/repo, or token lacks access to the repo. |
| `Dispatch failed: HTTP 401/403` | Token missing/expired, or lacks `Contents: Read and write`. |
| Dispatch succeeds but no PR | Pipeline ran but failed. |
| `GitHub Actions is not permitted to create or approve pull requests` | Do **setup step 2 Option A** (tick the checkbox), or **Option B** (add the `MANIFEST_PAT` secret) if the checkbox is locked by org policy. |
| README shows as raw/binary on GitHub | File must be **UTF-8**, not UTF-16. This repo's files are UTF-8. |

---

## File structure

```
jp-gh-pages/
├── index.html                          ← single-file app (HTML + CSS + JS)
├── README.md                           ← this file
├── manifests/                          ← generated manifests land here (via PR)
└── .github/
    └── workflows/
        └── create-manifest.yml         ← pipeline that opens the PR
```
