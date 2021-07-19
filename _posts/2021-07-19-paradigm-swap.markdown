---
layout: post
title:  "Paradigm 2021 SWAP题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
这题很遗憾没做出来，没有想到这个点，以下逻辑来自官方wp

### 题目逻辑
1. Setup创建了一个基于四种稳定币的AMM（自动做市商），发行自己的LP（流动性提供者）币，提供mint（4种稳定币换LP）、burn（LP换4种稳定币）、swap（4种稳定币互换）的服务，经过分析其逻辑上没有重入、闪电贷攻击等漏洞
2. 题目目标分析如下
    1. 需要移除大部分资产（99%但不需要全部）
    2. 使资产流出的函数只有```swap```和```burn```
    3. ```swap```函数看起来正常，无法窃取
    4. ```burn```函数看起来正常，无法窃取
    5. ```swap```函数是独立逻辑，难以下手
    6. ```burn```函数是```two-step```逻辑，因为第一步需要LP币，但没规定LP币从何而来，所以新的目标是通过免费手段获取LP币然后走```burn```窃取资产
3. 能获取LP币的函数只有两个，```mint```和```transfer```，后者需要已经有LP币，既然没办法把setup的LP币偷走，那就只有前者能用

```js
function mint(uint[] memory amounts) public nonReentrant returns (uint) {
        MintVars memory v;
        v.totalSupply = supply;
        
        for (uint i = 0; i < underlying.length; i++) {
            v.token = underlying[i]; // underlying被setip锁死了4个
            
            v.preBalance = v.token.balanceOf(address(this));
            
            v.has = v.token.balanceOf(msg.sender);
            if (amounts[i] > v.has) amounts[i] = v.has; // 自动缩减到sender拥有的
            
            v.token.transferFrom(msg.sender, address(this), amounts[i]);
            
            v.postBalance = v.token.balanceOf(address(this));
            
            v.deposited = v.postBalance - v.preBalance;

            v.totalBalanceNorm += scaleFrom(v.token, v.preBalance);
            v.totalInNorm += scaleFrom(v.token, v.deposited);
            // usdc、usdt的decimals都是6，dai、tusd的decimals都是18
        }
        
        if (v.totalSupply == 0) {
            v.amountToMint = v.totalInNorm;
        } else {
            v.amountToMint = v.totalInNorm * v.totalSupply / v.totalBalanceNorm;
            // 这里相除了，Norm变大的又缩回来了，无法利用
        }
        
        supply += v.amountToMint;
        balances[msg.sender] += v.amountToMint;
        
        return v.amountToMint;
}
```
想要以极低的成本获取大量的LP币，那就是使```v.amountToMint```尽可能大

### 官解

漏洞触发点在amounts的长度，看反汇编代码更清晰

```python
def mint(array _param1): # not payable
  mem[128 len 32 * _param1.length] = call.data[_param1 + 36 len 32 * _param1.length]
  mem[(32 * _param1.length) + 160] = 0 # totalBalanceNorm, amounts[1]
  mem[(32 * _param1.length) + 192] = 0 # totalInNorm, amounts[2]
  mem[(32 * _param1.length) + 224] = 0 # amountToMint, amounts[3]
  mem[(32 * _param1.length) + 256] = 0 # token
  mem[(32 * _param1.length) + 288] = 0 # has
  mem[(32 * _param1.length) + 320] = 0 # preBalance
  mem[(32 * _param1.length) + 352] = 0 # postBalance
  mem[(32 * _param1.length) + 384] = 0 # deposited
  if stor0 == 2:
      revert with 0, 'ReentrancyGuard: reentrant call'
  stor0 = 2
  mem[(32 * _param1.length) + 128] = totalSupply # totalSupply, amounts[0]
  idx = 0
  while idx < unknownf8fdeb68.length:
      .... # 省略
      require idx < _param1.length
      if mem[(32 * idx) + 128] <= ext_call.return_data[0]: # amounts[idx] <= balanceOf
          .... # 省略
      else:
          mem[(32 * idx) + 128] = ext_call.return_data[0]
```

当_param1即amounts的长度乘以32刚好为ZINF时，v所占的内存和amounts是一致的，因此amounts[0]等价于v.totalSupply，其他1、2、3的等价关系也备注在了代码中


Because v and amounts now alias each other, both will share the same values and updating one will update the other. This means that while you can manipulate v by writing into amounts, the reverse is true as well.
As such, if you just tried to force some initial values into v by calling mint with some custom values, you might've found that your data kept getting clobbered by the writes to v.totalSupply, v.totalBalanceNorm, and v.totalInNorm. This means that some careful planning is needed to make sure that your values stay consistent over all four iterations of the loop.
Additionally, v.amountToMint is updated after the four iterations, meaning that even if we try to write directly to amounts[3], it'll just be overwritten with the correctly calculated value in the end. This means that we'll need to either make v.totalInNorm or v.totalSupply extremely big, or make v.totalBalanceNorm extremely small.
To do this, we first swap out all of the stablecoins in the AMM except for DAI. This means that v.totalBalanceNorm will not increase after the first iteration because the AMM has no balance. This just makes our lives a bit easier later on.
Next, we'll purchase the right amount of DAI/TUSD/USDC such that when amounts[i] = v.has is executed, we'll write our desired values into v. The balance of DAI will be written to v.totalSupply, so we'll want to set this to a large number such as 10000e18. The balance of TUSD will be written to v.totalBalanceNorm, so we'll update that to 1. Finally, the balance of USDC will be written to v.totalInNorm so we'll also set that to 10000e18. There's no point manipulating the USDT balance because the value will be clobbered anyways.
Finally, we just need to assemble our payload and send the transaction, giving us the flag we worked so hard for: 