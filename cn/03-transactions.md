# BOLT＃3：比特币交易和脚本格式

这详细说明了链上交易的确切格式，双方需要达成一致以确保签名有效。这包括资金交易输出脚本，承诺交易和HTLC交易。

# 目录

  * [交易](#transactions)
    * [交易输入和输出订购](#transaction-input-and-output-ordering)
    * [使用Segwit](#use-of-segwit)
    * [资金交易输出](#funding-transaction-output)
    * [承诺交易](#commitment-transaction)
        * [承诺交易输出](#commitment-transaction-outputs)
          * [`to_local`输出](#to_local-output)
          * [`to_remote`输出](#to_remote-output)
          * [提供HTLC输出](#offered-htlc-outputs)
          * [收到HTLC输出](#received-htlc-outputs)
        * [修剪输出](#trimmed-outputs)
    * [HTLC超时和HTLC成功事务](#htlc-timeout-and-htlc-success-transactions)
    * [结束交易](#closing-transaction)
    * [费用](#fees)
        * [费用计算](#fee-calculation)
        * [费用支付](#fee-payment)
  * [按键](#keys)
    * [关键衍生](#key-derivation)
        * [`localpubkey`，`remotepubkey`，`local_htlcpubkey`，`remote_htlcpubkey`，`local_delayedpubkey`和`remote_delayedpubkey`派生](#localpubkey-remotepubkey-local_htlcpubkey-remote_htlcpubkey-local_delayedpubkey-and-remote_delayedpubkey-derivation)
        * [`revocationpubkey`派生](#revocationpubkey-derivation)
        * [每承诺秘密要求](#per-commitment-secret-requirements)
    * [高效的每承诺秘密存储](#efficient-per-commitment-secret-storage)
  * [附录A：预期权重](#appendix-a-expected-weights)
      * [承诺交易的预期权重](#expected-weight-of-the-commitment-transaction)
      * [HTLC超时和HTLC成功事务的预期权重](#expected-weight-of-htlc-timeout-and-htlc-success-transactions)
  * [附录B：资金交易测试向量](#appendix-b-funding-transaction-test-vectors)
  * [附录C：承诺和HTLC交易测试向量](#appendix-c-commitment-and-htlc-transaction-test-vectors)
  * [附录D：按承诺秘密生成测试向量](#appendix-d-per-commitment-secret-generation-test-vectors)
    * [生成测试](#generation-tests)
    * [存储测试](#storage-tests)
  * [附录E：关键衍生测试向量](#appendix-e-key-derivation-test-vectors)
  * [参考](#references)
  * [作者](#authors)

# 交易

## 交易输入和输出订购

词典排序：见[BIP69]（https://github.com/bitcoin/bips/blob/master/bip-0069.mediawiki）。在相同的HTLC输出的情况下，输出按递增的“cltv_expiry”顺序排序。

## 合理

两个提供的HTLC具有相同的`amount_msat`和`payment_hash`
将具有相同的输出，即使它们的`cltv_expiry`不同。
这只是重要的，因为使用相同的顺序发送
`htlc_signatures`和HTLC交易本身不同，
因此，两个同行必须就此案件的规范排序达成一致。

## 使用Segwit

这里使用的大多数事务输出都是pay-to-witness-script-hash <sup> [BIP141]（https://github.com/bitcoin/bips/blob/master/bip-0141.mediawiki#witness-program)</sup>(P2WSH）输出：P2SH的Segwit版本。要花费此类输出，见证堆栈上的最后一项必须是用于生成正在花费的P2WSH输出的实际脚本。为简洁起见，本文档的其余部分省略了最后一项。

## 资金交易输出

* 资金输出脚本是P2WSH：

`2 <pubkey1> <pubkey2> 2 OP_CHECKMULTISIG`

* 其中`pubkey1`是两个DER编码的“funding_pubkey”中数字较小的，其中`pubkey2`在数字上大于两者。

## 承诺交易

* 版本：2
* 锁定时间：高8位是0x20，低24位是模糊承诺号的低24位
* txin数：1
   * `txin [0]`outpoint：来自`funding_created`消息的`txid`和`output_index`
   * `txin [0]`序列：高8位是0x80，低24位是模糊承诺号的高24位
   * `txin [0]`脚本字节：0
   * `txin [0]`见证：`0 <signature_for_pubkey1> <signature_for_pubkey2>`

48位承诺号被“XOR”模糊，低位为48位：

    SHA256（来自open_channel的payment_basepoint ||来自accept_channel的payment_basepoint）

这掩盖了在该频道上做出的承诺数量
单边收盘的情况，但仍为两者提供了有用的指数
节点（谁知道`payment_basepoint`s）快速找到被撤销的
承诺交易。

### 承诺交易输出

为了允许罚款交易的机会，在撤销的承诺交易的情况下，所有将资金返还给承诺交易的所有者（即“本地节点”）的输出必须延迟“to_self_delay”块。该延迟在第二阶段HTLC事务中完成（本地节点接受的HTLC的HTLC成功，本地节点提供的HTLC的HTLC超时）。

HTLC输出的单独事务处理阶段的原因是HTLC可以超时或满足，即使它们在'to_self_delay`延迟内。
否则，此延迟会延长HTLC所需的最小超时时间，从而导致HTLC穿越网络的时间更长。

每个输出的金额必须向下舍入到整个satoshis。如果此金额减去HTLC交易的费用，则小于承诺交易所有者设定的“dust_limit_satoshis”，则不得产生输出（因此资金增加了费用）。

#### `to_local`输出

此输出将资金发回给此承诺交易的所有者，因此必须使用“OP_CSV”进行时间锁定。如果他们知道撤销私钥，则可以立即声明另一方。输出是版本0 P2WSH，带有见证脚本：

    OP_IF
        #Penalty交易
         <revocationpubkey>
    OP_ELSE
        `to_self_delay`
        OP_CSV
        OP_DROP
         <local_delayedpubkey>
    OP_ENDIF
    OP_CHECKSIG

输出由`nSequence`字段设置为`to_self_delay`的事务处理（只能在该持续时间过后才有效）并见证：

     <local_delayedsig> 0

如果发布了已撤销的承诺交易，则另一方可以立即将此输出用于以下证人：

     <revocation_sig> 1

#### `to_remote`输出

此输出将资金发送给另一个对等方，因此是一个简单的P2WPKH到`remotepubkey`。

#### 提供HTLC输出

此输出将资金发送到HTLC超时后的HTLC超时事务，或使用支付原像或撤销密钥发送到远程节点。输出是一个P2WSH，带有见证脚本：

    ＃到具有吊销密钥的远程节点
    OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
    OP_IF
        OP_CHECKSIG
    OP_ELSE
         <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_NOTIF
            ＃通过HTLC超时事务（时间锁定）到本地节点。
            OP_DROP 2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            ＃到具有preimage的远程节点。
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF

远程节点可以使用见证人兑换HTLC：

     <remotehtlcsig> <payment_preimage>

如果发布了已撤销的承诺事务，则远程节点可以使用以下证据立即使用此输出：

     <revocation_sig> <revocationpubkey>

一旦HTLC过期，发送节点可以使用HTLC超时事务来超时HTLC，如下所示。

#### 收到HTLC输出

此输出在HTLC超时之后或使用撤销密钥将资金发送到远程节点，或者以成功的支付原像映射到HTLC成功事务。输出是一个P2WSH，带有见证脚本：

    ＃到具有吊销密钥的远程节点
    OP_DUP OP_HASH160 <RIPEMD160(SHA256(revocationpubkey))> OP_EQUAL
    OP_IF
        OP_CHECKSIG
    OP_ELSE
         <remote_htlcpubkey> OP_SWAP OP_SIZE 32 OP_EQUAL
        OP_IF
            ＃通过HTLC-success事务到本地节点。
            OP_HASH160 <RIPEMD160(payment_hash)> OP_EQUALVERIFY
            2 OP_SWAP <local_htlcpubkey> 2 OP_CHECKMULTISIG
        OP_ELSE
            ＃超时后到远程节点。
            OP_DROP <cltv_expiry> OP_CHECKLOCKTIMEVERIFY OP_DROP
            OP_CHECKSIG
        OP_ENDIF
    OP_ENDIF

要使HTLC超时，远程节点将与证人一起使用：

     <remotehtlcsig> 0

如果发布了已撤销的承诺事务，则远程节点可以使用以下证据立即使用此输出：

     <revocation_sig> <revocationpubkey>

要兑换HTLC，将使用HTLC成功交易，详情如下。

### 修剪输出

每个对等体指定一个“dust_limit_satoshis”，输出应该在其下面
不被生产;这些未产生的输出称为“修剪”。修剪后的输出是
被认为太小而不值得创建，而是被添加
承诺交易费。对于HTLC，需要加入
第二阶段HTLC交易的账户也可能低于
限制。

#### 要求

基本费用：
  - 在承诺交易产出确定之前：
    - 必须从`to_local`或`to_remote`中减去
    输出，如[费用计算]中所述（＃费用计算）。

承诺交易：
  - 如果承诺事务`to_local`输出的数量是
小于交易所有者设置的`dust_limit_satoshis`：
    - 绝不能包含那个输出。
  - 除此以外：
    - 必须按照[`to_local`输出]（＃to_local-output）中的指定生成。
  - 如果承诺事务`to_remote`输出的数量是多少
小于交易所有者设置的`dust_limit_satoshis`：
    - 绝不能包含那个输出。
  - 除此以外：
    - 必须按照[`to_remote`输出]（＃to_remote-output）中的指定生成。
  - 对于每个提供的HTLC：
    - 如果HTLC数量减去HTLC超时费用将小于
    交易所有者设置的`dust_limit_satoshis`：
      - 绝不能包含那个输出。
    - 除此以外：
      - 必须按照中的规定生成
      [提供的HTLC输出]（＃offers-htlc-outputs）。
  - 对于每个收到的HTLC：
    - 如果HTLC数量减去HTLC成功费用将小于
    交易所有者设置的`dust_limit_satoshis`：
      - 绝不能包含那个输出。
    - 除此以外：
      - 必须按照中的规定生成
      [收到的HTLC输出]（＃received-htlc-outputs）。

## HTLC-Timeout和HTLC-Success事务

除了HTLC超时事务是时间锁定之外，这些HTLC事务几乎相同。 HTLC超时/ HTLC成功事务都可以通过有效的惩罚事务来使用。

* 版本：2
* 锁定时间：HTLC-success为“0”，HTLC-timeout为“cltv_expiry”
* txin数：1
   * `txin [0]`outpoint：承诺事务的`txid`和HTLC事务的匹配HTLC输出的`output_index`
   * `txin [0]`序列：`0`
   * `txin [0]`脚本字节：`0`
   * `txin [0]`见证堆栈：`0 <remotehtlcsig> <localhtlcsig> <payment_preimage>`用于HTLC成功，`0 <remotehtlcsig> <localhtlcsig> 0`用于HTLC超时
* txout计数：1
   * `txout [0]`金额：HTLC金额减去费用（参见[费用计算]（＃费用计算））
   * `txout [0]`脚本：版本-0 P2WSH与见证脚本如下所示

输出的见证脚本是：

    OP_IF
        #Penalty交易
         <revocationpubkey>
    OP_ELSE
        `to_self_delay`
        OP_CSV
        OP_DROP
         <local_delayedpubkey>
    OP_ENDIF
    OP_CHECKSIG

为了通过惩罚来消费，远程节点使用见证堆栈“<revocationsig> 1”，并且为了收集输出，本地节点使用带有nSequence`to_self_delay`的输入和见证堆栈“<local_delayedsig> 0”。

## 结束交易

请注意，每个节点有两种可能的变体。

* 版本：2
* 锁定时间：0
* txin数：1
   * `txin [0]`outpoint：来自`funding_created`消息的`txid`和`output_index`
   * `txin [0]`序列：0xFFFFFFFF
   * `txin [0]`脚本字节：0
   * `txin [0]`见证：`0 <signature_for_pubkey1> <signature_for_pubkey2>`
* txout计数：0,1或2
   * `txout` amount：要支付给一个节点的最终余额（减去来自`closing_signed`的`fee_satoshis`，如果这个同伴资助了该频道）
   * `txout`脚本：在该节点的`scriptpubkey`中的`shutdown`消息中指定

### 要求

每个节点提供签名：
  - 必须将每个输出向下舍入到整个satoshis。
  - 必须从输出到资助者中减去“fee_satoshis”给出的费用。
  - 必须删除它自己的`dust_limit_satoshis`下面的任何输出。
  - 可以消除自己的输出。

### 合理

如果关闭，可能会出现无法弥补的差异
node认为对方的输出太小而无法传播
比特币网络（a.k.a。“尘埃”），而不是其他节点
认为输出太有价值而无法丢弃。这就是每个人的原因
side使用自己的`dust_limit_satoshis`，结果可以是a
签名验证失败，如果他们不同意结束什么
交易应该是这样的。

但是，如果一方选择消除自己的输出，那就没有了
另一方未能通过结案协议的原因;所以这是
明确允许。签名表示哪个变体
已经用过。

如果资金数额更大，将至少有一个产出
比两次`dust_limit_satoshis`。

## 费用

### 费用计算

承诺交易和HTLC的费用计算
交易基于当前的`feerate_per_kw`和
*交易的预期权重*。

实际和预期的重量因以下几个原因而异：

* 比特币使用DER编码的签名，其大小各不相同。
* 比特币也使用可变长度整数，因此大量输出将需要3个字节来编码而不是1。
* `to_remote`输出可能低于灰尘限制。
* 一旦提取费用，“to_local”输出可能低于粉尘限制。

因此，使用*预期权重*的简化公式，其假定：

* 签名长度为73个字节（最大长度）。
* 有少量输出（因此需要1个字节来计算它们）。
* 始终存在`to_local`输出和`to_remote`输出。

这产生以下*预期权重*（[附录A]中的计算细节（＃appendix-a-expected-weights））：

    承诺重量：724 + 172 * num-untrimmed-htlc-outputs
    HTLC超时重量：663
    HTLC-成功体重：703

请注意下面要求中承诺交易的“基本费用”的参考，这是资助者支付的。由于四舍五入和减少的输出，实际费用可能高于此处计算的数额。

#### 要求

HTLC超时交易的费用：
  - 必须计算匹配：
    1. 将`feerate_per_kw`乘以663并除以1000（四舍五入）。

HTLC成功交易的费用：
  - 必须计算匹配：
    1. 将`feerate_per_kw`乘以703并除以1000（四舍五入）。

承诺交易的基本费用：
  - 必须计算匹配：
    1. 从'weight` = 724开始。
    2. 对于每个已提交的HTLC，如果未按照指定修剪该输出
    [Trimmed Outputs]（＃trimmed-outputs），将172添加到`weight`。
    3. 将`feerate_per_kw`乘以'weight`，除以1000（四舍五入）。

#### 例

例如，假设有一个5000的'feerate_per_kw`，一个546 satoshis的`dust_limit_satoshis`，以及一个承诺交易：
* 两个提供5000000和1000000毫西的HTLC（5000和1000 satoshis）
* 两个收到7000000和800000毫瓦的HTLC（7000和800 satoshis）

HTLC超时事务“权重”是663，因此费用是3315 satoshis。
HTLC成功交易'权重'是703，因此费用是3515 satoshis

承诺交易“权重”计算如下：

* `weight`从724开始。

* 提供5000 satoshis的HTLC高于546 + 3315并导致：
  * 在承诺交易中输出5000 satoshi
  * 一个HTLC超时事务5000  -  3315 satoshis花费此输出
  * `weight`增加到896

* 提供1000 satoshis的HTLC低于546 + 3315所以它被修剪。

* 收到的7000 satoshis的HTLC高于546 + 3515，结果如下：
  * 在承诺交易中输出7000 satoshi
  * 一个HTLC成功交易7000  -  3515 satoshis花费此输出
  * `weight`增加到1068

* 收到的800 satoshis的HTLC低于546 + 3515所以它被修剪。

基本承诺交易费为5340 satoshi;实际上
费用（增加1000和800 satoshi HTLCs会产生灰尘
输出）是7140 satoshi。如果是，最终费用可能会更高
`to_local`或`to_remote`输出低于`dust_limit_satoshis`。

### 费用支付

基础承诺交易费用从出资者的金额中提取;如果该数量不足，则使用出资者输出的全部金额。

请注意，从to-funder输出中减去费用金额后，
输出可能低于`dust_limit_satoshis`，因此也是如此
有助于收费。

一个节点：
  - 如果产生的费率太低：
    - 可能会失败频道。

## 承诺交易建设

本节将前面的章节联系在一起以详细说明
构造一个对等体的承诺事务的算法：
鉴于同行的'dust_limit_satoshis`，当前的`feerate_per_kw`，
应付给每个同伴的金额（`to_local`和`to_remote`）等等
致力于HTLC：

1. 按指定初始化承诺事务输入和锁定时间
   在[承诺交易]（＃commitment-transaction）中。
1. 计算需要修剪哪些已提交的HTLC（参见[修剪输出]（＃trimmed-outputs））。
2. 计算基数[承诺交易费]（＃费用计算）。
3. 从资助者（“to_local”或“to_remote”）中减去这笔基本费用，
   楼层为0（见[费用支付]（＃付费））。
3. 对于每个提供的HTLC，如果没有修剪，添加一个
   [提供HTLC输出]（＃offers-htlc-outputs）。
4. 对于每个收到的HTLC，如果没有修剪，添加一个
   [收到HTLC输出]（＃received-htlc-outputs）。
5. 如果`to_local`数量大于或等于`dust_limit_satoshis`，
   添加[`to_local`输出]（＃to_local-output）。
6. 如果`to_remote`数量大于或等于`dust_limit_satoshis`，
   添加[`to_remote`输出]（＃to_remote-output）。
7. 将输出分为[BIP 69 + CLTV顺序]（＃transaction-input-and-output-ordering）。

# 按键

## 关键衍生

每个承诺事务使用一组唯一的键：`localpubkey`和`remotepubkey`。
HTLC-success和HTLC-timeout事务使用`local_delayedpubkey`和`revocationpubkey`。
基于`per_commitment_point`为每个事务更改这些。

关键变化的原因是无法看到被撤销的
交易可以外包。这样的_watcher_应该不能
确定承诺交易的内容 - 即使_watcher_知道
要注意哪个交易ID，可以做出合理的猜测
关于哪些HTLC和余额可能包括在内。尽管如此，去
避免存储每个承诺交易，_watcher_可以给出
`per_commitment_secret`值（可以紧凑地存储）和
`revocation_basepoint`和`delayed_payment_basepoint`用于重新生成
罚款交易所需的脚本;因此，_watcher_只需要
给出（并存储）每个惩罚输入的签名。

每次更改`localpubkey`和`remotepubkey`都可以确保承诺
交易ID无法猜到;每个承诺交易都使用一个ID
在其输出脚本中。拆分`local_delayedpubkey`，这是必需的
惩罚事务，允许它与_watcher_共享而不用
揭示`localpubkey`;即使两个对等体使用相同的_watcher_，也没有任何显示。

最后，即使在正常单侧闭合的情况下，HTLC-成功
和/或HTLC超时事务不会泄露任何内容
_watcher_，因为它不知道相应的`per_commitment_secret`和
不能将`local_delayedpubkey`或`revocationpubkey`与它们的基数联系起来。

为了提高效率，密钥是根据一系列每个承诺的秘密生成的
从单个种子生成的，允许接收器紧凑
存储它们（见[下面]（#efficient-per-commitment-secret-storage））。

### `localpubkey`，`remotepubkey`，`local_htlcpubkey`，`remote_htlcpubkey`，`local_delayedpubkey`和`remote_delayedpubkey`派生

这些pubkeys只是通过从基点添加来生成：

    pubkey = basepoint + SHA256（per_commitment_point || basepoint）* G.

`localpubkey`使用本地节点的'payment_basepoint`;
`remotepubkey`使用远程节点的'payment_basepoint`;
`local_htlcpubkey`使用本地节点的`htlc_basepoint`;
`remote_htlcpubkey`使用远程节点的`htlc_basepoint`;
`local_delayedpubkey`使用本地节点的`delayed_payment_basepoint`;
并且`remote_delayedpubkey`使用远程节点的`delayed_payment_basepoint`。

如果是基点，则可以类似地导出相应的私钥
秘密是已知的（即只对应于`localpubkey`，`local_htlcpubkey`和`local_delayedpubkey`的私钥）：

    privkey = basepoint_secret + SHA256（per_commitment_point || basepoint）

### `revocationpubkey`派生

`revocationpubkey`是一个盲目的键：本地节点希望创建一个新的
对远程节点的承诺，它使用自己的`revocation_basepoint`和远程节点
node的`per_commitment_point`为它派生一个新的`revocationpubkey`
承诺。远程节点显示后
使用`per_commitment_secret`（从而撤销该承诺），本地节点
然后可以导出`revocationprivkey`，因为它现在知道这两个秘密
导出密钥所必需的（`revocation_basepoint_secret`和
`per_commitment_secret`）。

`per_commitment_point`是使用椭圆曲线乘法生成的：

    per_commitment_point = per_commitment_secret * G.

这用于从远程节点派生revocation pubkey
`revocation_basepoint`：

    revocationpubkey = revocation_basepoint * SHA256（revocation_basepoint || per_commitment_point）+ per_commitment_point * SHA256（per_commitment_point || revocation_basepoint）

这种结构确保了节点都不提供
basepoint和提供`per_commitment_point`的节点都可以知道
私钥没有其他节点的秘密。

一旦`per_commitment_secret`就可以导出相应的私钥
众所周知：

    revocationprivkey = revocation_basepoint_secret * SHA256（revocation_basepoint || per_commitment_point）+ per_commitment_secret * SHA256（per_commitment_point || revocation_basepoint）

### 每承诺秘密要求

一个节点：
  - 必须为每个连接选择一个不可插入的256位种子，
  - 绝不能透露种子。

可以生成最多（2 ^ 48-1）个承诺秘密。

使用的第一个秘密：
  - 必须是索引281474976710655，
    - 从那里，索引递减。

我的秘密P：
  - 必须匹配此算法的输出：
```
generate_from_seed(seed, I):
    P = seed
    for B in 47 down to 0:
        if B set in I:
            flip(B) in P
            P = SHA256(P)
    return P
```

其中“flip（B）”交替值P中的第B个最低有效位。

接收节点：
  - 可以存储所有以前的每个承诺的秘密。
  - 可以从紧凑的表示中计算它们，如下所述。

## 高效的每承诺秘密存储

一系列秘密的接收器可以紧凑地存储它们
49个（值，索引）对的数组。因为，对于给定的秘密
2 ^ X边界，可以导出到下一个2 ^ X边界的所有秘密;
并且始终以降序开始接收秘密
`0xFFFFFFFFFFFF`。

在二进制文件中，根据*前缀*来考虑任何索引是有帮助的，
接着是一些尾随0。你可以为任何人获得秘密
与此*前缀*匹配的索引。

例如，秘密“0xFFFFFFFFFFF0”允许导出秘密
`0xFFFFFFFFFFF1`到'0xFFFFFFFFFFFF`，包括在内;和秘密`0xFFFFFFFFFF08`
允许从'0xFFFFFFFFFF09`到'0xFFFFFFFFFF0F`导出秘密，
包括的。

这是通过上面的`generate_from_seed`的略微概括来完成的：

    ＃给我的秘密给出基本秘密，其索引有bit..47相同。
    derive_secret（base，bits，I）：
        P =基数
        对于B的位 -  1到0：
            如果B在I中设置：
                在P中翻转（B）
                P = SHA256（P）
        返回P.

每个唯一前缀只需要保存一个秘密;实际上，数量
计算尾随0，这决定了存储阵列中的位置
秘密存储：

    ＃a.k.a.计数尾随0
    where_to_put_secret（I）：
        对于0到47的B：
            如果在B == 1的testbit（I）：
                返回B.
        #I = 0，这是种子。
        返回48

需要仔细检查，以前所有秘密都是正确得出的;
如果此检查失败，则不会从同一种子生成秘密：

    insert_secret（secret，I）：
        B = where_to_put_secret（秘密，I）

        ＃它跟踪遍历遍历的每个桶中的秘密索引。
        对于0到B中的b：
            如果derive_secret（秘密，B，已知[b] .index）！=已知[b]。秘密：
                错误我的秘密是不正确的
                返回

        ＃假设这会根据需要自动扩展已知的[]。
        已知[B] .index = I
        已知[B] .secret = secret

最后，如果需要导出索引“I”的未知秘密，那么它必须是
发现可以使用哪个已知秘密来导出它。最简单的
方法迭代所有已知的秘密，并测试每个秘密
可用于导出未知秘密：

    derive_old_secret（I）：
        对于b在0到len（秘密）：
            ＃掩盖索引的非零前缀。
            MASK =〜（（1 << b） -  1）
            if（I＆MASK）== secrets [b] .index：
                return derive_secret（known，i，I）
        错误索引'我'尚未收到。

这看起来很复杂，但请记住条目`b`中的索引
`b`尾随0;掩码和比较只是检查索引
在每个桶中是所需索引的前缀。

# 附录A：预期权重

## 承诺交易的预期权重

承诺交易的*预期权重*计算如下：

    p2wsh：34个字节
         -  OP_0：1个字节
         -  OP_DATA：1个字节（witness_script_SHA256长度）
         -  witness_script_SHA256：32个字节

    p2wpkh：22个字节
         -  OP_0：1个字节
         -  OP_DATA：1个字节（public_key_HASH160长度）
         -  public_key_HASH160：20个字节

    multi_sig：71个字节
         -  OP_2：1个字节
         -  OP_DATA：1个字节（pub_key_alice长度）
         -  pub_key_alice：33个字节
         -  OP_DATA：1个字节（pub_key_bob长度）
         -  pub_key_bob：33个字节
         -  OP_2：1个字节
         -  OP_CHECKMULTISIG：1个字节

    见证：222个字节
         -  number_of_witness_elements：1个字节
         -  nil_length：1个字节
         -  sig_alice_length：1个字节
         -  sig_alice：73个字节
         -  sig_bob_length：1个字节
         -  sig_bob：73个字节
         -  witness_script_length：1个字节
         -  witness_script（multi_sig）

    funding_input：41个字节
         -  previous_out_point：36个字节
             - 哈希：32个字节
             - 索引：4个字节
         -  var_int：1个字节（script_sig长度）
         -  script_sig：0个字节
         - 见证<----“见证”代替“script_sig”
                交易验证;然而，“见证”被存储
                单独，其尺寸的成本较小。所以，
                普通数据的计算是分开的
                来自证人数据。
         - 序列：4个字节

    output_paying_to_local：43个字节
         - 值：8个字节
         -  var_int：1个字节（pk_script长度）
         -  pk_script（p2wsh）：34个字节

    output_paying_to_remote：31个字节
         - 值：8个字节
         -  var_int：1个字节（pk_script长度）
         -  pk_script（p2wpkh）：22个字节

     htlc_output：43个字节
         - 值：8个字节
         -  var_int：1个字节（pk_script长度）
         -  pk_script（p2wsh）：34个字节

     witness_header：2个字节
         - 标志：1个字节
         - 标记：1个字节

     commitment_transaction：125 + 43 * num-htlc-outputs字节
         - 版本：4个字节
         -  witness_header <----见证数据的一部分
         -  count_tx_in：1个字节
         -  tx_in：41个字节
            funding_input
         -  count_tx_out：1个字节
         -  tx_out：74 + 43 * num-htlc-outputs字节
            output_paying_to_remote，
            output_paying_to_local，
            .... htlc_output的...
         -  lock_time：4个字节

将非见证数据乘以4会导致权重：

    // 500 + 172 * num-htlc-输出重量
    commitment_transaction_weight = 4 * commitment_transaction

    // 224重量
    witness_weight = witness_header +见证

    overall_weight = 500 + 172 * num-htlc-outputs + 224 weight

## HTLC超时和HTLC成功事务的预期权重

HTLC交易的*预期权重*计算如下：

    accepted_htlc_script：139个字节
         -  OP_DUP：1个字节
         -  OP_HASH160：1个字节
         -  OP_DATA：1个字节（RIPEMD160（SHA256（revocationpubkey））长度）
         -  RIPEMD160（SHA256（revocationpubkey））：20个字节
         -  OP_EQUAL：1个字节
         -  OP_IF：1个字节
         -  OP_CHECKSIG：1个字节
         -  OP_ELSE：1个字节
         -  OP_DATA：1个字节（remotepubkey长度）
         -  remotepubkey：33个字节
         -  OP_SWAP：1个字节
         -  OP_SIZE：1个字节
         -  OP_DATA：1个字节（32个长度）
         -  32：1字节
         -  OP_EQUAL：1个字节
         -  OP_IF：1个字节
         -  OP_HASH160：1个字节
         -  OP_DATA：1个字节（RIPEMD160（payment_hash）长度）
         -  RIPEMD160（payment_hash）：20个字节
         -  OP_EQUALVERIFY：1个字节
         -  2：1字节
         -  OP_SWAP：1个字节
         -  OP_DATA：1个字节（localpubkey长度）
         -  localpubkey：33个字节
         -  2：1字节
         -  OP_CHECKMULTISIG：1个字节
         -  OP_ELSE：1个字节
         -  OP_DROP：1个字节
         -  OP_DATA：1个字节（cltv_expiry长度）
         -  cltv_expiry：3个字节
         -  OP_CHECKLOCKTIMEVERIFY：1个字节
         -  OP_DROP：1个字节
         -  OP_CHECKSIG：1个字节
         -  OP_ENDIF：1个字节
         -  OP_ENDIF：1个字节

    provided_htlc_script：133个字节
         -  OP_DUP：1个字节
         -  OP_HASH160：1个字节
         -  OP_DATA：1个字节（RIPEMD160（SHA256（revocationpubkey））长度）
         -  RIPEMD160（SHA256（revocationpubkey））：20个字节
         -  OP_EQUAL：1个字节
         -  OP_IF：1个字节
         -  OP_CHECKSIG：1个字节
         -  OP_ELSE：1个字节
         -  OP_DATA：1个字节（remotepubkey长度）
         -  remotepubkey：33个字节
         -  OP_SWAP：1个字节
         -  OP_SIZE：1个字节
         -  OP_DATA：1个字节（32个长度）
         -  32：1字节
         -  OP_EQUAL：1个字节
         -  OP_NOTIF：1个字节
         -  OP_DROP：1个字节
         -  2：1字节
         -  OP_SWAP：1个字节
         -  OP_DATA：1个字节（localpubkey长度）
         -  localpubkey：33个字节
         -  2：1字节
         -  OP_CHECKMULTISIG：1个字节
         -  OP_ELSE：1个字节
         -  OP_HASH160：1个字节
         -  OP_DATA：1个字节（RIPEMD160（payment_hash）长度）
         -  RIPEMD160（payment_hash）：20个字节
         -  OP_EQUALVERIFY：1个字节
         -  OP_CHECKSIG：1个字节
         -  OP_ENDIF：1个字节
         -  OP_ENDIF：1个字节

    timeout_witness：285个字节
         -  number_of_witness_elements：1个字节
         -  nil_length：1个字节
         -  sig_alice_length：1个字节
         -  sig_alice：73个字节
         -  sig_bob_length：1个字节
         -  sig_bob：73个字节
         -  nil_length：1个字节
         -  witness_script_length：1个字节
         -  witness_script（provided_htlc_script）

    success_witness：325个字节
         -  number_of_witness_elements：1个字节
         -  nil_length：1个字节
         -  sig_alice_length：1个字节
         -  sig_alice：73个字节
         -  sig_bob_length：1个字节
         -  sig_bob：73个字节
         -  preimage_length：1个字节
         -  preimage：32个字节
         -  witness_script_length：1个字节
         -  witness_script（accepted_htlc_script）

    commitment_input：41个字节
         -  previous_out_point：36个字节
             - 哈希：32个字节
             - 索引：4个字节
         -  var_int：1个字节（script_sig长度）
         -  script_sig：0个字节
         - 见证（success_witness或timeout_witness）
         - 序列：4个字节

    htlc_output：43个字节
         - 值：8个字节
         -  var_int：1个字节（pk_script长度）
         -  pk_script（p2wsh）：34个字节

    htlc_transaction：
         - 版本：4个字节
         -  witness_header <----见证数据的一部分
         -  count_tx_in：1个字节
         -  tx_in：41个字节
            commitment_input
         -  count_tx_out：1个字节
         -  tx_out：43
            htlc_output
         -  lock_time：4个字节

将非见证数据乘以4会导致权重为376.添加
每个案例的见证数据（HTLC超时为285 + 2，对于HTLC超时为325 + 2）
HTLC成功）导致权重：

    663（HTLC超时）
    703（HTLC成功）

# 附录B：资金交易测试向量

在下面的：
 - 假设* local *是资助者。
 - 私钥显示为32字节加上尾随1（比特币的“压缩”私钥约定，即公钥被压缩的密钥）。
 - 使用RFC6979（使用HMAC-SHA256），事务签名都是确定性的。除非另有明确说明，否则有效签名必须签署相关交易的所有输入和输出（即必须使用`SIGHASH_ALL` [签名哈希]（https://bitcoin.org/en/glossary/signature-hash）创建） 。请注意，客户端必须以紧凑编码方式发送签名，而不是以比特币脚本格式发送签名，因此不会传输签名散列字节。

资金交易的输入是使用测试链创建的
以下前两个街区;第二个块包含一个可花费的
coinbase（请注意，这样的P2PKH输入是不可取的，详见[BOLT＃2]（02-peer-protocol.md #the- funding_created-message），但提供了最简单的例子）：

    块0（起源）：0100000000000000000000000000000000000000000000000000000000000000000000003ba3edfd7a7b12b27ac72c3e67768f617fc81bc3888a51323a9fb8aa4b1e5e4adae5494dffff7f20020000000101000000010000000000000000000000000000000000000000000000000000000000000000ffffffff4d04ffff001d0104455468652054696d65732030332f4a616e2f32303039204368616e63656c6c6f72206f6e206272696e6b206f66207365636f6e64206261696c6f757420666f722062616e6b73ffffffff0100f2052a01000000434104678afdb0fe5548271967f1a67130b7105cd6a828e03909a67962e0ea1f61deb649f6bc3f4cef38c4f35504e51ec112de5c384df7ba0b8d578a4c702b6bf11d5fac00000000
    块1：0000002006226e46111a0b59caaf126043eb5bbf28c34f3a5e332a1fc7b2b73cf188910fadbb20ea41a8423ea937e76e8151636bf6093b70eaff942930d20576600521fdc30f9858ffff7f20000000000101000000010000000000000000000000000000000000000000000000000000000000000000ffffffff03510101ffffffff0100f2052a010000001976a9143ca33c2e4446f4a305f23c80df8ad1afdcf652f988ac00000000
    Block 1 coinbase事务：010000000100000000000000000000000000000000000000000000000000000000000000000000ffffffff03510101ffffffff0100f2052a010000001976a9143ca33c2e4446f4a305f23c80df8ad1afdcf652f988ac00000000
    Block 1 coinbase privkey：6bd078650fcee8444e4e09825227b801a1ca928debb750eb36e6d56124bb20e801
    base58中的#privkey：cRCH7YNcarfvaiY1GWUKQrRGmoezvfAiqHtdRvxe16shzbd7LDMz
    base68中的#pubkey：mm3aPLSv9fBrbS68JzurAMp4xGoddJ6pSf

资助交易支付给以下公交：

    local_funding_pubkey：023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb
    remote_funding_pubkey：030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c1
    ＃funding witness script = 5221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae

资金交易有一个输入和一个变更输出（订单
在这种情况下由BIP69确定）：

    输入txid：fd2105607605d2302994ffea703b09f66b6351816ee737a93e42a841ea20bbad
    输入索引：0
    输入satoshis：5000000000
    资金satoshis：10000000
    ＃funding witness script = 5221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae
    #fifrate_per_kw：15000
    改变satoshis：4989986080
    资金产出：0

由此产生的资金交易是：

    资金TX：0200000001adbb20ea41a8423ea937e76e8151636bf6093b70eaff942930d20576600521fd000000006b48304502210090587b6201e166ad6af0227d3036a9454223d49a1f11839c1a362184340ef0240220577f7cd5cca78719405cbf1de7414ac027f0239ef6e214c90fcaab0454d84b3b012103535b32d5eb0a6ed0982a0479bbadc9868d9836f6ba94dd5a63be16d875069184ffffffff028096980000000000220020c015c4a6be010e21657068fc2e6a9d02b27ebe4d490a25846f7237f104d1a3cd20256d29010000001600143ca33c2e4446f4a305f23c80df8ad1afdcf652f900000000
    ＃txid：8984484a580b825b9972d7adb15050b3ab624ccd731946b3eeddb92f4e7ef6be

# 附录C：承诺和HTLC交易测试向量

在下面的：
 - *考虑本地*交易，这意味着所有对* local *的付款都会延迟。
 - 假设* local *是资助者。
 - 私钥显示为32字节加上尾随1（比特币的“压缩”私钥约定，即公钥被压缩的密钥）。
 - 使用RFC6979（使用HMAC-SHA256），事务签名都是确定性的。

首先，定义每个测试向量的常用基本参数：
HTLC不用于第一个“没有HTLCs的简单承诺tx”测试。

    funding_tx_id：8984484a580b825b9972d7adb15050b3ab624ccd731946b3eeddb92f4e7ef6be
    funding_output_index：0
    funding_amount_satoshi：10000000
    承诺号码：42
    local_delay：144
    local_dust_limit_satoshi：546
    htlc 0方向：remote-> local
    htlc 0 amount_msat：1000000
    htlc 0到期：500
    htlc 0 payment_preimage：0000000000000000000000000000000000000000000000000000000000000000
    htlc 1方向：remote-> local
    htlc 1 amount_msat：2000000
    htlc 1到期：501
    htlc 1 payment_preimage：010101010101010101010101010101010101010101010101010101010101010101
    htlc 2方向：local-> remote
    htlc 2 amount_msat：2000000
    htlc 2到期：502
    htlc 2 payment_preimage：020202020202020202020202020202020202020202020202020202020202020202
    htlc 3方向：local-> remote
    htlc 3 amount_msat：3000000
    htlc 3到期：503
    htlc 3 payment_preimage：0303030303030303030303030303030303030303030303030303030303030303
    htlc 4方向：remote-> local
    htlc 4 amount_msat：4000000
    htlc 4到期日：504
    htlc 4 payment_preimage：040404040404040404040404040404040404040404040404040404040404040404

<！ - 根据Key Derivation导出测试向量值，尽管不是
     这个测试需要。它们包含在这里是为了完整性和
     万一有人想要自己重现测试载体：

内部：remote_funding_privkey：1552dfba4f6cf29a62a0af13c8d6981d36d0ef8d61ba10fb0fe90da7634d7e1301
内部：local_payment_basepoint_secret：111111111111111111111111111111111111111111111111111111111111111101
内部：remote_revocation_basepoint_secret：2222222222222222222222222222222222222222222221
内部：local_delayed_payment_basepoint_secret：333333333333333333333333333333333333333333333333333333333333333301
内部：remote_payment_basepoint_secret：444444444444444444444444444444444444444444444444444444444444444444444443401
x_local_per_commitment_secret：1f1e1d1c1b1a191817161514131211100f0e0d0c0b0a0908070605040302010001
＃来自remote_revocation_basepoint_secret
内部：remote_revocation_basepoint：02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
＃来自local_delayed_payment_basepoint_secret
内部：local_delayed_payment_basepoint：023c72addb4fdf09af94f0c94d7fe92a386a7e70cf8a1d85916386bb2535c7b1b1
内部：local_per_commitment_point：025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486
内部：remote_privkey：8deba327a7cc6d638ab0eb025770400a6184afcba6713c210d8d10e199ff2fda01
＃来自local_delayed_payment_basepoint_secret，local_per_commitment_point和local_delayed_payment_basepoint
内部：local_delayed_privkey：adf3464ce9c2f230fd2582fda4c6965e4993ca5524e8c9580e3df0cf226981ad01
 - >

以下是用于推导承诺编号的模糊因子的点：

    local_payment_basepoint：034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    remote_payment_basepoint：032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991
    #camscured commitment number = 0x2bb038521914 ^ 42

而且，以下是创建交易所需的密钥：

    local_funding_privkey：30ff4956bbdd3222d44cc5e8a1261dab1e07957bdac5ae88fe3261ef321f374901
    local_funding_pubkey：023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb
    remote_funding_pubkey：030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c1
    local_privkey：bb13b121cdc357cd2e608b0aea294afca36e2b34cf958e2e6451a2f27469449101
    localpubkey：030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e7
    remotepubkey：0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b
    local_delayedpubkey：03fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c
    local_revocation_pubkey：0212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b19
    ＃funding wscript = 5221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae

而且，以下是测试向量本身：

    name: simple commitment tx with no HTLCs
    to_local_msat: 7000000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 15000
    # base commitment transaction fee = 10860
    # actual commitment transaction fee = 10860
    # to_local amount 6989140 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3045022100f51d2e566a70ba740fc5d8c0f07b9b93d2ed741c3c0860c613173de7d39e7968022041376d520e9c0e1ad52248ddf4b22e12be8763007df977253ef45a4ca3bdb7c0
    # local_signature = 3044022051b75c73198c6deee1a875871c3961832909acd297c6b908d59e3319e5185a46022055c419379c5051a78d00dbbce11b5b664a0c22815fbcc6fcef6b1937c3836939
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8002c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de84311054a56a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400473044022051b75c73198c6deee1a875871c3961832909acd297c6b908d59e3319e5185a46022055c419379c5051a78d00dbbce11b5b664a0c22815fbcc6fcef6b1937c383693901483045022100f51d2e566a70ba740fc5d8c0f07b9b93d2ed741c3c0860c613173de7d39e7968022041376d520e9c0e1ad52248ddf4b22e12be8763007df977253ef45a4ca3bdb7c001475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

    name: commitment tx with all five HTLCs untrimmed (minimum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 0
    # base commitment transaction fee = 0
    # actual commitment transaction fee = 0
    # HTLC 2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 0 received amount 1000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac6868
    # HTLC 1 received amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6988000 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 304402204fd4928835db1ccdfc40f5c78ce9bd65249b16348df81f0c44328dcdefc97d630220194d3869c38bc732dd87d13d2958015e2fc16829e74cd4377f84d215c0b70606
    # local_signature = 30440220275b0c325a5e9355650dc30c0eccfbc7efb23987c24b556b9dfdd40effca18d202206caceb2c067836c51f296740c7ae807ffcbfbf1dd3a0d56b6de9a5b247985f06
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8007e80300000000000022002052bfef0479d7b293c27e0f1eb294bea154c63a3294ef092c19af51409bce0e2ad007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de843110e0a06a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e04004730440220275b0c325a5e9355650dc30c0eccfbc7efb23987c24b556b9dfdd40effca18d202206caceb2c067836c51f296740c7ae807ffcbfbf1dd3a0d56b6de9a5b247985f060147304402204fd4928835db1ccdfc40f5c78ce9bd65249b16348df81f0c44328dcdefc97d630220194d3869c38bc732dd87d13d2958015e2fc16829e74cd4377f84d215c0b7060601475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 5
    # signature for output 0 (HTLC 0)
    remote_htlc_signature = 304402206a6e59f18764a5bf8d4fa45eebc591566689441229c918b480fb2af8cc6a4aeb02205248f273be447684b33e3c8d1d85a8e0ca9fa0bae9ae33f0527ada9c162919a6
    # signature for output 1 (HTLC 2)
    remote_htlc_signature = 3045022100d5275b3619953cb0c3b5aa577f04bc512380e60fa551762ce3d7a1bb7401cff9022037237ab0dac3fe100cde094e82e2bed9ba0ed1bb40154b48e56aa70f259e608b
    # signature for output 2 (HTLC 1)
    remote_htlc_signature = 304402201b63ec807771baf4fdff523c644080de17f1da478989308ad13a58b51db91d360220568939d38c9ce295adba15665fa68f51d967e8ed14a007b751540a80b325f202
    # signature for output 3 (HTLC 3)
    remote_htlc_signature = 3045022100daee1808f9861b6c3ecd14f7b707eca02dd6bdfc714ba2f33bc8cdba507bb182022026654bf8863af77d74f51f4e0b62d461a019561bb12acb120d3f7195d148a554
    # signature for output 4 (HTLC 4)
    remote_htlc_signature = 304402207e0410e45454b0978a623f36a10626ef17b27d9ad44e2760f98cfa3efb37924f0220220bd8acd43ecaa916a80bd4f919c495a2c58982ce7c8625153f8596692a801d
    # local_signature = 304402207cb324fa0de88f452ffa9389678127ebcf4cabe1dd848b8e076c1a1962bf34720220116ed922b12311bd602d67e60d2529917f21c5b82f25ff6506c0f87886b4dfd5
    output htlc_success_tx 0: 020000000001018154ecccf11a5fb56c39654c4deb4d2296f83c69268280b94d021370c94e219700000000000000000001e8030000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402206a6e59f18764a5bf8d4fa45eebc591566689441229c918b480fb2af8cc6a4aeb02205248f273be447684b33e3c8d1d85a8e0ca9fa0bae9ae33f0527ada9c162919a60147304402207cb324fa0de88f452ffa9389678127ebcf4cabe1dd848b8e076c1a1962bf34720220116ed922b12311bd602d67e60d2529917f21c5b82f25ff6506c0f87886b4dfd5012000000000000000000000000000000000000000000000000000000000000000008a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac686800000000
    # local_signature = 3045022100c89172099507ff50f4c925e6c5150e871fb6e83dd73ff9fbb72f6ce829a9633f02203a63821d9162e99f9be712a68f9e589483994feae2661e4546cd5b6cec007be5
    output htlc_timeout_tx 2: 020000000001018154ecccf11a5fb56c39654c4deb4d2296f83c69268280b94d021370c94e219701000000000000000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100d5275b3619953cb0c3b5aa577f04bc512380e60fa551762ce3d7a1bb7401cff9022037237ab0dac3fe100cde094e82e2bed9ba0ed1bb40154b48e56aa70f259e608b01483045022100c89172099507ff50f4c925e6c5150e871fb6e83dd73ff9fbb72f6ce829a9633f02203a63821d9162e99f9be712a68f9e589483994feae2661e4546cd5b6cec007be501008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
    # local_signature = 3045022100def389deab09cee69eaa1ec14d9428770e45bcbe9feb46468ecf481371165c2f022015d2e3c46600b2ebba8dcc899768874cc6851fd1ecb3fffd15db1cc3de7e10da
    output htlc_success_tx 1: 020000000001018154ecccf11a5fb56c39654c4deb4d2296f83c69268280b94d021370c94e219702000000000000000001d0070000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402201b63ec807771baf4fdff523c644080de17f1da478989308ad13a58b51db91d360220568939d38c9ce295adba15665fa68f51d967e8ed14a007b751540a80b325f20201483045022100def389deab09cee69eaa1ec14d9428770e45bcbe9feb46468ecf481371165c2f022015d2e3c46600b2ebba8dcc899768874cc6851fd1ecb3fffd15db1cc3de7e10da012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
    # local_signature = 30440220643aacb19bbb72bd2b635bc3f7375481f5981bace78cdd8319b2988ffcc6704202203d27784ec8ad51ed3bd517a05525a5139bb0b755dd719e0054332d186ac08727
    output htlc_timeout_tx 3: 020000000001018154ecccf11a5fb56c39654c4deb4d2296f83c69268280b94d021370c94e219703000000000000000001b80b0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100daee1808f9861b6c3ecd14f7b707eca02dd6bdfc714ba2f33bc8cdba507bb182022026654bf8863af77d74f51f4e0b62d461a019561bb12acb120d3f7195d148a554014730440220643aacb19bbb72bd2b635bc3f7375481f5981bace78cdd8319b2988ffcc6704202203d27784ec8ad51ed3bd517a05525a5139bb0b755dd719e0054332d186ac0872701008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 30440220549e80b4496803cbc4a1d09d46df50109f546d43fbbf86cd90b174b1484acd5402205f12a4f995cb9bded597eabfee195a285986aa6d93ae5bb72507ebc6a4e2349e
    output htlc_success_tx 4: 020000000001018154ecccf11a5fb56c39654c4deb4d2296f83c69268280b94d021370c94e219704000000000000000001a00f0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402207e0410e45454b0978a623f36a10626ef17b27d9ad44e2760f98cfa3efb37924f0220220bd8acd43ecaa916a80bd4f919c495a2c58982ce7c8625153f8596692a801d014730440220549e80b4496803cbc4a1d09d46df50109f546d43fbbf86cd90b174b1484acd5402205f12a4f995cb9bded597eabfee195a285986aa6d93ae5bb72507ebc6a4e2349e012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with seven outputs untrimmed (maximum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 647
    # base commitment transaction fee = 1024
    # actual commitment transaction fee = 1024
    # HTLC 2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 0 received amount 1000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac6868
    # HTLC 1 received amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6986976 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3045022100a5c01383d3ec646d97e40f44318d49def817fcd61a0ef18008a665b3e151785502203e648efddd5838981ef55ec954be69c4a652d021e6081a100d034de366815e9b
    # local_signature = 304502210094bfd8f5572ac0157ec76a9551b6c5216a4538c07cd13a51af4a54cb26fa14320220768efce8ce6f4a5efac875142ff19237c011343670adf9c7ac69704a120d1163
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8007e80300000000000022002052bfef0479d7b293c27e0f1eb294bea154c63a3294ef092c19af51409bce0e2ad007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de843110e09c6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040048304502210094bfd8f5572ac0157ec76a9551b6c5216a4538c07cd13a51af4a54cb26fa14320220768efce8ce6f4a5efac875142ff19237c011343670adf9c7ac69704a120d116301483045022100a5c01383d3ec646d97e40f44318d49def817fcd61a0ef18008a665b3e151785502203e648efddd5838981ef55ec954be69c4a652d021e6081a100d034de366815e9b01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 5
    # signature for output 0 (HTLC 0)
    remote_htlc_signature = 30440220385a5afe75632f50128cbb029ee95c80156b5b4744beddc729ad339c9ca432c802202ba5f48550cad3379ac75b9b4fedb86a35baa6947f16ba5037fb8b11ab343740
    # signature for output 1 (HTLC 2)
    remote_htlc_signature = 304402207ceb6678d4db33d2401fdc409959e57c16a6cb97a30261d9c61f29b8c58d34b90220084b4a17b4ca0e86f2d798b3698ca52de5621f2ce86f80bed79afa66874511b0
    # signature for output 2 (HTLC 1)
    remote_htlc_signature = 304402206a401b29a0dff0d18ec903502c13d83e7ec019450113f4a7655a4ce40d1f65ba0220217723a084e727b6ca0cc8b6c69c014a7e4a01fcdcba3e3993f462a3c574d833
    # signature for output 3 (HTLC 3)
    remote_htlc_signature = 30450221009b1c987ba599ee3bde1dbca776b85481d70a78b681a8d84206723e2795c7cac002207aac84ad910f8598c4d1c0ea2e3399cf6627a4e3e90131315bc9f038451ce39d
    # signature for output 4 (HTLC 4)
    remote_htlc_signature = 3045022100cc28030b59f0914f45b84caa983b6f8effa900c952310708c2b5b00781117022022027ba2ccdf94d03c6d48b327f183f6e28c8a214d089b9227f94ac4f85315274f0
    # local_signature = 304402205999590b8a79fa346e003a68fd40366397119b2b0cdf37b149968d6bc6fbcc4702202b1e1fb5ab7864931caed4e732c359e0fe3d86a548b557be2246efb1708d579a
    output htlc_success_tx 0: 020000000001018323148ce2419f21ca3d6780053747715832e18ac780931a514b187768882bb60000000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220385a5afe75632f50128cbb029ee95c80156b5b4744beddc729ad339c9ca432c802202ba5f48550cad3379ac75b9b4fedb86a35baa6947f16ba5037fb8b11ab3437400147304402205999590b8a79fa346e003a68fd40366397119b2b0cdf37b149968d6bc6fbcc4702202b1e1fb5ab7864931caed4e732c359e0fe3d86a548b557be2246efb1708d579a012000000000000000000000000000000000000000000000000000000000000000008a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a914b8bcb07f6344b42ab04250c86a6e8b75d3fdbbc688527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f401b175ac686800000000
    # local_signature = 304402207ff03eb0127fc7c6cae49cc29e2a586b98d1e8969cf4a17dfa50b9c2647720b902205e2ecfda2252956c0ca32f175080e75e4e390e433feb1f8ce9f2ba55648a1dac
    output htlc_timeout_tx 2: 020000000001018323148ce2419f21ca3d6780053747715832e18ac780931a514b187768882bb60100000000000000000124060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402207ceb6678d4db33d2401fdc409959e57c16a6cb97a30261d9c61f29b8c58d34b90220084b4a17b4ca0e86f2d798b3698ca52de5621f2ce86f80bed79afa66874511b00147304402207ff03eb0127fc7c6cae49cc29e2a586b98d1e8969cf4a17dfa50b9c2647720b902205e2ecfda2252956c0ca32f175080e75e4e390e433feb1f8ce9f2ba55648a1dac01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
    # local_signature = 3045022100d50d067ca625d54e62df533a8f9291736678d0b86c28a61bb2a80cf42e702d6e02202373dde7e00218eacdafb9415fe0e1071beec1857d1af3c6a201a44cbc47c877
    output htlc_success_tx 1: 020000000001018323148ce2419f21ca3d6780053747715832e18ac780931a514b187768882bb6020000000000000000010a060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e050047304402206a401b29a0dff0d18ec903502c13d83e7ec019450113f4a7655a4ce40d1f65ba0220217723a084e727b6ca0cc8b6c69c014a7e4a01fcdcba3e3993f462a3c574d83301483045022100d50d067ca625d54e62df533a8f9291736678d0b86c28a61bb2a80cf42e702d6e02202373dde7e00218eacdafb9415fe0e1071beec1857d1af3c6a201a44cbc47c877012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
    # local_signature = 3045022100db9dc65291077a52728c622987e9895b7241d4394d6dcb916d7600a3e8728c22022036ee3ee717ba0bb5c45ee84bc7bbf85c0f90f26ae4e4a25a6b4241afa8a3f1cb
    output htlc_timeout_tx 3: 020000000001018323148ce2419f21ca3d6780053747715832e18ac780931a514b187768882bb6030000000000000000010c0a0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004830450221009b1c987ba599ee3bde1dbca776b85481d70a78b681a8d84206723e2795c7cac002207aac84ad910f8598c4d1c0ea2e3399cf6627a4e3e90131315bc9f038451ce39d01483045022100db9dc65291077a52728c622987e9895b7241d4394d6dcb916d7600a3e8728c22022036ee3ee717ba0bb5c45ee84bc7bbf85c0f90f26ae4e4a25a6b4241afa8a3f1cb01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 304402202d1a3c0d31200265d2a2def2753ead4959ae20b4083e19553acfffa5dfab60bf022020ede134149504e15b88ab261a066de49848411e15e70f9e6a5462aec2949f8f
    output htlc_success_tx 4: 020000000001018323148ce2419f21ca3d6780053747715832e18ac780931a514b187768882bb604000000000000000001da0d0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100cc28030b59f0914f45b84caa983b6f8effa900c952310708c2b5b00781117022022027ba2ccdf94d03c6d48b327f183f6e28c8a214d089b9227f94ac4f85315274f00147304402202d1a3c0d31200265d2a2def2753ead4959ae20b4083e19553acfffa5dfab60bf022020ede134149504e15b88ab261a066de49848411e15e70f9e6a5462aec2949f8f012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with six outputs untrimmed (minimum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 648
    # base commitment transaction fee = 914
    # actual commitment transaction fee = 1914
    # HTLC 2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 1 received amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6987086 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3044022072714e2fbb93cdd1c42eb0828b4f2eff143f717d8f26e79d6ada4f0dcb681bbe02200911be4e5161dd6ebe59ff1c58e1997c4aea804f81db6b698821db6093d7b057
    # local_signature = 3045022100a2270d5950c89ae0841233f6efea9c951898b301b2e89e0adbd2c687b9f32efa02207943d90f95b9610458e7c65a576e149750ff3accaacad004cd85e70b235e27de
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8006d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de8431104e9d6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100a2270d5950c89ae0841233f6efea9c951898b301b2e89e0adbd2c687b9f32efa02207943d90f95b9610458e7c65a576e149750ff3accaacad004cd85e70b235e27de01473044022072714e2fbb93cdd1c42eb0828b4f2eff143f717d8f26e79d6ada4f0dcb681bbe02200911be4e5161dd6ebe59ff1c58e1997c4aea804f81db6b698821db6093d7b05701475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 4
    # signature for output 0 (HTLC 2)
    remote_htlc_signature = 3044022062ef2e77591409d60d7817d9bb1e71d3c4a2931d1a6c7c8307422c84f001a251022022dad9726b0ae3fe92bda745a06f2c00f92342a186d84518588cf65f4dfaada8
    # signature for output 1 (HTLC 1)
    remote_htlc_signature = 3045022100e968cbbb5f402ed389fdc7f6cd2a80ed650bb42c79aeb2a5678444af94f6c78502204b47a1cb24ab5b0b6fe69fe9cfc7dba07b9dd0d8b95f372c1d9435146a88f8d4
    # signature for output 2 (HTLC 3)
    remote_htlc_signature = 3045022100aa91932e305292cf9969cc23502bbf6cef83a5df39c95ad04a707c4f4fed5c7702207099fc0f3a9bfe1e7683c0e9aa5e76c5432eb20693bf4cb182f04d383dc9c8c2
    # signature for output 3 (HTLC 4)
    remote_htlc_signature = 3044022035cac88040a5bba420b1c4257235d5015309113460bc33f2853cd81ca36e632402202fc94fd3e81e9d34a9d01782a0284f3044370d03d60f3fc041e2da088d2de58f
    # local_signature = 3045022100a4c574f00411dd2f978ca5cdc1b848c311cd7849c087ad2f21a5bce5e8cc5ae90220090ae39a9bce2fb8bc879d7e9f9022df249f41e25e51f1a9bf6447a9eeffc098
    output htlc_timeout_tx 2: 02000000000101579c183eca9e8236a5d7f5dcd79cfec32c497fdc0ec61533cde99ecd436cadd10000000000000000000123060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022062ef2e77591409d60d7817d9bb1e71d3c4a2931d1a6c7c8307422c84f001a251022022dad9726b0ae3fe92bda745a06f2c00f92342a186d84518588cf65f4dfaada801483045022100a4c574f00411dd2f978ca5cdc1b848c311cd7849c087ad2f21a5bce5e8cc5ae90220090ae39a9bce2fb8bc879d7e9f9022df249f41e25e51f1a9bf6447a9eeffc09801008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
    # local_signature = 304402207679cf19790bea76a733d2fa0672bd43ab455687a068f815a3d237581f57139a0220683a1a799e102071c206b207735ca80f627ab83d6616b4bcd017c5d79ef3e7d0
    output htlc_success_tx 1: 02000000000101579c183eca9e8236a5d7f5dcd79cfec32c497fdc0ec61533cde99ecd436cadd10100000000000000000109060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100e968cbbb5f402ed389fdc7f6cd2a80ed650bb42c79aeb2a5678444af94f6c78502204b47a1cb24ab5b0b6fe69fe9cfc7dba07b9dd0d8b95f372c1d9435146a88f8d40147304402207679cf19790bea76a733d2fa0672bd43ab455687a068f815a3d237581f57139a0220683a1a799e102071c206b207735ca80f627ab83d6616b4bcd017c5d79ef3e7d0012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
    # local_signature = 304402200df76fea718745f3c529bac7fd37923e7309ce38b25c0781e4cf514dd9ef8dc802204172295739dbae9fe0474dcee3608e3433b4b2af3a2e6787108b02f894dcdda3
    output htlc_timeout_tx 3: 02000000000101579c183eca9e8236a5d7f5dcd79cfec32c497fdc0ec61533cde99ecd436cadd1020000000000000000010b0a0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100aa91932e305292cf9969cc23502bbf6cef83a5df39c95ad04a707c4f4fed5c7702207099fc0f3a9bfe1e7683c0e9aa5e76c5432eb20693bf4cb182f04d383dc9c8c20147304402200df76fea718745f3c529bac7fd37923e7309ce38b25c0781e4cf514dd9ef8dc802204172295739dbae9fe0474dcee3608e3433b4b2af3a2e6787108b02f894dcdda301008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 304402200daf2eb7afd355b4caf6fb08387b5f031940ea29d1a9f35071288a839c9039e4022067201b562456e7948616c13acb876b386b511599b58ac1d94d127f91c50463a6
    output htlc_success_tx 4: 02000000000101579c183eca9e8236a5d7f5dcd79cfec32c497fdc0ec61533cde99ecd436cadd103000000000000000001d90d0000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022035cac88040a5bba420b1c4257235d5015309113460bc33f2853cd81ca36e632402202fc94fd3e81e9d34a9d01782a0284f3044370d03d60f3fc041e2da088d2de58f0147304402200daf2eb7afd355b4caf6fb08387b5f031940ea29d1a9f35071288a839c9039e4022067201b562456e7948616c13acb876b386b511599b58ac1d94d127f91c50463a6012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with six outputs untrimmed (maximum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 2069
    # base commitment transaction fee = 2921
    # actual commitment transaction fee = 3921
    # HTLC 2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 1 received amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6985079 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3044022001d55e488b8b035b2dd29d50b65b530923a416d47f377284145bc8767b1b6a75022019bb53ddfe1cefaf156f924777eaaf8fdca1810695a7d0a247ad2afba8232eb4
    # local_signature = 304402203ca8f31c6a47519f83255dc69f1894d9a6d7476a19f498d31eaf0cd3a85eeb63022026fd92dc752b33905c4c838c528b692a8ad4ced959990b5d5ee2ff940fa90eea
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8006d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5d007000000000000220020748eba944fedc8827f6b06bc44678f93c0f9e6078b35c6331ed31e75f8ce0c2db80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de84311077956a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402203ca8f31c6a47519f83255dc69f1894d9a6d7476a19f498d31eaf0cd3a85eeb63022026fd92dc752b33905c4c838c528b692a8ad4ced959990b5d5ee2ff940fa90eea01473044022001d55e488b8b035b2dd29d50b65b530923a416d47f377284145bc8767b1b6a75022019bb53ddfe1cefaf156f924777eaaf8fdca1810695a7d0a247ad2afba8232eb401475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 4
    # signature for output 0 (HTLC 2)
    remote_htlc_signature = 3045022100d1cf354de41c1369336cf85b225ed033f1f8982a01be503668df756a7e668b66022001254144fb4d0eecc61908fccc3388891ba17c5d7a1a8c62bdd307e5a513f992
    # signature for output 1 (HTLC 1)
    remote_htlc_signature = 3045022100d065569dcb94f090345402736385efeb8ea265131804beac06dd84d15dd2d6880220664feb0b4b2eb985fadb6ec7dc58c9334ea88ce599a9be760554a2d4b3b5d9f4
    # signature for output 2 (HTLC 3)
    remote_htlc_signature = 3045022100d4e69d363de993684eae7b37853c40722a4c1b4a7b588ad7b5d8a9b5006137a102207a069c628170ee34be5612747051bdcc087466dbaa68d5756ea81c10155aef18
    # signature for output 3 (HTLC 4)
    remote_htlc_signature = 30450221008ec888e36e4a4b3dc2ed6b823319855b2ae03006ca6ae0d9aa7e24bfc1d6f07102203b0f78885472a67ff4fe5916c0bb669487d659527509516fc3a08e87a2cc0a7c
    # local_signature = 3044022056eb1af429660e45a1b0b66568cb8c4a3aa7e4c9c292d5d6c47f86ebf2c8838f022065c3ac4ebe980ca7a41148569be4ad8751b0a724a41405697ec55035dae66402
    output htlc_timeout_tx 2: 02000000000101ca94a9ad516ebc0c4bdd7b6254871babfa978d5accafb554214137d398bfcf6a0000000000000000000175020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100d1cf354de41c1369336cf85b225ed033f1f8982a01be503668df756a7e668b66022001254144fb4d0eecc61908fccc3388891ba17c5d7a1a8c62bdd307e5a513f99201473044022056eb1af429660e45a1b0b66568cb8c4a3aa7e4c9c292d5d6c47f86ebf2c8838f022065c3ac4ebe980ca7a41148569be4ad8751b0a724a41405697ec55035dae6640201008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
    # local_signature = 3045022100914bb232cd4b2690ee3d6cb8c3713c4ac9c4fb925323068d8b07f67c8541f8d9022057152f5f1615b793d2d45aac7518989ae4fe970f28b9b5c77504799d25433f7f
    output htlc_success_tx 1: 02000000000101ca94a9ad516ebc0c4bdd7b6254871babfa978d5accafb554214137d398bfcf6a0100000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100d065569dcb94f090345402736385efeb8ea265131804beac06dd84d15dd2d6880220664feb0b4b2eb985fadb6ec7dc58c9334ea88ce599a9be760554a2d4b3b5d9f401483045022100914bb232cd4b2690ee3d6cb8c3713c4ac9c4fb925323068d8b07f67c8541f8d9022057152f5f1615b793d2d45aac7518989ae4fe970f28b9b5c77504799d25433f7f012001010101010101010101010101010101010101010101010101010101010101018a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a9144b6b2e5444c2639cc0fb7bcea5afba3f3cdce23988527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f501b175ac686800000000
    # local_signature = 304402200e362443f7af830b419771e8e1614fc391db3a4eb799989abfc5ab26d6fcd032022039ab0cad1c14dfbe9446bf847965e56fe016e0cbcf719fd18c1bfbf53ecbd9f9
    output htlc_timeout_tx 3: 02000000000101ca94a9ad516ebc0c4bdd7b6254871babfa978d5accafb554214137d398bfcf6a020000000000000000015d060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100d4e69d363de993684eae7b37853c40722a4c1b4a7b588ad7b5d8a9b5006137a102207a069c628170ee34be5612747051bdcc087466dbaa68d5756ea81c10155aef180147304402200e362443f7af830b419771e8e1614fc391db3a4eb799989abfc5ab26d6fcd032022039ab0cad1c14dfbe9446bf847965e56fe016e0cbcf719fd18c1bfbf53ecbd9f901008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 304402202c3e14282b84b02705dfd00a6da396c9fe8a8bcb1d3fdb4b20a4feba09440e8b02202b058b39aa9b0c865b22095edcd9ff1f71bbfe20aa4993755e54d042755ed0d5
    output htlc_success_tx 4: 02000000000101ca94a9ad516ebc0c4bdd7b6254871babfa978d5accafb554214137d398bfcf6a03000000000000000001f2090000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004830450221008ec888e36e4a4b3dc2ed6b823319855b2ae03006ca6ae0d9aa7e24bfc1d6f07102203b0f78885472a67ff4fe5916c0bb669487d659527509516fc3a08e87a2cc0a7c0147304402202c3e14282b84b02705dfd00a6da396c9fe8a8bcb1d3fdb4b20a4feba09440e8b02202b058b39aa9b0c865b22095edcd9ff1f71bbfe20aa4993755e54d042755ed0d5012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with five outputs untrimmed (minimum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 2070
    # base commitment transaction fee = 2566
    # actual commitment transaction fee = 5566
    # HTLC 2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6985434 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3045022100f2377f7a67b7fc7f4e2c0c9e3a7de935c32417f5668eda31ea1db401b7dc53030220415fdbc8e91d0f735e70c21952342742e25249b0d062d43efbfc564499f37526
    # local_signature = 30440220443cb07f650aebbba14b8bc8d81e096712590f524c5991ac0ed3bbc8fd3bd0c7022028a635f548e3ca64b19b69b1ea00f05b22752f91daf0b6dab78e62ba52eb7fd0
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8005d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5b80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de843110da966a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e04004730440220443cb07f650aebbba14b8bc8d81e096712590f524c5991ac0ed3bbc8fd3bd0c7022028a635f548e3ca64b19b69b1ea00f05b22752f91daf0b6dab78e62ba52eb7fd001483045022100f2377f7a67b7fc7f4e2c0c9e3a7de935c32417f5668eda31ea1db401b7dc53030220415fdbc8e91d0f735e70c21952342742e25249b0d062d43efbfc564499f3752601475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 3
    # signature for output 0 (HTLC 2)
    remote_htlc_signature = 3045022100eed143b1ee4bed5dc3cde40afa5db3e7354cbf9c44054b5f713f729356f08cf7022077161d171c2bbd9badf3c9934de65a4918de03bbac1450f715275f75b103f891
    # signature for output 1 (HTLC 3)
    remote_htlc_signature = 3044022071e9357619fd8d29a411dc053b326a5224c5d11268070e88ecb981b174747c7a02202b763ae29a9d0732fa8836dd8597439460b50472183f420021b768981b4f7cf6
    # signature for output 2 (HTLC 4)
    remote_htlc_signature = 3045022100c9458a4d2cbb741705577deb0a890e5cb90ee141be0400d3162e533727c9cb2102206edcf765c5dc5e5f9b976ea8149bf8607b5a0efb30691138e1231302b640d2a4
    # local_signature = 3045022100a0d043ed533e7fb1911e0553d31a8e2f3e6de19dbc035257f29d747c5e02f1f5022030cd38d8e84282175d49c1ebe0470db3ebd59768cf40780a784e248a43904fb8
    output htlc_timeout_tx 2: 0200000000010140a83ce364747ff277f4d7595d8d15f708418798922c40bc2b056aca5485a2180000000000000000000174020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100eed143b1ee4bed5dc3cde40afa5db3e7354cbf9c44054b5f713f729356f08cf7022077161d171c2bbd9badf3c9934de65a4918de03bbac1450f715275f75b103f89101483045022100a0d043ed533e7fb1911e0553d31a8e2f3e6de19dbc035257f29d747c5e02f1f5022030cd38d8e84282175d49c1ebe0470db3ebd59768cf40780a784e248a43904fb801008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
    # local_signature = 3045022100adb1d679f65f96178b59f23ed37d3b70443118f345224a07ecb043eee2acc157022034d24524fe857144a3bcfff3065a9994d0a6ec5f11c681e49431d573e242612d
    output htlc_timeout_tx 3: 0200000000010140a83ce364747ff277f4d7595d8d15f708418798922c40bc2b056aca5485a218010000000000000000015c060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022071e9357619fd8d29a411dc053b326a5224c5d11268070e88ecb981b174747c7a02202b763ae29a9d0732fa8836dd8597439460b50472183f420021b768981b4f7cf601483045022100adb1d679f65f96178b59f23ed37d3b70443118f345224a07ecb043eee2acc157022034d24524fe857144a3bcfff3065a9994d0a6ec5f11c681e49431d573e242612d01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 304402200831422aa4e1ee6d55e0b894201770a8f8817a189356f2d70be76633ffa6a6f602200dd1b84a4855dc6727dd46c98daae43dfc70889d1ba7ef0087529a57c06e5e04
    output htlc_success_tx 4: 0200000000010140a83ce364747ff277f4d7595d8d15f708418798922c40bc2b056aca5485a21802000000000000000001f1090000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100c9458a4d2cbb741705577deb0a890e5cb90ee141be0400d3162e533727c9cb2102206edcf765c5dc5e5f9b976ea8149bf8607b5a0efb30691138e1231302b640d2a40147304402200831422aa4e1ee6d55e0b894201770a8f8817a189356f2d70be76633ffa6a6f602200dd1b84a4855dc6727dd46c98daae43dfc70889d1ba7ef0087529a57c06e5e04012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with five outputs untrimmed (maximum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 2194
    # base commitment transaction fee = 2720
    # actual commitment transaction fee = 5720
    # HTLC 2 offered amount 2000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6985280 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3045022100d33c4e541aa1d255d41ea9a3b443b3b822ad8f7f86862638aac1f69f8f760577022007e2a18e6931ce3d3a804b1c78eda1de17dbe1fb7a95488c9a4ec86203953348
    # local_signature = 304402203b1b010c109c2ecbe7feb2d259b9c4126bd5dc99ee693c422ec0a5781fe161ba0220571fe4e2c649dea9c7aaf7e49b382962f6a3494963c97d80fef9a430ca3f7061
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8005d007000000000000220020403d394747cae42e98ff01734ad5c08f82ba123d3d9a620abda88989651e2ab5b80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de84311040966a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402203b1b010c109c2ecbe7feb2d259b9c4126bd5dc99ee693c422ec0a5781fe161ba0220571fe4e2c649dea9c7aaf7e49b382962f6a3494963c97d80fef9a430ca3f706101483045022100d33c4e541aa1d255d41ea9a3b443b3b822ad8f7f86862638aac1f69f8f760577022007e2a18e6931ce3d3a804b1c78eda1de17dbe1fb7a95488c9a4ec8620395334801475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 3
    # signature for output 0 (HTLC 2)
    remote_htlc_signature = 30450221009ed2f0a67f99e29c3c8cf45c08207b765980697781bb727fe0b1416de0e7622902206052684229bc171419ed290f4b615c943f819c0262414e43c5b91dcf72ddcf44
    # signature for output 1 (HTLC 3)
    remote_htlc_signature = 30440220155d3b90c67c33a8321996a9be5b82431b0c126613be751d400669da9d5c696702204318448bcd48824439d2c6a70be6e5747446be47ff45977cf41672bdc9b6b12d
    # signature for output 2 (HTLC 4)
    remote_htlc_signature = 3045022100a12a9a473ece548584aabdd051779025a5ed4077c4b7aa376ec7a0b1645e5a48022039490b333f53b5b3e2ddde1d809e492cba2b3e5fc3a436cd3ffb4cd3d500fa5a
    # local_signature = 3044022004ad5f04ae69c71b3b141d4db9d0d4c38d84009fb3cfeeae6efdad414487a9a0022042d3fe1388c1ff517d1da7fb4025663d372c14728ed52dc88608363450ff6a2f
    output htlc_timeout_tx 2: 02000000000101fb824d4e4dafc0f567789dee3a6bce8d411fe80f5563d8cdfdcc7d7e4447d43a0000000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004830450221009ed2f0a67f99e29c3c8cf45c08207b765980697781bb727fe0b1416de0e7622902206052684229bc171419ed290f4b615c943f819c0262414e43c5b91dcf72ddcf4401473044022004ad5f04ae69c71b3b141d4db9d0d4c38d84009fb3cfeeae6efdad414487a9a0022042d3fe1388c1ff517d1da7fb4025663d372c14728ed52dc88608363450ff6a2f01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a914b43e1b38138a41b37f7cd9a1d274bc63e3a9b5d188ac6868f6010000
    # local_signature = 304402201707050c870c1f77cc3ed58d6d71bf281de239e9eabd8ef0955bad0d7fe38dcc02204d36d80d0019b3a71e646a08fa4a5607761d341ae8be371946ebe437c289c915
    output htlc_timeout_tx 3: 02000000000101fb824d4e4dafc0f567789dee3a6bce8d411fe80f5563d8cdfdcc7d7e4447d43a010000000000000000010a060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e05004730440220155d3b90c67c33a8321996a9be5b82431b0c126613be751d400669da9d5c696702204318448bcd48824439d2c6a70be6e5747446be47ff45977cf41672bdc9b6b12d0147304402201707050c870c1f77cc3ed58d6d71bf281de239e9eabd8ef0955bad0d7fe38dcc02204d36d80d0019b3a71e646a08fa4a5607761d341ae8be371946ebe437c289c91501008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 3045022100ff200bc934ab26ce9a559e998ceb0aee53bc40368e114ab9d3054d9960546e2802202496856ca163ac12c143110b6b3ac9d598df7254f2e17b3b94c3ab5301f4c3b0
    output htlc_success_tx 4: 02000000000101fb824d4e4dafc0f567789dee3a6bce8d411fe80f5563d8cdfdcc7d7e4447d43a020000000000000000019a090000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100a12a9a473ece548584aabdd051779025a5ed4077c4b7aa376ec7a0b1645e5a48022039490b333f53b5b3e2ddde1d809e492cba2b3e5fc3a436cd3ffb4cd3d500fa5a01483045022100ff200bc934ab26ce9a559e998ceb0aee53bc40368e114ab9d3054d9960546e2802202496856ca163ac12c143110b6b3ac9d598df7254f2e17b3b94c3ab5301f4c3b0012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with four outputs untrimmed (minimum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 2195
    # base commitment transaction fee = 2344
    # actual commitment transaction fee = 7344
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6985656 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 304402205e2f76d4657fb732c0dfc820a18a7301e368f5799e06b7828007633741bda6df0220458009ae59d0c6246065c419359e05eb2a4b4ef4a1b310cc912db44eb7924298
    # local_signature = 304402203b12d44254244b8ff3bb4129b0920fd45120ab42f553d9976394b099d500c99e02205e95bb7a3164852ef0c48f9e0eaf145218f8e2c41251b231f03cbdc4f29a5429
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8004b80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de843110b8976a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402203b12d44254244b8ff3bb4129b0920fd45120ab42f553d9976394b099d500c99e02205e95bb7a3164852ef0c48f9e0eaf145218f8e2c41251b231f03cbdc4f29a54290147304402205e2f76d4657fb732c0dfc820a18a7301e368f5799e06b7828007633741bda6df0220458009ae59d0c6246065c419359e05eb2a4b4ef4a1b310cc912db44eb792429801475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 2
    # signature for output 0 (HTLC 3)
    remote_htlc_signature = 3045022100a8a78fa1016a5c5c3704f2e8908715a3cef66723fb95f3132ec4d2d05cd84fb4022025ac49287b0861ec21932405f5600cbce94313dbde0e6c5d5af1b3366d8afbfc
    # signature for output 1 (HTLC 4)
    remote_htlc_signature = 3045022100e769cb156aa2f7515d126cef7a69968629620ce82afcaa9e210969de6850df4602200b16b3f3486a229a48aadde520dbee31ae340dbadaffae74fbb56681fef27b92
    # local_signature = 3045022100be6ae1977fd7b630a53623f3f25c542317ccfc2b971782802a4f1ef538eb22b402207edc4d0408f8f38fd3c7365d1cfc26511b7cd2d4fecd8b005fba3cd5bc704390
    output htlc_timeout_tx 3: 020000000001014e16c488fa158431c1a82e8f661240ec0a71ba0ce92f2721a6538c510226ad5c0000000000000000000109060000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100a8a78fa1016a5c5c3704f2e8908715a3cef66723fb95f3132ec4d2d05cd84fb4022025ac49287b0861ec21932405f5600cbce94313dbde0e6c5d5af1b3366d8afbfc01483045022100be6ae1977fd7b630a53623f3f25c542317ccfc2b971782802a4f1ef538eb22b402207edc4d0408f8f38fd3c7365d1cfc26511b7cd2d4fecd8b005fba3cd5bc70439001008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 30440220665b9cb4a978c09d1ca8977a534999bc8a49da624d0c5439451dd69cde1a003d022070eae0620f01f3c1bd029cc1488da13fb40fdab76f396ccd335479a11c5276d8
    output htlc_success_tx 4: 020000000001014e16c488fa158431c1a82e8f661240ec0a71ba0ce92f2721a6538c510226ad5c0100000000000000000199090000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100e769cb156aa2f7515d126cef7a69968629620ce82afcaa9e210969de6850df4602200b16b3f3486a229a48aadde520dbee31ae340dbadaffae74fbb56681fef27b92014730440220665b9cb4a978c09d1ca8977a534999bc8a49da624d0c5439451dd69cde1a003d022070eae0620f01f3c1bd029cc1488da13fb40fdab76f396ccd335479a11c5276d8012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with four outputs untrimmed (maximum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 3702
    # base commitment transaction fee = 3953
    # actual commitment transaction fee = 8953
    # HTLC 3 offered amount 3000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6984047 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3045022100c1a3b0b60ca092ed5080121f26a74a20cec6bdee3f8e47bae973fcdceb3eda5502207d467a9873c939bf3aa758014ae67295fedbca52412633f7e5b2670fc7c381c1
    # local_signature = 304402200e930a43c7951162dc15a2b7344f48091c74c70f7024e7116e900d8bcfba861c022066fa6cbda3929e21daa2e7e16a4b948db7e8919ef978402360d1095ffdaff7b0
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8004b80b000000000000220020c20b5d1f8584fd90443e7b7b720136174fa4b9333c261d04dbbd012635c0f419a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de8431106f916a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402200e930a43c7951162dc15a2b7344f48091c74c70f7024e7116e900d8bcfba861c022066fa6cbda3929e21daa2e7e16a4b948db7e8919ef978402360d1095ffdaff7b001483045022100c1a3b0b60ca092ed5080121f26a74a20cec6bdee3f8e47bae973fcdceb3eda5502207d467a9873c939bf3aa758014ae67295fedbca52412633f7e5b2670fc7c381c101475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 2
    # signature for output 0 (HTLC 3)
    remote_htlc_signature = 3045022100dfb73b4fe961b31a859b2bb1f4f15cabab9265016dd0272323dc6a9e85885c54022059a7b87c02861ee70662907f25ce11597d7b68d3399443a831ae40e777b76bdb
    # signature for output 1 (HTLC 4)
    remote_htlc_signature = 3045022100ea9dc2a7c3c3640334dab733bb4e036e32a3106dc707b24227874fa4f7da746802204d672f7ac0fe765931a8df10b81e53a3242dd32bd9dc9331eb4a596da87954e9
    # local_signature = 304402202765b9c9ece4f127fa5407faf66da4c5ce2719cdbe47cd3175fc7d48b482e43d02205605125925e07bad1e41c618a4b434d72c88a164981c4b8af5eaf4ee9142ec3a
    output htlc_timeout_tx 3: 02000000000101b8de11eb51c22498fe39722c7227b6e55ff1a94146cf638458cb9bc6a060d3a30000000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100dfb73b4fe961b31a859b2bb1f4f15cabab9265016dd0272323dc6a9e85885c54022059a7b87c02861ee70662907f25ce11597d7b68d3399443a831ae40e777b76bdb0147304402202765b9c9ece4f127fa5407faf66da4c5ce2719cdbe47cd3175fc7d48b482e43d02205605125925e07bad1e41c618a4b434d72c88a164981c4b8af5eaf4ee9142ec3a01008576a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c820120876475527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae67a9148a486ff2e31d6158bf39e2608864d63fefd09d5b88ac6868f7010000
    # local_signature = 30440220048a41c660c4841693de037d00a407810389f4574b3286afb7bc392a438fa3f802200401d71fa87c64fe621b49ac07e3bf85157ac680acb977124da28652cc7f1a5c
    output htlc_success_tx 4: 02000000000101b8de11eb51c22498fe39722c7227b6e55ff1a94146cf638458cb9bc6a060d3a30100000000000000000176050000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100ea9dc2a7c3c3640334dab733bb4e036e32a3106dc707b24227874fa4f7da746802204d672f7ac0fe765931a8df10b81e53a3242dd32bd9dc9331eb4a596da87954e9014730440220048a41c660c4841693de037d00a407810389f4574b3286afb7bc392a438fa3f802200401d71fa87c64fe621b49ac07e3bf85157ac680acb977124da28652cc7f1a5c012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with three outputs untrimmed (minimum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 3703
    # base commitment transaction fee = 3317
    # actual commitment transaction fee = 11317
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6984683 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 30450221008b7c191dd46893b67b628e618d2dc8e81169d38bade310181ab77d7c94c6675e02203b4dd131fd7c9deb299560983dcdc485545c98f989f7ae8180c28289f9e6bdb0
    # local_signature = 3044022047305531dd44391dce03ae20f8735005c615eb077a974edb0059ea1a311857d602202e0ed6972fbdd1e8cb542b06e0929bc41b2ddf236e04cb75edd56151f4197506
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8003a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de843110eb936a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400473044022047305531dd44391dce03ae20f8735005c615eb077a974edb0059ea1a311857d602202e0ed6972fbdd1e8cb542b06e0929bc41b2ddf236e04cb75edd56151f4197506014830450221008b7c191dd46893b67b628e618d2dc8e81169d38bade310181ab77d7c94c6675e02203b4dd131fd7c9deb299560983dcdc485545c98f989f7ae8180c28289f9e6bdb001475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 1
    # signature for output 0 (HTLC 4)
    remote_htlc_signature = 3044022044f65cf833afdcb9d18795ca93f7230005777662539815b8a601eeb3e57129a902206a4bf3e53392affbba52640627defa8dc8af61c958c9e827b2798ab45828abdd
    # local_signature = 3045022100b94d931a811b32eeb885c28ddcf999ae1981893b21dd1329929543fe87ce793002206370107fdd151c5f2384f9ceb71b3107c69c74c8ed5a28a94a4ab2d27d3b0724
    output htlc_success_tx 4: 020000000001011c076aa7fb3d7460d10df69432c904227ea84bbf3134d4ceee5fb0f135ef206d0000000000000000000175050000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500473044022044f65cf833afdcb9d18795ca93f7230005777662539815b8a601eeb3e57129a902206a4bf3e53392affbba52640627defa8dc8af61c958c9e827b2798ab45828abdd01483045022100b94d931a811b32eeb885c28ddcf999ae1981893b21dd1329929543fe87ce793002206370107fdd151c5f2384f9ceb71b3107c69c74c8ed5a28a94a4ab2d27d3b0724012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with three outputs untrimmed (maximum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 4914
    # base commitment transaction fee = 4402
    # actual commitment transaction fee = 12402
    # HTLC 4 received amount 4000 wscript 76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac6868
    # to_local amount 6983598 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 304402206d6cb93969d39177a09d5d45b583f34966195b77c7e585cf47ac5cce0c90cefb022031d71ae4e33a4e80df7f981d696fbdee517337806a3c7138b7491e2cbb077a0e
    # local_signature = 304402206a2679efa3c7aaffd2a447fd0df7aba8792858b589750f6a1203f9259173198a022008d52a0e77a99ab533c36206cb15ad7aeb2aa72b93d4b571e728cb5ec2f6fe26
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8003a00f0000000000002200208c48d15160397c9731df9bc3b236656efb6665fbfe92b4a6878e88a499f741c4c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de843110ae8f6a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e040047304402206a2679efa3c7aaffd2a447fd0df7aba8792858b589750f6a1203f9259173198a022008d52a0e77a99ab533c36206cb15ad7aeb2aa72b93d4b571e728cb5ec2f6fe260147304402206d6cb93969d39177a09d5d45b583f34966195b77c7e585cf47ac5cce0c90cefb022031d71ae4e33a4e80df7f981d696fbdee517337806a3c7138b7491e2cbb077a0e01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 1
    # signature for output 0 (HTLC 4)
    remote_htlc_signature = 3045022100fcb38506bfa11c02874092a843d0cc0a8613c23b639832564a5f69020cb0f6ba02206508b9e91eaa001425c190c68ee5f887e1ad5b1b314002e74db9dbd9e42dbecf
    # local_signature = 304502210086e76b460ddd3cea10525fba298405d3fe11383e56966a5091811368362f689a02200f72ee75657915e0ede89c28709acd113ede9e1b7be520e3bc5cda425ecd6e68
    output htlc_success_tx 4: 0200000000010110a3fdcbcd5db477cd3ad465e7f501ffa8c437e8301f00a6061138590add757f0000000000000000000122020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0500483045022100fcb38506bfa11c02874092a843d0cc0a8613c23b639832564a5f69020cb0f6ba02206508b9e91eaa001425c190c68ee5f887e1ad5b1b314002e74db9dbd9e42dbecf0148304502210086e76b460ddd3cea10525fba298405d3fe11383e56966a5091811368362f689a02200f72ee75657915e0ede89c28709acd113ede9e1b7be520e3bc5cda425ecd6e68012004040404040404040404040404040404040404040404040404040404040404048a76a91414011f7254d96b819c76986c277d115efce6f7b58763ac67210394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b7c8201208763a91418bc1a114ccf9c052d3d23e28d3b0a9d1227434288527c21030d417a46946384f88d5f3337267c5e579765875dc4daca813e21734b140639e752ae677502f801b175ac686800000000

    name: commitment tx with two outputs untrimmed (minimum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 4915
    # base commitment transaction fee = 3558
    # actual commitment transaction fee = 15558
    # to_local amount 6984442 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 304402200769ba89c7330dfa4feba447b6e322305f12ac7dac70ec6ba997ed7c1b598d0802204fe8d337e7fee781f9b7b1a06e580b22f4f79d740059560191d7db53f8765552
    # local_signature = 3045022100a012691ba6cea2f73fa8bac37750477e66363c6d28813b0bb6da77c8eb3fb0270220365e99c51304b0b1a6ab9ea1c8500db186693e39ec1ad5743ee231b0138384b9
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8002c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de843110fa926a00000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80e0400483045022100a012691ba6cea2f73fa8bac37750477e66363c6d28813b0bb6da77c8eb3fb0270220365e99c51304b0b1a6ab9ea1c8500db186693e39ec1ad5743ee231b0138384b90147304402200769ba89c7330dfa4feba447b6e322305f12ac7dac70ec6ba997ed7c1b598d0802204fe8d337e7fee781f9b7b1a06e580b22f4f79d740059560191d7db53f876555201475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

    name: commitment tx with two outputs untrimmed (maximum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 9651180
    # base commitment transaction fee = 6987454
    # actual commitment transaction fee = 6999454
    # to_local amount 546 wscript 63210212a140cd0c6539d07cd08dfe09984dec3251ea808b892efeac3ede9402bf2b1967029000b2752103fd5960528dc152014952efdb702a88f71e3c1653b2314431701ec77e57fde83c68ac
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3044022037f83ff00c8e5fb18ae1f918ffc24e54581775a20ff1ae719297ef066c71caa9022039c529cccd89ff6c5ed1db799614533844bd6d101da503761c45c713996e3bbd
    # local_signature = 30440220514f977bf7edc442de8ce43ace9686e5ebdc0f893033f13e40fb46c8b8c6e1f90220188006227d175f5c35da0b092c57bea82537aed89f7778204dc5bacf4f29f2b9
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b800222020000000000002200204adb4e2f00643db396dd120d4e7dc17625f5f2c11a40d857accc862d6b7dd80ec0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de84311004004730440220514f977bf7edc442de8ce43ace9686e5ebdc0f893033f13e40fb46c8b8c6e1f90220188006227d175f5c35da0b092c57bea82537aed89f7778204dc5bacf4f29f2b901473044022037f83ff00c8e5fb18ae1f918ffc24e54581775a20ff1ae719297ef066c71caa9022039c529cccd89ff6c5ed1db799614533844bd6d101da503761c45c713996e3bbd01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

    name: commitment tx with one output untrimmed (minimum feerate)
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 9651181
    # base commitment transaction fee = 6987455
    # actual commitment transaction fee = 7000000
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3044022064901950be922e62cbe3f2ab93de2b99f37cff9fc473e73e394b27f88ef0731d02206d1dfa227527b4df44a07599289e207d6fd9cca60c0365682dcd3deaf739567e
    # local_signature = 3044022031a82b51bd014915fe68928d1abf4b9885353fb896cac10c3fdd88d7f9c7f2e00220716bda819641d2c63e65d3549b6120112e1aeaf1742eed94a471488e79e206b1
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8001c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de8431100400473044022031a82b51bd014915fe68928d1abf4b9885353fb896cac10c3fdd88d7f9c7f2e00220716bda819641d2c63e65d3549b6120112e1aeaf1742eed94a471488e79e206b101473044022064901950be922e62cbe3f2ab93de2b99f37cff9fc473e73e394b27f88ef0731d02206d1dfa227527b4df44a07599289e207d6fd9cca60c0365682dcd3deaf739567e01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

    name: commitment tx with fee greater than funder amount
    to_local_msat: 6988000000
    to_remote_msat: 3000000000
    local_feerate_per_kw: 9651936
    # base commitment transaction fee = 6988001
    # actual commitment transaction fee = 7000000
    # to_remote amount 3000000 P2WPKH(0394854aa6eab5b2a8122cc726e9dded053a2184d88256816826d6231c068d4a5b)
    remote_signature = 3044022064901950be922e62cbe3f2ab93de2b99f37cff9fc473e73e394b27f88ef0731d02206d1dfa227527b4df44a07599289e207d6fd9cca60c0365682dcd3deaf739567e
    # local_signature = 3044022031a82b51bd014915fe68928d1abf4b9885353fb896cac10c3fdd88d7f9c7f2e00220716bda819641d2c63e65d3549b6120112e1aeaf1742eed94a471488e79e206b1
    output commit_tx: 02000000000101bef67e4e2fb9ddeeb3461973cd4c62abb35050b1add772995b820b584a488489000000000038b02b8001c0c62d0000000000160014ccf1af2f2aabee14bb40fa3851ab2301de8431100400473044022031a82b51bd014915fe68928d1abf4b9885353fb896cac10c3fdd88d7f9c7f2e00220716bda819641d2c63e65d3549b6120112e1aeaf1742eed94a471488e79e206b101473044022064901950be922e62cbe3f2ab93de2b99f37cff9fc473e73e394b27f88ef0731d02206d1dfa227527b4df44a07599289e207d6fd9cca60c0365682dcd3deaf739567e01475221023da092f6980e58d2c037173180e9a465476026ee50f96695963e8efe436f54eb21030e9f7b623d2ccc7c9bd44d66d5ce21ce504c0acf6385a132cec6d3c39fa711c152ae3e195220
    num_htlcs: 0

# 附录D：按承诺秘密生成测试向量

这些测试所有节点使用的生成算法。

## 生成测试

    name：generate_from_seed 0最终节点
    种子：0x00000000000000000000000000000000000000000000000000000000000000000000
    I：281474976710655
    输出：0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148

    name：generate_from_seed FF最终节点
    种子：0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    I：281474976710655
    输出：0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc

    name：generate_from_seed FF备用位1
    种子：0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    我：0xaaaaaaaaaa
    输出：0x56f4008fb007ca9acf0e15b054d5c9fd12ee06cea347914ddbaed70d1c13a528

    name：generate_from_seed FF备用位2
    种子：0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
    我：0x555555555555
    输出：0x9015daaeb06dba4ccc05b91b2f73bd54405f2be9f217fbacd3c5ac2e62327d31

    name：generate_from_seed 01 last verytrivial node
    种子：0x010101010101010101010101010101010101010101010101010101010101010101
    我：1
    输出：0x915c75942a26bb3a433a8ce2cb0427c29ec6c1775cfc78328b57f6ba7bfeaa9c

## 存储测试

这些测试可选的紧凑型存储系统。在很多情况下，一个
在显示其父项之前无法确定错误的条目：条目是
特别是腐败，以及它的所有孩子。

对于
这些测试使用了`0xFFF ... FF`的种子，错误的条目是
以“0x000 ... 00”播种。

    name：insert_secret正确的序列
    I：281474976710655
    秘密：0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
    输出：好的
    I：281474976710654
    秘密：0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
    输出：好的
    I：281474976710653
    秘密：0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
    输出：好的
    I：281474976710652
    秘密：0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
    输出：好的
    I：281474976710651
    秘密：0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
    输出：好的
    I：281474976710650
    秘密：0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
    输出：好的
    I：281474976710649
    秘密：0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
    输出：好的
    I：281474976710648
    秘密：0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
    输出：好的

    name：insert_secret＃1不正确
    I：281474976710655
    秘密：0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148
    输出：好的
    I：281474976710654
    秘密：0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
    输出：错误

    name：insert_secret＃2不正确（＃1派生自不正确）
    I：281474976710655
    秘密：0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148
    输出：好的
    I：281474976710654
    秘密：0xdddc3a8d14fddf2b68fa8c7fbad2748274937479dd0f8930d5ebb4ab6bd866a3
    输出：好的
    I：281474976710653
    秘密：0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
    输出：好的
    I：281474976710652
    秘密：0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
    输出：错误

    name：insert_secret＃3不正确
    I：281474976710655
    秘密：0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
    输出：好的
    I：281474976710654
    秘密：0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
    输出：好的
    I：281474976710653
    秘密：0xc51a18b13e8527e579ec56365482c62f180b7d5760b46e9477dae59e87ed423a
    输出：好的
    I：281474976710652
    秘密：0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
    输出：错误

    name：insert_secret＃4不正确（1,2,3派生不正确）
    I：281474976710655
    秘密：0x02a40c85b6f28da08dfdbe0926c53fab2de6d28c10301f8f7c4073d5e42e3148
    输出：好的
    I：281474976710654
    秘密：0xdddc3a8d14fddf2b68fa8c7fbad2748274937479dd0f8930d5ebb4ab6bd866a3
    输出：好的
    I：281474976710653
    秘密：0xc51a18b13e8527e579ec56365482c62f180b7d5760b46e9477dae59e87ed423a
    输出：好的
    I：281474976710652
    秘密：0xba65d7b0ef55a3ba300d4e87af29868f394f8f138d78a7011669c79b37b936f4
    输出：好的
    I：281474976710651
    秘密：0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
    输出：好的
    I：281474976710650
    秘密：0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
    输出：好的
    I：281474976710649
    秘密：0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
    输出：好的
    I：281474976710648
    秘密：0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
    输出：错误

    name：insert_secret＃5不正确
    I：281474976710655
    秘密：0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
    输出：好的
    I：281474976710654
    秘密：0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
    输出：好的
    I：281474976710653
    秘密：0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
    输出：好的
    I：281474976710652
    秘密：0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
    输出：好的
    I：281474976710651
    秘密：0x631373ad5f9ef654bb3dade742d09504c567edd24320d2fcd68e3cc47e2ff6a6
    输出：好的
    I：281474976710650
    秘密：0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
    输出：错误

    name：insert_secret＃6不正确（5来自不正确）
    I：281474976710655
    秘密：0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
    输出：好的
    I：281474976710654
    秘密：0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
    输出：好的
    I：281474976710653
    秘密：0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
    输出：好的
    I：281474976710652
    秘密：0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
    输出：好的
    I：281474976710651
    秘密：0x631373ad5f9ef654bb3dade742d09504c567edd24320d2fcd68e3cc47e2ff6a6
    输出：好的
    I：281474976710650
    秘密：0xb7e76a83668bde38b373970155c868a653304308f9896692f904a23731224bb1
    输出：好的
    I：281474976710649
    秘密：0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
    输出：好的
    I：281474976710648
    秘密：0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
    输出：错误

    name：insert_secret＃7不正确
    I：281474976710655
    秘密：0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
    输出：好的
    I：281474976710654
    秘密：0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
    输出：好的
    I：281474976710653
    秘密：0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
    输出：好的
    I：281474976710652
    秘密：0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
    输出：好的
    I：281474976710651
    秘密：0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
    输出：好的
    I：281474976710650
    秘密：0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
    输出：好的
    I：281474976710649
    秘密：0xe7971de736e01da8ed58b94c2fc216cb1dca9e326f3a96e7194fe8ea8af6c0a3
    输出：好的
    I：281474976710648
    秘密：0x05cde6323d949933f7f7b78776bcc1ea6d9b31447732e3802e1f7ac44b650e17
    输出：错误

    name：insert_secret＃8不正确
    I：281474976710655
    秘密：0x7cc854b54e3e0dcdb010d7a3fee464a9687be6e8db3be6854c475621e007a5dc
    输出：好的
    I：281474976710654
    秘密：0xc7518c8ae4660ed02894df8976fa1a3659c1a8b4b5bec0c4b872abeba4cb8964
    输出：好的
    I：281474976710653
    秘密：0x2273e227a5b7449b6e70f1fb4652864038b1cbf9cd7c043a7d6456b7fc275ad8
    输出：好的
    I：281474976710652
    秘密：0x27cddaa5624534cb6cb9d7da077cf2b22ab21e9b506fd4998a51d54502e99116
    输出：好的
    I：281474976710651
    秘密：0xc65716add7aa98ba7acb236352d665cab17345fe45b55fb879ff80e6bd0c41dd
    输出：好的
    I：281474976710650
    秘密：0x969660042a28f32d9be17344e09374b379962d03db1574df5a8a5a47e19ce3f2
    输出：好的
    I：281474976710649
    秘密：0xa5a64476122ca0925fb344bdc1854c1c0a59fc614298e50a33e331980a220f32
    输出：好的
    I：281474976710648
    秘密：0xa7efbc61aac46d34f77778bac22c8a20c6a46ca460addc49009bda875ec88fa4
    输出：错误

# 附录E：关键衍生测试向量

这些测试推导出`localpubkey`，`remotepubkey`，`local_htlcpubkey`，`remote_htlcpubkey`，`local_delayedpubkey`，以及
`remote_delayedpubkey`（使用相同的公式），以及`revocationpubkey`。

所有这些都使用以下秘密（因而衍生点）：

    base_secret：0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f
    per_commitment_secret：0x1f1e1d1c1b1a191817161514131211100f0e0d0c0b0a09080706050403020100
    base_point：0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2
    per_commitment_point：0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486

    name：从basepoint和per_commitment_point派生pubkey
    ＃SHA256（per_commitment_point || basepoint）
    ＃=> SHA256（0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2）
    ＃= 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
    ＃+ basepoint（0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2）
    ＃= 0x0235f2dbfaa89b57ec7b055afe29849ef7ddfeb1cefdb9ebdc43f5494984db29e5
    localpubkey：0x0235f2dbfaa89b57ec7b055afe29849ef7ddfeb1cefdb9ebdc43f5494984db29e5

    name：从basepoint secret和per_commitment_secret派生私钥
    ＃SHA256（per_commitment_point || basepoint）
    ＃=> SHA256（0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2）
    ＃= 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
    ＃+ basepoint_secret（0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f）
    ＃= 0xcbced912d3b21bf196a766651e436aff192362621ce317704ea2f75d87e7be0f
    localprivkey：0xcbced912d3b21bf196a766651e436aff192362621ce317704ea2f75d87e7be0f

    name：从basepoint和per_commitment_point派生revocation pubkey
    ＃SHA256（revocation_basepoint || per_commitment_point）
    ＃=> SHA256（0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2 || 0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486）
    ＃= 0xefbf7ba5a074276701798376950a64a90f698997cce0dff4d24a6d2785d20963
    #x revocation_basepoint = 0x02c00c4aadc536290422a807250824a8d87f19d18da9d610d45621df22510db8ce
    ＃SHA256（per_commitment_point || revocation_basepoint）
    ＃=> SHA256（0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2）
    ＃= 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
    #x per_commitment_point = 0x0325ee7d3323ce52c4b33d4e0a73ab637711057dd8866e3b51202a04112f054c43
    ＃0x02c00c4aadc536290422a807250824a8d87f19d18da9d610d45621df22510db8ce + 0x0325ee7d3323ce52c4b33d4e0a73ab637711057dd8866e3b51202a04112f054c43 => 0x02916e326636d19c33f13e8c0c3a03dd157f332f3e99c317c141dd865eb01f8ff0
    revocationpubkey：0x02916e326636d19c33f13e8c0c3a03dd157f332f3e99c317c141dd865eb01f8ff0

    name：从basepoint_secret和per_commitment_secret派生撤销密钥
    ＃SHA256（revocation_basepoint || per_commitment_point）
    ＃=> SHA256（0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2 || 0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486）
    ＃= 0xefbf7ba5a074276701798376950a64a90f698997cce0dff4d24a6d2785d20963
    ＃* revocation_basepoint_secret（0x000102030405060708090a0b0c0d0e0f101112131415161718191a1b1c1d1e1f）＃= 0x44bfd55f845f885b8e60b2dca4b30272d5343be048d79ce87879d9863dedc842
    ＃SHA256（per_commitment_point || revocation_basepoint）
    ＃=> SHA256（0x025f7117a78150fe2ef97db7cfc83bd57b2e2c0d0dd25eaf467a4a1c2a45ce1486 || 0x036d6caac248af96f6afa7f904f550253a0f3ef3f5aa2fe6838a95b216691468e2）
    ＃= 0xcbcdd70fcfad15ea8e9e5c5a12365cf00912504f08ce01593689dd426bca9ff0
    ＃* per_commitment_secret（0x1f1e1d1c1b1a191817161514131211100f0e0d0c0b0a09080706050403020100）＃= 0x8be02a96a97b9a3c1c9f59ebb718401128b72ec009d85ee1656319b52319b8ce
    ＃=> 0xd09ffff62ddb2297ab000cc85bcb4283fdeb6aa052affbc9dddcf33b61078110
    revocationprivkey：0xd09ffff62ddb2297ab000cc85bcb4283fdeb6aa052affbc9dddcf33b61078110

# 参考

# 作者

[ 整我： ]

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。

