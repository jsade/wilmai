# wilma-cli live test

## Prereqs
- Run the interactive CLI once so `~/.config/wilmai/config.json` exists.
- Build the CLI.

## Run
By default the test reads `.wilmai/config.json` at the repo root, so point it at your real config:
```bash
pnpm --filter @wilm-ai/wilma-cli build
WILMAI_CONFIG_PATH=~/.config/wilmai/config.json node test/live.spec.mjs
```
