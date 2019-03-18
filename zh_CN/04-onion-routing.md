# BOLT＃4：洋葱路由协议

## 概观

本文档描述了洋葱布线包的构造
用于将付款从_origin node_路由到_final node_。包裹
通过许多称为_hops_的中间节点进行路由。

路由模式基于
[斯芬克斯]（http://www.cypherpunks.ca/~iang/pubs/Sphinx_Oakland09.pdf）
构造并通过每跳有效载荷进行扩展。

转发消息的中间节点可以验证其完整性
数据包，可以了解他们应该转发哪个节点
包到。除了他们之外，他们无法了解哪些其他节点
前任或后继者，是数据包路由的一部分;他们也不能学习
路线的长度或其在其中的位置。包是
在每一跳都进行模糊处理，以确保网络级攻击者不能
关联属于同一路由的数据包（即所属的数据包）
到同一路线不共享任何相关信息）。请注意这一点
并不排除攻击者进行数据包关联的可能性
通过流量分析。

该路由由知道公众的起始节点构成
每个中间节点和最终节点的密钥。知道每个节点的公钥
允许源节点为每个创建共享密钥（使用ECDH）
中间节点和最终节点。然后是共享秘密
用于生成_pseudo-random stream_ of bytes（用于混淆）
数据包）和一些_keys_（用于加密有效载荷和
计算HMAC）。然后HMAC用于确保完整性
每一跳的数据包。

沿着路线的每一跳只能看到原始节点的临时密钥
为了隐藏发件人的身份。短暂的关键是每个人都是盲目的
转发到下一个中间跳，使洋葱不可链接
沿途。

该规范描述了包格式和路由的_version 0_
机制。

一个节点：
  - 在收到比它实现的更高版本的数据包时：
    - 必须向原始节点报告路由故障。
    - 务必丢弃数据包。

