# SecureOps — Static Site CI/CD Security Pipeline

A simple static website deployed via a security-focused GitHub Actions pipeline.
The application is intentionally minimal; the focus of this repository is the
pipeline itself — automated testing, multi-layer security scanning, and
continuous deployment to GitHub Pages on every push to `main`.

**Live site:** `https://yashitha2001.github.io/CI-CD-PIPELINE/`

---

## What the application is

A single-page static site documenting the pipeline itself. It contains no
server-side logic, no user input, and no third-party scripts loaded at runtime —
a deliberately low attack surface that lets the pipeline demonstrate its
principles cleanly.

---

## Pipeline overview

Three workflow files live in `.github/workflows/`:

### 1. `ci.yml` — Lint & Security Scan

Runs on **every push and every pull request** to any branch.

| Step | Tool | Purpose |
|---|---|---|
| Lint & Validate | `html-validate` | Catches malformed HTML before deployment |
| Trivy FS Scan | `aquasecurity/trivy-action` | Scans files for known CVEs; fails on HIGH/CRITICAL |
| Dependency Review | `actions/dependency-review-action` | Blocks PRs that introduce vulnerable dependencies |

The Trivy job only runs if the lint job passes (`needs: lint`), keeping the
feedback loop fast — no point scanning broken code.

### 2. `codeql.yml` — Static Code Analysis

Runs on **push to main, pull requests to main, and every Monday at 08:00 UTC**.

Uses GitHub's CodeQL with the `security-extended` query suite, which covers
cross-site scripting (XSS), injection, and path traversal patterns. The weekly
schedule ensures that newly disclosed vulnerabilities are caught even when no
code has changed.

### 3. `deploy.yml` — GitHub Pages Deployment

Runs on **push to `main` and manual dispatch**.

Publishes the `src/` directory to GitHub Pages using OIDC-based short-lived
tokens (`id-token: write`) — no long-lived deploy keys or secrets required.
A `concurrency` group prevents two deployments from racing each other.

---

## Security choices and reasoning

### Actions pinned to commit SHAs

Every `uses:` line references a full commit SHA rather than a mutable tag like
`@v4`. A tag can be silently re-pointed to malicious code; a SHA cannot. This
is the primary defense against supply-chain attacks in the Actions ecosystem.

```yaml
# Safe — immutable
uses: actions/checkout@11bd71901bbe5b1630ceea73d27597364c9af683 # v4.2.2

# Unsafe — tag can be re-pointed
uses: actions/checkout@v4
```

### Least-privilege permissions

Each workflow declares only the permissions it actually needs via the top-level
`permissions:` block. The default GitHub token is over-privileged; explicitly
scoping it limits the blast radius if a workflow is compromised.

### Trivy SARIF upload

Trivy results are uploaded to GitHub's Security tab in SARIF format. This keeps
findings visible alongside code rather than buried in log output, and surfaces
them in pull request checks automatically.

### Secret scanning (repository setting)

GitHub's built-in secret scanning should be enabled in
**Settings → Security → Secret scanning**. It monitors every commit for exposed
API keys, tokens, and credentials and alerts immediately — no workflow file
required.

### Branch protection (recommended repository setting)

In **Settings → Branches → Branch protection rules** for `main`:

- Require status checks to pass before merging (select `Lint & Validate` and
  `Trivy Security Scan`)
- Require branches to be up to date before merging
- Do not allow bypassing the above settings

This makes the pipeline a hard gate, not a suggestion.

---

## How to run or reproduce this locally

**Prerequisites:** Node.js 20+, [Trivy](https://aquasecurity.github.io/trivy/latest/getting-started/installation/)

```bash
# Clone the repository
git clone https://github.com/<your-username>/<your-repo>.git
cd <your-repo>

# Install and run html-validate
npm install -g html-validate
html-validate src/index.html

# Run Trivy locally
trivy fs . --severity HIGH,CRITICAL
```

To trigger the full pipeline, push any commit to a branch or open a pull
request. The Actions tab will show all three workflows running in parallel
(except `dependency-review`, which only runs on pull requests).

---

## Repository structure

```
.
├── .github/
│   └── workflows/
│       ├── ci.yml          # Lint + Trivy scan + dependency review
│       ├── codeql.yml      # CodeQL static analysis
│       └── deploy.yml      # GitHub Pages deployment
├── src/
│   ├── index.html          # Application
│   └── style.css           # Styles
├── .htmlvalidate.json      # html-validate rule configuration
└── README.md
```

---

## Looking ahead — what would come next

If this were heading toward production or I had another week:

1. **Content Security Policy header** — GitHub Pages doesn't support custom
   response headers natively, but a `_headers` file (Netlify/Cloudflare Pages)
   or a thin CDN layer would let me add `Content-Security-Policy`,
   `X-Frame-Options`, and `Strict-Transport-Security`.

2. **Automated SHA-pinning updates** — Dependabot can be configured to keep
   pinned Action SHAs current (`dependabot.yml` with `package-ecosystem:
   github-actions`). I intentionally left this out to keep the repo simple,
   but it's the right long-term approach.

3. **Lighthouse / accessibility CI check** — A `pa11y` or `axe` step would
   catch accessibility regressions before deployment.

4. **SBOM generation** — Trivy can output a CycloneDX or SPDX software bill of
   materials. For a real product this would be attached as a release artifact.

5. **Environment protection rules** — The `github-pages` deployment environment
   can require a manual approval step before production deploys go live.

---

*Built as a take-home challenge demonstrating CI/CD security pipeline practices.*
