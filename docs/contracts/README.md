# Contract Manifest Documentation

This directory contains the canonical specification for ZK Stack contract deployment and chain initialization.

## Overview

The documentation in this directory serves as the **single source of truth** for:

1. **Contract Inventory**: All L1 and L2 contracts required for a ZK Stack ecosystem
2. **Deployment Flow**: The exact order and parameters for deploying ecosystem contracts
3. **Initialization Flow**: The steps to register and initialize a new ZK chain
4. **Validation**: How to verify deployment success and troubleshoot issues

These manifests are designed to be:
- **Machine-consumable**: YAML and JSON formats for automation
- **Deterministic**: Clear ordering and dependencies
- **Idempotent**: Safe to re-run without side effects
- **Address-agnostic**: Uses symbolic placeholders for runtime substitution

## Files

### CONTRACTS.md
Human-readable contract inventory with:
- Complete list of L1 ecosystem contracts
- L1 diamond facets and supporting contracts
- L1 DA validators for different modes (rollup/validium)
- L2 predeployed and deployed contracts
- Contract relationships and deployment order

**Use when**: Understanding the overall architecture or looking up specific contract details.

### contracts.json
Machine-readable version of CONTRACTS.md containing:
- Structured contract metadata
- Constructor arguments (symbolic)
- Deployment types (create, create2, predeployed)
- Address sources (from_event, deterministic, known_constant)
- Admin/owner roles
- Events to watch

**Use when**: Building automation tools or parsing contract specifications programmatically.

### deploy-manifest.yaml
Step-by-step L1 ecosystem deployment manifest:
- Ordered deployment steps (Phase 1-15)
- Transaction parameters for each contract
- Dependencies between steps
- Idempotency checks
- Postcondition validations

**Use when**: 
- Implementing a single-command deployment tool
- Understanding the ecosystem deployment flow
- Debugging deployment issues

**Example step**:
```yaml
- id: bridgehub_proxy_deploy
  from_role: deployer
  to: "0x0"
  method: "TransparentUpgradeableProxy.constructor(address,address,bytes)"
  params:
    implementation: "<bridgehub_impl>"
    admin: "<proxy_admin>"
    data: "0x"
  deployment: create
  depends_on: [bridgehub_impl_deploy]
  idempotency_check: "code_at(<bridgehub_proxy>) != 0x"
  postcondition: "proxy_impl(<bridgehub_proxy>) == <bridgehub_impl>"
```

### init-manifest.yaml
Step-by-step chain initialization manifest:
- Funding wallets (Localhost only)
- Base token setup (custom tokens)
- Chain registration with Bridgehub
- Admin acceptance
- L2 contract deployment
- DA validator pairing
- Genesis generation

**Use when**:
- Implementing chain initialization automation
- Understanding the chain init flow
- Debugging chain registration issues

**Includes**:
- Conditional steps (only run if certain flags are set)
- Validation plan for end-state verification
- Role mapping with descriptions

### source-map.json
Maps each manifest step to its source code location:
- Solidity script files and line numbers
- Rust implementation functions
- Broadcast transaction locations
- Config file paths

**Use when**:
- Tracing a manifest step back to its implementation
- Understanding where a particular transaction comes from
- Debugging by examining the original source

**Example**:
```json
"register_zk_chain": {
  "file": "contracts/l1-contracts/deploy-scripts/RegisterZKChain.s.sol",
  "function": "run",
  "lines": [100, 250],
  "broadcast": "contracts/l1-contracts/broadcast/RegisterZKChain.s.sol/*/run-*.json",
  "rust_caller": "crates/zkstack/src/commands/chain/register_chain.rs:register_chain"
}
```

### validation.md
Comprehensive validation and troubleshooting guide:
- Prerequisites checks
- Role-to-key mapping
- Deployment validation procedures
- Chain initialization validation
- Idempotency testing
- Healthy end-state checklist
- Common issues and solutions

**Use when**:
- Validating a deployment
- Verifying idempotency
- Troubleshooting failures
- Confirming operational readiness

## Usage Scenarios

### Scenario 1: Understanding the Deployment Flow

1. Start with **CONTRACTS.md** to understand the overall architecture
2. Review **deploy-manifest.yaml** for ecosystem deployment steps
3. Review **init-manifest.yaml** for chain initialization steps
4. Check **source-map.json** to find the implementation details

### Scenario 2: Building an Automation Tool

1. Parse **contracts.json** for contract specifications
2. Parse **deploy-manifest.yaml** for deployment steps
3. Parse **init-manifest.yaml** for initialization steps
4. Implement idempotency checks from both manifests
5. Use **source-map.json** to link errors back to source code

### Scenario 3: Validating a Deployment

1. Follow the checklists in **validation.md**
2. Run precondition checks before deployment
3. Validate each phase of deployment using the provided scripts
4. Verify idempotency by re-running deployment
5. Confirm healthy end-state using the checklist

