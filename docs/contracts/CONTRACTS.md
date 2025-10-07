# ZK Stack Contract Inventory

This document describes the complete set of L1 and L2 contracts required for initializing a ZK Stack chain in **Localhost** rollup mode (no-proofs).

## L1 Ecosystem Contracts

These contracts are deployed once per ecosystem and shared across all chains.

| Name | Artifact | Deployment | Create2 Salt | Constructor Args | Address Source | Admin/Owner | Events to Watch | Mode |
|------|----------|------------|--------------|------------------|----------------|-------------|-----------------|------|
| **Governance** | Governance.sol | create | - | `<owner_address>`, `<security_council>`, `<min_delay>` | from_event | owner: `<owner_address>` | `OwnershipTransferStarted`, `OwnershipTransferred` | both |
| **ProxyAdmin** | TransparentUpgradeableProxy (admin) | create | - | - | from_event | - | - | both |
| **Create2Factory** | SingletonFactory.sol | create | - | - | deterministic | - | - | both |
| **Bridgehub (impl)** | Bridgehub.sol | create | - | - | from_event | - | - | both |
| **Bridgehub (proxy)** | TransparentUpgradeableProxy | create | - | `<bridgehub_impl>`, `<proxy_admin>`, `0x` | deterministic | admin: `<proxy_admin>`, owner: `<governance>` | `OwnershipTransferStarted` | both |
| **StateTransitionManager (impl)** | StateTransitionManager.sol | create | - | `<bridgehub_proxy>`, `<max_number_of_chains>` | from_event | - | - | both |
| **StateTransitionManager (proxy)** | TransparentUpgradeableProxy | create | - | `<stm_impl>`, `<proxy_admin>`, `<init_data>` | deterministic | admin: `<proxy_admin>`, owner: `<governance>` | `OwnershipTransferStarted` | both |
| **STMDeploymentTracker (impl)** | ChainTypeManagerDeploymentTracker.sol | create | - | `<bridgehub_proxy>`, `<l1_asset_router>` | from_event | - | - | both |
| **STMDeploymentTracker (proxy)** | TransparentUpgradeableProxy | create | - | `<tracker_impl>`, `<proxy_admin>`, `<init_data>` | deterministic | admin: `<proxy_admin>`, owner: `<governance>` | - | both |
| **ValidatorTimelock** | ValidatorTimelock.sol | create | - | `<owner_address>`, `<execution_delay>`, `<era_chain_id>` | from_event | owner: `<governance>` | - | both |
| **ChainAdmin** | ChainAdmin.sol | create | - | `<restriction_addresses>` | from_event | - | - | both |
| **AccessControlRestriction** | AccessControlRestriction.sol | create | - | `<min_delay>`, `<chain_admin>` | from_event | - | - | both |
| **L1AssetRouter (impl)** | L1AssetRouter.sol | create | - | `<l1_wrapped_base_token_store>`, `<bridgehub_proxy>`, `<l1_nullifier>`, `<era_chain_id>` | from_event | - | - | both |
| **L1AssetRouter (proxy)** | TransparentUpgradeableProxy | create | - | `<asset_router_impl>`, `<proxy_admin>`, `<init_data>` | deterministic | admin: `<proxy_admin>`, owner: `<governance>` | `OwnershipTransferStarted` | both |
| **L1Nullifier (impl)** | L1Nullifier.sol | create | - | `<bridgehub_proxy>`, `<era_chain_id>`, `<l1_asset_router>` | from_event | - | - | both |
| **L1Nullifier (proxy)** | TransparentUpgradeableProxy | create | - | `<nullifier_impl>`, `<proxy_admin>`, `<init_data>` | deterministic | admin: `<proxy_admin>`, owner: `<governance>` | `OwnershipTransferStarted` | both |
| **L1ERC20Bridge (impl)** | L1ERC20Bridge.sol | create | - | `<l1_nullifier>`, `<l1_asset_router>`, `<native_token_vault>`, `<era_chain_id>` | from_event | - | - | both |
| **L1ERC20Bridge (proxy)** | TransparentUpgradeableProxy | create | - | `<bridge_impl>`, `<proxy_admin>`, `<init_data>` | deterministic | admin: `<proxy_admin>` | - | both |
| **NativeTokenVault (impl)** | L1NativeTokenVault.sol | create | - | `<l1_wrapped_base_token_store>`, `<l1_asset_router>`, `<l1_nullifier>` | from_event | - | - | both |
| **NativeTokenVault (proxy)** | TransparentUpgradeableProxy | create | - | `<vault_impl>`, `<proxy_admin>`, `<init_data>` | deterministic | admin: `<proxy_admin>`, owner: `<governance>` | `OwnershipTransferStarted` | both |
| **MessageRoot (impl)** | MessageRoot.sol | create | - | `<bridgehub_proxy>` | from_event | - | - | both |
| **MessageRoot (proxy)** | TransparentUpgradeableProxy | create | - | `<msg_root_impl>`, `<proxy_admin>`, `0x` | deterministic | admin: `<proxy_admin>` | - | both |
| **ServerNotifier (impl)** | ServerNotifier.sol | create | - | - | from_event | - | - | both |
| **ServerNotifier (proxy)** | TransparentUpgradeableProxy | create | - | `<notifier_impl>`, `<proxy_admin>`, `0x` | deterministic | admin: `<proxy_admin>` | - | both |

