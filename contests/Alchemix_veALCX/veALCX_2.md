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
| Report Severity    | CRITICAL                                                                                                 |
| Report status      | N/A                                                                                                      |
| total bugs found   | 2 critical 1 medium (one closed incorrectly but confirmed for another whitehat)                          |
| amount payed       | N/A                                                                                                      |
| protocol team rate | 5/5 quick response from the team, providing full details for every response to a report, respectful team |

# Vulnerability Details

## [CRITICAL] Killed Gauge Still Accumulates Funds After Being Set to Killed State

### Description

According to the veALCX design, a killed gauge should never accrue amounts after being set to the killed state. This principle applies to almost all veToken designs in DeFi. However, in this case, a killed gauge can still accumulate claimable amounts because the \_updateFor function does not prevent shares from being added to it.

### in depth analysis

The `_updateFor` function is responsible for updating the claimable amount for a given gauge. However, it does not prevent this update from occurring even if the gauge has been marked as killed. This means that a killed gauge can still accumulate claimable amounts, which contradicts the expected behavior of veALCX and similar veToken designs:

```solidity
 function _updateFor(address _gauge) internal {
        require(isGauge[_gauge], "invalid gauge");

        address _pool = poolForGauge[_gauge];
        uint256 _supplied = weights[_pool];
        if (_supplied > 0) {
            uint256 _supplyIndex = supplyIndex[_gauge];
            uint256 _index = index; // get global index0 for accumulated distro
            supplyIndex[_gauge] = _index; // update _gauge current position to global position
            uint256 _delta = _index - _supplyIndex; // see if there is any difference that need to be accrued
            if (_delta > 0) {
                uint256 _share = (uint256(_supplied) * _delta) / 1e18; // add accrued difference for each supplied token
                claimable[_gauge] += _share;
            }
        } else {
            supplyIndex[_gauge] = index;
        }
    }

```

let's Breakdown how `_updateFor` work

- The function first verifies that the provided `_gauge` address is valid using **require(isGauge[_gauge], "invalid gauge");**.

- It retrieves the associated pool for the gauge and fetches the `weights[_pool]`, which represents the allocated supply.

- If the supply is greater than zero, the function calculates the difference `_delta` between the global index and the last recorded `supplyIndex[_gauge]`.

- If `_delta` is positive, meaning there has been a distribution update, the function calculates the accrued share and adds it to `claimable[_gauge]`.

the problem is that A) once a gauge is marked as killed, it should no longer accrue rewards. This is a fundamental rule in veToken models, where killed gauges should stop accumulating claimable amounts B) The `_updateFor` function does not include any checks to prevent updates for killed gauges. As a result, claimable amounts continue to increase. In the Velodrome (VELO) protocol, this issue is avoided by explicitly setting the claimable amount to zero when a gauge is killed. However, if the Alchemix protocol does not intend to follow this approach, then `_updateFor` should at least ensure that claimable amounts are not updated for killed gauges.

# Final word

I’m really glad I participated in this contest, it was an amazing experience. The communication with the Alchemix team was incredibly smooth and helpful, and i learned so much about the Alchemix protocol and the veALCX token. I’d be happy to participate in any future contests or BBP opportunities related to Alchemix and veTokens. Looking forward to more!
