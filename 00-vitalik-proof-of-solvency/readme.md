# vitalik 中心化交易的资产证明解决方案


Chaineye rust binance por 教程，供想要学习的小伙伴学习。

推特：@seek_web3

Chainey 社群： 官网 chaineye.info | Chaineye Rust 教程 | 微信: LGZAXE, 加微信之后拉各社群 

所有代码和教程开源在github: https://github.com/0xchaineye/chaineye-rust

------------------------------------------------------------------------------------------------------------------------------------------------------------------

每当一个主要的中心化交易所爆炸时，一个普遍的问题就是我们是否可以使用密码技术来解决这个问题。与其仅仅依靠政府许可、审计员和检查公司治理以及经营交易所的个人背景等“法定”方法，交易所还可以创建加密证明，表明他们在链上持有的资金足以支付其负债给他们的用户。

更雄心勃勃的是，交易所可以建立一个系统，在未经存款人同意的情况下，它根本无法提取存款人的资金。潜在地，我们可以探索“不作恶”有抱负的好人 CEX 和“不能作恶”之间的整个范围，但目前是低效和隐私泄露的链上 DEX。这篇文章将探讨使交易所更接近去信任化一两步的尝试历史、这些技术的局限性，以及一些依赖ZK-SNARK和其他先进技术的更新和更强大的想法。

## 1.余额列表和 Merkle 树：老式的偿付能力证明

交易所最早尝试以加密方式证明他们没有欺骗用户的尝试可以追溯到很久以前。2011 年，当时最大的比特币交易所 MtGox 通过发送一笔将424242 BTC 转移到预先公布的地址的交易来证明他们有资金。2013年开始讨论如何解决问题的另一面：证明客户存款的总规模。如果你证明客户的存款等于 X（“负债证明”），并证明拥有 X 个币的私钥（“资产证明”），那么你就有了偿付能力证明：你已经证明交易所有偿还所有存款人的资金。

证明存款的最简单方法是简单地发布(username, balance)货币对列表。每个用户都可以检查他们的余额是否包含在列表中，任何人都可以检查完整列表以查看（i）每个余额都是非负的，并且（ii）总金额是索赔金额。当然，这会破坏隐私，所以我们可以稍微改变一下方案：发布一个(hash(username, salt), balance)配对列表，然后私下向每个用户发送他们的salt值。但即使这样也泄露了余额，它泄露了余额变化的模式。保护隐私的愿望将我们带到了下一个发明：Merkle 树技术。

