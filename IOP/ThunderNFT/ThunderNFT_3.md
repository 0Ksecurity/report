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
| amount payed       | $699                                                                             |
| protocol team rate | 5/5 respectful team, quick response and validating reports                       |

# Vulnerability Details

## [HIGH] users can't withdraw their tokens when specific asset removed from the whitelist

### Description

The `pool.sw` contract allows users to deposit and withdraw whitelisted tokens by calling the deposit and withdraw functions. However, there is a critical issue with the withdrawal process. When a user attempts to withdraw tokens, the contract verifies whether the token is still whitelisted. This introduces a risk where, if the **assetManager** contract decides to remove a specific token from the whitelist, users will be unable to withdraw their funds, effectively locking their tokens in the contract temporary.

### in depth analysis

The deposit function in the pool.sw contract allows users to deposit only whitelisted tokens:

```sway

 /// Deposits the supported asset into this contract
    /// and assign the deposited amount to the depositer as bid balance
    #[storage(read, write), payable]
    fn deposit() {
        let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
        let asset_manager = abi(AssetManager, asset_manager_addr);
        require(asset_manager.is_asset_supported(msg_asset_id()), PoolErrors::AssetNotSupported); //@audit only supported

        let address = msg_sender().unwrap();
        let amount = msg_amount();
        let asset = msg_asset_id();

        let current_balance = _balance_of(address, asset);
        let new_balance = current_balance + amount;
        storage.balance_of.insert((address, asset), new_balance);

        log(Deposit {
            address,
            asset,
            amount
        });
    }

```

The function first verifies whether the asset is supported (whitelisted) by querying the asset_manager contract. If the asset is not supported, the transaction will revert. Once the asset is valid, the function proceeds by retrieving the amount the user intends to deposit, updates the user’s balance accordingly, and logs the transaction. However, there are certain checks in the withdraw function that could lead to users’ funds being temporarily locked, as demonstrated in the following code snippet:

```sway

    /// Withdraws the amount of assetId from the contract
    /// and sends to sender if sender has enough balance
    #[storage(read, write)]
    fn withdraw(asset: AssetId, amount: u64) {
        let sender = msg_sender().unwrap();
        let current_balance = _balance_of(sender, asset);
        require(current_balance >= amount, PoolErrors::AmountHigherThanBalance);

        let asset_manager_addr = storage.asset_manager.read().unwrap().bits();
        let asset_manager = abi(AssetManager, asset_manager_addr);
        require(asset_manager.is_asset_supported(asset), PoolErrors::AssetNotSupported); //@audit prevent if asset removed

        let new_balance = current_balance - amount;
        storage.balance_of.insert((sender, asset), new_balance);

        transfer(sender, asset, amount);

        log(Withdrawal {
            address: sender,
            asset,
            amount,
        });
    }

```

This issue can lead to funds being locked because assets can be added or removed from the whitelist using the `add/remove` asset functions, as shown below:

```sway
    /// Adds asset into supported assets vec
    #[storage(read, write)]
    fn add_asset(asset: AssetId) {
        storage.owner.only_owner();
        require(
            !_is_asset_supported(asset),
            AssetManagerErrors::AssetAlreadySupported
        );

        storage.is_supported.insert(asset, true);
        storage.assets.push(asset);
    }

    /// Removes asset from supported assets vec
    #[storage(read, write)]
    fn remove_asset(index: u64) {
        storage.owner.only_owner();
        let asset = storage.assets.remove(index);
        storage.is_supported.insert(asset, false);
    }
```

If the owner calls the remove_asset function while tokens are still in the pool, users holding those tokens will be unable to withdraw their amounts, effectively locking their funds temporary.

# Final word

I hope you enjoyed reading this report. I’m truly grateful for the opportunity to work on the first ever NFT marketplace on the **Fuel blockchain**. The codebase was challenging and battle-tested, but I did my best to secure it as much as possible by finding 8 H/M and L/I bugs.

A special thanks to the ThunderNFT protocol team and the Immunefi team for their support and for providing this incredible opportunity. I’ll be sharing more private and contest audits for Fuel dApps on my GitHub portfolio soon, stay tuned!
