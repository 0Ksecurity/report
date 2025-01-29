# Introduction

The ALCHEMIX veALCX boost on Immunefi was my first contest on the platform, and it was an incredible experience. although I hadn’t worked on any contests for a while, I decided to give this one a shot, and i’m glad i did. I spent around 7-10 days diving into the veALCX codebase. Initially, I had no deep understanding of how veTokens worked, but this contest gave me the chance to learn a great deal about them. i found two critical vulnerabilities, one of which was closed incorrectly due to lack of some details, but it was still valid for another whitehat and one medium and low severity bug. This experience has opened the door for me to participate in future contests on Immunefi.

# Disclaimer

The details shared in this report are for informational purposes only and are not intended to encourage or discourage users or investors from engaging with the mentioned bug bounty program. This report highlights a vulnerability I discovered while reviewing the specified protocol during a set period, offering insights from my personal experience. Please conduct your own research and due diligence before investing in or working on mentioned bounty/contest.

# About **ALCHEMIX veALCX**

veALCX is the tokenomics upgrade for ALCX, Alchemix's governance token. Users will lock 80/20 ALCX/ETH Balancer Liquidity Tokens into veALCX in exchange for voting power, ALCX emissions, and protocol revenue. Voting power is used to vote on snapshot proposals, on-chain governance of veALCX contracts, and gauge voting to direct ALCX emissions. veALCX users also earn a new ecosystem token called FLUX that allows for boosted gauge voting and early unlocks. for more information visit the contest page [here](https://immunefi.com/audit-competition/alchemix-boost/information/#top)

# Report Stats

