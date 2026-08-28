# Deployment of this fork

This is a fork of [sveltia/sveltia-cms-auth](https://github.com/sveltia/sveltia-cms-auth),
deployed as the OAuth relay that lets bremen.global editors sign in to Sveltia CMS
with their GitHub accounts. Upstream's README explains what the authenticator
does; this file records only what is specific to our deployment.

Everything below is configuration, not secrets. No credential belongs in this
repository.

## Where it runs

| | |
|---|---|
| Cloudflare account | the bremen.global organisation's own (owner `lt@ben-bremen.de`) |
| Worker name | `cms-auth` |
| URL | `https://cms-auth.it-954.workers.dev` |
| Deployed by | Cloudflare Workers Builds, from `main` of this repo |
| Consumed by | `public/admin/config.yml` in `bremen-global/website`, as `backend.base_url` |

## Two deliberate changes to `wrangler.toml`

**`name = "cms-auth"`, not upstream's `sveltia-cms-auth`.** Workers Builds derives
the Worker name from the repository, so every build logged *"Failed to match
Worker name … Overriding using the CI provided Worker name"*. That was more than
noise: a local `wrangler deploy` would have created a **second** Worker called
`sveltia-cms-auth`, with none of the environment variables set, while the real one
kept serving.

**`preview_urls = false`.** Preview URLs are additional hostnames on the same
Worker, and this Worker mints GitHub tokens with write access to a private
content repository. Nothing here needs them.

## Required environment variables

**Where each one lives is not a matter of taste.** Workers Builds deploys this
repo with `npx wrangler deploy`, and wrangler treats `wrangler.toml` as the source
of truth for `vars` — so a plain-text variable set only in the dashboard is erased
by the next build. Secrets are preserved ("Secrets not included in the file are
preserved from the previous version"), plain-text vars are not.

| Variable | Where | Why there |
|---|---|---|
| `ALLOWED_DOMAINS` | `wrangler.toml` `[vars]` | Must survive every deploy; see below. Public hostnames, nothing to hide. |
| `GITHUB_CLIENT_ID` | Dashboard **Secret** | Not actually sensitive, but stored as a secret so it survives deploys without publishing an account-specific value in this public fork. |
| `GITHUB_CLIENT_SECRET` | Dashboard **Secret** | Genuinely sensitive. Never in this repo. |

Do **not** add `GITHUB_CLIENT_ID` as a Text variable: it will disappear on the
next build, and the Worker will answer `MISCONFIGURED_CLIENT`. That happened once
already.

**`ALLOWED_DOMAINS` is mandatory, not optional.** Upstream's README calls it
"optional but recommended", which understates it. In `src/index.js` the
server-side origin check is guarded by `domainPatterns.length` (~line 191), and so
is the client-side one that decides whether to post the token to the opener
(~line 134). With the variable empty **both checks are skipped**, and the popup
hands the access token to whatever origin messaged it — so any website could
collect a token with write access to `bremen-global/website`.

That is exactly why it belongs in this file rather than the dashboard. A dashboard
value disappears silently on the next build while the secrets survive, which
leaves a *fully working* relay with its origin check switched off — the worst of
the possible states, and one nothing would alert on.

## The OAuth App

Owned by the **GitHub organisation**, not an individual account, so it survives
handover: <https://github.com/organizations/bremen-global/settings/applications>.
Its *Authorization callback URL* must be exactly
`https://cms-auth.it-954.workers.dev/callback`.

## Keeping up with upstream

This is a fork so that upstream fixes can be pulled in. The only local changes are
the two `wrangler.toml` lines above and this file — deliberately in a new file
rather than in upstream's README, so merges stay clean.