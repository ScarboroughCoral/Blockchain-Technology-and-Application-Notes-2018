# 状态树

以太网是基于账户的，系统中维护每一个账户的状态。
本节需要讲解的是如何实现从账户地址到状态的映射。

账户是40位16进制数。
根据上节内容账户状态包含余额、交易次数等、合约账户还包含代码和存储等


一些方案思考
### 哈希表（不可行）

全节点维护一个哈希表。查询更新时间复杂度为常数

#### 问题
- 哈希碰撞
- 无法生成 Merkle tree
  - 无法提供 Merkle proof，证明账户状态
    - 无法将哈希表组织为 Merkle proof（新区块发布哈希内容会发生变化以太网网络所有账户状态重新组成 Merkle tree，代价太大，变化内容为一小部分）
  - Merkle tree 可以维护全节点的一致性，根哈希值。

### 直接使用 Merkle Tree（不可行）
全账户状态组成一棵 Merkle Tree，修改只需要一部分内容

#### 问题
- 没有提供高效的查询和更新方法
- 不同节点无法提供相同的 Merkle Tree
  - 发布区块，其他节点需要根哈希验证一致性，因此需要知道所有账户状态
  - btw，比特币发布区块的结点说了算

#### Sorted Merkle Tree 问题
- 网络中新增账户 Sorted Merkle Tree 整个结构很可能会发生变化，导致重新生成 Sorted Merkle Tree


### Trie
> re**trie**val 意思是信息检索

根据字符前缀拼接成一棵树。

#### 例子
cat, car, cap, bat, bar。

```
(root)
 └── g
     ├── e
     │   ├── n
     │   │   ├── e
     │   │   │   ├── r
     │   │   │   │   └── a
     │   │   │   │       └── l (end)
     │   │   │   └── s
     │   │   │       └── i
     │   │   │           └── s (end)
     └── o
         ├── (end)
         ├── d (end)
         └── o
             └── d (end)

- (root) 是根节点。
- g 下有 e，代表单词以 ge 开头。
- ge 下分支为 n，形成 general 和 genesis。
- go 下有直接结束的单词 go 和分支 d、od，形成 god 和 good。
```


```rust
struct TrieNode {
    children: HashMap<char, TrieNode>,
    is_end_of_word: bool,
}
```

#### 特点
- 节点的最大分支个数就是字符项的合法取值集合大小。
- 查找效率取决于字符串长度。字符串长度越长、访问内存次数越多。
- 不会出现“哈希”碰撞
- 不同插入顺序结果树的结构是一样的
- 更新操作的局部性，不影响其他分支
- 长链只有单个子节点会有存储浪费


### Patricia Trie/Tree

路径压缩的 Trie。新加入字符串时可能会导致压缩展开。对于分布稀疏的Trie特别有效。


#### 例子
```
(root)
 └── gen
     ├── eral (end)
     └── esis (end)
 └── go
     ├── (end)
     ├── d (end)
     └── od (end)

- (root) 是根节点。
- gen 是公共前缀，接着分支为 eral 和 esis，分别代表 general 和 genesis。
- go 是另一个公共前缀，接着有结束的 go 和分支 d、od，分别代表 god 和 good。
```

### MPT(Merkle Patricia Tree)

类似于区块链之于普通链表，MPT 是使用“哈希”指针的 Patricia Tree。

#### 特点
- 防篡改
  - 账户状态组成的 MPT 的根哈希不变，就表明整棵树没有变化
- Merkle Proof
  - 证明账户余额。将账户所在分支所需的 Merkle Proof 变量发给轻节点，轻节点即可验证。
  - 证明账户不在全节点中。将证明账户插入后所在的分支所需的 Merkle Proof 变量发给轻节点，轻节点即可验证。

### Modified MPT（以太坊中真实用到的）


- 扩展节点
  - 有路径压缩包含路径压缩信息
  - 字段
    - prefix。1字节，高位表示是否是扩展结点或者叶子节点，低位表示共享前缀nibbles的奇偶性，半字节即4比特。
    - shared nibbles。压缩共享的字符串序列。
    - next node。下一个分支节点的地址。
- 分支节点
  - 用于划分不同的后续节点
- 叶子节点
  - 引用真实账户状态信息
  - 字段
    - prefix。1字节，高位表示是否是扩展结点或者叶子节点，低位表示共享前缀nibbles的奇偶性，半字节即4比特。
    - key-end。表示结束字符串序列
    - value。账户状态，子树 Sub MPT。
      - 如果是合约账户则有合约代码、storage root。
      - 余额

#### 优势

- 修改不是原地修改，而是未修改的使用引用，已修改的创建新的，生成最后的MPT。有点类似COW、flux/Redux 架构。
- 可时间回溯。有效保存历史，方便回滚账户余额、合约代码、合约状态存储等内容。


### 数据结构

#### 区块
- header。区块头
- uncles。叔父节点的区块头数组
- transactions。包含的交易列表
#### 区块头
- ParentHash。父节点区块头Hash
- UncleHash。叔父节点的哈希
- Coinbase。矿工地址
- Root。状态树根哈希
- TxHash。交易树根哈希
- ReceiptHash。收据树根哈希
- Bloom。跟收据树相关的提供高效查询的字段
- Difficulty。挖矿难度
- GasLimit。汽油费相关
- GasUsed。汽油费相关
- Time。区块大致产生时间
- MixDigest。挖矿依赖参数
- None。挖矿依赖参数

#### 发布区块 external block

- Header 区块头
- Txs。交易列表
- Uncles。叔父节点区块头。

### 序列化

使用 RLP（Recursive Length Prefix），超简易版本的 protocol buffer
- 只支持 nested array of bytes。
- 更具体的类型由上层应用层实现。