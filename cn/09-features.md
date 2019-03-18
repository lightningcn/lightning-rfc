# BOLT＃9：分配的功能标志

本文档跟踪“localfeatures”和“globalfeatures”的分配
`init`消息中的标志（[BOLT＃1]（01-messaging.md））以及
`channel_announcement`和`node_announcement`中的`features`标志字段
消息（[BOLT＃7]（07-routing-gossip.md））。
标记被单独跟踪，因为随着时间的推移可能会添加新标记。

路由消息中的`features`标志是其中的一个子集
根据定义，`globalfeatures`标志作为“localfeatures”只是有意义的
指导同行。

标志从最低有效位编号，位0（即0x1，
一个_even_位）。它们通常成对分配以便具有特征
可以作为可选（_odd_位）引入，然后升级为必修
（_even_ bits），将被过时的节点拒绝：
见[BOLT＃1：`init`消息]（01-messaging.md #the-init-message）。

## 分配了`localfeatures`标志

这些标志只能在`init`消息中使用：

| 位 | 名称             |描述                                     | 链接                                                                |
|------|------------------|------------------------------------------------|---------------------------------------------------------------------|
| 0/1  | `option_data_loss_protect` | 需要或支持额外的`channel_reestablish`字段 | [BOLT＃2](02-peer-protocol.md#message-retransmission) |
| 3  | `initial_routing_sync` | 表示发送节点需要完整的路由信息转储 | [BOLT＃7](07-routing-gossip.md#initial-sync) |
| 4/5  | `option_upfront_shutdown_script` | 打开频道时提交关闭scriptpubkey | [BOLT＃2](02-peer-protocol.md#the-open_channel-message) |
| 6/7  | `gossip_queries`           | 更复杂的八卦控制 | [BOLT＃7](07-routing-gossip.md#query-messages) |

## 分配了`globalfeatures`标志

目前没有`globalfeatures`标志。

## 要求

接收特定位的要求在上表的链接部分中定义。
未定义的要素位的要求
上面的内容可以在[BOLT＃1：`init`消息]中找到（01-messaging.md #the-init-message）。

## 合理

`initial_routing_sync`没有_even_位，因为几乎没有
point：本地节点无法确定远程节点是否符合，并且必须符合
解释标志，如初始规范中所定义。

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
