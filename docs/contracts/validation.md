# Validation Runbook for ZK Stack Deployment

This document describes how to validate a ZK Stack ecosystem and chain deployment, check preconditions, verify idempotency, and confirm a healthy end-state.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Role-to-Key Mapping](#role-to-key-mapping)
3. [Precondition Checks](#precondition-checks)
4. [Deployment Validation](#deployment-validation)
5. [Chain Initialization Validation](#chain-initialization-validation)
6. [Idempotency Testing](#idempotency-testing)
7. [Healthy End-State Checklist](#healthy-end-state-checklist)
8. [Troubleshooting](#troubleshooting)

## Prerequisites

Before validating, ensure you have:

- Access to the L1 RPC endpoint (e.g., `http://localhost:8545` for Anvil)
- Access to the L2 RPC endpoint (e.g., `http://localhost:3050`)
- `cast` (from Foundry) installed for on-chain queries
- `zkstack` CLI installed
- Access to the ecosystem configuration files

```bash
# Set environment variables
export L1_RPC_URL="http://localhost:8545"
export L2_RPC_URL="http://localhost:3050"
export ECOSYSTEM_PATH="/path/to/ecosystem"
```

## Role-to-Key Mapping

The ZK Stack deployment uses several roles, each with specific permissions. Map these roles to actual private keys using the `wallets.yaml` configuration:

### Role Definitions

| Role | Responsibility | Config Location | Used In |
|------|---------------|----------------|---------|
| **deployer** | Deploys all contracts, funds initial wallets | `wallets.yaml:deployer` | Ecosystem deployment |
| **governor** | Governance operations, chain registration, ownership acceptance | `wallets.yaml:governor` | Chain init, admin functions |
| **operator** | Batch commits and execution | `wallets.yaml:operator` | Chain operation |
| **blob_operator** | Blob submission for batches | `wallets.yaml:blob_operator` | Chain operation |
| **fee_account** | Collects transaction fees | `wallets.yaml:fee_account` | Chain operation |
| **token_multiplier_setter** | Sets token price multipliers (custom base tokens) | `wallets.yaml:token_multiplier_setter` | Chain init (optional) |

### Reading Wallet Configuration

```bash
# Extract addresses from wallets.yaml
DEPLOYER=$(yq '.deployer.address' configs/wallets.yaml)
GOVERNOR=$(yq '.governor.address' configs/wallets.yaml)
OPERATOR=$(yq '.operator.address' configs/wallets.yaml)
BLOB_OPERATOR=$(yq '.blob_operator.address' configs/wallets.yaml)

echo "Deployer: $DEPLOYER"
echo "Governor: $GOVERNOR"
echo "Operator: $OPERATOR"
echo "Blob Operator: $BLOB_OPERATOR"
```

### Private Key Usage

Private keys should **never** be stored in the manifest files. They are read at runtime from:

1. **Environment variables**: `DEPLOYER_PRIVATE_KEY`, `GOVERNOR_PRIVATE_KEY`, etc.
2. **Wallet configuration**: `wallets.yaml` (addresses only, keys in secure storage)
3. **Hardware wallets**: Ledger/Trezor integration (recommended for production)

## Precondition Checks

### 1. L1 Network Availability

```bash
# Check L1 network is accessible
cast block-number --rpc-url $L1_RPC_URL
```

**Expected**: Returns current block number (e.g., `12345`)

### 2. Wallet Balances

Check that deployer and governor have sufficient ETH for gas:

```bash
# Check deployer balance
cast balance $DEPLOYER --rpc-url $L1_RPC_URL --ether

# Check governor balance
cast balance $GOVERNOR --rpc-url $L1_RPC_URL --ether
```

**Expected**: Both wallets have > 1 ETH for Localhost deployments

### 3. Contract Artifacts Exist

Verify that contract artifacts are compiled:

```bash
# Check if L1 contracts are built
ls -la contracts/l1-contracts/out/

# Check for key contract artifacts
test -f contracts/l1-contracts/out/Bridgehub.sol/Bridgehub.json && echo "✓ Bridgehub artifact exists"
test -f contracts/l1-contracts/out/StateTransitionManager.sol/StateTransitionManager.json && echo "✓ STM artifact exists"
```

## Deployment Validation

### 1. Governance Contract

```bash
GOVERNANCE=$(yq '.l1.governance_addr' configs/contracts.yaml)

# Check code exists
cast code $GOVERNANCE --rpc-url $L1_RPC_URL

# Check owner
OWNER=$(cast call $GOVERNANCE "owner()(address)" --rpc-url $L1_RPC_URL)
echo "Governance owner: $OWNER"
```

**Expected**: Owner should be the governor address

### 2. Bridgehub

```bash
BRIDGEHUB=$(yq '.ecosystem_contracts.bridgehub_proxy_addr' configs/contracts.yaml)

# Check code exists
cast code $BRIDGEHUB --rpc-url $L1_RPC_URL

# Check implementation
IMPL=$(cast call $BRIDGEHUB "implementation()(address)" --rpc-url $L1_RPC_URL)
echo "Bridgehub implementation: $IMPL"

# Check owner
OWNER=$(cast call $BRIDGEHUB "owner()(address)" --rpc-url $L1_RPC_URL)
echo "Bridgehub owner: $OWNER"
```

**Expected**: 
- Implementation address is non-zero
- Owner is the governance contract

### 3. StateTransitionManager

```bash
STM=$(yq '.ecosystem_contracts.state_transition_proxy_addr' configs/contracts.yaml)

# Check code exists
cast code $STM --rpc-url $L1_RPC_URL

# Check owner
OWNER=$(cast call $STM "owner()(address)" --rpc-url $L1_RPC_URL)
echo "STM owner: $OWNER"
```

**Expected**: Owner is the governance contract

### 4. Bridge Contracts

```bash
ASSET_ROUTER=$(yq '.bridges.shared.l1_address' configs/contracts.yaml)
NULLIFIER=$(yq '.bridges.l1_nullifier_addr' configs/contracts.yaml)
VAULT=$(yq '.ecosystem_contracts.native_token_vault_addr' configs/contracts.yaml)

# Check L1AssetRouter
cast code $ASSET_ROUTER --rpc-url $L1_RPC_URL
OWNER=$(cast call $ASSET_ROUTER "owner()(address)" --rpc-url $L1_RPC_URL)
echo "AssetRouter owner: $OWNER"

# Check L1Nullifier
cast code $NULLIFIER --rpc-url $L1_RPC_URL

# Check NativeTokenVault
cast code $VAULT --rpc-url $L1_RPC_URL
```

**Expected**: All contracts have code deployed and correct ownership

### 5. Diamond Facets

```bash
ADMIN_FACET=$(yq '.l1.admin_facet_addr' contracts/l1-contracts/script-out/output-deploy-l1.toml 2>/dev/null)
GETTERS_FACET=$(yq '.deployed_addresses.state_transition.getters_facet_addr' contracts/l1-contracts/script-out/output-deploy-l1.toml 2>/dev/null)
MAILBOX_FACET=$(yq '.deployed_addresses.state_transition.mailbox_facet_addr' contracts/l1-contracts/script-out/output-deploy-l1.toml 2>/dev/null)
EXECUTOR_FACET=$(yq '.deployed_addresses.state_transition.executor_facet_addr' contracts/l1-contracts/script-out/output-deploy-l1.toml 2>/dev/null)

# Check each facet has code
cast code $ADMIN_FACET --rpc-url $L1_RPC_URL | grep -q "0x" && echo "✓ AdminFacet deployed"
cast code $GETTERS_FACET --rpc-url $L1_RPC_URL | grep -q "0x" && echo "✓ GettersFacet deployed"
cast code $MAILBOX_FACET --rpc-url $L1_RPC_URL | grep -q "0x" && echo "✓ MailboxFacet deployed"
cast code $EXECUTOR_FACET --rpc-url $L1_RPC_URL | grep -q "0x" && echo "✓ ExecutorFacet deployed"
```

## Chain Initialization Validation

### 1. Chain Registration

```bash
CHAIN_ID=$(yq '.chain_id' chains/era1/config.yaml)
DIAMOND_PROXY=$(yq '.l1.diamond_proxy_addr' chains/era1/configs/contracts.yaml)

# Check chain is registered in Bridgehub
REGISTERED=$(cast call $BRIDGEHUB "getZKChain(uint256)(address)" $CHAIN_ID --rpc-url $L1_RPC_URL)
echo "Registered chain address: $REGISTERED"

# Verify it matches the diamond proxy
if [ "$REGISTERED" = "$DIAMOND_PROXY" ]; then
    echo "✓ Chain correctly registered"
else
    echo "✗ Chain registration mismatch"
fi
```

**Expected**: Bridgehub returns the DiamondProxy address for the chain ID

### 2. DiamondProxy Deployment

```bash
# Check DiamondProxy has code
cast code $DIAMOND_PROXY --rpc-url $L1_RPC_URL | grep -q "0x" && echo "✓ DiamondProxy deployed"

# Check facets are attached
FACET_COUNT=$(cast call $DIAMOND_PROXY "facets()(tuple[])" --rpc-url $L1_RPC_URL | grep -c "0x")
echo "Number of facets attached: $FACET_COUNT"
```

**Expected**: DiamondProxy has code and at least 4 facets (Admin, Getters, Mailbox, Executor)

### 3. Chain Admin Acceptance

```bash
CHAIN_ADMIN=$(yq '.l1.chain_admin_addr' chains/era1/configs/contracts.yaml)

# Check admin has accepted role
ADMIN=$(cast call $DIAMOND_PROXY "getAdmin()(address)" --rpc-url $L1_RPC_URL)
echo "DiamondProxy admin: $ADMIN"

if [ "$ADMIN" = "$CHAIN_ADMIN" ]; then
    echo "✓ Chain admin accepted"
else
    echo "✗ Chain admin not accepted"
fi
```

### 4. L2 Bridge Initialization

```bash
L2_ASSET_ROUTER="0x0000000000000000000000000000000000010003"
L2_VAULT="0x0000000000000000000000000000000000010004"

# Check L2 AssetRouter is initialized (requires L2 RPC)
L1_ROUTER=$(cast call $L2_ASSET_ROUTER "l1AssetRouter()(address)" --rpc-url $L2_RPC_URL 2>/dev/null)
if [ -n "$L1_ROUTER" ] && [ "$L1_ROUTER" != "0x0000000000000000000000000000000000000000" ]; then
    echo "✓ L2 AssetRouter initialized, L1 router: $L1_ROUTER"
else
    echo "✗ L2 AssetRouter not initialized"
fi

# Check L2 NativeTokenVault is initialized
L1_VAULT=$(cast call $L2_VAULT "l1NativeTokenVault()(address)" --rpc-url $L2_RPC_URL 2>/dev/null)
if [ -n "$L1_VAULT" ] && [ "$L1_VAULT" != "0x0000000000000000000000000000000000000000" ]; then
    echo "✓ L2 NativeTokenVault initialized, L1 vault: $L1_VAULT"
else
    echo "✗ L2 NativeTokenVault not initialized"
fi
```

### 5. DA Validator Setup

```bash
# Check L1 DA validator is set
L1_DA_VALIDATOR=$(cast call $BRIDGEHUB "getL1DAValidator(uint256)(address)" $CHAIN_ID --rpc-url $L1_RPC_URL)
echo "L1 DA Validator: $L1_DA_VALIDATOR"

# Check L2 DA validator is set
L2_DA_VALIDATOR=$(cast call $BRIDGEHUB "getL2DAValidator(uint256)(address)" $CHAIN_ID --rpc-url $L1_RPC_URL)
echo "L2 DA Validator: $L2_DA_VALIDATOR"
```

**Expected**: Both validators are non-zero addresses

### 6. Base Token Configuration

```bash
# Check base token
BASE_TOKEN=$(cast call $DIAMOND_PROXY "getBaseToken()(address)" --rpc-url $L1_RPC_URL)
echo "Base token: $BASE_TOKEN"

# For ETH, address should be 0x0000000000000000000000000000000000000001
# For custom tokens, should be the token contract address
```

## Idempotency Testing

Idempotency means that running the deployment/initialization again should not fail or cause double-application. Test this by:

### 1. Re-run Ecosystem Init

```bash
# This should skip already-deployed contracts
zkstack ecosystem init --skip-contract-compilation

# Check logs for "already deployed" or "skipping" messages
```

**Expected**: All steps report "already exists" or skip deployment

### 2. Re-run Chain Init

```bash
# This should skip already-initialized components
zkstack chain init --chain era1

# Check for idempotency messages
```

**Expected**: 
- "Chain already registered" message
- "Admin already accepted" message
- No duplicate contract deployments

### 3. Manual Idempotency Check

For each step in the manifests, verify the `idempotency_check` condition:

```bash
# Example: Check if Bridgehub has code (from deploy-manifest.yaml)
BRIDGEHUB=$(yq '.ecosystem_contracts.bridgehub_proxy_addr' configs/contracts.yaml)
CODE=$(cast code $BRIDGEHUB --rpc-url $L1_RPC_URL)
if [ "$CODE" != "0x" ]; then
    echo "✓ Bridgehub already deployed (idempotent)"
else
    echo "✗ Bridgehub not deployed"
fi
```

## Healthy End-State Checklist

Use this checklist to confirm the deployment is successful and ready for chain operation:

### L1 Ecosystem Contracts

- [ ] Governance deployed with correct owner
- [ ] Bridgehub deployed and initialized
- [ ] StateTransitionManager deployed and initialized
- [ ] L1AssetRouter deployed and owned by governance
- [ ] L1Nullifier deployed and owned by governance
- [ ] NativeTokenVault deployed and owned by governance
- [ ] ValidatorTimelock deployed
- [ ] ChainAdmin deployed
- [ ] All diamond facets deployed
- [ ] DA validators deployed (Rollup/Validium)
- [ ] Multicall3 deployed

### Chain Initialization

- [ ] Chain registered in Bridgehub
- [ ] DiamondProxy deployed at deterministic address
- [ ] DiamondProxy has all facets attached
- [ ] Chain admin accepted admin role
- [ ] Base token configured correctly
- [ ] L2 AssetRouter initialized
- [ ] L2 NativeTokenVault initialized
- [ ] L2 DA Validator deployed
- [ ] L2 DefaultUpgrader deployed
- [ ] DA validator pair set (L1 <-> L2 link)
- [ ] Genesis database initialized

### Optional Components

- [ ] Testnet Paymaster deployed (if `deploy_paymaster=true`)
- [ ] ConsensusRegistry deployed (if configured)
- [ ] Multicall3 deployed on L2 (if configured)
- [ ] TimestampAsserter deployed (if configured)
- [ ] Token multiplier setter configured (if custom base token)
- [ ] EVM emulator enabled (if configured)

### Operational Readiness

- [ ] L2 RPC responds to requests (`eth_chainId`, `eth_blockNumber`)
- [ ] Can submit L1->L2 transactions via Bridgehub
- [ ] Operator wallet has sufficient balance
- [ ] Blob operator wallet has sufficient balance
- [ ] Server can connect to database
- [ ] Genesis batch (batch 0) exists in database

### Quick Validation Script

```bash
#!/bin/bash
# validate-deployment.sh

set -e

echo "=== ZK Stack Deployment Validation ==="

# Load config
BRIDGEHUB=$(yq '.ecosystem_contracts.bridgehub_proxy_addr' configs/contracts.yaml)
CHAIN_ID=$(yq '.chain_id' chains/era1/config.yaml)
DIAMOND=$(yq '.l1.diamond_proxy_addr' chains/era1/configs/contracts.yaml)

echo "Bridgehub: $BRIDGEHUB"
echo "Chain ID: $CHAIN_ID"
echo "DiamondProxy: $DIAMOND"

# Check Bridgehub
echo -n "Checking Bridgehub... "
cast code $BRIDGEHUB --rpc-url $L1_RPC_URL > /dev/null && echo "✓" || echo "✗"

# Check chain registration
echo -n "Checking chain registration... "
REGISTERED=$(cast call $BRIDGEHUB "getZKChain(uint256)(address)" $CHAIN_ID --rpc-url $L1_RPC_URL)
[ "$REGISTERED" = "$DIAMOND" ] && echo "✓" || echo "✗"

# Check DiamondProxy
echo -n "Checking DiamondProxy... "
cast code $DIAMOND --rpc-url $L1_RPC_URL > /dev/null && echo "✓" || echo "✗"

# Check L2 RPC
echo -n "Checking L2 RPC... "
cast block-number --rpc-url $L2_RPC_URL > /dev/null 2>&1 && echo "✓" || echo "✗ (L2 not running)"

echo "=== Validation Complete ==="
```

## Troubleshooting

### Common Issues

#### 1. "Owner not set" or "Owner is zero address"

**Cause**: Ownership transfer not completed  
**Solution**: Run accept ownership step manually

```bash
zkstack admin accept-ownership --target <contract_address>
```

#### 2. "Chain not registered"

**Cause**: `register_zk_chain` step failed or not executed  
**Solution**: Re-run chain registration

```bash
zkstack chain init --chain <chain-name>
```

#### 3. "L2 contracts not initialized"

**Cause**: L2 deployment transactions not sent or failed  
**Solution**: Check L1->L2 transaction status and re-run L2 deployment

```bash
# Check L1->L2 tx status
cast call $DIAMOND_PROXY "getL2SystemContractsUpgradeBatchNumber()(uint256)" --rpc-url $L1_RPC_URL

# Re-deploy L2 contracts
zkstack chain init --chain <chain-name>
```

#### 4. "Genesis not initialized"

**Cause**: Genesis generation failed or database not accessible  
**Solution**: Re-run genesis with correct database credentials

```bash
zkstack chain genesis --chain <chain-name> \
  --server-db-url postgres://postgres@localhost/zksync_server \
  --server-db-name zksync_server
```

#### 5. Idempotency check fails on re-run

**Cause**: Contract state changed unexpectedly  
**Solution**: Check contract events and state

```bash
# Check recent events
cast logs --from-block <start> --to-block latest --address $CONTRACT --rpc-url $L1_RPC_URL
```

### Debug Commands

```bash
# View all facets of DiamondProxy
cast call $DIAMOND_PROXY "facets()(tuple[])" --rpc-url $L1_RPC_URL

# Check protocol version
cast call $DIAMOND_PROXY "getProtocolVersion()(uint256)" --rpc-url $L1_RPC_URL

# Check admin
cast call $DIAMOND_PROXY "getAdmin()(address)" --rpc-url $L1_RPC_URL

# Check pending admin
cast call $DIAMOND_PROXY "getPendingAdmin()(address)" --rpc-url $L1_RPC_URL

# Check verifier
cast call $DIAMOND_PROXY "getVerifier()(address)" --rpc-url $L1_RPC_URL
```

## Conclusion

Following this validation runbook ensures:

1. ✅ All contracts are deployed correctly
2. ✅ Ownership is properly transferred
3. ✅ Chain is registered and operational
4. ✅ Idempotency is maintained
5. ✅ End-state is healthy and ready for operation

For issues not covered here, check the [zkstack-cli repository](https://github.com/dutterbutter/zkstack-cli) or the ZKsync Era documentation.
