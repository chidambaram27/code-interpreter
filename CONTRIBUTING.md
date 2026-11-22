# Contributing

Thanks for your interest in Code Interpreter!

## How this repository is maintained

This repository is published from an internal ClickHouse monorepo, which is
the source of truth. Internal changes that are not already public are mirrored
here as a snapshot commit on the `sync/main` branch (spot them by the
`Source: ClickHouse/ai@<sha>` trailer); a maintainer merges the resulting sync <!-- leak-check:allow -->
pull request to release it to `main`.

Practical consequences:

- **Pull requests are welcome.** CI runs on every PR. After a maintainer merges
  your change here, an automated bridge opens a matching internal pull request
  from the exact landed range. The internal review and CI gates still apply,
  and attribution is preserved with a `Co-authored-by:` trailer.
- **`main` accepts no direct pushes.** It advances through reviewed community
  pull requests and automated sync pull requests; branch rules enforce this
  with no exceptions.
- **History is snapshot-based.** Commits here intentionally do not mirror the
  internal commit history.

## Releases

Tagged releases are cut from `main` as `vMAJOR.MINOR.PATCH` (with `-rcN` for
release candidates), and each one carries the packaged Helm chart. Repository,
API, service, and chart versions advance independently; component version bumps
land on `main` through the pull request flow above before they are included in a
release. Successful `main` CI automatically releases deployable changes while
documentation, workflow, and test-only changes are skipped. See
[docs/RELEASING.md](docs/RELEASING.md) for the full process and manual path.

## Development

See the [README](README.md) for the architecture overview. Component-level
docs live in `api/`, `service/`, `packages/code/`, and `helm/codeapi/`.

### Set up and run

Run the full stack with Docker Compose:

```bash
cp .env.example .env            # then set CODEAPI_BRIDGE_TOKEN
openssl rand -hex 32            # generate a value of at least 32 bytes
docker compose up --build
```

- The default path runs the sandbox in a libkrun microVM and needs `/dev/kvm`.
  Set `KVM_ENABLED=false` for direct NsJail development without KVM.
- `docker-compose.local-dev.yml` (combined API + workers, no auth),
  `docker-compose.scalable.yml`, and `docker-compose.mac.yml` are alternative
  stacks; pass one with `docker compose -f <file> up`.
- Health checks: API `GET /v1/health`; worker and file server `GET /health`
  and `GET /ready`; tool call server `GET /health`.

To work on a component without Docker, use [Bun](https://bun.sh) (CI pins
1.4.2) for `api/` and `service/`, and Node >= 22.21 with npm for
`packages/code/`:

```bash
# api/ (sandbox runner)
cd api && bun ci
bun run dev                     # watch mode (needs nsjail to execute code)

# service/ (API, workers, file server, tool call server)
cd service && bun ci
# see the start:* scripts in service/package.json

# packages/code/ (@librechat/code)
cd packages/code && npm ci
```

### Test

Run the suite for each component you changed; CI (`.github/workflows/ci.yml`)
runs all of them on every PR.

| Component | Commands |
|---|---|
| `api/` | `bun ci && bun run build && bun run test` |
| `service/` | `bun ci && bun run build && bun run test` |
| `packages/code/` | `npm ci && npm test` (needs `ripgrep` installed) |
| `launcher/` | `cargo test` (needs Rust and libkrun headers) |
| Deployment config | scripts in `tests/`, e.g. `tests/kvm_guest_dns.sh`, `tests/block_root_package_delivery.sh`, `node tests/compose-bridge-config.cjs` |

Some CI jobs (NsJail IPC filter, guest API image tests) build Docker images and
run the scripts in `tests/` inside them; see the workflow for the exact
commands. Other files in `tests/` are exploit regressions: the sandbox must
resist each one (see `tests/readme.md`).

### Style

Formatting is Prettier (`.prettierrc.yaml`: single quotes, 4-space tabs,
`arrowParens: avoid`). The `pre-commit` script runs `prettier --write` on
staged files.

## Reporting issues

Open a GitHub issue with reproduction steps. For suspected security issues,
please do not open a public issue — contact the maintainers instead.
