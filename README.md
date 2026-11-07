# vesting-explorer

A public, read-only explorer for token vesting on Stellar. An indexer reads every schedule from a [soroban-vesting-platform](https://github.com/unajiogenyi05-hub/soroban-vesting-platform) deployment, and a dashboard shows what is locked, what has vested, who can claim what, and what unlocks next. No wallet and no account needed.

> **Status:** early development. This describes the v0.1 scope. Expect breaking changes until a tagged release. Unaudited.

---

## Why this exists

When a project locks tokens for its team and investors, the people who care most (contributors, investors, token holders) have no easy way to see the schedule. The vesting contract answers one schedule at a time: `get_schedule(id)`, `get_claimable(id)`, `get_beneficiary_schedules(address)`. That is enough to claim tokens, but not enough to answer simple questions:

- How many tokens are still locked for a given token, and how many are claimable today?
- Who unlocks what over the next three months?
- What has each beneficiary claimed, and when?
- Which schedules were revoked?

The platform's own interface is for administrators and beneficiaries acting on a single deployment. This explorer is for everyone else.

---

## What you can see

- **Overview:** total locked, vested, claimed and still-unvested amounts per token, and the number of active, completed and revoked schedules
- **Unlock calendar:** upcoming cliff ends, schedule ends and the amounts vesting each month, per token
- **Schedules:** a searchable, filterable table with a progress bar for each schedule
- **Schedule page:** the full schedule (start, cliff, duration), a vesting curve with "today" marked, claimed and claimable amounts, and every recorded event with a link to the transaction on the block explorer
- **Beneficiary page:** all schedules for one address, with totals
- **Data checks:** a notice on any schedule where the contract's own claimable figure differs from the exact calculation
- **Export:** download schedules and events as CSV
- **Embed:** a small widget you can place on another site to show one schedule or one token's unlocks

The dashboard is read-only in v0.1. Claiming and administration stay in the platform's own interface.

---

## How it works

```
Soroban RPC  ->  sync state (all schedules)  ->  PostgreSQL  ->  indexer API  ->  dashboard
             ->  poll events (new activity)  ->
```

1. **State sync.** On start and on every cycle, the indexer reads `schedule_count()` and fetches any schedule that is new or not yet finished. Active schedules are refreshed so claimed amounts stay right.
2. **Event follow.** It polls the contract's events and stores each one once, with its ledger time and transaction hash.
3. **Maths.** Vesting follows the contract's formula (nothing before `start_time + cliff_duration`, then linear from `start_time` over `total_duration`). The indexer uses exact big-integer maths, so very large amounts are not rounded or capped.
4. **Check.** For active schedules it compares the exact claimable amount with the contract's `get_claimable` and records a mismatch flag if they differ.
5. **Serve.** The indexer exposes a REST API, and the dashboard reads only from that API. The web app never talks to the contract, needs no keys, and does not depend on RPC for every page view.

The contract's events carry only part of a schedule (`created` has the ID, beneficiary and total amount, but not the token, start time, cliff or duration), and RPC nodes keep events for a limited time. That is why the indexer reads contract **state** for the full list of schedules and uses **events** only for history.

---

## Repository layout

| Folder | Purpose | Tech |
|---|---|---|
| `apps/indexer` | Reads the contract into PostgreSQL and serves the REST API | Node.js, TypeScript |
| `apps/web` | The dashboard | Next.js, Tailwind |
| `packages/api-client` | Typed client for the indexer API | TypeScript |
| `packages/vesting-math` | Vesting maths and amount formatting shared by the indexer and the pages | TypeScript |

---

## Indexer API

| Endpoint | Returns |
|---|---|
| `GET /health` | Service and sync status, last ledger indexed |
| `GET /contracts` | The deployments being followed |
| `GET /schedules` | Schedules, filterable by `status`, `token`, `beneficiary`; paginated |
| `GET /schedules/:contract/:id` | One schedule with vested, claimed, claimable and unvested amounts |
| `GET /schedules/:contract/:id/events` | The schedule's recorded events |
| `GET /beneficiaries/:address` | Every schedule for an address, across contracts, with totals |
| `GET /tokens/:token/summary` | Total locked, vested, claimed and unvested for a token |
| `GET /unlocks?from=&to=&token=` | Cliff ends, schedule ends and monthly vested amounts in a date range |
| `GET /export/schedules.csv` | All schedules as CSV |
| `GET /export/events.csv` | All recorded events as CSV |

All amounts are strings in the token's smallest unit, plus a formatted value using the token's decimals.

---

## Getting started

### Prerequisites

- Node.js 22 or newer
- Docker (for PostgreSQL)
- The contract ID of a deployed vesting contract (the [soroban-vesting-platform](https://github.com/unajiogenyi05-hub/soroban-vesting-platform) repo lists a testnet deployment)

### Setup

```bash
git clone https://github.com/unajiogenyi05-hub/vesting-explorer.git
cd vesting-explorer
npm ci
cp .env.example .env      # set DATABASE_URL and VESTING_CONTRACT_IDS
docker compose up -d
npm run db:migrate --workspace=apps/indexer
npm run dev               # indexer and dashboard together
```

The indexer API runs on port 4000 and the dashboard on port 3000.

**Setup notes**

- The indexer reads `.env` from `apps/indexer`, so link or copy the root `.env` there, and the web variables to `apps/web/.env.local`.
- In GitHub Codespaces, make port 4000 public and set `NEXT_PUBLIC_INDEXER_URL` and `CORS_ORIGIN` to the forwarded addresses with no trailing slash. Restart after changing `.env`.

### Environment variables

| Variable | Required | Description |
|---|---|---|
| `DATABASE_URL` | Yes | PostgreSQL connection string |
| `VESTING_CONTRACT_IDS` | Yes | Comma-separated vesting contract IDs to follow |
| `NEXT_PUBLIC_INDEXER_URL` | Yes | Base URL of the indexer API (no trailing slash) |
| `STELLAR_NETWORK` | No (default `testnet`) | `testnet` or `mainnet` |
| `SOROBAN_RPC_URL` | No | Soroban RPC endpoint |
| `POLL_INTERVAL_MS` | No (default `10000`) | How often the indexer looks for new activity |
| `PORT` | No (default `4000`) | Indexer API port |
| `CORS_ORIGIN` | No | Allowed web origin |
| `NEXT_PUBLIC_EXPLORER_URL` | No | Block explorer base URL for transaction and contract links |
| `NEXT_PUBLIC_SITE_NAME` | No | Name shown in the header |

### Scripts

| Command | What it does |
|---|---|
| `npm run dev` | Start the indexer and the dashboard |
| `npm run lint` | Lint every workspace |
| `npm run typecheck` | Typecheck every workspace |
| `npm test` | Run all tests |
| `npm run build` | Build everything |

---

## Embedding a schedule

```html
<iframe src="https://YOUR-SITE/embed/schedule/CONTRACT_ID/7" width="420" height="260" title="Vesting schedule"></iframe>
```

Embeds show one schedule or one token's next unlocks and run no scripts on the host page.

---

## Limits

- **History starts when you start.** RPC nodes keep events for a limited time. Schedules and their current claimed totals are always read from state, but the time of each individual claim is only known from the moment the indexer first ran, or as far back as the RPC still has events. Run the indexer continuously.
- **It follows the platform's contract.** A different vesting contract design needs its own adapter.
- **Read-only.** It never signs anything and holds no keys.
- **A schedule shown here is a public on-chain fact.** It does not say whether the people or the arrangement behind it are genuine.
- It is unaudited testnet software.

---

## Testing

Tests cover the vesting maths against an independent big-integer reference (including amounts near the `i128` limit and the boundaries at the cliff and the end), state sync with a mocked RPC, event de-duplication and out-of-order delivery, the mismatch flag, every API endpoint, CSV output, pagination, the unlock calendar buckets, each page's loading, empty and error states, filtering and search, and the embed pages.

---

## Roadmap

- [ ] State sync, event follow and the REST API
- [ ] Exact vesting maths with a mismatch check
- [ ] Overview, schedules and schedule pages
- [ ] Unlock calendar and beneficiary pages
- [ ] CSV export
- [ ] Embeddable widgets
- [ ] Claim button for beneficiaries (wallet-signed)
- [ ] Multiple contracts and tokens
- [ ] Webhooks for claims and revocations
- [ ] Docker image
- [ ] Mainnet guide

---

## Related

[soroban-vesting-platform](https://github.com/unajiogenyi05-hub/soroban-vesting-platform): the contracts this explorer follows.

---

## Contributing

Contributions are welcome. Good first areas: more API filters and export formats, chart and accessibility improvements, embed options, tests against testnet and documentation. Please open an issue before starting large changes. See [CONTRIBUTING.md](CONTRIBUTING.md).

---

## License

MIT
