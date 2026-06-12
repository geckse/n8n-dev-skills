# Validating n8n Nodes — The Agent Self-Validation Protocol

How to *prove* a generated node is valid before declaring it done. Run the gates **in order**; do not proceed past a failing gate. Loop fix → re-run until each passes. Report a table of gates with PASS/FAIL and evidence (command + last relevant output line) — that table is the proof the node works.

## Gate Overview

| Gate | What | Command | Pass signal |
|------|------|---------|-------------|
| 0 | Structure sanity | (file inspection) | All checklist items true |
| 1 | Lint | `npm run lint:fix && npm run lint` | Exit 0, zero errors (fix warnings too) |
| 2 | Build / typecheck | `npm run build` | Exit 0 + `✓ Build successful` + dist paths exist |
| 3 | Cloud eligibility | `npx n8n-node cloud-support` | `✅ Cloud support is ENABLED` |
| 4 | Runtime smoke test | `npm run dev` | `Editor is now accessible`, node loads & executes |
| 5 | Release readiness | `npm run release` prechecks | Clean main branch + publish.yml present |
| 6 | Verification scan | `npx @n8n/scan-community-package <pkg>` | `✅ ... has passed all security checks` |

**There is no unit-test gate.** The `n8n-node` toolchain has no `test` command and scaffolded projects have no `test` script — CI gates are exactly lint + build (the scaffolded `.github/workflows/ci.yml` runs `npm run lint` then `npm run build`). Do not invent a jest/vitest harness; runtime validation is Gate 4.

## Gate 0: Structure Sanity (fast, no tools)

Check before running anything:

- [ ] Package name matches `^(n8n-nodes-|@[\w-]+/n8n-nodes-)` (scoped form is valid)
- [ ] `keywords` includes `n8n-community-node-package`
- [ ] `n8n.n8nNodesApiVersion: 1` and `n8n.strict: true` in package.json
- [ ] Every node/credential has a corresponding `dist/...js` entry in `n8n.nodes` / `n8n.credentials`
- [ ] `peerDependencies` is exactly `{ "n8n-workflow": "*" }` (plus `@n8n/ai-node-sdk` for AI sub-nodes only)
- [ ] `dependencies` is absent or empty (no runtime dependencies)
- [ ] No lifecycle scripts: `prepare`, `preinstall`, `install`, `postinstall`, `prepublish` etc. are forbidden
- [ ] No `<...>` or `{{...}}` template placeholders left anywhere in package.json
- [ ] `description` is non-empty; `repository` points to the (public) GitHub repo
- [ ] Node class name matches its filename (`Acme` ↔ `Acme.node.ts`); credential class ends in `Api`
- [ ] CHANGELOG.md updated if the version was bumped

Most of these are also lint errors — fixing them first avoids noise in Gate 1.

## Gate 1: Lint

```bash
npm run lint:fix    # = n8n-node lint --fix  (note: 'lint:fix', NOT 'lintfix')
npm run lint        # = n8n-node lint → runs ESLint on '.' (TS files AND package.json)
```

Pass criterion: exit code 0 and zero `error`-level findings. Fix the three `warn`-level rules too — verification reviewers expect them clean.

`n8n-node lint` does TWO things before reporting success:

1. **Strict-mode config guard** — if package.json has `"n8n": { "strict": true }`, it verifies `eslint.config.mjs` is byte-equivalent (whitespace-normalized) to the default two-line config (`import { config } from '@n8n/node-cli/eslint'; export default config;`). If modified or missing, it prints `Strict mode violation:` and exits 1 **without running ESLint**. Fix: `npx n8n-node cloud-support enable` (restores the default config). **Never edit eslint.config.mjs.**
2. **ESLint run** with `@n8n/eslint-plugin-community-nodes` recommended config, plus eslint/typescript-eslint recommended, eslint-plugin-import-x, `no-console: error`, and the legacy `eslint-plugin-n8n-nodes-base` community/credentials/nodes rule sets. `package.json` is linted too — the package-level rules fire there.

If the failure output contains `@n8n/community-nodes/no-restricted-imports` or `no-restricted-globals`, the CLI appends `⚠️ n8n Cloud compatibility issues detected` and suggests `n8n-node cloud-support disable`. Treat this as: **remove the dependency/global usage, do NOT disable cloud support** — unless the user explicitly opts out of n8n Cloud.

