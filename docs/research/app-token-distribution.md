# How the repobuddy app should hand out tokens

Last updated: November 2026

## Question

How can one GitHub App, `repobuddy` (id 1056494), serve two groups without copying its private key into every org? The groups are:

- the owner's own CI jobs across many orgs
- outside users who install the app

The CI jobs come from repobuddy/.github#79:

- J1: add a changeset to Renovate PRs.
- J2: push rebuilt output onto Renovate PRs.
- J3: refresh the lockfile on Dependabot PRs.

Each job pushes onto a bot PR, and the push must start CI, so `GITHUB_TOKEN` will not do ([E1](#e1)). Today every org that runs them holds the app's key as an org secret, in an Actions copy and a Dependabot copy, and mints tokens with `actions/create-github-app-token`. The app is public. It is installed on 12 of the owner's 36 orgs, and all 36 are planned.

This page compares:

- **(A)** today's model: the key is copied into each org.
- **(B)** one app whose key lives only in a token-exchange service. The service trades a workflow's Actions OIDC token for an installation token scoped to the calling repository, as Chainguard's octo-sts does.
- **(C)** two apps: a CI app whose key is copied into the owner's orgs, and the public `repobuddy` app, whose key lives only in its service.
- **(D)** GitHub's own keyless app tokens for Actions, which are on GitHub's roadmap.
- **(E)–(H)** other approaches: `workflow_dispatch` with `GITHUB_TOKEN`, one app per org, fine-grained PATs, and a Vault plugin.

The bot design and the Cloudflare Workers router are in [#40](https://github.com/repobuddy/github-app/issues/40). The framework choice for the router is in [probot-and-alternatives.md](probot-and-alternatives.md). This page does not repeat them.

## Verdict

**Stop planning to copy the public app's key into every org. Split the identities now (C), and move the CI jobs to GitHub's keyless app tokens (D) when GitHub ships them. Build a token-exchange service (B) only if D does not arrive before outside users are invited.**

- **No native answer exists yet, but GitHub has announced one.**
  - `GITHUB_TOKEN` still cannot start CI with a push ([E1](#e1)).
  - Fine-grained PATs still cannot span orgs ([E13](#e13)).
  - GitHub's public roadmap lists "Keyless GitHub App Tokens in Actions [Public Preview]" (opened 2026-06-12, last updated 2026-09-03). Workflows would act as the app without a stored key, and admin-defined policies would choose which workflows may mint tokens ([E3](#e3)).
  - A second item adds federated credentials from any issuer to GitHub Apps [GA] (opened 2026-08-10) ([E4](#e4)).
  - On 2026-10-13 GitHub added a Credentials tab to app settings "in advance of our planned workload federation features" ([E5](#e5)).
  - None of these has shipped as of 2026-11-14. That is what B would build by hand.
- **(A) has the worst blast radius, and it grows with adoption.**
  - Anyone who reads the key can mint tokens for every installation, outside users' included, with every permission the app has.
  - App keys never expire ([E11](#e11), [E26](#e26)). GitGuardian found that 10% of 4,802 leaked app keys still worked in September 2026 ([E26](#e26)).
  - Grafana's April 2025 incident is this exact shape. An app key stored as an Actions secret leaked through a `pull_request_target` injection. The attacker minted a token and pushed a workflow that dumped every secret ([E27](#e27)).
  - Every org that adopts J1–J3 adds two more copies of the key.
- **(C) removes the outside users from the blast radius today, at no build cost.**
  - The `repobuddy` key then lives only in the Stage 2 Worker, which must hold it anyway for one-click installs (#40).
  - A leaked CI key reaches only the CI app's installations.
  - The CI app has to be public, because a private app can be installed only on the account that owns it ([E12](#e12)). Anyone can install it, but nobody outside has a reason to.
  - The cost: J1–J3 commits are authored by the CI app's bot, not `repobuddy[bot]`. That conflicts with the outcome #79 states.
- **(B) is sound, but today it means running security-critical code yourself.**
  - octo-sts is active and Chainguard maintains it (v0.14.3, 2026-11-06) ([E21](#e21)). It had 7 security advisories, 5 of them in the last two months, including a high-severity policy bypass on 2026-09-15 ([E22](#e22)).
  - The hosted instance is Chainguard's own app, so pushes come from `octo-sts[bot]` and Chainguard holds the key ([E23](#e23)).
  - Self-hosting runs Go servers on GCP or AWS. No Workers port exists ([E24](#e24)).
  - On Workers you would write the exchange yourself, about 200 lines on `jose` and `@octokit/auth-app` ([E25](#e25)).
  - D would replace all of it.
- **Dependabot is a gap for every OIDC-based option.** GitHub's docs list what Dependabot-triggered runs lose: a read-only `GITHUB_TOKEN` by default, and only Dependabot secrets ([E17](#e17)). They do not say whether `id-token: write` works there ([E18](#e18)). Test it before J3 depends on B or D, or remove J3 by making Renovate the only updater ([E32](#e32)).

### Comparison

| | (A) Key copied into orgs | (B) Exchange service, one app | (C) CI app plus public bot app | (D) GitHub keyless app tokens |
|---|---|---|---|---|
| Available | Yes, today | Yes: octo-sts and forks, or build your own | Yes, today | **No.** On the roadmap as a public preview since 2026-06-12 ([E3](#e3)) |
| Blast radius of a stolen secret | Every installation, outside users' included, with all app permissions, until the key is rotated | A stolen workflow token gives one repository, at the policy's permissions, for 1 hour. A breach of the service gives every installation | A stolen CI key gives the owner's orgs, and any stranger who installed the CI app. A breach of the bot service gives every `repobuddy` installation | Expected to match B's per-repo case, with GitHub running the exchange. Not yet documented |
| Where the key lives | 2 secrets per org (Actions and Dependabot), up to 72 copies | One place: KMS, or a Worker secret | CI key in each org. `repobuddy` key only in the Worker | Nowhere outside GitHub |
| Operational cost | Paste a secret per org, and rotate across up to 72 secrets | Build and run the service, plus a trust policy per repo or org. On Workers the free plan covers the request volume, but 10 ms of CPU for RS256 verify and sign is unmeasured | One more app to own: install it on 36 orgs and keep its secrets. No service | Turn it on and write admin policies. No service |
| Renovate PRs (J1, J2) | Works | Works. The PR is a same-repo `pull_request` run, so OIDC is available | Works | Should work, like B |
| Dependabot PRs (J3) | Works with the Dependabot copy of the secret | **Unconfirmed.** OIDC on Dependabot runs is undocumented ([E18](#e18)) | Works with the Dependabot copy of the CI key | **Unconfirmed**, as for B |
| Commit author | `repobuddy[bot]` | `repobuddy[bot]` if self-hosted, `octo-sts[bot]` if hosted | The CI app's bot | `repobuddy[bot]` |
| Maturity | Stable. `create-github-app-token` v3.3.0, 2026-10-12 ([E6](#e6)) | octo-sts is active, but advisories are frequent ([E21](#e21), [E22](#e22)). Every other tool is under a year old with a handful of stars ([E24](#e24)) | Stable, the same tools as A | Not released |
| Lock-in | Low | octo-sts policy format, or your own code. The hosted instance ties your identity to Chainguard's app | Low | Low beyond Actions, which J1–J3 already need |

The other approaches:

| | Approach | Why not, or when |
|---|---|---|
| (E) | `workflow_dispatch` with `GITHUB_TOKEN`, as #40 plans for outside users | No key at all. Checks from workflows that run only on `pull_request` (CodeQL, for example) never report ([E1](#e1), #40). Keep it as the fallback for repositories with no app. |
| (F) | One private CI app per org | Each org's blast radius ends at that org, but that means 36 apps, 36 keys, and 36 sets of settings to keep in step. Not worth it while C or D is available. |
| (G) | Fine-grained PAT | It belongs to a user, so commits show that user. It cannot span orgs yet: that is roadmap item #1118 ([E13](#e13)). It is worse than any app option. |
| (H) | Vault or OpenBao with `vault-plugin-secrets-github` (v2.4.0, 2026-11-05) | A mature way to run B, but it means running Vault. Too heavy for one owner ([E24](#e24)). |

## Recommendation

### Now, while no service exists

1. **Keep the `repobuddy` key out of org secrets.** This is the owner's call, because it changes the plan in repobuddy/.github#79:
   - #79 currently plans to store the `repobuddy` key in every org (#85, #86) and retire `cyberuni-ci-bot` (#88).
   - Instead, give J1–J3 a dedicated CI app. `cyberuni-ci-bot` already has the permissions they need (contents and pull requests write), so it could fill that role, renamed or not. Alternatively, register a new app.
   - Install the CI app on the orgs that run J1–J3, and store its key there.
   - Do this before the `repobuddy` app is promoted to outside users, or before it shows up in any outside account.
2. **Shrink the CI key's reach in each org.** None of these changes needs a service:
   - Set each org secret's visibility to the repositories that run J1–J3, not to all repositories.
   - Mint narrow tokens. Set `repositories:` to the current repository and `permission-contents: write` in `create-github-app-token` ([E6](#e6)).
   - Never expose the key to a `pull_request_target` workflow, or to any workflow that runs untrusted PR code. GitHub blocks `pull_request_target` by default in public repositories from 2026-12-02 ([E15](#e15)).
   - Pin `create-github-app-token` by SHA, in line with GitHub's SHA-pinning policy ([E14](#e14)).
3. **Drop the Dependabot copy of the CI key where you can.** Making Renovate the only updater (second-brain research R4, [E32](#e32)) removes J3 and every Dependabot secret. Where Dependabot stays, the Dependabot copy is the price of J3 under C.
4. **Teach Renovate to ignore the bot's commits.** Renovate stops updating a PR once another author has pushed to it. Add the CI bot's commit email to `gitIgnoredAuthors` in the shared preset ([E20](#e20)). Dependabot likewise stops rebasing a PR after extra commits ([E19](#e19)).
5. **Use more than one key.** GitHub allows up to 25 keys per app ([E10](#e10)). Give each group of orgs its own CI key, so a leak can be revoked for one group without rotating every org.
6. **Store the `repobuddy` key in Workers as PKCS#8** when Stage 2 is built ([probot-and-alternatives.md, E7](probot-and-alternatives.md#e7)). Keep the GitHub-side copy out of every repository and every org secret.

### From today's model to the target

| Step | Trigger | Change | Key copies afterwards |
|---|---|---|---|
| 1 | Now | Split identities (C). J1–J3 use the CI app. The `repobuddy` key is in no org | CI key: 2 per adopting org. `repobuddy`: none until Stage 2, then 1 in the Worker |
| 2 | Now | Harden the CI key (items 2–5 above) | Same count, smaller reach |
| 3 | When GitHub opens the keyless public preview ([E3](#e3)) | Pilot it in one org: run J1 on a Renovate PR and J3 on a Dependabot PR. Check that a token from an app owned by `repobuddy` works for an installation on another org, and that its push starts CI | Unchanged during the pilot |
| 4 | When the pilot passes | Move J1–J3 to keyless tokens. Decide whether they can run as `repobuddy` again, which restores #79's outcome with no key copies, and delete the CI key secrets | CI key: none. `repobuddy`: 1 in the Worker |
| 5 | **Fallback.** Outside users are about to be invited, D is not yet usable, and CI pushes must come from `repobuddy[bot]` | Build B. Either add an OIDC exchange to the Worker, or self-host octo-sts with the `repobuddy` app. Write trust policies in each org's `.github` repository, in octo-sts format, so moving to D later is a policy change | `repobuddy`: 1, in the service |

On step 5: adding the exchange to the Worker puts a new public endpoint on the component that holds the most sensitive key. A separate Worker with its own copy of the key keeps the two apart, at the cost of a second copy. Either way, read octo-sts's advisories ([E22](#e22)) first. Its high-severity bypass came from regex anchoring in its claim matching, which a hand-written policy matcher would get wrong in the same way.

## Confidence

- **High** for the problem with A, for C being available today, and for D not having shipped:
  - The roadmap items, the changelog, and the `create-github-app-token` issue are primary sources, checked on 2026-11-14 ([E3](#e3)–[E6](#e6)).
  - The incident record is consistent ([E26](#e26)–[E31](#e31)).
- **Medium** for the timing of D. A roadmap item is not a date, and the item has not been updated since 2026-09-03.
- **Low** for any OIDC-based option on Dependabot PRs. No source settles it ([E18](#e18)).

## Evidence

Items dated before 2025-11-14, more than 12 months ago, are marked **(possibly stale)**.

### GitHub platform

<a id="e1"></a>**E1.** Events triggered by `GITHUB_TOKEN` start no new workflow runs, except `workflow_dispatch` and `repository_dispatch`. The docs point to an app token or a PAT instead. No 2025–2026 change was found, and no option or permission lets a `GITHUB_TOKEN` push start CI. Sources: [GITHUB_TOKEN docs](https://docs.github.com/en/actions/concepts/security/github_token) (current, 2026-11); [changelog 2022-09-08](https://github.blog/changelog/2022-09-08-github-actions-use-github_token-with-workflow_dispatch-and-repository_dispatch/) **(possibly stale, but unchanged in current docs)**.

<a id="e2"></a>**E2.** Since 2026-06-11, pull requests created by `github-actions[bot]` can run workflows once a user with write access approves them. The post names only `github-actions[bot]`. It does not say whether pushes to an existing PR, or app tokens, are covered. Source: [changelog 2026-06-11](https://github.blog/changelog/2026-06-11-bot-created-pull-requests-can-run-workflows-if-approved/).

<a id="e3"></a>**E3.** Roadmap issue github/roadmap#1271, "Keyless GitHub App Tokens in Actions [Public Preview]". It was opened 2026-06-12, last updated 2026-09-03, and is still open, labelled Free, Team, and Enterprise. Quote: "securely run workflows that act as a GitHub App without needing to store or manage private keys as secrets … By allowing only trusted workflows to mint tokens based on admin-defined policies." No changelog post or docs page for it exists as of 2026-11-14. Source: [github/roadmap#1271](https://github.com/github/roadmap/issues/1271) (read through the GitHub API).

<a id="e4"></a>**E4.** Roadmap issue github/roadmap#1288, "GitHub Apps support arbitrary issuers for federated credentials [GA]". It was opened 2026-08-10, last updated 2026-09-23, and is still open. Apps would authenticate with any compatible identity provider (SPIFFE, Okta, GCP, AWS, Kubernetes) "to mint installation tokens for your app's installations" instead of a private key. Source: [github/roadmap#1288](https://github.com/github/roadmap/issues/1288).

<a id="e5"></a>**E5.** On 2026-10-13, app credentials moved to a Credentials tab, with Client secrets and Key pairs. The change was made "in advance of our planned workload federation features", and the tab "sets aside space for our upcoming workload federation credentials". The same post adds `repository_creation` and `installation_repositories` permissions. Source: [changelog 2026-10-13](https://github.blog/changelog/2026-10-13-github-app-improvements-for-least-privilege-and-management/).

<a id="e6"></a>**E6.** `actions/create-github-app-token` v3.3.0 (2026-10-12) has no OIDC mode. Issue #411, "Allow App to auth using Actions OIDC (Feature is not GA yet)", was opened on 2026-11-12. The action's `repositories` and `permission-<name>` inputs narrow a token. Sources: [issue #411](https://github.com/actions/create-github-app-token/issues/411), [README](https://github.com/actions/create-github-app-token).

<a id="e7"></a>**E7.** OIDC tokens can carry repository custom properties as claims (`repo_property_*`): public preview on 2026-03-12, GA on 2026-04-02. A trust policy could match "repositories tagged for J1" instead of listing names. Sources: [changelog 2026-03-12](https://github.blog/changelog/2026-03-12-actions-oidc-tokens-now-support-repository-custom-properties/), [changelog 2026-04-02](https://github.blog/changelog/2026-04-02-github-actions-early-april-2026-updates/).

<a id="e8"></a>**E8.** Immutable subject claims, announced 2026-04-23, put owner and repository IDs into `sub`. They are the default for repositories created after 2026-07-15, and existing repositories opt in. Any trust policy that matches on `sub` must expect both forms. Source: [changelog 2026-04-23](https://github.blog/changelog/2026-04-23-immutable-subject-claims-for-github-actions-oidc-tokens/).

<a id="e9"></a>**E9.** Installation tokens became stateless on 2026-10-02. They are about 520 characters long, still start with `ghs_`, still expire after 1 hour, and are scoped as before. Treat them as opaque strings. Source: [changelog 2026-10-02](https://github.blog/changelog/2026-10-02-stateless-github-app-installation-tokens-rolled-out/).

<a id="e10"></a>**E10.** An app can have up to 25 private keys, and GitHub encourages several keys for rotation without downtime. Source: [changelog 2024-11-08](https://github.blog/changelog/2024-11-08-updated-limits-for-github-app-private-keys-and-scoped-tokens/) **(possibly stale)**.

<a id="e11"></a>**E11.** GitHub's key-handling docs call the private key the app's most valuable secret. They say keys do not expire, rate an environment variable as weaker than a key vault, and recommend a vault that can sign but never reveals the key (Azure Key Vault is the example). Source: [Managing private keys for GitHub Apps](https://docs.github.com/en/apps/creating-github-apps/authenticating-with-a-github-app/managing-private-keys-for-github-apps) (current docs).

<a id="e12"></a>**E12.** An app with "Only on this account" visibility "can only be installed on the account that created it". "Any account" allows installation anywhere. Only an enterprise-owned app can be limited to a set of orgs. So a CI app shared by 36 orgs outside an enterprise must be public. Source: [Installing your own GitHub App](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app) (docs source last changed 2026-03-17).

<a id="e13"></a>**E13.** Fine-grained PATs reached GA on 2025-03-18 **(possibly stale)**. Support for several orgs at once is still an open roadmap item (#1118, updated 2026-09-23). Sources: [changelog 2025-03-18](https://github.blog/changelog/2025-03-18-fine-grained-pats-are-now-generally-available/), [github/roadmap#1118](https://github.com/github/roadmap/issues/1118).

<a id="e14"></a>**E14.** Actions policy can block actions and require SHA pinning, since 2025-08-15. Source: [changelog 2025-08-15](https://github.blog/changelog/2025-08-15-github-actions-policy-now-supports-blocking-and-sha-pinning-actions/) **(possibly stale, about 15 months old)**.

<a id="e15"></a>**E15.** `pull_request_target` is blocked by default in public repositories from 2026-12-02, unless an Actions event policy allows it. Brownouts ran on 2026-11-09 and run again on 2026-11-16, both 16:00–18:00 UTC. Private and internal repositories are not affected. Separately, from 2025-12-08 its workflow file and checkout always come from the default branch. Sources: [changelog 2026-11-02](https://github.blog/changelog/2026-11-02-pull_request_target-blocked-by-default-december-2/), [changelog 2025-11-07](https://github.blog/changelog/2025-11-07-actions-pull_request_target-and-environment-branch-protections-changes/).

<a id="e16"></a>**E16.** GitHub's 2026 Actions security roadmap (2026-03-26) plans scoped secrets bound to a repository, branch, environment, or workflow, with no implicit inheritance, along with workflow rulesets and dependency locking. Preview was due in 3 to 6 months. Scoped secrets would let the CI key in C be bound to the J1–J3 workflows only. Source: [GitHub blog 2026-03-26](https://github.blog/news-insights/product-news/whats-coming-to-our-github-actions-2026-security-roadmap/).

### Dependabot and Renovate

<a id="e17"></a>**E17.** For workflows that Dependabot starts with `pull_request`, `push`, and related events, "`GITHUB_TOKEN` has read-only permissions by default" and secrets "are populated from Dependabot secrets. GitHub Actions secrets are not available". These restrictions apply even when another actor re-runs the workflow. On `pull_request_target`, where the base ref was created by Dependabot, the token is read-only and no secrets are available. Source: [Dependabot on GitHub Actions](https://docs.github.com/en/code-security/reference/supply-chain-security/troubleshoot-dependabot/dependabot-on-actions) (docs source last changed 2026-10-15; the rules date from 2021).

<a id="e18"></a>**E18.** The OIDC reference mentions Dependabot only for Dependabot *update jobs*, whose tokens carry `event_name: dynamic`, used for private-registry auth. Neither the OIDC docs nor the Dependabot docs say whether a Dependabot-triggered workflow run can get an Actions OIDC token with `id-token: write`. No issue thread settling it was found. Sources: [OIDC reference](https://docs.github.com/en/actions/reference/security/oidc), E17.

<a id="e19"></a>**E19.** "Dependabot will stop rebasing a pull request once extra commits have been pushed to it." A commit message containing `[dependabot skip]` lets Dependabot force-push over the extra commits. Source: [Managing pull requests for dependency updates](https://docs.github.com/en/code-security/dependabot/working-with-dependabot/managing-pull-requests-for-dependency-updates) (current docs).

<a id="e20"></a>**E20.** Renovate treats a branch with commits from any other author as modified and stops updating it. `gitIgnoredAuthors` exempts given emails. `postUpgradeTasks`, which could write the changeset inside Renovate's own commit, is blocked on the Mend-hosted app unless the command is allowlisted. Sources: [Renovate configuration options](https://docs.renovatebot.com/configuration-options/#gitignoredauthors), [Mend-hosted FAQ](https://docs.renovatebot.com/mend-hosted/faq/).

### Token-exchange services

<a id="e21"></a>**E21.** `octo-sts/app`:

- Latest release v0.14.3 on 2026-11-06, with releases on 2026-10-22, 10-27, and 11-05 as well. Last commit 2026-11-13.
- 431 stars, Apache-2.0, written in Go, maintained by Chainguard staff.
- Trust policies live in `.github/chainguard/<name>.sts.yaml` in the target repository, or in the org's `.github` repository for org-level policies.
- A policy matches `issuer`, `subject` or `subject_pattern`, and `claim_pattern`. The regexes are auto-anchored.
- An org-level policy can name `repositories`, so one token can reach other repositories.
- `octo-sts/action` v1.1.1 (2025-12-03) takes `scope` and `identity` and returns `token`.

Sources: [octo-sts/app](https://github.com/octo-sts/app) README and releases, [octo-sts/action](https://github.com/octo-sts/action) (GitHub API, 2026-11-14).

<a id="e22"></a>**E22.** octo-sts has 7 security advisories:

- CVE-2025-52477: high-severity unauthenticated SSRF, 2025-06 **(possibly stale)**.
- GHSA-mwqh-2vg8-rhj3: high-severity regex-alternation anchoring bypass, fixed in 0.10.0 on 2026-09-15.
- A medium-severity policy disclosure, 2026-10-06.
- A medium-severity stale cache, 2026-11-05.
- A medium-severity blind SSRF through `jwks_uri`, 2026-11-06.
- A low-severity unbounded webhook body, 2026-11-06.
- A low-severity advisory from 2024 **(possibly stale)**.

Source: [octo-sts security advisories](https://github.com/octo-sts/app/security/advisories).

<a id="e23"></a>**E23.** The hosted service at octo-sts.dev works by installing Chainguard's public `octo-sts` app, so tokens and commits belong to that app's bot. That is inferred from the README's "install the octo-sts App" step. No page says it in so many words. The README says the hosted service does not support orgs with an IP allow list. No SLA was found. Chainguard's 2024 launch post names "a dedicated GitHub App (private key is long-lived)" as the option octo-sts replaces. Sources: [octo-sts/app README](https://github.com/octo-sts/app), [Chainguard 2024-04-11](https://chainguard.dev/unchained/the-end-of-github-pats-you-cant-leak-what-you-dont-have) **(possibly stale)**.

<a id="e24"></a>**E24.** Self-hosted octo-sts runs signed `app` and `webhook` images as long-running HTTP servers. Its Terraform targets GCP, with KMS and Firestore. No Cloudflare Workers port was found. Alternatives and forks:

| Tool | What it is | Last release | Stars |
|---|---|---|---|
| [DND-IT/terraform-aws-github-token-broker](https://github.com/DND-IT/terraform-aws-github-token-broker) | octo-sts on Lambda, key in non-exportable KMS. Its README describes replacing `create-github-app-token` with a key in each caller | v0.5.1, 2026-10-09 | 1 |
| [cruxstack/terraform-aws-octo-sts-serverless](https://github.com/cruxstack/terraform-aws-octo-sts-serverless) | octo-sts on Lambda | v1.1.3, 2026-11-13 | 1 |
| [sorah/sts-cat](https://github.com/sorah/sts-cat) | Rust reimplementation, Lambda or HTTP, TOML policies | v0.4.1, 2026-10-20 | 15 |
| [abinnovision/gh-token-broker](https://github.com/abinnovision/gh-token-broker) | Go, CEL policies held in the broker | v0.8.1, 2026-09-30 | 4 |
| [smpio/gha-token-broker](https://github.com/smpio/gha-token-broker) | Go, OIDC to a repository-scoped token | v1.0.0, 2026-10-25 | 0 |
| [martinbaillie/vault-plugin-secrets-github](https://github.com/martinbaillie/vault-plugin-secrets-github) | Vault or OpenBao plugin that mints installation tokens | v2.4.0, 2026-11-05 | 330 |

Source: GitHub API, 2026-11-14.

<a id="e25"></a>**E25.** `jose` v6.2.17 (2026-11-09) lists Cloudflare Workers as a supported runtime. `createRemoteJWKSet` and `jwtVerify` against the issuer `https://token.actions.githubusercontent.com` verify an Actions OIDC token with WebCrypto. Signing the app JWT on Workers needs the key in PKCS#8 ([probot-and-alternatives.md, E7](probot-and-alternatives.md#e7)). Source: [panva/jose](https://github.com/panva/jose).

### Incidents

<a id="e26"></a>**E26.** GitGuardian, 2026-09-22: of 4,802 leaked app keys tested, 474 (440 apps) still authenticated. Of those, 44 had org admin, 207 repository write, 98 workflow write, and 40 runner admin. 156 keys leaked in repositories the app's maintainer does not own. Named cases: a CDC app key leaked in 2025-04 and revoked on 2026-09-18, and a BuildBuddy development app installed on 10 orgs. No misuse was confirmed. The write-up recommends rotation and monitoring, and does not mention OIDC. Source: [GitGuardian](https://blog.gitguardian.com/github-app-private-keys-leaked/).

<a id="e27"></a>**E27.** Grafana, detected 2025-04-26 and disclosed 2025-04-28. A `pull_request_target` workflow with a script injection leaked `GRAFANA_DELIVERY_BOT_APP_ID` and `_APP_PEM`, both stored as Actions secrets. The attacker minted an app token and pushed a workflow that exported every Actions secret. Grafana reports no production or customer impact. Sources: [StepSecurity](https://stepsecurity.io/blog/grafana-github-actions-security-incident), [Grafana](https://grafana.com/blog/grafana-security-update-no-customer-impact-from-github-workflow-vulnerability/).

<a id="e28"></a>**E28.** Grafana again, around 2026-05-17. A pwn-request workflow led to a stolen privileged GitHub token, its source code was downloaded, and the attackers demanded a ransom. Sources disagree on whether the token came from an app or a PAT. Secondary sources only, low confidence. Source: [TechRadar](https://www.techradar.com/pro/security/grafana-says-hackers-hit-its-github-environment-demand-ransom-to-prevent-codebase-release-but-its-refusing-to-pay).

<a id="e29"></a>**E29.** The tj-actions chain:

- 2024-12-06: a SpotBugs maintainer's PAT leaked through `pull_request_target`.
- 2025-03-11: a reviewdog PAT was stolen.
- 2025-03-14: the `tj-actions/changed-files` tags were repointed, and secrets were printed into the logs of the more than 23,000 repositories that used the action.

Unit 42 recommends SHA pinning, a least-privilege `GITHUB_TOKEN`, and short-lived credentials. Source: [Unit 42](https://unit42.paloaltonetworks.com/github-actions-supply-chain-attack/) **(possibly stale)**.

<a id="e30"></a>**E30.** Workflow-secret theft at scale in 2025:

- Nx "s1ngularity", 2025-08-26: `pull_request_target` with a PR-title injection exposed the npm token ([postmortem](https://nx.dev/blog/s1ngularity-postmortem)).
- GhostAction, 2025-09-05: malicious workflows took 3,325 secrets from 817 repositories ([GitGuardian](https://blog.gitguardian.com/ghostaction-campaign-3-325-secrets-stolen/)).
- Shai-Hulud 2.0, 2025-11-21 to 23: injected secret-dumping workflows and backdoor self-hosted runners in more than 25,000 repositories ([Wiz](https://www.wiz.io/blog/shai-hulud-2-0-ongoing-supply-chain-attack)).

The details come from search summaries, so treat them as medium confidence.

<a id="e31"></a>**E31.** OIDC tokens can be stolen too, within a run:

- TanStack, 2026-05-11: `pull_request_target` and cache poisoning led to an OIDC publishing token being read from runner memory, producing packages with valid provenance ([Snyk](https://snyk.io/blog/tanstack-npm-packages-compromised/)).
- AntV, 2026-05-19: masked secrets were read from runner process memory.
- ChainDrop, 2026-08-04: a self-propagating worm that steals CI credentials ([Microsoft](https://www.microsoft.com/en-us/security/blog/2026/08/04/chaindrop-supply-chain-compromise-anatomy-self-propagating-worm/)).

Under B or D, a stolen OIDC token is worth one repository for minutes. A stolen key is worth every installation, with no expiry. Medium confidence, from search summaries.

### Prior research

<a id="e32"></a>**E32.** Second-brain research `cyberuni-dep-pr-automerge` (2026-11-08), recommendation R4: make Renovate the only updater. Turn off Dependabot security updates, keep the alerts, and delete `dependabot.yml`. Source: `unional/second-brain`, `.research/cyberuni-dep-pr-automerge/conclusion.md` (private).

## Weakening and conflicting evidence

- **D may be close, or far.** The roadmap item is labelled Public Preview, but nothing has shipped, and the item has not moved since 2026-09-03 ([E3](#e3)). If it lands soon, C is a detour. If it slips by a year, C is how the app runs for that year.
- **C breaks #79's outcome.** #79 wants J1–J3 commits authored by `repobuddy[bot]`. Under C they are not, until D lets the CI jobs come back to `repobuddy` (step 4).
- **The CI app is public too.** A leaked CI key reaches any stranger who installed the CI app ([E12](#e12)). The app is never advertised, but nothing stops an install. Only an enterprise-owned app can be limited to a set of orgs.
- **OIDC is no cure-all.** In 2026, attackers read OIDC tokens and masked secrets from runner memory ([E31](#e31)). B and D shrink what a theft gets you; they don't stop the theft.
- **Building B well is hard.** octo-sts, maintained full time by a security company, shipped a high-severity policy bypass in 2026 ([E22](#e22)).

## Not supported

- That GitHub has shipped keyless app tokens, OIDC-federated app credentials, or any way to trade an OIDC token for an installation token. Only roadmap items exist ([E3](#e3)–[E5](#e5)).
- That any `GITHUB_TOKEN` option makes a push start CI ([E1](#e1)).
- That a leaked app key has been used against outside users' installations. The incidents found hit the key owner's own repositories ([E26](#e26), [E27](#e27)).
- That a maintained token-exchange service runs on Cloudflare Workers ([E24](#e24)).

## Where evidence is thin

- **OIDC on Dependabot-triggered runs** ([E18](#e18)). Test it in one repository: on a Dependabot PR, run a `pull_request` workflow with `permissions: id-token: write` and check whether `ACTIONS_ID_TOKEN_REQUEST_URL` is set. If it is not, J3 under B or D needs a `workflow_run` follow-up or no Dependabot at all. Whether `workflow_run` gets full secrets after a Dependabot run was not confirmed either. It is not in the restricted event list ([E17](#e17)).
- **How D will behave across orgs.** Whether a keyless token works for an app owned by `repobuddy` and installed on another org, and who writes the policy, the app owner or the installing org. The roadmap text does not say ([E3](#e3)).
- **Whether app-token pushes onto a bot PR need approval** under the 2026-06-11 change. The post names only `github-actions[bot]` ([E2](#e2)).
- **CPU time on Workers** for one RS256 verification plus one RS256 signature, against the free plan's 10 ms. Not measured here or in the Probot research.
- **Incident details** for E28, E30, and E31 come from secondary sources or search summaries.

## Check again later

- github/roadmap#1271 and #1288, the GitHub changelog for "keyless" or "workload federation", and `create-github-app-token` issue #411. Any of these moving starts step 3.
- GitHub's Actions security roadmap for scoped secrets ([E16](#e16)). They would let C's CI key be bound to the J1–J3 workflows.
- octo-sts advisories and releases, if B becomes the fallback.
- The Dependabot docs, for any statement on OIDC in Dependabot-triggered runs.