# 目录

  * [约定](#conventions)
  * [密钥生成](#key-generation)
  * [伪随机字节流](#pseudo-random-byte-stream)
  * [数据包结构](#packet-structure)
    * [最后一个节点的有效负载](#payload-for-the-last-node)
  * [共享秘密](#shared-secret)
  * [致盲的短暂关键](#blinding-ephemeral-keys)
  * [数据包构建](#packet-construction)
  * [数据包转发](#packet-forwarding)
  * [填充物生成](#filler-generation)
  * [返回错误](#returning-errors)
    * [失败消息](#failure-messages)
    * [接收失败代码](#receiving-failure-codes)
  * [测试矢量](#test-vector)
    * [数据包创建](#packet-creation)
      * [参数](#parameters)
      * [每跳信息](#per-hop-information)
      * [每包信息](#per-packet-information)
      * [包装洋葱](#wrapping-the-onion)
      * [最终数据包](#final-packet)
    * [返回错误](#returning-errors)
  * [参考](#references)
  * [作者](#authors)

# 约定

本文件中遵循了许多公约：

 - 长度：最大路由长度限制为20跳。
 - HMAC：数据包的完整性验证基于Keyed-Hash
 消息认证码，由。定义
 [FIPS 198标准]（http://csrc.nist.gov/publications/fips/fips198-1/FIPS-198-1_final.pdf）/ [RFC 2104]（https://tools.ietf.org/html/ RFC2104）
 并使用`SHA256`散列算法。
 - 椭圆曲线：对于涉及椭圆曲线的所有计算，
   使用比特币曲线，如[`secp256k1`]（http://www.secg.org/sec2-v2.pdf）中所述。
 - 伪随机流：[`ChaCha20`]（https://tools.ietf.org/html/rfc7539）是
 用于生成伪随机字节流。对于它的一代，一个固定的
 使用null-nonce（`0x0000000000000000`），以及从a派生的密钥
 共享密钥和所需输出大小的`0x00`字节流作为
 信息。

 - 术语_origin node_和_final node_指的是初始数据包发送方
 和最终的数据包接收者。
 - 术语_hop_和_node_有时可互换使用，但_hop_
 通常是指路由中的中间节点而不是端节点。
        _origin node_  - > _hop_  - > ...  - > _hop_  - > _final node_
 - 术语_processing node_指的是沿着路径的特定节点
 当前处理转发的数据包。
 - 术语_peers_仅指作为直接邻居的跳跃（在
 覆盖网络）：更具体地说，_sending peers_转发数据包到
 _接收同伴_。
        _sending peer_  - > _ceceiving peer_


# 澄清

支持的最长路由有20个跃点，不计算_origin node_和_final node_，因此19个_intermediate nodes_和最多20个要遍历的通道。

# 密钥生成

许多加密和验证密钥都是从共享密钥派生的：

 - _rho_：在生成使用的伪随机字节流时用作密钥
 混淆每跳信息
 - _mu_：在HMAC生成期间使用
 - _um_：在错误报告期间使用

密钥生成函数采用密钥类型（_rho_ =`0x72686F`，_mu_ =`0x6d75`，
或_um_ =``0x756d`）和一个32字节的密钥作为输入并返回一个32字节的密钥。

通过计算HMAC生成密钥（使用`SHA256`作为散列算法）
使用适当的密钥类型（即_rho_，_mu_或_um_）作为HMAC密钥和
32字节共享密钥作为消息。然后返回得到的HMAC作为
键。

请注意，键类型不包括C风格的“0x00”终止字节，
例如_rho_ key-type的长度是3个字节，而不是4个字节。

# 伪随机字节流

伪随机字节流用于在数据包的每一跳处对数据包进行模糊处理
路径，使每一跳只能恢复下一跳的地址和HMAC。
通过加密（使用`ChaCha20`）a生成伪随机字节流
“0x00”字节流，具有所需长度，使用密钥初始化
派生自共享密钥和零nonce（`0x00000000000000`）。

使用固定的随机数是安全的，因为密钥永远不会被重用。

# 数据包结构

该数据包由四部分组成：

 - 一个`版本`字节
 - 在共享密钥期间使用的33字节压缩`secp256k1``public_key`
 代
 - 一个1300字节的“hops_data”，由20个固定大小的数据包组成，每个包含一个
 在消息转发期间由其关联的跃点使用的信息
 - 一个32字节的“HMAC”，用于验证数据包的完整性

数据包的网络格式由各个部分组成
序列化为一个连续的字节流，然后传输到数据包
接受者。由于数据包的固定大小，它不需要以它为前缀
通过连接传输时的长度。

数据包的整体结构如下：

1. 输入：`onion_packet`
2. 数据：
   * [`1`：`version`]
   * [`33`：`public_key`]
   * [`20 * 65`：`hops_data`]
   * [`32`：`hmac`]

对于此规范（_version 0_），`version`的常量值为“0x00”。

`hops_data`字段是保存下一跳的混淆的结构
地址，转移信息及其相关的HMAC。长度为1300字节（`20x65`）
并具有以下结构：

1. 输入：`hops_data`
2. 数据：
   * [`1`：`realm`]
   * [`32`：`per_hop`]
   * [`32`：`HMAC`]
   * ...
   * `filler`

其中，`realm`，`per_hop`（内容依赖于`realm`）和`HMAC`
每一跳重复;在哪里，`填充'由混淆，
确定性生成的填充，详见
[填充物生成]（＃filler-generation）。另外，`hops_data`是
在每一跳逐渐混淆。

`realm`字节确定`per_hop`字段的格式;目前，只有`境界'
定义了0，其中“per_hop”格式如下：

1. 类型：`per_hop`（对于`realm` 0）
2. 数据：
   * [`8`：`short_channel_id`]
   * [`8`：`amt_to_forward`]
   * [`4`：`outgoing_cltv_value`]
   * [`12`：`padding`]

使用`per_hop`字段，原始节点能够精确地指定路径和
每一跳转发的HTLC结构。因为`per_hop`受到保护
在数据包范围的HMAC下，它包含的信息是完全认证的
具有HTLC发送者（起始节点）和每个之间的每个成对关系
跳进小路。

使用此端到端身份验证，
每
跳是能够的
用'per_hop`的指定值交叉检查HTLC参数
并确保发送对等方没有转发
精心制作的HTLC。

领域描述：

   * `short_channel_id`：用于路由的传出信道的ID
      信息;接收对等体应该操作该信道的另一端。

   * `amt_to_forward`：以毫秒为单位的数量，转发到下一个
     接收路由信息中指定的对等体。

     该值必须包括原始节点的计算_fee_
     接收同伴。处理传入的Sphinx数据包和HTLC时
     如果以下不等式不成立，则封装在其中的消息，
     那么HTLC应该被拒绝，因为它表明前一跳有
     偏离指定的参数：

        incoming_htlc_amt  - 费用> = amt_to_forward

     其中“费用”是根据接收对方的广告费计算的
     模式（如[BOLT＃7]（07-routing-gossip.md＃htlc-fees）中所述）
     如果处理节点是最终节点，则为0或0。

   * `outgoing_cltv_value`：_outgoing_ HTLC携带的CLTV值
     包应该有。

        cltv_expiry  -  cltv_expiry_delta> = outgoing_cltv_value

     包含该字段允许跳转以验证信息
     由原始节点指定，并转发HTLC的参数，
     并确保origin节点使用当前的`cltv_expiry_delta`值。
     如果没有下一跳，则“cltv_expiry_delta”为0。
     如果值不对应，则HTLC应该失败并拒绝，如
     这表示转发节点已经篡改了预期的HTLC
     值或原始节点具有过时的`cltv_expiry_delta`值。
     跳跃必须在响应意外时保持一致
     `outgoing_cltv_value`，无论是否是最终节点，都要避免
     泄漏其在路线中的位置。

   * `padding`：此字段供将来使用，也用于确保未来非0-`realm`
     `per_hop`不会改变整个`hops_data`大小。

转发HTLC时，节点必须按照指定的方式构造输出HTLC
上面的'per_hop`;否则，偏离指定的HTLC参数
可能导致无关的路由故障。

## 非严格转发

节点可以沿着除一个之外的输出通道转发HTLC
只要接收器具有相同的节点，由`short_channel_id`指定
`short_channel_id`意图的公钥。因此，如果`short_channel_id`连接
节点A和B，HTLC可以通过连接A和B的任何信道转发。
如果不遵守将导致接收者无法解密下一个
跳进洋葱包里。

### 合理

如果两个对等体具有多个通道，则下游节点将是
无论数据包是哪个通道，都能够解密下一跳有效负载
发送过来。

实现非严格转发的节点能够进行实时评估
与特定对等体的信道带宽，并使用该信道
局部最优。

例如，如果由'short_channel_id`指定的通道连接A和B.
在转发时没有足够的带宽，那么A能够使用a
不同的渠道。这可以通过阻止来减少支付延迟
HTLC仅因“short_channel_id”带宽限制而失败
让发送者尝试相同的路由只在两者之间的通道不同
A和B.

非严格转发允许节点使用私有通道连接
它们到接收节点，即使公众不知道该频道
通道图。

### 建议

使用非严格转发的实现应考虑应用相同的实现
收费计划到同一个同行的所有渠道，因为发件人可能会选择
导致总体成本最低的渠道。有不同的政策
可能导致转发节点基于最优费用接受费用
发送方的计划，即使它们提供聚合带宽
跨越具有相同对等体的所有通道。

或者，实现可以选择仅应用非严格转发
喜欢政策渠道，以确保他们的预期费用收入不会偏离
使用备用频道。

## 最后一个节点的有效负载

构建路由时，原始节点必须使用有效负载
具有以下值的最终节点：
* `outgoing_cltv_value`：设置为收件人指定的最终到期时间
* `amt_to_forward`：设置为收件人指定的最终金额

这允许最终节点检查这些值并在需要时返回错误，
但它也消除了倒数第二次探测攻击的可能性
节点。否则，此类攻击可能会尝试发现接收对等方是否为
最后一个通过重新发送具有不同金额/到期的HTLC。
最终节点将从它收到的HTLC中提取洋葱有效载荷
将其值与HTLC的值进行比较。见
有关详细信息，请参阅下面的[返回错误]（#recovery-errors）部分。

如果不是上述情况，由于无需转发付款，最终节点可以
简单地丢弃其有效载荷。

# 共享秘密

原始节点使用沿着路径的每一跳建立共享秘密
椭圆曲线Diffie-Hellman在发送者的短暂密钥和那一跳之间
跳的节点ID密钥。生成的曲线点序列化为
使用`SHA256`进行DER压缩表示和散列。使用哈希输出
作为32字节的共享密钥。

椭圆曲线Diffie-Hellman（ECDH）是对EC私钥的操作
输出曲线点的EC公钥。对于这个协议，ECDH
使用`libsecp256k1`中实现的变体，它是在
`secp256k1`椭圆曲线。在数据包构建期间，发件人使用
短暂的私钥和跳的公钥作为ECDH的输入，而
在数据包转发期间，跃点使用短暂的公钥和它自己的公钥
节点ID私钥。由于ECDH的特性，它们都会衍生出来
相同的价值。

# 致盲的短暂关键

为了确保沿途的多跳不能被链接
他们看到的短暂的公钥，钥匙在每一跳都是盲目的。致盲是
以确定的方式完成，允许发送者计算
在数据包构造期间相应的盲目私钥。

EC公钥的盲法是单个标量乘法
表示具有32字节盲目因子的公钥的EC点。由于
标量乘法的交换性质，盲法私钥是
输入对应私钥的乘法乘积
相同的致盲因素。

致盲因子本身被计算为短暂公钥的函数
和32字节的共享密钥。具体来说，是`SHA256`哈希值
以压缩格式序列化的公钥的串联和
共同的秘密。

# 数据包构建

在以下示例中，假设_sending node_（原始节点），
`n_0`，想要将数据包路由到_receiving node_（最终节点），`n_r`。
首先，发送者计算路由`{n_0，n_1，...，n_ {r-1}，n_r}`，其中`n_0`
是发件人本身，`n_r`是最终收件人。所有节点都是'n_i`和
`n_ {i + 1}`必须是覆盖网络路由中的对等体。然后发件人收集了
`n_1`到`n_r`的公钥，并生成一个随机的32字节`sessionkey`。
可选地，发送方可以传入_associated data_，即数据
数据包提交但不包含在数据包本身中。关联的
数据将包含在HMAC中，并且必须与提供的相关数据相匹配
在每一跳的完整性验证期间。

为了构造洋葱，发件人初始化了临时私钥
第一跳'ek_1`到`sessionkey`并从中得到相应的
短暂的公钥`epk_1`乘以`secp256k1`基点。对于
沿着路线的每个“k”跳，发送者然后迭代地计算
共享秘密`ss_k`和下一跳'ek_ {k + 1}`的临时密钥如下：

 - 发件人使用hop的公钥和短暂的私有来执行ECDH
 获取曲线点的关键，使用`SHA256`进行散列来产生
 共享秘密`ss_k`。
 - 致盲因素是连接之间的连接的“SHA256”哈希
 短暂的公钥`epk_k`和共享的秘密`ss_k`。
 - 下一跳“ek_ {k + 1}”的临时私钥由计算
 将当前短暂的私钥“ek_k”乘以盲目因子。
 - 下一跳“epk_ {k + 1}”的短暂公钥来源于
 短暂的私钥“ek_ {k + 1}”乘以基点。

一旦发件人拥有上述所有必需信息，它就可以构建
包。构造在`r`跳上路由的数据包需要`r`32字节
短暂的公钥，`r` 32字节共享秘密，`r` 32字节的盲目因素，
和`r` 65字节`per_hop`有效载荷。构造返回一个1366字节
数据包以及第一个接收对等体的地址。

分组构造以与路由相反的顺序执行，即
最后一跳的操作首先应用。

数据包初始化为1366“0x00”字节。

生成一个65字节的填充程序（参见[填充程序生成]（＃filler-generation））
使用共享密钥。

对于路径中的每一跳，按相反的顺序，发送方应用
以下操作：

 - _rho_-key和_mu_-key是使用hop的共享密钥生成的。
 - `hops_data`字段右移65字节，丢弃最后65字节
 超过1300字节大小的字节。
 - `version`，`short_channel_id`，`amt_to_forward`，`outgoing_cltv_value`，
   `padding`和`HMAC`被复制到以下65个字节中。
 - _rho_-key用于生成1300字节的伪随机字节流
 然后使用`XOR`将其应用于`hops_data`字段。
 - 如果这是最后一跳，即第一次迭代，那么它的尾部
 `hops_data`字段被路由信息`filler`覆盖。
 - 计算下一个HMAC（使用_mu_-key作为HMAC-key）
 连接`hops_data`和相关数据。

得到的最终HMAC值是第一个将使用的HMAC
接收路由中的对等体。

数据包生成返回包含`version`的序列化数据包
byte，第一跳的短暂pubkey，第一跳的HMAC，以及
混淆的`hops_data`。

以下Go代码是数据包构造的示例实现：

```去
func NewOnionPacket（paymentPath [] * btcec.PublicKey，sessionKey * btcec.PrivateKey，
    hopsData [] HopData，assocData [] byte）（* OnionPacket，error）{

    numHops：= len（paymentPath）
    hopSharedSecrets：= make（[] [sha256.Size] byte，numHops）

    //初始化第一跳到会话密钥的临时密钥。
    var ephemeralKey big.Int
    ephemeralKey.Set（sessionKey.D）

    对于i：= 0;我<numHops;我++ {
        //执行ECDH并散列结果。
        ecdhResult：= scalarMult（paymentPath [i]，ephemeralKey）
        hopSharedSecrets [i] = sha256.Sum256（ecdhResult.SerializeCompressed（））

        //从私钥中导出短暂的公钥。
        ephemeralPrivKey：= btcec.PrivKeyFromBytes（btcec.S256（），ephemeralKey.Bytes（））
        ephemeralPubKey：= ephemeralPrivKey.PubKey（）

        //计算致盲因子。
        sha：= sha256.New（）
        sha.Write（ephemeralPubKey.SerializeCompressed（））
        sha.Write（hopSharedSecrets [I]）

        var blindingFactor big.Int
        blindingFactor.SetBytes（sha.Sum（无））

        //下一跳的盲目临时密钥。
        ephemeralKey.Mul（＆ephemeralKey，＆blindingFactor）
        ephemeralKey.Mod（＆ephemeralKey，btcec.S256（）。Params（）。N）
    }

    //生成填充，在文中称为“填充字符串”。
    filler：= generateHeaderPadding（“rho”，numHops，hopDataSize，hopSharedSecrets）

    //将字段分配并初始化为零填充切片
    var mixHeader [routingInfoSize] byte
    var nextHmac [hmacSize] byte

    //计算每个跃点的路由信息以及a
    //使用该跳的共享密钥的路由信息的MAC。
    对于i：= numHops  -  1; i> = 0;一世 -  {
        rhoKey：= generateKey（“rho”，hopSharedSecrets [i]）
        muKey：= generateKey（“mu”，hopSharedSecrets [i]）

        hopsData [i] .HMAC = nextHmac

        //移动和混淆路由信息
        streamBytes：= generateCipherStream（rhoKey，numStreamBytes）

        rightShift（mixHeader [：]，hopDataSize）
        buf：=＆bytes.Buffer {}
        hopsData [I] .Encode（BUF）
        copy（mixHeader [：]，buf.Bytes（））
        xor（mixHeader [：]，mixHeader [：]，streamBytes [：routingInfoSize]）

        //这些需要被覆盖，因此每个节点都会生成正确的填充
        if i == numHops-1 {
            copy（mixHeader [len（mixHeader）-len（填充）：]，填充）
        }

        packet：= append（mixHeader [：]，assocData ...）
        nextHmac = calcMac（muKey，包）
    }

    packet：=＆OnionPacket {
        版本：0x00，
        EphemeralKey：sessionKey.PubKey（），
        RoutingInfo：mixHeader，
        HeaderMAC：nextHmac，
    }
    返回包，没有
}
```

# 数据包转发

该规范仅限于`version``0`数据包;结构
未来版本可能会改变。

在接收到分组时，处理节点比较该分组的版本字节
具有自己支持的版本的数据包，如果数据包中止连接
指定它不支持的版本号。
对于支持版本号的数据包，处理节点首先解析
数据包进入其各个字段。

接下来，处理节点使用私钥计算共享秘密
对应于自己的公钥和来自数据包的短暂密钥，如
在[共享秘密]（#shared-secret）中描述。

上述要求可防止沿线路的任何跳跃重试付款
多次，试图通过流量跟踪付款的进度
分析。请注意，禁用此类探测可以使用日志来完成
以前的共享秘密或HMAC，一旦HTLC将被遗忘
无论如何都不被接受（即在'outgoing_cltv_value`过去之后）。这样的日志
可能会使用概率数据结构，但它必须将承诺限制为
必要的，以限制最坏情况的存储要求或错误
这个日志的好消息。

接下来，处理节点使用共享密钥来计算_mu_-key
反过来用来计算`hops_data`的HMAC。然后产生HMAC
与数据包的HMAC进行比较。

计算的HMAC和数据包的HMAC必须比较
时间常数，以避免信息泄漏。

此时，处理节点可以生成_rho_-key和_gamma_-key。

然后对路由信息进行反混淆处理，并提供有关的信息
提取下一跳。
为此，处理节点复制`hops_data`字段，附加65` 0x00`字节，
生成1365个伪随机字节（使用_rho_-key），并应用结果
使用`XOR`来复制`hops_data`。
结果路由信息的前65个字节变为`per_hop`
用于下一跳的字段。接下来的1300个字节是`hops_data`
传出包。

一个特殊的`per_hop``HMAC`值为32！0x00`-字节表示当前
处理跃点是预期的接收者，不应转发数据包。

如果HMAC没有指示路由终止，并且下一跳是对等的
处理节点;然后组装新数据包。数据包组装完成
通过将短暂密钥与处理节点的公钥一起使用，以及
共享秘密，并通过序列化`hops_data`。
然后将得到的数据包转发到被寻址的对等体。

## 要求

处理节点：
  - 如果短暂的公钥不在`secp256k1`曲线上：
    - 必须中止处理数据包。
    - 必须向原始节点报告路由故障。
  - 如果数据包先前已转发或本地兑换，即
  数据包包含重复的路由信息到先前接收的数据包：
    - 如果已知原像：
      - 可以使用preimage立即兑换HTLC。
    - 除此以外：
      - 必须中止处理并报告路由失败。
  - 如果计算的HMAC和数据包的HMAC不同：
    - 必须中止处理。
    - 务必报告路线故障。
  - 如果`领域'是未知的：
    - 必须丢弃数据包。
    - 必须发出路由故障信号。
  - 必须将数据包发送到另一个作为其直接邻居的对等体。
  - 如果处理节点没有匹配地址的对等体：
    - 必须丢弃数据包。
    - 必须发出路由故障信号。


# 填充物生成

在接收到分组时，处理节点提取目的地的信息
从路由信息和每跳有效载荷中获取它。
通过反混淆和左移场来完成提取。
这将使每一跳的字段更短，允许攻击者推断出
路线长度。出于这个原因，该字段在转发之前预先填充。
由于填充是HMAC的一部分，因此原始节点必须预先生成一个
相同的填充（到每个跃点将生成的填充）以便计算
HMAC适用于每一跳。
在所选择的情况下，填充物也用于填充场长
路由比最大允许路由长度20短。

在对'hops_data`进行反混淆处理之前，处理节点用65填充它
`0x00`-字节，这样总长度为`（20 + 1）* 65`。
然后，它生成匹配长度的伪随机字节流，并应用
它与`hops_data`的`XOR`。
这会同时对指定给它的信息进行反混淆处理
在末尾混淆添加的“0x00”字节。

为了计算正确的HMAC，原始节点必须预先生成
每个跃点的`hops_data`，包括添加的增量混淆填充
每一跳。这种递增的混淆填充被称为
`filler`。

以下示例代码显示了如何在Go中生成填充：

```去
func generateFiller（key string，numHops int，hopSize int，sharedSecrets [] [sharedSecretSize] byte）[] byte {
    fillerSize：= uint（（numMaxHops + 1）* hopSize）
    filler：= make（[] byte，fillerSize）

    //最后一跳没有混淆，它不再转发。
    对于i：= 0;我<numHops-1;我++ {

        //左移场
        copy（填充[：]，填充[hopSize：]）

        //零填充最后一跳
        copy（filler [len（filler）-hopSize：]，bytes.Repeat（[] byte {0x00}，hopSize））

        //生成伪随机字节流
        streamKey：= generateKey（key，sharedSecrets [i]）
        streamBytes：= generateCipherStream（streamKey，fillerSize）

        //混淆
        xor（填料，填料，streamBytes）
    }

    //将填充物缩减到正确的长度（numHops + 1）* hopSize
    //数据包将由数据包生成预先添加。
    return filler [（numMaxHops-numHops + 2）* hopSize：]
}
```

请注意，此示例实现仅用于演示目的;该
`filler`可以更有效地生成。
最后一跳不需要混淆`filler`，因为它不会转发数据包
任何进一步的，因此也不需要提取HMAC。

# 返回错误

洋葱路由协议包括一个返回加密的简单机制
错误消息到原始节点。
返回的错误消息可能是任何跃点报告的失败，包括
最终节点。
转发数据包的格式不能用于返回路径，因为没有跳
此外，原产地可以获得其生成所需的信息。
请注意，这些错误消息不可靠，因为它们不是链接的
由于跳跃失败的可能性。

中间跃点存储来自前向路径的共享密钥并将其重用
在每一跳期间混淆任何相应的返回数据包。
此外，每个节点本地存储有关其自己的发送对等体的数据
路由，因此它知道在哪里返回任何最终的返回数据包。
生成错误消息的节点（_erring node_）构建返回数据包
由以下字段组成：

1. 数据：
   * [`32`：`hmac`]
   * [`2`：`failure_len`]
   * [`failure_len`：`failuremsg`]
   * [`2`：`pad_len`]
   * [`pad_len`：`pad`]

其中`hmac`是一个HMAC，用密钥验证数据包的其余部分
使用上面的过程生成，使用键类型`um`，`failuremsg`定义
下面，`pad`作为用于隐藏长度的额外字节。

然后，错误的节点使用密钥类型“ammag”生成一个新密钥。
然后，该密钥用于生成伪随机流，该伪随机流又是
使用`XOR`应用于数据包。

沿着返回路径的每一跳重复混淆步骤。
收到返回数据包后，每一跳产生其“ammag”，生成
伪随机字节流，并将结果应用于返回数据包之前
返回它。

原始节点能够检测到它是预期的最终接收者
返回消息，因为它当然是相应的发起者
转发数据包。
当原始节点收到与其启动的传输匹配的错误消息时
（即它不能再向前返回错误）它会生成`ammag`
和路径中每一跳的“um”键。
然后使用每个跃点的“ammag”迭代地解密错误消息
键，并使用每一跳的'um`键计算HMAC。
origin节点可以通过匹配来检测错误消息的发送者
带有计算HMAC的`hmac`字段。

正向和返回数据包之间的关联在外部处理
这种洋葱路由协议，例如，通过与支付中的HTLC关联
渠道。

### 要求

_erring node_：
  - 应该设置`pad`，使`failure_len`加上'pad_len`等于256。
    - 注意：此值比当前定义的最长值长118个字节
    信息。

_origin node_：
  - 一旦返回消息被解密：
    - 应该存储邮件的副本。
    - 应该继续解密，直到循环重复20次。
    - 应该使用常量`ammag`和`um`键来混淆路径长度。

## 失败消息

封装在`failuremsg`中的失败消息具有相同的格式
正常消息：2字节类型`failure_code`后跟适用的数据
到那种类型。下面是当前支持的`failure_code`列表
值，然后是用例要求。

请注意，`failure_code`s与其他消息类型的类型不同，
在其他BOLT中定义，因为它们不直接在传输层上发送
而是包裹在返回数据包中。
因此，“failure_code”的数值可以重用值
也分配给其他消息类型，没有任何导致冲突的危险。

`failure_code`的顶部字节可以读作一组标志：
* 0x8000（BADONION）：通过发送对等方加密的不可解析的洋葱
* 0x4000（PERM）：永久性故障（否则为瞬态）
* 0x2000（NODE）：节点故障（否则为通道）
* 0x1000（更新）：附带新的频道更新

请注意，`channel_update`字段在其邮件中必填
`failure_code`包含`UPDATE`标志。

定义了以下`failure_code`：

1. 类型：PERM | 1（`invalid_realm`）

处理节点不理解`realm`字节。

1. 类型：NODE | 2（`temporary_node_failure`）

处理节点的一般临时故障。

1. 类型：PERM | NODE | 2（`permanent_node_failure`）

处理节点的一般永久性故障。

1. 类型：PERM | NODE | 3（`required_node_feature_missing`）

处理节点具有不在此洋葱中的所需功能。

1. 类型：BADONION | PERM | 4（`invalid_onion_version`）
2. 数据：
   * [`32`：`sha256_of_onion`]

处理节点不理解`version`字节。

1. 类型：BADONION | PERM | 5（`invalid_onion_hmac`）
2. 数据：
   * [`32`：`sha256_of_onion`]

洋葱的HMAC在到达处理节点时是不正确的。

1. 类型：BADONION | PERM | 6（`invalid_onion_key`）
2. 数据：
   * [`32`：`sha256_of_onion`]

处理节点无法解析临时密钥。

1. type：UPDATE | 7（`temporary_channel_failure`）
2. 数据：
   * [`2`：`len`]
   * [`len`：`channel_update`]

来自处理节点的通道无法处理此HTLC，
但以后可能能够处理它或其他人。

1. 类型：PERM | 8（`permanent_channel_failure`）

处理节点的通道无法处理任何HTLC。

1. 类型：PERM | 9（`required_channel_feature_missing`）

处理节点的通道需要不存在的功能
洋葱。

1. 类型：PERM | 10（`unknown_next_peer`）

洋葱指定了一个与任何不匹配的`short_channel_id`
从处理节点开始。

1. 类型：UPDATE | 11（`amount_below_minimum`）
2. 数据：
   * [`8`：`htlc_msat`]
   * [`2`：`len`]
   * [`len`：`channel_update`]

HTLC数量低于通道的`htlc_minimum_msat`
处理节点。

1. 类型：UPDATE | 12（`fee_insufficient`）
2. 数据：
   * [`8`：`htlc_msat`]
   * [`2`：`len`]
   * [`len`：`channel_update`]

费用金额低于渠道要求的金额
处理节点。

1. 类型：UPDATE | 13（`incorrect_cltv_expiry`）
2. 数据：
   * [`4`：`cltv_expiry`]
   * [`2`：`len`]
   * [`len`：`channel_update`]

`cltv_expiry`不符合所需的`cltv_expiry_delta`
来自处理节点的通道：它不满足以下要求
需求：

        cltv_expiry  -  cltv_expiry_delta> = outgoing_cltv_value

1. 类型：UPDATE | 14（`expiry_too_soon`）
2. 数据：
   * [`2`：`len`]
   * [`len`：`channel_update`]

CLTV到期时太接近当前的区块高度以确保安全
处理节点处理。

1. 类型：PERM | 15（`incorrect_or_unknown_payment_details`）
2. 数据：
   * [`8`：`htlc_msat`]

`payment_hash`对于最终节点或其数量是未知的
`payment_hash`不正确。

注意：最初的PERM | 16（`incorrect_payment_amount`）是
用于区分不正确的最终金额与未知付款
哈希值。遗憾的是，发送此响应允许探测节点的攻击
接收HTLC进行转发可以检查其最终的猜测
目的地通过发送具有相同哈希但更低值的付款
潜在目的地并检查响应。

1. 类型：17（`final_expiry_too_soon`）

CLTV到期时太接近当前的区块高度以确保安全
由最终节点处理。

1. 类型：18（`final_incorrect_cltv_expiry`）
2. 数据：
   * [`4`：`cltv_expiry`]

HTLC中的CLTV到期与洋葱中的值不匹配。

1. 类型：19（`final_incorrect_htlc_amount`）
2. 数据：
   * [`8`：`incoming_htlc_amt`]

HTLC中的量与洋葱中的值不匹配。

1. type：UPDATE | 20（`channel_disabled`）
2. 数据：
   * [`2`：`flags`]
   * [`2`：`len`]
   * [`len`：`channel_update`]

处理节点的通道已被禁用。

1. 类型：21（`expiry_too_far`）

HTLC的CLTV到期时间太长了。

### 要求

_erring node_：
  - 创建错误消息时必须选择上述错误代码之一。
  - 必须包含该特定错误类型的适当数据。
  - 如果有多个错误：
    - 应该从上面的列表中选择它遇到的第一个错误。

任何_erring node_ MAY：
  - 如果`realm`字节未知：
    - 返回`invalid_realm`错误。
  - 如果整个节点发生其他未指定的瞬态错误：
    - 返回`temporary_node_failure`错误。
  - 如果整个节点发生其他未指定的永久性错误：
    - 返回`permanent_node_failure`错误。
  - 如果节点的“node_announcement`` features`中有广告要求，
  哪些不包括在洋葱中：
    - 返回`required_node_feature_missing`错误。

一个_forwarding node_ MAY，但是_final node_绝不能：
  - 如果洋葱`版本`字节未知：
    - 返回`invalid_onion_version`错误。
  - 如果洋葱HMAC不正确：
    - 返回`invalid_onion_hmac`错误。
  - 如果洋葱中的短暂钥匙是不可解的：
    - 返回`invalid_onion_key`错误。
  - 如果在转发到其接收对等体期间，否则未指定，
  输出信道中发生瞬时错误（例如，达到信道容量，
  太多的飞行HTLC等）：
    - 返回`temporary_channel_failure`错误。
  - 如果在转发到其期间发生了其他未指定的永久性错误
  接收对等方（例如最近关闭的频道）：
    - 返回`permanent_channel_failure`错误。
  - 如果传出通道在其中公布了要求
  `channel_announcement`的`features`，不包含在洋葱中：
    - 返回`required_channel_feature_missing`错误。
  - 如果洋葱指定的接收对等体未知：
    - 返回`unknown_next_peer`错误。
  - 如果HTLC金额低于当前指定的最低金额：
    - 报告输出HTLC的数量和当前通道设置
    传出渠道。
    - 返回`amount_below_minimum`错误。
  - 如果HTLC没有支付足够的费用：
    - 报告传入HTLC的数量和当前频道设置
    传出渠道。
    - 返回`fee_insufficient`错误。
 -  如果传入的`cltv_expiry`减去`outgoing_cltv_value`低于
    传出通道的`cltv_expiry_delta`：
    - 报告输出HTLC的`cltv_expiry`和输出的当前通道设置
    渠道。
    - 返回`incorrect_cltv_expiry`错误。
  - 如果`cltv_expiry`在现在附近是不合理的：
    - 报告传出通道的当前通道设置。
    - 返回`expiry_too_soon`错误。
  - 如果`cltv_expiry`在将来是不合理的：
    - 返回`expiry_too_far`错误。
  - 如果频道被禁用：
    - 报告传出通道的当前通道设置。
    - 返回`channel_disabled`错误。

一个_intermediate hop_绝不可以，但_final node_：
  - 如果支付哈希已经支付：
    - 可以将支付哈希视为未知。
    - 可以成功接受HTLC。
  - 如果支付的金额低于预期金额：
    - 必须通过HTLC失败。
    - 必须返回一个`incorrect_or_unknown_payment_details`错误。
  - 如果支付哈希值未知：
    - 必须通过HTLC失败。
    - 必须返回一个`incorrect_or_unknown_payment_details`错误。
  - 如果支付的金额超过预期金额的两倍：
    - 应该让HTLC失败。
    - 应该返回一个`incorrect_or_unknown_payment_details`错误。
      - 注意：这允许源节点减少信息泄漏
      在不允许意外超额支付的情况下改变金额。
  - 如果`cltv_expiry`值不合理地接近现在：
    - 必须通过HTLC失败。
    - 必须返回`final_expiry_too_soon`错误。
  - 如果`outgoing_cltv_value`与`cltv_expiry`不对应
  最终节点的HTLC：
    - 必须返回`final_incorrect_cltv_expiry`错误。
  - 如果`amt_to_forward`大于`incoming_htlc_amt`
  最终节点的HTLC：
    - 必须返回`final_incorrect_htlc_amount`错误。

## 接收失败代码

### 要求

_origin node_：
  - 必须忽略`failuremsg`中的任何额外字节。
  - 如果_final node_返回错误：
    - 如果PERM位置位：
      - 应该不付款。
    - 除此以外：
      - 如果错误代码被理解并且有效：
        - 可以重试付款。特别是，`final_expiry_too_soon`可以
        如果自发送以来块高度发生了变化，则会发生这种情况
        `temporary_node_failure`可以在几秒钟内解决。
  - 否则，_intermediate hop_返回错误：
    - 如果NODE位置位：
      - 应该删除与错误节点连接的所有通道
      考虑。
    - 如果未设置PERM位：
      - 应该在收到新的channel_update时恢复通道。
    - 除此以外：
      - 如果设置了UPDATE，则`channel_update`有效且更新
      比用于发送付款的`channel_update`：
        - 如果`channel_update`不应该导致失败：
          - 可以将`channel_update`视为无效。
        - 除此以外：
          - 应该应用`channel_update`。
        - 可以将`channel_update`排队等待广播。
      - 除此以外：
        - 应该消除从错误节点传出的通道
        考虑。
        - 如果未设置PERM位：
          - 应该在接收到新的channel_update时恢复通道。
    - 然后应该重试路由并发送付款。
  - 可以使用各种故障类型中指定的数据进行调试
  目的。

# 测试矢量

## 数据包创建

以下是示例的深入跟踪（包括中间数据）
数据包创建：

### 参数

    pubkey [0] = 0x02eec7245d6b7d2ccb30380bfbe2a3648cd7a942653f5aa340edcea1f283686619
    pubkey [1] = 0x0324653eac434488002cc06bbfb7f10fe18991e35f9fe4302dbea6d2353dc0ab1c
    pubkey [2] = 0x027f31ebc5462c1fdce1b737ecff52d37d75dea43ce11c74d25aa297165faa2007
    pubkey [3] = 0x032c0b7cf95324a07d05398b240174dc0c2be444d96b159aa6c7f7b1e668680991
    pubkey [4] = 0x02edabbd16b41c8371b92ef2f04c1185b4f03b6dcd52ba9b78d9d7c89c8f221145

    nhops = 5/20
    sessionkey = 0x414141414141414141414141414141414141414141414141414141414141414141
    相关数据= 0x424242424242424242424242424242424242424242424242424242424242424242

HMAC在以下`hop_data`中被省略，因为它可能被填充
通过洋葱建设。因此，下面的值是“领域”，即
`short_channel_id`，`amt_to_forward`，`outgoing_cltv`和12字节
`padding`。它们通过字节填充`short_channel_id`来初始化
每一跳在路线中的相应位置，然后从0开始设定
`amt_to_forward`和`outgoing_cltv`到同一个路线位置。

    hop_payload [0] = 0x000000000000000000000000000000000000000000000000000000000000000000
    hop_payload [1] = 0x000101010101010101000000000000000100000001000000000000000000000000
    hop_payload [2] = 0x000202020202020202000000000000000200000002000000000000000000000000
    hop_payload [3] = 0x000303030303030303000000000000000300000003000000000000000000000000
    hop_payload [4] = 0x000404040404040404000000000000000400000004000000000000000000000000

### 每跳信息

    hop_shared_secret [0] = 0x53eb63ea8a3fec3b3cd433b85cd62a4b145e1dda09391b348c4e1cd36a03ea66
    hop_blinding_factor [0] = 0x2ec2e5da605776054187180343287683aa6a51b4b1c04d6dd49c45d8cffb3c36
    hop_ephemeral_pubkey [0] = 0x02eec7245d6b7d2ccb30380bfbe2a3648cd7a942653f5aa340edcea1f283686619

    hop_shared_secret [1] = 0xa6519e98832a0b179f62123b3567c106db99ee37bef036e783263602f3488fae
    hop_blinding_factor [1] = 0xbf66c28bc22e598cfd574a1931a2bafbca09163df2261e6d0056b2610dab938f
    hop_ephemeral_pubkey [1] = 0x028f9438bfbf7feac2e108d677e3a82da596be706cc1cf342b75c7b7e22bf4e6e2

    hop_shared_secret [2] = 0x3a6b412548762f0dbccce5c7ae7bb8147d1caf9b5471c34120b30bc9c04891cc
    hop_blinding_factor [2] = 0xa1f2dadd184eb1627049673f18c6325814384facdee5bfd935d9cb031a1698a5
    hop_ephemeral_pubkey [2] = 0x03bfd8225241ea71cd0843db7709f4c222f62ff2d4516fd38b39914ab6b83e0da0

    hop_shared_secret [3] = 0x21e13c2d7cfe7e18836df50872466117a295783ab8aab0e7ecc8c725503ad02d
    hop_blinding_factor [3] = 0x7cfe0b699f35525029ae0fa437c69d0f20f7ed4e3916133f9cacbb13c82ff262
    hop_ephemeral_pubkey [3] = 0x031dde6926381289671300239ea8e57ffaf9bebd05b9a5b95beaf07af05cd43595

    hop_shared_secret [4] = 0xb5756b9b542727dbafc6765a49488b023a725d631af688fc031217e90770c328
    hop_blinding_factor [4] = 0xc96e00dddaf57e7edcd4fb5954be5b65b09f17cb6d20651b4e90315be5779205
    hop_ephemeral_pubkey [4] = 0x03a214ebd875aab6ddfd77f22c5e7311d7f77f17a169e599f157bbcdae8bf071f4

### 每包信息

    填料= 0xc6b008cf6414ed6e4c42c291eb505e9f22f5fe7d0ecdd15a833f4d016ac974d33adc6ea3293e20859e87ebfb937ba406abd025d14af692b12e9c9c2adbe307a679779259676211c071e614fdb386d1ff02db223a5b2fae03df68d321c7b29f7c7240edd3fa1b7cb6903f89dc01abf41b2eb0b49b6b8d73bb0774b58204c0d0e96d3cce45ad75406be0bc009e327b3e712a4bd178609c00b41da2daf8a4b0e1319f07a492ab4efb056f0f599f75e6dc7e0d10ce1cf59088ab6e873de377343880f7a24f0e36731a0b72092f8d5bc8cd346762e93b2bf203d00264e4bc136fc142de8f7b69154deb05854ea88e2d7506222c95ba1aab065c8a851391377d3406a35a9af3ac

### 包装洋葱

    rhokey[4] = 0x034e18b8cc718e8af6339106e706c52d8df89e2b1f7e9142d996acf88df8799b
    mukey[4] = 0x8e45e5c61c2b24cb6382444db6698727afb063adecd72aada233d4bf273d975a
    routing_info[4] (unencrypted) = 0x00040404040404040400000004000000040000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    routing_info[4] (encrypted) = 0xf6a9e85e5c6a0458c98282b786ffe8efb2f3f0e2cee0283a71aa4d873686d7efeac03dcb113fd91113df4bb5d5e8c13cde4bd12eced21d367381fdc31efe09695d6ecc095083c1571d10ed076ee914745479dfedcab84efd3889e6375f544562ea4a6522035738a9ba1a9ab4204339569d3d0c217324f1d5f3a107e930ade50b913777d1140096e2b26ce47486b66a6740f026ae91429115e17270c5a542b4c82ce438725060248726956e7639da64a55cec00b61719383271e0ab97dafe44ad194e7338ea841bf25a505d0041eb53dc0d0e6c3f7986f6647ede9327467212a3ca5724ece8503bfaba3800372d747abe2c174cbc2da01dd967fc0bae56d2bc7184e1d5d823661b8ff06eb466d483d6fa69a4f8b592ebc3006b7c047055b12407925ea74c2bb09ca21f4bc9d39dc4a7237ef6facbb15f5ba56d8e2ab94ad24267bf85cea55b0605cd6260b4b0f17994a03e39bebfcad0834849b188e83bf5c954dc112d33b0fada26f25d76d53ec9b95ed344059a279719d5edc953a4de2b1cc4f47d5da090c3b0f4c7eef14830f9a2230bffb722f11fd2ea82c43d58261e5a868361262abbd81b9f1e6145562ed6765aac732e7d91bfff7716509d72314d46904bcc8053cfd1e3cd7824b4b6499bf8c77762275fa4f651d163a8701f02a5ca67bb7d37787a66d269d4079aa92a489493020b11d87791a3e3ef66780bd0007c8fa2782bac7a839b57e8ace7618a75fe03604097960d2236616d579207fb943c85bef4f804e7f13d7d78ff3aab0f7f440a3005a363f32c31567b8d64669a7ae0299b06b80e4d224d2d70b80077ff966cfb8c11cf763954f398c592ca619acc2e0e86dad54b9cfa47bc10520f3c9ba2e0d2e27e0ba8ef8ece3e71c8a1ff75fe0c8369a3555e42db30565b2e12a8e862d0f873bd781ebf8255372e540bf6fbf4c1dae200823144f8da8df82ccd41b1b678eb91a289b88c6ee5aff71d98b64e8b17e2027fcb6826c418dbdb51f06bec866d69d9554931412808bd90021be2e0dad1d1ececfdd41fcdf6f67b88ef09eb9edcc4c226e07bdfe86b8e50e3bf68b19c0c9e0bf9aaaaddb15bc05a5666733789fa6efde866a49d0329185d852dc84a7e16c41f6c2daadc04665197152d9c0db63d0bd926c06bf3646b810186ae908a2f27a33d010b3262b0c0805c3adf135b41f15a033bb82a8db2c38160dbce672290878d7ad8d08fdd483ef9d95b3a1b2ea2b6ff0adde762e9df7b78fe5f3644df54c6d98709d70924ce6ec94650cb207b4f90d35569acb55811ad3f01b28bb2b16cfde454531b9462024abf419fb3bef80db9bb9fa5bd62a7e94f61a667e74140c8da4b27ad7b934c0824756fbc56f8ac7529fc4c5224158fd4915eba25a3f2a72a7f718a5cda6395bd8b2bc484a950aba483c8cadec376f9bee62c93f886d83371b41c361a36c15679ff933422e09c5eaf98d3bdff98680040079daed1d88cfdb74a922c59c81327133971ae5755ffe589152b2d9423a0dcd6b9540cb57a3146fdc256c573c26aa148a00ef28441b3c49ceedff190da2380ebc7bc19ef3082a195f47fdf453e0e0a54e10811b604df1d3c2238d4ec13ce0feb73216a7f100fa6eb7953f42858ef190aad1d271a5262aefb8d7ee6430061899695058feb51c914dc93a24dd1b5cf2b54334b385c54bc9b83748b966fda0d55d8da3324aa1ce4f835d44b379e785b0a95c7c9f7bae6df9cad098f40b0b159e416c5ed4d60950b40a30ecc0c125dbe6ff2cfcec7f8d36925d4252f276279c4d3680bd7180b991a320eba22c5d71079630deae6bf4f766da7cf62d6c410eb5f
    hmac_data[4] = 0xf6a9e85e5c6a0458c98282b786ffe8efb2f3f0e2cee0283a71aa4d873686d7efeac03dcb113fd91113df4bb5d5e8c13cde4bd12eced21d367381fdc31efe09695d6ecc095083c1571d10ed076ee914745479dfedcab84efd3889e6375f544562ea4a6522035738a9ba1a9ab4204339569d3d0c217324f1d5f3a107e930ade50b913777d1140096e2b26ce47486b66a6740f026ae91429115e17270c5a542b4c82ce438725060248726956e7639da64a55cec00b61719383271e0ab97dafe44ad194e7338ea841bf25a505d0041eb53dc0d0e6c3f7986f6647ede9327467212a3ca5724ece8503bfaba3800372d747abe2c174cbc2da01dd967fc0bae56d2bc7184e1d5d823661b8ff06eb466d483d6fa69a4f8b592ebc3006b7c047055b12407925ea74c2bb09ca21f4bc9d39dc4a7237ef6facbb15f5ba56d8e2ab94ad24267bf85cea55b0605cd6260b4b0f17994a03e39bebfcad0834849b188e83bf5c954dc112d33b0fada26f25d76d53ec9b95ed344059a279719d5edc953a4de2b1cc4f47d5da090c3b0f4c7eef14830f9a2230bffb722f11fd2ea82c43d58261e5a868361262abbd81b9f1e6145562ed6765aac732e7d91bfff7716509d72314d46904bcc8053cfd1e3cd7824b4b6499bf8c77762275fa4f651d163a8701f02a5ca67bb7d37787a66d269d4079aa92a489493020b11d87791a3e3ef66780bd0007c8fa2782bac7a839b57e8ace7618a75fe03604097960d2236616d579207fb943c85bef4f804e7f13d7d78ff3aab0f7f440a3005a363f32c31567b8d64669a7ae0299b06b80e4d224d2d70b80077ff966cfb8c11cf763954f398c592ca619acc2e0e86dad54b9cfa47bc10520f3c9ba2e0d2e27e0ba8ef8ece3e71c8a1ff75fe0c8369a3555e42db30565b2e12a8e862d0f873bd781ebf8255372e540bf6fbf4c1dae200823144f8da8df82ccd41b1b678eb91a289b88c6ee5aff71d98b64e8b17e2027fcb6826c418dbdb51f06bec866d69d9554931412808bd90021be2e0dad1d1ececfdd41fcdf6f67b88ef09eb9edcc4c226e07bdfe86b8e50e3bf68b19c0c9e0bf9aaaaddb15bc05a5666733789fa6efde866a49d0329185d852dc84a7e16c41f6c2daadc04665197152d9c0db63d0bd926c06bf3646b810186ae908a2f27a33d010b3262b0c0805c3adf135b41f15a033bb82a8db2c38160dbce672290878d7ad8d08fdd483ef9d95b3a1b2ea2b6ff0adde762e9df7b78fe5f3644df54c6d98709d70924ce6ec94650cb207b4f90d35569acb55811ad3f01b28bb2b16cfde454531b9462024abf419fb3bef80db9bb9fa5bd62a7e94f61a667e74140c8da4b27ad7b934c0824756fbc56f8ac7529fc4c5224158fd4915eba25a3f2a72a7f718a5cda6395bd8b2bc484a950aba483c8cadec376f9bee62c93f886d83371b41c361a36c15679ff933422e09c5eaf98d3c6b008cf6414ed6e4c42c291eb505e9f22f5fe7d0ecdd15a833f4d016ac974d33adc6ea3293e20859e87ebfb937ba406abd025d14af692b12e9c9c2adbe307a679779259676211c071e614fdb386d1ff02db223a5b2fae03df68d321c7b29f7c7240edd3fa1b7cb6903f89dc01abf41b2eb0b49b6b8d73bb0774b58204c0d0e96d3cce45ad75406be0bc009e327b3e712a4bd178609c00b41da2daf8a4b0e1319f07a492ab4efb056f0f599f75e6dc7e0d10ce1cf59088ab6e873de377343880f7a24f0e36731a0b72092f8d5bc8cd346762e93b2bf203d00264e4bc136fc142de8f7b69154deb05854ea88e2d7506222c95ba1aab065c8a851391377d3406a35a9af3ac4242424242424242424242424242424242424242424242424242424242424242
    hmac[4] = 0x9b122c79c8aee73ea2cdbc22eca15bbcc9409a4cdd73d2b3fcd4fe26a492d376

    rhokey[3] = 0xcbe784ab745c13ff5cffc2fbe3e84424aa0fd669b8ead4ee562901a4a4e89e9e
    mukey[3] = 0x5052aa1b3d9f0655a0932e50d42f0c9ba0705142c25d225515c45f47c0036ee9
    routing_info[3] (unencrypted) = 0x00030303030303030300000003000000030000000000000000000000000000000042c10947e06bda75b35ac2a9e38005479a6feac51468712e751c71a1dcf3e31bf6a9e85e5c6a0458c98282b786ffe8efb2f3f0e2cee0283a71aa4d873686d7efeac03dcb113fd91113df4bb5d5e8c13cde4bd12eced21d367381fdc31efe09695d6ecc095083c1571d10ed076ee914745479dfedcab84efd3889e6375f544562ea4a6522035738a9ba1a9ab4204339569d3d0c217324f1d5f3a107e930ade50b913777d1140096e2b26ce47486b66a6740f026ae91429115e17270c5a542b4c82ce438725060248726956e7639da64a55cec00b61719383271e0ab97dafe44ad194e7338ea841bf25a505d0041eb53dc0d0e6c3f7986f6647ede9327467212a3ca5724ece8503bfaba3800372d747abe2c174cbc2da01dd967fc0bae56d2bc7184e1d5d823661b8ff06eb466d483d6fa69a4f8b592ebc3006b7c047055b12407925ea74c2bb09ca21f4bc9d39dc4a7237ef6facbb15f5ba56d8e2ab94ad24267bf85cea55b0605cd6260b4b0f17994a03e39bebfcad0834849b188e83bf5c954dc112d33b0fada26f25d76d53ec9b95ed344059a279719d5edc953a4de2b1cc4f47d5da090c3b0f4c7eef14830f9a2230bffb722f11fd2ea82c43d58261e5a868361262abbd81b9f1e6145562ed6765aac732e7d91bfff7716509d72314d46904bcc8053cfd1e3cd7824b4b6499bf8c77762275fa4f651d163a8701f02a5ca67bb7d37787a66d269d4079aa92a489493020b11d87791a3e3ef66780bd0007c8fa2782bac7a839b57e8ace7618a75fe03604097960d2236616d579207fb943c85bef4f804e7f13d7d78ff3aab0f7f440a3005a363f32c31567b8d64669a7ae0299b06b80e4d224d2d70b80077ff966cfb8c11cf763954f398c592ca619acc2e0e86dad54b9cfa47bc10520f3c9ba2e0d2e27e0ba8ef8ece3e71c8a1ff75fe0c8369a3555e42db30565b2e12a8e862d0f873bd781ebf8255372e540bf6fbf4c1dae200823144f8da8df82ccd41b1b678eb91a289b88c6ee5aff71d98b64e8b17e2027fcb6826c418dbdb51f06bec866d69d9554931412808bd90021be2e0dad1d1ececfdd41fcdf6f67b88ef09eb9edcc4c226e07bdfe86b8e50e3bf68b19c0c9e0bf9aaaaddb15bc05a5666733789fa6efde866a49d0329185d852dc84a7e16c41f6c2daadc04665197152d9c0db63d0bd926c06bf3646b810186ae908a2f27a33d010b3262b0c0805c3adf135b41f15a033bb82a8db2c38160dbce672290878d7ad8d08fdd483ef9d95b3a1b2ea2b6ff0adde762e9df7b78fe5f3644df54c6d98709d70924ce6ec94650cb207b4f90d35569acb55811ad3f01b28bb2b16cfde454531b9462024abf419fb3bef80db9bb9fa5bd62a7e94f61a667e74140c8da4b27ad7b934c0824756fbc56f8ac7529fc4c5224158fd4915eba25a3f2a72a7f718a5cda6395bd8b2bc484a950aba483c8cadec376f9bee62c93f886d83371b41c361a36c15679ff933422e09c5eaf98d3c6b008cf6414ed6e4c42c291eb505e9f22f5fe7d0ecdd15a833f4d016ac974d33adc6ea3293e20859e87ebfb937ba406abd025d14af692b12e9c9c2adbe307a679779259676211c071e614fdb386d1ff02db223a5b2fae03df68d321c7b29f7c7240edd3fa1b7cb6903f89dc01abf41b2eb0b49b6b8d73bb0774b58204c0d0e96d3cce45ad75406be0bc009e327b3e712a4bd178609c00b41da2daf8a4b0e1319f07a492ab4efb056f0f599f75e6dc7e0d10ce1cf59088ab6e873de377343880f7a24f
    routing_info[3] (encrypted) = 0x35dd8ba6f4e53e2f0372992045827fc994c3312b54591844fc713c0cca433626a09629ccb8c0e8f28648628269f29e8bcd5941429852a5e34fdfe080f74b0f19ffaf3e17e529ce3c0c58316f255e42848df14bce0781a4653c84ac8f58bf89671cd0d4b692e9e42584a52509e03f8769bb4ef6a0f6f4002c81011da7f5f9a8da9b19bd25cb25aeb3873c5fa95396b56900847794a97935e542637ef5f42f088c6e24e8e95de7be71d110872c84a60a53bbd789f3a58c1072c5fbb84192dd5f780deb14bdbbf458be297eb3244499a7921db98912625c88350cb29e71ca8d788bfb9bd11146e48659b0d86afe13b47685ec49d36e57292d819442ba516d365be31e7980c0e7b14f40807393b1391d9b83c30191d4bfe49c143da2848d2d7bfd54c32bdfdaefcd4cb84ea895e2305745d2ae257b0414563224af66f3b25d3d451bdd8331670e9dcac78b0690c4ae849d0bd8e956dbc56b5a8114008e9dc54f1252544a45861521efd6376b6c0fb34c1c433b677c08d6bf5356e81772bfeed5a894b590859cd409dedd1cd9881c3b8caed81d4d8d834dbd3e870a39e69d6a1ce2dc15df8f8c34d64d292b0b99bf17bb92b263176b65634dc8f368cf7b558fef5b259964f42ec94eeac74d0abd829baa9b04f89ea3606074a4aace03e7f2217f2dec855f4028392f64f0baddd178d56d84cab16c8484633f7c5e8e7f8651ce8a7f5ce1ca3a966562a0a2a7247d105a3114ab2adbd10cfd70b2765113c42e4d1cbcb8721a719f46b25d8579a670be085d1afd2d2e541c4a27bdfffe49674798cf690f28bf4f58ec1495561be071d1c94156f15c95f3426a7409680ef8cf335a0d4786bbb0bf108589c3c8baf10f04dc7f8beda2f70117f28c771a83890d4e1c47903d642fef8a93a216576f377db3ad5be3abe7eaa924e1e7de9ac88e07177e57cad41ef5a561f6bdb741961eddee56f3d9aef9fc4e52343f2efda3a2d40b8d2b4afa735cd5655d3014d21b41eba3a8cf34a8c29cbf94dfb9a4312ee0851c6e32896b33d52d991560e3ae97c769071453a4969374aecf7423f07ee17ce71111dc925294b0130249ad00d49e26dbfddcaf214dc8356cee3a0d7486fa6144c6c826be0892ceea30e52d7b64a13b08eda3e991f37e210ccd216108c847adb58df3b2a53553689f5119e41b4f4f221474c21e197d8a13e9cf7cf23875a0dbc6ddaeca9eb6d95add2a0054528d9dc03d59bc108157291b4caa0faa53f2511bec3088a164a0768d38d7fe3281ea8459abbe3a5b5f43c697e5cbdc995832220a3c6fcf1e82bd35f4b94b13cc9312efc3ca4e3c8199e862c551fa7916e3cb0ad17ba24e456fcd2159f9e81628c508da4c2de4825100fee1c3226d1d431a740d2d8ab0c04953b073d9a20919e0f2d03e97b3f34a1644cbeeab7d0c78d898e8c523fbccc75ffc329fafe45288e21c82817e23589d0c6db453be7d3f8ecfa0ae1d24ae80c63511193c718195255a576c1b43c8d941ba22f6d9f743a7dd558e08ac2262fd67056d60e837edd3e21ce23ef27d48c9180d845ce8c5d6d4c488fcf55af99aef02f3bdf6a07833660628a672b878f8b15427189e49bf2d9e0e7e63e3c5632d9ef1e412e9bbd732b616a353097e90494a098df6a21729f1d3658f91b1bde4aaaae58530ab0e402fc10eb0910c07cace1afd0aacb579690e6dcbc184025e4160cf4de3e47106339046724d098b5b7b92f5a2bb33c11f86d4953f372fdd9ebc260b0ee2e391420c4b11145bd439954834d9a79e78abc57e03d3ee20d239d8a13014976e3f057ab3c38ca79ee81ff8849d94dca37b0920cc3e72
    hmac_data[3] = 0x35dd8ba6f4e53e2f0372992045827fc994c3312b54591844fc713c0cca433626a09629ccb8c0e8f28648628269f29e8bcd5941429852a5e34fdfe080f74b0f19ffaf3e17e529ce3c0c58316f255e42848df14bce0781a4653c84ac8f58bf89671cd0d4b692e9e42584a52509e03f8769bb4ef6a0f6f4002c81011da7f5f9a8da9b19bd25cb25aeb3873c5fa95396b56900847794a97935e542637ef5f42f088c6e24e8e95de7be71d110872c84a60a53bbd789f3a58c1072c5fbb84192dd5f780deb14bdbbf458be297eb3244499a7921db98912625c88350cb29e71ca8d788bfb9bd11146e48659b0d86afe13b47685ec49d36e57292d819442ba516d365be31e7980c0e7b14f40807393b1391d9b83c30191d4bfe49c143da2848d2d7bfd54c32bdfdaefcd4cb84ea895e2305745d2ae257b0414563224af66f3b25d3d451bdd8331670e9dcac78b0690c4ae849d0bd8e956dbc56b5a8114008e9dc54f1252544a45861521efd6376b6c0fb34c1c433b677c08d6bf5356e81772bfeed5a894b590859cd409dedd1cd9881c3b8caed81d4d8d834dbd3e870a39e69d6a1ce2dc15df8f8c34d64d292b0b99bf17bb92b263176b65634dc8f368cf7b558fef5b259964f42ec94eeac74d0abd829baa9b04f89ea3606074a4aace03e7f2217f2dec855f4028392f64f0baddd178d56d84cab16c8484633f7c5e8e7f8651ce8a7f5ce1ca3a966562a0a2a7247d105a3114ab2adbd10cfd70b2765113c42e4d1cbcb8721a719f46b25d8579a670be085d1afd2d2e541c4a27bdfffe49674798cf690f28bf4f58ec1495561be071d1c94156f15c95f3426a7409680ef8cf335a0d4786bbb0bf108589c3c8baf10f04dc7f8beda2f70117f28c771a83890d4e1c47903d642fef8a93a216576f377db3ad5be3abe7eaa924e1e7de9ac88e07177e57cad41ef5a561f6bdb741961eddee56f3d9aef9fc4e52343f2efda3a2d40b8d2b4afa735cd5655d3014d21b41eba3a8cf34a8c29cbf94dfb9a4312ee0851c6e32896b33d52d991560e3ae97c769071453a4969374aecf7423f07ee17ce71111dc925294b0130249ad00d49e26dbfddcaf214dc8356cee3a0d7486fa6144c6c826be0892ceea30e52d7b64a13b08eda3e991f37e210ccd216108c847adb58df3b2a53553689f5119e41b4f4f221474c21e197d8a13e9cf7cf23875a0dbc6ddaeca9eb6d95add2a0054528d9dc03d59bc108157291b4caa0faa53f2511bec3088a164a0768d38d7fe3281ea8459abbe3a5b5f43c697e5cbdc995832220a3c6fcf1e82bd35f4b94b13cc9312efc3ca4e3c8199e862c551fa7916e3cb0ad17ba24e456fcd2159f9e81628c508da4c2de4825100fee1c3226d1d431a740d2d8ab0c04953b073d9a20919e0f2d03e97b3f34a1644cbeeab7d0c78d898e8c523fbccc75ffc329fafe45288e21c82817e23589d0c6db453be7d3f8ecfa0ae1d24ae80c63511193c718195255a576c1b43c8d941ba22f6d9f743a7dd558e08ac2262fd67056d60e837edd3e21ce23ef27d48c9180d845ce8c5d6d4c488fcf55af99aef02f3bdf6a07833660628a672b878f8b15427189e49bf2d9e0e7e63e3c5632d9ef1e412e9bbd732b616a353097e90494a098df6a21729f1d3658f91b1bde4aaaae58530ab0e402fc10eb0910c07cace1afd0aacb579690e6dcbc184025e4160cf4de3e47106339046724d098b5b7b92f5a2bb33c11f86d4953f372fdd9ebc260b0ee2e391420c4b11145bd439954834d9a79e78abc57e03d3ee20d239d8a13014976e3f057ab3c38ca79ee81ff8849d94dca37b0920cc3e724242424242424242424242424242424242424242424242424242424242424242
    hmac[3] = 0x548e58057ab0a0e6c2d8ad8e855d89f9224279a5652895ea14f60bffb81590eb

    rhokey[2] = 0x11bf5c4f960239cb37833936aa3d02cea82c0f39fd35f566109c41f9eac8deea
    mukey[2] = 0xcaafe2820fa00eb2eeb78695ae452eba38f5a53ed6d53518c5c6edf76f3f5b78
    routing_info[2] (unencrypted) = 0x0002020202020202020000000200000002000000000000000000000000000000004e888d0cc6a90e7f857af18ac858834ac251d0d1c196d198df48a0c5bf81680335dd8ba6f4e53e2f0372992045827fc994c3312b54591844fc713c0cca433626a09629ccb8c0e8f28648628269f29e8bcd5941429852a5e34fdfe080f74b0f19ffaf3e17e529ce3c0c58316f255e42848df14bce0781a4653c84ac8f58bf89671cd0d4b692e9e42584a52509e03f8769bb4ef6a0f6f4002c81011da7f5f9a8da9b19bd25cb25aeb3873c5fa95396b56900847794a97935e542637ef5f42f088c6e24e8e95de7be71d110872c84a60a53bbd789f3a58c1072c5fbb84192dd5f780deb14bdbbf458be297eb3244499a7921db98912625c88350cb29e71ca8d788bfb9bd11146e48659b0d86afe13b47685ec49d36e57292d819442ba516d365be31e7980c0e7b14f40807393b1391d9b83c30191d4bfe49c143da2848d2d7bfd54c32bdfdaefcd4cb84ea895e2305745d2ae257b0414563224af66f3b25d3d451bdd8331670e9dcac78b0690c4ae849d0bd8e956dbc56b5a8114008e9dc54f1252544a45861521efd6376b6c0fb34c1c433b677c08d6bf5356e81772bfeed5a894b590859cd409dedd1cd9881c3b8caed81d4d8d834dbd3e870a39e69d6a1ce2dc15df8f8c34d64d292b0b99bf17bb92b263176b65634dc8f368cf7b558fef5b259964f42ec94eeac74d0abd829baa9b04f89ea3606074a4aace03e7f2217f2dec855f4028392f64f0baddd178d56d84cab16c8484633f7c5e8e7f8651ce8a7f5ce1ca3a966562a0a2a7247d105a3114ab2adbd10cfd70b2765113c42e4d1cbcb8721a719f46b25d8579a670be085d1afd2d2e541c4a27bdfffe49674798cf690f28bf4f58ec1495561be071d1c94156f15c95f3426a7409680ef8cf335a0d4786bbb0bf108589c3c8baf10f04dc7f8beda2f70117f28c771a83890d4e1c47903d642fef8a93a216576f377db3ad5be3abe7eaa924e1e7de9ac88e07177e57cad41ef5a561f6bdb741961eddee56f3d9aef9fc4e52343f2efda3a2d40b8d2b4afa735cd5655d3014d21b41eba3a8cf34a8c29cbf94dfb9a4312ee0851c6e32896b33d52d991560e3ae97c769071453a4969374aecf7423f07ee17ce71111dc925294b0130249ad00d49e26dbfddcaf214dc8356cee3a0d7486fa6144c6c826be0892ceea30e52d7b64a13b08eda3e991f37e210ccd216108c847adb58df3b2a53553689f5119e41b4f4f221474c21e197d8a13e9cf7cf23875a0dbc6ddaeca9eb6d95add2a0054528d9dc03d59bc108157291b4caa0faa53f2511bec3088a164a0768d38d7fe3281ea8459abbe3a5b5f43c697e5cbdc995832220a3c6fcf1e82bd35f4b94b13cc9312efc3ca4e3c8199e862c551fa7916e3cb0ad17ba24e456fcd2159f9e81628c508da4c2de4825100fee1c3226d1d431a740d2d8ab0c04953b073d9a20919e0f2d03e97b3f34a1644cbeeab7d0c78d898e8c523fbccc75ffc329fafe45288e21c82817e23589d0c6db453be7d3f8ecfa0ae1d24ae80c63511193c718195255a576c1b43c8d941ba22f6d9f743a7dd558e08ac2262fd67056d60e837edd3e21ce23ef27d48c9180d845ce8c5d6d4c488fcf55af99aef02f3bdf6a07833660628a672b878f8b15427189e49bf2d9e0e7e63e3c5632d9ef1e412e9bbd732b616a353097e90494a098df6a21729f1d3658f91b1bde4aaaae58530ab0e402fc10eb0910c07cace1afd0aacb579690e6dcbc184025e4160cf4de3e47106339046724d098b5b7b92f5a2bb33c11f86d4
    routing_info[2] (encrypted) = 0xe22ca641baa077154733abc586fae578ea0ed4c1851d0554235171e45e1e2a1868352b14951972fbeec9ffa500e74d24fd6d3b925c191c3454b97f518d68585db436277ec6404d85da476af5cfbfe7a392d4b8c931b57629c04ff70c08e51fa8d73e8ef55fd23b09430d4807dbdc4a7419c97b2e46d0568ca77526944e7c6a615475faea10de1025016f6fcb6f685ebdaf55c7908fd926f531d27e206307f7c9086355a351298fd6ed1f05d8724a6b159f52ed4cb988cde07f1d0eb0384803511b480ed31a1ecb7320bbd98da168ae64de73347d59df8ed0588aba661dc2d0bba2bc74ac0d2c479f466ee73def4a32cd806b0966b8f535e742c6a907af0f3dad003016db95194d381109c53499efcf4d868677c3a6dc4a184eccc2198aec260673a801814e60405670f557e618bb3b5df1e51cc90d995d0832b307c102b3fd1d083f52794ccb5a185885280dbd9e36ce64ddf320b6a7e340180e4f525c682bcd1f9cfa3ae5bbd8cdbd83e3d291e98606a7890bd8abbcf57aea1492fc1d6c876cad86446fc85b2db47ff2c6ea384fdda8bc53e81152687a702318c91f657253403612393ec7f864ddf5e807497cb88816f736f08a593d8d1e2c46c13675de2b2aa8c383b8610a5c772bbcf831e426f750c6358e623d9938890aa1d70297c4e4d238b14967f48da074ca469212336a77a2fe0fc80dc2ecd59a2ca389440cb2c1f9835786ad7e1f5bf3ceb27c8abe10178bffcbd888a9b31b9b9a5e308cefe00bdc25c82597dc22ff814b4dcb717be6ec6d87cf25dbb9fde6726461d504d4b9707d708238415ca3dc6e86d3738b902ff4a449597fcac06757fd97e0fe52c1addf7b34731366e2a5694883c60f4c9676ea6167a9fbf8b7d64932676e11fd38f7338e8954236ae31dbd7e2732d74a16b4d0809229ce44e696ceb383c41375ea0e40686f4e0c3967fa22a9e8a2ebb637816c8d1875edf86b1a4998b8525708a027a57852e28e7e84ab3d0db6b73b7e6775604d317139bc0146fedf45a17bc8b3559d4921cb87c51a21f374e040da55e013f4748b22e58f3a53f8ba733bf823eec2f2e8250d3584bd0719bdc36c1f411bfc3d6baf0db966ff2f60dc348da8ed5d011634677ff04c705b6a6942eaffbea4dc0ff5451083f5f7117df044b868df8ae2291ee68998f4c159e885ebcd1073f751899557546e5d8ddbee87a5747e5fa3f5610adf7dece55a339276e7efc3d9a152f222dd3f0452ee4ec56af7a5c1f3b9130ccf1c689e8e1401e8b885ca67349b69526ff523f217c3b36c643a9c46c46b0dc2d0f1d80c83435f752740ee7e423a59badfd5981706ec62d38ce07295ef4fbe23a4ab6cf85a2289e09cc5ad0bae981d3f42e9c6c85eeaff1f257e8ab99c2e93754e88ccd23fe3ad9c78be182b3944b79877aa1eced48b8fcf1f51c5cd01c4b2b111c97f0338e0efccc61e728e784c51d50371f453d926b02fae5c46e118a2f23a28d6f0830ec04b736ed84cf09ea1b0e228d13d7c8794ae6f363d100538a9baa9fbe533a909717dcce4c012d7f258aaee4b41c2e3f1bfe44a652ba8da51dc67164c43112805d474372b9076f3f3f0e7af94604f4fcddd2136dd7dc80ce3ff845924462cf52f5818aebf3b64f38f98edf8cb9c460477a2f94b891573929c4b51deafe6db81bc30680f7226a68567588f195ce96a791e28204b9b5844c28a61736ac20722fe156175210c7b9b6e1804c89c0a7ee136597b5b3de5d54be23671bc9477805ba10d03afb0715782845d2ab45df012f6644207cc5fa4739aa3eaf6bf84e790128aa08aede33bf30c6be2b264b33fac566209
    hmac_data[2] = 0xe22ca641baa077154733abc586fae578ea0ed4c1851d0554235171e45e1e2a1868352b14951972fbeec9ffa500e74d24fd6d3b925c191c3454b97f518d68585db436277ec6404d85da476af5cfbfe7a392d4b8c931b57629c04ff70c08e51fa8d73e8ef55fd23b09430d4807dbdc4a7419c97b2e46d0568ca77526944e7c6a615475faea10de1025016f6fcb6f685ebdaf55c7908fd926f531d27e206307f7c9086355a351298fd6ed1f05d8724a6b159f52ed4cb988cde07f1d0eb0384803511b480ed31a1ecb7320bbd98da168ae64de73347d59df8ed0588aba661dc2d0bba2bc74ac0d2c479f466ee73def4a32cd806b0966b8f535e742c6a907af0f3dad003016db95194d381109c53499efcf4d868677c3a6dc4a184eccc2198aec260673a801814e60405670f557e618bb3b5df1e51cc90d995d0832b307c102b3fd1d083f52794ccb5a185885280dbd9e36ce64ddf320b6a7e340180e4f525c682bcd1f9cfa3ae5bbd8cdbd83e3d291e98606a7890bd8abbcf57aea1492fc1d6c876cad86446fc85b2db47ff2c6ea384fdda8bc53e81152687a702318c91f657253403612393ec7f864ddf5e807497cb88816f736f08a593d8d1e2c46c13675de2b2aa8c383b8610a5c772bbcf831e426f750c6358e623d9938890aa1d70297c4e4d238b14967f48da074ca469212336a77a2fe0fc80dc2ecd59a2ca389440cb2c1f9835786ad7e1f5bf3ceb27c8abe10178bffcbd888a9b31b9b9a5e308cefe00bdc25c82597dc22ff814b4dcb717be6ec6d87cf25dbb9fde6726461d504d4b9707d708238415ca3dc6e86d3738b902ff4a449597fcac06757fd97e0fe52c1addf7b34731366e2a5694883c60f4c9676ea6167a9fbf8b7d64932676e11fd38f7338e8954236ae31dbd7e2732d74a16b4d0809229ce44e696ceb383c41375ea0e40686f4e0c3967fa22a9e8a2ebb637816c8d1875edf86b1a4998b8525708a027a57852e28e7e84ab3d0db6b73b7e6775604d317139bc0146fedf45a17bc8b3559d4921cb87c51a21f374e040da55e013f4748b22e58f3a53f8ba733bf823eec2f2e8250d3584bd0719bdc36c1f411bfc3d6baf0db966ff2f60dc348da8ed5d011634677ff04c705b6a6942eaffbea4dc0ff5451083f5f7117df044b868df8ae2291ee68998f4c159e885ebcd1073f751899557546e5d8ddbee87a5747e5fa3f5610adf7dece55a339276e7efc3d9a152f222dd3f0452ee4ec56af7a5c1f3b9130ccf1c689e8e1401e8b885ca67349b69526ff523f217c3b36c643a9c46c46b0dc2d0f1d80c83435f752740ee7e423a59badfd5981706ec62d38ce07295ef4fbe23a4ab6cf85a2289e09cc5ad0bae981d3f42e9c6c85eeaff1f257e8ab99c2e93754e88ccd23fe3ad9c78be182b3944b79877aa1eced48b8fcf1f51c5cd01c4b2b111c97f0338e0efccc61e728e784c51d50371f453d926b02fae5c46e118a2f23a28d6f0830ec04b736ed84cf09ea1b0e228d13d7c8794ae6f363d100538a9baa9fbe533a909717dcce4c012d7f258aaee4b41c2e3f1bfe44a652ba8da51dc67164c43112805d474372b9076f3f3f0e7af94604f4fcddd2136dd7dc80ce3ff845924462cf52f5818aebf3b64f38f98edf8cb9c460477a2f94b891573929c4b51deafe6db81bc30680f7226a68567588f195ce96a791e28204b9b5844c28a61736ac20722fe156175210c7b9b6e1804c89c0a7ee136597b5b3de5d54be23671bc9477805ba10d03afb0715782845d2ab45df012f6644207cc5fa4739aa3eaf6bf84e790128aa08aede33bf30c6be2b264b33fac5662094242424242424242424242424242424242424242424242424242424242424242
    hmac[2] = 0x28430b210c0af631ef80dc8594c08557ce4626bdd3593314624a588cc083a1d9

    rhokey[1] = 0x450ffcabc6449094918ebe13d4f03e433d20a3d28a768203337bc40b6e4b2c59
    mukey[1] = 0x05ed2b4a3fb023c2ff5dd6ed4b9b6ea7383f5cfe9d59c11d121ec2c81ca2eea9
    routing_info[1] (unencrypted) = 0x00010101010101010100000001000000010000000000000000000000000000000028430b210c0af631ef80dc8594c08557ce4626bdd3593314624a588cc083a1d9e22ca641baa077154733abc586fae578ea0ed4c1851d0554235171e45e1e2a1868352b14951972fbeec9ffa500e74d24fd6d3b925c191c3454b97f518d68585db436277ec6404d85da476af5cfbfe7a392d4b8c931b57629c04ff70c08e51fa8d73e8ef55fd23b09430d4807dbdc4a7419c97b2e46d0568ca77526944e7c6a615475faea10de1025016f6fcb6f685ebdaf55c7908fd926f531d27e206307f7c9086355a351298fd6ed1f05d8724a6b159f52ed4cb988cde07f1d0eb0384803511b480ed31a1ecb7320bbd98da168ae64de73347d59df8ed0588aba661dc2d0bba2bc74ac0d2c479f466ee73def4a32cd806b0966b8f535e742c6a907af0f3dad003016db95194d381109c53499efcf4d868677c3a6dc4a184eccc2198aec260673a801814e60405670f557e618bb3b5df1e51cc90d995d0832b307c102b3fd1d083f52794ccb5a185885280dbd9e36ce64ddf320b6a7e340180e4f525c682bcd1f9cfa3ae5bbd8cdbd83e3d291e98606a7890bd8abbcf57aea1492fc1d6c876cad86446fc85b2db47ff2c6ea384fdda8bc53e81152687a702318c91f657253403612393ec7f864ddf5e807497cb88816f736f08a593d8d1e2c46c13675de2b2aa8c383b8610a5c772bbcf831e426f750c6358e623d9938890aa1d70297c4e4d238b14967f48da074ca469212336a77a2fe0fc80dc2ecd59a2ca389440cb2c1f9835786ad7e1f5bf3ceb27c8abe10178bffcbd888a9b31b9b9a5e308cefe00bdc25c82597dc22ff814b4dcb717be6ec6d87cf25dbb9fde6726461d504d4b9707d708238415ca3dc6e86d3738b902ff4a449597fcac06757fd97e0fe52c1addf7b34731366e2a5694883c60f4c9676ea6167a9fbf8b7d64932676e11fd38f7338e8954236ae31dbd7e2732d74a16b4d0809229ce44e696ceb383c41375ea0e40686f4e0c3967fa22a9e8a2ebb637816c8d1875edf86b1a4998b8525708a027a57852e28e7e84ab3d0db6b73b7e6775604d317139bc0146fedf45a17bc8b3559d4921cb87c51a21f374e040da55e013f4748b22e58f3a53f8ba733bf823eec2f2e8250d3584bd0719bdc36c1f411bfc3d6baf0db966ff2f60dc348da8ed5d011634677ff04c705b6a6942eaffbea4dc0ff5451083f5f7117df044b868df8ae2291ee68998f4c159e885ebcd1073f751899557546e5d8ddbee87a5747e5fa3f5610adf7dece55a339276e7efc3d9a152f222dd3f0452ee4ec56af7a5c1f3b9130ccf1c689e8e1401e8b885ca67349b69526ff523f217c3b36c643a9c46c46b0dc2d0f1d80c83435f752740ee7e423a59badfd5981706ec62d38ce07295ef4fbe23a4ab6cf85a2289e09cc5ad0bae981d3f42e9c6c85eeaff1f257e8ab99c2e93754e88ccd23fe3ad9c78be182b3944b79877aa1eced48b8fcf1f51c5cd01c4b2b111c97f0338e0efccc61e728e784c51d50371f453d926b02fae5c46e118a2f23a28d6f0830ec04b736ed84cf09ea1b0e228d13d7c8794ae6f363d100538a9baa9fbe533a909717dcce4c012d7f258aaee4b41c2e3f1bfe44a652ba8da51dc67164c43112805d474372b9076f3f3f0e7af94604f4fcddd2136dd7dc80ce3ff845924462cf52f5818aebf3b64f38f98edf8cb9c460477a2f94b891573929c4b51deafe6db81bc30680f7226a68567588f195ce96a791e28204b9b5844c28a61736ac20722fe156175210c7b9b6e1804c89c0a7ee136
    routing_info[1] (encrypted) = 0x03445185327b8cbf5c5bfa27f925f3a9af4f431f6f7a16ad786704887cbd85bd9396df49c699c185373e4e7ac62eaf2d709eff47aa3705a235186ca412925b98902e464b80d7772900ad5040e65174840ee298625c77768d6199fa32154615bb0bd89aa137d124cb1ffcc83b85bb15ec01132794fa24cd81f94e30cc9f3de57ab16f00b29fe5456be97ea49b4b0b4601a96b8432e111ca36ae7d94c9de13ecda4ed4b7bd20a156a0697d11943871402bb1c32a312fa4f0f6ee1a0e2eb87c1fcb64d57985a7e9c07d9fba335719c04ec18607918874c513ad4c6426bd0181a3233ba12283b03be24cb8e2a530a8cb92b9bed43889267aab86d69d3d72f734e6768324125c75b25988ac2aab93da5939e2059fc0d5479d7c404b6bd1e5a9e995e47cb6966bb063377aedd689052f0a72c6576548f7264989ccd9838068b56cf7e3620f88627276674be6376007208afb027234b9e2a0267a51141ffa8ae1f3da72a8604417e63cffc3d5c4263c7e96c7ef2baa91dcf629efb87a088550b6165686c75ff9f60f24d79b5e560ceb38e987f0371cd002c1ebd09cfe3bfa1191ea1e4b23886d20a64a4526584d7beb2db9c51c3895d84cc9ecb15bff52cc64320c1f79f7d3659e2ee54a4aeed1cebb412eacd782f83ae2288c623e1a1256370262fd1afbb1a61d5e323d5bd679690105b597f027141d035a0f6ec4ef468d9a3ab39b9d54f1817878b745567f181b6678b9900299c11f21c92618da90cb10b2ecbef3d07a0b78cd35ddcc11d02e2e859f9296687268979557b99a9eaffd0875bdce1c03b1f507537916d0aba5aa437c3611ddf9e6975fc82545b67344a4f02125b08a27873079e4490519ec839a105638728d6b2dffa8024b6c4eb08dd793fbd7e53aac0e327401e1de38f826d7ac66beea3925232b0c7d086095a474b1c4f5dbb5651f5de1d1699b90bb500202673d867d00654244ece624c820d7a84db2c5e9515ace71158e45d52790a47b3b7c33ffda69b04dcf107a61a5bb0e6ab6bf06d6b44552d823f2b5969ccbe2f3986cb7180bf7acad66684117c391ff428a2023d7d29063766bb9fbb2c276c3067fab08c7fcf18fa3741b2bd409a58a4fc67d9cffde1faeb39dd86b21fa9b7930c2c6b8d28e4aeea263b3bec2bbc95b939c6ad208689204b83ea8eec2e90a42c0fde1ca8de5aaacd8a614328f83480b4a5bb5e12546ba0e56aef9e88d813dcc7c83f905c91bc5853b7d734a160f3e0efc10237aae97767abea0181c48861f2fabe52a38bcd7859d25ac5dc80eb457615d5064d77a81516b97f90c2feab7861dd9728d51a05247ef5defa05d77486812bd909195ef11c772b374d6cbe18a218af26aa19c72de077307567fa90b18eb1bc77a357ab09cc637b93d5d55793ea075285d60b04ca48bcdd45a32f55932358687db09440ff4ba255fa72e4dacee47e9029bfc16af51e121efdeb189b31dd5be103ef9dd2b5eac641768a81fce1056e6416cb8fc53b8fe946f8e37ab1f94520c3fb0d01dc15207cdec370b0fe76e5234343db30b2f8e0afdc50f1d52d761fbcd3dfcca053f01cfd3c43763546f9d5fc66e1adba7c9d66af0f61d27622f12372c4a75af5861151dcd2da571c7e90c7bde4dae9aea645f85c0781e4472e2b68d63d6789e3dc7ccb543ec47d68d1fa700c0081908a8d2e4687b6e826f4254bc169921b7e02643f3faa0264c7ff77a52205a9388d0a98c0394dc4dee81989115ade30d309374fea8435815418038534d12e4ffe88b91406a71d89d5a083e3b8224d86b2be11be32169afb04b9ea997854be9085472c342ef5fca19bf5479
    hmac_data[1] = 0x03445185327b8cbf5c5bfa27f925f3a9af4f431f6f7a16ad786704887cbd85bd9396df49c699c185373e4e7ac62eaf2d709eff47aa3705a235186ca412925b98902e464b80d7772900ad5040e65174840ee298625c77768d6199fa32154615bb0bd89aa137d124cb1ffcc83b85bb15ec01132794fa24cd81f94e30cc9f3de57ab16f00b29fe5456be97ea49b4b0b4601a96b8432e111ca36ae7d94c9de13ecda4ed4b7bd20a156a0697d11943871402bb1c32a312fa4f0f6ee1a0e2eb87c1fcb64d57985a7e9c07d9fba335719c04ec18607918874c513ad4c6426bd0181a3233ba12283b03be24cb8e2a530a8cb92b9bed43889267aab86d69d3d72f734e6768324125c75b25988ac2aab93da5939e2059fc0d5479d7c404b6bd1e5a9e995e47cb6966bb063377aedd689052f0a72c6576548f7264989ccd9838068b56cf7e3620f88627276674be6376007208afb027234b9e2a0267a51141ffa8ae1f3da72a8604417e63cffc3d5c4263c7e96c7ef2baa91dcf629efb87a088550b6165686c75ff9f60f24d79b5e560ceb38e987f0371cd002c1ebd09cfe3bfa1191ea1e4b23886d20a64a4526584d7beb2db9c51c3895d84cc9ecb15bff52cc64320c1f79f7d3659e2ee54a4aeed1cebb412eacd782f83ae2288c623e1a1256370262fd1afbb1a61d5e323d5bd679690105b597f027141d035a0f6ec4ef468d9a3ab39b9d54f1817878b745567f181b6678b9900299c11f21c92618da90cb10b2ecbef3d07a0b78cd35ddcc11d02e2e859f9296687268979557b99a9eaffd0875bdce1c03b1f507537916d0aba5aa437c3611ddf9e6975fc82545b67344a4f02125b08a27873079e4490519ec839a105638728d6b2dffa8024b6c4eb08dd793fbd7e53aac0e327401e1de38f826d7ac66beea3925232b0c7d086095a474b1c4f5dbb5651f5de1d1699b90bb500202673d867d00654244ece624c820d7a84db2c5e9515ace71158e45d52790a47b3b7c33ffda69b04dcf107a61a5bb0e6ab6bf06d6b44552d823f2b5969ccbe2f3986cb7180bf7acad66684117c391ff428a2023d7d29063766bb9fbb2c276c3067fab08c7fcf18fa3741b2bd409a58a4fc67d9cffde1faeb39dd86b21fa9b7930c2c6b8d28e4aeea263b3bec2bbc95b939c6ad208689204b83ea8eec2e90a42c0fde1ca8de5aaacd8a614328f83480b4a5bb5e12546ba0e56aef9e88d813dcc7c83f905c91bc5853b7d734a160f3e0efc10237aae97767abea0181c48861f2fabe52a38bcd7859d25ac5dc80eb457615d5064d77a81516b97f90c2feab7861dd9728d51a05247ef5defa05d77486812bd909195ef11c772b374d6cbe18a218af26aa19c72de077307567fa90b18eb1bc77a357ab09cc637b93d5d55793ea075285d60b04ca48bcdd45a32f55932358687db09440ff4ba255fa72e4dacee47e9029bfc16af51e121efdeb189b31dd5be103ef9dd2b5eac641768a81fce1056e6416cb8fc53b8fe946f8e37ab1f94520c3fb0d01dc15207cdec370b0fe76e5234343db30b2f8e0afdc50f1d52d761fbcd3dfcca053f01cfd3c43763546f9d5fc66e1adba7c9d66af0f61d27622f12372c4a75af5861151dcd2da571c7e90c7bde4dae9aea645f85c0781e4472e2b68d63d6789e3dc7ccb543ec47d68d1fa700c0081908a8d2e4687b6e826f4254bc169921b7e02643f3faa0264c7ff77a52205a9388d0a98c0394dc4dee81989115ade30d309374fea8435815418038534d12e4ffe88b91406a71d89d5a083e3b8224d86b2be11be32169afb04b9ea997854be9085472c342ef5fca19bf54794242424242424242424242424242424242424242424242424242424242424242
    hmac[1] = 0x0daed5f832ef34ea8d0d2cc0699134287a2739c77152d9edc8fe5ccce7ec838f

    rhokey[0] = 0xce496ec94def95aadd4bec15cdb41a740c9f2b62347c4917325fcc6fb0453986
    mukey[0] = 0xb57061dc6d0a2b9f261ac410c8b26d64ac5506cbba30267a649c28c179400eba
    routing_info[0] (unencrypted) = 0x0000000000000000000000000000000000000000000000000000000000000000002bdc5227c8eb8ba5fcfc15cfc2aa578ff208c106646d0652cd289c0a37e445bb03445185327b8cbf5c5bfa27f925f3a9af4f431f6f7a16ad786704887cbd85bd9396df49c699c185373e4e7ac62eaf2d709eff47aa3705a235186ca412925b98902e464b80d7772900ad5040e65174840ee298625c77768d6199fa32154615bb0bd89aa137d124cb1ffcc83b85bb15ec01132794fa24cd81f94e30cc9f3de57ab16f00b29fe5456be97ea49b4b0b4601a96b8432e111ca36ae7d94c9de13ecda4ed4b7bd20a156a0697d11943871402bb1c32a312fa4f0f6ee1a0e2eb87c1fcb64d57985a7e9c07d9fba335719c04ec18607918874c513ad4c6426bd0181a3233ba12283b03be24cb8e2a530a8cb92b9bed43889267aab86d69d3d72f734e6768324125c75b25988ac2aab93da5939e2059fc0d5479d7c404b6bd1e5a9e995e47cb6966bb063377aedd689052f0a72c6576548f7264989ccd9838068b56cf7e3620f88627276674be6376007208afb027234b9e2a0267a51141ffa8ae1f3da72a8604417e63cffc3d5c4263c7e96c7ef2baa91dcf629efb87a088550b6165686c75ff9f60f24d79b5e560ceb38e987f0371cd002c1ebd09cfe3bfa1191ea1e4b23886d20a64a4526584d7beb2db9c51c3895d84cc9ecb15bff52cc64320c1f79f7d3659e2ee54a4aeed1cebb412eacd782f83ae2288c623e1a1256370262fd1afbb1a61d5e323d5bd679690105b597f027141d035a0f6ec4ef468d9a3ab39b9d54f1817878b745567f181b6678b9900299c11f21c92618da90cb10b2ecbef3d07a0b78cd35ddcc11d02e2e859f9296687268979557b99a9eaffd0875bdce1c03b1f507537916d0aba5aa437c3611ddf9e6975fc82545b67344a4f02125b08a27873079e4490519ec839a105638728d6b2dffa8024b6c4eb08dd793fbd7e53aac0e327401e1de38f826d7ac66beea3925232b0c7d086095a474b1c4f5dbb5651f5de1d1699b90bb500202673d867d00654244ece624c820d7a84db2c5e9515ace71158e45d52790a47b3b7c33ffda69b04dcf107a61a5bb0e6ab6bf06d6b44552d823f2b5969ccbe2f3986cb7180bf7acad66684117c391ff428a2023d7d29063766bb9fbb2c276c3067fab08c7fcf18fa3741b2bd409a58a4fc67d9cffde1faeb39dd86b21fa9b7930c2c6b8d28e4aeea263b3bec2bbc95b939c6ad208689204b83ea8eec2e90a42c0fde1ca8de5aaacd8a614328f83480b4a5bb5e12546ba0e56aef9e88d813dcc7c83f905c91bc5853b7d734a160f3e0efc10237aae97767abea0181c48861f2fabe52a38bcd7859d25ac5dc80eb457615d5064d77a81516b97f90c2feab7861dd9728d51a05247ef5defa05d77486812bd909195ef11c772b374d6cbe18a218af26aa19c72de077307567fa90b18eb1bc77a357ab09cc637b93d5d55793ea075285d60b04ca48bcdd45a32f55932358687db09440ff4ba255fa72e4dacee47e9029bfc16af51e121efdeb189b31dd5be103ef9dd2b5eac641768a81fce1056e6416cb8fc53b8fe946f8e37ab1f94520c3fb0d01dc15207cdec370b0fe76e5234343db30b2f8e0afdc50f1d52d761fbcd3dfcca053f01cfd3c43763546f9d5fc66e1adba7c9d66af0f61d27622f12372c4a75af5861151dcd2da571c7e90c7bde4dae9aea645f85c0781e4472e2b68d63d6789e3dc7ccb543ec47d68d1fa700c0081908a8d2e4687b6e826f4254bc169921b7e02643f3faa0264c7ff77a52205a9388d0a98c0394dc4dee81
    routing_info[0] (encrypted) = 0xe5f14350c2a76fc232b5e46d421e9615471ab9e0bc887beff8c95fdb878f7b3a716a996c7845c93d90e4ecbb9bde4ece2f69425c99e4bc820e44485455f135edc0d10f7d61ab590531cf08000179a333a347f8b4072f216400406bdf3bf038659793d4a1fd7b246979e3150a0a4cb052c9ec69acf0f48c3d39cd55675fe717cb7d80ce721caad69320c3a469a202f1e468c67eaf7a7cd8226d0fd32f7b48084dca885d56047694762b67021713ca673929c163ec36e04e40ca8e1c6d17569419d3039d9a1ec866abe044a9ad635778b961fc0776dc832b3a451bd5d35072d2269cf9b040f6b7a7dad84fb114ed413b1426cb96ceaf83825665ed5a1d002c1687f92465b49ed4c7f0218ff8c6c7dd7221d589c65b3b9aaa71a41484b122846c7c7b57e02e679ea8469b70e14fe4f70fee4d87b910cf144be6fe48eef24da475c0b0bcc6565ae82cd3f4e3b24c76eaa5616c6111343306ab35c1fe5ca4a77c0e314ed7dba39d6f1e0de791719c241a939cc493bea2bae1c1e932679ea94d29084278513c77b899cc98059d06a27d171b0dbdf6bee13ddc4fc17a0c4d2827d488436b57baa167544138ca2e64a11b43ac8a06cd0c2fba2d4d900ed2d9205305e2d7383cc98dacb078133de5f6fb6bed2ef26ba92cea28aafc3b9948dd9ae5559e8bd6920b8cea462aa445ca6a95e0e7ba52961b181c79e73bd581821df2b10173727a810c92b83b5ba4a0403eb710d2ca10689a35bec6c3a708e9e92f7d78ff3c5d9989574b00c6736f84c199256e76e19e78f0c98a9d580b4a658c84fc8f2096c2fbea8f5f8c59d0fdacb3be2802ef802abbecb3aba4acaac69a0e965abd8981e9896b1f6ef9d60f7a164b371af869fd0e48073742825e9434fc54da837e120266d53302954843538ea7c6c3dbfb4ff3b2fdbe244437f2a153ccf7bdb4c92aa08102d4f3cff2ae5ef86fab4653595e6a5837fa2f3e29f27a9cde5966843fb847a4a61f1e76c281fe8bb2b0a181d096100db5a1a5ce7a910238251a43ca556712eaadea167fb4d7d75825e440f3ecd782036d7574df8bceacb397abefc5f5254d2722215c53ff54af8299aaaad642c6d72a14d27882d9bbd539e1cc7a527526ba89b8c037ad09120e98ab042d3e8652b31ae0e478516bfaf88efca9f3676ffe99d2819dcaeb7610a626695f53117665d267d3f7abebd6bbd6733f645c72c389f03855bdf1e4b8075b516569b118233a0f0971d24b83113c0b096f5216a207ca99a7cddc81c130923fe3d91e7508c9ac5f2e914ff5dccab9e558566fa14efb34ac98d878580814b94b73acbfde9072f30b881f7f0fff42d4045d1ace6322d86a97d164aa84d93a60498065cc7c20e636f5862dc81531a88c60305a2e59a985be327a6902e4bed986dbf4a0b50c217af0ea7fdf9ab37f9ea1a1aaa72f54cf40154ea9b269f1a7c09f9f43245109431a175d50e2db0132337baa0ef97eed0fcf20489da36b79a1172faccc2f7ded7c60e00694282d93359c4682135642bc81f433574aa8ef0c97b4ade7ca372c5ffc23c7eddd839bab4e0f14d6df15c9dbeab176bec8b5701cf054eb3072f6dadc98f88819042bf10c407516ee58bce33fbe3b3d86a54255e577db4598e30a135361528c101683a5fcde7e8ba53f3456254be8f45fe3a56120ae96ea3773631fcb3873aa3abd91bcff00bd38bd43697a2e789e00da6077482e7b1b1a677b5afae4c54e6cbdf7377b694eb7d7a5b913476a5be923322d3de06060fd5e819635232a2cf4f0731da13b8546d1d6d4f8d75b9fce6c2341a71b0ea6f780df54bfdb0dd5cd9855179f602f9172
    hmac_data[0] = 0xe5f14350c2a76fc232b5e46d421e9615471ab9e0bc887beff8c95fdb878f7b3a716a996c7845c93d90e4ecbb9bde4ece2f69425c99e4bc820e44485455f135edc0d10f7d61ab590531cf08000179a333a347f8b4072f216400406bdf3bf038659793d4a1fd7b246979e3150a0a4cb052c9ec69acf0f48c3d39cd55675fe717cb7d80ce721caad69320c3a469a202f1e468c67eaf7a7cd8226d0fd32f7b48084dca885d56047694762b67021713ca673929c163ec36e04e40ca8e1c6d17569419d3039d9a1ec866abe044a9ad635778b961fc0776dc832b3a451bd5d35072d2269cf9b040f6b7a7dad84fb114ed413b1426cb96ceaf83825665ed5a1d002c1687f92465b49ed4c7f0218ff8c6c7dd7221d589c65b3b9aaa71a41484b122846c7c7b57e02e679ea8469b70e14fe4f70fee4d87b910cf144be6fe48eef24da475c0b0bcc6565ae82cd3f4e3b24c76eaa5616c6111343306ab35c1fe5ca4a77c0e314ed7dba39d6f1e0de791719c241a939cc493bea2bae1c1e932679ea94d29084278513c77b899cc98059d06a27d171b0dbdf6bee13ddc4fc17a0c4d2827d488436b57baa167544138ca2e64a11b43ac8a06cd0c2fba2d4d900ed2d9205305e2d7383cc98dacb078133de5f6fb6bed2ef26ba92cea28aafc3b9948dd9ae5559e8bd6920b8cea462aa445ca6a95e0e7ba52961b181c79e73bd581821df2b10173727a810c92b83b5ba4a0403eb710d2ca10689a35bec6c3a708e9e92f7d78ff3c5d9989574b00c6736f84c199256e76e19e78f0c98a9d580b4a658c84fc8f2096c2fbea8f5f8c59d0fdacb3be2802ef802abbecb3aba4acaac69a0e965abd8981e9896b1f6ef9d60f7a164b371af869fd0e48073742825e9434fc54da837e120266d53302954843538ea7c6c3dbfb4ff3b2fdbe244437f2a153ccf7bdb4c92aa08102d4f3cff2ae5ef86fab4653595e6a5837fa2f3e29f27a9cde5966843fb847a4a61f1e76c281fe8bb2b0a181d096100db5a1a5ce7a910238251a43ca556712eaadea167fb4d7d75825e440f3ecd782036d7574df8bceacb397abefc5f5254d2722215c53ff54af8299aaaad642c6d72a14d27882d9bbd539e1cc7a527526ba89b8c037ad09120e98ab042d3e8652b31ae0e478516bfaf88efca9f3676ffe99d2819dcaeb7610a626695f53117665d267d3f7abebd6bbd6733f645c72c389f03855bdf1e4b8075b516569b118233a0f0971d24b83113c0b096f5216a207ca99a7cddc81c130923fe3d91e7508c9ac5f2e914ff5dccab9e558566fa14efb34ac98d878580814b94b73acbfde9072f30b881f7f0fff42d4045d1ace6322d86a97d164aa84d93a60498065cc7c20e636f5862dc81531a88c60305a2e59a985be327a6902e4bed986dbf4a0b50c217af0ea7fdf9ab37f9ea1a1aaa72f54cf40154ea9b269f1a7c09f9f43245109431a175d50e2db0132337baa0ef97eed0fcf20489da36b79a1172faccc2f7ded7c60e00694282d93359c4682135642bc81f433574aa8ef0c97b4ade7ca372c5ffc23c7eddd839bab4e0f14d6df15c9dbeab176bec8b5701cf054eb3072f6dadc98f88819042bf10c407516ee58bce33fbe3b3d86a54255e577db4598e30a135361528c101683a5fcde7e8ba53f3456254be8f45fe3a56120ae96ea3773631fcb3873aa3abd91bcff00bd38bd43697a2e789e00da6077482e7b1b1a677b5afae4c54e6cbdf7377b694eb7d7a5b913476a5be923322d3de06060fd5e819635232a2cf4f0731da13b8546d1d6d4f8d75b9fce6c2341a71b0ea6f780df54bfdb0dd5cd9855179f602f91724242424242424242424242424242424242424242424242424242424242424242
    hmac[0] = 0x62cc962876e734e089e79eda497077fb411fac5f36afd43329040ecd1e16c6d9

### 最终数据包

    onionpacket = 0x0002eec7245d6b7d2ccb30380bfbe2a3648cd7a942653f5aa340edcea1f283686619e5f14350c2a76fc232b5e46d421e9615471ab9e0bc887beff8c95fdb878f7b3a71da571226458c510bbadd1276f045c21c520a07d35da256ef75b4367962437b0dd10f7d61ab590531cf08000178a333a347f8b4072e216400406bdf3bf038659793a86cae5f52d32f3438527b47a1cfc54285a8afec3a4c9f3323db0c946f5d4cb2ce721caad69320c3a469a202f3e468c67eaf7a7cda226d0fd32f7b48084dca885d15222e60826d5d971f64172d98e0760154400958f00e86697aa1aa9d41bee8119a1ec866abe044a9ad635778ba61fc0776dc832b39451bd5d35072d2269cf9b040d6ba38b54ec35f81d7fc67678c3be47274f3c4cc472aff005c3469eb3bc140769ed4c7f0218ff8c6c7dd7221d189c65b3b9aaa71a01484b122846c7c7b57e02e679ea8469b70e14fe4f70fee4d87b910cf144be6fe48eef24da475c0b0bcc6565ae82cd3f4e3b24c76eaa5616c6111343306ab35c1fe5ca4a77c0e314ed7dba39d6f1e0de791719c241a939cc493bea2bae1c1e932679ea94d29084278513c77b899cc98059d06a27d171b0dbdf6bee13ddc4fc17a0c4d2827d488436b57baa167544138ca2e64a11b43ac8a06cd0c2fba2d4d900ed2d9205305e2d7383cc98dacb078133de5 f6fb6bed2ef26ba92cea28aafc3b9948dd9ae5559e8bd6920b8cea462aa445ca6a95e0e7ba52961b181c79e73bd581821df2b10173727a810c92b83b5ba4a0403eb710d2ca10689a35bec6c3a708e9e92f7d78ff3c5d9989574b00c6736f84c199256e76e19e78f0c98a9d580b4a658c84fc8f2096c2fbea8f5f8c59d0fdacb3be2802ef802abbecb3aba4acaac69a0e965abd8981e9896b1f6ef9d60f7a164b371af869fd0e48073742825e9434fc54da837e120266d53302954843538ea7c6c3dbfb4ff3b2fdbe244437f2a153ccf7bdb4c92aa08102d4f3cff2ae5ef86fab4653595e6a5837fa2f3e29f27a9cde5966843fb847a4a61f1e76c281fe8bb2b0a181d096100db5a1a5ce7a910238251a43ca556712eaadea167fb4d7d75825e440f3ecd782036d7574df8bceacb397abefc5f5254d2722215c53ff54af8299aaaad642c6d72a14d27882d9bbd539e1cc7a527526ba89b8c037ad09120e98ab042d3e8652b31ae0e478516bfaf88efca9f3676ffe99d2819dcaeb7610a626695f53117665d267d3f7abebd6bbd6733f645c72c389f03855bdf1e4b8075b516569b118233a0f0971d24b83113c0b096f5216a207ca99a7cddc81c130923fe3d91e7508c9ac5f2e914ff5dccab9e558566fa14efb34ac98d878580814b94b73acbfde9072f30b881f7f0fff42d4045d1ace6322d86a 97d164aa84d93a60498065cc7c20e636f5862dc81531a88c60305a2e59a985be327a6902e4bed986dbf4a0b50c217af0ea7fdf9ab37f9ea1a1aaa72f54cf40154ea9b269f1a7c09f9f43245109431a175d50e2db0132337baa0ef97eed0fcf20489da36b79a1172faccc2f7ded7c60e00694282d93359c4682135642bc81f433574aa8ef0c97b4ade7ca372c5ffc23c7eddd839bab4e0f14d6df15c9dbeab176bec8b5701cf054eb3072f6dadc98f88819042bf10c407516ee58bce33fbe3b3d86a54255e577db4598e30a135361528c101683a5fcde7e8ba53f3456254be8f45fe3a56120ae96ea3773631fcb3873aa3abd91bcff00bd38bd43697a2e789e00da6077482e7b1b1a677b5afae4c54e6cbdf7377b694eb7d7a5b913476a5be923322d3de06060fd5e819635232a2cf4f0731da13b8546d1d6d4f8d75b9fce6c2341a71b0ea6f780df54bfdb0dd5cd9855179f602f917265f21f9190c70217774a6fbaaa7d63ad64199f4664813b955cff954949076dcf

## 返回错误

使用与上面相同的参数（节点ID，共享秘密等）。

    #node 4返回错误
    failure_message = 2002
    #create error message
    shared_secret = b5756b9b542727dbafc6765a49488b023a725d631af688fc031217e90770c328
    有效载荷= 00022002000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    um_key = 4da7f2923edce6c2d85987d1d9fa6d88023e6c3a9c3d20f07d3b10b61a78d646
    raw_error_packet = 4c2fc8bc08510334b6833ad9c3e79cd1b52ae59dfe5c2a4b23ead50f09f7ee0b0002200200fe0000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000000
    ＃转发错误包
    shared_secret = b5756b9b542727dbafc6765a49488b023a725d631af688fc031217e90770c328
     ammag_key = 2f36bb8822e1f0d04c27b7d8bb7d7dd586e032a3218b8d414afbba6f169a4d68
    流= e9c975b07c9a374ba64fd9be3aae955e917d34d1fa33f2e90f53bbf4394713c6a8c9b16ab5f12fd45edd73c1b0c8b33002df376801ff58aaa94000bf8a86f92620f343baef38a580102395ae3abf9128d1047a0736ff9b83d456740ebbb4aeb3aa9737f18fb4afb4aa074fb26c4d702f42968888550a3bded8c05247e045b866baef0499f079fdaeef6538f31d44deafffdfd3afa2fb4ca9082b8f1c465371a9894dd8c243fb4847e004f5256b3e90e2edde4c9fb3082ddfe4d1e734cacd96ef0706bf63c9984e22dc98851bcccd1c3494351feb458c9c6af41c0044bea3c47552b1d992ae542b17a2d0bba1a096c78d169034ecb55b6e3a7263c26017f033031228833c1daefc0dedb8cf7c3e37c9c37ebfe42f3225c326e8bcfd338804c145b16e34e4
    对节点4错误分组：a5e6bd0c74cb347f10cce367f949098f2457d14c046fd8a22cb96efb30b0fdcda8cb9168b50f2fd45edd73c1b0c8b33002df376801ff58aaa94000bf8a86f92620f343baef38a580102395ae3abf9128d1047a0736ff9b83d456740ebbb4aeb3aa9737f18fb4afb4aa074fb26c4d702f42968888550a3bded8c05247e045b866baef0499f079fdaeef6538f31d44deafffdfd3afa2fb4ca9082b8f1c465371a9894dd8c243fb4847e004f5256b3e90e2edde4c9fb3082ddfe4d1e734cacd96ef0706bf63c9984e22dc98851bcccd1c3494351feb458c9c6af41c0044bea3c47552b1d992ae542b17a2d0bba1a096c78d169034ecb55b6e3a7263c26017f033031228833c1daefc0dedb8cf7c3e37c9c37ebfe42f3225c326e8bcfd338804c145b16e34e4
    ＃转发错误包
    shared_secret = 21e13c2d7cfe7e18836df50872466117a295783ab8aab0e7ecc8c725503ad02d
    ammag_key = cd9ac0e09064f039fa43a31dea05f5fe5f6443d40a98be4071af4a9d704be5ad
    流= 617ca1e4624bc3f04fece3aa5a2b615110f421ec62408d16c48ea6c1b7c33fe7084a2bd9d4652fc5068e5052bf6d0acae2176018a3d8c75f37842712913900263cff92f39f3c18aa1f4b20a93e70fc429af7b2b1967ca81a761d40582daf0eb49cef66e3d6fbca0218d3022d32e994b41c884a27c28685ef1eb14603ea80a204b2f2f474b6ad5e71c6389843e3611ebeafc62390b717ca53b3670a33c517ef28a659c251d648bf4c966a4ef187113ec9848bf110816061ca4f2f68e76ceb88bd6208376460b916fb2ddeb77a65e8f88b2e71a2cbf4ea4958041d71c17d05680c051c3676fb0dc8108e5d78fb1e2c44d79a202e9d14071d536371ad47c39a05159e8d6c41d17a1e858faaaf572623aa23a38ffc73a4114cb1ab1cd7f906c6bd4e21b29694
    为节点3错误分组：c49a1ce81680f78f5f2000cda36268de34a3f0a0662f55b4e837c83a8773c22aa081bab1616a0011585323930fa5b9fae0c85770a2279ff59ec427ad1bbff9001c0cd1497004bd2a0f68b50704cf6d6a4bf3c8b6a0833399a24b3456961ba00736785112594f65b6b2d44d9f5ea4e49b5e1ec2af978cbe31c67114440ac51a62081df0ed46d4a3df295da0b0fe25c0115019f03f15ec86fabb4c852f83449e812f141a9395b3f70b766ebbd4ec2fae2b6955bd8f32684c15abfe8fd3a6261e52650e8807a92158d9f1463261a925e4bfba44bd20b166d532f0017185c3a6ac7957adefe45559e3072c8dc35abeba835a8cb01a71a15c736911126f27d46a36168ca5ef7dccd4e2886212602b181463e0dd30185c96348f9743a02aca8ec27c0b90dca270
    转发错误包
    shared_secret = 3a6b412548762f0dbccce5c7ae7bb8147d1caf9b5471c34120b30bc9c04891cc
    ammag_key = 1bf08df8628d452141d56adfd1b25c1530d7921c23cecfc749ac03a9b694b0d3
    流= 6149f48b5a7e8f3d6f5d870b7a698e204cf64452aab4484ff1dee671fe63fd4b5f1b78ee2047dfa61e3d576b149bedaf83058f85f06a3172a3223ad6c4732d96b32955da7d2feb4140e58d86fc0f2eb5d9d1878e6f8a7f65ab9212030e8e915573ebbd7f35e1a430890be7e67c3fb4bbf2def662fa625421e7b411c29ebe81ec67b77355596b05cc155755664e59c16e21410aabe53e80404a615f44ebb31b365ca77a6e91241667b26c6cad24fb2324cf64e8b9dd6e2ce65f1f098cfd1ef41ba2d4c7def0ff165a0e7c84e7597c40e3dffe97d417c144545a0e38ee33ebaae12cc0c14650e453d46bfc48c0514f354773435ee89b7b2810606eb73262c77a1d67f3633705178d79a1078c3a01b5fadc9651feb63603d19decd3a00c1f69af2dab259593
    节点2错误分组：a5d3e8634cfe78b2307d87c6d90be6fe7855b4f2cc9b1dfb19e92e4b79103f61ff9ac25f412ddfb7466e74f81b3e545563cdd8f5524dae873de61d7bdfccd496af2584930d2b566b4f8d3881f8c043df92224f38cf094cfc09d92655989531524593ec6d6caec1863bdfaa79229b5020acc034cd6deeea1021c50586947b9b8e6faa83b81fbfa6133c0af5d6b07c017f7158fa94f0d206baf12dda6b68f785b773b360fd0497e16cc402d779c8d48d0fa6315536ef0660f3f4e1865f5b38ea49c7da4fd959de4e83ff3ab686f059a45c65ba2af4a6a79166aa0f496bf04d06987b6d2ea205bdb0d347718b9aeff5b61dfff344993a275b79717cd815b6ad4c0beb568c4ac9c36ff1c315ec1119a1993c4b61e6eaa0375e0aaf738ac691abd3263bf937e3
    ＃转发错误包
    shared_secret = a6519e98832a0b179f62123b3567c106db99ee37bef036e783263602f3488fae
    ammag_key = 59ee5867c5c151daa31e36ee42530f429c433836286e63744f2020b980302564
    流= 0f10c86f05968dd91188b998ee45dcddfbf89fe9a99aa6375c42ed5520a257e048456fe417c15219ce39d921555956ae2ff795177c63c819233f3bcb9b8b28e5ac6e33a3f9b87ca62dff43f4cc4a2755830a3b7e98c326b278e2bd31f4a9973ee99121c62873f5bfb2d159d3d48c5851e3b341f9f6634f51939188c3b9ff45feeb11160bb39ce3332168b8e744a92107db575ace7866e4b8f390f1edc4acd726ed106555900a0832575c3a7ad11bb1fe388ff32b99bcf2a0d0767a83cf293a220a983ad014d404bfa20022d8b369fe06f7ecc9c74751dcda0ff39d8bca74bf9956745ba4e5d299e0da8f68a9f660040beac03e795a046640cf8271307a8b64780b0588422f5a60ed7e36d60417562938b400802dac5f87f267204b6d5bcfd8a05b221ec2
    节点1错误分组：aac3200c4968f56b21f53e5e374e3a2383ad2b1b6501bbcc45abc31e59b26881b7dfadbb56ec8dae8857add94e6702fb4c3a4de22e2e669e1ed926b04447fc73034bb730f4932acd62727b75348a648a1128744657ca6a4e713b9b646c3ca66cac02cdab44dd3439890ef3aaf61708714f7375349b8da541b2548d452d84de7084bb95b3ac2345201d624d31f4d52078aa0fa05a88b4e20202bd2b86ac5b52919ea305a8949de95e935eed0319cf3cf19ebea61d76ba92532497fcdc9411d06bcd4275094d0a4a3c5d3a945e43305a5a9256e333e1f64dbca5fcd4e03a39b9012d197506e06f29339dfee3331995b21615337ae060233d39befea925cc262873e0530408e6990f1cbd233a150ef7b004ff6166c70c68d9f8c853c1abca640b8660db2921
    ＃转发错误包
    shared_secret = 53eb63ea8a3fec3b3cd433b85cd62a4b145e1dda09391b348c4e1cd36a03ea66
    ammag_key = 3761ba4d3e726d8abb16cba5950ee976b84937b61b7ad09e741724d7dee12eb5
    流= 3699fd352a948a05f604763c0bca2968d5eaca2b0118602e52e59121f050936c8dd90c24df7dc8cf8f1665e39a6c75e9e2c0900ea245c9ed3b0008148e0ae18bbfaea0c711d67eade980c6f5452e91a06b070bbde68b5494a92575c114660fb53cf04bf686e67ffa4a0f5ae41a59a39a8515cb686db553d25e71e7a97cc2febcac55df2711b6209c502b2f8827b13d3ad2f491c45a0cafe7b4d8d8810e805dee25d676ce92e0619b9c206f922132d806138713a8f69589c18c3fdc5acee41c1234b17ecab96b8c56a46787bba2c062468a13919afc18513835b472a79b2c35f9a91f38eb3b9e998b1000cc4a0dbd62ac1a5cc8102e373526d7e8f3c3a1b4bfb2f8a3947fe350cb89f73aa1bb054edfa9895c0fc971c2b5056dc8665902b51fced6dff80c
    对节点0错误分组：9c5add3963fc7f6ed7f148623c84134b5647e1306419dbe2174e523fa9e2fbed3a06a19f899145610741c83ad40b7712aefaddec8c6baf7325d92ea4ca4d1df8bce517f7e54554608bf2bd8071a4f52a7a2f7ffbb1413edad81eeea5785aa9d990f2865dc23b4bc3c301a94eec4eabebca66be5cf638f693ec256aec514620cc28ee4a94bd9565bc4d4962b9d3641d4278fb319ed2b84de5b665f307a2db0f7fbb757366067d88c50f7e829138fde4f78d39b5b5802f1b92a8a820865af5cc79f9f30bc3f461c66af95d13e5e1f0381c184572a91dee1c849048a647a1158cf884064deddbf1b0b88dfe2f791428d0ba0f6fb2f04e14081f69165ae66d9297c118f0907705c9c4954a199bae0bb96fad763d690e7daa6cfda59ba7f2c8d11448b604d12d

# 参考

# 作者

[ 整我： ]

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
