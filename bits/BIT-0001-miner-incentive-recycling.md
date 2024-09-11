# BIT-0001: Subnet Miner Emissions Recycling - Partial Incentive

- **BIT Number:** 0001
- **Title:** Miner Emission Recycling
- **Author(s):** Thomas Dougherty (tdougherty@taoshi.io) and Arrash Yasavolian (arrash@taoshi.io)
- **Discussions-to:** https://discord.com/channels/1120750674595024897/1242999357436071956/1273127932491075638
- **Status:** Draft
- **Type:** Core
- **Created:** September 11, 2024
- **Updated:** Sepetmber 11, 2024
- **Requires:**
- **Replaces:**

## Abstract

Subnet owners may recycle all or a portion of miner emissions based on the performance of miners if they fail to meet predefined quality criteria.

## Motivation

Upon release, subnets with advanced requirements may be affected by low-quality miners as their talent pool is still developing. This negatively impacts the subnet’s reputation, as top-performing miners may deliver low-quality results, and it is inefficient for Bittensor to invest TAO in the subnet before it has attracted higher-caliber talent. Subnets should be able to adjust incentives based on miner performance benchmarks. If miners fail to meet these benchmarks, subnets should be able to recycle some or all of their emissions. This allows for careful investment into the project without causing deflationary pressure on TAO due to initial exploitation.

## Specification

This feature requires the following: _The ability to specify a percentage value between 0 and 1, indicating the extent to which the full round of emissions should be utilized._

Miner weights will still be normalized against each other but will now be multiplied by this factor to reduce their emissions before distribution.

## Rationale

This functionality allows subnets to tailor emission distribution based on miner performance. A 0% emission allocation indicates that no miners met the required performance criteria and miners will receive no incentive, while 100% allocation means miners are eligible for full emissions distribution (currently standard).

## Backwards Compatibility

Subnets that do not adopt emissions recycling will continue to distribute all emissions to miners each round without any changes.

## Security Considerations

By reducing emissions to underperforming miners, gaming the system becomes more difficult across subnets, decreasing the potential attack surface through traditional routes.

## Copyright

This document is licensed under [The Unlicense](https://unlicense.org/).
