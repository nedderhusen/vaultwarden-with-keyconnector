# Vaultwarden + Authentik SSO + Key Connector (homelab)

Self-hosted Vaultwarden with Authentik as the SSO identity provider, and an
open-source Key Connector so SSO users never need a separate master password.

## Architecture

```
client (browser/desktop/mobile)
   │  1. SSO login
   ▼
Authentik  ── OIDC ──►  Vaultwarden  ── validates its own tokens ──►  key-connector
                         (data/vaultwarden)                           (data/key-connector)
```

- Authentik authenticates the user and hands Vaultwarden an OIDC identity.
- Vaultwarden issues its *own* signed access token to the client (its normal
  API auth), and publishes an OIDC discovery document at `/identity`.
- `key-connector` trusts that Vaultwarden-issued token (not Authentik's) to
  store/retrieve one opaque encrypted key blob per user. It never sees vault
  contents or passwords.

## Status / why this isn't the official image

Key Connector support is **not merged into upstream Vaultwarden**. It's split
across two projects by [acul021](https://github.com/acul021):

- [dani-garcia/vaultwarden#7419](https://github.com/dani-garcia/vaultwarden/pull/7419) —
  open PR adding the server-side Key Connector protocol to Vaultwarden.
- [acul021/key-connector](https://github.com/acul021/key-connector) — the
  standalone key-storage service itself (AGPL-3.0, written independently so
  it isn't bound by Bitwarden's Key Connector license restrictions).
- [acul021/vaultwarden](https://github.com/acul021/vaultwarden) — a fork
  (`main` = upstream `main` + #7419, rebased/force-pushed on updates),
  published as `ghcr.io/acul021/vaultwarden:testing` (**amd64 only**).

This means you're running a third party's unmerged fork. Accepted here since
this instance is homelab-only, reachable exclusively over VPN. Re-evaluate if
that changes -- and switch to the official `vaultwarden/server` image (keeping
SSO, dropping Key Connector) once #7419 merges, or if exposure changes.

Known consequences of that choice:
- No security audit of this specific fork/service pair.
- `:testing` is a moving tag rebased onto upstream -- pin a digest once you're
  happy with a build, or updates can change under you without warning.
- Losing `KC_ENCRYPTION_KEY` permanently locks out every SSO/Key-Connector
  user's vault. Back it up as carefully as you would a master password.
- Org owners/admins are deliberately blocked from enrolling in Key Connector
  in this design -- they keep a master password regardless of SSO.

## 1. Configure Authentik

Create an **OAuth2/OpenID Provider**, then a matching **Application**.

**Provider** (Applications → Providers → Create → OAuth2/OpenID Provider):
- Name: `vaultwarden`
- Client type: `Confidential`
- Redirect URIs / Origins (strict, one per line):
  - `https://vault.example.internal/sso-connector.html` (web vault)
  - `bitwarden://sso-callback` (desktop/mobile apps)
  - `http://localhost:*` if you want the Bitwarden CLI to work too (regex
    match, since the CLI picks a random local port)
- Signing key: any available RSA key
- Scopes: `openid`, `email`, `profile` at minimum

**Application** (Applications → Applications → Create):
- Provider: the one you just created
- Note the generated **Client ID** and **Client Secret** -- these go in `.env`.

Your `SSO_AUTHORITY` is the provider's issuer URL, typically:
`https://authentik.example.internal/application/o/<slug>/`
(Vaultwarden appends `/.well-known/openid-configuration` itself; don't
include that suffix.)

## 2. Configure this deployment

```sh
cp .env.example .env
```

Fill in:
- `DOMAIN` — the HTTPS URL clients will use to reach Vaultwarden (TLS is
  terminated by your VPN gateway/reverse proxy in front of this; Web Crypto
  requires a secure context)
- `SSO_AUTHORITY`, `SSO_CLIENT_ID`, `SSO_CLIENT_SECRET` — from step 1
- `KEY_CONNECTOR_ORG_NAME` — shown to users during enrollment
- `KC_ENCRYPTION_KEY` — generate with:

  ```sh
  openssl rand -base64 32
  ```

## 3. Bring it up

```sh
docker compose up -d --build
```

`key-connector` is built from source (no published image exists yet) using
the git repo directly as the Docker build context, pinned to its `main`
branch. First build takes a few minutes (Rust). `vaultwarden` pulls the
prebuilt fork image.

Check both are healthy:

```sh
docker compose logs -f vaultwarden key-connector
curl http://127.0.0.1:8000/alive                       # from the LXC itself
docker compose exec key-connector wget -qO- http://localhost:8081/alive
```

If `key-connector` crash-loops with `502 Bad Gateway` fetching
`${DOMAIN}/identity/.well-known/jwks`, that's not a Docker networking problem
between the two containers -- Vaultwarden's OIDC discovery document reports
`jwks_uri` as your **public** `DOMAIN`, so key-connector has to reach that
same URL your reverse proxy serves, round-trip. Confirm the chain from
outside the LXC:

```sh
curl -v https://your-domain/identity/.well-known/jwks
```

A `502` with your proxy's name in the `server:` header means the proxy
matched the site but couldn't reach its upstream -- point it at this LXC's
LAN IP on port 8000, and make sure `docker-compose.yml`'s port binding
(`0.0.0.0:8000:80`) is reachable from wherever the proxy runs, not just from
loopback on the LXC. Once that curl returns `200`, restart `key-connector`.

## 4. Verify the flow

1. Visit `DOMAIN`, click "Enterprise Single Sign-On", enter your org's SSO
   identifier if prompted, and log in via Authentik.
2. On first SSO login you should be enrolled into Key Connector automatically
   (no "create a master password" screen) instead of the normal registration
   flow.
3. Log out and back in via SSO on a second device/client to confirm the key
   round-trips through `key-connector` correctly rather than prompting for a
   master password.
4. Only once this works reliably, set `SSO_ONLY=true` in `docker-compose.yml`
   to hide the password-login form for everyone. Keep at least one admin
   account with a master password (org owners/admins can't enroll in Key
   Connector by design) so you always have a break-glass path in.

## Backups

Back up, separately:
- `data/vaultwarden` — the vault database + attachments
- `data/key-connector` — `keyconnector.db` (the encrypted key blobs)
- `KC_ENCRYPTION_KEY` (from `.env`) — store this somewhere independent of
  both of the above (password manager, offline note, secrets vault). Losing
  it is unrecoverable for every SSO user.

## Upgrading

`acul021/vaultwarden:testing` is rebased and force-pushed, not append-only —
treat every pull as a new, unreviewed build. Pin to a digest
(`ghcr.io/acul021/vaultwarden@sha256:...`) once satisfied, and re-pull
deliberately rather than tracking `:testing` automatically. Rebuild
`key-connector` periodically the same way (`docker compose build --pull
key-connector`), and watch dani-garcia/vaultwarden#7419 for merge — once
official images support Key Connector, migrate off this fork.