[![merkle_tree_1](images/merkle_tree_1.png)]([https://github.com/savour-labs](https://github.com/0xchaineye/chaineye-binance-por/))
绿色：Charlie 的节点。蓝色：Charlie 将收到的节点作为他证明的一部分。黄色：根节点，公开给大家看。


Merkle 树技术包括将客户余额表放入Merkle 和树中。在 Merkle 和树中，每个节点都是一(balance, hash)对。底层叶节点代表个人客户的余额和盐渍用户名哈希。在每个上层节点中，余额是下面两个余额的总和，哈希是下面两个节点的哈希。Merkle 和证明与 Merkle 证明一样，是树的一个“分支”，由从叶到根的路径上的姐妹节点组成。

交易所将向每个用户发送其余额的 Merkle 总和证明。然后用户可以保证他们的余额正确地包含在总数中。可以在此处找到一个简单的[示例代码](https://github.com/ethereum/research/blob/master/proof_of_solvency/merkle_sum_tree.py)实现。

代码如下：

```
# Example code for building and getting proofs in a Merkle sum tree,
# used to make proofs of solvency in an exchange
#
# THIS IS EDUCATIONAL CODE, NOT PRODUCTION! HIRE A SECURITY AUDITOR
# WHEN BUILDING SOMETHING FOR PRODUCTION USE.

import hashlib

# Mathematical helper methods

def hash(x):
    return hashlib.sha256(x).digest()

def log2(x):
    return len(bin(x)) - 2

def get_next_power_of_2(x):
    return 2 * get_next_power_of_2((x+1)//2) if x > 1 else 1

# Each user has a username and balance, and gets a salt generated
# This gets converted into a leaf, which does not reveal the username
def userdata_to_leaf(username, salt, balance):
    return (hash(salt + username), balance)

EMPTY_LEAF = (b'\x00' * 32, 0)

# The function for computing a parent node given two child nodes
def combine_tree_nodes(L, R):
    L_hash, L_balance = L
    R_hash, R_balance = R
    assert L_balance >= 0 and R_balance >= 0
    new_node_hash = hash(
        L_hash + L_balance.to_bytes(32, 'big') +
        R_hash + R_balance.to_bytes(32, 'big')
    )
    return (new_node_hash, L_balance + R_balance)

# Builds a full Merkle tree. Stored in flattened form where
# node i is the parent of nodes 2i and 2i+1
def build_merkle_sum_tree(user_table: "List[(username, salt, balance)]"):
    tree_size = get_next_power_of_2(len(user_table))
    tree = (
        [None] * tree_size +
        [userdata_to_leaf(*user) for user in user_table] +
        [EMPTY_LEAF for _ in range(tree_size - len(user_table))]
    )
    for i in range(tree_size - 1, 0, -1):
        tree[i] = combine_tree_nodes(tree[i*2], tree[i*2+1])
    return tree

# Root of a tree is stored at index 1 in the flattened form
def get_root(tree):
    return tree[1]

# Gets a proof for a node at a particular index
def get_proof(tree, index):
    branch_length = log2(len(tree)) - 1
    # ^ = bitwise xor, x ^ 1 = sister node of x
    index_in_tree = index + len(tree) // 2
    return [tree[(index_in_tree // 2**i) ^ 1] for i in range(branch_length)]

# Verifies a proof (duh)
def verify_proof(username, salt, balance, index, user_table_size, root, proof):
    leaf = userdata_to_leaf(username, salt, balance)
    branch_length = log2(get_next_power_of_2(user_table_size)) - 1
    for i in range(branch_length):
        if index & (2**i):
            leaf = combine_tree_nodes(proof[i], leaf)
        else:
            leaf = combine_tree_nodes(leaf, proof[i])
    return leaf == root

def test():
    import os
    user_table = [
        (b'Alice',   os.urandom(32), 20),
        (b'Bob',     os.urandom(32), 50),
        (b'Charlie', os.urandom(32), 10),
        (b'David',   os.urandom(32), 164),
        (b'Eve',     os.urandom(32), 870),
        (b'Fred',    os.urandom(32), 6),
        (b'Greta',   os.urandom(32), 270),
        (b'Henry',   os.urandom(32), 90),
    ]
    tree = build_merkle_sum_tree(user_table)
    root = get_root(tree)
    print("Root:", root)
    proof = get_proof(tree, 2)
    print("Proof:", proof)
    assert verify_proof(b'Charlie', user_table[2][1], 10, 2, 8, root, proof)
    print("Proof checked")

if __name__ == '__main__':
    test()
```

这种设计中的隐私泄漏比完全公开的列表要低得多，并且可以通过每次发布根时重新洗牌来进一步减少，但仍然存在一些隐私泄漏：Charlie 得知某人有 164 ETH的余额，一些两个用户的余额加起来高达 70 ETH，等等。控制许多帐户的攻击者仍然可能了解有关交易所用户的大量信息。

该计划的一个重要微妙之处在于可能出现负余额：如果一家拥有 1390 ETH 客户余额但只有 890 ETH 储备金的交易所试图通过在某处的虚假账户下添加 -500 ETH 余额来弥补差额，事实证明，这种可能性并没有破坏方案，尽管这就是我们特别需要 Merkle sum 树而不是常规 Merkle 树的原因。假设 Henry 是交易所控制的假账户，交易所将 -500 ETH 放在那里：

[![merkle_tree_2](images/merkle_tree_2.png)]([https://github.com/savour-labs](https://github.com/0xchaineye/chaineye-binance-por/))

Greta 的证明验证会失败：交易所必须给她 Henry 的 -500 ETH 节点，她会认为该节点无效而拒绝。Eve 和 Fred 的 proof 验证也会失败，因为 Henry 上面的中间节点总共有 -230 ETH，所以也是无效的！为了摆脱盗窃，交易所必须希望整个树的右半部分都没有人检查他们的余额证明。

如果交易所可以识别出价值 500 ETH 的用户，他们有信心不会费心去检查证明，或者当他们抱怨他们从未收到证明时不会被相信，他们就可以逃脱盗窃。但随后交易所也可以将这些用户从树中排除并产生相同的效果。因此，如果只以实现责任证明为目标，Merkle 树技术基本上与责任证明方案一样好。但其隐私属性仍不理想。你可以通过更聪明的方式使用默克尔树来更进一步，比如让每个 satoshi 或 wei 成为一个单独的叶子，但最终随着更现代的技术，会有更好的方法来做到这一点。

## 2.使用 ZK-SNARKs 改善隐私和稳健性

ZK-SNARKs 是一项强大的技术。ZK-SNARKs 之于密码学可能就像 Transformer 之于 AI：一种非常强大的通用技术，它将完全推动一大堆特定于应用程序的技术来解决几十年前开发的一大堆问题。因此，当然，我们可以使用 ZK-SNARKs 来极大地简化和改进责任证明协议中的隐私。

我们能做的最简单的事情就是将所有用户的存款放入一棵 Merkle 树中（或者更简单，一个KZG commitment），并使用 ZK-SNARK 来证明树中的所有余额都是非负的，加起来为一些声称的价值。如果我们为隐私添加一层散列，则提供给每个用户的 Merkle 分支（或 KZG 证明）将不会透露任何其他用户的余额。

[![merkle_tree_3](images/merkle_tree_3.png)]([https://github.com/savour-labs](https://github.com/0xchaineye/chaineye-binance-por/))

我们可以用一个专用的 ZK-SNARK 来证明上述 KZG 中余额的和和非负性。这是一个简单的示例方法来执行此操作。我们引入一个辅助多项式I(x)，它“建立了每个余额的位”（为了举例，我们假设余额低于 2^15
) 并且每第 16 个位置跟踪一个带有偏移量的运行总计，以便只有当实际总计与声明的总计相匹配时，它的总和才为零。如果 z 是 128 阶单位根，我们可以证明方程：


[![kzg](images/kzg.png)](https://github.com/0xchaineye/chaineye-binance-por/)

有效设置的第一个值 I(s)会是 `0 0 0 0` `0 0 0 0` `0 0 1 2` `5 10 20 -165` `0 0 0 0 ` `0 0 0 0` ` 0 1 3 6` `12 25 50 -300`...

只需几个额外的方程式，像这样的约束系统就可以适应更复杂的设置。例如，在杠杆交易系统中，个人用户的负余额是可以接受的，但前提是他们有足够的其他资产来覆盖具有一定抵押保证金的资金。SNARK 可以用来证明这个更复杂的约束，让用户放心，交易所不会通过秘密豁免其他用户遵守规则来冒他们的资金风险。

从长远来看，这种 ZK 负债证明可能不仅可以用于交易所的客户存款，还可以用于更广泛的借贷。任何获得贷款的人都会将记录放入包含该贷款的多项式或树中，并且该结构的根将在链上发布。这将使任何寻求贷款的人都可以向贷方证明他们尚未获得太多其他贷款。最终，法律创新甚至可以使以这种方式承诺的贷款比没有承诺的贷款具有更高的优先级。

## 3.资产证明

最简单的资产证明版本就是我们上面看到的协议：为了证明你持有 X 个币，你只需在某个预先约定的时间或在数据字段包含“这些资金属于”字样的交易中移动 X 个币到币安”。为了避免支付交易费用，您可以改为签署链下消息；比特币和以太坊都有链下签名消息的标准。

这种简单的资产证明技术存在两个实际问题：

- 冷库处理
- 抵押物两用
- 
出于安全原因，大多数交易所将大部分客户资金保存在“冷库”中：在离线计算机上，交易需要手动签署并转移到互联网上。字面上的气隙是常见的：我曾经用于个人资金的冷存储设置涉及一台永久离线的计算机生成一个包含签名交易的 QR 码，我会从我的手机上扫描它。由于涉及高价值，交易所使用的安全协议更加疯狂，并且通常涉及在多个设备之间使用多方计算，以进一步降低黑客攻击单个设备泄露密钥的可能性。鉴于这种设置，即使是制作一条额外的消息来证明对地址的控制也是一项昂贵的操作！

交易所可以采取多种途径：

保留几个公共长期使用地址。交易所会生成一些地址，对每个地址发布一次证明以证明所有权，然后重复使用这些地址。这是迄今为止最简单的选择，尽管它确实在如何保护安全和隐私方面增加了一些限制。
有很多地址，随机证明几个。交易所将有许多地址，甚至可能每个地址只使用一次并在单笔交易后退出。在这种情况下，交易所可能有一个协议，有时会随机选择一些地址，并且必须“打开”以证明所有权。一些交易所已经与审计员一起做这样的事情，但原则上这种技术可以变成一个完全自动化的程序。
更复杂的 ZKP 选项。例如，交易所可以将其所有地址设置为 1-of-2 多重签名，其中一个密钥每个地址不同，另一个是存储在一些复杂但非常安全的方式，例如。16 中的 12 多重签名。为了保护隐私并避免泄露其整个地址集，交易所甚至可以在区块链上运行零知识证明，证明链上所有具有这种格式的地址的总余额。
另一个主要问题是防止抵押品的双重用途。在彼此之间来回穿梭抵押品以进行准备金证明是交易所可以轻松做到的事情，并且可以让他们假装有偿付能力，而实际上却没有。理想情况下，偿付能力证明将实时完成，并在每个区块后更新证明。如果这不切实际，那么下一个最好的办法就是在不同的交易所之间按照固定的时间表进行协调，例如。在每周二 1400 UTC 时间探明储量。

最后一个问题是：你能在法币上进行资产证明吗？交易所不仅持有加密货币，它们还在银行系统内持有法定货币。在这里，答案是：是的，但这样的程序将不可避免地依赖于“法定”信任模型：银行本身可以证明余额，审计师可以证明资产负债表等。鉴于法定不可通过密码验证，这就是可以在该框架内完成的最好的，但它仍然值得做。

另一种方法是将一个运行交易所并处理 USDC 等资产支持稳定币的实体与另一个处理在加密货币和传统银行业务之间移动的现金进出流程的实体（USDC 本身）完全分开系统。因为 USDC 的“负债”只是链上 ERC20 代币，所以负债证明是“免费”的，只需要资产证明。

## 4.Plasma 和 validiums：我们可以让 CEX 成为非托管的吗？

假设我们想走得更远：我们不想仅仅证明交易所有资金来偿还用户。相反，我们要防止交易所完全窃取用户的资金。

对此的第一个主要尝试是Plasma，这是一种在 2017 年和 2018 年在以太坊研究圈流行的扩展解决方案。Plasma 的工作原理是将余额拆分为一组单独的“硬币”，其中每个硬币都分配有一个索引并存在于Plasma 区块的 Merkle 树中的特定位置。进行有效的代币转移需要将交易放入树的正确位置，树的根在链上发布。

[![plasmacash](images/plasmacash.png)]([https://github.com/savour-labs](https://github.com/0xchaineye/chaineye-binance-por/))
Plasma 一个版本的过度简化图。代币保存在智能合约中，该合约在提款时执行 Plasma 协议的规则。

OmiseGo 试图基于该协议进行去中心化交易，但从那时起他们就转向了其他想法——就此而言，Plasma Group 本身也是如此，它现在是乐观的 EVM 汇总项目Optimism。

不值得将 2018 年设想的 Plasma 的技术局限性（例如，证明硬币碎片整理）看作是关于整个概念的某种道德故事，这是不值得的。自 2018 年 Plasma 讨论达到顶峰以来，ZK-SNARKs 在与扩展相关的用例中变得更加可行，正如我们上面所说，ZK-SNARKs 改变了一切。

Plasma 想法的更现代版本是 Starkware 所说的validium：基本上与 ZK-rollup 相同，除了数据保存在链外。这种结构可以用于很多用例，可以想象任何中央服务器需要运行一些代码并证明它正在正确执行代码的地方。在 validium 中，运营商无法窃取资金，但根据实施的细节，如果运营商消失，一些用户资金可能会卡住。

这一切都非常好：远非 CEX 与 DEX 是二元对立的，事实证明，有各种各样的选择，包括各种形式的混合中心化，你可以获得一些好处，比如效率，但仍然有很多加密护栏来防止中心化运营商免于从事大多数形式的滥用行为。

[![spectrum](images/spectrum.png)]([https://github.com/savour-labs](https://github.com/0xchaineye/chaineye-binance-por/))

但是值得解决这个设计空间右半部分的基本问题：处理用户错误。到目前为止，最重要的错误类型是：如果用户忘记密码、丢失设备、被黑客入侵或无法访问其帐户怎么办？

交易所可以解决这个问题：首先是电子邮件恢复，如果仍然失败，则通过 KYC 进行更复杂的恢复。但为了能够解决此类问题，交易所需要实际控制硬币。为了能够出于正当理由收回用户账户的资金，交易所需要拥有也可以用于出于不良理由窃取用户账户资金的权力。这是一个不可避免的权衡。

理想的长期解决方案是依靠自我托管，在未来用户可以轻松访问多重签名和社交恢复钱包等技术来帮助处理紧急情况。但在短期内，有两种明确的替代方案，它们的成本和收益截然不同：

|选项	| 交易所风险	| 用户端风险 |
|:----:|----:|----:|
|托管交易所（例如今天的 Coinbase）|	交易所端出现问题，用户资金可能会丢失|	Exchange 可以帮助恢复帐户|
|非托管交易所（例如今天的 Uniswap）|	即使交易所恶意行为，用户也可以退出	|如果用户搞砸了，用户资金可能会丢失|

另一个重要的问题是跨链支持：交易所需要支持许多不同的链，像 Plasma 和 validiums 这样的系统需要用不同的语言编写代码以支持不同的平台，并且根本无法在其他（尤其是比特币）上实现他们目前的形式。从长远来看，这有望通过技术升级和标准化得到解决；然而，在短期内，这是支持托管交易所暂时保持托管状态的另一个论据。

## 5.结论：更好的交流的未来

短期内，交易所分为两个明确的“类别”：托管交易所和非托管交易所。今天，后一类只是 DEXes，例如 Uniswap，未来我们可能还会看到加密“受限”的 CEXes，其中用户资金保存在类似于 validium 智能合约的东西中。我们也可能会看到半托管交易所，我们用法定货币而不是加密货币来信任它们。

两种类型的交易所都将继续存在，而提高托管交易所安全性的最简单的向后兼容方式是添加储备证明。这包括资产证明和负债证明的组合。为两者制定良好的协议存在技术挑战，但我们可以而且应该尽可能地在两者上取得进展，并尽可能开源软件和流程，以便所有交易所都能受益。

从长远来看，我希望我们越来越接近于所有交易所都是非托管的，至少在加密方面是这样。钱包恢复将存在，并且可能需要为处理小额交易的新用户以及出于法律原因需要此类安排的机构提供高度集中的恢复选项，但这可以在钱包层而不是在交易所本身内完成. 在法定货币方面，传统银行系统和加密生态系统之间的移动可以通过 USDC 等资产支持的稳定币原生的进/出流程来完成。但是，我们还需要一段时间才能完全到达那里。








