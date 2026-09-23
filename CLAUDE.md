# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

wilmai is an unofficial CLI (`wilma` / `wilmai` binaries) for reading Finland's Wilma school system: schedules, homework, exams, grades, attendance, messages, news. It serves two audiences: parents running it interactively, and AI agents calling it non-interactively with `--json`. The JSON output is a contract that agent skills depend on.

pnpm monorepo (`pnpm@9`, workspaces `packages/*` and `apps/*`):

- `packages/wilma-client` (`@wilm-ai/wilma-client`): TypeScript library for auth, HTTP session, HTML/JSON parsers and the tenant list. Published to npm.
- `packages/wilma-cli` (`@wilm-ai/wilma-cli`): interactive and command-mode CLI built on the client. Published to npm.
- `apps/site` (`@wilm-ai/site`): Next.js 14 landing page for wilm.ai, deployed on Vercel (`vercel.json` at the root).
- `skills/`: agent skills (`wilma`, `wilma-triage`) indexed by `skills.json` and installed via `npx skills add aikarjal/wilmai`.

## Commands

```bash
pnpm install
pnpm build                                   # client first, then tsc for all packages + next build
pnpm test                                    # offline tests for all packages
pnpm --filter @wilm-ai/wilma-client test     # builds, then runs every client test script
pnpm --filter @wilm-ai/wilma-cli test        # builds client + CLI, then runs CLI e2e test
node packages/wilma-cli/dist/index.js        # run the local CLI build
pnpm --filter @wilm-ai/site dev              # landing page dev server
```

Tests are plain Node scripts using `node:assert/strict`. There is no test runner. They import from `../dist/`, so **build first**, then run one file directly:

```bash
pnpm --filter @wilm-ai/wilma-client build && node packages/wilma-client/test/parser-news.mjs
```

A new client test file must also be appended to the `test` script chain in `packages/wilma-client/package.json`, or `pnpm test` will not run it.

The CLI e2e test (`packages/wilma-cli/test/news-resources.mjs`) starts local HTTP servers that mimic Wilma, writes a temp config, and runs the built CLI with `execFile`. Follow that pattern for new CLI tests.

Live tests (`test:live`, `test:smoke`) need real Wilma credentials, either `packages/wilma-client/test/.env.local` or an existing CLI config. Do not run them unless asked. The `lint` scripts in both packages are placeholders. Only `apps/site` has real linting (`next lint`).

## Architecture

**Session layer (`wilma-client/src/session.ts`).** `WilmaSession` holds a `tough-cookie` jar and uses `undici` fetch with a browser user agent. It does the Wilma form login (SESSIONID token, then POST `/login`) and handles MFA by throwing `MfaRequiredError(formkey)`. The caller supplies an OTP via `submitMfaCode`. Guardian accounts scope requests per child by prefixing paths with `/!<studentNumber>` (`getPrefixedPath`). A client is therefore bound to one student. Multi-student operations create one client per student.

**Client (`wilma-client/src/client.ts`).** `WilmaClient.login(profile, onMfa)` returns namespaced APIs (`messages`, `news`, `exams`, `schedule`, `overview`, …). Each one fetches a Wilma page or endpoint and hands the body to a parser in `src/parsers/`. Wilma returns JSON or HTML depending on version and endpoint, so many calls check the content type and fall back between `parse*Json` and `parse*Html` (cheerio). Student discovery (`listStudents`) tries `/api/v1/accounts/me/roles` first and falls back to scraping the homepage.

**Network errors (`network-error.ts`).** Every fetch call is wrapped so that transport failures become `NetworkError` with `code` and `hint`, instead of a bare `fetch failed`. Error messages may contain only the request **origin**, never the path, because paths embed student numbers.

**News resources.** `NewsItem.resources` carries an `authContext`. `"wilma"` resources download through the authenticated session. `"external"` resources use an isolated fetch with a fresh cookie jar and no Wilma credentials. This separation is a security property, so keep it. The CLI never guesses whether a URL is a file. It attempts the download and reports `downloaded` or `not_a_file`.

**Tenants.** `tenant_list.json` is a bundled snapshot of all Wilma instances. It ships in the client package's `files` and is loaded relative to `import.meta.url`.

**CLI (`wilma-cli/src/index.ts`).** A single large file:
- `main()` dispatches to `runInteractive()` (`@inquirer/prompts` menus) when there are no args, otherwise to `parseArgs()` → `handleCommand()`.
- Each command has an `output*` function (single student) and an `outputAll*` variant for `--all-students`. `--json` switches every output path to structured JSON.
- Non-interactive MFA uses a stored TOTP secret (`generateTOTP`).

**Config (`wilma-cli/src/config.ts`).** Profiles live in `~/.config/wilmai/config.json` (`$XDG_CONFIG_HOME` and `WILMAI_CONFIG_PATH` are respected). Secrets are base64-obfuscated with a salt, not encrypted. `loadConfig` migrates old single-student profile fields, so keep it backward-compatible.

## Releases and versioning

Versions are bumped by hand. `@changesets/cli` is installed but not used.
- The CLI depends on the client with a real semver range (`"^1.5.2"`), never `workspace:^`. Leaking `workspace:^` into the published package broke global installs in 1.5.3. `linkWorkspacePackages` resolves it locally. `pnpm -r` still doesn't build the client before the CLI, so the root `build` script builds the client first explicitly. Keep that step.
- `skills/wilma/SKILL.md` `version` tracks the CLI version. `wilma-triage` has its own version.
- `CHANGELOG.md` entries are headed by the CLI version and list the matching client version (`_Releases: wilma-cli X, wilma-client Y._`).
- When CLI commands, flags or JSON shapes change, update `skills/wilma/SKILL.md`, and `README.md` if user-facing.

## Site

The Next.js app uses `[lang]` routes (`fi`/`en`). `middleware.ts` redirects `/` based on the `lang` cookie or `Accept-Language`. Strings live in `lib/i18n.tsx`. The www→apex redirect is in `next.config.mjs`, because `vercel.json` redirects are ignored for Next builds.
