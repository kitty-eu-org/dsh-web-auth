# dsh-web-auth

Password gate for the [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) Web shell — a login page in front of the GUI, as a standalone installable package.

The official `@deepseek-ai/dsh-host-webserver` has no request-interception extension, so a password gate cannot be built as a pure plugin against it. This package ships a **drop-in webserver fork** that adds one extension point — `webServer.registerGate(gate)`, a request-gate seat running before route matching on every HTTP request and WebSocket upgrade — plus the **auth plugin** (`dsh-web-auth/web-auth`) that implements the gate: a `/login` page, an HMAC-signed session cookie, and 401 for `/api` and upgrade requests.

Everything else (SPA dist serving, `/api` bridge, WebSocket downlinks, trust fence) is untouched and comes from the official packages as usual. Verified against the official `@deepseek-ai/dsh` (`0.1.0-rc.6`): SPA, `/api`, and WebSocket all work behind the gate.

## Prerequisites

- **Node.js** `^22.19 || >=24` (the official dsh requirement)
- The official CLI works at least once: `npx @deepseek-ai/dsh web` (stop it with `Ctrl+C` after it prints the URL line) — this creates your harness home and the `web` profile
- Your harness home defaults to `~/.dsh` (override with the `DSH_HOME` environment variable); the profile lives at `~/.dsh/profiles/web`

You do **not** need pnpm installed — the install command below runs pnpm 11 via `npx`, matching the pnpm major the official dsh declares. Using a locally installed pnpm of a different major against the same profile produces `ERR_PNPM_UNEXPECTED_STORE` (see Troubleshooting).

## Quick start

### 1. Install the package

```sh
cd ~/.dsh/profiles/web && npx -y pnpm@11 add -w dsh-web-auth
```

Why this exact command:

- `dsh-web-auth` is a **plugin library, not a CLI** — it has no `bin` and cannot be "run" with `npx`. It is loaded by the cordis loader from `node_modules` by name, so it must be *installed* into the profile with a package manager, which is what `add` does.
- `npx -y pnpm@11` runs **pnpm 11** without requiring pnpm on your machine. The official dsh declares `packageManager: pnpm@11.7.0`; a locally installed pnpm of a different major operating on the same `node_modules` fails with `ERR_PNPM_UNEXPECTED_STORE` (see Troubleshooting). If you already have pnpm 11 (`npm install -g pnpm@11`), the plain `pnpm add -w dsh-web-auth` is equivalent.
- The `cd` is required: newer pnpm (11.21+) rejects `--workspace-root` unless the *current* directory is inside a workspace, and `--dir` does not satisfy that check.
- `-w` itself is required: the profile is a pnpm workspace root, and adding to it without `-w` fails with `ERR_PNPM_ADDING_TO_ROOT`. `dsh plugin --profile web add ...` hits the same wall.

### 2. Configure

Replace the contents of `~/.dsh/profiles/web/cordis.patch.yml` with:

```yaml
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

What this does: disables the official webserver row (it has no gate), inserts the gated fork in its place, and mounts the auth plugin. The password is read from the `DSH_WEB_PASSWORD` environment variable at startup — never commit it to a config file.

### 3. Start

```sh
DSH_WEB_PASSWORD='your-password' npx @deepseek-ai/dsh web
```

### 4. Verify

- Open `http://127.0.0.1:3080` — it must redirect to a login page ("此界面受密码保护").
- Enter the wrong password — it must show "密码错误，请重试。".
- Enter the right password — you land on the GUI.
- Without a session cookie, `/api` requests and WebSocket connections are refused with 401.

Optional: set `DSH_WEB_SESSION_SECRET` to a random string so sessions survive restarts (without it, everyone logs in again after a restart).

## Remote access

- **LAN / Tailscale**: change `host` to `'0.0.0.0'` in the `webserver-gated` row above and restart. The `/api` trust fence automatically trusts the machine's own LAN/Tailscale IPs when the server binds `0.0.0.0`. Tailscale traffic is encrypted by WireGuard, so the password stays safe on that path.
- **Internet**: front the server with TLS — Cloudflare Tunnel or a reverse proxy — and keep the origin bound to `127.0.0.1`. The login page works behind TLS without changes.
- **Custom hostname / public domain**: pass `--trusted-host <host>` to `dsh web` (or add the authority to the `connection` row's `trustedHosts`), otherwise `/api` answers 403.

## Troubleshooting

- **`ERR_PNPM_UNEXPECTED_STORE` when running `pnpm add`** — your `node_modules` was linked by a different pnpm major. Use pnpm 11 (`npm install -g pnpm@11`), which is the official dsh requirement.
- **`--workspace-root may only be used inside a workspace`** — newer pnpm requires the *current* directory to be inside a workspace for `-w`; `cd ~/.dsh/profiles/web` first, then run the `add` from there (the README command already does this).
- **Login page does not appear** — the gate is not armed. Check that `dsh-web-auth` is in `~/.dsh/profiles/web/package.json` and that the patch file has no YAML errors.
- **`/api` answers 403 through a custom host** — the trust fence needs the authority; see "Custom hostname" above.
- **Port 3080 already in use** — another `dsh web` is running; stop it (`pkill -f "dsh web"`) or pass `--port <other>`.
- **Verify your composition without starting the server**: `npx @deepseek-ai/dsh web --dump-config` shows the merged rows — you should see `webserver` marked `disabled: true`, plus `webserver-gated` and `web-auth` inserted.
- **Official dsh updated** — the webserver fork is a small delta over `@deepseek-ai/dsh-host-webserver` (`registerGate` + upgrade gating). If the official package changes, sync `src/webserver.ts` from upstream and re-run `npm run build` (see `npm run build` below).

## Building from source

```sh
npm install
npm run build      # tsc -> lib/
```

## How it works

- `dsh-web-auth` (default export): `WebServer`, the official webserver plus `registerGate(gate)` — a single-owner seat consulted before route matching on every request and before upgrade dispatch on every upgrade. Denials own writing the response (the real `ServerResponse` on HTTP, a raw-socket adapter on upgrades).
- `dsh-web-auth/web-auth`: the auth plugin. Idle when no password is configured; when armed it whitelists `/login`, admits valid-cookie requests, answers `/api` and upgrades with 401, and redirects other GET/HEAD navigations to `/login`. Password and signature comparisons are constant-time.

## License

MIT. The webserver fork derives from `@deepseek-ai/dsh-host-webserver` (MIT, DeepSeek Harness); the auth plugin is original.