**Never suppress rules with inline `eslint-disable` comments.** They only silence local lint; n8n's verification scanner (Gate 6) runs with `allowInlineConfig: false` and ignores them entirely.

### Rule catalog: `@n8n/community-nodes/<rule>`

The plugin ships 37 rules; the active `recommended` config enables 36 (one, `node-class-description-icon-missing`, is deprecated in favor of `require-node-description-fields`). All are `error` severity unless marked **(warn)**. Rules marked **(fix)** are auto-fixable via `lint:fix`.

**package.json rules:**

| Rule | Enforces |
|------|----------|
| `package-name-convention` | Name is `n8n-nodes-*` or `@scope/n8n-nodes-*`; flags leftover `<...>` placeholders |
| `n8n-object-validation` | `n8n` object exists; `n8nNodesApiVersion` positive int; `nodes` non-empty array of `dist/` paths; `credentials` array of `dist/` paths; `strict` boolean |
| `valid-description` | Non-empty `description` field |
| `valid-peer-dependencies` **(fix)** | `peerDependencies` contains exactly `"n8n-workflow": "*"` (optionally the AI node SDK) |
| `no-runtime-dependencies` | `dependencies` must be empty or absent |
| `no-overrides-field` | No `overrides` field |
| `no-forbidden-lifecycle-scripts` | No `prepare`/`preinstall`/`install`/`postinstall`/`prepublish` etc. scripts |
| `no-template-placeholders` | No `<...>` or `{{...}}` strings anywhere in package.json |
| `require-community-node-keyword` **(warn, fix)** | `keywords` includes `n8n-community-node-package` |
| `ai-node-package-json` | AI sub-nodes: `n8n.aiNodeSdkVersion` and the `@n8n/ai-node-sdk` peer dependency must be declared together |

**Node rules (`*.node.ts`):**

| Rule | Enforces |
|------|----------|
| `require-node-description-fields` | Description defines `icon` AND `subtitle` |
| `node-filename-against-convention` | Filename matches `description.name` |
| `node-connection-type-literal` **(fix)** | Use `NodeConnectionTypes.Main` — never the string `'main'` — in `inputs`/`outputs` |
| `node-usable-as-tool` **(fix)** | The node class must declare `usableAsTool` (set `true` unless there's a reason not to) |
| `icon-validation` | Icon file exists, is `.svg`, uses the `file:` prefix; light/dark variants must be different files |
| `valid-credential-references` / `no-credential-reuse` | Every credential a node references must be a credential class in THIS package and listed in package.json — referencing another package's credential is flagged as a security issue |
| `require-continue-on-fail` | `execute()` checks `this.continueOnFail()` in a try/catch around item processing |
| `require-node-api-error` | catch blocks throw `NodeApiError`/`NodeOperationError`, never raw `Error` |
| `node-operation-error-itemindex` | Errors thrown inside item loops pass `{ itemIndex }` |
| `missing-paired-item` | Every `INodeExecutionData` pushed from `execute()` carries `pairedItem` |
| `no-deprecated-workflow-functions` | `request`→`httpRequest`, `requestWithAuthentication`/`requestOAuth1`/`requestOAuth2`→`httpRequestWithAuthentication`, `IRequestOptions`→`IHttpRequestOptions`; `copyBinaryFile`/`prepareOutputData` removed |
| `no-http-request-with-manual-auth` | Don't combine `this.getCredentials()` with `this.helpers.httpRequest()` — use `httpRequestWithAuthentication` |
| `webhook-lifecycle-complete` | Webhook trigger nodes must implement ALL of `webhookMethods.default.checkExists/create/delete` |
| `no-builder-hint-leakage` | No AI-builder hint artifacts left in code |
| `options-sorted-alphabetically` **(warn)** | Options arrays sorted alphabetically |
| `resource-operation-pattern` **(warn)** | More than 5 operations requires a Resource parameter |

**Credential rules (`*.credentials.ts`):**

| Rule | Enforces |
|------|----------|
| `cred-class-name-suffix` **(fix)** | Class name ends in `Api` |
| `cred-class-name-field-conventions` **(fix)** | `name` field ends in `Api` and starts lowercase |
| `cred-class-oauth2-naming` **(fix)** | OAuth2 credentials: class ends `OAuth2Api`, names include "OAuth2" |
| `cred-class-field-icon-missing` | Credential class has an `icon` property |
| `credential-documentation-url` **(fix)** | `documentationUrl` is a valid URL or lowercase slug |
| `credential-test-required` | Every credential has a `test` property or is covered by a node's `testedBy` (classes extending `oAuth2Api` are exempt — don't add `test` to them) |
| `credential-password-field` **(fix)** | Fields named like password/secret/token/cert/passphrase/apiKey get `typeOptions: { password: true }` |

