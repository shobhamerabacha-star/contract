# 📦 Storage Layout & Optimization Documentation

Soroban uses state archival to prevent state bloat, where all data has a Time To Live (TTL) that must be periodically extended. The Rentar Contract is designed to minimize fees and optimize gas by choosing appropriate storage types and automatically extending TTLs.

---

## 🔑 Data Key Schema (`DataKey` Enum)

| Key Name | Storage Type | Rust Definition | Purpose |
| :--- | :--- | :--- | :--- |
| `AdminConfig` | **Instance** | `DataKey::AdminConfig` | Stores system parameters including admin address, pause state, token address, and transaction fees. |
| `Vault` | **Persistent** | `DataKey::Vault(Address)` | Tracks the tenant's individual vault state (balance, goals, lock thresholds, metadata). |
| `Escrow` | **Persistent** | `DataKey::Escrow(Address, u64)` | Tracks the status of secure landlord escrows indexed by landlord address and escrow counter ID. |
| `EscrowIdCounter` | **Instance** | `DataKey::EscrowIdCounter` | Tracks the next unique ID used to index landlord escrows. |

---

## ⚡ Storage Optimization Strategy

### 1. Instance Storage for Small Global Configs
Instance storage is bundled with the contract instance. We store `AdminConfig` and `EscrowIdCounter` here. This is extremely gas-efficient because these parameters are loaded and processed on almost every transaction, avoiding multiple separate storage reads.

### 2. Persistent Storage for Large User Entities
`SavingsVault` and `LandlordEscrow` are placed in persistent storage because they vary in size, grow linearly with user count, and are unique to user accounts. Placing them in instance storage would make the instance size grow prohibitively large.

### 3. Dynamic TTL Extensions & Rent-Cycle Lifecycle Safeguards
To prevent storage archival and ensure user funds remain un-archived across real-world monthly rental cycles:
- Previous defaults (10,000 ledgers threshold / 100,000 ledgers extension) provided only ~16 hours minimum threshold and ~6.9 days maximum extension at ~6 seconds per Stellar ledger. For a monthly rent product where users interact once per 30 days, this introduced high risk of state eviction.
- **Updated Production Constants**:
  - `VAULT_TTL_THRESHOLD`: `50_000` ledgers (~3.5 days minimum)
  - `VAULT_TTL_EXTEND`: `500_000` ledgers (~35–40 days maximum extension)
  - `ESCROW_TTL_THRESHOLD`: `50_000` ledgers
  - `ESCROW_TTL_EXTEND`: `500_000` ledgers
- When a user vault is created, updated, or deposited into:
  ```rust
  env.storage().persistent().extend_ttl(
      &DataKey::Vault(tenant),
      VAULT_TTL_THRESHOLD,
      VAULT_TTL_EXTEND,
  );
  ```
- Creating a landlord escrow executes an identical TTL extension for `DataKey::Escrow(landlord, escrow_id)`.
- **On-Chain & Off-Chain Bump Instructions**:
  - For long-lived vaults or escrows waiting for future claim dates without deposits, the contract exposes `bump_vault_ttl(env, tenant)` and `bump_escrow_ttl(env, landlord, escrow_id)` callable permissionlessly or via automated off-chain cron heartbeats to bump TTL before expiration.
