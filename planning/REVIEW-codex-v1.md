# Review of `planning/PLAN.md`

## Findings

1. High: Price coverage breaks if a user removes a ticker that they still hold.
The plan says the real-data poller watches "the union of all watched tickers" ([PLAN.md](./PLAN.md):163), trades auto-add tickers to the watchlist on buy ([PLAN.md](./PLAN.md):262), and the watchlist endpoint also allows unconditional removal ([PLAN.md](./PLAN.md):270). At the same time, portfolio value, unrealized P&L, snapshots, and chat context all depend on current prices for held positions ([PLAN.md](./PLAN.md):230, [PLAN.md](./PLAN.md):261, [PLAN.md](./PLAN.md):294). If a held ticker is removed from the watchlist, the plan no longer guarantees a live price for that position. The contract should explicitly say one of these:
- price subscriptions are based on `watchlist union held positions`
- held tickers cannot be removed from the watchlist
- portfolio pricing uses a separate source from the watchlist feed

2. Medium: Persistence strategy is internally inconsistent.
The directory structure says `db/` in the repo is the runtime mount point and that `db/finally.db` is created there ([PLAN.md](./PLAN.md):101, [PLAN.md](./PLAN.md):115). Later, the Docker section shows a named volume mount, `finally-data:/app/db` ([PLAN.md](./PLAN.md):411), which does not use the repo's `db/` directory. Those are two different persistence models with different operational behavior. The plan should pick one and keep it consistent across scripts, Docker examples, and local-development expectations.

3. Medium: Trade execution needs an explicit atomicity/transaction rule.
A single trade can affect `users_profile`, `positions`, `trades`, `portfolio_snapshots`, and possibly `watchlist` ([PLAN.md](./PLAN.md):200, [PLAN.md](./PLAN.md):212, [PLAN.md](./PLAN.md):221, [PLAN.md](./PLAN.md):230, [PLAN.md](./PLAN.md):262). Chat-driven actions can trigger the same write path automatically ([PLAN.md](./PLAN.md):299, [PLAN.md](./PLAN.md):300). The plan never states that these writes happen in one SQLite transaction. Without that requirement, partial failures can leave cash, positions, and audit history out of sync. This should be a hard requirement in the backend section, not an implementation detail left implicit.

4. Medium: The SSE design conflicts with the stated deployment targets.
The plan explicitly says "no heartbeat if nothing changed" ([PLAN.md](./PLAN.md):180), then later says the container is intended for AWS App Runner, Render, or similar platforms ([PLAN.md](./PLAN.md):434). In practice, idle intermediaries often terminate long-lived SSE connections without periodic traffic. If the design keeps SSE, the plan should define a small keepalive comment/event cadence and the expected reconnect behavior. Otherwise the connection-status indicator will be unreliable outside local Docker.

5. Medium: The mock chat payload does not actually test the watchlist-add path it claims to exercise.
The default seed watchlist already contains `NVDA` ([PLAN.md](./PLAN.md):247), but the deterministic mock response uses `{"ticker": "NVDA", "action": "add"}` ([PLAN.md](./PLAN.md):349, [PLAN.md](./PLAN.md):355). On a fresh database that operation is a duplicate, not a real add. That weakens both development testing and E2E coverage, because the "full action pipeline" is not actually being exercised. The mock should use a ticker outside the default seed set.

6. Low: The first-launch experience overpromises what a raw Docker command can do.
The UX section says a single Docker command opens a browser automatically ([PLAN.md](./PLAN.md):15), but the scripts section only promises that behavior for wrapper scripts and even there it is optional ([PLAN.md](./PLAN.md):418, [PLAN.md](./PLAN.md):422). `docker run` itself will not open the host browser. This should be reworded so the UX promise matches the actual launch mechanisms.

## Open Questions

- Is the watchlist meant to be only a UI preference, or is it intended to be the authoritative subscription list for market data?
- Do you want local persistence to be inspectable in `./db/finally.db`, or do you want Docker-managed named-volume persistence? The plan currently implies both.
- Is cloud deployment a real target for the capstone, or only a stretch goal? That answer changes whether SSE keepalives are mandatory.

## Summary

The plan is strong on scope and product shape, but it still needs a tighter backend contract. The main issue is that pricing, watchlist behavior, and portfolio valuation are not fully aligned yet. Fixing those rules in the plan now will prevent avoidable churn across backend, frontend, and test implementation.
