# rails-encrypted-credentials-cli

CRUD Rails encrypted credentials non-interactively. Works with the default `config/credentials.yml.enc` and per-env `config/credentials/<env>.yml.enc` files.

## Commands

```bash
rails-encrypted-credentials-cli envs                              # List credentials files in project
rails-encrypted-credentials-cli get <dotted.path>                 # Get a value (masked)
rails-encrypted-credentials-cli get <dotted.path> --reveal        # Get raw value
rails-encrypted-credentials-cli set <dotted.path> <value>         # Set a string value
rails-encrypted-credentials-cli unset <dotted.path>               # Remove a key
rails-encrypted-credentials-cli dump                              # Dump full tree (masked)
rails-encrypted-credentials-cli dump --reveal                     # Dump full tree (raw)
```

## Options

- `--project PATH` — Rails project directory (default: cwd)
- `--env ENV` — Per-env file (`development`, `production`, etc). Omit for default `config/credentials.yml.enc`.
- `--reveal` — Return raw values instead of masked previews
- `--pretty` — Pretty-print JSON output

## Examples

```bash
# Default credentials, current dir
rails-encrypted-credentials-cli get stripe.api_key
rails-encrypted-credentials-cli set stripe.api_key sk_test_xyz

# Per-env credentials in another project
rails-encrypted-credentials-cli --project ~/projects/myapp --env production dump
rails-encrypted-credentials-cli --project ~/projects/myapp --env production get aws.secret --reveal
```

## Output

Standard envelope: `{"ok": true, "data": {...}}` on stdout, `{"ok": false, "error": "...", "code": "..."}` on stderr (exit 1).

## Masking

Values are masked by default. Strings ≥12 chars: `"abcd••••••••wxyz (len=51)"`. Strings <12 chars: `"•••••• (len=4)"` (no preview at all). Use `--reveal` to get the raw value. Hashes/arrays are walked recursively — only leaves are masked.

## Key resolution

For each target file the tool checks (in order):
1. The matching key file (`config/master.key` or `config/credentials/<env>.key`)
2. The `RAILS_MASTER_KEY` env var

Errors with `KEY_NOT_FOUND` if neither is present. Errors with `CREDENTIALS_FILE_NOT_FOUND` if the `.yml.enc` doesn't exist (no auto-create — use `rails credentials:edit --environment X` for that).
