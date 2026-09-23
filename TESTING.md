# Testing

wilmai uses two tiers of tests:

## 1) Offline tests (no credentials)
These are safe to run in CI and do not require Wilma access.

- Client parser and client tests live in `packages/wilma-client/test/*.mjs` (inline HTML/JSON fixtures, local HTTP servers).
- The CLI end-to-end test lives in `packages/wilma-cli/test/news-resources.mjs`.

Run them with:
```bash
pnpm test
```

## 2) Live tests (opt-in, require your own credentials)
These tests require access to a real Wilma account and should be run locally.
No secrets are committed to the repo.

### Wilma client live test
1. Create `packages/wilma-client/test/.env.local` with your credentials:
   ```bash
   WILMA_BASE_URL=https://yourcity.inschool.fi
   WILMA_USERNAME=you@example.com
   WILMA_PASSWORD=...
   # WILMA_STUDENT_ID=12345   # optional
   ```
2. Run:
   ```bash
   pnpm --filter @wilm-ai/wilma-client test:live
   ```

### Wilma CLI live test
1. Run the interactive CLI once to create `~/.config/wilmai/config.json` (or `$XDG_CONFIG_HOME/wilmai/config.json`).
2. Run it against that config (the test otherwise defaults to `.wilmai/config.json` at the repo root):
   ```bash
   WILMAI_CONFIG_PATH=~/.config/wilmai/config.json pnpm --filter @wilm-ai/wilma-cli test:live
   ```

### Notes
- Tests only assume you have **at least one student** in one tenant.
- For CLI tests, the `kids list --json` command must return at least one student.
