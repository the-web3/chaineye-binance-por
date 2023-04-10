# Proof of solvency

Chaineye rust binance por 教程，供想要学习的小伙伴学习。

推特：@seek_web3

Chainey 社群： 官网 chaineye.info | Chaineye Rust 教程 | 微信: LGZAXE, 加微信之后拉各社群 

所有代码和教程开源在github: https://github.com/0xchaineye/chaineye-binance-por

-----------------------------------------------------------------------------------------------------------------------------------------------------------

## 例子

假设有2个资产（ETH、USDT）和3个用户，以USDT为基础计价资产。 假设以下用户行为：

- Alice向CEX存入10000 USDT，然后以10000 USDT为抵押借入2 ETH，再以1 ETH为抵押借入1000 USDT； 并用 Bob 的 2000 USDT 交换 1 ETH
- Bob 向 CEX 存入 10 ETH 和 10000 USDT； 用 Alice 的 1 ETH 交换 2000 USDT
- Carl 存入10 ETH到CEX，然后用1 ETH抵押借入1000 USDT

用户的资产负债表如下：

|  | ETH (price:2000 USDT) | ETH | USDT (price: 1 USDT) | USDT | Total Net Balance (USDT) |
| --- | --- | --- | --- | --- | --- |
|  | Equity | Debt | Equity | Debt |  |
| Alice | 1 | 2 | 13000 | 1000 | 10000 |
| Bob | 11 | 0 | 8000 | 0 | 30000 |
| Carl | 10 | 0 | 1000 | 1000 | 20000 |


CEX持有的资产等于每个用户净资产余额（权益-债务）的总和。 所以CEX至少需要持有20个ETH和20000个USDT。

在偿付能力证明中，保证了以下属性：

- 每个用户的总净余额大于0；
- 用户的每笔资产都是CEX申报的净资产总额的一部分：
     - 对于alice，-1 eth是total net eth的一部分，12000 usdt是total net usdt的一部分；
     - 对于Bob，11 eth是total net eth的一部分，8000 usdt是total net usdt的一部分；
     - 对于Carl来说，10 eth是total net eth的一部分，0 usdt是total net usdt的一部分
- CEX申报的净资产总额等于每个用户净资产余额的总和

## 2.设计

设计的主要思想是通过zksnark保证世界状态树构建过程的正确性。

假设有 `ASSET_NUM` 资产和 `ACCOUNT_NUM` 账户。 定义以下结构：

```
type CexAssetInfo {
  total_equity    u64
  total_debt      u64
  usdt_price      u64
}

type AssetInfo {
   equity    u64
   debt      u64
}

type CreateUserOp {
  before_account_tree_root  Hash
  assets                    [ASSET_NUM]AssetInfo
  after_account_tree_root   Hash
  account_index             u32
  account_proof             [ACCOUNT_TREE_DEPTH]Hash
}

type AccountTreeLeafNode {
  equity_usdt          u64
  debt_usdt            u64
  assets_commitment    Hash
}

type CexAssetInfoList  [ASSET_NUM]CexAssetInfo
type AssetInfoList     [ASSET_NUM]AssetInfo
type CreateUserOpList  [BATCH_NUM]CreateUserOp
```

CEX 可以根据上面定义的用户资产负债表快照为每个账户构建 CexAssetInfoList 和 AssetInfoList。 并且 `CexAssetInfoList` 被发布，`AssetInfoList` 对每个用户保密。

在本设计中，CEX 使用索引来表示用户和资产类型，例如：

| 0 | 1 | … | n |
| --- | --- | --- | --- |
| alice | bob | … | dummy account |

| 0 | 1 | … | n |
| --- | --- | --- | --- |
| ETH | USDT | … | dummy asset |

## 3.资产承诺

在电路中，CEX 需要计算 CexAssetInfoList 和 AssetInfoList 的承诺，有两种方式：

- 直接散列所有元素：`commitment(CexAssetInfoList) = hash(ele0, ele1, ..., elen)` ；
- 使用默克尔树来组织所有元素，那么承诺就是默克尔树根

在这个设计中，我们使用第一种方法来计算资产信息列表的承诺，因为当不需要为用户验证单个资产时，与 merkle 树相比，电路约束数量更少。

## 用户树

使用 smt 组织所有帐户如下：

- `hash(AccountTreeLeafNode) = hash(accountId, equity, debt, commitment(AssetInfo))`;
- `hash(parent) = hash(left_node, right_node)`

