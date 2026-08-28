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

Set on the Worker (Settings > Variables), never in this repo:

| Variable | Notes |
|---|---|
| `GITHUB_CLIENT_ID` | From the OAuth App owned by the `bremen-global` GitHub org |
| `GITHUB_CLIENT_SECRET` | Same app. **Encrypt it.** |
| `ALLOWED_DOMAINS` | `bremen-global.de,*.bremen-global.de` |

**`ALLOWED_DOMAINS` is mandatory, not optional.** Upstream's README calls it
"optional but recommended", which understates it. In `src/index.js` the
server-side origin check is guarded by `domainPatterns.length` (~line 191), and so
is the client-side one that decides whether to post the token to the opener
(~line 134). With the variable empty **both checks are skipped**, and the popup
hands the access token to whatever origin messaged it — so any website could
collect a token with write access to `bremen-global/website`. Set it in the same
save as the client secret; never set the credentials and come back to it later.

The value deliberately excludes `bremen-global.pages.dev`: CMS sign-in works on
the real hostname only.

## The OAuth App

Owned by the **GitHub organisation**, not an individual account, so it survives
handover: <https://github.com/organizations/bremen-global/settings/applications>.
Its *Authorization callback URL* must be exactly
`https://cms-auth.it-954.workers.dev/callback`.

## Keeping up with upstream

This is a fork so that upstream fixes can be pulled in. The only local changes are
the two `wrangler.toml` lines above and this file — deliberately in a new file
rather than in upstream's README, so merges stay clean.
