# Security Scanning

This app is scanned by two workflows:

- `.github/workflows/git-internals-dashboard-security.yml` — SAST, SCA, secrets,
  and IaC config scanning. Runs on every PR and push to `main` touching this app,
  weekly on a schedule (dependency advisories change even when the code doesn't),
  and on demand.
- `.github/workflows/git-internals-dashboard-dast.yml` — DAST, manual dispatch
  only. See [DAST](#dast) below.

## SAST

- **gosec** (`sast-go`) over `backend/`, gated on HIGH confidence + HIGH severity.
  Full results (including lower-confidence findings) are uploaded as SARIF to the
  repo's Security tab for visibility even though they don't fail the build.
- **CodeQL** (`sast-codeql`) over both the Go backend and the TS/React webapp.
  Free on this public repo; results land in the Security tab independently of
  the job-level gates below.
- **ESLint** (`sast-web`) over `webapp/src`, using `eslint-plugin-security`
  layered onto the existing flat config (`webapp/eslint.config.js`) rather than
  a parallel lint setup.

## SCA

- **govulncheck** (`sca-go`) over `backend/` — reachability-aware, so it only
  flags vulnerabilities in code paths the binary actually calls.
- **pnpm audit** (`sca-web`) over `webapp/`, gated at `--audit-level=high`.
- **trivy fs** (`sca-fs`) over the whole app, gated on HIGH/CRITICAL, and
  produces a CycloneDX SBOM as a build artifact on every run.

## Secrets

**gitleaks** (`secrets`) over the app's git history. PR/push runs check only the
new commits; a `workflow_dispatch` run checks full history instead.

## IaC config

**trivy config** (`iac-config`) over `apps/git-internals-dashboard/` and this
app's own two workflow files. The IaC surface here is genuinely thin: there is
no Terraform, no Kubernetes manifest, and no Dockerfile — Choreo owns the
infrastructure. What trivy actually has to look at is `.choreo/component.yaml`,
`backend/project.toml`, `backend/docker-compose.yml` (local dev only), and the
CI workflow files themselves.

## Workflow lint

**actionlint** and **zizmor** (`workflow-lint`) check both of this app's
workflow files for syntax errors and Actions-specific security issues
(unpinned actions, missing `persist-credentials: false`, unsound expressions).

## DAST

`.github/workflows/git-internals-dashboard-dast.yml` is `workflow_dispatch`
only — never push, never a schedule, never production. It takes `target_url`,
`scan_mode` (`baseline` | `api` | `full`, default `baseline`), and the
optional `use_bearer_token` and `confirm_active_scan` flags.

- `baseline` — OWASP ZAP passive scan against the given webapp URL. Safe
  against any environment.
- `api` — OWASP ZAP scan of the backend driven by `backend/openapi.yaml`. Runs
  ZAP's default active API-scan rules against the documented endpoints only
  (not an unbounded crawl), so it's bounded to what's actually in the spec.
  With `use_bearer_token: true`, it reads the repository secret
  `GID_DEV_BEARER_TOKEN` and passes it to ZAP as `ZAP_AUTH_HEADER_VALUE`
  (`Bearer <token>`), which ZAP core injects as the `Authorization` header on
  every request — the token itself is never printed or committed.
- `full` — an unbounded active scan against the shared Choreo data plane,
  indistinguishable from an attack. Refuses to run unless
  `confirm_active_scan: true`, which must only be set once the platform team
  has actually cleared the scan. The safe, clearance-free target for active
  scanning is a local instance — see below.

For active scanning, use `apps/git-internals-dashboard/scripts/dast-local.sh`.
It brings up `backend/docker-compose.yml`'s Postgres, runs migrations and a
synthetic seed, starts the backend and the webapp locally, and runs a ZAP full
scan against both — entirely on `localhost`, never touching Choreo. The
webapp is gated behind Asgardeo OIDC login; without real IdP credentials the
script can only exercise the pre-login surface (static assets, the login
redirect, response headers). The backend has no such gate — it is
deliberately authless, since the Choreo gateway owns JWT validation in every
real deployment — so it's the part of the local scan that actually exercises
API behavior, driven by the same `openapi.yaml`.

## Findings register

Record of what each scanner found on first run, and what was done about it.

| Finding | Scanner | Disposition |
|---|---|---|
| `golang.org/x/text@v0.29.0`: infinite loop on invalid input (GO-2026-5970), reachable via `pgx.Connect` | govulncheck | **Fixed** — bumped to v0.39.0 (indirect dep, no API surface used directly; `go build`/`go vet`/tests pass). |
| `pnpm-workspace.yaml` missing required `packages:` field, blocking `pnpm install` under pnpm 9/10 | manual (blocked `sca-web`/`sast-web` from running at all) | **Fixed** — added `packages: ["."]`. Single-package app; no workspace semantics change. |
| `vite@7.2.4`: dev-server `fs.deny`/websocket/path-traversal issues (multiple HIGH) | pnpm audit, trivy fs | **Fixed** — bumped to 7.3.6 (dev-only tooling, not shipped to production; build/lint/tests pass). |
| `vitest@4.0.18`: CRITICAL arbitrary file read via UI server; MODERATE `@vitest/mocker` path traversal | pnpm audit | **Fixed** — bumped to 4.1.11 (dev/test-only; tests pass). |
| `react-router@7.1.5`: multiple HIGH advisories including an RCE-class turbo-stream deserialization bug, open redirect, DoS, XSS in ScrollRestoration | pnpm audit, trivy fs | **Needs a decision** — see below. Not bumped. |
| `dompurify@3.3.1`: several MODERATE/LOW sanitizer bypass advisories | pnpm audit | **Accepted for now, tracked** — transitive via `@asgardeo/react`'s own dependency tree, not a direct dependency; all below the `--audit-level=high` gate. The latest `@asgardeo/react` (0.25.13) still ships the same `dompurify` version, so there's no upstream fix to pick up yet. Revisit when Asgardeo bumps it, or if this needs escalating independently of the react-router decision. |
| `elliptic@6.6.1`: LOW, "risky cryptographic primitive" | pnpm audit | **Accepted, no fix available** — advisory has no patched version; deep transitive via `@asgardeo/browser`'s `crypto-browserify` polyfill, not used directly by this app. |
| `internal/appconfig/validate.go:27`: gosec G101 "potential hardcoded credentials" on `headerTokenChars` | gosec (default severity) | **Suppressed, false positive** — `#nosec G101` added; the flagged line is a charset constant for validating header names, not a credential. Below the HIGH-confidence gate regardless. |
| `webapp/public/config.js`, `webapp/dist/config.js` contain what looks like an OAuth client ID | gitleaks (initial run with `--no-git`, scanning the filesystem directly) | **Not a finding** — both files are gitignored and were never committed; re-running gitleaks the way CI actually will (against git history) found nothing. The value itself is also a public OIDC client ID for a PKCE flow, not a secret, even hypothetically. |
| Hand-written SQL (`internal/handler/issues.go`, `internal/db/configsync.go`), GitHub search query construction (`internal/github/client.go`, `internal/sync/incremental.go`) | gosec + manual review | **Reviewed, no issue** — all dynamic SQL uses `$N` placeholders for values; table/column names interpolated via `fmt.Sprintf` come only from hardcoded literals at two call sites, never from request input. GitHub search query fields (`owner`, `name`, `issueQuery`) come from the operator-controlled YAML config, not any HTTP request path. |

### Needs a decision: react-router

`react-router@7.1.5` has several HIGH-severity advisories, the most serious
being an RCE-class arbitrary constructor invocation in its vendored
`turbo-stream` deserialization (fixed 7.14.2), plus open-redirect, DoS, and
XSS-in-`ScrollRestoration` issues, all fixed by 7.18.0 or earlier — still
within the v7 line, not a major-version jump. `@asgardeo/react-router`
(2.0.0)'s own peer range (`react-router: >=6.30.1`) permits the bump.

Not bumped unilaterally because react-router underlies every route in this
app, including the ones behind `AuthGuard`, and this task's instructions call
for a decision on anything touching the auth model rather than guessing. If
you're fine with `7.1.5 → 7.18.3`, that's a one-line `package.json` change
plus `pnpm install`; flag it and it'll be done as its own commit.
