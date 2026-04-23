# rails-encrypted-credentials-cli

A non-interactive CLI for CRUDing Rails encrypted credentials. Works with both the default `config/credentials.yml.enc` and per-environment `config/credentials/<env>.yml.enc` files.

Built so LLM agents (and humans) can read and modify credentials without shelling into `EDITOR=cat rails credentials:edit` and dealing with editor round-trips, partial writes, or accidentally splatting secrets into terminal transcripts.

## Why

`rails credentials:edit` is interactive by design — it decrypts to a temp file, opens `$EDITOR`, and re-encrypts on save. That's painful for agents because:

- They have to fake an editor with `EDITOR=cat` and parse the YAML themselves
- It's easy to corrupt the file with whitespace or YAML quoting issues
- It can't do `set one.dotted.key value` in a single shot
- Agents who `dump` the full file also dump every secret they see into their transcript

This CLI gives you `get`, `set`, `unset`, `dump`, and `envs` subcommands with masked-by-default output, dotted paths, and a single-shot `--reveal` flag (audit-able) for when you actually need the raw value.

## Prerequisites

- Ruby 3.x (tested on 3.3)
- Bundler

## Setup

```bash
cd ~/tools/rails-encrypted-credentials-cli
bundle install
ln -s ~/tools/rails-encrypted-credentials-cli/rails-encrypted-credentials-cli ~/bin/rails-encrypted-credentials-cli
# (a wrapper at ~/bin uses `bundle exec` if needed)
```

## Usage

```bash
# Default credentials.yml.enc in current dir
rails-encrypted-credentials-cli get stripe.api_key
rails-encrypted-credentials-cli set stripe.api_key sk_test_xyz
rails-encrypted-credentials-cli unset stripe.deprecated_key
rails-encrypted-credentials-cli dump
rails-encrypted-credentials-cli envs

# Targeting a different project + per-env file
rails-encrypted-credentials-cli --project ~/projects/myapp --env production get aws.secret_access_key --reveal
rails-encrypted-credentials-cli --project ~/projects/myapp --env development set sendgrid.api_key SG.xxx
```

### Output

JSON envelope on stdout. Add `--pretty` for pretty-printed JSON.

```json
{"ok":true,"data":{"project":"/Users/cgenco/projects/myapp","env":"production","path":"stripe.api_key","value":"sk_t••••••••3xY9 (len=51)","revealed":false}}
```

Errors go to stderr with exit code 1:

```json
{"ok":false,"error":"key not found: stripe.api_key","code":"KEY_PATH_NOT_FOUND"}
```

### Masking

By default, string values are masked:
- **Length ≥ 12**: `first 4 chars + •••••••• + last 4 chars + " (len=N)"`
- **Length < 12**: `"•••••• (len=N)"` (no preview — too short to safely show edges)

Hashes and arrays recurse normally; only leaf values are masked. Pass `--reveal` to get raw values.

### Key resolution

For the **default** credentials file (`config/credentials.yml.enc`):
1. `config/master.key`
2. `RAILS_MASTER_KEY` env var

For **per-env** files (`config/credentials/<env>.yml.enc`):
1. `config/credentials/<env>.key`
2. `RAILS_MASTER_KEY` env var

## How it works

Uses `ActiveSupport::EncryptedConfiguration` directly (the same class Rails uses internally). No Rails boot required — just `activesupport` as a gem dep. Decrypts → mutates the parsed YAML hash → re-serializes to YAML → re-encrypts. Comments are not preserved (Rails' own `credentials:edit` doesn't preserve them through the YAML round-trip either).

## Errors

| Code | Meaning |
|------|---------|
| `PROJECT_NOT_FOUND` | `--project` path doesn't exist |
| `CREDENTIALS_FILE_NOT_FOUND` | `.yml.enc` doesn't exist for the given env |
| `KEY_NOT_FOUND` | No `.key` file and `RAILS_MASTER_KEY` not set |
| `DECRYPT_FAILED` | Wrong key for this file |
| `KEY_PATH_NOT_FOUND` | The dotted path doesn't exist in the credentials |
| `BAD_USAGE` | Wrong number of args for the subcommand |