## L1 Diamond Facets & Supporting Contracts

These contracts support the ZK chain diamond proxy pattern.

| Name | Artifact | Deployment | Create2 Salt | Constructor Args | Address Source | Admin/Owner | Events to Watch | Mode |
|------|----------|------------|--------------|------------------|----------------|-------------|-----------------|------|
| **Verifier** | Verifier.sol | create | - | - | from_event | - | - | both |
| **AdminFacet** | AdminFacet.sol | create | - | - | from_event | - | - | both |
| **GettersFacet** | GettersFacet.sol | create | - | - | from_event | - | - | both |
| **MailboxFacet** | MailboxFacet.sol | create | - | `<era_chain_id>`, `<l1_gas_per_pubdata_byte>` | from_event | - | - | both |
| **ExecutorFacet** | ExecutorFacet.sol | create | - | `<l1_gas_per_pubdata_byte>` | from_event | - | - | both |
| **DiamondInit** | DiamondInit.sol | create | - | - | from_event | - | - | both |
| **GenesisUpgrade** | GenesisUpgrade.sol | create | - | - | from_event | - | - | both |
| **DefaultUpgrade** | DefaultUpgrade.sol | create | - | - | from_event | - | - | both |
| **L1BytecodesSupplier** | L1BytecodesSupplier.sol | create | - | - | from_event | - | - | both |

## L1 DA Validator Contracts

Data availability validators for different modes.

| Name | Artifact | Deployment | Create2 Salt | Constructor Args | Address Source | Admin/Owner | Events to Watch | Mode |
|------|----------|------------|--------------|------------------|----------------|-------------|-----------------|------|
| **RollupL1DAValidator** | RollupL1DAValidator.sol | create | - | - | from_event | - | - | rollup |
| **ValidiumL1DAValidator** | ValidiumL1DAValidator.sol | create | - | - | from_event | - | - | validium |
| **AvailDAValidator** | AvailDAValidator.sol | create | - | - | from_event | - | - | validium |

## L1 Chain-Specific Contracts

These contracts are deployed per chain during chain registration.

| Name | Artifact | Deployment | Create2 Salt | Constructor Args | Address Source | Admin/Owner | Events to Watch | Mode |
|------|----------|------------|--------------|------------------|----------------|-------------|-----------------|------|
| **DiamondProxy** | DiamondProxy.sol | create2 | `<bridgehub_create_new_chain_salt>` | `<chain_id>`, `<diamond_cut>` | from_event(ChainRegistered) | admin: `<chain_admin>` | `ChainRegistered` | both |
| **ChainProxyAdmin** | ProxyAdmin.sol | create | - | `<owner>` | from_event | owner: `<chain_admin>` | - | both |

