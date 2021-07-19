---
layout: post
title:  "Paradigm 2021 Broker题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 逻辑
1. 用户可抵押weth到broker，然后借amt
2. 当weth-amt的比率变化到用户还不起时，别人可以替他还，换取他抵押的weth
3. 比率取决于单一pair的值，可砸入大量weth换出大量amt，使得weth价格跳水（！注意这里不要算错K，解题的时候换出来的amt少了1位，结果最后自己亏在里边了200weth）