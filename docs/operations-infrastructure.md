# Operations Infrastructure

Operational tooling for environment configuration, state snapshots, canary
rollouts, and centralized logging. All of it is stdlib-only Python under
`scripts/ops/`, tested by the `ops-tooling` CI job.

Run every command below from the repository root.

## Environment configuration (#1726)

`environments.toml` is the single source for per-environment settings. Each
environment defines the same schema:

| Section | Keys |
|---|---|
| top level | `rpc_url`, `network_passphrase` |
| `[env.limits]` | `max_fee_stroops`, `max_batch_size` |
| `[env.timeouts]` | `rpc_timeout_secs`, `tx_timeout_secs` |
| `[env.rates]` | `max_requests_per_minute`, `max_submissions_per_hour` |
| `[env.logging]` | `level`, `retention_days` |
| `[env.canary]` | `stages`, `max_error_rate`, `max_p95_latency_ms`, `min_samples`, `observe_secs` |

Environments: `dev`, `staging`, `testnet`, `mainnet`, `futurenet`, `standalone`.

Validation rejects unknown keys (to catch typos), enforces `https://` RPC
URLs except for localhost in `dev`/`standalone`, and requires
`max_submissions_per_hour > 0` outside dev (0 disables rate limiting, see the
[deployment runbook](deployment-runbook.md#45-post-deploy-configuration-max_submissions_per_hour)).
Mainnet additionally requires at least 90 days of log retention and a canary
error budget of at most 5%.

```bash
# Validate every environment (what CI runs)
python3 -m scripts.ops.config validate --env all

# Print the resolved config for one environment (default: $MAINSTAY_ENV or testnet)
python3 -m scripts.ops.config show --env mainnet

# Preview promotable differences before promoting staging to mainnet
python3 -m scripts.ops.config diff --from staging --to mainnet
```

Settings:

- `MAINSTAY_ENV` selects the environment when `--env` is omitted.
- `MAINSTAY_ENVIRONMENTS_FILE` loads an external config file instead of the
  repository copy, so secrets-free overrides can live outside git.

### Audit trail

Pass `--audit-file <path>` to `show` or `diff` to append a JSONL record:

```json
{"action": "diff:staging->mainnet", "actor": "alice", "environment": "mainnet", "sha256": "…", "ts": "2026-10-16T12:00:00+00:00"}
```

The `sha256` is a digest of the environment's canonical JSON, so an audit entry
can be matched to the exact configuration that was in effect. Identity fields
(`rpc_url`, `network_passphrase`) are excluded from promotion diffs.