**Cloud-only rules** (in `recommended` only; dropped by `recommendedWithoutN8nCloudSupport`):

| Rule | Enforces |
|------|----------|
| `no-restricted-imports` | Only relative imports plus this allowlist: `n8n-workflow`, `@n8n/ai-node-sdk`/`ai-node-sdk`, `lodash`, `moment`, `p-limit`, `luxon`, `zod`, `crypto`/`node:crypto`. Everything else (axios, form-data, fs, path, ...) is forbidden |
| `no-restricted-globals` | Forbidden globals: `setTimeout`, `setInterval`, `setImmediate`, `clearTimeout`, `clearInterval`, `clearImmediate`, `process`, `global`, `globalThis`, `__dirname`, `__filename` (use `sleep` from `n8n-workflow` instead of `setTimeout`) |

Also enforced by the base config: `no-console: 'error'` — remove all `console.log` from node code.

Agent loop: `npm run lint:fix` first (auto-fixes ~10 rules), then `npm run lint`, fix remaining findings by rule name using this catalog, re-run until exit 0.

## Gate 2: Build & Typecheck

```bash
npm run build        # = n8n-node build
```

What it does, in order: (1) deletes `dist/`, (2) runs `tsc` with the project's strict tsconfig — this IS the typecheck (`strict`, `noImplicitAny`, `noUnusedLocals`, `noImplicitReturns`, `strictNullChecks`), (3) copies static assets matching `**/*.{png,svg}` and `**/__schema__/**/*.json` into `dist/`.

Pass criteria:

- Exit code 0 and final line `✓ Build successful`. Any TypeScript error prints `TypeScript build failed` and exits 1.
- Every path listed in package.json `n8n.nodes` and `n8n.credentials` exists on disk after the build. Verify mechanically:

```bash
node -e "const p=require('./package.json'),fs=require('fs');const m=[...(p.n8n.nodes||[]),...(p.n8n.credentials||[])].filter(f=>!fs.existsSync(f));process.exit(m.length?(console.error('MISSING in dist:',m),1):0)"
```

- The node's `.svg` icon appears in `dist/nodes/<Name>/` (the build copies it; `icon-validation` lint catches the source side, this check catches the dist side).

For a fast typecheck-only loop while editing: `npx tsc --noEmit`. For longer sessions: `npm run build:watch` (= `tsc --watch`).

Do NOT write a custom build script (`tsc && gulp build:icons` is the legacy starter pattern; CLI projects use `n8n-node build`). Unused variables are hard errors (`noUnusedLocals`), so delete dead imports rather than suppressing them.

## Gate 3: Cloud Eligibility — `n8n-node cloud-support`

```bash
npx n8n-node cloud-support           # show current status (read-only, safe anytime)
npx n8n-node cloud-support enable    # restore default eslint.config.mjs + set n8n.strict=true
npx n8n-node cloud-support disable   # interactive confirm; switches to configWithoutCloudSupport + strict=false
```

Eligibility = BOTH of:

1. package.json → `n8n.strict === true`
2. `eslint.config.mjs` is exactly the default (`import { config } from '@n8n/node-cli/eslint'; export default config;`, whitespace-insensitive)

Output to parse:

```
✅ Cloud support is ENABLED
  • Strict mode: enabled
  • ESLint config: using default config
  • Status: eligible for n8n Cloud verification (if lint passes)
```

or `⚠️  Cloud support is DISABLED` with per-item bullets and `Status: NOT eligible`.

Interpretation rules:

- ENABLED ≠ verified. It only means the strict lint config (including the two cloud-only rules) is active — the node is "eligible (if lint passes)".
- If DISABLED and the user wants a verifiable node: run `cloud-support enable`, then re-run Gate 1 — the stricter config may surface new errors (remove third-party imports and restricted globals).
- Only run `cloud-support disable` when the user explicitly accepts losing n8n Cloud verification (e.g. a private node that genuinely needs an extra library). It prompts for confirmation, so it's unsuitable for non-interactive runs — prefer leaving it enabled.
- `enable` is also the documented repair for the `Strict mode violation: eslint.config.mjs has been modified` lint failure.

