# BOLT＃5：对链式交易处理的建议

## 抽象

Lightning允许双方（本地节点和远程节点）进行交易
通过给予每一方*交叉签署的承诺交易*来实现脱链，
它描述了通道的当前状态（基本上是当前的平衡）。
每次付款时，此*承诺交易*都会更新
在任何时候都可以度过。

渠道可以通过三种方式结束：

1. 好方法（*相互关闭*）：在某些时候本地和远程节点都同意
关闭频道。他们生成一个*结束交易*（类似于
承诺交易，但没有任何待处理的付款）并将其发布在
区块链（参见[BOLT＃2：频道关闭]（02-peer-protocol.md＃channel-close））。
2. 糟糕的方式（*单方面关闭*）：出现问题，可能没有恶意
意图在任何一方。例如，也许有一方坠毁了。一边
发布其*最新承诺交易*。
3. 丑陋的方式（*撤销交易关闭*）：其中一方故意
试图通过发布*过时的承诺交易来欺骗*（大概是，
以前的版本，更有利于它。

因为Lightning的设计是无信任的，所以不存在资金损失的风险
在这三种情况中的任何一种;只要情况得到妥善处理。
本文档的目标是准确解释节点应该如何反应
遇到任何上述情况，在链上。

# 目录
  * [一般命名法](#general-nomenclature)
  * [承诺交易](#commitment-transaction)
  * [失败的频道](#failing-a-channel)
  * [相互关闭处理](#mutual-close-handling)
  * [单方面关闭处理：本地承诺交易](#unilateral-close-handling-local-commitment-transaction)
      * [HTLC输出处理：本地承诺，本地优惠](#htlc-output-handling-local-commitment-local-offers)
      * [HTLC输出处理：本地承诺，远程优惠](#htlc-output-handling-local-commitment-remote-offers)
  * [单边关闭处理：远程承诺交易](#unilateral-close-handling-remote-commitment-transaction)
      * [HTLC输出处理：远程承诺，本地优惠](#htlc-output-handling-remote-commitment-local-offers)
      * [HTLC输出处理：远程承诺，远程优惠](#htlc-output-handling-remote-commitment-remote-offers)
  * [撤销交易关闭处理](#revoked-transaction-close-handling)
      * [罚款交易权重计算](#penalty-transactions-weight-calculation)
  * [一般要求](#general-requirements)
  * [附录A：预期权重](#appendix-a-expected-weights)
    * [“to_local”刑罚交易证人的预期权重](#expected-weight-of-the-to-local-penalty-transaction-witness)
    * ['provided_htlc`惩罚交易证人的预期权重](#expected-weight-of-the-offered-htlc-penalty-transaction-witness)
    * [“accepted_htlc”刑罚交易证人的预期权重](#expected-weight-of-the-accepted-htlc-penalty-transaction-witness)
  * [作者](#authors)

# 一般命名法

任何未花费的输出都被视为*未解析*并且可以*解析*
详见本文件。通常这是通过花钱来实现的
另一个*解决*交易。虽然，有时只是注意到输出
对于以后的钱包支出是足够的，在这种情况下包含的交易
输出被认为是它自己的* resolving *事务。

*已解决*的输出被视为*不可撤销地解决*
一旦远程的*解析*事务包含在一个块中至少100
在最有效的区块链上。 100个街区远远大于
已知最长的比特币分叉并且使用的等待时间相同
确认矿工的奖励（参见[参考实施]（https://github.com/bitcoin/bitcoin/blob/4db82b7aab4ad64717f742a7318e3dc6811b41be/src/consensus/tx_verify.cpp#L223））。

## 要求

一个节点：
  - 一旦它已经广播了资金交易或发送了承诺签名
  对于包含HTLC输出的承诺事务：
    - 直到所有输出*不可逆转地解决*：
      - 必须监视区块链以查找花费任何输出的交易
      不会*不可逆转地解决*。
  - 必须*解析*所有输出，如下所述。
  - 在区块链的情况下，必须准备好多次解决输出
  重组。
  - 在筹集资金交易时，如果渠道尚未完成
  关闭：
    - 应该通道失败。
    - 可以发送描述性错误包。
  - 应该忽略无效的交易。

## 合理

一旦本地节点有一些资金处于危险之中，就需要监控区块链
确保远程节点不会单方面关闭。

任何人都可以生成无效的交易（例如，不良签名），
（无论如何都会被区块链忽略），所以他们不应该这样做
触发任何动作。

# 承诺交易

本地和远程节点各自持有*承诺交易*。每一个
承诺交易有四种类型的产出：

1. _local节点的主输出_：零或一个输出，支付给*本地节点的*
承诺驴。
2. _remote节点的主输出_：零或一个输出，以支付*远程节点的*
承诺驴。
3. _local node提供的HTLCs_：零或更多待处理付款（* HTLC *），付款
*远程节点*以换取付款原像。
4. _remote节点提供的HTLCs_：零或更多待处理付款（* HTLCs *），to
支付*本地节点*以换取付款原像。

为了激励本地和远程节点合作，一个'OP_CHECKSEQUENCEVERIFY`
相对超时会阻塞*本地节点的输出*（在*本地节点中）
承诺事务*）和*远程节点的输出*（在*远程节点中）
承诺交易*）。例如，如果本地节点发布它
承诺交易，它将不得不等待自己的资金，
而远程节点可以立即访问自己的资金。作为一个
结果，两个承诺交易不一样，但它们是
（通常）对称的。

见[BOLT＃3：承诺交易]（03-transactions.md＃commitment-transaction）
更多细节。

# 失败的频道

尽管关闭频道可以通过多种方式完成，但最多
有效是首选。

各种错误案例涉及关闭频道。发送要求
向对等方的错误消息指定在
[BOLT＃1：`error`消息]（01-messaging.md #the-error-message）。

## 要求

一个节点：
  - 如果*本地承诺交易*没有包含`to_local`
  或HTLC输出：
    - 可能只是忘记了频道。
  - 除此以外：
    - 如果*当前承诺交易*不包含`to_local`或
    其他HTLC输出：
      - 可以简单地等待远程节点关闭通道。
      - 直到远程节点关闭：
        - 绝不能忘记频道。
    - 除此以外：
      - 如果它收到一个包含a的有效`closing_signed`消息
      足够的费用：
        - 应该使用这笔费用来执行*相互关闭*。
      - 除此以外：
        - 必须使用*最后承诺交易*，它有一个
        签名，执行*单边关闭*。

## 合理

因为`dust_limit_satoshis`应该防止创造不经济
输出（否则将永远保留，区块链未花费），全部
承诺交易产出必须花费。

在频道的早期阶段，一方常见
渠道中很少或根本没有资金;在这种情况下，没有任何利害关系，一个节点
不需要消耗监控信道状态的资源。

在单边收盘时偏向相互收盘存在偏向，
因为前者的产出不受延迟的影响而且是直接的
可靠的钱包。此外，相互关闭的费用往往不那么夸张
承诺交易。所以，不使用的唯一理由
如果提供的费用太小，那么来自`closing_signed`的签名就是
它被处理。

# 相互关闭处理

结账交易*解决了融资交易输出。

在相互关闭的情况下，节点不需要像其那样做任何其他事情
已经同意输出，它被发送到它指定的`scriptpubkey`（参见
[BOLT＃2：关闭启动：`shutdown`]（02-peer-protocol.md #closing-initiation-shutdown））。

# 单方面关闭处理：本地承诺交易

这是涉及单方面关闭的两起案件中的第一起案件。在这种情况下，一个
节点发现其*本地承诺交易*，*解析*资金
交易输出。

但是，一个节点不能从单边收盘的输出中索取资金
它启动，直到'OP_CHECKSEQUENCEVERIFY`延迟已经过去（如指定的那样）
由远程节点的`to_self_delay`字段）。在相关的情况下，这种情况是
如下所述。

## 要求

一个节点：
  - 在发现其*本地承诺交易*时：
    - 应该将`to_local`输出用于方便的地址。
    - 必须等到'OP_CHECKSEQUENCEVERIFY`延迟过去（如
    在花费之前，由远程节点的“to_self_delay”字段指定
    输出。
      - 注意：如果输出已用完（按推荐），则输出*已解决*
      通过支出交易，否则它被认为*解决了*
      承诺交易本身。
    - 可以忽略`to_remote`输出。
      - 注意：本地节点不需要执行任何操作，因为`to_remote`是
      由承诺交易本身考虑*解决*。
    - 必须按照中的规定处理自己提供的HTLC
    [HTLC输出处理：本地承诺，本地优惠]（＃htlc-output-handling-local-commitment-local-offers）。
    - 必须处理远程节点提供的HTLC
    在[HTLC输出处理：本地承诺，远程优惠]中指定（#htlc-output-handling-local-commitment-remote-offers）。

## 合理

花费'to_local`输出避免了必须记住复杂的
见证脚本，与该特定频道相关联，以供日后使用
开支。

`to_remote`输出完全是远程节点的业务，并且
可以忽略。

## HTLC输出处理：本地承诺，本地优惠

每个HTLC输出只能由本地提供者使用HTLC超时
超时后的交易，或远程收件人，如果有付款
原像。

可能存在未由输出表示的HTLC：或者
因为它们被削减为灰尘，或者因为交易只是
部分承诺。

一旦最新块的深度等于，HTLC就会*超时*
或者大于HTLC`cltv_expiry`。

### 要求

一个节点：
  - 如果承诺交易HTLC输出用于支付
  preimage，输出被视为*不可撤销地解决*：
    - 必须从交易输入见证中提取付款原像。
  - 如果承诺交易HTLC输出*已超时*且尚未
  *解决*：
    - 必须通过使用HTLC超时来解决*输出
    交易。
    - 一旦解决交易达到合理的深度：
      - 必须使相应的输入HTLC失败（如果有的话）。
      - 必须解决该HTLC超时事务的输出。
      - 应该通过花费它来解决HTLC超时事务
      方便的地址。
        - 注意：如果输出已用完（按推荐），则输出为
        *通过支出交易解决*，否则考虑
        *由HTLC超时事务本身解决*。
      - 必须等到'OP_CHECKSEQUENCEVERIFY`延迟过去（如
      由远程节点的`open_channel``to_self_delay`字段指定）
      在花费HTLC超时输出之前。
  - 对于任何承诺的HTLC，该承诺中没有输出
  交易：
    - 一旦承诺交易达到合理的深度：
      - 必须使相应的输入HTLC失败（如果有的话）。
    - 如果没有*有效*承诺事务包含对应的输出
    HTLC。
      - 可能会更快地失败相应的传入HTLC。

### 合理

支付原像用于证明支付（当提供节点时）
发起付款）或兑换相应的传入HTLC
另一个对等体（当提供节点转发支付时）。一旦节点有
提取付款后，它不再关心HTLC支出的命运
交易本身。

在两种分辨率都可能的情况下（例如，当节点收到付款时）
超时后成功），要么解释是可以接受的;它是
收件人在此之前花费它的责任。

需要使用本地HTLC超时事务来超时HTLC（到
防止远程节点履行它并声明资金
使用时，本地节点可以使任何相应的传入HTLC反向失败
`update_fail_htlc`（大概是因为`permanent_channel_failure`），as
详细介绍
[BOLT＃2]（02-peer-protocol.md＃forwarding-htlcs）。
如果传入的HTLC也是链上的，则节点必须等待它
超时：没有办法发出早期失败的信号。

如果HTLC太小而无法出现在*任何承诺交易*中，则可以
立即安全失败。否则，如果HTLC不在*本地承诺中
transaction *，一个节点需要确保一个区块链重组，或者
比赛，不会切换到包含HTLC的承诺交易
在节点失败之前（因此等待）。传入的要求
HTLC在其自身超时仍然作为上限应用之前会失败。

## HTLC输出处理：本地承诺，远程优惠

每个HTLC输出只能由接收者使用HTLC成功
交易，只有付款才能填写
原像。如果它没有preimage（并且没有发现它），那就是
一旦超时，提供者有责任花费HTLC输出。

提供的HTLC有几种可能的情况：

1. 提议者并非不可撤销地致力于此。收件人通常会
   不知道preimage，因为它不会转发HTLC直到它们完全
   承诺。所以使用preimage会发现这个收件人是
   最后一跳;因此，在这种情况下，最好让HTLC超时。
2. 提议者不可撤销地致力于提供的HTLC，但接收者
   尚未致力于传出HTLC。在这种情况下，收件人可以
   提供HTLC的转发或超时。
3. 收件人已承诺传出HTLC，以换取所提供的
   HTLC。在这种情况下，收件人收到后必须使用preimage
   来自即将离任的HTLC;否则，它将通过发送一个传出失去资金
   付款而不兑换收款。

### 要求

本地节点：
  - 如果它收到（或已经拥有）未解决的付款原像
  它已经提供的HTLC输出和它已经承诺的
  传出的HTLC：
    - 必须*通过花费它来解决*输出，使用HTLC成功
    交易。
    - 必须解决该HTLC成功事务的输出。
  - 除此以外：
    - 如果*远程节点*不能不可撤销地提交给HTLC：
      - 绝不能*通过花费来解决*输出。
  - 应该通过花费它来解决HTLC成功事务输出
  方便的地址。
  - 必须等到`OP_CHECKSEQUENCEVERIFY`延迟已经过去（如指定的那样）
    之前由*远程节点的*`open_channel`的`to_self_delay`字段）
    花费HTLC成功交易输出。

如果输出已用完（如建议的那样），则输出*解析*
支出交易，否则HTLC成功将其视为*解决*
交易本身。

如果没有另外解决，一旦HTLC输出到期，它就是
考虑*不可撤销地解决*。

# 单边关闭处理：远程承诺交易

*远程节点的*承诺交易*解决了资金问题
交易输出。

在这种情况下，没有延迟约束节点行为，所以它更简单
要处理的节点，而不是发现其本地承诺的情况
交易（参见[单边关闭处理：本地承诺交易]（#unlateral-close-handling-local-commitment-transaction））。

## 要求

本地节点：
  - 在发现由...广播的*有效*承诺交易时
  *远程节点*：
    - 如果可能的话：
      - 必须按照下面的规定处理每个输出。
      - 对于相关的`to_remote`，可以不采取任何行动
      只是P2WPKH输出到*本地节点*。
        - 注意：承诺交易会将`to_remote`视为*已解决*
        本身。
      - 可以对关联的`to_local`采取不采取行动，这是一个
      付款输出到*远程节点*。
        - 注意：承诺交易将'to_local`视为*已解决*
        本身。
      - 必须按照中的规定处理自己提供的HTLC
      [HTLC输出处理：远程承诺，本地优惠](#htlc-output-handling-remote-commitment-local-offers)
      - 必须按照中的规定处理远程节点提供的HTLC
      [HTLC输出处理：远程承诺，远程优惠](#htlc-output-handling-remote-commitment-remote-offers)
    - 否则（由于某种原因，它无法处理广播）：
      - 必须发送有关资金损失的警告。

## 合理

a之后可能有多个有效的*未撤销*承诺交易
签名已通过`commitment_signed`并在相应之前收到
`revoke_and_ack`。因此，任何承诺都可以作为*远程节点*
承诺交易;因此，本地节点需要处理两者。

在数据丢失的情况下，本地节点可能达到其不存在的状态
识别所有*远程节点的*承诺事务HTLC输出。它可以
检测数据丢失状态，因为它已经签署了事务，而且
承诺数量大于预期。如果两个节点都支持
`option_data_loss_protect`，本地节点将拥有遥控器
`per_commitment_point`，因此可以派生出自己的`remotepubkey`
交易，以打捞自己的资金。注意：在此方案中，节点
将无法挽救HTLC。

## HTLC输出处理：远程承诺，本地优惠

每个HTLC输出只能由* offerer *，超时后或者
*收件人*，如果它有付款preimage。

一旦最新块的深度等于，HTLC输出就会*超时*
或者大于HTLC`cltv_expiry`。

可能存在未被任何输出表示的HTLC：或者
因为输出被修剪为灰尘或因为远程节点有两个
*有效*承诺交易与不同的HTLC。

### 要求

本地节点：
  - 如果承诺交易HTLC输出用于支付
  原像：
    - 必须从HTLC成功事务输入中提取支付原像
    见证人。
      - 注意：输出被视为*不可撤销地解决*。
  - 如果承诺交易HTLC输出已经*超时*并且没有
  *解决*：
    - 必须*通过将其输入到方便的地址来解析*输出。
  - 对于任何承诺的HTLC，该承诺中没有输出
  交易：
    - 一旦承诺交易达到合理的深度：
      - 必须使相应的输入HTLC失败（如果有的话）。
    - 除此以外：
      - 如果没有*有效*承诺事务包含对应的输出
      HTLC：
        - 可能很快就会失败。

### 合理

如果承诺事务属于* remote *节点，则唯一的方法就是它
花费HTLC输出（使用支付preimage）是为了它使用
HTLC成功交易。

支付原像用于证明支付（当提供节点是
支付的发起人）或从中兑换相应的输入HTLC
另一个对等体（当提供节点转发支付时）。节点之后
提取付款后，不再需要关注命运了
HTLC支出交易本身。

在两种分辨率都可能的情况下（例如，当节点收到付款时）
超时后成功），要么解释是可以接受的：它就是
收件人在此之前花费它的责任。

一旦超时，本地节点需要花费HTLC输出（以防止
远程节点可以使用HTLC成功事务
使用`update_fail_htlc`返回任何相应的传入HTLC
（大概有理由`permanent_channel_failure`），如详述
[BOLT＃2]（02-peer-protocol.md＃forwarding-htlcs）。
如果传入的HTLC也是链上的，则节点只是等待它
超时，因为没有办法发出早期失败的信号。

如果HTLC太小而无法出现在*任何承诺交易*中，那么
可以安全地立即失败。除此以外，
如果HTLC不在*本地承诺交易*中，则节点需要确保
区块链重组或种族没有切换到
在节点失败之前确实包含它的承诺事务：因此
这段等待。要求输入HTLC在其之前失败
自己的超时仍然适用于上限。

## HTLC输出处理：远程承诺，远程优惠

如果收件人使用付款，则每个HTLC输出只能由收件人使用
原像。如果节点不具有原像（并且没有发现
它），提供者有责任在定时后花费HTLC输出
出。

远程HTLC输出只能由本地节点使用（如果有）
支付preimage。如果本地节点没有preimage（并且没有
发现它，远程节点负责花费HTLC输出
一旦它超时。

提供的HTLC实际上有几种可能的情况：

1. 提议者并非不可逆转地致力于此。在这种情况下，收件人
   通常不会知道原像，因为它不会转发HTLC直到
   他们完全投入了。使用preimage会显示出来
   这个收件人是最后一跳，最好让HTLC超时。
2. 提议者不可撤销地致力于提供的HTLC，但接收者
   尚未致力于传出HTLC。在这种情况下，收件人可以
   转发或等待它超时。
3. 收件人已承诺转发HTLC以换取所提供的
   HTLC。在这种情况下，收件人必须使用preimage，如果收到它
   来自即将离任的HTLC;否则，它将通过发送一个传出失去资金
   付款而不兑换收到的付款。

### 要求

本地节点：
  - 如果它收到（或已经拥有）未解决的付款原像
  它提供的HTLC输出和它已经承诺的
传出的HTLC：
    - 务必*通过将输出花费在方便的地址来解决*输出。
  - 除此以外：
    - 如果远程节点不是不可撤销地提交给HTLC：
      - 绝不能*通过花费来解决*输出。

如果没有另外解决，一旦HTLC输出到期，就会考虑
*不可撤销地解决*。

# 撤销交易关闭处理

如果任何节点试图通过广播过时的承诺交易作弊
（除了最新的承诺交易之外的任何其他承诺交易），另一个
频道中的节点可以使用其撤销私钥来声明所有资金
渠道的原始资金交易。

## 要求

一旦节点发现* it *具有的承诺事务
撤销私钥，资金交易输出*解决*。

本地节点：
  - 绝不能广播*它*已经公开的承诺交易
  `per_commitment_secret`。
  - 可能对_local节点的主输出_不采取任何操作，因为这是一个
  简单的P2WPKH输出到它自己。
    - 注意：此输出被承诺事务视为*已解决*
      本身。
  - 务必*通过使用它来解析* _remote节点的主输出_
  撤销私钥。
  - 必须以三种方式之一解析* _remote节点提供的HTLCs_：
    * 使用付款撤销私钥支付*承诺tx *。
    * 使用付款预图像（如果已知）花费*承诺tx *。
    * 如果远程节点已发布，则花费* HTLC-timeout tx *。
  - 务必*以三种方式之一解析* _local节点提供的HTLCs_：
    * 使用付款撤销私钥支付*承诺tx *。
    * 一旦HTLC超时已经过去，就花费*承诺tx *。
    * 如果远程节点已发布，则花费* HTLC-success tx *。
  - 必须通过花费来解决* _remote节点的HTLC超时事务
  使用撤销私钥。
  - 必须通过花费来解决* _remote节点的HTLC成功事务_
  使用撤销私钥。
  - 如果是，应该从交易输入见证中提取付款原像
  它还不知道。
  - 可以使用单个事务*解析*所有输出。
  - 必须处理由HTLC交易无效的交易。

## 合理

解决所有输出的单个事务将在
标准尺寸限制因为483 HTLC-per-party限制（见
[BOLT＃2]（02-peer-protocol.md＃the-open_channel-message））。

注意：如果使用单个事务，则远程节点可能无效
拒绝及时广播HTLC超时和HTLC成功交易
方式。虽然，直到所有输出都是持久性的要求
不可逆转的解决，仍然应该防止这种情况发生。 [FIXME：可能必须在这里划分和征服，因为远程节点可能能够延迟本地节点足够长的时间以避免成功的惩罚花费？ ]

## 罚款交易权重计算

惩罚交易有三种不同的脚本，具体如下
见证权重（权重计算的细节在
[附录A]（＃appendix-a-expected-weights））：

    to_local_penalty_witness：160个字节
    provided_htlc_penalty_witness：243个字节
    accepted_htlc_penalty_witness：249个字节

惩罚* txinput *本身占用41个字节，权重为164字节，
这导致每个输入的以下权重：

    to_local_penalty_input_weight：324个字节
    provided_htlc_penalty_input_weight：407个字节
    accepted_htlc_penalty_input_weight：413个字节

惩罚交易的其余部分占用4 + 1 + 1 + 8 + 1 + 34 + 4 = 53个字节
非见证数据：假设它有一个pay-to-witness-script-hash（最大的
标准输出脚本），以及2字节见证头。

除了花费这些输出之外，还可以选择处罚
花费承诺交易的'to_remote`输出（例如减少总数
支付金额）。这样做需要包含一个P2WPKH证人和一个
额外的* txinput *，导致额外的108 + 164 = 272字节。

在最坏的情况下，节点只保留传入的HTLC，并且
未发布HTLC超时事务，这会强制节点花费
承诺交易。

最大标准权重为400000字节，即HTLC的最大数量
可以在单个事务中扫描如下：

    max_num_htlcs =（400000  -  324  -  272  - （4 * 53） -  2）/ 413 = 966

因此，483个双向HTLC（包含`to_local`和
`to_remote`输出）可以在单个惩罚事务中解决。
注意：即使`to_remote`输出没有被扫描，结果也是如此
`max_num_htlcs`是967;这产生了483个HTLC的相同单向限制。

# 一般要求

一个节点：
  - 在发现花费资金交易输出的交易时
  不属于上述类别之一（相互接近，单方面）
  关闭或撤销交易关闭）：
    - 必须发送有关资金损失的警告。
      - 注意：这种流氓交易的存在意味着它的私有
      密钥已泄露，其资金可能因此而丢失。
  - 可以简单地监视交易最多工作链的内容。
    - 注意：链上HTLC应该足够稀少，速度不需要
    被认为是关键
  - 可以监控（有效）广播交易（a.k.a the mempool）。
    - 注意：观察mempool事务应该会降低延迟
    HTLC兑换。

# 附录A：预期权重

## “to_local”刑罚交易证人的预期权重

如[BOLT＃3]（03-transactions.md）中所述，此交易的见证人
是：

     <sig> 1 {OP_IF <revocationpubkey> OP_ELSE to_self_delay OP_CSV OP_DROP <local_delayedpubkey> OP_ENDIF OP_CHECKSIG}

'to_local`惩罚交易证人的*预期权重*是
计算如下：

    to_local_script：83个字节
         -  OP_IF：1个字节
             -  OP_DATA：1个字节（revocationpubkey长度）
             -  revocationpubkey：33个字节
         -  OP_ELSE：1个字节
             -  OP_DATA：1个字节（延迟长度）
             - 延迟：8个字节
             -  OP_CHECKSEQUENCEVERIFY：1个字节
             -  OP_DROP：1个字节
             -  OP_DATA：1个字节（local_delayedpubkey长度）
             -  local_delayedpubkey：33个字节
         -  OP_ENDIF：1个字节
         -  OP_CHECKSIG：1个字节

    to_local_penalty_witness：160个字节
         -  number_of_witness_elements：1个字节
         -  revocation_sig_length：1个字节
         -  revocation_sig：73个字节
         -  one_length：1个字节
         -  witness_script_length：1个字节
         -  witness_script（to_local_script）

## 'provided_htlc`惩罚交易证人的预期权重

`submitted_htlc`惩罚交易证人的*预期权重*是
计算如下（已经进行了一些计算
[BOLT＃3]（03-transactions.md））：

    provided_htlc_script：133个字节

    provided_htlc_penalty_witness：243个字节
         -  number_of_witness_elements：1个字节
         -  revocation_sig_length：1个字节
         -  revocation_sig：73个字节
         -  revocation_key_length：1个字节
         -  revocation_key：33个字节
         -  witness_script_length：1个字节
         -  witness_script（provided_htlc_script）

## “accepted_htlc”刑罚交易证人的预期权重

`accepted_htlc`惩罚交易证人的*预期权重*是
计算如下（已经进行了一些计算
[BOLT＃3]（03-transactions.md））：

    accepted_htlc_script：139个字节

    accepted_htlc_penalty_witness：249个字节
         -  number_of_witness_elements：1个字节
         -  revocation_sig_length：1个字节
         -  revocation_sig：73个字节
         -  revocationpubkey_length：1个字节
         -  revocationpubkey：33个字节
         -  witness_script_length：1个字节
         -  witness_script（accepted_htlc_script）

# 作者

[整我：]

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
