# BOLT＃11：闪电付款的发票协议

一种简单，可扩展，QR码准备好的协议，用于请求付款
在闪电。

# 目录

  * [编码概述](#encoding-overview)
  * [人类可读的部分](#human-readable-part)
  * [数据部分](#data-part)
    * [标记字段](#tagged-fields)
  * [付款人/收款人互动](#payer--payee-interactions)
    * [付款人/收款人要求](#payer--payee-requirements)
  * [履行](#implementation)
  * [例子](#examples)
  * [作者](#authors)

# 编码概述

Lightning发票的格式使用
[bech32编码]（https://github.com/bitcoin/bips/blob/master/bip-0173.mediawiki），
已经用于比特币隔离证人。有可能
简单地重复使用Lightning发票，即使它的6字符校验和已经过优化
手动输入，根据长度不太可能经常发生
闪电发票

如果需要URI方案，则当前建议是
在BOLT-11编码之前使用'lightning：'作为前缀（注意：不是
'lightning：//'），或者用于比特币支付的后备，使用'比特币：'，
按照BIP-21，键“闪电”，值等于BOLT-11
编码。

## 要求

一个作家：
   - 必须在Bech32中对付款请求进行编码（参见BIP-0173）
   - 可能超过BIP-0173中指定的90个字符的限制。

一位读者：
  - 务必将地址解析为Bech32，如BIP-0173中所指定（也没有字符限制）。
  - 如果校验和不正确：
    - 必须通过付款失败。

# 人类可读的部分

Lightning发票的人类可读部分包括两部分：
1. `prefix`：`ln` + BIP-0173货币前缀（例如比特币主网的`lnbc`，
   用于比特币testnet的`lntb`和用于比特币regtest的`lnbcrt`）
1. `amount`：该货币的可选数字，后跟可选数字
   `乘数`字母。这里编码的单位是支付单位的“社交”惯例 - 在比特币的情况下，单位是'比特币'而不是satoshis。

定义了以下`乘数'字母：

* `m`（毫）：乘以0.001
* `u`（微）：乘以0.000001
* `n`（纳米）：乘以0.000000001
* `p`（微微）：乘以0.000000000001

## 要求

一个作家：
  - 必须使用成功付款所需的货币对`prefix`进行编码。
  - 如果成功付款需要特定的最低金额：
      - 必须包括'金额'。
    - 必须将`amount`编码为正十进制整数，不带前导0。
    - 应该使用最大的表示，使用最大的表示
      乘数或省略乘数。

一位读者：
  - 如果它不理解`前缀`：
    - 必须通过付款失败。
  - 如果`amount`为空：
      - 应该向付款人表明金额未指明。
  - 除此以外：
    - 如果`amount`包含非数字OR，则后跟除了之外的任何内容
    一个“乘数”（见上表）：
      - 必须通过付款失败。
    - 如果存在`乘数'：
        - 必须将`amount`乘以`multiplier`值来得到
      付款金额。

## 合理

`amount`被编码到人类可读部分，因为它是公平的
可读和有用的指标，表明要求的数量。

捐赠地址通常没有相关的金额，所以`金额'
在这种情况下是可选的。通常需要最低付款
无论提供什么作为回报。

# 数据部分

Lightning发票的数据部分包含多个部分：

1. `timestamp`：自1970年以来的秒数（35位，大端）
1. 零个或多个标记的部分
1. `签名`：上面的比特币式签名（520位）

## 要求

一个作家：
  - 必须将`timestamp`设置为自1970年1月1日午夜（UTC）以来的秒数
  大端。
  - 必须将`signature`设置为SHA2的256位散列的有效512位secp256k1签名
  人类可读的部分，表示为UTF-8字节，与
  数据部分（不包括签名）附加0位以填充
  数据到下一个字节边界，尾随字节包含
  恢复ID（0,1,2或3）。

一位读者：
  - 必须检查`signature`是否有效（参见下面指定的`n`标记字段）。

## 合理

即使SHA2，`signature`也包含确切的字节数
标准实际上支持位边界的散列，因为它并不广泛
实现。恢复ID允许公钥恢复，所以
可以隐含收款人节点的身份。

## 标记字段

每个标记字段的形式如下：

1. `type`（5位）
1. `data_length`（10位，big-endian）
1. `data`（`data_length` x 5 bits）

当前定义的标记字段是：

* `p`（1）：`data_length` 52. 256位SHA256 payment_hash。 Preimage的这个提供付款证明。
* `d`（13）：`data_length`变量。支付目的的简短描述（UTF-8），例如'1杯咖啡'或'ナンセンス1杯'
* `n`（19）：`data_length` 53.收款人节点的33字节公钥
* `h`（23）：`data_length` 52.对支付目的的256位描述（SHA256）。这用于提交超过639字节的关联描述，但在这种情况下描述的传输机制是特定于传输的，并且此处未定义。
* `x`（6）：`data_length`变量。 `expiry`时间以秒为单位（big-endian）。如果未指定，默认值为3600（1小时）。
* `c`（24）：`data_length`变量。 `min_final_cltv_expiry`用于路由中的最后一个HTLC。如果未指定，则默认值为9。
* `f`（9）：`data_length`变量，取决于版本。后备链地址：对于比特币，它以5位“版本”开头，包含见证程序或P2PKH或P2SH地址。
* `r`（3）：`data_length`变量。一个或多个条目，包含私有路由的额外路由信息;可能有不止一个`r`字段
   * `pubkey`（264位）
   * `short_channel_id`（64位）
   * `fee_base_msat`（32位，big-endian）
   * `fee_proportional_millionths`（32位，big-endian）
   * `cltv_expiry_delta`（16位，big-endian）

### 要求

一个作家：
  - 必须包含一个'p`字段。
  - 必须将`payment_hash`设置为`payment_preimage`的SHA2 256位哈希
  这将作为回报付款。
  - 必须包括恰好一个`d`或恰好一个`h`字段。
    - 如果包括`d`：
      - 必须将`d`设置为有效的UTF-8字符串。
      - 应该使用付款目的的完整描述。
    - 如果包含`h`：
      - 必须使`h`中的散列描述的原像可用
      通过一些未指明的手段。
        - 应该使用付款目的的完整描述。
  - 可以包括一个“x”字段。
  - 可能包括一个`c`字段。
    - 必须将`c`设置为它将接受的最小`cltv_expiry`
    HTLC在路线上。
  - 应该对`x`和`c`字段使用最小的`data_length`。
  - 可以包括一个“n”字段。
    - 必须将`n`设置为用于创建`签名`的公钥。
  - 可以包括一个或多个`f`字段。
    - 比特币支付：
      - 必须将`f`字段设置为有效的见证版本和程序，或者设置为`17`
      后跟一个公钥哈希，或者是'18`后跟一个脚本哈希。
  - 如果没有与其公钥关联的公共频道：
    - 必须包括至少一个`r`字段。
      - `r`字段必须包含一个或多个有序条目，表示来自的前向路由
      到最终目的地的公共节点。
        - 注意：对于每个条目，`pubkey`是通道开头的节点ID;
        `short_channel_id`是识别频道的短信道ID字段;和
        `fee_base_msat`，`fee_proportional_millionths`和`cltv_expiry_delta`是
        在[BOLT＃7]中指定（07-routing-gossip.md #the-channel_update-message）。
    - 可以包含多个`r`字段以提供多个路由选项。
  - 必须使用0将字段数据填充为5位的倍数。
  - 如果作者提供多种字段类型，它：
    - 必须先按顺序指定最喜欢的字段，然后按顺序指定不太优先的字段。

一位读者：
  - 必须跳过未知的字段，或者一个带有未知`版本`的`f`字段，或``p`，`h`或
  `n`字段的数据长度分别为52,52或53。
  - 必须检查`h`字段中的SHA2 256位散列是否与散列完全匹配
  描述。
  - 如果提供了有效的`n`字段：
    - 必须使用`n`字段来验证签名，而不是执行签名恢复。

### 合理

类型和长度格式允许将来的扩展是向后的
兼容。 `data_length`总是5位的倍数，这很容易
编码和解码。读者也会忽略不同长度的字段，
对于预期的字段可能会更改。

`p`字段支持当前的256位支付哈希，但未来
规范可以添加不同长度的新变体：在这种情况下，
作家可以支持旧的和新的变体，老读者会
忽略变量而不是正确的长度。

`d`字段允许内联描述，但可能不够
复杂的订单。因此，`h`字段允许摘要：尽管方法
用于描述的是尚未指定的并且将是
可能是依赖运输的。 `h`格式将来可能会改变，
通过改变长度，如果它不是256位，读者会忽略它。

`n`字段可用于显式指定目标节点ID，
而不是要求签名恢复。

“x”字段会显示付款时间的警告
拒绝：主要是为了避免混淆。选择了默认值
对大多数付款都是合理的，并留出足够的时间
如有必要，可以进行在线支付。

`c`字段允许目标节点需要特定的方式
传入HTLC的最低CLTV到期时间。目标节点可以使用它
需要比默认值更高，更保守的值（取决于
他们的费用估算政策及其对时间锁的敏感度）。注意
路由中的远程节点指定了所需的`cltv_expiry_delta`
在`channel_update`消息中，他们可以随时更新。

`f`字段允许链上回退;然而，这可能没有意义
微小或时间敏感的付款。它可能是新的
地址表格将出现;因此，多个`f`字段（隐含的
首选顺序）帮助转换，以及版本19-31的`f`字段
将被读者忽略。

`r`字段允许有限的路由辅助：如指定的那样，它只是
允许最少的信息使用私人渠道，但也可以
协助未来的部分知识路由。

### 付款说明的安全注意事项

付款说明是用户定义的，并提供了一个潜在的途径
注入攻击：在渲染和持久化过程中。

付款说明应始终在显示之前进行清理
HTML / Javascript上下文（或任何其他动态解释的呈现
构架）。实施者应该对这种可能性有额外的洞察力
在解码和显示付款说明时，反映了XSS攻击。避免
乐观地呈现支付请求的内容直到全部
验证，验证和清理过程已成功完成
完成。

此外，请考虑使用预准备语句，输入验证和/或
逃避，以防止持久性注入漏洞
支持SQL或其他动态解释的查询语言的引擎。

* [存储和反映的XSS预防](https://www.owasp.org/index.php/XSS_(Cross_Site_Scripting)_Prevention_Cheat_Sheet)
* [基于DOM的XSS预防](https://www.owasp.org/index.php/DOM_based_XSS_Prevention_Cheat_Sheet)
* [SQL注入预防](https://www.owasp.org/index.php/SQL_Injection_Prevention_Cheat_Sheet)

不要像[小鲍比表]的学校（https://xkcd.com/327/）。

# 付款人/收款人互动

这些通常由Lightning BOLT系列的其他部分定义，
但值得注意的是[BOLT＃5]（05-onchain.md）指定收款人应该
接受最多两倍的预期“金额”，所以付款人可以
通过添加小变化更难以跟踪付款。

目的是付款人从中恢复收款人的节点ID
签名，并在检查费用等条件后，
到期和块超时是可以接受的，尝试付款。它可以使用`r`字段
如果需要到达最终节点，请增加其路由信息。

如果付款成功但后来有争议，付款人可以
证明收款人签署的要约和成功
付款。

## 付款人/收款人要求

付款人：
  - 在'timestamp`加上'expiry`之后：
    - 不应该尝试付款。
  - 除此以外：
    - 如果Lightning付款失败：
      - 可能会尝试使用第一个`f`字段中给出的地址
      了解付款。
  - 可以使用由`r`字段指定的通道序列来路由到收款人。
  - 在开始付款之前，我们应该考虑费用金额和付款超时。
  - 应该使用它没有跳过的第一个`p`字段作为支付哈希值。

收款人：
  - 在'timestamp`加上'expiry`之后：
    - 不应该接受付款。

# 履行

https://github.com/rustyrussell/lightning-payencode

# 例子

注意：所有以下示例都使用`priv_key` =`e126f68f7eafcc8b74f54d269fe206be715000f94dac067d1c04a8ca3b2db734`进行签名。

> ###请使用payment_hash 0001020304050607080900010203040506070809000102030405060708090102向我捐赠任何金额@ 03e7156ae33b0a208d0744199163177e909e80176e55d97a2f221ede0f934dd9ad
> lnbc1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpl2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq8rkx3yf5tcsyz3d73gafnh3cax9rn449d9p5uxz9ezhhypd0elx87sjle52x86fux2ypatgddc6k63n7erqz25le42c4u4ecky03ylcqca784w

分解：

* `lnbc`：前缀，比特币主网上的Lightning
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `p`：支付哈希
  * `p5`：`data_length`（`p` = 1，`5` = 20; 1 * 32 + 20 == 52）
  * `qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypq`：支付哈希值0001020304050607080900010203040506070809000102030405060708090102
* `d`：简短说明
  * `pl`：`data_length`（`p` = 1，`l` = 31; 1 * 32 + 31 == 63）
  * `2pkx2ctnv5sxxmmwwd5kgetjypeh2ursdae8g6twvus8g6rfwvs8qun0dfjkxaq`：'请考虑支持这个项目'
* `8rkx3yf5tcsyz3d73gafnh3cax9rn449d9p5uxz9ezhhypd0elx87sjle52x86fux2ypatgddc6k63n7erqz25le42c4u4ecky03ylcq`：签名
* `ca784w`：Bech32校验和
* 签名细分：
  * `38ec6891345e204145be8a3a99de38e98a39d6a569434e1845c8af7205afcfcc7f425fcd1463e93c32881ead0d6e356d467ec8c02553f9aab15e5738b11f127f`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`0`（int）恢复标志
  * `6c6e62630b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a1fa83632b0b9b29031b7b739b4b232b91039bab83837b93a34b733903a3434b990383937b532b1ba0`用于签名的数据的十六进制（前缀+分隔符之后的数据直到签名的开头）
  * 原像的SHA256的`c3d4e83f646fa79a393d75277b1d858db1d1f7ab7137dcb7835db2ecd518e1c9`十六进制

> ###请在一分钟内向同一个同行送3美元换一杯咖啡
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdq5xysxxatsyp3k7enxv4jsxqzpuaztrnwngzn3kdzw5hydlzf03qdgm2hdq27cqv3agm2awhz5se903vruatfhq77w3ls4evs3ch9zw97j25emudupq63nyw24cg27h2rspfj9srp

分解：

* `lnbc`：前缀，比特币主网上的Lightning
* `2500u`：金额（2500微比特币）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `p`：付款哈希......
* `d`：简短说明
  * `q5`：`data_length`（`q` = 0，`5` = 20; 0 * 32 + 20 == 20）
  * `xysxxatsyp3k7enxv4js`：'1杯咖啡'
* `x`：到期时间
  * `qz`：`data_length`（`q` = 0，`z` = 2; 0 * 32 + 2 == 2）
  * `pu`：60秒（`p` = 1，`u` = 28; 1 * 32 + 28 == 60）
* `aztrnwngzn3kdzw5hydlzf03qdgm2hdq27cqv3agm2awhz5se903vruatfhq77w3ls4evs3ch9zw97j25emudupq63nyw24cg27h2rsp`：签名
* `fj9srp`：Bech32校验和
* 签名细分：
  * `e89639ba6814e36689d4b91bf125f10351b55da057b00647a8dabaeb8a90c95f160f9d5a6e0f79d1fc2b964238b944e2fa4aa677c6f020d466472ab842bd750e`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`1`（int）恢复标志
  * `6c6e626332353030750b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404081a0a189031bab81031b7b33332b2818020f00`用于签名的十六进制数据（前缀+分隔符后直到签名开头的数据）
  * 原像的SHA256的`3cd6ef07744040556e01be64f68fd9e1565fb47d78c42308b1ee005aca5a0d86`六角形

> ###请在一分钟内向同一个同伴发送0.0025 BTC一杯废话（ナンセンス1杯）
> lnbc2500u1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqdpquwpc4curk03c9wlrswe78q4eyqc7d8d0xqzpuyk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esqchsrny

分解：

* `lnbc`：前缀，比特币主网上的Lightning
* `2500u`：金额（2500微比特币）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `p`：付款哈希......
* `d`：简短说明
  * `pq`：`data_length`（`p` = 1，`q` = 0; 1 * 32 + 0 == 32）
  * `uwpc4curk03c9wlrswe78q4eyqc7d8d0`：'ナンセンス1杯'
* `x`：到期时间
  * `qz`：`data_length`（`q` = 0，`z` = 2; 0 * 32 + 2 == 2）
  * `pu`：60秒（`p` = 1，`u` = 28; 1 * 32 + 28 == 60）
* `yk0sg5g70me25alkluzd2x62aysf2pyy8edtjeevuv4p2d5p76r4zkmneet7uvyakky2zr4cusd45tftc9c5fh0nnqpnl2jfll544esq`：签名
* `chsrny`：Bech32校验和
* 签名细分：
  * `259f04511e7efaa77f6ff4d51b4ae9209504843e5ab9672ce32a153681f687515b73ce57ee309db588a10eb8e41b5a2d2bc17144ddf398033faa49ffe95ae6`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`0`（int）恢复标志
  * `6c6e626332353030750b25fe40410d00004080c1014181c20240004080c10014181c20240004080c1014181c202404081a1071c1c571c1d9f1c15df1c1c971c1d9f1c15df1c1d9f1c15c9018f34ed798020f0`用于签名的十六进制数据（前缀+分隔符后的数据直到签名开头）
  * `原始图像的SHA256的'197a3061f4f333d86669b8054592222b488f3c657a9d3e74f34f586fb3e7931c`六角形

> ###现在送24美元买一整套东西（哈希）
> lnbc20m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqscc6gd6ql3jrc5yzme8v4ntcewwz5cnw92tz0pc8qcuufvq7khhr8wpald05e92xw006sq94mg8v2ndf4sefvf9sygkshp5zfem29trqq2yxxz7

分解：

* `lnbc`：前缀，比特币主网上的Lightning
* `20m`：数量（20 milli-bitcoin）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `p`：付款哈希......
* `h`：标记字段：描述的哈希
  * `p5`：`data_length`（`p` = 1，`5` = 20; 1 * 32 + 20 == 52）
  * `8yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqs`：'一块巧克力蛋糕，一个冰淇淋蛋筒，一个泡菜，一片瑞士奶酪，一片意大利腊肠，一个棒棒糖，一片樱桃馅饼，一片香肠，一块蛋糕和一片西瓜'
* `cc6gd6ql3jrc5yzme8v4ntcewwz5cnw92tz0pc8qcuufvq7khhr8wpald05e92xw006sq94mg8v2ndf4sefvf9sygkshp5zfem29trqq`：签名
* `2yxxz7`：Bech32校验和
* 签名细分：
  * `c63486e81f8c878a105bc9d959af1973854c4dc559c4f0e0e0c7389603d6bdc67707bf6be992a8ce7bf50016bb41d8a9b5358652c4960445a170d049ced4558c`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`0`（int）恢复标志
  * `6c6e626332306d0b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404082e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc60800`用于签名的数据十六进制（前缀+分隔符后的数据直到签名的开头）
  * `原始图像的SHA256的'b6025e8a10539dddbcbe6840a9650707ae3f147b8dcfda338561ada710508916`六角形

> ###同样，在testnet上，带有备用地址mk2QpYatsKicvFVuTAQLBryyccRXMUaGHP
> lntb20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfpp3x9et2e20v6pu37c5d9vax37wxq72un98kmzzhznpurw9sgl2v0nklu2g4d0keph5t7tj9tcqd8rexnd07ux4uv2cjvcqwaxgj7v4uwn5wmypjd5n69z2xm3xgksg28nwht7f6zspwp3f9t

分解：

* `lntb`：前缀，比特币testnet上的Lightning
* `20m`：数量（20 milli-bitcoin）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `h`：标记字段：描述哈希...
* `p`：付款哈希......
* `f`：标记字段：后备地址
  * `pp`：`data_length`（`p` = 1; 1 * 32 + 1 == 33）
  * `3` = 17，所以P2PKH地址
  * `x9et2e20v6pu37c5d9vax37wxq72un98`：160位P2PKH地址
* `kmzzhznpurw9sgl2v0nklu2g4d0keph5t7tj9tcqd8rexnd07ux4uv2cjvcqwaxgj7v4uwn5wmypjd5n69z2xm3xgksg28nwht7f6zsp`：签名
* `wp3f9t`：Bech32校验和
* 签名细分：
  * `b6c42b8a61e0dc5823ea63e76ff148ab5f6c86f45f9722af0069c7934daff70d5e315893300774c897995e3a7476c8193693d144a36e2645a0851e6ebafc9d0a`签名数据的十六进制（32字节r，32字节s）
  * `signature`中包含的`1`（int）恢复标志
  * `数据的6c6e746232306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a000081018202830384048000810182028303840480008101820283038404808102421898b95ab2a7b341e47d8a34ace9a3e7181e5726538`六角签署（分离器之后的前缀+数据到签名的开始）
  * 原像素的SHA256的'00c17b39642becc064615ef196a6cc0cce262f1d8dde7b3c23694aeeda473abe`六角形

> ###在主网上，后备地址为1RustyRX2oai4EYYDpQGWvEL62BBGqN9T，附加路由信息通过节点029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255然后039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255
> lnbc20m1pvjluezpp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqsfpp3qjmp7lwpagxun9pygexvgpjdc4jdj85fr9yq20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzqj9n4evl6mr5aj9f58zp6fyjzup6ywn3x6sk8akg5v4tgn2q8g4fhx05wf6juaxu9760yp46454gpg5mtzgerlzezqcqvjnhjh8z3g2qqdhhwkj

分解：

* `lnbc`：前缀，比特币主网上的Lightning
* `20m`：数量（20 milli-bitcoin）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `p`：付款哈希......
* `h`：标记字段：描述哈希...
* `f`：标记字段：后备地址
  * `pp`：`data_length`（`p` = 1; 1 * 32 + 1 == 33）
  * `3` = 17，所以P2PKH地址
  * `qjmp7lwpagxun9pygexvgpjdc4jdj85f`：160位P2PKH地址
* `r`：标记字段：路由信息
  * `9y`：`data_length`（`9` = 5，`y` = 4; 5 * 32 + 4 == 164）
    * `q20q82gphp2nflc7jtzrcazrra7wwgzxqc8u7754cdlpfrmccae92qgzqvzq2ps8pqqqqqqpqqqqq9qqqvpeuqafqxu92d8lr6fvg0r5gv0heeeqgcrqlnm6jhphu9y00rrhy4grqszsvpcgpy9qqqqqqgqqqqq7qqzq`：
      * pubkey：`029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`：0102030405060708
      * `fee_base_msat`：1毫士摩
      * `fee_proportional_millionths`：20
      * `cltv_expiry_delta`：3
      * pubkey：`039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255`
      * `short_channel_id`：030405060708090a
      * `fee_base_msat`：2毫士摩
      * `fee_proportional_millionths`：30
      * `cltv_expiry_delta`：4
* `j9n4evl6mr5aj9f58zp6fyjzup6ywn3x6sk8akg5v4tgn2q8g4fhx05wf6juaxu9760yp46454gpg5mtzgerlzezqcqvjnhjh8z3g2qq`：签名
* `dhhwkj`：Bech32校验和
* 签名细分：
  * `91675cb3fad8e9d915343883a49242e074474e26d42c7ed914655689a8074553733e8e4ea5ce9b85f69e40d755a55014536b12323f8b220600c94ef2b9c51428`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`0`（int）恢复标志
  * `数据的6c6e626332306d0b25fe64410d00004080c1014181c20240004080c1014181c20240004080c1014181c202404082e1a1c92db7b3f161a001b7689049eea2701b46f8db7513629edf2408fac7eaedc60824218825b0fbee0f506e4ca122326620326e2b26c8f448ca4029e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255010203040506070800000001000000140003039e03a901b85534ff1e92c43c74431f7ce72046060fcf7a95c37e148f78c77255030405060708090a000000020000001e00040`六角签署（分离器之后的前缀+数据到签名的开始）
  * 原像的SHA256的`ff68246c5ad4b48c90cf8ff3b33b5cea61e62f08d0e67910ffdce1edecade71b`十六进制

> ###在主网上，后备（P2SH）地址3EktnHQD7RiAE6uzMj2ZifT9YgRrkSgzQX
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppj3a24vwu6r8ejrss3axul8rxldph2q7z9kmrgvr7xlaqm47apw3d48zm203kzcq357a4ls9al2ea73r8jcceyjtya6fu5wzzpe50zrge6ulk4nvjcpxlekvmxl6qcs9j3tz0469gq5g658y

分解：

* `lnbc`：前缀，比特币主网上的Lightning
* `20m`：数量（20 milli-bitcoin）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `h`：标记字段：描述哈希...
* `p`：付款哈希......
* `f`：标记字段：后备地址
  * `pp`：`data_length`（`p` = 1; 1 * 32 + 1 == 33）
  * `j` = 18，所以P2SH地址
  * `3a24vwu6r8ejrss3axul8rxldph2q7z9`：160位P2SH地址
* `kmrgvr7xlaqm47apw3d48zm203kzcq357a4ls9al2ea73r8jcceyjtya6fu5wzzpe50zrge6ulk4nvjcpxlekvmxl6qcs9j3tz0469gq`：签名
* `5g658y`：Bech32校验和
* 签名细分：
  * `b6c6860fc6ff41bafba1745b538b6a7c6c2c0234f76bf817bf567be88cf2c632492c9dd279470841cd1e21a33ae7ed59b25809bf9b3366fe81881651589f5d15`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`0`（int）恢复标志
  * `数据的6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a000081018202830384048000810182028303840480008101820283038404808102421947aaab1dcd0cf990e108f4dcf9c66fb437503c228`六角签署（分离器之后的前缀+数据到签名的开始）
  * 原像的SHA256的`64f1ff500bcc62a1b211cd6db84a1d93d1f77c6a132904465b6ff912420176be`六角形

> ###在主网上，后备（P2WPKH）地址bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfppqw508d6qejxtdg4y5r3zarvary0c5xw7kepvrhrm9s57hejg0p662ur5j5cr03890fa7k2pypgttmh4897d3raaq85a293e9jpuqwl0rnfuwzam7yr8e690nd2ypcq9hlkdwdvycqa0qza8

* `lnbc`：前缀，比特币主网上的Lightning
* `20m`：数量（20 milli-bitcoin）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `h`：标记字段：描述哈希...
* `p`：付款哈希......
* `f`：标记字段：后备地址
  * `pp`：`data_length`（`p` = 1; 1 * 32 + 1 == 33）
  * `q`：0，见证版本0
  * `w508d6qejxtdg4y5r3zarvary0c5xw7k`：160位= P2WPKH。
* `epvrhrm9s57hejg0p662ur5j5cr03890fa7k2pypgttmh4897d3raaq85a293e9jpuqwl0rnfuwzam7yr8e690nd2ypcq9hlkdwdvycq`：签名
* `a0qza8`：Bech32校验和
* 签名细分：
  * `c8583b8f65853d7cc90f0eb4ae0e92a606f89caf4f7d65048142d7bbd4e5f3623ef407a75458e4b20f00efbc734f1c2eefc419f3a2be6d51038016ffb35cd613`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`0`（int）恢复标志
  * `数据的6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a00008101820283038404800081018202830384048000810182028303840480810242103a8f3b740cc8cb6a2a4a0e22e8d9d191f8a19deb0`六角签署（分离器之后的前缀+数据到签名的开始）
  * 原像素的SHA256的'b3df27aaa01d891cc9de272e7609557bdf4bd6fd836775e4470502f71307b627`六角形

> ###在主网上，带有后备（P2WSH）地址bc1qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3qccfmv3
> lnbc20m1pvjluezhp58yjmdan79s6qqdhdzgynm4zwqd5d7xmw5fk98klysy043l2ahrqspp5qqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqqqsyqcyq5rqwzqfqypqfp4qrp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q28j0v3rwgy9pvjnd48ee2pl8xrpxysd5g44td63g6xcjcu003j3qe8878hluqlvl3km8rm92f5stamd3jw763n3hck0ct7p8wwj463cql26ava

* `lnbc`：前缀，比特币主网上的Lightning
* `20m`：数量（20 milli-bitcoin）
* `1`：Bech32分隔符
* `pvjluez`：时间戳（1496314658）
* `h`：标记字段：描述哈希...
* `p`：付款哈希......
* `f`：标记字段：后备地址
  * `p4`：`data_length`（`p` = 1，`4` = 21; 1 * 32 + 21 == 53）
  * `q`：0，见证版本0
  * `rp33g0q5c5txsp9arysrx4k6zdkfs4nce4xj0gdcccefvpysxf3q`：260位= P2WSH。
* `28j0v3rwgy9pvjnd48ee2pl8xrpxysd5g44td63g6xcjcu003j3qe8878hluqlvl3km8rm92f5stamd3jw763n3hck0ct7p8wwj463cq`：签名
* `l26ava`：Bech32校验和
* 签名细分：
  * `51e4f6446e410a164a6da9f39507e730c26241b4456ab6ea28d1b12c71ef8ca20c9cfe3dffc07d9f8db671ecaa4d20beedb193bda8ce37c59f85f82773a55d47`十六进制签名数据（32字节r，32字节s）
  * `signature`中包含的`0`（int）恢复标志
  * `数据的6c6e626332306d0b25fe64570d0e496dbd9f8b0d000dbb44824f751380da37c6dba89b14f6f92047d63f576e304021a00008101820283038404800081018202830384048000810182028303840480810243500c318a1e0a628b34025e8c9019ab6d09b64c2b3c66a693d0dc63194b02481931000`六角签署（分离器之后的前缀+数据到签名的开始）
  * '399a8b167029fda8564fd2e99912236b0b8017e7d17e416ae17307812c92cf42`原像的SHA256的十六进制

# 作者

[ 整我： ]

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