## Gate 4: Runtime Smoke Test

### Option A — CLI dev mode

```bash
npm run dev                                          # = n8n-node dev
npx n8n-node dev --external-n8n                      # only watch+link; run n8n yourself (set N8N_DEV_RELOAD=true there)
npx n8n-node dev --custom-user-folder /tmp/n8n-test  # isolate n8n state (default: ~/.n8n-node-cli)
```

What it does: copies static assets, **symlinks the project into `<user-folder>/.n8n/custom/node_modules/<package-name>`** (default `~/.n8n-node-cli/.n8n/custom/node_modules/...`), then runs two panels: `tsc --watch --pretty` and `npx -y n8n@latest` with `N8N_DEV_RELOAD=true` and `N8N_USER_FOLDER` pointing at the user folder. Ready signal: the n8n panel prints `Editor is now accessible` — the editor is at http://localhost:5678 (press `o` to open, `q` to quit). The first run installs n8n and can take minutes.

Agent automation notes: `n8n-node dev` takes over the terminal with a TUI and never exits on its own. When automating, launch it as a background process, poll output for `Editor is now accessible`, probe `curl -fsS http://localhost:5678 >/dev/null`, then kill the process group. An invalid package name aborts immediately with a validation error — itself a useful check.

Runtime pass criteria:

1. n8n starts with no "Problem loading node" / package-load errors mentioning your package in the n8n panel output.
2. Searching the nodes panel by the node's **displayName** (NOT the npm package name) finds the node, with icon rendered.
3. Credentials can be created and the credential test passes against the real API (it runs automatically on save) — this validates the `test`/`authenticate` blocks end-to-end.
4. Each resource/operation executes against the real API; Get Many honours returnAll/limit; Delete returns `{ deleted: true }`.
5. After editing description properties, restart n8n — description changes are not hot-reloaded into an open editor; refresh the browser after rebuild.

### Option B — manual linking into an existing n8n

```bash
# in the node project
npm run build && npm link
# in n8n's custom dir (create if absent)
mkdir -p ~/.n8n/custom && cd ~/.n8n/custom && npm init -y
npm link <package-name-from-package.json>
n8n start            # then open http://localhost:5678
```

If n8n uses `N8N_CUSTOM_EXTENSIONS`, link inside that folder instead of `~/.n8n/custom`.

For webhook triggers needing a public URL: `n8n start --tunnel` was **removed in n8n 2.0** — expose port 5678 with ngrok, localtunnel, or Cloudflare Tunnel instead.

### Runtime troubleshooting map

| Symptom | Cause / fix |
|---------|-------------|
| `Credentials of type "x" aren't known` | Node's `credentials[].name` ≠ credential class `name` property |
| Node missing from panel | Not registered under `n8n.nodes`, wrong dist path, or stale build — rebuild + restart |
| Icon missing/clipped | Icon not next to the node file, missing `file:` prefix/extension, or non-square SVG canvas |
| Changes not appearing | Stop n8n, `npm run build`, re-link if using npm link, restart |

## Gate 5: Release Readiness

`npm run release` (= `n8n-node release`) behaves differently by environment:

**Local (default):** runs release-it with built-in prechecks that must ALL pass:

- `--hooks.before:init="npm run lint && npm run build"` → lint and build are hard release gates
- `--git.requireBranch main` → must be on main
- `--git.requireCleanWorkingDir` → no uncommitted changes
- `--git.requireUpstream` → branch has an upstream remote
- `--git.requireCommits` → there are new commits to release

It then bumps the version, generates a changelog (auto-changelog), commits, tags, pushes, and creates a GitHub release. **It does NOT publish to npm locally** (`--npm.publish=false`). Pushing the `*.*.*` tag triggers `.github/workflows/publish.yml`, which publishes WITH npm provenance.

**In GitHub Actions** (auto-detected via `GITHUB_ACTIONS`): runs `npm run lint`, `npm run build`, then `npm publish` with `NPM_CONFIG_PROVENANCE=true`. The workflow needs `permissions: { id-token: write, contents: read }` and either npm Trusted Publishing or an `NPM_TOKEN` secret; requires `@n8n/node-cli` >= 0.23.0.

**Provenance is mandatory:** since May 1, 2026, n8n requires all community nodes to be published via GitHub Actions with npm provenance; locally-published packages cannot become verified. `n8n-node release --publish` exists but prints a warning and disqualifies the package from verification.

