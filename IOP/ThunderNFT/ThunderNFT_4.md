# Introduction

Before and during the Fuel Attackathon, Immunefi announced an exciting opportunity: anyone who found at least one valid bug during the contest would qualify to participate in four exclusive IOPs (Invite-Only Programs). One of these was ThunderNFT, the first NFT marketplace built on the Fuel blockchain.

I spent 17 days working on the ThunderNFT codebase and was proud to secure second place on the leaderboard. My findings included 3 high bugs and 2 medium bugs, earning a reward of $10.5k, I came close to winning the contest but missed out on first place due to a invalid critical bug submitted and accepted.

This report shares the story behind one of the high severity bug I discovered in the ThunderNFT codebase.

# Disclaimer

The details shared in this report are for informational purposes only and are not intended to encourage or discourage users or investors from engaging with the mentioned bug bounty program. This report highlights a vulnerability I discovered while reviewing the specified protocol during a set period, offering insights from my personal experience. Please conduct your own research and due diligence before investing in or working on mentioned bounty/contest.

# About **ThunderNFT**

Thunder — is an NFT marketplace with superior experience built on the Fuel Network, What is notable about this platform is that it supports bulk execution. This unique feature will allow you to perform a series of operations in one go streamlining the process of managing and trading NFTs, for more information read this [blog](https://medium.com/@ThunderbyFuel/thunder-first-ever-nft-marketplace-on-fuel-88629c812d2b) or visit their [app](https://thundernft.market/marketplace)

# Report Stats

| Field              | Details                                                                          |
| ------------------ | -------------------------------------------------------------------------------- |
| Attackathon name   | [ThunderNFT](https://immunefi.com/audit-competition/thundernft-iop/leaderboard/) |
| Report state       | PAID                                                                             |
| Report Severity    | HIGH                                                                             |
| Report status      | duplicate Finding                                                                |
| total bugs found   | 3 High(1 cheif finder), 2 Medium(2 solo), 3 low/insight                          |
| amount payed       | $1k                                                                              |
| protocol team rate | 5/5 respectful team, quick response and validating reports                       |

# Vulnerability Details

## [HIGH] Permanent freezing of seller NFTS can occur when the strategy whitelist address get updated

### Description

The `cancel_order` function in ThunderNFT Exchange allows users with active orders to cancel them and retrieve their NFTs. However, this process may not work as expected if the strategy whitelist address is updated whether due to a new strategy being added or issues with the previous one. This update triggers a check that prevents using strategies removed from the whitelist. As a result, the seller’s NFT becomes stuck temporary, as it can no longer be accessed or sold. Additionally, since the `execute_order` function relies on the strategy, the NFT cannot be purchased or accept any offers, effectively freezing the asset.

### in depth analysis

When the seller wants to cancel their order, the following function should be invoked, which ensures the NFT is transferred back to the owner:

```sway
/// Cancels MakerOrder
    #[storage(read)]
    fn cancel_order(
        strategy: ContractId,
        nonce: u64,
        side: Side
    ) {
        let caller = get_msg_sender_address_or_panic();

        let execution_manager_addr = storage.execution_manager.read().unwrap().bits();
        let execution_manager = abi(ExecutionManager, execution_manager_addr);

        require(strategy != ZERO_CONTRACT_ID, ThunderExchangeErrors::StrategyMustBeNonZeroContract);
        require(execution_manager.is_strategy_whitelisted(strategy), ThunderExchangeErrors::StrategyNotWhitelisted); //@audit

        let strategy_caller = abi(ExecutionStrategy, strategy.bits());
        let order = strategy_caller.get_maker_order_of_user(caller, nonce, side); // get the order for the caller

        match side {
            Side::Buy => {
                // Cancels buy MakerOrder (e.g. offer)
                strategy_caller.cancel_order(caller, nonce, side);
            },
            Side::Sell => {
                // Cancel sell MakerOrder (e.g. listing)
                if (order.is_some()) {
                    // If order is valid, then transfers the asset back to the user
                    let unwrapped_order = order.unwrap();
                    strategy_caller.cancel_order(caller, nonce, side);
                    transfer(
                        Identity::Address(unwrapped_order.maker),
                        AssetId::new(unwrapped_order.collection, unwrapped_order.token_id),
                        unwrapped_order.amount
                    );
                }
            },
        }

        log(OrderCanceled {
            user: caller,
            strategy,
            side,
            nonce,
        });
    }

```

If the strategy address is updated, it becomes impossible for users to transfer back their NFTs. This issue arises because the codebase lacks upgradability requirements, which are necessary for handling updates to the strategy. Even if the seller had initially set the whitelisted strategy, they would be unable to retrieve their NFT under the new strategy address, as there would be no valid order in the new strategy.

When combining this with the previous bug report i found, it becomes clear that the seller will be unable to cancel or execute their order. Moreover, updating the sell order will not result in the NFT being sent back to the seller. This creates a situation where the seller’s NFTs are effectively locked, and no action can be taken to retrieve them due to the lack of support for strategy address changes in the contract.

# Final word

I hope you enjoyed reading this report. I’m truly grateful for the opportunity to work on the first ever NFT marketplace on the **Fuel blockchain**. The codebase was challenging and battle-tested, but I did my best to secure it as much as possible by finding 8 H/M and L/I bugs.

A special thanks to the ThunderNFT protocol team and the Immunefi team for their support and for providing this incredible opportunity. I’ll be sharing more private and contest audits for Fuel dApps on my GitHub portfolio soon, stay tuned!
