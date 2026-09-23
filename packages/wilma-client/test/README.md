# wilma-client tests

## Offline tests
The `parser-*.mjs`, `client-*.mjs` and `network-error.mjs` scripts need no secrets. They use inline fixtures and local HTTP servers. Add new ones to the `test` script in `package.json`.

```bash
pnpm --filter @wilm-ai/wilma-client test
```

## Live test (opt-in)
1. Create `test/.env.local` with `WILMA_BASE_URL`, `WILMA_USERNAME`, `WILMA_PASSWORD` and optionally `WILMA_STUDENT_ID`.
2. Build the client:
   ```bash
   pnpm --filter @wilm-ai/wilma-client build
   ```
3. Run:
   ```bash
   node test/live.spec.mjs
   ```

You can also point to a different env file with `WILMA_ENV_PATH=/path/to/.env`.

**Note:** `.env.local` should never be committed.
