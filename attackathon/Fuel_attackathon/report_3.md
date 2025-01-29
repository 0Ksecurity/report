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