## L2 Predeployed System Contracts

These are L2 contracts predeployed at genesis or deployed during chain initialization.

| Name | Artifact | Deployment | Create2 Salt | Constructor Args | Address Source | Admin/Owner | Events to Watch | Mode |
|------|----------|------------|--------------|------------------|----------------|-------------|-----------------|------|
| **L2AssetRouter** | L2AssetRouter.sol | predeployed | - | - | known_constant (`0x10003`) | - | - | both |
| **L2NativeTokenVault** | L2NativeTokenVault.sol | predeployed | - | - | known_constant (`0x10004`) | - | - | both |
| **L2DefaultUpgrader** | L2DefaultUpgrader.sol | deployed | - | - | from_event | - | - | both |
| **L2DAValidator** | L2DAValidator.sol | deployed | - | `<l1_da_validator>` | from_event | - | - | both |
| **ConsensusRegistry** | ConsensusRegistry.sol (optional) | deployed | - | `<owner>`, `<nodes>` | from_event | owner: `<governor>` | - | both |
| **Multicall3** | Multicall3.sol (optional) | deployed | - | - | from_event | - | - | both |
| **TimestampAsserter** | TimestampAsserter.sol (optional) | deployed | - | `<l1_asset_router>` | from_event | - | - | both |
| **TestnetPaymaster** | TestnetPaymaster.sol (optional) | deployed | - | - | from_event | - | - | both |

## Utility Contracts

| Name | Artifact | Deployment | Create2 Salt | Constructor Args | Address Source | Admin/Owner | Events to Watch | Mode |
|------|----------|------------|--------------|------------------|----------------|-------------|-----------------|------|
| **Multicall3** | Multicall3.sol | create | - | - | from_event | - | - | both |
| **BlobVersionedHashRetriever** | BlobVersionedHashRetriever.sol | create | - | - | from_event | - | - | both |

## Contract Relationships

```
Governance (owner)
  ├─> Bridgehub (proxy)
  ├─> StateTransitionManager (proxy)
  ├─> L1AssetRouter (proxy)
  ├─> L1Nullifier (proxy)
  └─> NativeTokenVault (proxy)

ChainAdmin
  ├─> DiamondProxy (per chain)
  └─> AccessControlRestriction

Bridgehub
  ├─> registers chains
  ├─> manages base tokens
  └─> routes messages via MessageRoot

StateTransitionManager
  ├─> manages chain upgrades
  ├─> validates state transitions
  └─> provides DiamondCut data

DiamondProxy (per chain)
  ├─> AdminFacet
  ├─> GettersFacet
  ├─> MailboxFacet
  └─> ExecutorFacet
```

## Deployment Order

1. **Infrastructure**: Governance, ProxyAdmin, Create2Factory
2. **Core Ecosystem**: Bridgehub, StateTransitionManager, ValidatorTimelock
3. **Bridges**: L1AssetRouter, L1Nullifier, L1ERC20Bridge, NativeTokenVault
4. **Utilities**: MessageRoot, Multicall3, BlobVersionedHashRetriever
5. **Diamond Components**: Verifier, Facets, DiamondInit, Upgrades
6. **DA Validators**: RollupL1DAValidator, ValidiumL1DAValidator, AvailDAValidator
7. **Chain-Specific** (per chain): DiamondProxy, ChainProxyAdmin
8. **L2 Contracts** (post-registration): L2DefaultUpgrader, L2DAValidator, etc.

## Notes

- All proxy contracts use the TransparentUpgradeableProxy pattern
- `<placeholders>` indicate symbolic values that will be substituted at runtime
- Events marked in "Events to Watch" are critical for determining deployment success
- Ownership transfer follows a two-step process: propose → accept
- The exact bytecode and ABI artifacts are located in the `zksync-era` repository under `contracts/l1-contracts/artifacts`
