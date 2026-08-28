# Runbook: Contract Upgrade Procedure

**Scope:** Upgrading the `vesting_cliff_drip_stream` Soroban smart contract on Stellar Mainnet.  
**Audience:** On-call engineers with admin key access.  
**Estimated time:** 60–90 minutes including testnet validation.  
**Last reviewed:** 2026-08-28

> ⚠️  **On-chain code is immutable once deployed.** An upgrade deploys a *new* contract WASM
> and updates the contract entry to point to it via the `upgrade` entry point. There is no
> on-chain rollback. Read the [Rollback Procedure](#7-rollback-procedure) section before
> starting.

---

## Table of Contents

1. [Pre-Upgrade Checklist](#1-pre-upgrade-checklist)
2. [Building and Verifying the New WASM Binary](#2-building-and-verifying-the-new-wasm-binary)
3. [Deploying New WASM to Testnet and Running Smoke Tests](#3-deploying-new-wasm-to-testnet-and-running-smoke-tests)
4. [Executing the Upgrade on Mainnet](#4-executing-the-upgrade-on-mainnet)
5. [Post-Upgrade Verification](#5-post-upgrade-verification)
6. [Incident Response During Upgrade](#6-incident-response-during-upgrade)
7. [Rollback Procedure](#7-rollback-procedure)

---

## 1. Pre-Upgrade Checklist

Complete every item before touching any keys or running build commands.

### 1.1 Communication

- [ ] Post upgrade notice in `#ops` Slack at least **24 hours before** the maintenance window.
  ```
  [NOTICE] Contract upgrade scheduled for <DATE TIME UTC>.
  Affected contract: VESTING_CONTRACT=<contract-id>
  Expected downtime: none (contract calls succeed throughout; only logic changes)
  IC: <your-name>
  ```
- [ ] Notify affected teams (frontend, indexer) of any interface changes.
- [ ] Confirm a second engineer is available on-call during the maintenance window.

### 1.2 State Backup

- [ ] Export all active `VestingSchedule` entries from PostgreSQL:
  ```bash
  psql "$DATABASE_URL" \
    -c "\COPY (SELECT * FROM indexed_events ORDER BY ledger) TO 'backup-$(date +%Y%m%d).csv' CSV HEADER"
  ```
- [ ] Note the current contract WASM hash:
  ```bash
  stellar contract info \
    --contract-id "$VESTING_CONTRACT" \
    --network mainnet \
    | grep wasm_hash
  ```
  Record the hash as `OLD_WASM_HASH=<value>` for later comparison.

### 1.3 Admin Key Custody

- [ ] Confirm the admin / upgrader key is accessible (hardware wallet, Vault, or secure env var).
- [ ] If using a hardware wallet, have it plugged in and unlocked before the window.
- [ ] Verify the upgrader account has sufficient XLM for transaction fees (minimum 1 XLM recommended).

### 1.4 Code Review

- [ ] The PR introducing the upgrade has been reviewed and approved by at least two engineers.
- [ ] `CHANGELOG.md` entry added for the new version.
- [ ] No breaking changes to the public ABI unless a migration plan is in place.

---

## 2. Building and Verifying the New WASM Binary

### 2.1 Clean build

```bash
# Ensure you are on the release commit
git checkout <release-tag-or-sha>

# Clean artefacts from any previous build
cargo clean

# Compile to WASM
cargo build --target wasm32-unknown-unknown --release
```

Expected output ends with:
```
Compiling vesting_cliff_drip_stream v<version>
 Finished release [optimized] target(s)
```

### 2.2 Run the full test suite

```bash
make test
```

All tests must pass. Do not proceed if any test fails.

### 2.3 Optimize the WASM binary

```bash
CONTRACT_NAME="vesting_cliff_drip_stream"
WASM="target/wasm32-unknown-unknown/release/${CONTRACT_NAME}.wasm"
OPTIMIZED="target/${CONTRACT_NAME}.optimized.wasm"

stellar contract optimize --wasm "$WASM" --wasm-out "$OPTIMIZED"
```

### 2.4 Compute and record the WASM hash

```bash
NEW_WASM_HASH=$(stellar contract upload \
  --wasm "$OPTIMIZED" \
  --source "$SOURCE_ACCOUNT" \
  --network testnet \
  --build-only \
  | grep -oP '(?<=hash: )[0-9a-f]+')

echo "NEW_WASM_HASH=$NEW_WASM_HASH"
```

> `stellar contract upload --build-only` prints the hash without submitting to the network.

### 2.5 Verify WASM size

```bash
wasm_size=$(stat -c%s "$OPTIMIZED")
echo "Optimized WASM size: ${wasm_size} bytes"
```

The Soroban network limit is **64 KB**. Fail if the binary exceeds this.

---

## 3. Deploying New WASM to Testnet and Running Smoke Tests

### 3.1 Upload WASM to testnet

```bash
stellar contract upload \
  --wasm "$OPTIMIZED" \
  --source "$SOURCE_ACCOUNT" \
  --network testnet
```

Record the output hash as `TESTNET_WASM_HASH`.

### 3.2 Deploy a fresh testnet instance

```bash
TESTNET_CONTRACT=$(stellar contract deploy \
  --wasm "$OPTIMIZED" \
  --source "$SOURCE_ACCOUNT" \
  --network testnet)

echo "TESTNET_CONTRACT=$TESTNET_CONTRACT"
```

### 3.3 Smoke test: create a stream

```bash
export VESTING_CONTRACT="$TESTNET_CONTRACT"

stellar contract invoke \
  --id "$VESTING_CONTRACT" \
  --source "$SOURCE_ACCOUNT" \
  --network testnet \
  -- create_vesting_stream \
  --sponsor "$SOURCE_ACCOUNT" \
  --recipient "$TEST_RECIPIENT" \
  --token "$TEST_TOKEN" \
  --rate 10 \
  --cliff_duration 10 \
  --total_duration 100
```

Expected: `null` (no error).

### 3.4 Smoke test: read back the schedule

```bash
stellar contract invoke \
  --id "$VESTING_CONTRACT" \
  --source "$SOURCE_ACCOUNT" \
  --network testnet \
  -- get_schedule \
  --recipient "$TEST_RECIPIENT"
```

Expected: a `VestingSchedule` struct with the values supplied above.

### 3.5 Smoke test: advance and claim

```bash
# Wait for cliff (ledger advances ~5 s each)
sleep 60

stellar contract invoke \
  --id "$VESTING_CONTRACT" \
  --source "$TEST_RECIPIENT_ACCOUNT" \
  --network testnet \
  -- claim_vested \
  --recipient "$TEST_RECIPIENT"
```

Expected: a positive `i128` amount.

### 3.6 Check the upgrade entry point (if present)

If the contract exposes an `upgrade` function, verify it is callable by the admin key only:

```bash
stellar contract invoke \
  --id "$VESTING_CONTRACT" \
  --source "$SOURCE_ACCOUNT" \
  --network testnet \
  -- upgrade \
  --new_wasm_hash "$TESTNET_WASM_HASH"
```

Expected: `null` (no error). Re-run `get_schedule` to confirm existing state is intact.

- [ ] All testnet smoke tests passed.
- [ ] Upgrade entry point tested on testnet.

---

## 4. Executing the Upgrade on Mainnet

> Do **not** proceed unless all testnet smoke tests passed.

### 4.1 Upload the WASM to mainnet

```bash
stellar contract upload \
  --wasm "$OPTIMIZED" \
  --source "$ADMIN_ACCOUNT" \
  --network mainnet
```

Record the output as `MAINNET_WASM_HASH`. Verify it matches `NEW_WASM_HASH` from step 2.4.

```bash
# Must be identical
[[ "$MAINNET_WASM_HASH" == "$NEW_WASM_HASH" ]] && echo "✅ Hash match" || echo "❌ Hash mismatch — ABORT"
```

### 4.2 Invoke the upgrade entry point

```bash
stellar contract invoke \
  --id "$VESTING_CONTRACT" \
  --source "$ADMIN_ACCOUNT" \
  --network mainnet \
  -- upgrade \
  --new_wasm_hash "$MAINNET_WASM_HASH"
```

Note the transaction hash from the output as `UPGRADE_TX_HASH`.

### 4.3 Record the transaction

```bash
stellar transaction view --hash "$UPGRADE_TX_HASH" --network mainnet
```

Save the full output to the incident/change-management ticket.

---

## 5. Post-Upgrade Verification

### 5.1 Confirm WASM hash changed

```bash
stellar contract info \
  --contract-id "$VESTING_CONTRACT" \
  --network mainnet \
  | grep wasm_hash
```

- [ ] The printed hash equals `MAINNET_WASM_HASH`.
- [ ] The printed hash differs from `OLD_WASM_HASH` (recorded in step 1.2).

### 5.2 Confirm upgrade event emitted

Query the indexer or Horizon directly for an upgrade event on the contract:

```bash
curl -s "https://horizon.stellar.org/contracts/${VESTING_CONTRACT}/events?order=desc&limit=5" \
  | jq '._embedded.records[] | select(.type=="contract") | {id, type: .topic[0]}'
```

Look for a topic string containing `contract_upgraded` or `upgrade` within the last few ledgers.

### 5.3 End-to-end smoke test on mainnet (read-only)

```bash
stellar contract invoke \
  --id "$VESTING_CONTRACT" \
  --source "$SOURCE_ACCOUNT" \
  --network mainnet \
  -- get_schedule \
  --recipient "$KNOWN_ACTIVE_RECIPIENT"
```

Expected: the existing `VestingSchedule` is still intact. Active streams must not be disrupted.

### 5.4 Monitor for 15 minutes

- [ ] Watch `#ops` for any alerts triggered by the indexer or API.
- [ ] Check CloudWatch / application logs for unexpected errors.
- [ ] Confirm indexer continues to ingest new events normally.

### 5.5 Close the maintenance window

- [ ] Post completion notice in `#ops`:
  ```
  [DONE] Contract upgrade complete.
  New WASM hash: <MAINNET_WASM_HASH>
  Upgrade TX: <UPGRADE_TX_HASH>
  All smoke tests passing.
  ```
- [ ] Update the contract ID / hash in `terraform/` or `.env.mainnet` if applicable.
- [ ] Mark the change-management ticket as resolved.

---

## 6. Incident Response During Upgrade

If something goes wrong during steps 4–5:

1. **Do not attempt a second upgrade immediately.** Assess the blast radius first.
2. Check whether existing streams are still readable via `get_schedule` on a known recipient.
3. If the `upgrade` call returned an error, the previous WASM is still active — no state change occurred.
4. Page the IC via PagerDuty if no resolution within 15 minutes.
5. Open an incident in `#incidents` with:
   - The failed transaction hash
   - The current WASM hash (from `stellar contract info`)
   - Last 50 lines of indexer logs

---

## 7. Rollback Procedure

> **On-chain WASM upgrades cannot be reversed.** Once the contract entry points to the new
> WASM hash, you cannot revert to the old binary without a further upgrade.

### What "rollback" means here

A rollback is a **forward fix**: prepare a hotfix release that restores the prior behaviour,
run it through the full upgrade process (steps 2–5), and deploy it.

### Escalation path

If a critical bug is discovered post-upgrade and a hotfix cannot be prepared quickly:

1. **Immediately pause the indexer** to stop surfacing incorrect data:
   ```bash
   # Scale down the indexer ECS task
   aws ecs update-service \
     --cluster vesting-prod \
     --service indexer \
     --desired-count 0 \
     --region us-east-1
   ```
2. **Disable the frontend** or display a maintenance banner to prevent user interaction.
3. **Notify all active recipients** via the communication channel registered at stream creation.
4. **Prepare a hotfix contract** — cherry-pick or revert the breaking change, tag a new release,
   and re-run this runbook from step 1.
5. **Do not cancel active streams** unless absolutely necessary and legally reviewed.

### Contacts

| Role | Contact |
|------|---------|
| On-call engineer | PagerDuty rotation |
| Smart contract lead | `@contract-lead` in Slack |
| Security team | `security@example.com` |
