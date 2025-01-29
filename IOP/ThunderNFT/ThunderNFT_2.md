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
| Report status      | Chief Finding                                                                    |
| total bugs found   | 3 High(1 cheif finder), 2 Medium(2 solo), 3 low/insight                          |
| amount payed       | $1.4k                                                                            |
| protocol team rate | 5/5 respectful team, quick response and validating reports                       |

# Vulnerability Details

## [MEDIUM] owner of NFT who have sell order(listing NFT) can not accept any bid offers.

### Description

The execute_order function is intended to allow users to either buy an NFT directly by paying the listed price or for sellers to accept specific bid offers. However, there’s an issue when a seller attempts to accept a bid. The core issue lies in the check within the `_execute_sell_taker_order` function, which validates the asset being sold via `msg_asset()`. This check ensures that the seller is interacting with the correct asset when executing an order.

The problem occurs because, during the `place_order` function, the seller is required to transfer the NFT to the Thunder Exchange contract before they can list it. This means the seller no longer holds the NFT once it’s listed on the marketplace, which makes it impossible for them to provide the valid **asset_id** when trying to accept a bid through the `execute_order` function. As a result, when the seller attempts to accept a bid, the validation in `_execute_sell_taker_order` fails because the seller doesn’t have direct access to the NFT. Since the execute_order function depends on the seller holding the asset to process the bid, they are unable to complete the transaction, effectively preventing the acceptance of bids. This design flaw creates a situation where the seller cannot accept any bids.

### in depth analysis

he `place_order` function is used when the owner of an NFT wants to list it for sale on the Thunder Exchange platform. When a seller wants to list their NFT, the place_order function is called with **Side::Sell** , signaling their intent to sell the NFT:

```sway
/// Places MakerOrder by calling the strategy contract
    /// Checks if the order is valid
    #[storage(read), payable] // @audit when user set sell order(listing) the function force the user to sent its NFT in checks below
    fn place_order(order_input: MakerOrderInput) {
        _validate_maker_order_input(order_input); // sanity checks

        let strategy = abi(ExecutionStrategy, order_input.strategy.bits());
        let order = MakerOrder::new(order_input);

        match order.side {
            Side::Buy => { //users make offer for specific nft(bid)
                // Buy MakerOrder (e.g. make offer)
                // Checks if user has enough bid balance
                let pool_balance = _get_pool_balance(order.maker, order.payment_asset); // get the maker balance of the payment asset
                require(order.price <= pool_balance, ThunderExchangeErrors::AmountHigherThanPoolBalance); // example: price is 5 eth and user have 6 eth
            },
            Side::Sell => {
                // Sell MakerOrder (e.g. listing)
                // Checks if assetId and amount mathces with the order
                //@audit forced to send the NFT
                require(msg_asset_id() == AssetId::new(order.collection, order.token_id), ThunderExchangeErrors::AssetIdNotMatched);
                require(msg_amount() == order_input.amount, ThunderExchangeErrors::AmountNotMatched);
            }, //transfer the NFT to this contract
        }

        strategy.place_order(order); // call fixed strategy

        log(OrderPlaced {
            order
        });
    }
```

The core issue here lies in the fact that the place_order function requires the seller to transfer the NFT to the Thunder Exchange contract before the listing is made. This means the seller no longer holds the NFT once it’s listed. However, for the `execute_order` function (used for accepting bids), the contract expects the seller to hold the asset. This creates a conflict, as the NFT is no longer in the seller’s possession, and they cannot provide a valid msg_asset_id() for accepting bids

```sway
/// Executes order by either
    /// filling the sell MakerOrder (e.g. purchasing NFT)
    /// or the buy MakerOrder (e.g. accepting an offer)
    #[storage(read), payable]
    fn execute_order(order: TakerOrder) {
        _validate_taker_order(order);

        match order.side {
            Side::Buy => _execute_buy_taker_order(order), // buy the NFT directly by buyer
            Side::Sell => _execute_sell_taker_order(order), // accept bid by seller
        }

        log(OrderExecuted {
            order
        });
    }



#[storage(read), payable] //@audit seller is forced to send the NFT which doesn't exist for the user instead its in the thunder exchange itself when place_order called
fn _execute_sell_taker_order(order: TakerOrder) {
    let strategy = abi(ExecutionStrategy, order.strategy.bits());
    let execution_result = strategy.execute_order(order);
    require(execution_result.is_executable, ThunderExchangeErrors::ExecutionInvalid);
    require(
        msg_asset_id() == AssetId::new(execution_result.collection, execution_result.token_id), //@audit
        ThunderExchangeErrors::PaymentAssetMismatched
    ); // but the seller does not have the NFT ?
    require(msg_amount() == execution_result.amount, ThunderExchangeErrors::AmountMismatched);

    // Transfer the NFT
    transfer(
        Identity::Address(order.maker),
        AssetId::new(execution_result.collection, execution_result.token_id),
        execution_result.amount
    );

    // Deduct the fees from bid balance
    _transfer_fees_and_funds_with_pool(
        order.strategy,
        execution_result.collection,
        order.maker,
        order.taker,
        order.price,
        execution_result.payment_asset,
    );
}

```

The call to execute_order will fail due to a restriction caused by the transfer process. When the seller lists an NFT using the place_order function, they are required to transfer the NFT to the Thunder Exchange contract. This transfer is governed by the built in transfer function in Sway, which includes crucial checks enforced by the FuelVM. One of these checks ensures that the caller (in this case, the seller) holds the specified `asset_id` and the correct amount of the asset. Since the seller no longer hold the NFT after transferring it to the Thunder Exchange contract, the necessary conditions are no longer met when attempting to execute the order. As a result, the transaction reverts.

This design limitation makes it impossible for the seller to accept bids. Instead, the seller has two options: they can either wait for a direct purchase by a buyer or cancel the order to regain control of the NFT.

# Final word

I hope you enjoyed reading this report. I’m truly grateful for the opportunity to work on the first ever NFT marketplace on the **Fuel blockchain**. The codebase was challenging and battle-tested, but I did my best to secure it as much as possible by finding 8 H/M and L/I bugs.

A special thanks to the ThunderNFT protocol team and the Immunefi team for their support and for providing this incredible opportunity. I’ll be sharing more private and contest audits for Fuel dApps on my GitHub portfolio soon, stay tuned!