Agent pre-release checklist:

```bash
test -f .github/workflows/publish.yml || npx n8n-node release --init-workflow   # scaffold the publish workflow
git status --porcelain           # must print nothing
git rev-parse --abbrev-ref HEAD  # must print main
git rev-parse --abbrev-ref @{u}  # upstream must exist
npm run lint && npm run build    # the same gates release-it will enforce
```

Why plain `npm publish` fails: the scaffold sets `"prepublishOnly": "n8n-node prerelease"`, which exits 1 with `Run npm run release to publish the package` unless `RELEASE_MODE` is set. This is intentional — do not remove it.

## Gate 6: Verification Scan — `@n8n/scan-community-package`

This is the exact automated check n8n applies to submitted nodes — the verification guidelines define passing it as the bar.

```bash
npx @n8n/scan-community-package n8n-nodes-mypackage          # latest version
npx @n8n/scan-community-package n8n-nodes-mypackage@1.2.3    # specific version
```

It runs against the **published** npm package (so run it after publishing), in order:

1. **Provenance check** against the npm registry: the version must carry an npm provenance attestation with predicate type `https://slsa.dev/provenance/v1`. Fails with `Package was not published with npm provenance` if published from a local machine — only the GitHub Actions release flow fixes this.
2. **Download** the tarball (`npm pack`) and extract it.
3. **Static analysis**: ESLint with the `@n8n/eslint-plugin-community-nodes` `recommended` config (the full cloud ruleset, including `no-restricted-imports`/`no-restricted-globals`) plus `no-console: error`, over every `.js` and `.json` file in the published package — including package.json, so all the package-level rules fire there too. **Inline `eslint-disable` comments are ignored** (`allowInlineConfig: false`) — you cannot suppress rules to pass the scan.

Output to parse:

```
✅ Provenance check passed for n8n-nodes-x@1.0.0
✅ Package n8n-nodes-x@1.0.0 has passed all security checks
```

or `❌ Package ... has failed security checks` with `Reason:` (provenance vs ESLint violations) and `Details:` (file:line + rule names).

Interpretation: the scan uses the same `recommended` ruleset as `n8n-node lint` with cloud support enabled — **a project that passes Gate 1 with cloud support ENABLED and was published via the provenance workflow will pass the scan**. If the scan fails on rules that pass locally, the published dist is stale or cloud support was disabled at publish time: rebuild, re-lint with `cloud-support enable`, and release a new version (npm versions are immutable).

## Quick Reference: Full Command Sequence

```bash
# After implementing, in order:
npm run lint:fix && npm run lint                 # Gate 1 — exit 0, zero errors
npm run build                                    # Gate 2 — '✓ Build successful' + dist paths exist
npx n8n-node cloud-support                       # Gate 3 — 'Cloud support is ENABLED'
npm run dev                                      # Gate 4 — 'Editor is now accessible'; smoke test in editor
# When publishing:
npm run release                                  # Gate 5 — tags & pushes; GitHub Actions publishes with provenance
npx @n8n/scan-community-package <package-name>   # Gate 6 — '✅ passed all security checks'
```

### Failure triage

| Failure signature | Gate | Fix |
|---|---|---|
| `Strict mode violation: eslint.config.mjs has been modified` | 1 | `npx n8n-node cloud-support enable`; never hand-edit the config |
| `@n8n/community-nodes/<rule>` error | 1 | Fix per the rule catalog above; see also `common-mistakes.md` |
| `⚠️ n8n Cloud compatibility issues detected` | 1 | Remove restricted imports/globals — do NOT `cloud-support disable` without explicit user opt-out |
| `TypeScript build failed` | 2 | Fix type errors; `noUnusedLocals` makes unused imports hard errors |
| Missing dist paths | 2 | Align `n8n.nodes`/`n8n.credentials` entries with actual file locations |
| `Cloud support is DISABLED` | 3 | `cloud-support enable`, then re-run Gate 1 |
| Node not in panel / load errors | 4 | Runtime troubleshooting map above |
| `Run npm run release to publish the package` | 5 | Use `npm run release` — don't remove `prepublishOnly` |
| `Package was not published with npm provenance` | 6 | Publish via the GitHub Actions workflow (`release --init-workflow`) |
| Scan ESLint violations that pass locally | 6 | Stale dist or cloud support was off at publish; rebuild + release new version |
