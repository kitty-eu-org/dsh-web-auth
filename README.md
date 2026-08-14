# dsh-web-auth

Password gate for the [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) Web shell, as a standalone installable package.

The official `@deepseek-ai/dsh-host-webserver` has no request-interception extension, so a password gate cannot be built as a pure plugin against it. This package ships a **drop-in webserver fork** that adds one documented extension point — `webServer.registerGate(gate)`, a request-gate seat running before route matching on every HTTP request and WebSocket upgrade — plus the **auth plugin** (`dsh-web-auth/web-auth`) that implements the gate: a `/login` page, an HMAC-signed session cookie, and 401 for `/api` and upgrade requests.

Everything else (SPA dist serving, `/api` bridge, WebSocket downlinks, trust fence) is untouched and comes from the official packages as usual. Verified against the official `@deepseek-ai/dsh` (`0.1.0-rc.6`): SPA, `/api`, and WebSocket all work behind the gate.

## Install

The official CLI refuses to install into its profile via `dsh plugin` (the profile is a pnpm workspace root), so add the package manually:

```sh
npx @deepseek-ai/dsh web          # run once so the profile initializes
pnpm --dir "$DSH_HOME/profiles/web" add -w dsh-web-auth
```

If `DSH_HOME` is unset it defaults to the platform harness home; the profile lives under `$DSH_HOME/profiles/web`.

## Configure

Edit `$DSH_HOME/profiles/web/cordis.patch.yml`:

```yaml
# Replace the official webserver row (no request-gate extension) with the
# gated fork and mount the password gate.
- id: webserver
  disabled: true
- insert:
    - id: webserver-gated
      name: 'dsh-web-auth'
      inject: [webStartup]
      config:
        host: !!js ctx.webStartup.host ?? '127.0.0.1'
        port: !!js ctx.webStartup.port ?? 3080
    - id: web-auth
      name: 'dsh-web-auth/web-auth'
      config:
        password: !!js process.env.DSH_WEB_PASSWORD
```

Read the password from the environment (never a committed config file) and start:

```sh
DSH_WEB_PASSWORD='your-password' npx @deepseek-ai/dsh web
```

Open `http://127.0.0.1:3080` — the surface redirects to `/login` until the password is entered. `/api` and WebSocket requests are answered 401 without a valid session cookie; a successful login mints an `HttpOnly; SameSite=Strict` cookie signed with a random per-boot secret.

### Remote access

- **LAN / Tailscale**: bind all interfaces so other machines reach the server:
  ```yaml
  config:
    host: '0.0.0.0'
    port: !!js ctx.webStartup.port ?? 3080
  ```
  The `/api` trust fence automatically trusts the machine's own LAN/Tailscale IPs when the server binds `0.0.0.0` (the web profile prints them in the URL line). Tailscale traffic is encrypted by WireGuard, so the password and cookie stay safe on that path.
- **Internet**: front the server with TLS — Cloudflare Tunnel or a reverse proxy — so the password and cookie are not sent in cleartext. Keep the origin bound to `127.0.0.1` in that case.
- Accessing through a **custom hostname or public domain**: pass `--trusted-host <host>` to `dsh web`, or add the authority to the `connection` row's `trustedHosts`, or `/api` answers 403.

## Publish your own

```sh
npm install
npm run build
npm login
npm publish          # add --otp <code> if your account has 2FA
```

If the plain name is taken, publish under your own scope (`@<you>/dsh-web-auth`) and adjust the `name:` values in the patch above.

## How it works

- `dsh-web-auth` (default export): `WebServer`, the official webserver plus `registerGate(gate)` — a single-owner seat consulted before route matching on every request and before upgrade dispatch on every upgrade. Denials own writing the response (the real `ServerResponse` on HTTP, a raw-socket adapter on upgrades).
- `dsh-web-auth/web-auth`: the auth plugin. Idle when no password is configured; when armed it whitelists `/login`, admits valid-cookie requests, answers `/api` and upgrades with 401, and redirects other GET/HEAD navigations to `/login`. Password and signature comparisons are constant-time.

## Maintenance note

The webserver fork is a small delta over the official `@deepseek-ai/dsh-host-webserver` (`registerGate` + upgrade gating). When the official package updates, sync `src/webserver.ts` from upstream and re-run `npm run build`. The auth plugin is original and does not track upstream.

## License

MIT. The webserver fork derives from `@deepseek-ai/dsh-host-webserver` (MIT, DeepSeek Harness); the auth plugin is original.
