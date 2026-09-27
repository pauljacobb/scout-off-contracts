# FeeConfig Activation and History

## Overview

Every `FeeConfig` change in the `scout_access` contract is routed through the
internal `apply_fee_config` helper, which:

1. Replaces the active `FeeConfig` in instance storage.
2. Appends a `FeeConfigHistoryEntry` to a capped ring-buffer (max 5 entries,
   controlled by `FEE_CONFIG_HISTORY_CAP`).
3. Emits a `fee_config_updated` event with the old and new configs.

This ensures `get_fee_config_history` captures every real fee change regardless
of which code path triggered it, giving auditors and off-chain indexers a
consistent view of fee configuration evolution.

## Activation Sources

Each history entry records a `FeeConfigSource` variant that identifies which
code path applied the change:

| Source variant | Function | Description |
|----------------|----------|-------------|
| `Immediate` | `update_fee_config` | Direct admin update, takes effect immediately |
| `Decrease` | `propose_fee_config` (decrease branch) | Instant decrease — bypasses the proposal delay |
| `Proposal` | `activate_fee_config` | Time-locked proposal activated after delay |
| `Seed` | `admin_seed_fee_config` | One-time migration / seed helper |

## History Ring-Buffer

- **Storage**: Instance storage under `DataKey::FeeConfigHistory`.
- **Cap**: 5 entries (`FEE_CONFIG_HISTORY_CAP`). When the cap is exceeded, the
  oldest entry is evicted.
- **Order**: Entries are stored in chronological order — index 0 is the oldest
  retained entry, the last index is the most recent.
- **Query**: Call `get_fee_config_history()` to retrieve the full ring-buffer.

## Querying History

### Via the Stellar CLI

```bash
stellar contract invoke \
  --id $SCOUT_ACCESS_CONTRACT_ID \
  --network testnet \
  -- get_fee_config_history
```

### Via TypeScript bindings

```typescript
import { Client as ScoutAccessClient } from "@scoutchain/bindings-scout_access";

const client = new ScoutAccessClient({ contractId: SCOUT_ACCESS_CONTRACT_ID, ... });
const history = await client.get_fee_config_history();
history.forEach((entry, i) => {
  console.log(`[${i}] source=${entry.source} activated_at=${entry.activated_at}`);
  console.log(`      contact_fee_stroops=${entry.config.contact_fee_stroops}`);
});
```

## Database Indexing

The `fee_config_history` table in the off-chain PostgreSQL schema
(`migrations/001_initial_schema.sql`) stores an audit trail of fee config
changes sourced from `fee_config_updated` events emitted by `apply_fee_config`.
The `source` column mirrors the `FeeConfigSource` variant.

## Notes

- The ring-buffer cap (5) is intentionally small because `FeeConfig` entries
  are held in instance storage, which is shared across the whole contract
  instance. Instance storage has a size budget; storing large history would
  increase ledger fees for every contract invocation.
- For long-term history, rely on the off-chain indexer's `fee_config_history`
  table, which is unbounded and populated from `fee_config_updated` events.
