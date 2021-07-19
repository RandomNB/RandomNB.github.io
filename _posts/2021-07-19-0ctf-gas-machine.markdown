---
layout: post
title:  "0ctf 2021 GasMachine题解"
date:   2021-07-19 11:07:32 +0800
categories: 0ctf
---

## 解题思路

先消耗大量gas，然后用JUMPDEST销毁剩余GAS

```
JUMPDEST
PUSH2 61
GAS
LT
ISZERO
PUSH1 0x00
JUMPI
GAS
PUSH1 92
SUB
JUMP
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
JUMPDEST
STOP
```


## 拓展：GAS的计算

1. 只要进行操作，gas就已经减去了21000，进入第一个指令的gas为```提供的gas-21000```
2. Transaction Receipt中的```gasUsed```通常是```合约执行消耗的gas+21000```
3. 存在SSTORE归0的操作时，```gasUsed```会变少，但合约执行过程中消耗的gas不变，**存疑**
4. 合约中第一次写指定slot的storage，如果slot本身有非0值，消耗5000，之后消耗800，如果本身是0值，消耗20000，之后消耗800


## MSTORE的GAS计算

根据EIP-3336中的描述
Presently, the total cost to extend the memory to ```a``` words long is ```Cmem(a) = 3 * a + floor(a ** 2 / 512)```. If the memory is already ```b``` words long, the incremental cost is ```Cmem(a) - Cmem(b)```. ```a``` is the number of words required to cover the range from memory address 0 to the last word that has been read or written by the EVM.

此处```a```是字节数除以32

例如

```
PUSH2 0x0001
PUSH2 0x3000
MSTORE
PUSH2 0x0001
PUSH2 0x5000
MSTORE
PUSH2 0x0001
PUSH2 0x7000
MSTORE
STOP
```
第一次MSTORE的```a```值为```ceil((0x3000+32)/32)```即```a=385```（多加32是因为MSTORE是从起始地址往后存32字节），根据公式消耗为1444，此外MSTORE的基础fee为3，总和为1447
第二次MSTORE的```a```值为```ceil((0x5000+32)/32)```即```a=641```，根据公式消耗为2725，减去之前的消耗1444，加上基础fee的3，总和为1284
第三次MSTORE的```a```值为```ceil((0x7000+32)/32)```即```a=897```，根据公式消耗为4262，减去之前的消耗2725，加上基础fee的3，总和为1540