# Registry Events Reference

Events are the integration surface for downstream consumers, serving as the interface for both `lumina-backend`'s indexer and `lumina-frontend`'s registry history view.

## Downstream Consumers

- **Registry History (`lumina-frontend`)**: The frontend's history view rebuilds the per-contract event timeline by matching on the **first topic** (which must be the event name) and treating the **first data slot** as the subject ID (`contract_id`). Any events matching this shape will be attributed to the respective contract's history. Unknown topics will still be displayed as generic "Registry event" rows.
- **Indexer (`lumina-backend`)**: The backend indexer discovers contracts and listens to registry events to keep its database synchronized with the on-chain manifest. It specifically looks for registration, deactivation, and metadata/category changes to maintain an up-to-date registry graph.

## Events

| Topic | Payload Shape | When it Fires | Consumers | Example Test Reference |
|---|---|---|---|---|
| `proposal_proposed` | `(proposal_id: u32, proposer: Address, action: Symbol, data: T)` | When an admin proposes an action (e.g., slash, upgrade, change settings). | | `propose_deactivate_requires_admin` |
| `proposal_approved` | `(proposal_id: u32, admin: Address, approvals_len: u32)` | When an admin approves an existing proposal. | | `approve_proposal_records_approval_and_returns_total` |
| `proposal_ready` | `(proposal_id: u32, ready_at: u64)` | When a proposal receives enough approvals and enters the timelock. | | `proposal_enters_timelock_after_sufficient_approvals` |
| `proposal_executed` | `(proposal_id: u32, executed_at: u64)` | When a ready proposal is executed after the timelock elapses. | | `execute_proposal_applies_action` |
| `contract_deactivated` | `(contract_id: Address, caller: Address)` or `(contract_id: Address, Symbol("governance"))` | When a contract is deactivated by its owner or governance. | Indexer, History | `deactivate_requires_contract_owner` |
| `contract_deregistered` | `(contract_id: Address, owner: Address)` | When a deactivated and unstaked contract is fully deregistered. | Indexer, History | `deregister_removes_every_index_reference...` |
| `contract_registered` | `(contract_id: Address, owner: Address, name: String, categories: Vec<String>)` | When a new contract is registered to the manifest. | Indexer, History | `registration_records_its_categories` |
| `categories_updated` | `(contract_id: Address, owner: Address, categories: Vec<String>)` | When a contract's categories are updated by its owner. | Indexer, History | `set_categories_moves_a_registration_between_categories` |
| `tags_updated` | `(contract_id: Address, owner: Address, tags_len: u32)` | When a contract's tags are updated by its owner. | History | `tags_are_updated_and_returned` |
| `metadata_updated` | `(contract_id: Address, owner: Address, name: String)` | When the contract's metadata (name) is updated. | Indexer, History | `update_metadata_succeeds_with_real_owner_signature` |
| `ownership_transferred`| `(contract_id: Address, previous_owner: Address, new_owner: Address)` | When the contract's ownership is transferred to a new address. | History | `ownership_transfer_preserves_stake_and_verification` |
| `stake_deposited` | `(contract_id: Address, owner: Address, amount: i128, total_staked: i128)` | When the owner deposits tokens to top up their stake. | History | `stake_tops_up_an_existing_stake` |
| `stake_withdrawn` | `(contract_id: Address, owner: Address, total_staked: i128)` | When the owner withdraws their staked tokens after deactivation. | History | `withdraw_returns_the_full_stake_once_the_owner_has_deactivated` |
| `stake_slashed` | `(contract_id: Address, amount: i128, reason: String, treasury: Address)` | When governance slashes a contract's stake for a violation. The `amount` is split between the treasury and the staker reward pool by the configured proportion. | History | `slash_moves_stake_to_the_treasury_and_records_the_reason` |
| `verification_set` | `(contract_id: Address, verified: bool)` | When governance grants or revokes verified status for a contract. | History | `governance_can_attest_and_later_revoke_verification` |
| `category_pruned` | `(category: String, removed: u32)` | When dead references in a category's index are cleaned up. | | `prune_category_drops_dead_references_and_is_safe_to_repeat` |
| `all_contracts_pruned` | `(removed: u32,)` | When dead references in the global index are cleaned up. | | `contract_count_is_live_and_total_registered_is_lifetime` |
| `registry_upgraded` | `(new_wasm_hash: BytesN<32>, version: u32)` | When the registry contract's WASM is upgraded. | | `upgrade_carries_admin_across_swap` |
| `admin_added` | `(new_admin: Address,)` | When a new governance admin is added via executed proposal. | | `propose_add_admin_adds_a_new_admin` |
| `admin_removed` | `(admin_to_remove: Address,)` | When a governance admin is removed via executed proposal. | | `propose_remove_admin_removes_the_admin` |
| `threshold_changed` | `(new_threshold: u32,)` | When the multisig approval threshold is changed. | | `propose_change_threshold_changes_the_threshold` |
| `staking_configured` | `(token_id: Address, treasury: Address)` | When governance configures the staking token and treasury. | | `configure_staking_records_token_and_treasury` |
| `slash_split_changed` | `(treasury_bps: u32,)` | When governance updates the proportion of a slash routed to the treasury (remainder accrues to the staker reward pool). | | `slash_split_can_be_changed` |
| `staker_reward_claimed` | `(staker: Address, amount: i128)` | When a staker claims their accrued share of slashed stake from the reward pool. | History | `staker_can_claim_their_share_of_a_slash` |
| `allowlist_mode_changed`| `(enabled: bool,)` | When governance enables or disables the owner allowlist. | | `allowlist_mode_can_be_toggled` |
| `owner_allowlisted` | `(owner: Address, allowed: bool)` | When governance adds or removes an owner from the allowlist. | | `owner_can_be_added_to_allowlist` |
| `registration_rate_limit_changed` | `(limit: u32, window: u32)` | When governance updates the rate limit parameters. | | `registration_rate_limit_can_be_changed` |
| `registration_fee_set` | `(fee: i128,)` | When governance sets a flat fee for new registrations. | | `registration_fee_can_be_set` |

## Slash Distribution Economics

A slash is split between the treasury and a staker reward pool according to a governance-set proportion (`treasury_bps`, in basis points). The remainder (`10_000 - treasury_bps`) accrues to the reward pool and is shared among honest stakers on other registrations. With `treasury_bps = 10_000`, the entire slash goes to the treasury, matching the pre-change behaviour.

Distribution is **pull-based**: the slash credits the reward pool and records each eligible staker's accrued share, and stakers later call a claim entry point to withdraw. A push-based distribution is not viable here because the cost of a slash would scale with the number of stakers: the slashing transaction would have to iterate over every honest staker and transfer to each, making the gas cost unbounded and the slash itself griefable (a large staker set could make slashing prohibitively expensive or exceed the transaction resource limits). A claim path keeps the slash cost O(1) in the staker count and shifts the per-staker transfer cost to the staker who benefits, so distribution cost does not scale with staker count on the governance path.

The split is what makes staking a judgement rather than a lottery: honest stakers who back registrations against a bad actor recover a portion of the slashed stake, so the expected value of staking against a violator is not a pure loss to the treasury.

### Acceptance Criteria

- A slash splits by the configured proportion between the treasury and the staker reward pool.
- A staker can claim their share of the reward pool via the claim path.
- With a 100% treasury split (`treasury_bps = 10_000`), behaviour matches today: the full slash goes to the treasury and no reward pool accrues.
