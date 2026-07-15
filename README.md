# IPI Swap

An experimental Cosmos swap interface used to study routing, pricing, wallet
connection, slippage presentation, and transaction review.

> **Status: inherited prototype.** It has no production safety claim, no
> canonical liquidity source, and no audit. Do not use it with assets of value.

## Development

The codebase uses an older Create Cosmos App / Next.js stack. Review and update
its dependencies before exposing a deployment.

```sh
yarn
yarn dev
yarn build
```

The local application is served at `http://localhost:3000`. Asset, routing, and
network defaults live under `config/` and must be verified against the intended
network and trusted data sources.

## Contributing safely

Production-quality work would require explicit chain and asset identity,
decimal and denomination tests, price-impact and slippage bounds, transaction
simulation, route provenance, adversarial UI tests, failure-safe approvals, and
an independent security assessment.

## Provenance and license

This repository derives from Hyperweb Create Cosmos App's `swap-tokens`
example. See [UPSTREAM.md](UPSTREAM.md) for provenance and migration context.
Upstream and IPI modifications are distributed under the [MIT License](LICENSE).
