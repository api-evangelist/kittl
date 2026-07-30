---
name: Build and release a Kittl app
description: >-
  Scaffold a sandboxed Kittl app, declare the right SDK scopes in manifest.json, develop
  it against a local dev server, and push it through draft, upload and review with the
  Kittl CLI.
api: https://sdk-docs.kittl.dev/
surface: cli
operations:
  - kittl auth login
  - kittl app init
  - kittl app link
  - kittl app update
  - kittl app upload
  - kittl app release
  - kittl app delete
generated: '2026-07-19'
method: generated
source: https://sdk-docs.kittl.dev/getting-started/kittl-cli
---

# Build and release a Kittl app

Kittl apps are sandboxed web apps that run in an isolated iframe inside the Kittl editor
and talk to the host over `postMessage` through the global `kittl` object. This skill
covers the full publish loop.

## Preconditions

- A Kittl account.
- **Beta gating:** the account must be approved for app creation by a Kittl admin, or
  `kittl app init` will not succeed.

## Steps

1. **Install the CLI.**
   `npm install -g @kittl/cli` (or one-off: `npx @kittl/cli`). The binary is `kittl`.

2. **Sign in.** Run `kittl auth login`. It tries to open a browser and also prints a URL
   plus a short code you can complete on another device. A human must be present —
   non-interactive token auth is not supported, so this step cannot run unattended in CI.
   Confirm with `kittl auth whoami`.

3. **Initialize the project.** From the app root run `kittl app init`. It is interactive:
   choose or create a developer organization, then name a new app under it. On success it
   writes `.kittl/config.json` and creates the draft from your `manifest.json`.
   - If `.kittl/config.json` already exists the command stops immediately. Use
     `kittl app link --overwrite` to re-point an existing folder instead.
   - Commit `.kittl/config.json` — it holds identifiers, not secrets, and keeps teammates
     and CI linked to the same app.
   - In a fresh directory the CLI scaffolds a small Vite + HTML entrypoint layout. If
     `package.json` already exists scaffolding is skipped, but `manifest.json` is still
     created when missing.

4. **Declare scopes in `manifest.json`.** `config.scopes` is required. Request the
   smallest set that covers every SDK method you call — calling a method without its
   scope fails with an SDK error result (`isOk: false`) and a permission-denied message.
   See `scopes/kittl-scopes.yml` for the full method-to-scope mapping. A read-and-write
   design app needs `["design:state:read", "design:state:write"]`; add `uploads:create`
   if you upload images and `ai:credit:spend` if you call `kittl.ai`.
   Add `"$schema": "https://api.kittl.com/extensions/manifest/schema.json"` so your editor
   validates the file.

5. **Develop locally.** Install the draft to your own account (it stays private until
   approved), then pick the **"local development"** version from the editor's version
   dropdown. The editor mounts `localhost:5173` by default; override with
   `config.appDevelopment.local.port`, and set `requireHTTPS: true` when your dev server
   runs SSL or a third-party OAuth provider demands a secure origin.

6. **Push manifest changes.** After editing `manifest.json`, run `kittl app update`.

7. **Upload the build.** After your build script produces output, run `kittl app upload`
   (defaults to `dist/`; override with `--dist`).

8. **Submit for review.** Run `kittl app release`. This marks the draft ready for review
   and a Kittl admin takes over.

## Rules

- **A released version is immutable.** You cannot change its manifest or uploaded files
  afterward. Run `kittl app update` and `kittl app upload` *before* `kittl app release`
  if anything you care about is still only on disk. Further work continues on a new draft
  at the next auto-incremented version.
- Most commands accept `--non-interactive`; `auth login` does not.
- Use `--help` at each level for flags and subcommands.
- `kittl app delete [ID]` defaults to the app linked in `.kittl/config.json` — be
  deliberate.
