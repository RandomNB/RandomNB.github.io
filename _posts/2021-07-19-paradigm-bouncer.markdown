---
layout: post
title:  "Paradigm 2021 Bouncer题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 逻辑
1. 看到redeem，以为有个下溢，结果死活过不去，报错revert，仔细一查，发现solidity 8.0引入了```集成 SafeMath```机制
2. convertMany函数有漏洞，对于ETH，只需要转账1次就能多次计算tokens+=
3. bouncer现有52ETH，先付1ETH小费申请ETH的entry，然后```convertMany(0xeee..., [0]*(52+1+1)).value(1 ether)```
4. 即可使balance为54ETH且自己的额度为54ETH
5. 调用```redeem(player, 54ether)```即可