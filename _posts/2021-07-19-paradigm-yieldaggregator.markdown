---
layout: post
title:  "Paradigm 2021 YieldAggregator题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 分析
1. 要求清空bank和aggregator的WETH，bank一开始有50 WETH，aggregator一开始没有WETH
2. 要想拿走WETH，走withdraw函数，但要求```poolTokens[msg.sender]```大于等于amount即50 ether
3. aggregator的deposit函数将amounts的tokens存入银行，银行返回凭据，然后```poolTokens[msg.sender] += diff;```，那我们可以构造一个假银行，主要逻辑和MiniBank一样，增加一个give_me函数把自己拥有的WETH全都给用户（总想回本）
4. deposite到假银行，增加```poolTokens[msg.sender]```，然后withdraw从真银行拿走WETH