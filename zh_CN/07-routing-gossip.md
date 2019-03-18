# BOLT＃7：P2P节点和频道发现

该规范描述了简单的节点发现，信道发现和信道更新机制，它们不依赖于第三方来传播信息。

节点和通道发现有两个不同的用途：

 - 通道发现允许创建和维护网络拓扑的本地视图，以便节点可以发现到达所需目标的路由。
 - 节点发现允许节点广播其ID，主机和端口，以便其他节点可以打开连接并与它们建立支付渠道。

为了支持频道发现，支持三个*八卦消息*。在网络交换中的同行
`channel_announcement`消息包含有关新的信息
两个节点之间的通道。他们还可以交换`channel_update`
消息，用于更新有关频道的信息。只有
任何渠道都有一个有效的`channel_announcement`，但至少有两个
期望`channel_update`消息。

为了支持节点发现，对等体交换`node_announcement`
消息，提供有关节点的其他信息。可能有
多个`node_announcement`消息，以便更新节点信息。

# 目录

  * [`announcement_signatures`消息](#the-announcement_signatures-message)
  * [`channel_announcement`消息](#the-channel_announcement-message)
  * [`node_announcement`消息](#the-node_announcement-message)
  * [`channel_update`消息](#the-channel_update-message)
  * [查询消息](#query-messages)
  * [初始同步](#initial-sync)
  * [转播](#rebroadcasting)
  * [HTLC费用](#htlc-fees)
  * [修剪网络视图](#pruning-the-network-view)
  * [路由建议](#recommendations-for-routing)
  * [参考](#references)

## `announcement_signatures`消息

这是信道的两个端点之间的直接消息，并且用作允许将信道通告给网络其余部分的选择加入机制。
它包含发送者必要的签名，以构造`channel_announcement`消息。

1. 类型：259（`announcement_signatures`）
2. 数据：
    * [`32`：`channel_id`]
    * [`8`：`short_channel_id`]
    * [`64`：`node_signature`]
    * [`64`：`bitcoin_signature`]

通过在“channel_flags”中设置`announce_channel`位，在信道打开期间发信号通知发起节点宣告信道的意愿（参见[BOLT＃2]（02-peer-protocol.md #the-open_channel-message））。

### 要求

通过构造对应于新建立的信道的“channel_announcement”消息，并使用与端点的“node_id”和“bitcoin_key”匹配的秘密对其进行签名来创建`announcement_signatures`消息。签字后，
可以发送`announcement_signatures`消息。

`short_channel_id`是融资交易的唯一描述。
它的构造如下：
  1. 最重要的3个字节：表示块高度
  2. 接下来的3个字节：表示块内的事务索引
  3. 最不重要的2个字节：表示支付给频道的输出索引。

创建了“short_channel_id”的标准人类可读格式
按以下顺序打印上述组件：
块高度，事务索引和输出索引。
每个组件都以十进制数打印，
并用小写字母`x`分开。
例如，`short_channel_id`可能被写为`539268x845x1`，
在索引845处指示交易的输出1上的信道
块高度539268。

一个节点：
  - 如果`open_channel`消息设置了'announce_channel`位并且还没有发送`shutdown`消息：
    - 必须发送`announcement_signatures`消息。
      - 不得发送`announcement_signatures`消息，直到`funding_locked`
      已发送和收到并且资金交易至少有六个确认。
  - 除此以外：
    - 绝不能发送`announcement_signatures`消息。
  - 重新连接后（一旦满足上述时间要求）：
    - 必须用自己的第一个`announcement_signatures`消息响应
    `announcement_signatures`消息。
    - 如果它没有收到`announcement_signatures`消息：
      - 应该重新传输`announcement_signatures`消息。

收件人节点：
  - 如果`node_signature`或`bitcoin_signature`不正确：
    - 可能会失败频道。
  - 如果已发送并收到有效的`announcement_signatures`消息：
    - 应该为其对等体排队`channel_announcement`消息。
  - 如果它还没有发送funding_locked：
    - 可以推迟处理announcement_signatures，直到它发送了funding_locked
    - 除此以外：
      - 必须忽略它。


### 合理

允许推迟过早的announcement_signatures的原因是
该规范的早期版本不需要等待收到
资金锁定：推迟而不是忽略它允许兼容
这种行为。

设计了`short_channel_id`人类可读格式
因此，双击或双击它将选择整个ID
在大多数系统上。
人们在阅读数字时更喜欢小数，
因此ID组件以十进制形式写入。
从大多数字体开始使用小写字母`x`，
`x`明显小于十进制数字，
可以轻松地对ID的每个组成部分进行分组。

## `channel_announcement`消息

该八卦消息包含有关频道的所有权信息。它有联系
每个on-chain比特币密钥到关联的Lightning节点密钥，反之亦然。
在至少一方宣布之前，该频道实际上并不可用
它的费用水平和到期时间，使用`channel_update`。

证明`node_1`和`node_2`之间存在通道需要：

1. 证明资金交易支付给'bitcoin_key_1`和
   `bitcoin_key_2`
2. 证明`node_1`拥有`bitcoin_key_1`
3. 证明`node_2`拥有`bitcoin_key_2`

假设所有节点都知道未使用的事务输出，则第一个证明是
由一个节点找到由`short_channel_id`和。给出的输出完成的
验证它确实是这些密钥的P2WSH资金交易输出
在[BOLT＃3]（03-transactions.md＃funding-transaction-output）中指定。

最后两个证明是通过显式签名完成的：
为每个生成`bitcoin_signature_1`和`bitcoin_signature_2`
`bitcoin_key`和每个相应的`node_id`都是签名的。

还有必要证明`node_1`和`node_2`都同意
公告信息：这是通过每个人签名来完成的
签署消息的`node_id`（`node_signature_1`和`node_signature_2`）。

1. 类型：256（`channel_announcement`）
2. 数据：
    * [`64`：`node_signature_1`]
    * [`64`：`node_signature_2`]
    * [`64`：`bitcoin_signature_1`]
    * [`64`：`bitcoin_signature_2`]
    * [`2`：`len`]
    * [`len`：`features`]
    * [`32`：`chain_hash`]
    * [`8`：`short_channel_id`]
    * [`33`：`node_id_1`]
    * [`33`：`node_id_2`]
    * [`33`：`bitcoin_key_1`]
    * [`33`：`bitcoin_key_2`]

### 要求

原点节点：
  - 必须将`chain_hash`设置为唯一标识链的32字节哈希
  该频道在以下内容开放：
    - _Bitcoin区块链_：
      - 必须设置`chain_hash`值（以十六进制编码）等于`6fe28c0ab6f1b372c1a6a246ae63f74f931e8365e15a089c68d6190000000000`。
  - 必须设置`short_channel_id`来参考确认的融资交易，
  如[BOLT＃2]（02-peer-protocol.md #the-funding_locked-message）中所述。
    - 注意：相应的输出必须是P2WSH，如[BOLT＃3]（03-transactions.md＃funding-transaction-output）中所述。
  - 必须将`node_id_1`和`node_id_2`设置为两个节点的公钥
  操作通道，使得`node_id_1`在数字上较小
  两个DER编码的键按升序排序。
  - 必须将`bitcoin_key_1`和`bitcoin_key_2`设置为`node_id_1`和`node_id_2`
  各自`funding_pubkey`s。
  - 必须从偏移量开始计算消息的双SHA256哈希“h”
  256，直到消息结束。
    - 注意：哈希跳过4个签名，但哈希消息的其余部分，
    包括附加到末尾的任何未来字段。
  - 必须将`node_signature_1`和`node_signature_2`设置为有效
    hash`h`的签名（使用`node_id_1`和`node_id_2`各自的
    秘密）。
  - 必须将`bitcoin_signature_1`和`bitcoin_signature_2`设置为有效
  hash`h`的签名（使用`bitcoin_key_1`和`bitcoin_key_2`'
  各自的秘密）。
  - 应该将`len`设置为保存`features`位所需的最小长度
  它设定。

接收节点：
  - 必须通过验证来验证消息的完整性和真实性
  签名。
  - 如果`features`字段中有一个未知的偶数位：
    - 绝不能解析消息的其余部分。
    - 绝不能将频道添加到其本地网络视图。
    - 不应该转发公告。
  - 如果`short_channel_id`的输出不对应P2WSH（使用
    `bitcoin_key_1`和`bitcoin_key_2`，如
    [BOLT＃3]（03-transactions.md＃funding-transaction-output））或者输出是
    花费：
    - 必须忽略该消息。
  - 如果接收者不知道指定的`chain_hash`：
    - 必须忽略该消息。
  - 除此以外：
    - 如果是'bitcoin_signature_1`，`bitcoin_signature_2`，`node_signature_1`或
    `node_signature_2`无效或不正确：
      - 应该连接失败。
    - 除此以外：
      - 如果`node_id_1`或`node_id_2`被列入黑名单：
        - 应该忽略这条消息。
      - 除此以外：
        - 如果提到的交易之前没有宣布为
        渠道：
          - 应该将消息排队以进行重播。
          - 可以选择NOT来处理超过预期最小值的消息
          长度。
      - 如果它以前收到过有效的`channel_announcement`，那么
      相同的事务，在同一个块中，但是对于不同的`node_id_1`或
      `node_id_2`：
        - 应该将上一条消息的'node_id_1`和`node_id_2`列入黑名单，
        以及这个`node_id_1`和`node_id_2`并忘记任何频道
        连接到他们。
      - 除此以外：
        - 应该存储这个`channel_announcement`。
  - 一旦其资金产出已用完或重组：
    - 应该忘记一个频道。

### 合理

两个节点都需要签名以表明他们愿意路由其他节点
通过该渠道付款（即成为公共网络的一部分）;需要他们的
比特币签名证明他们控制频道。

冲突节点的黑名单不允许多种不同
公告。这种相互矛盾的公告绝不应该由任何人播出
节点，因为这意味着密钥已泄露。

虽然渠道不应该在它们足够深之前做广告，但是
反对转播的要求仅适用于交易未移动的情况
到另一个街区。

为了避免存储过大的消息，仍然允许
合理的未来扩展，允许节点限制转播
（也许在统计上）。

未来可能有新的频道功能：向后兼容（或
可选）功能将具有_odd_功能位，同时具有不兼容的功能
将有_even_功能位
（[“可以奇怪！”]（00-introduction.md＃glossary-and-terminology-guide））。
不兼容的功能将导致通知未被转发
不理解它们的节点。

## `node_announcement`消息

这个八卦消息允许节点指示与其相关的额外数据
除了公钥。为了避免琐碎的拒绝服务攻击，
与已知通道无关的节点将被忽略。

1. 类型：257（`node_announcement`）
2. 数据：
   * [`64`：`signature`]
   * [`2`：`flen`]
   * [`flen`：`features`]
   * [`4`：`timestamp`]
   * [`33`：`node_id`]
   * [`3`：`rgb_color`]
   * [`32`：`alias`]
   * [`2`：`addrlen`]
   * [`addrlen`：`addresses`]

`timestamp`允许在多个情况下对消息进行排序
公告。 `rgb_color`和`alias`允许分配情报服务
像“IRATEMONK”和“WISTFULTOLL”这样的节点颜色像黑色和酷炫的明星。

`addresses`允许节点宣布其接受传入网络的意愿
连接：它包含一系列用于连接的地址描述符
节点。第一个字节描述了地址类型，后面是
该类型的适当字节数。

定义了以下`地址描述符`类型：

   * `1`：ipv4; data =`[4：ipv4_addr] [2：port]`（长度6）
   * `2`：ipv6; data =`[16：ipv6_addr] [2：port]`（长度18）
   * `3`：Tor v2洋葱服务; data =`[10：onion_addr] [2：port]`（长度12）
       * 版本2洋葱服务地址;编码一个80位，截断的`SHA-1`
       洋葱服务的1024位`RSA`公钥的哈希值（a.k.a. Tor
       隐藏的服务）。
   * `4`：Tor v3洋葱服务; data =`[35：onion_addr] [2：port]`（长度37）
       * 版本3（[prop224]（https://gitweb.torproject.org/torspec.git/tree/proposals/224-rend-spec-ng.txt））
         洋葱服务地址;编码：
         `[32：32_byte_ed25519_pubkey] || [2：校验和] || [1：版本]`，在哪里
         `checksum = sha3（“。洋葱校验和”| pubkey ||版本）[：2]`。

### 要求

原点节点：
  - 必须将`timestamp`设置为大于之前的任何时间
  它以前创建的`node_announcement`。
    - 可以将它基于UNIX时间戳。
  - 必须将`signature`设置为整个双SHA256的签名
  `signature`后的剩余数据包（使用`node_id`给出的密钥）。
  - 可以设置`alias`和`rgb_color`来自定义它在地图中的外观
  图表。
    - 注意：`rgb_color`的第一个字节是红色值，第二个字节是
    绿色值，最后一个字节是蓝色值。
  - 必须将`alias`设置为有效的UTF-8字符串，并带有任何`alias` trailing-bytes
  等于0。
  - 应该用每个公共网络的地址描述符填充`address`
  期望传入连接的地址。
  - 必须将`addrlen`设置为`address`中的字节数。
  - 必须按升序放置地址描述符。
  - 不应该在任何地方放置任何零类型的地址描述符。
  - 应该只使用放置来对齐“地址”后面的字段。
  - 绝不能创建一个`type 1`或`type 2`地址描述符，`port`相等
  到0。
  - 应该确保`ipv4_addr`和`ipv6_addr`是可路由的地址。
  - 绝不能包含多个相同类型的“地址描述符”。
  - 应该将`flen`设置为保持`features`所需的最小长度
  它设置的位。

接收节点：
  - 如果`node_id`不是有效的压缩公钥：
    - 应该连接失败。
    - 绝不能进一步处理消息。
  - 如果`signature`不是有效的签名（使用`的node_id`）
  在`signature`字段后面的整个消息的双SHA256，包括
任何未来的字段附加到结尾）：
    - 应该连接失败。
    - 绝不能进一步处理消息。
  - 如果`features`字段包含_unknown even bits_：
    - 绝不能解析消息的其余部分。
    - 可以完全丢弃该消息。
    - 不应该连接到节点。
  - 可以转发包含_unknown_` features` _bit_的`node_announcement`，
  无论是否解析了公告。
  - 应该忽略与类型不匹配的第一个“地址描述符”
  定义如上。
  - 如果`addrlen`不足以保存地址描述符
  已知类型：
    - 应该连接失败。
  - 如果`port`等于0：
    - 应该忽略`ipv6_addr`或`ipv4_addr`。
  - 如果之前从`channel_announcement`消息中找不到`node_id`，
  或者，如果`timestamp`不大于最后收到的`node_announcement`
  从这个`node_id`：
    - 应该忽略这条消息。
  - 除此以外：
    - 如果`timestamp`大于最后收到的`node_announcement`
    这个`node_id`：
      - 应该将消息排队以进行重播。
      - 可以选择NOT将消息排队的时间长于最小预期长度。
  - 可以使用`rgb_color`和`alias`来引用接口中的节点。
    - 应该暗示他们的自签名起源。

### 合理

未来可能有新的节点功能：向后兼容（或
可选的）会有_odd_`feature`_bits_，不兼容的会有
_even_`feature`_bits_。这些可以由节点传播，即使它们也是如此
不能自己处理公告。

将来可能会添加新的地址类型;作为地址描述符
按升序排序，可以安全地忽略未知的。
“地址”之外的其他字段也可能在将来添加
在`地址`中可选填充，如果它们需要某些对齐。

### 节点别名的安全注意事项

节点别名是用户定义的，并提供了一个潜在的注入途径
在渲染过程中和持久性过程中都会发生攻击。

节点别名应始终在显示之前进行清理
HTML / Javascript上下文或任何其他动态解释的呈现
构架。同样，考虑使用预准备语句，输入验证，
逃避以防止注入漏洞和持久性
支持SQL或其他动态解释的查询语言的引擎。

* [存储和反映的XSS预防](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)
* [基于DOM的XSS预防](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL注入预防](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

不要像[小鲍比表]的学校（https://xkcd.com/327/）。

## `channel_update`消息

最初宣布一个频道后，各方独立
宣布中继HTLC所需的费用和最低到期增量
通过这个渠道。每个都使用与之匹配的8字节通道shortid
`channel_announcement`和1位`channel_flags`字段表示该结尾
通道它（原点或最终）。节点可以多次执行此操作
为了改变费用。

请注意，`channel_update`八卦消息仅在上下文中有用
*转发*付款，而非*发送*付款。付款时
 `A`  - >`B`  - >`C`  - >`D`，只有与通道有关的`channel_update`
 `B`  - >`C`（由'B`宣布）和`C`  - >`D`（由`C`宣布）将
 参加进来。在构建路线时，HTLC的数量和到期需要
 从目的地向后计算。确切的初始
 `amount_msat`的值和`cltv_expiry`的最小值，用于
 支付请求中提供了路线中的最后一个HTLC
 （参见[BOLT＃11]（11-payment-encoding.md #tagged-fields））。

1. 类型：258（`channel_update`）
2. 数据：
    * [`64`：`signature`]
    * [`32`：`chain_hash`]
    * [`8`：`short_channel_id`]
    * [`4`：`timestamp`]
    * [`1`：`message_flags`]
    * [`1`：`channel_flags`]
    * [`2`：`cltv_expiry_delta`]
    * [`8`：`htlc_minimum_msat`]
    * [`4`：`fee_base_msat`]
    * [`4`：`fee_proportional_millionths`]
    * [`8`：`htlc_maximum_msat`]（option_channel_htlc_max）

`channel_flags`位域用于指示通道的方向：它
标识此更新源自的节点并发出各种选项的信号
关于频道。下表指定了其含义
个别位：

| 位位置  | 名称        | 含义                          |
| ------------- | ----------- | -------------------------------- |
| 0             | `direction` | 此更新所指的方向。 |
| 1             | `disable`   | 禁用频道。             |

`message_flags`位域用于指示是否存在可选项
`channel_update`消息中的字段：

| 位位置  | 名称                      | 领域                            |
| ------------- | ------------------------- | -------------------------------- |
| 0             | `option_channel_htlc_max` | `htlc_maximum_msat`              |

请注意，`htlc_maximum_msat`字段在当前是静态的
在频道的生命周期内的协议：它不是*设计的
表示每个方向的实时信道容量
将是一个巨大的数据泄漏和无用的垃圾邮件网络（它
绯闻传播每一跳平均需要30秒。

用于签名验证的`node_id`取自相应的
`channel_announcement`：`node_id_1`，如果标志的最低有效位为0
或者`node_id_2`。

### 要求

原点节点：
  - 可以创建一个`channel_update`来将通道参数传递给
  频道同伴，即使该频道尚未公布（即
  `announce_channel`位未设置）。
    - 为了保护隐私，不得将这样的`channel_update`转发给其他同行
    原因。
    - 注意：这样的`channel_update`，一个不以a开头
    `channel_announcement`，对任何其他对等方都无效，将被丢弃。
  - 必须将`signature`设置为整个双SHA256的签名
  `signature`之后的剩余数据包，使用自己的`node_id`。
  - 必须设置`chain_hash`和`short_channel_id`以匹配32字节的哈希AND
  8字节通道ID，唯一标识在指定的通道
  `channel_announcement`消息。
  - 如果消息中的origin节点是`node_id_1`：
    - 必须将`channel_flags`的`direction`位设置为0。
  - 除此以外：
    - 必须将`channel_flags`的`direction`位设置为1。
  - 如果存在`htlc_maximum_msat`字段：
    - 必须将`message_flags`的`option_channel_htlc_max`位设置为1。
    - 必须将`htlc_maximum_msat`设置为它将通过该通道发送给单个HTLC的最大值。
        - 必须将其设置为小于或等于通道容量。
        - 必须将其设置为小于或等于`max_htlc_value_in_flight_msat`
          它是从同行那里收到的。
        - 对于带有'chain_hash`标识比特币区块链的渠道：
          - 必须将其设置为小于2 ^ 32。
  - 除此以外：
    - 必须将`message_flags`的`option_channel_htlc_max`位设置为0。
  - 必须在`channel_flags`和`message_flags`中设置未赋值为0的位。
  - 可以创建并发送`channel_update`，并将`disable`位设置为1
  信号通道暂时不可用（例如由于丢失
  连接）或永久不可用（例如在链上之前）
  沉降）。
    - 可以发送一个后续的`channel_update`，并将`disable`位设置为0
    重新启用频道。
  - 必须将`timestamp`设置为大于0，AND设置为大于任何
  先前为此`short_channel_id`发送了`channel_update`。
    - 应该在UNIX时间戳上建立`timestamp`。
  - 必须将`cltv_expiry_delta`设置为它将减去的块数
  传入的HTLC的`cltv_expiry`。
  - 必须将`htlc_minimum_msat`设置为最小HTLC值（毫秒）
  通道同伴将接受。
  - 必须将`fee_base_msat`设置为它将收取的基本费用（以毫士通为单位）
  对于任何HTLC。
  - 必须将'fee_proportional_millionths`设置为金额（百万分之一）
  satoshi）它将按照转移的satoshi收费。
  - 不应该创建冗余的`channel_update`s

接收节点：
  - 如果`short_channel_id`与之前的`channel_announcement`不匹配，
  或者如果在此期间关闭了频道：
    - 必须忽略`channel_update'，它不对应自己的一个
    通道。
  - 应该接受`channel_update为自己的频道（即使是非公开频道），
  为了学习相关的原始节点的转发参数。
  - 如果`signature`不是有效签名，则使用`node_id`
  在`signature`字段后面的整个消息的双SHA256（包括
  `fee_proportional_millionths`后面的未知字段：
    - 绝不能进一步处理消息。
    - 应该连接失败。
  - 如果指定的`chain_hash`值未知（意味着它未激活）
  指定的链）：
    - 必须忽略频道更新。
  - 如果`timestamp`不大于最后收到的那个
  这个`short_channel_id`和`node_id`的`channel_update`：
    - 应该忽略这条消息。
  - 除此以外：
    - 如果`timestamp`等于最后收到的`channel_update`
    AND字段（除了`signature`）不同：
      - 可以将这个`node_id`列入黑名单。
      - 可能会忘记与之相关的所有频道。
  - 如果`timestamp`在将来不合理的话：
    - 可以丢弃`channel_update`。
  - 除此以外：
    - 应该将消息排队以进行重播。
    - 可以选择NOT来处理超过最小预期长度的消息。
  - 如果`message_flags`的`option_channel_htlc_max`位为0：
    - 必须考虑`htlc_maximum_msat`不存在。
  - 除此以外：
    - 如果`htlc_maximum_msat`不存在或大于信道容量：
      - 可以将这个`node_id`列入黑名单
      - 在路由考虑期间应该忽略此通道。
    - 除此以外：
      - 路由时应该考虑`htlc_maximum_msat`。

### 合理

节点使用`timestamp`字段来修剪`channel_update'
要么在将来太远，要么在两周内没有更新;所以
将它作为UNIX时间戳（即自UTC以来的秒数）是有意义的
1970-01-01）。然而，考虑到可能的情况，这不是一个严格的要求
一秒钟内两个`channel_update`

显式的“option_channel_htlc_max”标志表示存在
`htlc_maximum_msat`（而不是暗示`htlc_maximum_msat`
通过消息长度）允许我们扩展`channel_update`
将来会有不同的领域。由于频道限制为2 ^ 32-1
比特币中的millisatoshis，`htlc_maximum_msat`具有相同的限制。

针对冗余`channel_update'的建议最大限度地减少了对网络的垃圾邮件，
但它有时是不可避免的。例如，一个带有的通道
无法访问的peer最终会导致`channel_update`
表示通道已禁用，另一个更新重新启用
对等体重新建立联系时的通道。因为八卦
消息被批处理并替换以前的消息，结果可能是a
单个看似冗余的更新。

## 查询消息

通过`init`协商`gossip_queries`选项可以启用一个数字
对八卦同步的扩展查询这些明确
请求收到什么八卦。

有几条消息包含很长的数组
`short_channel_id`s（称为`encoded_short_ids`）所以我们使用了
简单的压缩方案：第一个字节表示编码，
rest包含数据。

编码类型：
* `0`：`short_channel_id`类型的未压缩数组，按升序排列。
* `1`：`short_channel_id`类型的数组，按升序排列，用zlib压缩压缩<sup> [1]（＃reference-1）</sup>

请注意，65535字节的zlib消息可以解压缩到67632120中
bytes <sup> [2]（＃reference-2）</sup>，但由于唯一有效的内容
是唯一的8字节值，不超过14个字节可以复制
跨流：因为每个副本至少占用2位，所以无效
内容可以解压缩到3669960字节以上。

### `query_short_channel_ids` /`reply_short_channel_ids_end`消息

1. type：261（`query_short_channel_ids`）（`gossip_queries`）
2. 数据：
    * [`32`：`chain_hash`]
    * [`2`：`len`]
    * [`len`：`encoded_short_ids`]

1. 类型：262（`reply_short_channel_ids_end`）（`gossip_queries`）
2. 数据：
    * [`32`：`chain_hash`]
    * [`1`：`complete`]

这是一个允许节点查询的通用机制
特定频道的`channel_announcement`和`channel_update`消息
（通过`short_channel_id`s确定）。这通常是因为使用
一个节点看到一个`channel_update`，它没有`channel_announcement`或
因为它已经获得了以前未知的`short_channel_id`s
来自`reply_channel_range`。

#### 要求

寄件人：
  - 如果它已向此对等方发送了先前的`query_short_channel_ids`并且未收到`reply_short_channel_ids_end`，则不得发送`query_short_channel_ids`。
  - 必须将`chain_hash`设置为唯一标识链的32字节哈希
  `short_channel_id`s指的是。
  - 必须将`encoded_short_ids`的第一个字节设置为编码类型。
  - 必须将整数`short_channel_id`s编码为`encoded_short_ids`
  - 如果它收到一个'channel_update`，可以发送它
   `short_channel_id`，它没有`channel_announcement`。
  - 如果引用的通道不是未使用的输出，则不应发送此信息。

收件人：
  - 如果`encoded_short_ids`的第一个字节不是已知的编码类型：
    - 可能连接失败
  - 如果`encoded_short_ids`没有解码为整数`short_channel_id`：
    - 可能连接失败。
  - 如果它还没有将`reply_short_channel_ids_end`发送到此发件人先前收到的`query_short_channel_ids`：
    - 可能连接失败。
  - 必须使用`channel_announcement`响应每个已知的`short_channel_id`
    以及每一端的最新`channel_update`
    - 不应该等待下一个传出的八卦冲洗发送这些。
  - 必须遵循每个`channel_announcement`的`node_announcement`
    - 应该避免发送重复的`node_announcements`以响应单个`query_short_channel_ids`。
  - 必须使用`reply_short_channel_ids_end`跟随这些回复。
  - 如果不保持`chain_hash`的最新频道信息：
    - 必须将`complete`设置为0。
  - 除此以外：
    - 应该将`complete`设置为1。

#### 合理

未来的节点可能没有完整的信息;他们当然不会
关于未知`chain_hash`链的完整信息。虽然这个'完整'
字段不可信，0表示发件人应该搜索
其他地方的其他数据。

显式的`reply_short_channel_ids_end`消息意味着接收者可以
表明它什么都不知道，发件人不需要依赖
超时。它还会导致查询的自然速率限制。

### `query_channel_range`和`reply_channel_range`消息

1. type：263（`query_channel_range`）（`gossip_queries`）
2. 数据：
    * [`32`：`chain_hash`]
    * [`4`：`first_blocknum`]
    * [`4`：`number_of_blocks`]

1. 类型：264（`reply_channel_range`）（`gossip_queries`）
2. 数据：
    * [`32`：`chain_hash`]
    * [`4`：`first_blocknum`]
    * [`4`：`number_of_blocks`]
    * [`1`：`complete`]
    * [`2`：`len`]
    * [`len`：`encoded_short_ids`]

这允许查询特定块内的通道。

#### 要求

`query_channel_range`的发件人：
  - 如果它已向此对等方发送了先前的`query_channel_range`并且未收到所有'reply_channel_range`回复，则不得发送此消息。
  - 必须将`chain_hash`设置为唯一标识链的32字节哈希
  它希望`reply_channel_range`引用
  - 必须将`first_blocknum`设置为它想知道通道的第一个块
  - 必须将`number_of_blocks`设置为1或更大。

`query_channel_range`的接收者：
  - 如果它还没有将所有`reply_channel_range`发送到此发件人之前收到的`query_channel_range`：
    - 可能连接失败。
  - 必须回复一个或多个组合范围的`reply_channel_range`
    将请求的`first_blocknum`覆盖到`first_blocknum`加上
    `number_of_blocks`减1。
  - 对于每个`reply_channel_range`：
    - 必须设置`chain_hash`等于`query_channel_range`的，
    - 它必须为它知道的每个开放通道编码一个`short_channel_id`，它在块`first_blocknum`到`first_blocknum`加上`number_of_blocks`减一。
    - 必须将`number_of_blocks`限制为最大块数
      结果可能适合`encoded_short_ids`
    - 如果不保持`chain_hash`的最新频道信息：
      - 必须将`complete`设置为0。
    - 除此以外：
      - 应该将`complete`设置为1。

#### 合理

对于单个数据包，单个响应可能太大，而对等体也可能
存储（例如）1000块范围的罐头结果，并简单地提供每个回复
它与请求的范围重叠。

### `gossip_timestamp_filter`消息

1. 类型：265（`gossip_timestamp_filter`）（`gossip_queries`）
2. 数据：
    * [`32`：`chain_hash`]
    * [`4`：`first_timestamp`]
    * [`4`：`timestamp_range`]

此消息允许节点将未来的八卦消息约束到
特定范围。想要任何八卦消息的节点都有
发送这个，否则`gossip_queries`谈判意味着没有八卦
将收到消息。

请注意，此过滤器会替换之前的过滤器，因此可以使用它
多次改变同伴的八卦。

#### 要求

寄件人：
  - 必须将`chain_hash`设置为唯一标识链的32字节哈希
  它想要八卦参考。

收件人：
  - 应该发送所有'timestamp`更大或更高的八卦消息
    等于`first_timestamp`，小于`first_timestamp`加
    `timestamp_range`。
    - 可以等待下一个传出的八卦冲洗发送这些。
  - 应该将未来的八卦消息限制在那些“时间戳”的人身上
    大于或等于`first_timestamp`，小于
    `first_timestamp`加上`timestamp_range`。
  - 如果`channel_announcement`没有相应的`channel_update`s：
    - 绝不能发送`channel_announcement`。
  - 除此以外：
      - 必须将`channel_announcement`的`timestamp`视为相应`channel_update`的`timestamp`。
      - 必须考虑在收到第一个相应的`channel_update`后是否发送`channel_announcement`。
  - 如果发送`channel_announcement`：
      - 必须在任何相应的`channel_update`s和`node_announcement`s之前发送`channel_announcement`。

#### 合理

由于`channel_announcement`没有时间戳，我们生成一个可能的时间戳
一。如果没有`channel_update`那么它根本就不会发送，这是最多的
可能是修剪过的频道。

否则，`channel_announcement`通常紧跟着a
`channel_update`。理想情况下，我们会指定第一个（最旧的）`channel_update`
timestamp将被用作`channel_announcement`的时间，但新节点将被用作
网络不会有这个，并且还需要第一个`channel_update`
要存储的时间戳。相反，我们允许使用任何更新
很容易实现。

如果遗漏了`channel_announcement`，
`query_short_channel_ids`可用于检索它。

## 初始同步

如果节点需要初始同步的八卦消息，它将被标记
在`init`消息中，通过功能标志（[BOLT＃9]（09-features.md＃assigned-localfeatures-flags））。

请注意，`initial_routing_sync`功能被覆盖（并且应该被覆盖）
被认为是等于0）的'gossip_queries`功能
后者通过`init`协商。

请注意，`gossip_queries`不适用于较旧的节点，因此
`initial_routing_sync`的值对于控制仍然很重要
与他们的互动。

### 要求

一个节点：
  - 如果'gossip_queries`功能已经协商：
    - 除非明确要求，否则不得转发任何八卦消息。
  - 除此以外：
    - 如果需要对等体路由状态的完整副本：
      - 应该将`initial_routing_sync`标志设置为1。
    - 收到`init`消息并将`initial_routing_sync`标志设置为
    1：
      - 应该为所有已知的频道和节点发送八卦消息，就好像它们只是一样
      接收。
    - 如果`initial_routing_sync`标志设置为0，或者如果初始同步是
    完成：
      - 应该恢复正常操作，如下所示
      [转播]（#revroadcasting）部分。

## 转播

### 要求

接收节点：
  - 收到一个新的`channel_announcement`或`channel_update`或
  带有更新的`timestamp`的`node_announcement`：
    - 应该相应地更新其网络拓扑的本地视图。
  - 应用公告中的更改后：
    - 如果没有与相应源节点关联的通道：
      - MAY可以从其已知节点集清除原始节点。
    - 除此以外：
      - 应该更新适当的元数据并存储签名
      与公告有关。
        - 注意：稍后将允许节点重建通知
        对于同行。

一个节点：
  - 如果'gossip_queries`功能已经协商：
    - 在收到`gossip_timestamp_range`之前一定不要发八卦。
  - 应该每隔60秒刷一次传出的八卦消息，独立于
  消息的到达时间。
    - 注意：这导致交错的公告是独特的（不是
    复制）。
  - 可以定期重新宣布其频道。
    - 注意：不鼓励这样做，以保持较低的资源需求。
  - 连接建立时：
    - 应该发送所有`channel_announcement`消息，然后发送最新消息
    `node_announcement`和`channel_update`消息。

### 合理

一旦处理了八卦消息，就会将其添加到外发列表中
消息，发往处理节点的对等体，替换任何旧的消息
来自原始节点的更新。这个八卦消息列表将被刷新
定期;这种存储和延迟转发广播称为a
_staggered broadcast_。而且，这种批处理形成了自然的速度
低开销限制。

在重新连接时发送所有八卦是天真的，但很简单，
并允许自动引导新节点以及更新节点
已经离线一段时间了。 `gossip_queries`选项
允许更精细的同步。

## HTLC费用

### 要求

原点节点：
  - 应该接受付费等于或大于以下费用的HTLC：
    - fee_base_msat +（amount_to_forward * fee_proportional_millionths / 1000000）
  - 在接下来的一段合理时间内，应该接受支付较旧费用的HTLC
  发送`channel_update`。
    - 注意：这允许任何传播延迟。

## 修剪网络视图

### 要求

一个节点：
  - 应该监控区块链中的资金交易，以便识别
  正在关闭的渠道。
  - 如果正在花费一个频道的资金输出：
    - 应该从本地网络视图中删除并被视为已关闭。
  - 如果宣布的节点不再有任何关联的开放频道：
    - 可以修剪通过`node_announcement`消息添加的节点
    当地的看法。
      - 注意：这是`node_announcement`的依赖性的直接结果
      前面有一个`channel_announcement`。

### 关于修剪陈旧条目的建议

#### 要求

一个节点：
  - 如果一个频道的最新`channel_update``时间戳'超过两周
  （1209600秒）：
    - 可以修剪频道。
    - 可以忽略该频道。
    - 注意：这是一个单独的节点策略，绝不能强制执行
    转发对等体，例如收到过时的八卦时关闭频道
    消息。

#### 合理

几种情况可能导致通道变得无法使用及其端点
无法发送这些频道的更新。例如，如果发生这种情况
两个端点都无法访问其私钥，也无法签名
`channel_update`s也不关闭通道链。在这种情况下，渠道是
不太可能是计算路线的一部分，因为它们将被分割
来自网络的其他部分;但是，他们会留在本地网络中
视图将无限期地转发给其他同行。

## 路由建议

在计算HTLC的路线时，既要有“cltv_expiry_delta”和费用
需要考虑：`cltv_expiry_delta`有助于时间
在最坏情况下失败的情况下，资金将无法使用。关系
这两个属性之间的区别还不清楚，因为它取决于它的可靠性
涉及的节点。

如果通过简单地路由到预期接收者并求和来计算路由
`cltv_expiry_delta`s，然后中间节点可能猜测
他们在路线上的位置。知道HTLC的CLTV，周围
网络拓扑结构，`cltv_expiry_delta'为攻击者提供了一种猜测方式
预期的收件人。因此，非常希望添加随机偏移
到目标收件人将收到的CLTV，这会破坏所有CLTV
沿途。

为了创建合理的偏移，原始节点可以启动有限的
随机走在图表上，从预期的收件人开始并总结
`cltv_expiry_delta`s，并使用得到的和作为偏移量。
这有效地为实际路线创建了_shadow路线扩展_
提供更好的防御这种攻击向量的保护，而不仅仅是选择
随机偏移会。

其他更高级的考虑涉及路线选择的多样化，
避免单点故障和检测，以及本地的平衡
通道。

### 路由示例

考虑四个节点：


```
   乙
  / \
 / \
一个C.
 \ / /
  \ / /
   d
```

每个都在每个结尾处公布以下`cltv_expiry_delta`
渠道：

1. 答：10个街区
2. B：20个街区
3. C：30个街区
4. D：40个街区

在请求时，C还使用9的“min_final_cltv_expiry”（默认值）
付款。

此外，每个节点都有一个集合费用方案，它用于每个节点
渠道：

1. 答：100基数+ 1000万分之一
2. B：200基数+ 2000百万分之一
3. C：300基数+3000亿分之一
4. D：400基数+ 4000百万分之一

网络将看到八个`channel_update`消息：

1. A-> B：`cltv_expiry_delta` = 10，`fee_base_msat` = 100，`fee_proportional_millionths` = 1000
1. A-> D：`cltv_expiry_delta` = 10，`fee_base_msat` = 100，`fee_proportional_millionths` = 1000
1. B-> A：`cltv_expiry_delta` = 20，`fee_base_msat` = 200，`fee_proportional_millionths` = 2000
1. D-> A：`cltv_expiry_delta` = 40，`fee_base_msat` = 400，`fee_proportional_millionths` = 4000
1. B-> C：`cltv_expiry_delta` = 20，`fee_base_msat` = 200，`fee_proportional_millionths` = 2000
1. D-> C：`cltv_expiry_delta` = 40，`fee_base_msat` = 400，`fee_proportional_millionths` = 4000
1. C-> B：`cltv_expiry_delta` = 30，`fee_base_msat` = 300，`fee_proportional_millionths` = 3000
1. C-> D：`cltv_expiry_delta` = 30，`fee_base_msat` = 300，`fee_proportional_millionths` = 3000

** B-> C. **如果B要直接向C发送4,999,999 millisatoshi，它会
既不收费也不加自己的`cltv_expiry_delta`，所以它会
使用C的请求`min_final_cltv_expiry`为9.据推测它也会添加一个
_shadow route_提供额外的CLTV 42.另外，它可以增加额外的
其他跃点的CLTV增量，因为这些值代表最小值，但选择不是
这样做是为了简单起见：

   * `amount_msat`：4999999
   * `cltv_expiry`：current-block-height + 9 + 42
   * `onion_routing_packet`：
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

** A-> B-> C. **如果A要通过B向C发送4,999,999毫希，它需要
支付B在B-> C`channel_update`中指定的费用，计算方式为
按[HTLC费用]（＃htlc-fees）：

        fee_base_msat +（amount_to_forward * fee_proportional_millionths / 1000000）

    200 +（4999999 * 2000/1000000）= 10199

同样，它需要添加B-> C的`channel_update``cltv_expiry`（20），C的
请求`min_final_cltv_expiry`（9），以及_shadow route_（42）的费用。
因此，A-> B的`update_add_htlc`消息将是：

   * `amount_msat`：5010198
   * `cltv_expiry`：current-block-height + 20 + 9 + 42
   * `onion_routing_packet`：
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

B-> C的`update_add_htlc`与B-> C的直接付款相同。

** A-> D-> C. **最后，如果由于某种原因A通过D选择了更昂贵的路线，
A-> D的`update_add_htlc`消息将是：

   * `amount_msat`：5020398
   * `cltv_expiry`：current-block-height + 40 + 9 + 42
   * `onion_routing_packet`：
     * `amt_to_forward` = 4999999
     * `outgoing_cltv_value` = current-block-height + 9 + 42

并且D-> C的`update_add_htlc`将再次与B-> C的直接付款相同
以上。

## 参考

1.  <a id="reference-1"> [RFC 1950“ZLIB压缩数据格式规范版本3.3]（https://www.ietf.org/rfc/rfc1950.txt)</a>
2.  <a id="reference-2"> [最大压缩因子]（https://zlib.net/zlib_tech.html)</a>

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
