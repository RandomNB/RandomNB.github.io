---
layout: post
title:  "Paradigm 2021 Upgrade题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 分析
1. 给USDC新增了两个函数```lend(address to, uint amount)```和```reclaim(address from, uint amount)```
2. 对于提供USDC闪电贷服务的合约，如果偿还USDC采用PUSH（用户主动偿还）而不是PULL（强制平仓），那我们可走lend偿还，表面上USDC还了回去，满足闪电贷条件。但实际上我们还拥有这些USDC的所有权，可以在接下来的一步reclaim回来这些USDC
3. 找到足够多的USDC闪电贷提供商，反复套利即可