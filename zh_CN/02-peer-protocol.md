# BOLT＃2：通道管理的对等协议

对等通道协议有三个阶段：建立，正常
操作和关闭。

# 目录

  * [渠道](#channel)
    * [渠道建立](#channel-establishment)
      * [`open_channel`消息](#the-open_channel-message)
      * [`accept_channel`消息](#the-accept_channel-message)
      * [`funding_created`消息](#the-funding_created-message)
      * [`funding_signed`消息](#the-funding_signed-message)
      * [`funding_locked`消息](#the-funding_locked-message)
    * [频道关闭](#channel-close)
      * [关闭启动：`shutdown`](#closing-initiation-shutdown)
      * [结束谈判：`closing_signed`](#closing-negotiation-closing_signed)
    * [普通手术](#normal-operation)
      * [转发HTLC](#forwarding-htlcs)
      * [`cltv_expiry_delta`选择](#cltv_expiry_delta-selection)
      * [添加HTLC：`update_add_htlc`](#adding-an-htlc-update_add_htlc)
      * [删除HTLC：`update_fulfill_htlc`，`update_fail_htlc`和`update_fail_malformed_htlc`](#removing-an-htlc-update_fulfill_htlc-update_fail_htlc-and-update_fail_malformed_htlc)
      * [到目前为止提交更新：`commitment_signed`](#committing-updates-so-far-commitment_signed)
      * [完成向更新状态的转换：`revoke_and_ack`](#completing-the-transition-to-the-updated-state-revoke_and_ack)
      * [更新费用：`update_fee`](#updating-fees-update_fee)
    * [消息重传：`channel_reestablish`消息](#message-retransmission)
  * [作者](#authors)

# 渠道

## 渠道建立

验证并初始化连接后（[BOLT＃8]（08-transport.md）
和[BOLT＃1]（分别为01-messaging.md #the-init-message），可以开始建立信道。
这包括发送 `open_channel` 消息的资金节点（资助者），
然后是响应节点（fundee）发送 `accept_channel` 。随着
通道参数被锁定，资助者能够创建资金
交易和承诺交易的两个版本，如中所述
[BOLT＃3]（03-transactions.md＃bolt-3-bitcoin-transaction-and-script-formats）。
然后，出资者用 `funding_created` 发送资金输出的出口点
消息，以及fundee版本承诺的签名
交易。一旦fundee获得资金外包，它就能够
生成出资者对承诺交易的承诺并发送
过度使用`funding_signed`消息。

一旦渠道出资者收到`funding_signed`消息，它就会出现
必须将资金交易广播到比特币网络。后
发送/接收`funding_signed`消息，双方都应该等待
为融资交易进入区块链并达到
指定深度（确认数量）。双方派出后
`funding_locked`消息，通道已建立并可以开始
普通手术。 `funding_locked` 消息包括信息
这将用于构建频道验证证明。


        +-------+                              +-------+
        |       |--(1)---  open_channel  ----->|       |
        |       |<-(2)--  accept_channel  -----|       |
        |       |                              |       |
        |   A   |--(3)--  funding_created  --->|   B   |
        |       |<-(4)--  funding_signed  -----|       |
        |       |                              |       |
        |       |--(5)--- funding_locked  ---->|       |
        |       |<-(6)--- funding_locked  -----|       |
        +-------+                              +-------+

        - where node A is 'funder' and node B is 'fundee'

如果在任何阶段失败，或者一个节点决定通道条款
由其他节点提供的不适合，频道建立
失败。

请注意，多个通道可以作为所有通道并行运行
消息由`temporary_channel_id`（在...之前）标识
资金交易已创建）或`channel_id`（源自
资金交易）。

### `open_channel`消息

此消息包含有关节点的信息并指示其
渴望建立一个新的渠道。这是创造的第一步
融资交易和承诺交易的两个版本。

1. 类型：32（`open_channel`）
2. 数据：
   * [`32`：`chain_hash`]
   * [`32`：`temporary_channel_id`]
   * [`8`：`funding_satoshis`]
   * [`8`：`push_msat`]
   * [`8`：`dust_limit_satoshis`]
   * [`8`：`max_htlc_value_in_flight_msat`]
   * [`8`：`channel_reserve_satoshis`]
   * [`8`：`htlc_minimum_msat`]
   * [`4`：`feerate_per_kw`]
   * [`2`：`to_self_delay`]
   * [`2`：`max_accepted_htlcs`]
   * [`33`：`funding_pubkey`]
   * [`33`：`revocation_basepoint`]
   * [`33`：`payment_basepoint`]
   * [`33`：`delayed_payment_basepoint`]
   * [`33`：`htlc_basepoint`]
   * [`33`：`first_per_commitment_point`]
   * [`1`：`channel_flags`]
   * [`2`：`shutdown_len`]（`option_upfront_shutdown_script`）
   * [`shutdown_len`：`shutdown_scriptpubkey`]（`option_upfront_shutdown_script`）

`chain_hash`值表示打开的通道将确切的区块链
居住在。这通常是相应区块链的起源哈希。
`chain_hash`的存在允许节点打开通道
跨越许多不同的区块链以及多个区域内的渠道
区块链向同一对等体开放（如果它支持目标链）。

`temporary_channel_id`用于在每个对等体的基础上识别该信道，直到
建立资金交易，此时它被取代
通过`channel_id`，它来自融资交易。

`funding_satoshis`是发件人投入的金额
渠道。 `push_msat`是发件人的初始资金数额
无条件地给予接收者。 `dust_limit_satoshis`是
阈值低于该阈值，不应为该节点生成输出
承诺或HTLC交易（即低于此金额的HTLC加上
HTLC交易费用在链上是不可执行的）。这反映了
现实，微小的产出不被视为标准交易和
不会通过比特币网络传播。 `channel_reserve_satoshis`
是另一个节点作为直接保留的最小量
付款。 `htlc_minimum_msat`表示HTLC的最小值
节点将接受。

`max_htlc_value_in_flight_msat`是未偿还总值的上限
HTLC，允许节点限制其暴露于HTLC;同样，
`max_accepted_htlcs`限制另一个未完成HTLC的数量
节点可以提供。

`feerate_per_kw`表示每千克重量的初始费率
（即，通常使用的每千字节的“satoshi”的1/4）
方将支付承诺和HTLC交易，如中所述
[BOLT＃3]（03-transactions.md＃费用计算）（这可以调整
稍后使用`update_fee`消息）。

`to_self_delay`是另一个节点自身的块数
输出必须延迟，使用'OP_CHECKSEQUENCEVERIFY`延迟;这个
在兑换之前，如果发生故障需要等待多长时间
自有资金。

`funding_pubkey`是2-of-2 multisig脚本中的公钥
资金交易产出。

各种`_basepoint`字段用于派生唯一
每个承诺的[BOLT＃3]（03-transactions.md＃key-derivation）中描述的密钥
交易。改变这些密钥可确保事务ID
每个承诺交易对外部观察者来说都是不可预测的，
即使看到一笔承诺交易;这个属性非常好
在将罚款交易外包时保护隐私非常有用
第三方。

`first_per_commitment_point`是要使用的每承诺点
对于第一笔承诺交易，

目前只有“channel_flags”的最低位
定义：`announce_channel`。这表明是否是发起者
资金流动希望公开宣传这个频道
网络，详见[BOLT＃7]（07-routing-gossip.md＃bolt-7-p2p-node-and-channel-discovery）。

`shutdown_scriptpubkey`允许发送节点提交到哪里
资金将继续相互关闭，远程节点应该强制执行
即使节点稍后被泄露。

[FIXME：描述较大通道数量的危险特征位。 ]

#### 要求

发送节点：
  - 必须确保`chain_hash`值标识它希望在其中打开通道的链。
  - 必须确保`temporary_channel_id`与具有相同对等体的任何其他通道ID是唯一的。
  - 必须将`funding_satoshis`设置为小于2 ^ 24 satoshi。
  - 必须将`push_msat`设置为等于或小于1000 *`funding_satoshis`。
  - 必须将`funding_pubkey`，`revocation_basepoint`，`htlc_basepoint`，`payment_basepoint`和`delayed_payment_basepoint`设置为有效的DER编码，压缩，secp256k1 pubkeys。
  - 必须将`first_per_commitment_point`设置为用于初始承诺事务的每承诺点，按照[BOLT＃3]（03-transactions.md＃per-commitment-secret-requirements）中的规定派生。
  - 必须将`channel_reserve_satoshis`设置为大于或等于`dust_limit_satoshis`。
  - 必须将`channel_flags`中的未定义位设置为0。
  - 如果两个节点都宣传了`option_upfront_shutdown_script`功能：
    - 必须包含`shutdown``scriptpubkey`所需的有效`shutdown_scriptpubkey`，或者零长度`shutdown_scriptpubkey`。
  - 除此以外：
    - 可以包含一个`shutdown_scriptpubkey`。

发送节点应该：
  - 设置`to_self_delay`足以确保发送方可以不可逆转地花费承诺事务输出，以防接收方的行为不当。
  - 将`feerate_per_kw`设置为至少它估计的速率会导致交易立即包含在一个区块中。
  - 将`dust_limit_satoshis`设置为足够的值，以允许承诺事务通过比特币网络传播。
  - 将`htlc_minimum_msat`设置为它愿意接受来自该对等体的最小值HTLC。

接收节点必须：
  - 忽略`channel_flags`中的未定义位。
  - 如果在收到前一个连接后重新建立连接
 `open_channel`，但在收到`funding_created`消息之前：
    - 接受一个新的`open_channel`消息。
    - 丢弃之前的`open_channel`消息。

如果接收节点可能使通道失败：
  - `announce_channel`是'false`（`0`），但它希望公开宣布该频道。
  - `funding_satoshis`太小了。
  - 它认为`htlc_minimum_msat`太大了。
  - 它认为`max_htlc_value_in_flight_msat`太小了。
  - 它认为`channel_reserve_satoshis`太大了。
  - 它认为`max_accepted_htlcs`太小了。
  - 它认为`dust_limit_satoshis`太小，并且计划在数据丢失的情况下依赖发送节点发布其承诺事务（参见[message-retransmission]（02-peer-protocol.md＃message-retransmission））。

在以下情况下，接收节点必须使通道失败：
  - `chain_hash`值设置为接收器未知的链的哈希值。
  - `push_msat`大于`funding_satoshis` * 1000。
  - `to_self_delay`非常大。
  - `max_accepted_htlcs`大于483。
  - 它认为`feerate_per_kw`太小而不能及时处理或不合理地大。
  - `funding_pubkey`，`revocation_basepoint`，`htlc_basepoint`，`payment_basepoint`或`delayed_payment_basepoint`
是无效的DER编码压缩secp256k1 pubkeys。
  - `dust_limit_satoshis`大于`channel_reserve_satoshis`。
  - 初始承诺交易的出资者金额不足以支付全额[费用]（03-transactions.md＃付费）。
  - 初始承诺交易的“to_local”和“to_remote”金额均小于或等于`channel_reserve_satoshis`（参见[BOLT 3]（03-transactions.md＃commitment-transaction-outputs））。

接收节点不得：
  - 考虑收到的资金，使用“push_msat”，直到资金交易达到足够的深度。

#### 合理

“funding_satoshi”小于2 ^ 24 satoshi的要求是临时自我施加的限制，而实施尚未被认为是稳定的。
它可以在任何时间点解除，或者针对其他货币进行调整，因为它仅由渠道的端点强制执行。
具体而言，[路由八卦协议]（07-routing-gossip.md）不丢弃容量较大的信道。

* channel reserve *由对等体的'channel_reserve_satoshis`指定：建议使用1％的通道总数。频道的每一方都保留这个预留，因此如果要尝试广播旧的，已撤销的承诺交易，它总会丢失一些东西。最初，这个储备可能无法满足，因为只有一方有资金;但是该协议确保在满足此储备方面始终取得进展，并且一旦满足，就会得到维护。

发送方可以使用非零的“push_msat”无条件地向接收方提供初始资金，但即使在这种情况下，我们也确保资助者有足够的剩余资金来支付费用，并且一方有一些可以支出的金额（这也意味着至少有一个无尘输出）。请注意，与任何其他在线交易一样，这笔付款在资金交易得到充分确认之前是不确定的（在发生这种情况之前存在双重支出的危险）并且可能需要单独的方法来证明通过链上确认付款。

`feerate_per_kw`通常只关注发件人（支付费用），但也有HTLC交易支付的费率;因此，不合理的大费率也会对收件人造成不利影响。

将`htlc_basepoint`与`payment_basepoint`分开可以提高安全性：节点需要与`htlc_basepoint`关联的秘密来为协议生成HTLC签名，但是'payment_basepoint`的秘密可以在冷存储中。

要求`channel_reserve_satoshis`不被视为灰尘
根据`dust_limit_satoshis`消除所有输出的情况
将作为灰尘消除。类似的要求
`accept_channel`确保双方'`channel_reserve_satoshis`
在'dust_limit_satoshis`之上。

有关如何处理通道故障的详细信息，请参见[BOLT 5：故障通道]（05-onchain.md #failing-a-channel）。

#### temporary_channel_id的实际注意事项

请注意，由于重复的`temporary_channel_id'可能存在于不同的位置
同行，API，在资金提供之前通过渠道ID引用渠道
创建事务本质上是不安全的。唯一提供的协议
交换funding_created之前的频道的标识符是
（source_node_id，destination_node_id，temporary_channel_id）元组。注意
任何此类API在资助前通过其渠道ID引用渠道
事务被确认也不是持久的 - 直到你知道脚本
对应于资金输出的pubkey没有任何阻止重复的通道
IDS。

#### 未来

有一个局部特征位很容易表明a
接收节点准备为一个频道提供资金，这将扭转这一局面
协议。

### `accept_channel`消息

此消息包含有关节点的信息并指示其
接受新渠道。这是创建的第二步
资金交易和承诺交易的两个版本。

1. 类型：33（`accept_channel`）
2. 数据：
   * [`32`：`temporary_channel_id`]
   * [`8`：`dust_limit_satoshis`]
   * [`8`：`max_htlc_value_in_flight_msat`]
   * [`8`：`channel_reserve_satoshis`]
   * [`8`：`htlc_minimum_msat`]
   * [`4`：`minimum_depth`]
   * [`2`：`to_self_delay`]
   * [`2`：`max_accepted_htlcs`]
   * [`33`：`funding_pubkey`]
   * [`33`：`revocation_basepoint`]
   * [`33`：`payment_basepoint`]
   * [`33`：`delayed_payment_basepoint`]
   * [`33`：`htlc_basepoint`]
   * [`33`：`first_per_commitment_point`]
   * [`2`：`shutdown_len`]（`option_upfront_shutdown_script`）
   * [`shutdown_len`：`shutdown_scriptpubkey`]（`option_upfront_shutdown_script`）

#### 要求

`temporary_channel_id`必须与`中的`temporary_channel_id`相同
`open_channel`消息。

寄件人：
  - 应该将`minimum_depth`设置为它认为合理的块数
避免资金交易的双重支出。
  - 必须从`open_channel`消息中将`channel_reserve_satoshis`设置为大于或等于`dust_limit_satoshis`。
  - 必须从`open_channel`消息中设置'dust_limit_satoshis`小于或等于`channel_reserve_satoshis`。

收件人：
  - 如果`minimum_depth`不合理地大：
    - 可以拒绝该频道。
  - 如果`channel_reserve_satoshis`小于`open_channel`消息中的`dust_limit_satoshis`：
    - 必须拒绝频道。
  - 如果来自`open_channel`消息的`channel_reserve_satoshis`小于`dust_limit_satoshis`：
    - 必须拒绝频道。
其他字段与`open_channel`中的字段具有相同的要求。

### `funding_created`消息

此消息描述了资助者为其创建的外部点
初始承诺交易。收到同行后
签名，通过`funding_signed`，它将广播资金交易。

1. 类型：34（`funding_created`）
2. 数据：
    * [`32`：`temporary_channel_id`]
    * [`32`：`funding_txid`]
    * [`2`：`funding_output_index`]
    * [`64`：`signature`]

#### 要求

发件人必须设置：
  - `temporary_channel_id`与`open_channel`消息中的`temporary_channel_id`相同。
  - `funding_txid`为非可延展交易的交易ID，
    - 并且不得广播此交易。
  - `funding_output_index`指对应于资金交易输出的交易的输出数，如[BOLT＃3]（03-transactions.md＃funding-transaction-output）中所定义。
  - 如[BOLT＃3]（03-transactions.md＃commitment-transaction）中所定义的那样，使用`funding_pubkey`对初始承诺交易进行有效签名的签名。

寄件人：
  - 在创建融资交易时：
    - 应该只使用BIP141（隔离见证）输入。

收件人：
  - 如果`签名'不正确：
    - 必须通道失败。

#### 合理

`funding_output_index`只能是2个字节，因为它是如何打包到`channel_id`并在整个八卦协议中使用的。 65535输出的限制不应过于繁琐。

具有所有隔离见证输入的交易不具有可延展性，因此是资金交易建议。

### `funding_signed`消息

此消息为资助者提供了第一个所需的签名
承诺交易，因此它可以在知道资金的情况下广播交易
如果需要，可以兑换。

此消息引入了`channel_id`来标识通道。它是通过组合“funding_txid”和“funding_output_index”从资金交易中获得的，使用big-endian异或（即`funding_output_index`改变最后2个字节）。

1. 类型：35（`funding_signed`）
2. 数据：
    * [`32`：`channel_id`]
    * [`64`：`signature`]

#### 要求

发件人必须设置：
  - `channel_id`是来自`funding_txid`和`funding_creput`消息的`funding_output_index`的异或。
  - `签名`到有效签名，使用其“funding_pubkey”进行初始承诺交易，如[BOLT＃3]（03-transactions.md＃commitment-transaction）中所定义。

收件人：
  - 如果`签名'不正确：
    - 必须通道失败。
  - 在收到有效的“funding_signed”之前，不得广播资金交易。
  - 在收到有效的“funding_signed”时：
    - 应该广播资金交易。

### `funding_locked`消息

此消息表明资金交易已达到`accept_channel`中要求的'minimum_depth`。一旦两个节点都发送了它，通道就进入正常操作模式。

1. 类型：36（`funding_locked`）
2. 数据：
    * [`32`：`channel_id`]
    * [`33`：`next_per_commitment_point`]

#### 要求

发件人必须：
  - 等到资金交易达成
发送此消息之前的“minimum_depth”。
  - 将`next_per_commitment_point`设置为
每承诺点用于以下承诺
事务，按照中的规定派生
[BOLT＃3]（03-transactions.md＃per-commitment-secret-requirements）。

非资助节点（fundee）：
  - 如果它看不到，应该忘记频道
合理超时后的资金交易。

从等待`funding_locked`开始，任一节点都可以
如果它没有收到来自的所需响应，则通道失败
合理超时后的其他节点。

#### 合理

非资助者可以简单地忘记曾经存在的频道，因为没有
资金面临风险。如果玩家永远记住频道，那么
会造成拒绝服务风险;因此，建议忘记它
（即使`push_msat`的承诺很重要）。

#### 未来

可以添加SPV证明，并且可以单独路由块哈希
消息。

## 频道关闭

节点可以协商连接的相互关闭，这与a不同
单方面关闭，允许他们立即获得资金
可以较低的费用协商。

关闭分两个阶段进行：
1. 一方表示它想要清除频道（因此不会接受新的HTLC）
2. 一旦所有HTLC都得到解决，最终的通道关闭协商就开始了。

        +-------+                              +-------+
        |       |--(1)-----  shutdown  ------->|       |
        |       |<-(2)-----  shutdown  --------|       |
        |       |                              |       |
        |       | <complete all pending HTLCs> |       |
        |   A   |                 ...          |   B   |
        |       |                              |       |
        |       |--(3)-- closing_signed  F1--->|       |
        |       |<-(4)-- closing_signed  F2----|       |
        |       |              ...             |       |
        |       |--(?)-- closing_signed  Fn--->|       |
        |       |<-(?)-- closing_signed  Fn----|       |
        +-------+                              +-------+

### 关闭启动：`shutdown`

两个节点（或两者）都可以发送`shutdown`消息来启动关闭，
它与`scriptpubkey`一起想要付钱。

1. 类型：38（`shutdown`）
2. 数据：
   * [`32`：`channel_id`]
   * [`2`：`len`]
   * [`len`：`scriptpubkey`]

#### 要求

发送节点：
  - 如果它没有发送`funding_created`（如果它是资助者）或'funding_signed`（如果它是一个fundee）：
    - 绝不能发送'shutdown`
  - 可以在“funding_locked”之前发送`shutdown`，即在资金交易达到“minimum_depth”之前。
  - 如果接收节点的承诺事务中有待更新的更新：
    - 绝不能发送'shutdown`。
  - 在'shutdown`之后不能发送`update_add_htlc`。
  - 如果任何承诺交易中都没有HTLC：
    - 在'shutdown`之后不得发送任何`update`消息。
  - 在发送`shutdown`之后，应该无法路由任何HTLC。
  - 如果它在`open_channel`或`accept_channel`中发送了一个非零长度的`shutdown_scriptpubkey`：
    - 必须在`scriptpubkey`中发送相同的值。
  - 必须以下列形式之一设置`scriptpubkey`：

    1. `OP_DUP``OP_HASH160``20`20-bytes`OP_EQUALVERIFY``OP_CHECKSIG`
   （支付给pubkey哈希），或者
    2. `OP_HASH160``20`20-bytes`OP_EQUAL`（支付给脚本哈希），或
    3. `OP_0``20`20-bytes（版本0付费见证pubkey），或
    4. `OP_0``32` 32字节（版本0支付见证脚本哈希）

接收节点：
  - 如果它没有收到`funding_signed`（如果它是资助者）或'funding_created`（如果它是一个fundee）：
    - 应该连接失败
  - 如果`scriptpubkey`不是上述形式之一：
    - 应该连接失败。
  - 如果还没有发送`funding_locked`：
    - 可以用`shutdown`回复`shutdown`消息
  - 一旦对等体上没有未完成的更新，除非它已经发送了'shutdown`：
    - 必须用`shutdown`回复`shutdown`消息
  - 如果两个节点都通告了`option_upfront_shutdown_script`功能，并且接收节点在`open_channel`或`accept_channel`中收到了非零长度的`shutdown_scriptpubkey`，那么`shutdown_scriptpubkey`不等于`scriptpubkey`：
    - 必须使连接失败。

#### 合理

当a时，如果通道状态总是“干净”（没有挂起的更改）
关闭开始，如果不是这样就避免了如何表现的问题：
发件人总是先发送一个`commitment_signed`。

由于关闭意味着终止的愿望，它意味着没有新的
HTLC将被添加或接受。一旦任何HTLC被清除，对等体
可能会立即开始结束谈判，因此我们禁止进一步更新
到承诺交易（特别是`update_fee`）
否则可能）。

`scriptpubkey`表单仅包含接受的标准表单
比特币网络，确保产生交易的意愿
传播给矿工。

`option_upfront_shutdown_script`功能表示该节点
想要预先提交`shutdown_scriptpubkey`以防万一
不知何故妥协了。这是一个微弱的承诺（一个恶意的
实现往往忽略像这样的规范！），但它
通过要求合作提供安全性的逐步改进
接收节点更改`scriptpubkey`。

`shutdown`响应要求意味着节点在回复之前发送`commitment_signed`来提交任何未完成的更改;但是，它理论上可以重新连接，这将简单地擦除所有未完成的未提交的更改。

### 结束谈判：`closing_signed`

一旦关闭完成并且通道没有HTLC，最后
目前的承诺交易将没有HTLC和关闭费用
谈判开始了。资助者选择它认为公平的费用，并且
使用来自的`scriptpubkey`字段签署关闭事务
`shutdown`消息（连同其选择的费用）并发送签名;
然后另一个节点回复类似，使用它认为合理的费用。这个
交换继续进行，直到双方同意收费或一方失败
这个频道。

1. type：39（`closing_signed`）
2. 数据：
   * [`32`：`channel_id`]
   * [`8`：`fee_satoshis`]
   * [`64`：`signature`]

#### 要求

资金节点：
  - 收到`shutdown`后，任何承诺交易中都没有HTLC：
    - 应该发送一个`closing_signed`消息。

发送节点：
  - 必须将`fee_satoshis`设置为小于或等于
 最终承诺交易的基本费用，在[BOLT＃3]（03-transactions.md＃fee-calculation）中计算。
  - 应该根据它设置最初的`fee_satoshis`
 估计一个区块的包含成本。
  - 必须将“签名”设置为关闭的比特币签名
 事务，如[BOLT＃3]（03-transactions.md＃closinging-transaction）中所述。

接收节点：
  - 如果`签名`对任何一个结算交易的变种无效
  在[BOLT＃3]中指定（03-transactions.md＃closinging-transaction）：
    - 必须使连接失败。
  - 如果`fee_satoshis`等于之前发送的'fee_satoshis`：
    - 应该签署并广播最终的结账交易。
    - 可以关闭连接。
  - 否则，如果`fee_satoshis`大于
最终承诺交易的基本费用
[BOLT＃3]（03-transactions.md＃fee-calculation）：
    - 必须使连接失败。
  - 如果`fee_satoshis`不严格
它最后发送的'fee_satoshis`和之前收到的
`fee_satoshis`，除非它重新连接：
    - 应该连接失败。
  - 如果接收方同意该费用：
    - 应该回复一个带有相同`fee_satoshis`值的`closing_signed`。
  - 除此以外：
    - 必须提出一个“严格地”在收到的`fee_satoshis`之间的值
  以及之前发送的`fee_satoshis`。

#### 合理

“严格之间”要求确保了前进
即使只是一次只有一个satoshi，也会取得进步。避免
保持状态并处理收费转移的角落案件
在断开连接和重新连接之间，协商会在重新连接时重新启动。

请注意，如果结算交易是风险，则风险有限
延迟了，但很快就会播出;所以通常没有
为快速处理支付溢价的原因。

## 普通手术

一旦两个节点都交换了“funding_locked”（以及可选的[`announcement_signatures`]（07-routing-gossip.md #the-announcement_signatures-message）），该频道可用于通过哈希时间锁定合同进行支付。

批量发送更改：在a之前发送一条或多条`update_`消息
`commitment_signed`消息，如下图所示：

        +-------+                               +-------+
        |       |--(1)---- update_add_htlc ---->|       |
        |       |--(2)---- update_add_htlc ---->|       |
        |       |<-(3)---- update_add_htlc -----|       |
        |       |                               |       |
        |       |--(4)--- commitment_signed --->|       |
        |   A   |<-(5)---- revoke_and_ack ------|   B   |
        |       |                               |       |
        |       |<-(6)--- commitment_signed ----|       |
        |       |--(7)---- revoke_and_ack ----->|       |
        |       |                               |       |
        |       |--(8)--- commitment_signed --->|       |
        |       |<-(9)---- revoke_and_ack ------|       |
        +-------+                               +-------+

与直觉相反，这些更新适用于*其他节点的*
承诺交易;节点只将这些更新添加到自己的更新中
远程节点确认它时的承诺事务
通过`revoke_and_ack`应用它们。

因此，每次更新都会遍历以下状态：

1. 等待接收器
2. 在接收方的最新承诺交易中
3. ...并且接收方先前的承诺交易已被撤销，
   并且发件人正在等待更新
4. ......以及发件人的最新承诺交易
5. ...并且发件人之前的承诺交易已被撤销


由于两个节点的更新是独立的，因此两个承诺
交易可能无限期地不同步。这不关心：
重要的是双方是否已经不可逆转地致力于
是否更新（最终状态，上面）。

### 转发HTLC

通常，节点提供HTLC有两个原因：发起自己的支付，
或者转发另一个节点的付款。在转发的情况下，必须小心
被采取以确保*传出* HTLC无法兑换，除非*传入*
HTLC可以兑换。以下要求确保始终如此。

在以下情况下，HTLC的相应**添加/删除**被视为*不可撤销地承诺*：

1. 承诺交易**有/没有**它由两个节点和任何一个承诺
先前承诺交易**没有/有**它已被撤销，或
2. 承诺交易**有/无**它已被不可逆转地承诺
区块链。

#### 要求

一个节点：
  - 直到传入的HTLC被不可撤销地提交：
    - 绝不能提供相应的传出HTLC（`update_add_htlc`）来响应传入的HTLC。
  - 直到删除传出的HTLC不可撤销地提交，或者直到通过HTLC超时事务（具有足够的深度）消耗了传出的链上HTLC输出：
    - 绝不能让对应的输入HTLC（`update_fail_htlc`）失败
对那个外向的HTLC。
  - 一旦达到传入HTLC的`cltv_expiry`，或者如果`cltv_expiry`减去`current_height`小于相应传出HTLC的`cltv_expiry_delta`：
    - 必须通过传入的HTLC（`update_fail_htlc`）失败。
  - 如果传入的HTLC的`cltv_expiry`在将来不合理的话：
    - 应该输入HTLC（`update_fail_htlc`）失败。
  - 在收到传出HTLC的`update_fulfill_htlc`时，或者在从链上HTLC支出中发现`payment_preimage`时：
    - 必须满足与输出HTLC相对应的输入HTLC。

#### 合理

一般来说，交易所的一方需要先处理另一方。
实现HTLC是不同的：根据定义，对于原像的了解是
不可撤销和即将到来的HTLC应该尽快完成
减少延迟。

具有不合理的长期到期的HTLC是拒绝服务向量和
因此是不允许的。请注意，“不合理”的确切值目前尚不清楚
并且可能取决于网络拓扑。

### `cltv_expiry_delta`选择

一旦HTLC超时，它就可以实现或超时;
对于提供和接收的HTLC，必须注意这种转变。

考虑以下场景，其中A向B发送HTLC，谁
转发到C，付款后立即交付货物
接收。

1. C需要确保来自B的HTLC不能超时，即使B变为
   反应迟钝;即C可以在B可以之前完成输入的HTLC链
   把它计时在链上。

2. B需要确保如果C从B满足HTLC，它就可以实现
   来自A的传入HTLC;即B可以从C获得原像并完成传入
   在A之前的HTLC on-chain可以在链上计时。

这里的关键设置是`cltv_expiry_delta`
[BOLT＃7]（07-routing-gossip.md #the-channel_update-message）和
在[BOLT＃11]（11-payment-encoding.md #tagged-fields）中相关的`min_final_cltv_expiry`。
`cltv_expiry_delta`是HTLC CLTV超时的最小差异
转发案例（B）。 `min_final_cltv_expiry`是最小的差异
在HTLC CLTV超时和当前块高度之间
终端案例（C）。

请注意，如果某个频道的此值太低，则风险仅为
节点*接受* HTLC，而不是提供它的节点。为了这
原因是，* outgoing *通道的`cltv_expiry_delta`用作
跨节点的增量。

传出和传出之间的最坏情况块数
在给出一些假设的情况下，可以导出传入的HTLC分辨率：

* 最糟糕的重组深度`R`块
* 在放弃之前，HTLC超时之后的宽限期“G”会阻塞
  一个没有反应的同伴并且掉到了连锁店
* 交易广播和交易广播之间的多个块`S`
  交易包含在一个区块中

最坏的情况是转发节点（B）花费的时间最长
可能的时间来发现即将离任的HTLC履行并且也需要
在链上兑换它的最长时间：

1. B-> C HTLC在块“N”处超时，B等待“G”块直到
   它放弃等待C. B或C提交到区块链，
   和B花费HTLC，它包含`S`块。
2. 不好的情况：C赢得比赛（正好）并完成HTLC，B只看到
   当它看到块'N + G + S + 1`时的那个事务。
3. 最糟糕的情况：重组“R”深入其中C胜出
   满足。 B只看到'N + G + S + R`处的交易。
4. B现在需要完成传入的A-> B HTLC，但A没有响应：B等待'G`更多
   在放弃等待A之前的块.A或B提交到区块链。
5. 不好的情况：B在块'N + G + S + R + G + 1`中看到A的承诺交易并且有
   花费HTLC输出，需要开采`S`块。
6. 最糟糕的情况：A使用了另一个重组“R”深度
   花费承诺交易，所以B看到A的承诺
   块'N + G + S + R + G + R`中的事务并且必须花费HTLC输出
   将'S`块打开。
7. B的HTLC花费需要在超时之前至少达到“R”深度，
   否则另一次重组可能会让A超时
   交易。

因此，最坏的情况是`3R + 2G + 2S`，假设`R`至少为1.注意
三个重组的机会是另一个节点赢得所有这些重组
低于'R`为2或更高。由于使用了高额费用（HTLC花费可以使用
几乎是乱收费），`S`应该很小;但是，鉴于阻止时间是
仍然会出现不规则和空的块，`S = 2`应该被认为是a
最小。类似地，宽限期“G”可以是低（1或2），如节点所示
要求尽快超时或履行;但如果`G`太低，它会增加
由于网络延迟导致不必要的频道关闭的风险。

需要导出四个值：

1. 通道的`cltv_expiry_delta`，`3R + 2G + 2S`：如有疑问，a
   `cltv_expiry_delta`为12是合理的（R = 2，G = 1，S = 2）。

2. 提供HTLC的截止日期：通道必须失败的截止日期
   并在链上超时。这是HTLC之后的“G”块
   `cltv_expiry`：1块是合理的。

3. 此节点已完成的收到HTLC的截止日期：截止日期之后
渠道必须失败，HTLC在其之前完成链上
   `cltv_expiry`。参见上面的步骤4-7，这意味着“2R + G + S”的截止日期
   在`cltv_expiry`之前的块：7个块是合理的。

4. 终端支付接受的最小`cltv_expiry`：
   终端节点C的最坏情况是“2R + G + S”块（再次，步骤
   1-3以上不适用）。默认值为
   [BOLT＃11]（11-payment-encoding.md）是9，稍微多一点
   比这个计算所暗示的更为保守。

#### 要求

提供节点：
  - 必须估计它提供的每个HTLC的超时截止日期。
  - 绝不能在它的`cltv_expiry`之前提供一个超时截止日期的HTLC。
  - 如果它提供的HTLC在任一节点的当前
  承诺交易，AND超过此超时截止日期：
    - 必须通道失败。

一个充实的节点：
  - 对于它试图实现的每个HTLC：
    - 必须估计一个履行截止日期。
  - 必须失败（而不是转发）HTLC，其履行期限已经过去。
  - 如果它已经实现的HTLC在任一节点的当前承诺中
  交易，AND已超过此履行截止日期：
    - 必须使连接失败。

### 添加HTLC：`update_add_htlc`

任何一个节点都可以发送`update_add_htlc`来向另一个提供HTLC，
这是可兑换的，以换取付款原像。金额在
millisatoshi，虽然链上执法只适用于整体
satoshi数量大于尘埃限制（在承诺交易中，这些数据向下舍入为
在[BOLT＃3]（03-transactions.md）中指定。

`onion_routing_packet`部分的格式，表示付款的位置
注定，在[BOLT＃4]（04-onion-routing.md）中有描述。

1. 类型：128（`update_add_htlc`）
2. 数据：
   * [`32`：`channel_id`]
   * [`8`：`id`]
   * [`8`：`amount_msat`]
   * [`32`：`payment_hash`]
   * [`4`：`cltv_expiry`]
   * [`1366`：`onion_routing_packet`]

#### 要求

发送节点：
  - 绝不能提供它无法支付的“amount_msat”
当前`feerate_per_kw`的远程承诺交易（参见“更新
费用“）同时保持其渠道储备。
  - 必须提供大于0的`amount_msat`。
  - 绝不能在接收节点的`htlc_minimum_msat`下面提供`amount_msat`
  - 必须将`cltv_expiry`设置为小于500000000。
  - 对于带有'chain_hash`标识比特币区块链的渠道：
    - 必须将`amount_msat`的四个最重要的字节设置为0。
  - 如果结果将提供超过遥控器
  `max_accepted_htlcs` HTLCs，在远程承诺交易中：
    - 绝不能添加HTLC。
  - 如果提供的HTLC总数超过遥控器的总和
`max_htlc_value_in_flight_msat`：
    - 绝不能添加HTLC。
  - 对于它提供的第一个HTLC：
    - 必须将`id`设置为0。
  - 必须为每个连续的报价将`id`的值增加1。

更新完成后（即`revoke_and_ack`之后），`id`不能重置为0
已收到）。它必须继续递增。

接收节点：
  - 接收`amount_msat`等于0，或者小于它自己的`htlc_minimum_msat`：
    - 应该通道失败。
  - 在当前的“feerate_per_kw”接收到发送节点无法承受的“amount_msat”（同时保持其频道预留）：
    - 应该通道失败。
  - 如果发送节点添加超过接收者`max_accepted_htlcs` HTLCs
    它的本地承诺交易，OR增加了超过接收者`max_htlc_value_in_flight_msat`值的提供的HTLC到其本地承诺交易：
    - 应该通道失败。
  - 如果发送节点将`cltv_expiry`设置为大于或等于500000000：
    - 应该通道失败。
  - 对于带有'chain_hash`标识比特币区块链的通道，如果`amount_msat`的四个最重要的字节不是0：
    - 必须通道失败。
  - 必须允许多个HTLC具有相同的“payment_hash”。
  - 如果发件人之前没有承认HTLC的承诺：
    - 必须在重新连接后忽略重复的`id`值。
  - 如果发生其他“id”违规行为：
    - 可能会失败频道。

`onion_routing_packet`包含一个混淆的跳跃列表和沿路径的每一跳的指令。
它通过将“payment_hash”设置为关联数据来提交给HTLC，即在HMAC的计算中包括“payment_hash”。
这可以防止重用攻击，这些攻击会重复使用不同的`payment_hash`之前的`onion_routing_packet`。

#### 合理

无效金额是明确的协议违规并指示故障。

如果一个节点不接受具有相同支付哈希值的多个HTLC，则为
攻击者可以探测节点是否有现有的HTLC。这个
要求，处理重复，导致使用单独的
标识符;它假设一个64位计数器永远不会包装。

明确允许重传未确认的更新
重新连接的目的;允许他们在其他时间简化
收件人代码（虽然严格的检查可能有助于调试）。

`max_accepted_htlcs`限制为483，以确保即使两者兼而有之
双方发送最大数量的HTLC，`commitment_signed`消息将
仍然在最大邮件大小。它也确保了这一点
单一罚款交易可以花费整个承诺交易，
在[BOLT＃5]中计算（05-onchain.md＃penalty-transaction-weight-calculation）。

`cltv_expiry`值等于或大于500000000表示时间
秒，并且协议仅支持块中的到期。

`amount_msat`是故意限制的
规格;在这期间，更大的数量是不必要的，也不是明智的
网络的自举阶段。

### 删除HTLC：`update_fulfill_htlc`，`update_fail_htlc`和`update_fail_malformed_htlc`

为简单起见，节点只能删除其他节点添加的HTLC。
删除HTLC有四个原因：提供付款前映像，
它已超时，无法路由，或者格式不正确。

提供原像：

1. 类型：130（`update_fulfill_htlc`）
2. 数据：
   * [`32`：`channel_id`]
   * [`8`：`id`]
   * [`32`：`payment_preimage`]

对于超时或路由失败的HTLC：

1. 类型：131（`update_fail_htlc`）
2. 数据：
   * [`32`：`channel_id`]
   * [`8`：`id`]
   * [`2`：`len`]
   * [`len`：`reason`]

`reason`字段是一个不透明的加密blob，有利于
原始HTLC引发剂，如[BOLT＃4]（04-onion-routing.md）中所定义;
但是，对于这种情况，存在特殊的畸形故障变体
对等体无法解析它：在这种情况下，当前节点采取行动，加密
它进入`update_fail_htlc`进行中继。

对于不可解决的HTLC：

1. 类型：135（`update_fail_malformed_htlc`）
2. 数据：
   * [`32`：`channel_id`]
   * [`8`：`id`]
   * [`32`：`sha256_of_onion`]
   * [`2`：`failure_code`]

#### 要求

一个节点：
  - 应该尽快删除HTLC。
  - 应该失败的HTLC已经超时了。
  - 直到相应的HTLC在双方都不可撤销地承诺
  承诺交易：
    - 绝不能发送`update_fulfill_htlc`，`update_fail_htlc`，或
`update_fail_malformed_htlc`。

接收节点：
  - 如果`id`与其当前承诺交易中的HTLC不对应：
    - 必须通道失败。
  - 如果`update_fulfill_htlc`中的`payment_preimage`值
  没有SHA256哈希到相应的HTLC`hayment_hash`：
    - 必须通道失败。
  - 如果没有设置`failure_code`中的`BADONION`位
  `update_fail_malformed_htlc`：
    - 必须通道失败。
  - 如果`update_fail_malformed_htlc`中的`sha256_of_onion`与
  洋葱它发送：
    - 可以重试或选择备用错误响应。
  - 否则，一个接收节点有一个由`update_fail_malformed_htlc`取消的传出HTLC：
    - 必须在发送到链接的`update_fail_htlc`中返回错误
      最初发送HTLC，使用给出的`failure_code`并设置
      数据到`sha256_of_onion`。

#### 合理

没有超时HTLC的节点存在通道故障的风险（参见
[`cltv_expiry_delta` Selection]（＃cltv_expiry_delta-selection））。

在发送者之前发送`update_fulfill_htlc`的节点也是
致力于HTLC并冒着失去资金的风险。

如果洋葱格式不正确，上游节点将无法提取
生成响应的共享密钥 - 因此是特殊的失败消息
使这个节点做到这一点。

节点可以检查上游正在抱怨的SHA256
确实匹配它发送的洋葱，这可能允许它检测随机位
错误。但是，如果不重新检查发送的实际加密数据包，
它不会知道错误是它自己还是遥控器;所以
这种检测作为选项留下。

### 到目前为止提交更新：`commitment_signed`

当节点有远程承诺的更改时，它可以应用它们，
签署生成的事务（在[BOLT＃3]（03-transactions.md）中定义），并发送一个
`commitment_signed`消息。

1. 类型：132（`commitment_signed`）
2. 数据：
   * [`32`：`channel_id`]
   * [`64`：`signature`]
   * [`2`：`num_htlcs`]
   * [`num_htlcs * 64`：`htlc_signature`]

#### 要求

发送节点：
  - 绝不能发送不包含任何内容的“commitment_signed”消息
更新。
  - 可以只发送一个`commitment_signed`消息
改变费用。
  - 可以发送一个没有的'commitment_signed`消息
除了新的撤销号码之外，更改承诺交易
（由于灰尘，相同的HTLC替换，或无关紧要或多重
费用变动）。
  - 必须为每个对应的HTLC事务包含一个`htlc_signature`
    到承诺交易的顺序（参见[BOLT＃3]（03-transactions.md #transaction-input-and-output-ordering））。
  - 如果它最近没有收到来自远程节点的消息：
      - 在发送`commitment_signed`之前，应该使用`ping`并等待回复`pong`。

接收节点：
  - 一旦应用了所有挂起的更新：
    - 如果`签名`对其本地承诺交易无效：
      - 必须通道失败。
    - 如果`num_htlcs`不等于本地的HTLC输出数
    承诺交易：
      - 必须通道失败。
  - 如果任何`htlc_signature`对相应的HTLC交易无效：
    - 必须通道失败。
  - 必须用`revoke_and_ack`消息回复。

#### 合理

提供垃圾邮件更新毫无意义：它意味着一个错误。

`num_htlcs`字段是冗余的，但是使数据包长度检查完全自包含。

要求最近消息的建议承认现实
网络不可靠：节点可能没有意识到他们的同行
离线，直到发送`commitment_signed`。一旦
发送`commitment_signed`，发送者认为自己受到约束
那些HTLC，并且不会失败相关的传入HTLC，直到
输出HTLC完全解析。

### 完成向更新状态的转换：`revoke_and_ack`

一旦`commitment_signed`的接收者检查签名并且知道
它有一个有效的新承诺交易，它回复承诺
'revoke_and_ack`中前一个承诺事务的preimage
信息。

此消息也隐含地用作确认收据
`commitment_signed`，这是`commitment_signed`发送者的逻辑时间
申请（自己承诺）之前发送的任何未决更新
那个`commitment_signed`。

密钥推导的描述在[BOLT＃3]（03-transactions.md＃key-derivation）中。

1. 类型：133（`revoke_and_ack`）
2. 数据：
   * [`32`：`channel_id`]
   * [`32`：`per_commitment_secret`]
   * [`33`：`next_per_commitment_point`]

#### 要求

发送节点：
  - 必须将`per_commitment_secret`设置为用于生成密钥的秘密
  先前的承诺交易。
  - 必须将`next_per_commitment_point`设置为其下一个承诺的值
  交易。

接收节点：
  - 如果`per_commitment_secret`没有生成以前的`per_commitment_point`：
    - 必须通道失败。
  - 如果[per_commitment_secret`不是由[BOLT＃3]中的协议生成的（03-transactions.md＃per-commitment-secret-requirements）：
    - 可能会失败频道。

一个节点：
  - 绝不能广播旧的（已撤销的）承诺交易，
    - 注意：这样做将允许其他节点获取所有渠道资金。
  - 除非即将播出，否则不应签署承诺交易
  他们（由于连接失败），
    - 注意：这是为了降低上述风险。

### 更新费用：`update_fee`

“update_fee”消息由支付的节点发送
比特币费用。像任何更新一样，它首先致力于接收器
承诺交易然后（一旦承认）致力于
发件人。与HTLC不同，`update_fee`永远不会关闭，只是简单
更换。

有可能发生竞赛，因为接收者可以添加新的HTLC
在收到`update_fee`之前。在这种情况下，发件人可以
一旦“update_fee”，就无法承担自己承诺交易的费用
最终得到了收件人的承认。在这种情况下，费用将会减少
比费率，如[BOLT＃3]（03-transactions.md＃fee-payment）中所述。

用于从费率中获得费用的确切计算是
在[BOLT＃3]中给出（03-transactions.md＃fee-calculation）。

1. 类型：134（`update_fee`）
2. 数据：
   * [`32`：`channel_id`]
   * [`4`：`feerate_per_kw`]

#### 要求

用于支付比特币费用的节点_responsible_：
  - 应该发送`update_fee`以确保当前的费率是足够的（通过a
      及时处理承诺交易的重大利润。

节点_不负责支付比特币费用：
  - 绝不能发送`update_fee`。

接收节点：
  - 如果`update_fee`太低而无法及时处理，或者OR太大了：
    - 应该通道失败。
  - 如果发件人不负责支付比特币费用：
    - 必须通道失败。
  - 如果发送方无法承担接收节点上的新费率
  当前承诺交易：
    - 应该通道失败，
      - 但是可以延迟这个检查，直到`update_fee`被提交。

#### 合理

单方面关闭需要比特币费用才有效 - 
特别是因为没有广播节点使用的通用方法
儿童 - 为父母支付以增加其有效费用。

鉴于费用的差异以及交易可能的事实
在未来度过，收费者保持良好状态是一个好主意
保证金（比预期费用要求的5倍）;但是，由于方法的不同
费用估算，未指定准确值。

由于费用目前是片面的（要求的费用的一方）
渠道创建总是支付承诺交易的费用），
最简单的是只允许它设定费用水平;然而，同样如此
费率适用于HTLC交易，接收节点也必须
关心费用的合理性。

## 消息重传

因为通信传输不可靠，可能需要
不时重新建立，运输的设计已经
明确地与协议分开。

尽管如此，我们认为我们的运输是有序且可靠的。
重新连接引起了对收到的内容的怀疑，所以有
那时的明确的确认。

在频道建立的情况下，这是相当简单的
并且关闭，消息具有明确的顺序，但在正常情况下
操作，更新的确认被推迟到
`commitment_signed` /`revoke_and_ack` exchange;所以不能假设
已收到更新。这也意味着接收
节点只需在收到`commitment_signed`时存储更新。

请注意，[BOLT＃7]（07-routing-gossip.md）中描述的消息是
独立于特定渠道;他们的传输要求
被覆盖在那里，除了在'init`之后传播（如同所有
消息是），它们与这里的要求无关。

1. 类型：136（`channel_reestablish`）
2. 数据：
   * [`32`：`channel_id`]
   * [`8`：`next_local_commitment_number`]
   * [`8`：`next_remote_revocation_number`]
   * [`32`：`your_last_per_commitment_secret`]（option_data_loss_protect）
   * [`33`：`my_current_per_commitment_point`]（option_data_loss_protect）

`next_local_commitment_number`：承诺号是48位
每个承诺交易的递增计数器;计数器
对于通道中的每个对等体是独立的并且从0开始。
在这种情况下，它们只是显式地转发到另一个节点
重建，否则他们是隐含的。


### 要求

资金节点：
  - 断开时：
    - 如果它已经广播了融资交易：
      - 必须记住重新连接的通道。
    - 除此以外：
      - 不应该记住重新连接的渠道。

非资金节点：
  - 断开时：
    - 如果它已发送`funding_signed`消息：
      - 必须记住重新连接的通道。
    - 除此以外：
      - 不应该记住重新连接的渠道。

一个节点：
  - 必须处理新加密传输上的前一个频道的延续。
  - 断开时：
    - 必须反转另一方发送的任何未提交的更新（即全部
    以`update_`开头的消息，没有`commitment_signed`
    已收到）。
      - 注意：节点可能已经使用了来自的“payment_preimage”值
    `update_fulfill_htlc`，所以`update_fulfill_htlc`的效果不是
    完全逆转。
  - 重新连接时：
    - 如果通道处于错误状态：
      - 应该重新传输错误数据包并忽略任何其他数据包
      那个频道。
    - 除此以外：
      - 必须为每个频道传输`channel_reestablish`。
      - 必须等待接收另一个节点的`channel_reestablish`
        发送该频道的任何其他消息之前的消息。

发送节点：
  - 必须将`next_local_commitment_number`设置为的承诺号
  它希望收到的下一个`commitment_signed`。
  - 必须将`next_remote_revocation_number`设置为的承诺号
  它希望收到的下一个`revoke_and_ack`消息。
  - 如果它支持`option_data_loss_protect`：
    - 如果`next_remote_revocation_number`等于0：
      - 必须将`your_last_per_commitment_secret`设置为全零
    - 除此以外：
      - 必须将`your_last_per_commitment_secret`设置为最后一个`per_commitment_secret`
    它收到了

一个节点：
  - 如果`next_local_commitment_number`在`channel_reestablish`中都是1
  发送和接收：
    - 必须重新发送`funding_locked`。
  - 除此以外：
    - 绝不能重新发送`funding_locked`。
  - 重新连接时：
    - 必须忽略它收到的任何冗余的“funding_locked”。
  - 如果`next_local_commitment_number`等于承诺号
  接收节点发送的最后一个`commitment_signed`消息：
    - 必须为下一个`commitment_signed`重用相同的承诺号。
  - 除此以外：
    - 如果`next_local_commitment_number`不大于1
  收到的最后一个'commitment_signed`消息的承诺号
  节点已发送：
      - 应该通道失败。
    - 如果它还没有发送`commitment_signed`和`next_local_commitment_number`
    不等于1：
      - 应该通道失败。
  - 如果`next_remote_revocation_number`等于承诺号
  接收节点发送的最后一个`revoke_and_ack`，以及接收节点
  还没有收到`closing_signed`：
    - 必须重新发送`revoke_and_ack`。
  - 除此以外：
    - 如果`next_remote_revocation_number`不等于1大于
    接收节点发送的最后一个`revoke_and_ack`的承诺号：
      - 应该通道失败。
    - 如果它还没有发送`revoke_and_ack`和`next_remote_revocation_number`
    不等于0：
      - 应该通道失败。

 接收节点：
  - 如果它支持`option_data_loss_protect`，那么'option_data_loss_protect`
  字段存在：
    - 如果`next_remote_revocation_number`大于上面的预期，AND
    `your_last_per_commitment_secret`对此是正确的
    `next_remote_revocation_number`减去1：
      - 绝不能广播其承诺交易。
      - 应该通道失败。
      - 应该存储`my_current_per_commitment_point`来检索资金
        如果发送节点在链上广播其承诺事务。
    - 否则（`your_last_per_commitment_secret`或`my_current_per_commitment_point`
    与预期值不匹配）：
      - 应该通道失败。

一个节点：
  - 绝不能假设先前传输的消息丢失了，
    - 如果它已发送先前的'commitment_signed`消息：
      - 务必处理相应承诺交易的情况
      任何时候由另一方广播，
        - 注意：如果节点不简单，这一点尤为重要
        重新发送先前发送的确切的`update_`消息。
  - 重新连接时：
    - 如果它已经发送了一个先前的`shutdown`：
      - 必须重新传输`shutdown`。

### 合理

上述要求确保开放阶段接近
原子：如果没有完成，它会再次启动。唯一的例外
是否发送了'funding_signed`消息但未收到。在
在这种情况下，资助者会忘记频道，并且可能会打开
重新连接时新的;同时，另一个节点最终会忘记
由于从未收到过“funding_locked”或看到的原始频道
资金交易在链上。

没有确认“错误”，所以如果发生重新连接则是
在重新断开连接之前重新发送礼貌;但是，它不是必须的，
因为也有节点可以简单地忘记的情况
完全是渠道。

`closing_signed`也没有确认，所以必须重新传输
在重新连接时（虽然协商在重新连接时重新启动，因此它需要
不是一个精确的转播）。
对`shutdown`的唯一确认是`closing_signed`，所以一个或另一个
需要重传。

更新的处理类似于原子：如果提交不是
确认（或未发送）更新被重新发送。但事实并非如此
坚持认为它们是相同的：它们可能处于不同的顺序，
涉及不同的费用，甚至丢失现在太旧的HTLC
待补充。要求它们是相同的，实际上意味着a
每次传输时发送方写入磁盘，而方案
这里鼓励对每个磁盘执行一次持久的写入操作
发送或接收`commitment_signed`。

在a之后永远不应该要求重新传送`revoke_and_ack`
已收到`closing_signed`，因为这意味着已经关闭了
已完成 - 只能在收到`revoke_and_ack`之后发生
由远程节点。

注意，`next_local_commitment_number`从1开始，因为
承诺编号0在开业期间创建。
`next_remote_revocation_number`将为0，直到
收到承诺编号1的`commitment_signed`，在此处
指出承诺号0的撤销被发送。

正常情况下，“funding_locked”被隐含地承认
操作，已知在“commitment_signed”之后开始
收到 - 因此，对`next_local_commitment_number`的测试更大
比1。

之前的草案坚持认为资助者“必须记住......如果有的话
广播资金交易，否则它不得“：这是在
事实上是不可能的要求节点必须首先提交
磁盘，然后广播事务，反之亦然。新的
语言反映了这一现实：记住一个人肯定会更好
没有广播的频道比忘记一个！
同样，对于fundee的'funding_signed`消息：它更好
记住一个永远不会打开（和超时）的频道
当fundee忘记它时，资助者打开它。

添加了`option_data_loss_protect`以允许某个节点落后于某个节点
（例如已从旧备份中恢复），以检测它是否落后。倒下了
节点必须知道它不能广播其当前的承诺交易 - 这将导致
资金全部损失 - 因为远程节点可以证明它知道了
撤销原像。落后节点返回的错误
（或者只是它所拥有的`channel_reestablish`中的无效数字
应该使另一个节点放弃其当前的承诺
交易到连锁店。这将至少允许倒下的节点恢复
非HTLC资金，如果是'my_current_per_commitment_point`
已验证。然而，这也意味着倒下的节点已经揭示了这一点
事实（虽然不可证明：它可能在撒谎），而另一个节点可以使用它
播放以前的状态。

# 作者

[FIXME：插入作者列表]

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
