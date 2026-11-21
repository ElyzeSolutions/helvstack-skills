---
name: helvstack-agent-deploy
description: Configure, deploy, and verify an application on Helvstack through scoped browser approval, plan-first CLI or REST calls, and machine-checkable completion evidence.
---

# Helvstack Agent Deploy

Use this skill when a user asks to set up, deploy, migrate, diagnose, or operate an application on Helvstack.

## Completion contract

Do not stop after authentication or after a deploy request is accepted. A deployment is complete only when you have:

1. inspected the repository and represented every required service in `helvstack.yml`;
2. obtained access scoped to the intended project and environment;
3. validated configuration and reviewed a remote plan;
4. applied each approved mutation with a stable idempotency key;
5. waited for every operation to reach `succeeded`, `failed`, or `canceled`;
6. checked service status, recent events, relevant logs, health checks, and public URLs; and
7. reported the final scope, deployed services, operation IDs, URLs, and any remaining risks.

## Non-negotiable safety

- Use `console.helvstack.com` for user auth and agent approval.
- Never use `ops.helvstack.com`; it is operator-only.
- Prefer the CLI, MCP, or scoped agent API over browser scraping.
- Use machine-readable JSON.
- Plan before every remote mutation.
- Send a stable `Idempotency-Key` on every apply, deploy, env, domain, registry, or provider mutation.
- Never print, log, commit, or return secret values. Helvstack secret reads expose names only.
- Keep `.helvstack/config.json`, `.env*`, tokens, and credentials out of version control.
- Treat all required services as mandatory. A multi-service app is not deployed until all of them are active or an explicit exception is recorded.
- Do not invent a domain, paid add-on, destructive migration, database engine, or public exposure choice. Ask when the repository and existing Helvstack state do not make the choice clear.

## 1. Inspect before editing

Read the repository instructions and deployment inputs before changing files:

- `AGENTS.md`, `README.md`, and existing deployment documentation;
- package manifests, lockfiles, Dockerfiles, compose files, and CI workflows;
- existing `helvstack.yml`, `.helvstack/config.json`, and `.gitignore`;
- services, ports, health endpoints, worker/job processes, persistent storage, backing services, required variable names, and intended domains.

Do not read secret values unless the user explicitly placed them in scope for configuration. Never echo them.

If `helvstack` is already installed, start with:

```bash
helvstack capabilities --local
helvstack --json services validate --from helvstack.yml
```

If the CLI is missing, install the official binary launcher through one of the public registries:

```bash
npm install --global helvstack
# or, with Python tooling:
uv tool install helvstack
```