| Field              | Details                                                                                                  |
| ------------------ | -------------------------------------------------------------------------------------------------------- |
| contest name       | [ALCHEMIX veALCX](https://immunefi.com/audit-competition/alchemix-boost/leaderboard/)                    |
| Report state       | PAID                                                                                                     |
| Report Severity    | MEDIUM                                                                                                   |
| Report status      | N/A                                                                                                      |
| total bugs found   | 2 critical 1 medium                                                                                      |
| amount payed       | N/A                                                                                                      |
| protocol team rate | 5/5 quick response from the team, providing full details for every response to a report, respectful team |

# Vulnerability Details

## [MEDIUM] malicious user can front run any call to the `swapReward` and cause reverting admin calls and prevent setting the correct index to the newToken

### Description

The voter.sol#swapReward function is designed to update the reward token from an old token to a new one. This function is restricted to the admin and internally calls Bribe.sol#swapOutRewardToken, which updates the isReward status for the new token to true and sets the old token’s index to the address of the new token. However, an attacker can frontrun the admin’s transaction, causing griefing and preventing the correct index from being set for the new token. This issue can result in:

- Gas Loss: The admin incurs unnecessary costs due to failed transactions.
- Reward Length Bloat: Each front-running attempt increases the rewards array’s length, adding inefficiency.
- Operational Disruption: The intended token swap is blocked, preventing the admin from correctly updating to the desired new reward token.

Since the impact involved griefing and blocking functionality, the report’s severity was classified as Medium for this contest

### in depth analysis

To invoke the `swapReward` function, the owner must first call the `whitelist` function to add the new token to the whitelist. Without this step, the swapReward function cannot be executed due to the whitelist checks enforced within the function, The swapReward function invokes the `swapOutRewardToken` function as shown below:

```solidity
 function swapReward(address gaugeAddress, uint256 tokenIndex, address oldToken, address newToken) external {
        require(msg.sender == admin, "only admin can swap reward tokens");
        IBribe(bribes[gaugeAddress]).swapOutRewardToken(tokenIndex, oldToken, newToken);
    }
```

As showed above, the **swapReward** function allows the owner to update the reward token by calling the `swapOutRewardToken` function.

```solidity
function swapOutRewardToken(uint256 oldTokenIndex, address oldToken, address newToken) external {
        require(msg.sender == voter, "Only voter can execute");
        require(IVoter(voter).isWhitelisted(newToken), "New token must be whitelisted");
        require(rewards[oldTokenIndex] == oldToken, "Old token mismatch");

        // Check that the newToken does not already exist in the rewards array
        for (uint256 i = 0; i < rewards.length; i++) {
            require(rewards[i] != newToken, "New token already exists");
        } // @audit check here can revert too

        isReward[oldToken] = false;
        isReward[newToken] = true;

        // Since we've now ensured the new token doesn't exist, we can safely update
        rewards[oldTokenIndex] = newToken; // set the old index to the new token
}

```

The tokenIndex is used to correctly update the token’s index within the rewards array. However, a potential vulnerability exists that allows a malicious user to exploit the system by front running the owner’s call to **swapReward**. If the attacker notices that a new token has been added to the whitelist, they can invoke the `notifyRewardAmount` function directly from the Bribe contract before the admin does. Since the Bribe contract is external and allows anyone to interact with it, the attacker can use the newly whitelisted token address to call the function. In this scenario, the notifyRewardAmount function will not properly set the index of the new token when calling the `_addRewardToken` function. The `_addRewardToken` function increases the rewards list by adding the newly whitelisted token but does not set its correct index. Instead of placing the token at the appropriate index, it will simply push the token to the end of the list.

This causes the owner’s call to swapReward to fail because the new token index is not set as expected, potentially causing the transaction to revert. The attacker’s front running leads to a situation where the rewards list grows unnecessarily and disrupts the expected token indexing process.

```solidity
function notifyRewardAmount(address token, uint256 amount) external lock {
        require(amount > 0, "reward amount must be greater than 0");

        // If the token has been whitelisted by the voter contract, add it to the rewards list
        require(IVoter(voter).isWhitelisted(token), "bribe tokens must be whitelisted");
        _addRewardToken(token);

        // bribes kick in at the start of next bribe period
        uint256 adjustedTstamp = getEpochStart(block.timestamp);
        uint256 epochRewards = tokenRewardsPerEpoch[token][adjustedTstamp];

        IERC20(token).safeTransferFrom(msg.sender, address(this), amount);

        tokenRewardsPerEpoch[token][adjustedTstamp] = epochRewards + amount;
        periodFinish[token] = adjustedTstamp + DURATION;

        emit NotifyReward(msg.sender, token, adjustedTstamp, amount);
    }

function _addRewardToken(address token) internal {
        if (!isReward[token] && token != address(0)) {
            require(rewards.length < MAX_REWARD_TOKENS, "too many rewards tokens");
            require(IVoter(voter).isWhitelisted(token), "bribe tokens must be whitelisted");

            isReward[token] = true;
            rewards.push(token); // @audit new token added here, lead the call to the swapOutRewardToken revert
        }
    }

```

In the above flow, once the malicious user calls **notifyRewardAmount**, the new token will be pushed to the end of the rewards array without its index being correctly set. As a result, the swapOutRewardToken function, when called by the admin, may not update the correct index for the new token(mostly revert). This not only prevents the proper indexing but also increases the rewards list, potentially leading to additional inefficiencies and a loss of control for the admin. In summary, the core issues are:

- Front-running vulnerability: A malicious user can front run the admin’s call to `swapReward` by invoking the `notifyRewardAmount` function. This results in a failure of the admin’s transaction due to a “New token already exists” error.

- Incorrect token index: The new token is incorrectly placed at the end of the rewards array, This leads to unintended behavior in the veALCX system and disrupts proper token management.

# Final word

I’m really glad I participated in this contest, it was an amazing experience. The communication with the Alchemix team was incredibly smooth and helpful, and i learned so much about the Alchemix protocol and the veALCX token. I’d be happy to participate in any future contests or BBP opportunities related to Alchemix and veTokens. Looking forward to more!
