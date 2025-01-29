# Introduction

Last year, Immunefi hosted its first and most exciting Attackathon on their platform: the Fuel Attackathon, with a total bounty of $1 million. The Attackathon was divided into three categories based on programming languages: Rust, Sway, Solidity, TypeScript.

This event marked my first participation in a major contest/Attackathon as a security researcher. During this journey, I began learning about the Fuel blockchain and, most importantly, the Sway programming language. At the time, I had no prior experience with Rust-like languages or non-EVM blockchains. Despite this, I successfully secured 5th place in the Attackathon. In this Attackathon,

I focused full time on the Sway language category, which included exploring the Sway standard libraries, Sway libraries and standards, and Sway tooling.

This report highlights how I discovered one of the vulnerabilities that earned me $33.6k out of a total of $86,000 in rewards.

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
| amount payed       | $8.8k                                                                                        |
| protocol team rate | 5/5 respectful team, quick response, helped during the attackathon                           |

# Vulnerability Details

## [HIGH] the function subtract in signed libs(I8.sw) did not correclty handle the case when self.value is smaller than other.value value

### Description

The subtract function in the signed library is designed to subtract two I8 numbers, represented internally as u8. This contract utilizes a bias mechanism to accurately handle signed values, given that the FuelVM does not support negative numbers. However, the subtract function fails to properly handle scenarios where `self.value < indent and other.value > indent`. This issue can disrupt the subtraction logic, particularly when self is set to a smaller value than other, resulting in unintended behavior.

### in depth analysis

The subtract function in the provided code snippet is designed to handle subtraction of two **I8** numbers using a bias mechanism, as the FuelVM does not natively support signed integers. However, the implementation contains a critical flaw in the case where `self.underlying` is less than indent(128) and other.underlying is greater than or equal to the indent. In this scenario, an underflow occurs because the calculation performs `self.underlying - (other.underlying - Self::indent())`, and since self.underlying is smaller than `other.underlying - Self::indent()`, the FuelVM will panic due to the underflow:

```sway
impl core::ops::Subtract for I8 {
    /// Subtract a I8 from a I8. Panics of overflow.
    fn subtract(self, other: Self) -> Self {
        let mut res = Self::new();

        if self.underlying >= Self::indent()
            && other.underlying >= Self::indent()
        {
            if self.underlying > other.underlying {
                res = Self::from_uint(self.underlying - other.underlying + Self::indent());
            } else {
                res = Self::from_uint(self.underlying - (other.underlying - Self::indent()));
            }

        } else if self.underlying >= Self::indent()
            && other.underlying < Self::indent()
        {
            res = Self::from_uint(self.underlying - Self::indent() + other.underlying);

        } else if self.underlying < Self::indent()
            && other.underlying >= Self::indent()
        { //@audit

            res = Self::from_uint(self.underlying - (other.underlying - Self::indent())); // PANIC

        } else if self.underlying < Self::indent()
            && other.underlying < Self::indent()
        {

            if self.underlying < other.underlying {
                res = Self::from_uint(other.underlying - self.underlying + Self::indent());
            } else {
                res = Self::from_uint(self.underlying + other.underlying - Self::indent());
            }
        }
        res
    }
}
```

The function above should strictly accept values that are wrapped using the **I8...256.from_uint(u8... u64)** method. This ensures that the values are correctly aligned with the bias mechanism used in the contract. Using the from function instead of from_uint can disrupt the entire mathematical functionality of the contract, leading to critical issues, in another meaning, the key issue is that If a valid `u8` value is used for `self.value` and is smaller than `other.value`, while other.value is greater than indent (128), the subtraction operation will fail. Specifically, the subtract functionality will be blocked and result in a panic within the FuelVM. This is due to the inherent limitations of unsigned integer handling, which cannot accommodate negative values. While attempting to use the from function, it was observed that this approach breaks the entire mathematical consistency of the contract. If a u8 value is wrapped using `from` function, there is no scenario where a value smaller than indent can be processed. This would create a more severe issue, especially if the team intends to use from for signed integers instead of **from_uint**.

## Final words

I’m proud to have dedicated 17–20 days to working on the Fuel Attackathon. During this time, I not only learned a completely new programming language in a short period but also secured 5th place on the Attackathon leaderboard. It was a challenging yet rewarding experience that expanded my skills and knowledge significantly.