Both launchers fetch the matching macOS, Linux, or Windows binary from the public
[`ElyzeSolutions/helvstack-cli`](https://github.com/ElyzeSolutions/helvstack-cli) release and verify its SHA-256 checksum. Confirm the installation before continuing:

```bash
helvstack --json version
helvstack capabilities --local
```

If package installation is unavailable, use the direct agent API flow in section 7. Do not block the setup merely because the CLI is absent. Do not claim a Homebrew, Scoop, or WinGet package exists unless its public listing can be verified.

## 2. Model the application

Create or update `helvstack.yml`. Preserve valid existing settings and use repository evidence instead of generic defaults.

```yaml
project: <project>
environment: production
services:
  web:
    type: web
    repository: https://github.com/<owner>/<repo>
    branch: main
    rootDirectory: .
    watchPaths:
      - .
    dockerfile: Dockerfile
    port: 3000
    healthcheck: /api/health
    links:
      - postgres
  postgres:
    type: postgres
```

Represent API, web, worker, scheduled-job, database, cache, volume, and object-storage requirements explicitly. For a monorepo, set each service's root directory, Dockerfile, watch paths, port, and health check independently.

New web, worker, and scheduled-job services run as non-root by default: the image's own `USER` with `runAsNonRoot`, every capability dropped, and no privilege escalation. A service deployed before that default keeps its root-compatible template until it stores a numeric identity; its plans and deploys carry a warning that names the fix. Prefer images that already run as a non-root user and listen on a port at or above 1024, such as `nginxinc/nginx-unprivileged` on port 8080 instead of the official nginx image. Kubernetes cannot verify a named image `USER` such as `node` or `nextjs` under the non-root default, so the deploy fails until a numeric identity is declared. When repository or image evidence proves the numeric non-root identity an application runs as, declare it explicitly on that application service:

```yaml
    runtimeIdentity:
      runAsUser: 10001
      runAsGroup: 10001
      capabilities:
        drop: [ALL]
```

Do not guess a UID or GID. Do not set `fsGroup` merely because `runAsGroup` is set: `fsGroup` may recursively change mounted-volume ownership and is a separate, opt-in field. Only when the image must start as root, and the user accepts that, set the per-service opt-out `runtimeIdentity.allowRoot: true`. It permits uid 0 only: every capability stays dropped except `NET_BIND_SERVICE` below port 1024, so it does not fix images that chown or switch user at startup. The official nginx image is one; use `nginxinc/nginx-unprivileged` on port 8080 instead of `allowRoot`. If a deploy fails with a non-root or permission error, follow the fix it names instead of retrying. Confirm the effective security context in service-apply and deployment plan evidence before applying.

## 3. Request scoped access

Infer a stable project slug from existing configuration or the repository name. Default the environment to `production` only when the user did not provide another target.

```bash
helvstack auth login \
  --project <project> \
  --environment <environment> \
  --name "<repository> deploy agent"
```

The CLI opens a browser approval page and stores a scoped token in `.helvstack/config.json`. Tell the user when the approval page is ready and include the printed `Code:`; the page asks the user to type that code before Approve works, and five wrong codes close the request. Resume automatically after approval.

For a new self-service workspace, the approval page can require the user to activate the Launch plan first. Keep inspecting, editing, and validating locally while access is pending. Never start checkout, purchase a subscription, or imply that payment was completed without the user's explicit action. After activation, resume the same request if it is still pending. Device requests expire after 10 minutes; if it expired during activation, start a fresh scoped login and continue from the remote plan.

For agents that explicitly control a browser:

```bash
helvstack --json auth start --project <project> --environment <environment> --name "<repository> deploy agent"
helvstack --json auth poll --device-code <deviceCode> --wait
```

Use `verificationUriComplete` for the approval page. If you drive the browser for the user, read `userCode` from the start JSON and type it into the code field; never approve a request you did not start. Pending responses expose `browser.opened` and `browser.signedIn`; use those fields instead of scraping the approval page.

## 4. Validate and plan

Run local validation first, then remote plans:

```bash
helvstack --json services validate --from helvstack.yml
helvstack --json services apply --from helvstack.yml --plan
helvstack --json up --service <service> --plan
```

Create or switch environments explicitly; environment creation is a pollable,
idempotent namespace-reconciliation operation:

```bash
helvstack --json environment create <environment> --plan
helvstack --json --idempotency-key "environment-<environment>" environment create <environment>
helvstack environment select <environment>
```

For Git sources, inspect the provider App before connecting the service. Use the
browser installation flow when the repository is not yet authorized, then
review the source plan. Push automation fails closed until the exact repository
grant is visible:

```bash
helvstack --json git-provider apps status
helvstack git-provider apps install --provider github
helvstack --json source connect --service web --repository https://github.com/<owner>/<repo> --branch main --automation push --plan
helvstack --json --idempotency-key "source-web-<git-sha>" source connect --service web --repository https://github.com/<owner>/<repo> --branch main --automation push
```

Managed Harbor hosting is deferred. GitHub source builds default to the repository owner's GHCR namespace; GitLab source builds default to that project's GitLab Container Registry path. Before the first source build in a workspace, configure the matching `ghcr` or `gitlab-registry` provider once with credentials that can push and pull that destination; every other project and environment in the workspace reuses it (`registry-provider get` reports `scope: workspace`), so do not ask the user for a token again. Use `services.<name>.build.destination` for an explicit destination. Never copy a platform-wide registry credential into a tenant environment, and never print or read back credential values.

Summarize the exact project, environment, services, resource changes, domains, and monthly-price deltas shown by the plan. Obtain user confirmation at the normal action-time boundary before a consequential apply when the harness requires it.

## 5. Apply deterministically

Use keys derived from the operation, service, and source revision. Reuse the same key when retrying the same logical action.

```bash
revision="$(git rev-parse --short HEAD)"
helvstack --json --idempotency-key "services-${revision}" services apply --from helvstack.yml --no-wait
helvstack --json --idempotency-key "deploy-web-${revision}" up --service web --no-wait
```

Deploy every source-built or image-backed runtime service explicitly. Do not deploy database, cache, volume, or object-storage declarations as if they were app images.

Volume-backed web and worker rollouts stay Kubernetes `Recreate`. Helvstack prefers the current RWO node and warms the replacement image before detach; web traffic then receives a retryable `503` with `Retry-After: 10` until the new writer is Ready. Plans and results report `zeroDowntime: false`, `sameNodeHandoff`, and `imagePrePulled`. Treat that window as expected, not as a failed deploy.

For variables, always plan first:

```bash
helvstack --json env import --from-file .env.production --plan
helvstack --json --idempotency-key "env-web-${revision}" env import --from-file .env.production --no-wait
```

Do not display the file contents. Prefer trusted secret stores or protected local files.

For domains:

```bash
helvstack --json domain add app.example.com --service web --port 3000 --plan
helvstack --json --idempotency-key domain-app-example-com domain add app.example.com --service web --port 3000 --no-wait
helvstack --json domain verification app.example.com --service web
helvstack --json domain cutover-plan app.example.com --service web
```

Custom domains require ownership verification and DNS routing before activation.

## 6. Wait and verify

Poll returned operation IDs. Do not treat `queued`, `accepted`, `pending`, or `running` as success.

```bash
helvstack --json operations wait <operation-id> --timeout 10m
helvstack --json status <service>
helvstack --json events
helvstack --json logs <service>
helvstack logs <service> --follow
helvstack --json doctor <service>
```

`logs --follow` emits redacted lines continuously; with global `--json` it
emits one structured JSON object per line so agents can process the stream.
Service suspension is deliberately reversible and retains configuration,
history, variables, and persistent volumes:

```bash
helvstack --json down --service <service> --plan
helvstack --json --idempotency-key "suspend-<service>" down --service <service>
helvstack --json resume --service <service> --plan
helvstack --json --idempotency-key "resume-<service>" resume --service <service>
```

Verify every required service, relevant health endpoint, and expected public URL. Check that no secret values appear in status, events, logs, or the final report.

For local commands that need the same variable names, use `helvstack run` or
`helvstack shell` with values already present in the caller's process or an
explicit local `--env-file`. Helvstack retrieves key names only; it never
decrypts stored values. `--json run` validates and returns metadata without
executing the child process.

Return a concise evidence report:

- project and environment;
- config path and services represented;
- plans reviewed;
- operation IDs and terminal states;
- service health;
- public URLs and HTTP health result;
- variables configured by name only;
- unresolved warnings or manual DNS/provider steps.

## 7. Direct agent API fallback

Use this only when the CLI cannot be installed. It is the same scoped product boundary the CLI uses.

Start login without transmitting repository contents:

```bash
umask 077
mkdir -p .helvstack
curl -fsS https://console.helvstack.com/api/agent/auth/start \
  -H 'Content-Type: application/json' \
  -d '{"name":"coding agent","project":"<project>","environment":"<environment>"}' \
  > .helvstack/auth-start.json
jq '{verificationUriComplete,userCode,expiresAt,intervalSeconds}' .helvstack/auth-start.json
```

Ask the user to open `verificationUriComplete` and type the `userCode` shown above, then poll the returned `pollUri` at `intervalSeconds`. The authorized response returns the token once. Save it with mode `0600` and never print it:

```bash
umask 077
cleanup_auth_files() {
  rm -f .helvstack/auth-poll.json .helvstack/auth-start.json
}
trap cleanup_auth_files EXIT
poll_uri="$(jq -r .pollUri .helvstack/auth-start.json)"
curl -fsS "$poll_uri" > .helvstack/auth-poll.json
jq -e '.status == "authorized"' .helvstack/auth-poll.json >/dev/null
jq '{apiUrl,apiToken:.token,project,environment}' .helvstack/auth-poll.json > .helvstack/config.json
chmod 600 .helvstack/config.json
cleanup_auth_files
trap - EXIT
```

Read capabilities and OpenAPI through the scoped proxy, then call only paths permitted by the token:

```bash
api_url="$(jq -r .apiUrl .helvstack/config.json)"
api_token="$(jq -r .apiToken .helvstack/config.json)"
curl -fsS "$api_url/api/v1/capabilities" -H "Authorization: Bearer $api_token"
curl -fsS "$api_url/api/v1/openapi.json" -H "Authorization: Bearer $api_token"
operation_id="<operation-id>"
curl -fsS "$api_url/api/v1/operations/$operation_id" -H "Authorization: Bearer $api_token"
```

Keep token-bearing variables out of command output and shell tracing. Follow the same plan, idempotency, polling, and verification rules as the CLI path.

## No-Git deploys and app passwords

When the app exists only as files (for example an assistant wrote it in chat), deploy it without a repository: `POST .../services/<service>/deployments/plan` with `{"source": {"type": "files", "preset": "static", "files": [{"path": "index.html", "content": "..."}]}}`, then the same body to `.../deployments` with an `Idempotency-Key`. Use `preset: "static"` for plain HTML, CSS and JavaScript, or include a Dockerfile that listens on the service port and runs as a non-root user. Limits: 200 files, 700 KiB of UTF-8 text. Never put secrets in files; set them as variables.

To keep a public preview private, protect the web service with a shared password: `PUT .../services/<service>/access/password` with `{"username": "guest", "password": "<at least 10 characters>"}` after `POST .../access/password/plan`. Helvstack stores only a bcrypt hash. Give the credentials to the user once and never echo them again. With sign-in off, the browser asks for the username and password. With sign-in on, visitors type the password on a Helvstack page at `/.helvstack/password` on the app's link (the username `guest` is implicit, no browser popup). Scripts and `check_app` keep sending `Authorization: Basic guest:<password>` (`curl -u guest:<password>`). Plain HTTP requests to an app's host redirect to HTTPS.

## Mini apps

For a small app with its own data (a feedback form, a photo wall, a sign-up sheet), use the mini preset instead of writing a Dockerfile:

1. `POST /api/v1/mini-apps/plan` with `{"project": "<slug>", "environment": "production", "service": "web", "shortName": "<name>", "files": [...], "setGuestPassword": true}`. It works before the project exists, validates the files, says whether `<name>.apps.helvstack.com` is free (`miniApp.hostname.status`), and lists the apply steps. A new mini app gets Helvstack sign-in (`access.signIn`); pass `"signIn": false` to leave it off. Stop and ask for another name when the plan is `blocked`.
2. Apply the listed steps in order, each with an `Idempotency-Key`. Create the project only if it is missing. Then `POST .../services` with `{"slug": "web", "type": "web", "preset": "mini", "shortName": "<name>"}`, which creates the 1 GiB `/data` volume and claims the name. Turn on sign-in with `PUT .../access/sign-in` (the `set_app_sign_in` step), and set the guest password with `PUT .../access/password` when the user wants one. Last, `POST .../deployments` with `{"source": {"type": "files", "preset": "mini", "files": [...]}}`.
3. Wait for the deploy operation, then verify the URL and `/health`.

The files are only `server.ts` and files under `public/`; Helvstack writes the Dockerfile. `server.ts` runs on Bun with `bun:sqlite` and no npm packages, and it listens on port 8080. Keep data in `process.env.DATABASE_PATH` (`/data/app.db`) and uploads in `process.env.UPLOADS_DIR` (`/data/uploads`): only `/data` survives a redeploy. The app runs one replica. Over MCP, `create_mini_app` does all of this in one tool call. To change an app that is already online, call `get_app_files` first (`GET .../services/{service}/source-files` returns the files of the latest successful files deploy). Edit them, then deploy every file again to the same project and service, so the link and the data stay.

## App sign-in: who is visiting

Never build a login form, store or compare passwords, set an auth cookie, or ask the user for an owner password. Helvstack signs people in at the edge. With sign-in on (`helvstack access sign-in on`, `PUT .../access/sign-in`, or MCP `set_app_sign_in`), every request reaches the app with headers that only Helvstack can set; the edge deletes any copy a client sends:

- `X-Helvstack-Role`: `owner` (an owner or admin of the workspace) or `visitor` (everyone else).
- `X-Helvstack-User`: a stable id for a signed-in person, different in every app. It is absent for anonymous visitors.
- `X-Helvstack-Name`: the display name, percent-encoded. A person typed it, so escape it when you print it.
- `X-Helvstack-Identity`: a 120-second EdDSA JWT with the same claims. Verify it against `https://signin.helvstack.com/.well-known/jwks.json` (`aud` is the app's origin, `iss` is `https://signin.helvstack.com`) when the service is reachable other than through its public domain.

```ts
function who(req: Request) {
  const h = req.headers;
  // HELVSTACK_DEV_ROLE is only for your own machine: Helvstack always sets HELVSTACK_APP_SIGNIN (on or off), so it never applies there.
  const sent = h.get("x-helvstack-role") ?? (process.env.HELVSTACK_APP_SIGNIN ? "" : process.env.HELVSTACK_DEV_ROLE ?? "");
  let name = "";
  try { name = decodeURIComponent(h.get("x-helvstack-name") ?? ""); } catch {}
  return { role: sent === "owner" ? "owner" : "visitor", id: h.get("x-helvstack-user"), name };
}
```

Compare roles by allowlist (`=== "owner"`): any other value, including a missing or unknown one, is a visitor. Link the owner to `/.helvstack/sign-in?return=/admin` and to `/.helvstack/sign-out`. Change data only on POST, PUT or DELETE, never on GET. A signed-in owner is not asked for the guest password. Run the app locally with `HELVSTACK_DEV_ROLE=owner bun server.ts`. On Helvstack every web app gets `HELVSTACK_APP_SIGNIN` (`on` or `off`), so the snippet ignores `HELVSTACK_DEV_ROLE` there, and Helvstack refuses `HELVSTACK_DEV_*` variables and `.env` files or Dockerfiles that set one.

An app with sign-in on and a guest password can also have a guest link, `https://<app>/.helvstack/join#<token>`, off until the owner turns it on: anyone with it opens the app as a guest (`visitor`) without typing the password, so give it to the user and let them decide who gets it. Read it with `GET .../access/guest-link` (`helvstack access guest-link show`, MCP `get_guest_link`); it is a bearer link, so it needs the `guest-link` scope, which only a workspace owner or admin can approve, and every read is recorded. Make a new one with `POST .../access/guest-link/reset-plan` then `POST .../access/guest-link/reset` (`reset_guest_link`), and turn it on or off with `PUT` or `DELETE .../access/guest-link` after their `plan` or `remove-plan` (`set_guest_link`). The link also changes when the guest password changes. Every change signs all guests out: they get back in with the password or the new link.

## MCP

Chat assistants without a shell (Claude.ai, ChatGPT) connect to the remote MCP server at `https://console.helvstack.com/mcp` as a custom connector. It uses OAuth 2.1 with dynamic client registration; the user approves once in the browser. Start with `get_helvstack_guide`; tools plan by default and `get_operation` waits for builds. `create_mini_app` takes a whole mini app from files to a live link in one call, with sign-in on: it checks its plan and, when that adds no charge, applies and waits until the link answers. Send the finished app, never a placeholder, and let Helvstack make the guest code (`makeGuestCode`) instead of inventing one; the owner signs in with Helvstack and needs no code. `check_app` opens the live app from the server to verify it. `set_app_sign_in` turns sign-in on or off for any web service. What else the tools do depends on the connection, and the server's own instructions and tool list say which applies:

- Claude and other connections: when a change would cost money it changes nothing and returns the price, so tell the user the link name and price and apply with `plan: false` only after they agree. `get_guest_link` returns the guest link (also usable as a QR code) that skips the guest code; `reset_guest_link` and `set_guest_link` replace it or turn it off.
- ChatGPT: no prices, codes or guest links appear in the chat, and no change from it can start or add a charge. Helvstack decides that, not the assistant: an account that can be charged (a paid plan, Launch, or an open checkout) is read-only from ChatGPT, so Helvstack shows its apps and plans but changes nothing, and names the Helvstack console (or Claude) for changes; anything that needs a paid plan does the same. Tell the user that and stop; do not retry with `plan: false`. A guest code is made and kept without being shown, and the owner manages it and the guest link in the console. `get_guest_link_status` shows whether guest links are available and enabled. `get_service_log_summary` reports the count, time range and streams of recent logs and links to the console for the full text. An operation reports its failure status and links to the console for details.

After CLI login, an MCP-capable harness can use:

```bash
helvstack mcp serve
```

The stdio server reads the scoped local config. Start with `helvstack_capabilities`, plan before mutation, and use the typed tools when available. Environment inventory/creation, provider-App status/installation, source inspection/connection, and reversible lifecycle are exposed as `helvstack_environments`, `helvstack_environment_create`, `helvstack_git_provider_apps`, `helvstack_git_provider_app_install`, `helvstack_source_status`, `helvstack_source_connect`, and `helvstack_service_lifecycle`; mutation tools default to plan mode.

The signed-in browser console also exposes scoped WebMCP tools for onboarding guidance, context, deploy planning, source status, reversible lifecycle planning/apply, app sign-in planning/apply, reviewed deployment apply, operation watching, and opening relevant human-visible surfaces. Browser mutations require exact reviewed scope plus explicit confirmation.

## Cleanup

Delete temporary auth response files. Keep `.helvstack/config.json` only when the user wants persistent repository-scoped access. Revoke access when retiring it:

```bash
helvstack auth logout --revoke
```
