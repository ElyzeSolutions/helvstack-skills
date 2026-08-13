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

When repository or image evidence proves an application must run as a fixed non-root Linux identity, declare it explicitly on that application service:

```yaml
    runtimeIdentity:
      runAsNonRoot: true
      runAsUser: 10001
      runAsGroup: 10001
      capabilities:
        drop: [ALL]
```

Do not guess a UID or GID. Do not set `fsGroup` merely because `runAsGroup` is set: `fsGroup` may recursively change mounted-volume ownership and is a separate, opt-in field. Omit `runtimeIdentity` for root-based third-party images. Confirm the effective security context in service-apply and deployment plan evidence before applying.

## 3. Request scoped access

Infer a stable project slug from existing configuration or the repository name. Default the environment to `production` only when the user did not provide another target.

```bash
helvstack auth login \
  --project <project> \
  --environment <environment> \
  --name "<repository> deploy agent"
```

The CLI opens a browser approval page and stores a scoped token in `.helvstack/config.json`. Tell the user when the approval page is ready and include the displayed code. Resume automatically after approval.

For a new self-service workspace, the approval page can require the user to activate the Launch plan first. Keep inspecting, editing, and validating locally while access is pending. Never start checkout, purchase a subscription, or imply that payment was completed without the user's explicit action. After activation, resume the same request if it is still pending. Device requests expire after 10 minutes; if it expired during activation, start a fresh scoped login and continue from the remote plan.

For agents that explicitly control a browser:

```bash
helvstack --json auth start --project <project> --environment <environment> --name "<repository> deploy agent"
helvstack --json auth poll --device-code <deviceCode> --wait
```

Use `verificationUriComplete` for the approval page. Pending responses expose `browser.opened` and `browser.signedIn`; use those fields instead of scraping the approval page.

## 4. Validate and plan

Run local validation first, then remote plans:

```bash
helvstack --json services validate --from helvstack.yml
helvstack --json services apply --from helvstack.yml --plan
helvstack --json deploy --service <service> --plan
```

Managed Harbor hosting is deferred. GitHub source builds default to the repository owner's GHCR namespace; GitLab source builds default to that project's GitLab Container Registry path. Before apply, configure the matching environment-scoped `ghcr` or `gitlab-registry` provider with credentials that can push and pull that destination. Use `services.<name>.build.destination` for an explicit destination. Never copy a platform-wide registry credential into a tenant environment, and never print or read back credential values.

Summarize the exact project, environment, services, resource changes, domains, and monthly-price deltas shown by the plan. Obtain user confirmation at the normal action-time boundary before a consequential apply when the harness requires it.

## 5. Apply deterministically

Use keys derived from the operation, service, and source revision. Reuse the same key when retrying the same logical action.

```bash
revision="$(git rev-parse --short HEAD)"
helvstack --json --idempotency-key "services-${revision}" services apply --from helvstack.yml --no-wait
helvstack --json --idempotency-key "deploy-web-${revision}" deploy --service web --no-wait
```

Deploy every source-built or image-backed runtime service explicitly. Do not deploy database, cache, volume, or object-storage declarations as if they were app images.

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
helvstack --json operations get <operation-id>
helvstack --json status <service>
helvstack --json events
helvstack --json logs <service>
helvstack --json doctor <service>
```

Verify every required service, relevant health endpoint, and expected public URL. Check that no secret values appear in status, events, logs, or the final report.

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

Ask the user to approve `verificationUriComplete`, then poll the returned `pollUri` at `intervalSeconds`. The authorized response returns the token once. Save it with mode `0600` and never print it:

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

## MCP

After CLI login, an MCP-capable harness can use:

```bash
helvstack mcp serve
```

The stdio server reads the scoped local config. Start with `helvstack_capabilities`, plan before mutation, and use the typed tools when available.

The signed-in browser console also exposes scoped WebMCP tools for onboarding guidance, context, deploy planning, reviewed apply, operation watching, and opening relevant human-visible surfaces.

## Cleanup

Delete temporary auth response files. Keep `.helvstack/config.json` only when the user wants persistent repository-scoped access. Revoke access when retiring it:

```bash
helvstack auth logout --revoke
```