### Scenario 4: Troubleshooting Issues

1. Identify the failing step from logs
2. Check **validation.md** for common issues
3. Use **source-map.json** to find the source code
4. Review the step in the manifest for expected behavior
5. Use debug commands from **validation.md**

## Symbolic Placeholders

All manifests use symbolic placeholders for addresses and parameters that will be substituted at runtime:

| Placeholder | Description | Source |
|-------------|-------------|--------|
| `<deployer>` | Deployer wallet address | `wallets.yaml:deployer.address` |
| `<governor>` | Governor wallet address | `wallets.yaml:governor.address` |
| `<operator>` | Operator wallet address | `wallets.yaml:operator.address` |
| `<blob_operator>` | Blob operator wallet address | `wallets.yaml:blob_operator.address` |
| `<bridgehub_proxy>` | Bridgehub proxy address | Deployed in deploy-manifest |
| `<stm_proxy>` | State Transition Manager proxy | Deployed in deploy-manifest |
| `<diamond_proxy>` | Chain's DiamondProxy address | Created during chain registration |
| `<chain_id>` | Chain ID | `chains/<name>/config.yaml:chain_id` |
| `<base_token_addr>` | Base token address | `chains/<name>/config.yaml:base_token.address` |
| `<era_chain_id>` | Era chain ID | `configs/general.yaml:era_chain_id` |

These placeholders ensure the manifests are reusable across different deployments.

## Network Modes

The manifests support different network and chain modes:

### Network Types
- **Localhost**: Local development (Anvil/Reth) with funding steps
- **Testnet**: Public testnet (Sepolia/Holesky) without funding
- **Mainnet**: Production deployment (referenced but not detailed)

### Chain Modes
- **Rollup**: Full data availability on L1 (default)
- **Validium**: Data availability off-chain (requires DA layer)
- **Both**: Contract works in both modes

The current manifests focus on the **Localhost + Rollup** happy path as specified in the requirements.

## Idempotency

Every step in the manifests includes:

1. **Idempotency Check**: Condition to check if step is already done
2. **Postcondition**: Verification that step succeeded

This allows safe re-execution of the entire deployment/initialization flow.

**Example**:
```yaml
idempotency_check: "code_at(<bridgehub_proxy>) != 0x"
postcondition: "proxy_impl(<bridgehub_proxy>) == <bridgehub_impl>"
```

If `code_at(<bridgehub_proxy>) != 0x` is true, the step is skipped.
After execution (or skip), verify `proxy_impl(<bridgehub_proxy>) == <bridgehub_impl>`.

## Future Work

### Validium Deltas
The current manifests focus on rollup mode. Validium differences include:
- Different DA validator selection (`ValidiumL1DAValidator` vs `RollupL1DAValidator`)
- Avail DA validator integration (if using Avail)
- Different DA layer configuration

**TODO**: Document validium-specific steps and parameter differences.

### Production Deployment
For production deployments, additional considerations:
- Multi-sig governance instead of single governor
- Hardware wallet integration
- Timelock delays on admin operations
- Security audits and verification
- Monitoring and alerting setup

### Contract Artifacts
Future work should include:
- Bundling ABI/bytecode artifacts
- Contract versioning and compatibility
- Upgrade paths and migration guides

## Integration with zkstack CLI

The zkstack CLI currently implements these flows in Rust and Solidity:

- **Ecosystem Init**: `crates/zkstack/src/commands/ecosystem/init.rs`
- **Chain Init**: `crates/zkstack/src/commands/chain/init/mod.rs`
- **Deploy L1**: `crates/zkstack/src/commands/ecosystem/common.rs`
- **Register Chain**: `crates/zkstack/src/commands/chain/register_chain.rs`

The manifests in this directory provide a canonical specification that can be used to:
1. Validate the current implementation
2. Build alternative implementations (e.g., in other languages)
3. Create simplified deployment tools (e.g., `zksup` single-command setup)

## Contributing

When updating the deployment flow:

1. Update the relevant manifest (deploy-manifest.yaml or init-manifest.yaml)
2. Update contracts.json if adding/removing contracts
3. Update CONTRACTS.md with human-readable descriptions
4. Update source-map.json with new code locations
5. Update validation.md if new validation steps are needed
6. Test idempotency of the updated flow

## References

- [ZK Stack Documentation](https://docs.zksync.io/zk-stack)
- [zkstack-cli Repository](https://github.com/dutterbutter/zkstack-cli)
- [zksync-era Repository](https://github.com/matter-labs/zksync-era)
- [Foundry Scripts](https://github.com/matter-labs/zksync-era/tree/main/contracts/l1-contracts/deploy-scripts)

## Questions?

For questions about these manifests or the deployment flow:
- Open an issue in the [zkstack-cli repository](https://github.com/dutterbutter/zkstack-cli/issues)
- Check the validation.md troubleshooting section
- Review the source-map.json to trace back to implementation
