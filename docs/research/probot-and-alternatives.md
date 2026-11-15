# Probot and its alternatives for the repobuddy GitHub App

Last updated: November 2026

## Question

What does Probot still give a GitHub App built today, and what are the alternatives? This page compares:

- Probot
- Octokit used directly (`@octokit/app`, `@octokit/webhooks`, `@octokit/auth-app`)
- bots that run only in GitHub Actions
- serverless webhook handlers (Cloudflare Workers, Vercel Functions, Netlify Functions)
- newer frameworks

Each one is rated on maintenance, TypeScript support, edge runtimes (Cloudflare Workers in particular), local development and testing, auth and tokens, and lock-in.

The bot's design and the hosting comparison (Workers limits and pricing, Heroku, Render, Fly.io, Tailscale Funnel) are in [#40](https://github.com/repobuddy/github-app/issues/40). This page does not repeat them. It answers which library, if any, the Stage 1 workflows and the Stage 2 router should be built on.

## Verdict

**Drop Probot. Build Stage 2 on Octokit directly, and keep Stage 1 free of any bot framework.**

- **Stage 1 (Actions only)** needs no bot framework. Use a plain reusable workflow:
  - Read the `issue_comment` payload.
  - Call `npx repobuddy@<version>` with `GITHUB_TOKEN`.
  - Where a push must start CI, use [`actions/create-github-app-token`](https://github.com/actions/create-github-app-token) for an app token.
- **Stage 2 (Workers router)** fits `@octokit/webhooks` and `@octokit/auth-app`, or `@octokit/app`, which bundles them. The router's three jobs are:
  - `createWebMiddleware()` or `verify()` checks the signature.
  - `@octokit/auth-app` mints the installation token.
  - One `octokit.request("POST /repos/{owner}/{repo}/dispatches")` sends the dispatch.

  Probot has no supported path to Workers. What it adds on top of Octokit is pino logging, `context.config`, `probot run`, and the adapters. The router needs none of these.
- **The current Probot starter** (`probot@^13`, an Express-era server) is a placeholder, and nothing builds on it. Replace it when Stage 2 is built; don't upgrade it to v14 first.
- **One setup step for Workers.** Convert the app's private key from PKCS#1 to PKCS#8 before storing it as a Worker secret. On Workers, Octokit signs with WebCrypto, which only accepts PKCS#8. On Node, the library converts the key itself.

### Comparison

| | Probot v14 | Octokit directly | Actions-only bot | Hand-rolled serverless handler |
|---|---|---|---|---|
| Maintenance | Active. v14.3.7 on 2026-11-03, five patches in 5 days after a 7-month gap ([E1](#e1)) | Active. `app.js` v16.1.4 (2026-08), `auth-app.js` v8.3.1 (2026-09), `webhooks.js` v14.2.0 (2025-12) ([E5](#e5)) | GitHub platform. `create-github-app-token` v3.3.0 (2026-10) ([E12](#e12)) | You maintain it |
| TypeScript | Yes. Events are typed from GitHub's OpenAPI webhook spec since v14 ([E2](#e2)) | Yes. The same `@octokit/openapi-webhooks-types` ([E6](#e6)) | Workflow YAML, plus whatever the CLI is written in | Whatever you write |
| Cloudflare Workers | No official support. A maintainer said Probot would not run there ([E3](#e3)) | Yes. `createWebMiddleware` is documented for Workers. Auth uses WebCrypto and needs a PKCS#8 key ([E6](#e6), [E7](#e7)) | Not applicable | Yes. Usually built on Octokit anyway |
| Other serverless | Adapters for Lambda, Azure, GCF, and Actions. Vercel and Netlify use `createNodeMiddleware` ([E4](#e4)) | Node middleware, web/fetch middleware ([E6](#e6)) | Not applicable | Vercel: 300 s max on Hobby. Netlify: 60 s synchronous ([E16](#e16)) |
| Local dev | smee.io proxy, `probot receive` to replay a payload ([E4](#e4)) | Bring your own: smee client, `wrangler dev`, the `gh webhook` extension ([E17](#e17)) | `act`, or a test repository | Same as Octokit |
| Testing | Documented: nock with `probot.receive()` ([E4](#e4)) | Call `webhooks.receive()` or `verifyAndReceive()` with a fixture. Mock HTTP yourself | Run the CLI locally. The workflow is thin | Same as Octokit |
| Auth and tokens | `APP_ID` and `PRIVATE_KEY` env vars. Installation tokens are minted and cached for you ([E4](#e4)) | `@octokit/auth-app` mints and caches installation tokens ([E7](#e7)) | `GITHUB_TOKEN`, or an app token from `create-github-app-token` that lasts 1 hour and is revoked when the job ends ([E12](#e12)) | Through `@octokit/auth-app`, or a hand-signed JWT |
| Lock-in | Moderate. The `app.on` and `context` API is Probot-only, but `context.octokit` is plain Octokit | Low. Octokit is GitHub's own SDK | Low for the logic, which lives in the CLI. High for the trigger model, which is Actions | Low in the code. The runtime's APIs (KV, queues) can add some |

### What each option does well

- **Probot.** It is the fastest way to get a long-running Node app answering many event types. It handles routing, auth, logging, config files in repositories, retries, and throttling, and it ships a testing recipe ([E2](#e2), [E4](#e4)). That fits an app whose logic runs inside the webhook handler. #40 puts the logic in Actions and keeps the service a router, so most of Probot would go unused.
- **Octokit directly.** Probot is built on the same packages. Each package does one job: signatures, app JWTs, installation tokens, or REST calls. Its web builds run on Workers ([E6](#e6), [E7](#e7), [E8](#e8)). It suits a router of about 50 lines.
- **Actions-only.** No service, no key held anywhere outside GitHub, and free on public repositories. Its limits are the ones #40 already lists. A `GITHUB_TOKEN` push starts no runs, except `workflow_dispatch` and `repository_dispatch` ([E10](#e10)). `issue_comment` runs from the default branch ([E11](#e11)). An Actions run takes longer to start than a webhook reply.
- **Serverless handlers.** A handler on Workers has no cold start that misses GitHub's 10-second window ([E9](#e9)). Vercel and Netlify functions also work. They give more execution time than Workers, but they run Node, so the Probot adapters apply there, while Workers is the host #40 chose.
- **Newer frameworks.** None is a better fit:
  - `gh-aw` (GitHub Agentic Workflows) is active and has a `slash_command:` trigger. It runs AI agents from natural-language workflows, not deterministic `buddy` commands ([E19](#e19)).
  - The one Workers-specific framework found, `octoflare`, is gone. Its GitHub repository returns 404, and its npm package was unpublished on 2025-06-19 ([E20](#e20)).
  - Active frameworks in other languages, such as Go's `palantir/go-githubapp` and Python's `gidgethub` ([E21](#e21)), would mean leaving TypeScript and the buddy CLI's ecosystem.

## Confidence

High for the verdict. The deciding facts come from primary sources:

- Probot does not support Workers.
- `@octokit/webhooks` documents Workers support.
- WebCrypto in `universal-github-app-jwt` requires a PKCS#8 key.

Medium for anything about runtime cost. Nobody has measured Octokit's bundle or CPU time on Workers.

## Evidence

<a id="e1"></a>**E1.** Probot v14.3.7 was released on 2026-11-03. v14.3.3 to v14.3.7 shipped between 2026-10-30 and 2026-11-03, after v14.3.2 on 2026-04-03. The repository is not archived and has about 9.6k stars. No maintenance-mode or funding statement was found. Source: [probot/probot releases](https://github.com/probot/probot/releases) (release notes, GitHub API).

<a id="e2"></a>**E2.** Probot v14.0.0 (2025-07-30) made these breaking changes:

- It is ESM only.
- It needs Node 20.18 or later, or Node 22.
- Webhook types come from GitHub's OpenAPI spec.
- The legacy `octokit.issues.*` access is removed in favour of `octokit.rest.*`.
- The built-in Express server is removed.
- `createNodeMiddleware()` is now async.
- `@sentry/node` and `ioredis` are installed separately.
- The server listens on `localhost` by default.

v14 depends on pino, `@octokit/core` v7, `@octokit/webhooks` v14, and the retry, throttling, and config plugins. Sources: [v14.0.0 release notes](https://github.com/probot/probot/releases/tag/v14.0.0), [package.json](https://github.com/probot/probot/blob/master/package.json).

<a id="e3"></a>**E3.** Probot maintainer gr2m: "I don't think Probot will be compatible with Cloudflare workers, but I did get it to work using the underlying Octokit." The Probot deployment docs do not list Cloudflare. Sources: [probot/probot#1656](https://github.com/probot/probot/issues/1656) (issue, 2021), [deployment docs](https://github.com/probot/probot/blob/master/docs/deployment.md).

<a id="e4"></a>**E4.** Probot's documented deployment targets are a Node server, AWS Lambda (`@probot/adapter-aws-lambda-serverless`), Azure Functions, Google Cloud Functions, GitHub Actions (`@probot/adapter-github-actions`), and Vercel and Netlify through `createNodeMiddleware`. It is configured with `APP_ID`, `WEBHOOK_SECRET`, and `PRIVATE_KEY`. Local development proxies webhooks through smee.io. `probot receive` replays a payload but not its headers. The testing docs use nock with `probot.receive()`, with retry and throttling turned off. Sources: [deployment](https://github.com/probot/probot/blob/master/docs/deployment.md), [development](https://github.com/probot/probot/blob/master/docs/development.md), [simulating webhooks](https://github.com/probot/probot/blob/master/docs/simulating-webhooks.md), [testing](https://github.com/probot/probot/blob/master/docs/testing.md) (docs).

<a id="e5"></a>**E5.** Latest Octokit releases:

- `octokit/app.js` v16.1.4, 2026-08-02
- `octokit/auth-app.js` v8.3.1, 2026-09-01
- `octokit/webhooks.js` v14.2.0, 2025-12-03
- `octokit/octokit.js` v5.0.5, 2025-10-31

All four were pushed to in October or November 2026. Source: GitHub releases API for each repository.

<a id="e6"></a>**E6.** The `@octokit/webhooks` README: "The middleware returned from `createWebMiddleware` can also be used in serverless environments like AWS Lambda, Cloudflare Workers, and Vercel." The package also exposes `verify()` and `verifyAndReceive()`. Its signature helper `@octokit/webhooks-methods` has a WebCrypto build behind the `browser` export condition. The `node` and `default` conditions point to the `node:crypto` build. Events are typed by `@octokit/openapi-webhooks-types`. Sources: [webhooks.js README](https://github.com/octokit/webhooks.js#createwebmiddleware), [webhooks-methods.js](https://github.com/octokit/webhooks-methods.js), `npm view @octokit/webhooks-methods exports`.

<a id="e7"></a>**E7.** The `@octokit/auth-app` README: "The private keys provided by GitHub are in `PKCS#1` format, but the WebCrypto API only supports `PKCS#8`." It gives the conversion `openssl pkcs8 -topk8 -inform PEM -outform PEM -nocrypt -in private-key.pem -out private-key-pkcs8.key`. Source: [auth-app.js README](https://github.com/octokit/auth-app.js#readme).

<a id="e8"></a>**E8.** `universal-github-app-jwt`, which signs the app JWT for `auth-app`, maps `#crypto` to `crypto-node.js` under the `node` condition and to `crypto-native.js` (`globalThis.crypto.subtle`) otherwise. Only the Node build converts a PKCS#1 key, through `createPrivateKey`. Sources: the package's `package.json` and `lib/` as published on npm, and the [README section on key formats](https://github.com/gr2m/universal-github-app-jwt#about-private-key-formats).

<a id="e9"></a>**E9.** GitHub expects a 2XX response within 10 seconds, or it ends the connection and marks the delivery failed. It advises queueing the work and replying at once. GitHub does not redeliver failed deliveries automatically. They can be redelivered by hand, or through the API, for 3 days. This confirms the figure that #40 recorded from memory. Sources: [webhook best practices](https://docs.github.com/en/webhooks/using-webhooks/best-practices-for-using-webhooks), [redelivering webhooks](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/redelivering-webhooks) (GitHub docs).

<a id="e10"></a>**E10.** Events triggered by `GITHUB_TOKEN` create no new workflow runs, except `workflow_dispatch` and `repository_dispatch`. Source: [GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token) (GitHub docs).

<a id="e11"></a>**E11.** An `issue_comment` workflow runs from the default branch, and only if the workflow file exists there. `check_run` with `requested_action` is a workflow trigger, but it does not fire for check suites that GitHub Actions created. Source: [events that trigger workflows](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows) (GitHub docs).

<a id="e12"></a>**E12.** `actions/create-github-app-token`:

- Latest release is v3.3.0, 2026-10-12.
- It takes `client-id` (or the legacy `app-id`), `private-key`, `owner`, `repositories`, and `permission-<name>`.
- It mints an installation token that expires after 1 hour and is revoked in a post step.

Source: [README and releases](https://github.com/actions/create-github-app-token).

<a id="e13"></a>**E13.** `peter-evans/slash-command-dispatch`:

- Latest release is v5.0.2, 2025-12-18.
- It turns slash commands into `repository_dispatch` or `workflow_dispatch` events.
- Its README says `GITHUB_TOKEN` "does not work here" and asks for a PAT.

Source: [README](https://github.com/peter-evans/slash-command-dispatch).

<a id="e14"></a>**E14.** `@probot/adapter-github-actions` (v5.0.1, 2025-10-20) runs a Probot app inside a workflow with only `GITHUB_TOKEN`. The app cannot authenticate as itself. Source: [README](https://github.com/probot/adapter-github-actions).

<a id="e15"></a>**E15.** `gr2m/cloudflare-worker-github-app-example` was last pushed on 2022-09-23. It does not verify webhook signatures, because `@octokit/webhooks` used Node's crypto at the time. Source: [README](https://github.com/gr2m/cloudflare-worker-github-app-example).

<a id="e16"></a>**E16.** Vercel Functions with fluid compute default to 300 s, which is also the Hobby maximum. Pro allows up to 800 s. Netlify synchronous functions are limited to 60 s and background functions to 15 minutes. Sources: [Vercel duration](https://vercel.com/docs/functions/configuring-functions/duration), [Netlify functions configuration](https://docs.netlify.com/build/functions/configuration/) (docs).

<a id="e17"></a>**E17.** smee.io is "intended for use in development, not for production", and anyone with a channel ID can see its payloads. `probot/smee-client` released v4.4.3 on 2025-11-25. `cli/gh-webhook` released v0.2.0 on 2025-10-21, and its README documents only the install command. Sources: [smee.io README](https://github.com/probot/smee.io), [smee-client](https://github.com/probot/smee-client), [cli/gh-webhook](https://github.com/cli/gh-webhook).

<a id="e18"></a>**E18.** With `nodejs_compat`, Workers provides `node:crypto` alongside Web Crypto. The flag is on by default for compatibility dates of 2026-08-04 or later. On the Free plan, time spent waiting on `fetch` does not count toward the 10 ms CPU limit. Sources: [Workers Node.js compatibility](https://developers.cloudflare.com/workers/runtime-apis/nodejs/), [Workers limits](https://developers.cloudflare.com/workers/platform/limits/) (docs).

<a id="e19"></a>**E19.** `github/gh-aw` released v1.2.3 on 2026-11-13. It compiles natural-language workflows into Actions workflows that run AI agents, and it has a `slash_command:` trigger. Sources: [repository](https://github.com/github/gh-aw), [command triggers](https://github.github.com/gh-aw/reference/command-triggers/) (docs).

<a id="e20"></a>**E20.** `octoflare`, a "framework for GitHub Apps on Cloudflare Workers", is gone. `jill64/octoflare` returns 404 on the GitHub API, and `npm view octoflare` reports "Unpublished on 2025-06-19". Sources: GitHub API, npm registry.

<a id="e21"></a>**E21.** Active frameworks in other languages:

- `palantir/go-githubapp` (Go): v0.50.0, 2026-11-12
- `gidgethub` (Python): v6.0.0, 2026-09-25
- `sanitizers/octomachinery` (Python): v0.3.11, 2024-08-26
- `XAMPPRocky/octocrab`: v0.54.9, 2026-10-31. It is a Rust API client, not a bot framework.

Source: GitHub releases API.

## Weakening and conflicting evidence

- **The Workers build of the signature helper.** `@octokit/webhooks-methods` puts its WebCrypto build behind the `browser` export condition, and its `default` condition is the `node:crypto` build ([E6](#e6)). On Workers the Node build should still work, because `nodejs_compat` provides `node:crypto` ([E18](#e18)). This holds either way, but check which build Wrangler bundles.
- **`slash-command-dispatch` and `GITHUB_TOKEN`.** Its README says `GITHUB_TOKEN` does not work and asks for a PAT ([E13](#e13)). GitHub's docs say `repository_dispatch` sent with `GITHUB_TOKEN` does start runs ([E10](#e10)). The likely explanation is that `GITHUB_TOKEN` cannot dispatch to other repositories. This was not tested. #40's design dispatches within the same repository, so it should not be affected.
- **Probot is active.** A burst of releases in late 2026 means it is not abandoned ([E1](#e1)). The case against it here is fit, not health.

## Not supported

- That Probot can run on Cloudflare Workers. No docs, adapter, or maintainer statement says so ([E3](#e3)).
- That a maintained, Workers-native GitHub App framework exists. The one found was unpublished ([E20](#e20)).

## Where evidence is thin

- **CPU time.** No measurement was found of RS256 JWT signing or HMAC verification against the 10 ms CPU limit on Workers. Waiting on I/O does not count toward it ([E18](#e18)), and both operations are single WebCrypto calls. Measure on a deployed Worker before relying on the Free plan.
- **Bundle size** of `@octokit/app` against `@octokit/webhooks` and `@octokit/auth-app` alone was not measured.
- **`check_run` `requested_action`** as an Actions trigger: the docs list it but do not say whether a button defined by the app reaches a workflow ([E11](#e11)). #40 assumes it needs the app webhook. Check before relying on either path.
- **Normal Actions start latency** has no official figure. #40's estimate of 10 to 30 seconds is not sourced here either.
- **`gh webhook forward`**: the extension's README does not document it ([E17](#e17)). Try it before replacing smee with it.

## Check again later

- Probot release notes, for a fetch-based handler or edge runtime support.
- Whether `@octokit/webhooks-methods` adds a `workerd` or `worker` export condition.
- `gh-aw`'s maturity, if the `/buddy` commands ever call for an AI agent rather than a fixed CLI command.
