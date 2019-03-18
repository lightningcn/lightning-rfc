# BOLT＃10：DNS引导程序和辅助节点位置

## 概观

该规范描述了基于域名系统（DNS）的节点发现机制。
它的目的有两个：

 - Bootstrap：为网络中没有已知联系人的节点提供初始节点发现
 - 辅助节点位置：支持发现先前已知对等体的当前网络地址的节点

实现此规范的域名服务器称为
_DNS Seed_并回答类型为“A”，“AAAA”或“SRV”的传入DNS查询
在RFC 1035中指定<sup> [1]（＃ref-1）</sup>,3596 <sup> [2]（＃ref-2）</sup>，以及
2782 <sup> [3]（＃ref-3）</sup>。
DNS服务器对子域具有权威性，称为a
_seed root domain_，客户端可以查询子域名。

子域由多个以点分隔的_conditions_组成，这些_conditions_进一步缩小了所需的结果。

## 目录

  * [DNS种子查询](#dns-seed-queries)
    * [查询语义](#query-semantics)
  * [回复施工](#reply-construction)
  * [政策](#policies)
  * [例子](#examples)
  * [参考](#references)
  * [作者](#authors)

## DNS种子查询

客户端可以使用`A`，`AAAA`或`SRV`查询类型发出查询，
指定种子应该返回的期望结果的条件。

查询区分_wildcard_查询和_node_查询，具体取决于
是否设置了'l`键。

### 查询语义

条件是键值对：键是单个字母，而键
键值对的剩余部分是值。
DNS种子必须支持以下键值对：

 - `r`：领域字节
   - 用于指定返回的节点必须支持的域
   - 默认值：0（比特币）
 - `a`：地址类型
   - 使用[BOLT＃7]（07-routing-gossip.md）中的类型作为位的位域
   指数
   - 用于指定应为`SRV`查询返回的地址类型
   - 可能只用于`SRV`查询
   - 默认值：6（即“2 || 4”，因为第1位和第2位设置为IPv4和
     IPv6，分别）
 - `l`：`node_id`
   - 特定节点的bech32编码的“node_id”
   - 曾经要求单个节点而不是随机选择
   - 默认值：null
 - `n`：所需答复记录的数量
   - 默认值：25

条件在DNS种子查询中作为单独的点分隔子域组件传递。

例如，对`r0.a2.n10.lseed.bitcoinstats.com`的查询意味着：返回
支持比特币（`r0`）的节点的10（`n10`）IPv4（`a2`）记录。

### 要求

DNS种子：
  - 必须从_seed root domain_ by评估条件
  '上升树'，即在完全合格的域中评估从右到左
名称。
    - 例如。评估上述情况：首先评估`n10`，然后评估`a2`，最后评估`r0`。
  - 如果多次指定条件（键）：
    - 必须丢弃该条件的任何早期值并使用新值
    代替。
      - 例如。对于`n5.r0.a2.n10.lseed.bitcoinstats.com`，结果是：
      ~~`n10` ~~，`a2`，`r0`，`n5`。
  - 应该返回符合所有条件的结果。
  - 如果它没有按给定条件实现过滤：
    - 可以完全忽略条件（即种子过滤仅是最好的努力）。
  - 对于'A`和`AAAA`查询：
    - 必须只返回监听默认端口9735的节点，如中定义的那样
    [BOLT＃1]（01-messaging.md）。
  - 对于`SRV`查询：
    - 可以返回正在侦听非默认端口的节点，因为`SRV`
    记录返回_（主机名，端口）_-元组。
  - 收到_wildcard_查询后：
    - 必须选择最多n个IPv4或节点的IPv6地址的随机子集
    正在侦听传入的连接。
  - 收到_node_查询后：
    - 必须选择与`node_id`匹配的记录，如果有，则返回all
    与该节点关联的地址。

查询客户：
  - 绝不能依赖于结果满足的任何给定条件。

### 回复施工

结果在回复中序列化，查询类型与客户端匹配
查询类型。例如，分别导致`A`，`AAAA`和`SRV`查询
`A`，`AAAA`和`SRV`回复。此外，回复可能会增加
附加记录（例如添加与返回的匹配的`A`或`AAAA`记录
`SRV`记录）。

对于“A”和“AAAA”查询，回复包含域名和IP
结果的地址。

域名必须与查询中的域匹配，以便不进行过滤
由中间解析器。

对于`SRV`查询，答复由（_virtual hostnames_，port）-tuples组成。
虚拟主机名是唯一的种子根域的子域
标识网络中的节点。
它是通过将`node_id`条件添加到种子根域来构造的。

DNS种子：
  - 可另外返回相应的`A`和`AAAA`记录
  在附加部分中指出`SRV`条目的IP地址
  答复。
- 可以在检测到重复查询时省略这些附加记录。
  - 原因：由于结果回复的大小，回复可能是
  由中间解析器掉落。
- 如果没有条目符合所有条件：
  - 必须返回一个空的回复。

## 政策

DNS种子：
  - 不得返回TTL小于60秒的回复。
  - 可能由于各种原因（包括故障）从本地视图中过滤节点
  节点，片状节点或垃圾邮件防护。
  - 必须回复随机查询（即查询种子根域和
    使用_random和unbiased_的`_nodes._tcp`别名为`SRV`查询）
    根据比特币DNS种子策略<sup> [4]（＃ref-4）</sup>，来自所有已知良好节点集合的样本。

## 例子

查询“AAAA”记录：

    $ dig lseed.bitcoinstats.com AAAA
    lseed.bitcoinstats.com。 60在AAAA 2a02：aa16：1105：4a80：1234：1234：37c1：9c9

查询`SRV`记录：

    $ dig lseed.bitcoinstats.com SRV
    lseed.bitcoinstats.com。 59 IN SRV 10 10 6331 ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com。
    lseed.bitcoinstats.com。 59 IN SRV 10 10 9735 ln1qv2w3tledmzczw227nnkqrrltvmydl8gu4w4d70g9td7avke6nmz2tdefqp.lseed.bitcoinstats.com。
    lseed.bitcoinstats.com。 59 IN SRV 10 10 9735 ln1qtynyymv99pqf0r9cuexvvqtxrlgejuecf8myfsa96vcpflgll5cqmr2xsu.lseed.bitcoinstats.com。
    lseed.bitcoinstats.com。 59 IN SRV 10 10 4280 ln1qdfvlysfpyh96apy3w3qdwlu8jjkdhnuxa689ka540tnde6gnx86cf7ga2d.lseed.bitcoinstats.com。
    lseed.bitcoinstats.com。 59 IN SRV 10 10 4281 ln1qwf789tlcpe4n34649xrqllxt97whsvfk5pm07ggqms3vrjwdj3cu6332zs.lseed.bitcoinstats.com。

查询上一个示例中第一个虚拟主机名的“A”：

    $ dig ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com A
    ln1qwktpe6jxltmpphyl578eax6fcjc2m807qalr76a5gfmx7k9qqfjwy4mctz.lseed.bitcoinstats.com。 60 IN A 139.59.143.87

仅通过种子过滤查询IPv4节点（`a2`）：

    $ dig a2.lseed.bitcoinstats.com SRV
    a2.lseed.bitcoinstats.com。 59 IN SRV 10 10 9735 ln1q2jy22cg2nckgxttjf8txmamwe9rtw325v4m04ug2dm9sxlrh9cagrrpy86.lseed.bitcoinstats.com。
    a2.lseed.bitcoinstats.com。 59 IN SRV 10 10 9735 ln1qfrkq32xayuq63anmc2zp5vtd2jxafhdzzudmuws0hvxshtgd2zd7jsqv7f.lseed.bitcoinstats.com。

仅通过种子过滤查询支持比特币（`r0`）的IPv6节点（`a4`）：

    $ dig r0.a4.lseed.bitcoinstats.com SRV
    r0.a4.lseed.bitcoinstats.com。 59 IN SRV 10 10 9735 ln1qwx3prnvmxuwsnaqhzwsrrpwy4pjf5m8fv4m8kcjkdvyrzymlcmj5dakwrx.lseed.bitcoinstats.com。
    r0.a4.lseed.bitcoinstats.com。 59 IN SRV 10 10 9735 ln1qwr7x7q2gvj7kwzzr7urqq9x7mq0lf9xn6svs8dn7q8gu5q4e852znqj3j7.lseed.bitcoinstats.com。

## 参考
-  <a id="ref-1"> [RFC 1035  - 域名]（https://www.ietf.org/rfc/rfc1035.txt)</a>
-  <a id="ref-2"> [RFC 3596  - 支持IP版本6的DNS扩展]（https://tools.ietf.org/html/rfc3596)</a>
-  <a id="ref-3"> [RFC 2782  - 用于指定服务位置的DNS RR（DNS SRV）]（https://www.ietf.org/rfc/rfc2782.txt)</a>
-  <a id="ref-4"> [对DNS种子运营商的期望]（https://github.com/bitcoin/bitcoin/blob/master/doc/dnsseed-policy.md)</a>

## 作者

[FIXME：插入作者列表]

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
