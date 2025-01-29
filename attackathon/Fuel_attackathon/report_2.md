# Introduction

Last year, Immunefi hosted its first and most exciting Attackathon on their platform: the Fuel Attackathon, with a total bounty of $1 million. The Attackathon was divided into three categories based on programming languages: Rust, Sway, Solidity, TypeScript.

This event marked my first participation in a major contest/Attackathon as a security researcher. During this journey, I began learning about the Fuel blockchain and, most importantly, the Sway programming language. At the time, I had no prior experience with Rust-like languages or non-EVM blockchains. Despite this, I successfully secured 5th place in the Attackathon.

This report highlights how I discovered one of the vulnerabilities that earned me $33k out of a total of $86,000 in rewards.

# Disclaimer

The details shared in this report are for informational purposes only and are not intended to encourage or discourage users or investors from engaging with the mentioned bug bounty program. This report highlights a vulnerability I discovered while reviewing the specified protocol during a set period, offering insights from my personal experience. Please conduct your own research and due diligence before investing in or working on mentioned bounty/contest.

# About **FUEL blockchain**

Fuel is an operating system purpose built for Ethereum Rollups. Fuel allows rollups to solve for PSI (parallelization, state minimized execution, interoperability) without making any, for more information read their docs [here](https://docs.fuel.network/docs/intro/what-is-fuel/).

# Report Stats

| Field              | Details                                                                                      |
| ------------------ | -------------------------------------------------------------------------------------------- |
| Attackathon name   | [Fuel Network](https://immunefi.com/audit-competition/fuel-network-attackathon/leaderboard/) |
| Report state       | PAID                                                                                         |
| Report Severity    | HIGH                                                                                         |
| Report status      | Chief Finding                                                                                |
| total bugs found   | 3 High(3 cheif finder), 5 low/insight                                                        |
| amount payed       | $33.6k                                                                                       |
| protocol team rate | 5/5 respectful team, quick response, helped during the attackathon                           |

# Vulnerability Details

## [HIGH] the non_negative set incorrectly in the ceil function

### Description

In addition to the [double increasing bug]() I discovered in the `ceil` function, I identified another issue that could lead to incorrect logical flow when the ceil function is invoked by third parties. This issue arises from setting an incorrect non_negative value, which is later returned by the function.
The ceil function is designed to return two values: the underlying value and the `non_negative` flag, which determines whether the returned underlying value is negative. The function has two primary cases:

- If self.non_negative == true, the function processes the value accordingly.

- If self.non_negative == false, the else clause is executed.

Within the[ else clause](https://github.com/FuelLabs/sway-libs/blob/0f47d33d6e5da25f782fc117d4be15b7b12d291b/libs/src/fixed_point/ifp64.sw#L472-L479), the logic sets the non_negative value to true, this step is crucial step for ensuring that the returned value correctly indicates whether it is negative or not. However, when the final result is returned by the ceil function, it uses the original self.non_negative value instead of the updated one from the else clause logic. As a result, the function incorrectly returns values with an outdated non_negative flag, leading to potential logical inconsistencies and unexpected behaviors in protocols that rely on this librar

The returned value from the ceil function in this case will look something like this:

```sway
underlying value: postive number
non_negative: false --> negative value
```

### in depth analysis

let's take a closer look at the ceil function:

```sway
 pub fn ceil(self) -> Self {
        let mut underlying = self.underlying;
        let mut non_negative = self.non_negative;

        if self.non_negative {
            underlying = self.underlying.ceil();
        } else {

            let ceil = self.underlying.ceil();

            if ceil != self.underlying {
                underlying = ceil + UFP64::from(1);
                if ceil == UFP64::from(1) {
                    non_negative = true; //@audit in this situation non negative should be true
                }
            } else {
                underlying = ceil;
            }
        }
        Self {
            underlying: underlying,
            non_negative: self.non_negative,//@audit we return the unupdated value instead of the non_negative
        }
    }

```

This highlights the issue, even though the else clause of the function attempts to ensure that the non_negative flag is correctly updated to true, the final result still reflects the outdated non_negative value. This discrepancy leads to the incorrect interpretation of the underlying value as negative, despite it being positive. Such behavior can cause significant logical errors in protocols that depend on the ceil function for accurate computations.

The ceil function should correctly enforce the setting of non_negative to true when appropriate, ensuring that the returned value aligns with the actual computation.

# Final word

While many security researchers avoid working on Sway language libraries and standards, given that the language was created by the team itself, I believed that vulnerabilities can exist in any codebase even when authored by the language’s creators. After all, mistakes can happen, and bugs are often present in fresh codebases that haven’t undergone audits. By maintaining this mindset, I was able to identify another high severity issue along with several low severity and insightful findings in the Sway language. These contributions brought my total rewards to $86k. I’m truly grateful for the opportunity to work on one of the most promising blockchain ecosystems of the future and contribute to securing it to the best of my abilities.
