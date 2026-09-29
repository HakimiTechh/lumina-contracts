# Registry Events Reference

Events are the integration surface for downstream consumers, serving as the interface for both `lumina-backend`s indexer and `lumina-frontend`s registry history view.

## Downstream Consumers

- `Registry History (`lumina-frontend`)`: The frontend's history view rebuilds the per-contract event timeline by matching on the **first topic** (which must be the event name) and treating the **first data slot** as the subject ID (`contract_id`). Any events matching this shape will be attributed to the respective contract's history. Unknown topics will still be displayed as generic "Registry event" rows.
- `Indexer (`lumina-backend`)`: The backend indexer discovers contracts and listens to registry events to keep its database synchronized with the on-chain manifest. It specifically looks for registration, deactivation, and metadata/category changes to maintain an up-to-date registry graph.

## Events

| Topic | Payload Shape | When It Fires | Consumers | Example Test Reference |
|---|---|---|---|---|
| `proposal_proposed` | `(proposal_id: u32, proposer: Address, action: Symbol, data: T)` | When an admin proposes an action (e.g., slash, upgrade, change settings). | | `propose_deactivate_requires_admin` |
| `proposal_approved` | `(proposal_id: u32, admin: Address, approvals_len: u32)` | When an admin approves an existing proposal. | | `approve_proposal_records_approval_and_returns_total` |
| `proposal_ready` | `(proposal_id: u32, ready_at: u64)` | When a proposal receives enough approvals and enters the timelock. | | `proposal_enters_timelock_after_sufficient_approvals` |
| `proposal_executed` | `(proposal_id: u32, executed_at: u64)` | When a ready proposal is executed after the timelock elapses. | | `execute_proposal_applies_action` |
| `contract_deactivated` | `(contract_id: Address, caller: Address)` or `(contract_id: Address, Symbol("governance"))` | When a contract is deactivated by its owner or governance. | Indexer, History | `deactivate_requires_contract_owner` |
| `contract_deregistered` | `(contract_id: Address, owner: Address)` | When a deactivated and unstaked contract is fully deregistered. | Indexer, History | `deregister_removes_every_index_reference...` |
| `contract_registered` | `(contract_id: Address, owner: Address, name: String, categories: Vec<String>)``| When a new contract is registered to the manifest. | Indexer, History | `registration_records_its_categories` |
| `categories_updated` | `(contract_id: Address, owner: Address, categories: Vec<String>)` | When a contract's categories are updated by its owner. | Indexer, History | `set_categories_moves_a_registration_between_categories` |
| `tags_updated` | `(contract_id: Address, owner: Address, tags_len: u32)` | When a contract's tags are updated by its owner. | History | `tags_are_updated_and_returned` |
| `metadata_updated` | `(contract_id: Address, owner: Address, name: String)` | When the contract's metadata (name) is updated. | Indexer, History | `update_metadata_succeeds_with_real_owner_signature` |
| `ownership_transferred` | `(contract_id: Address, previous_owner: Address, new_owner: Address)` | When the contract's ownership is transferred to a new address. | History | `ownership_transfer_preserves_stake_and_verification` |
| `stake_deposited` | `(contract_id: Address, owner: Address, amount: i128, total_staked: i128)` | When the owner deposits tokens to top up their stake. | History | `stake_tops_up_an_existing_stake` |
| `stake_withdrawn` | `(contract_id: Address, owner: Address, total_staked: i128)` | When the owner withdraws their staked tokens after deactivation. | History | `withdraw_returns_the_full_stake_once_the_owner_has_deactivated` |
| `stake_slashed` | `(contract_id: Address, amount: i128, reason: String, treasury: Address, treasury_amount: i128, reward_pool_amount: i128)` | When governance slashes a contract's stake for a violation. `amount` is the total slashed; `treasury_amount` and `reward_pool_amount` split it by the configured proportion. | History | `slash_splits_stake_between_treasury_and_reward_pool` |
| `staker_reward_claimed` | `(staker: Address, amount: i128, pool_remaining: i128)` | When a staker claims their share of the slash reward pool. | History | `staker_can_claim_their_share_of_a_slash` |
| `slash_split_changed` | `(treasury_bps: u32, reward_pool_bps: u32)` | When governance sets the split of slashed stake between the treasury and the staker reward pool. | History | `slash_split_can_be_changed_by_the_governance` |
| `verification_set` | `(contract_id: Address, verified: bool)` | When governance grants or revokes verified status for a contract. | History | `governance_can_attest_and_later_revoke_verification` |
| `category_pruned` | `(category: String, removed: u32)` | When dead references in a category's index are cleaned up. | | `prune_category_drops_dead_references_and_is_safe_to_repeat` |
| all_contracts_pruned` | `(removed: u32,)` | When dead references in the global index are cleaned up. | | `contract_count_is_live_and_total_registered_is_lifetime` |
| `registry_upgraded` | `(new_wasm_hash: BytesN<32>, version: u32)` | When the registry contract's WASM is upgraded. | | `upgrade_carries_admin_across_swap` |
| `admin_added` | `(new_admin: Address,)` | When a new governance admin is added via executed proposal. | | `propose_add_admin_adds_a_new_admin` |
| `admin_removed` | `(admin_to_remove: Address,)` | When a governance admin is removed via executed proposal. | | `propose_remove_admin_removes_the_admin` |
| `threshold_changed` | `(new_threshold: u32,)` | When the multisig approval threshold is changed. | | `propose_change_threshold_changes_the_threshold` |
| `staking_configured` | `(token_id: Address, treasury: Address)` | When governance configures the staking token and treasury. | | `configure_staking_records_token_and_treasury` |
| `allowlist_mode_changed` | `(enabled: bool,)` | When governance enables or disables the owner allowlist. | | `allowlist_mode_can_be_toggled` |
| `owner_allowlisted` | `(owner: Address, allowed: bool)` | When governance adds or removes an owner from the allowlist. | | `owner_can_be_added_to_allowlist` |
| `registration_rate_limit_changed` | `(limit: u32, window: u32)` | When governance updates the rate limit parameters. | | `registration_rate_limit_can_be_changed` |
| `registration_fee_set` | `(fee: i128,)` | When governance sets a flat fee for new registrations. | | `registration_fee_can_be_set` |

## Slash Distribution Economics

When governance slashes a registration's stake, the slashed amount is split by a governance-set proportion between the **treasury** and a **staker reward pool**. The split is expressed in basis points (`10000` = 100%). A `treasury_bps` of `10000` means the entire slash goes to the treasury, which is the behaviour before this feature existed.

### Why a claim path and not a push

The reward pool is distributed by **claim**, not by push. The reason is cost scaling:

- A **push** distribution would have to iterate every honest staker and transfer to each one inside the slash transaction. The cost of the slash would then grow with the number of stakers, and a sufficiently large staker set would make the slash unexecutable because it exceeds the transaction's resource budget. The governance action that levies the penalty would be blocked by the very people it is meant to compensate.
- A **claim** distribution moves the cost of distribution onto each staker, who pays for their own transfer at the time they choose to claim. The slash itself is a constant-cost operation regardless of how many stakers are eligible, and a staker who never claims never costs anyone anything.

### How the share is computed

Each staker's share is proportional to their stake at the time of the slash, relative to the total stake on other registrations. The contract tracks the reward pool balance and each staker's claimable amount so that a claim is a single constant-cost transfer. Claiming is idempotent in the sense that a staker who has already claimed their share cannot claim it again.

The configured split is readable off-chain via `get_slash_split_bps`, and a contract that wants to know what a staker can claim without spending a transaction can read `get_claimable_reward`.