[![merkle-tree-1](https://github.com/0xchaineye/chaineye-binance-por/blob/main/images/merkle-tree-1.png)](https://github.com/0xchaineye/chaineye-binance-por/)

## 4.电路设计

BatchCreateUserCircuit 用于生成账户树和所有账户负债的总和，电路定义如下：

```
type BatchCreateUserCircuit struct {
  before_account_tree_root         Hash
  after_account_tree_root          Hash
  before_asset_list_commitment     Hash
  after_asset_list_commitment      Hash
  batch_commitment                 Hash `public`
  before_asset_list                CexAssetInfoList
  create_user_ops                  CreateUserOpList
}
```

- 输入/见证
     - 公众意见
         - `batch_commitment`：`(before_account_tree_root, after_account_tree_root, before_asset_list_commitment, after_asset_list_commitment)`的散列
     - 私人输入
         - `before_account_tree_root`：在当前批处理创建用户操作执行之前的帐户树的根目录；
         - `after_account_tree_root`：当前批量创建用户操作执行后账户树的根目录；
         - `before_asset_list_commitment`：在执行当前批量创建用户操作之前提交cex资产信息列表；
         - `after_asset_list_commitment`：在执行当前批量创建用户操作后提交cex资产信息列表；
         - `before_asset_list`：在执行当前批创建用户操作之前的cex资产信息列表；
         - `create_user_ops`：创建用户操作列表
- 约束：
     - 验证 `batch_commitment` 是否计算正确；
     - 使用“before_asset_list”验证是否正确计算了“before_asset_list_commitment”；
     - 验证 `before_account_tree_root` 是否等于 `create_user_ops` 第一个元素的 `before_account_tree_root` ；
     - 验证 `after_account_tree_root` 是否等于 `create_user_ops` 最后一个元素的 `after_account_tree_root` ；
     - 根据 `before_asset_list` 和 `create_user_ops` 生成 `after_asset_list` ；
         - 检查列表中的每个资产值是否在 [0, 2^64) 中；
         - 检查每项资产的价格在 [0, 2^64) 中；
         - 求和后检查潜在的溢出
     - 验证是否使用 `new_asset_list` 正确计算了 `after_asset_list_commitment` ；
     - 对于“create_user_ops”：
         - 使用空帐户叶节点和 account_proof 验证是否正确计算了 `before_account_tree_root` ；
         - 根据`before_asset_list`中的`assets`和`price`生成account叶节点，检查总权益是否大于总负债
         - 使用范围检查验证“资产”的有效性：每个值都在 [0, 2^64)
         - 使用新账户叶节点和 account_proof 验证是否正确计算了 after_account_tree_root ；
         - 验证 `after_account_tree_root` 是否等于下一个创建用户操作的 `before_account_tree_root`

当所有用户创建完成后，我们会得到一个证明列表`[proof_0, proof_1, ..., proof_n]`，一个列表`[batch_commitment_0, batch_commitment_1, ..., batch_commitment_n]`，一个列表`[asset_list_commitment_0] , asset_list_commitment_1, ..., asset_list_commitment_n]` 和 `[account_tree_root_0, account_tree_root_1, ..., account_tree_root_n]` 的列表。 所有名单均已公布。 我们可以使用智能合约来验证这些状态转换是否正确，审计人员或任何感兴趣的用户可以下载这些列表并自行验证。 验证过程如下：

- 检查等式：`batch_commitment_n ?= hash(asset_list_commitment_(n-1), asset_list_commitment_n, account_tree_(n-1), account_tree_n)`。
     - 当 n = 0 时，`asset_list_commitment_-1` 等于初始资产列表的承诺。 假设0号资产为ETH，1号资产为USDT，那么初始资产列表为`[(equity: 0, debt: 0, usdt_price: 20000), (equity: 0, debt: 0, usdt_price: 1)…] `;
     - 当 n = 0 时，`account_tree_root_-1` 等于空帐户树的根。
- 检查 zk.verify(batch_commitment_0, proof_0, verifying key) 输出；
- 检查`asset_list_commitment_n`是否等于CEX公布的`asset_list`的承诺，例如`[(equity: 22, debt: 2, usdt_price: 20000), (equity: 22000, debt: 2000, usdt_price: 1)] ` , `equity-debt`其实就是asset的总负债

用户可以从 CEX 获取账户信息和 merkle 证明，并进行以下检查：

- 首先计算账户信息的哈希值：`account_hash = hash(equity_usdt, debt_usdt, commitment(account_asset_list))` ;
- 然后计算默克尔根：`actual_root = computeMerkleRoot(account_index, account_hash, merkle_proof)` ;
- 检查 `actual_root ?= account_tree_root_n` 其中 `account_tree_root_n` 由 CEX 发布并由审计员或智能合约验证。

如果以上所有检查均通过，则用户可以确信：

- 他拥有或欠下的每一项资产都包含在CEX申报的资产负债总额中；
- 总资产负债中包含的每个用户净资产大于0；
- CEX 公布的总资产负债是 `account_tree_root_n` 代表的账户树中包含的所有用户的总和

## 5.性能分析

`BatchCreateUser` 电路性能分析基于`gnark` 实现的`groth16`：

- 使用poseidon哈希计算资产信息列表的承诺
     - `commitment(asset_info_list) = poseidon(asset0_equity, asset0_debt, asset1_equity, asset1_debt, …)` ; 输入的数量是 `ASSET_NUM*2`
     - 假设 ASSET_NUM 为 256，根据我们的 poseidon 哈希测试，256 个 poseidon 哈希输入的成本约为 10944 个约束；
- 使用poseidon hash构建账户树
     - 电路中有1个merkle proof验证和1个merkle root计算；
     - 根据我们的 poseidon 哈希测试，poseidon 哈希的 2 个输入成本约为 241 个约束；
     - 假设 `ACCOUNT_TREE_DEPTH` 是 28，所以约束数是 `2*28*241=13496`
- 每个资产的“权益”和“债务”的“范围检查”：“2*64*256=32768”
- 所以每个创建用户操作消耗`10944+32768+13496=58000`，有`BATCH_NUM`创建用户操作，假设`BATCH_NUM`是910，所以约束数是`910*58000=52.8m`

根据我们的测试结果，5400万个约束需要一台服务器（64G 32核）2分钟来生成证明； 假设有 1 亿个账户，它需要 `1 亿 / 910 * 2 / 60 / 24 = 152 天` 。 而且证明生成可以很容易地并行化，所以如果我们使用 100 台服务器，那么大约需要 1.5 天。











