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
| Report status      | Chief Finding                                                                    |
| total bugs found   | 3 High(1 cheif finder), 2 Medium(2 solo), 3 low/insight                          |
| amount payed       | $4.2k                                                                            |
| protocol team rate | 5/5 respectful team, quick response and validating reports                       |

# Vulnerability Details

## [HIGH] users can't call update_order to update the strategy which prevent the NFT to be canceled or executed

### Description

The `update_order` function in the thunder_exchange contract is intended to allow order makers to modify critical parameters, such as price, amount, strategy, and paymentAsset. This functionality is particularly important when new **tokens or strategies** are added to the whitelist. However, due to the implementation in the thunder_exchange and fixed_strategy contracts, users are unable to update the strategy address properly.

the issue arises When the `thunder_exchange#update_order` function is called, it invokes the `_validate_maker_order_input` function, which enforces that the order nonce must be greater than **zero**. Next, the `fixed_strategy#update_order` function retrieves the user’s order nonce using the new strategy address. However, the logic fails because the user’s storage data in the new strategy contract is reset to zero. This happens because the system is non-upgradable, meaning that storage data from the previous strategy contract is not carried over to the new one. Here’s the problematic code from the fixed_strategy#update_order function:

```sway

/// Updates sell MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_sell_order(order: MakerOrder) {
    let nonce = _user_sell_order_nonce(order.maker); //@audit get the nonce for the maker, this will be zero in new strategy contract since user have no storage data in the new deployed contract
    let min_nonce = _user_min_sell_order_nonce(order.maker);

    if ((min_nonce < order.nonce) && (order.nonce <= nonce)) { //@audit order.nonce is 1 and nonce is zero so this won't execute and revert happened
        // Update sell order
        let sell_order = _sell_order(order.maker, order.nonce);
        _validate_updated_order(sell_order, order);
        storage.sell_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(115);
    }
}
```

In simpler terms, the core issues are:

- When users attempt to update their order, they must use a nonce greater than zero (as place_order does not permit orders with a nonce of zero).

- Since the system does not support upgradability, storage data from the old strategy is not retained or migrated to the new strategy.

- The condition `(min_nonce < order.nonce) && (order.nonce <= nonce)` fails because the nonce in the new strategy contract is zero, while the order.nonce is set to 1 or higher. This results in a revert.

This design flaw prevents users from updating critical parameters such as price, nonce, paymentAsset, or strategy. As a result, the NFTs associated with the order are effectively “stuck” and cannot be updated, canceled, or executed.

### in depth analysis

When a user attempts to update their order, the first step is to call the update_order function in the `thunder_exchange` contract. This function performs several sanity checks to ensure the validity of the update. Here’s an example of how these checks might look:

```sway

 /// Updates the existing MakerOrder
    #[storage(read), payable]
    fn update_order(order_input: MakerOrderInput) {
        _validate_maker_order_input(order_input);

        let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
        let order = MakerOrder::new(order_input);
        match order.side {
            Side::Buy => {
                // Checks if user has enough bid balance
                let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
                require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance); // the price bidder set should be smaller or equal to their balance
            },
            Side::Sell => {}, // if order is selling nft then nothing to check for
        }

        strategy.update_order(order); //@audit call update_order in the new strategy

        log(OrderUpdated {
            order
        });
    }


/// Validates the maker order input
#[storage(read)]
fn _validate_maker_order_input(input: MakerOrderInput) {
    require(input.maker != ZERO_ADDRESS, ThunderExchangeErrors::MakerMustBeNonZeroAddress);
    require(input.maker == get_msg_sender_address_or_panic(), ThunderExchangeErrors::CallerMustBeMaker);

    // !!! Info !!! -> This check will be removed as mentioned
    require(
        (storage.min_expiration.read() <= input.expiration_range) &&
        (input.expiration_range <= storage.max_expiration.read()),
        ThunderExchangeErrors::ExpirationRangeOutOfBound
    );

    require(input.nonce > 0, ThunderExchangeErrors::NonceMustBeNonZero); //@audit nonce should be above zero
    require(input.price > 0, ThunderExchangeErrors::PriceMustBeNonZero);
    require(input.amount > 0, ThunderExchangeErrors::AmountMustBeNonZero); // no matter how much you set this will be 1

    // Checks if the strategy contracId in the order is whitelisted
    let execution_manager_addr = storage.execution_manager.read().unwrap().bits();
    let execution_manager = abi(ExecutionManager, execution_manager_addr);
    require(execution_manager.is_strategy_whitelisted(input.strategy), ThunderExchangeErrors::StrategyNotWhitelisted); // used strategy should be whitelisted

    // Checks if the payment_asset in the order is supported
    let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
    let asset_manager = abi(AssetManager, asset_manager_addr);
    require(asset_manager.is_asset_supported(input.payment_asset), ThunderExchangeErrors::AssetNotSupported);
}

```

As shown above, the nonce input must be greater than zero. This creates a problem when a new strategy is deployed and added to the whitelist, as the old storage data from the previous strategy is not transferred to the new one. The issue can be found in the following lines:

```sway
/// Updates sell MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_sell_order(order: MakerOrder) {
    let nonce = _user_sell_order_nonce(order.maker); //@audit get the nonce for the maker, this will be zero in new strategy contract since user have no storage data in the new deployed contract
    let min_nonce = _user_min_sell_order_nonce(order.maker);

    if ((min_nonce < order.nonce) && (order.nonce <= nonce)) { // @audit this check won't pass since order.nonce is 1 and nonce is zero
        // Update sell order
        let sell_order = _sell_order(order.maker, order.nonce);
        _validate_updated_order(sell_order, order);
        storage.sell_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(115);
    }
}


#[storage(read)]
fn _user_sell_order_nonce(address: Address) -> u64 {
    let status = storage.user_sell_order_nonce.get(address).try_read(); //@audit this will be zero since there is no data for maker in the new strategy that deployed
    match status {
        Option::Some(nonce) => nonce,
        Option::None => 0, //@audit zero returned
    }
}

```

As shown, the call to update_order will revert, preventing other functions from executing because the strategy or paymentAsset cannot be updated.
This issue primarily affects the NFT seller, as they are unable to cancel or execute their NFT sell order. However, bidders remain unaffected, as they can still transfer their tokens out of the pool. this issue lead to temporary freezing of NFTS

# Final word

I hope you enjoyed reading this report. I’m truly grateful for the opportunity to work on the first ever NFT marketplace on the **Fuel blockchain**. The codebase was challenging and battle-tested, but I did my best to secure it as much as possible by finding 8 H/M and L/I bugs.

A special thanks to the ThunderNFT protocol team and the Immunefi team for their support and for providing this incredible opportunity. I’ll be sharing more private and contest audits for Fuel dApps on my GitHub portfolio soon, stay tuned!
