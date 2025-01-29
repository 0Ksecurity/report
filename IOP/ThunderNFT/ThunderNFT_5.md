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
| Report Severity    | MEDIUM                                                                           |
| Report status      | cheif Finding                                                                    |
| total bugs found   | 3 High(1 cheif finder), 2 Medium(2 solo), 3 low/insight                          |
| amount payed       | $1.4k                                                                            |
| protocol team rate | 5/5 respectful team, quick response and validating reports                       |

# Vulnerability Details

## [MEDIUM] users with current bid order can not update their order when payment token changed

### Description

The `update_order` function is designed to allow users with active bids to update their orders with important input changes. However, there is a check that prevents users from updating their bids, forcing them to cancel their bids instead. This leads to a loss of gas for buy order users. The issue arises because the update_order function calls **update_order** in the fixed strategy, which in turn calls `_update_buy_order`. This check evaluates whether the payment asset remains unchanged. The payment asset is the same asset that was set in the whitelist when the assetManager calls `add_asset` or `remove_asset`.

### in depth analysis

the function update_order implemented as below:

```sway
 #[storage(read), payable]
    fn update_order(order_input: MakerOrderInput) {
        _validate_maker_order_input(order_input);

        let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
        let order = MakerOrder::new(order_input);
        match order.side {
            Side::Buy => {
                // Checks if user has enough bid balance
                let pool_balance = _get_pool_balance(order.maker, order.payment_asset);
                require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance);
            },
            Side::Sell => {}, // if order is selling nft then nothing to check for
        }

        strategy.update_order(order);

        log(OrderUpdated {
            order
        });

    }

```

which invoke the update_order function for both buy and sell order:

```sway
 /// Updates the existing MakerOrder of the user
    /// Only callable by Thunder Exchange contract
    #[storage(read, write)]
    fn update_order(order: MakerOrder) {
        // only_exchange();

        match order.side {
            Side::Buy => {
                _update_buy_order(order)
            },
            Side::Sell => {
                _update_sell_order(order)
            }
        }
    }

```

both \_update_buy_order and \_update_sell_order invoke the \_validate_updated_order function as shown below:

```sway
/// Updates buy MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_buy_order(order: MakerOrder) {
    let nonce = _user_buy_order_nonce(order.maker); // nonce should not be changed for update purpose same to maker address

    if ((order.nonce <= nonce)) {
        // Update buy order
        let buy_order = _buy_order(order.maker, order.nonce); // get the old order
        _validate_updated_order(buy_order, order);
        storage.buy_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(114);
    }
}

/// Updates sell MakerOrder if the nonce is in the right range
#[storage(read, write)]
fn _update_sell_order(order: MakerOrder) {
    let nonce = _user_sell_order_nonce(order.maker);
    let min_nonce = _user_min_sell_order_nonce(order.maker);

    if ((min_nonce < order.nonce) && (order.nonce <= nonce)) {
        // Update sell order
        let sell_order = _sell_order(order.maker, order.nonce);
        _validate_updated_order(sell_order, order);
        storage.sell_order.insert((order.maker, order.nonce), Option::Some(order));
    } else {
        revert(115);
    }
}

#[storage(read)]
fn _validate_updated_order(order: Option<MakerOrder>, updated_order: MakerOrder) {
    require(
        (order.unwrap().maker == updated_order.maker) &&
        (order.unwrap().collection == updated_order.collection) &&
        (order.unwrap().token_id == updated_order.token_id) &&
        (order.unwrap().payment_asset == updated_order.payment_asset) && //@audit payment asset should be allowed to be changed
        _is_valid_order(order),
        StrategyFixedPriceErrors::OrderMismatchedToUpdate
    );
}

```

The highlighted lines in the code snippet above demonstrate how the checks prevent buyers from updating their bids. However, the scenario becomes even more problematic for sellers. Here’s a detailed breakdown of how the issue affect the sellers:

- Alice creates a sell order (lists an NFT) by calling place_order with the payment asset set to USDT.

- 10 users create buy orders to bid on Alice’s NFT, with their payment asset set to USDT.

- For some reason, USDT is removed from the whitelist by calling assetManager.sw#remove_asset, and ETH is added as the new payment asset by calling the add_asset function.

Now, when trying to update the order, the check **rder.unwrap().payment_asset == updated_order.payment_asset** is triggered. This check compares the old payment asset with the new one and is invoked by `_validate_updated_order`, which is part of both `_update_buy_order` and `_update_sell_orderAs` a result, both Alice and the 10 users are forced to cancel their existing orders and recreate them, leading to a loss of gas for all parties involved. This problem is especially critical for sell order users like Alice, as they are unable to update their order without incurring additional transaction costs.

The problem is particularly critical for sell order users, who are unable to update their orders without facing extra costs. It also significantly impacts users who need to cancel and recreate their orders in order to proceed with the updated payment asset, causing frustration and financial loss.

# Final word

I hope you enjoyed reading this report. I’m truly grateful for the opportunity to work on the first ever NFT marketplace on the **Fuel blockchain**. The codebase was challenging and battle-tested, but I did my best to secure it as much as possible by finding 8 H/M and L/I bugs.

A special thanks to the ThunderNFT protocol team and the Immunefi team for their support and for providing this incredible opportunity. I’ll be sharing more private and contest audits for Fuel dApps on my GitHub portfolio soon, stay tuned!
