# BOLT＃8：加密和认证传输

Lightning节点之间的所有通信都是加密的
为节点之间的所有成绩单提供机密性，并通过身份验证进行身份验证
避免恶意干扰。每个节点都有一个已知的长期标识符
是比特币'secp256k1`曲线上的公钥。这个长期的公钥是
在协议中用于建立加密和经过身份验证的连接
与同行，以及验证代表广告的任何信息
一个节点

# 目录

  * [加密消息传递概述](#cryptographic-messaging-overview)
    * [经过身份验证的密钥协议握手](#authenticated-key-agreement-handshake)
    * [握手版本控制](#handshake-versioning)
    * [噪声协议实例化](#noise-protocol-instantiation)
  * [经过身份验证的密钥交换握手规范](#authenticated-key-exchange-handshake-specification)
    * [握手状态](#handshake-state)
    * [握手状态初始化](#handshake-state-initialization)
    * [握手交换](#handshake-exchange)
  * [闪电消息规范](#lightning-message-specification)
    * [加密和发送消息](#encrypting-messages)
    * [接收和解密消息](#decrypting-messages)
  * [闪电消息密钥轮换](#lightning-message-key-rotation)
  * [安全考虑因素](#security-considerations)
  * [附录A：运输测试向量](#appendix-a-transport-test-vectors)
    * [发起人测试](#initiator-tests)
    * [响应者测试](#responder-tests)
    * [邮件加密测试](#message-encryption-tests)
  * [致谢](#acknowledgments)
  * [参考](#references)
  * [作者](#authors)

## 加密消息传递概述

在发送任何Lightning消息之前，节点必须首先启动
加密会话状态，用于加密和验证所有
节点之间发送的消息。此加密会话的初始化
state完全不同于任何内部协议消息头或
约定。

两个节点之间的脚本分为两个不同的段：

1. 在任何实际数据传输之前，两个节点都参与
   经过身份验证的密钥协商握手，它基于Noise
   协议框架<sup> [2]（＃reference-2）</sup>。
2. 如果初始握手成功，则节点进入Lightning
   消息交换阶段。在Lightning消息交换阶段，全部
   消息是带有关联数据（AEAD）密文的经过身份验证的加密。

### 经过身份验证的密钥协议握手

为经过身份验证的密钥交换选择的握手是“Noise_XK”。作为一个
预消息，发起者必须知道身份公钥
响应者。这提供了一定程度的身份隐藏
响应者，因为它的静态公钥在握手期间被_never_传输。代替，
通过一系列椭圆曲线隐式地实现认证
Diffie-Hellman（ECDH）操作后跟MAC检查。

经过身份验证的密钥协议（`Noise_XK`）以三种不同的方式执行
步骤（行为）。在握手的每个动作期间，发生以下情况：一些（可能是加密的）键控
材料被发送给另一方;完全基于ECDH执行
哪个行为正在执行，结果混合到当前的一组中
加密密钥（`ck`链接密钥和`k`加密密钥）;和
发送具有零长度密文的AEAD有效载荷。因为这个有效载荷没有
长度，只发送一个MAC。将ECDH输出混合成
哈希摘要形成增量TripleDH握手。

使用噪声协议的语言，`e`和`s`（两个公钥）
表示可能是加密的密钥材料，`es`，`ee`和`se`各表示一个
两个键之间的ECDH操作。握手的布局如下：
```
    Noise_XK（s，rs）：
       < -  s
       ...
        - > e，es
       < -  e，ee
        - > s，se
```
通过线路发送的所有握手数据，包括密钥材料，都是
逐步进入会话范围的“握手摘要”，`h`。请注意
握手状态`h`在握手期间从不传输;相反，消化
用作零长度AEAD消息中的关联数据。

对发送的每条消息进行身份验证可确保中间人（MITM）未进行修改
或替换作为握手一部分发送的任何数据，如MAC
如果是这样，检查会在另一方失败。

接收方成功检查MAC表明隐含所有
到目前为止，身份验证已经成功。如果MAC检查失败
在握手过程中，然后立即建立连接
终止。

### 握手版本控制

在初始握手期间发送的每条消息都以单个前导开始
byte，表示用于当前握手的版本。版本0
表示不需要进行任何更改，而非零版本表示不需要进行更改
客户端偏离了最初在此指定的协议
文献。

客户端必须拒绝使用未知版本启动的握手尝试。

### 噪声协议实例化

噪声协议的具体实例需要定义
三个抽象加密对象：哈希函数，椭圆曲线，
和AEAD密码方案。对于闪电，`SHA-256`是
选择哈希函数，`secp256k1`作为椭圆曲线，和
`ChaChaPoly-1305`作为AEAD结构。

使用的`ChaCha20`和`Poly1305`的组成必须符合
`RFC 7539` <sup> [1]（＃reference-1）</sup>。

Lightning Light的变体的官方协议名称是
`Noise_XK_secp256k1_ChaChaPoly_SHA256`。 ASCII字符串表示
此值被散列到用于初始化起始握手的摘要中
州。如果两个端点的协议名称不同，则握手
进程立即失败。

## 经过身份验证的密钥交换握手规范

握手进行三次，每次往返1.5次。每次握手都是
_fixed_ size有效负载，没有附加任何标头或附加的元数据。
每项法案的具体规模如下：

   * **第一幕**：50个字节
   * **第二幕**：50个字节
   * **第三幕**：66个字节

### 握手状态

在整个握手过程中，每一方都维护着这些变量：

 * `ck`：**链键**。该值是所有的累积哈希值
   以前的ECDH输出。在握手结束时，`ck`用于派生
   Lightning消息的加密密钥。

 * `h`：**握手哈希**。此值是_all_的累积哈希值
   握手期间到目前为止已发送和接收的握手数据
   处理。

 * `temp_k1`，`temp_k2`，`temp_k3`：**中间键**。这些习惯了
   在每次握手结束时加密和解密零长度AEAD有效载荷
   信息。

 * `e`：派对的**短暂密钥对**。对于每个会话，节点必须生成一个
   具有强加密随机性的新短暂密钥。

 * `s`：派对的**静态密钥对**（`ls`表示本地，`rs`表示远程）

还将引用以下函数：

  * `ECDH（k，rk）`：使用执行椭圆曲线Diffie-Hellman运算
    `k`，这是一个有效的私钥，``rk`，它是`secp256k1`公钥
    在有限域内，由曲线参数定义
      * 返回值是DER压缩格式的SHA256
        生成点。

  * `HKDF（salt，ikm）`：在`RFC 5869` <sup> [3]（＃reference-3）</sup>中定义的函数，
    使用零长度的`info`字段进行评估
     * 所有`HKDF`的调用都隐含地返回64个字节
       使用提取和扩展组件的加密随机性
       `HKDF`。

  * `encryptWithAD（k，n，ad，plaintext）`：输出`encrypt（k，n，ad，plaintext）`
     * `encrypt`是对`ChaCha20-Poly1305`（IETF变体）的评价
       使用传递的参数，将nonce`n`编码为32个零位，
       接着是* little-endian * 64位值。注意：这遵循噪音
       协议约定，而不是我们的正常endian。

  * `decryptWithAD（k，n，ad，ciphertext）`：输出`decrypt（k，n，ad，ciphertext）`
     * `decrypt`是对ChaCha20-Poly1305`（IETF变体）的评价
       使用传递的参数，将nonce`n`编码为32个零位，
       接着是* little-endian * 64位值。

  * `generateKey（）`：生成并返回一个新的`secp256k1`密钥对
     * `generateKey`返回的对象有两个属性：
         * `.pub`，返回表示公钥的抽象对象
         * `.priv`，表示用于生成的私钥
           公钥
     * 对象也有一个方法：
         * `.serializeCompressed（）`

  * `a || b`表示两个字节串“a”和“b”的串联

### 握手状态初始化

在第一幕开始之前，双方都会初始化每次会议
声明如下：

 1. `h = SHA-256（protocolName）`
    * 其中`protocolName =“Noise_XK_secp256k1_ChaChaPoly_SHA256”`编码为
      ASCII字符串

 2. `ck = h`

 3. `h = SHA-256（h || prologue）`
    * 其中`prologue`是ASCII字符串：`lightning`

作为结束步骤，双方将响应者的公钥混合到
握手摘要：

 * 发起节点混合在响应节点的静态公钥中
   以比特币的DER压缩格式序列化：
   * `h = SHA-256（h || rs.pub.serializeCompressed（））`

 * 响应节点混合在其序列化的本地静态公钥中
   比特币的DER压缩格式：
   * `h = SHA-256（h || ls.pub.serializeCompressed（））`

### 握手交换

#### 第一幕

```
     - > e，es
```

Act One从发起者发送到响应者。在第一幕中，发起者
试图满足响应者的隐含挑战。完成这个
挑战，发起者必须知道响应者的静态公钥。

握手消息是_exactly_ 50字节：握手1字节
version，33个字节，用于启动器的压缩短暂公钥，
和'poly1305`标签有16个字节。

**发件人行动：**

1. `e = generateKey（）`
2. `h = SHA-256（h || e.pub.serializeCompressed（））`
     * 新生成的临时密钥被累积到运行中
       握手摘要。
3. `es = ECDH（e.priv，rs）`
     * 发起者在其新生成的短暂之间执行ECDH
       密钥和远程节点的静态公钥。
4. `ck，temp_k1 = HKDF（ck，es）`
     * 生成新的临时加密密钥，即
       用于生成身份验证MAC。
5. `c = encryptWithAD（temp_k1,0，h，zero）`
     * 其中`zero`是零长度的明文
6. `h = SHA-256（h || c）`
     * 最后，生成的密文被累积到认证中
       握手摘要。
7. 发送`m = 0 || e.pub.serializeCompressed（）|| c`通过网络缓冲区响应者。

**接收者行动：**

1. 从网络缓冲区读取_exactly_ 50个字节。
2. 将读取消息（`m`）解析为`v`，`re`和`c`：
    * 其中`v`是`m`的_first_字节，`re`是下一个33
      `m`的字节和`c`是`m`的最后16个字节
    * 远程方的短暂公钥（`re`）的原始字节是
      使用仿射坐标作为编码反序列化到曲线上的点
      通过密钥的序列化组合格式。
3. 如果`v`是一个无法识别的握手版本，那么响应者必须
    中止连接尝试。
4. `h = SHA-256（h || re.serializeCompressed（））`
    * 响应者将发起者的短暂密钥累积到认证中
      握手摘要。
5. `es = ECDH（s.priv，re）`
    * 响应者在其静态私钥和。之间执行ECDH
      发起人的短暂公钥。
6. `ck，temp_k1 = HKDF（ck，es）`
    * 将生成一个新的临时加密密钥
      不久将用于检查身份验证MAC。
7. `p = decryptWithAD（temp_k1,0，h，c）`
    * 如果此操作中的MAC检查失败，则启动器确实_not_
      知道响应者的静态公钥。如果是这种情况，那么
      响应者必须在没有任何进一步消息的情况下终止连接。
8. `h = SHA-256（h || c）`
     * 收到的密文被混合到握手摘要中。这一步有用
       确保MITM不修改有效载荷。

#### 第二幕

```
   < -  e，ee
```

第二幕从响应者发送到发起者。第二幕将_only_
如果第一幕成功，就会发生。第一幕是成功的，如果
响应者能够正确解密并检查发送的标签的MAC
第一幕结束。

握手是_exactly_ 50字节：握手版本为1字节，33
响应者的压缩短暂公钥的字节数，以及16个字节
对于`poly1305`标签。

**发件人行动：**

1. `e = generateKey（）`
2. `h = SHA-256（h || e.pub.serializeCompressed（））`
     * 新生成的临时密钥被累积到运行中
       握手摘要。
3. `ee = ECDH（e.priv，re）`
     * 其中`re`是发起人的临时密钥，已收到
       在第一幕中
4. `ck，temp_k2 = HKDF（ck，ee）`
     * 生成新的临时加密密钥，即
       用于生成身份验证MAC。
5. `c = encryptWithAD（temp_k2,0，h，zero）`
     * 其中`zero`是零长度的明文
6. `h = SHA-256（h || c）`
     * 最后，生成的密文被累积到认证中
       握手摘要。
7. 发送`m = 0 || e.pub.serializeCompressed（）|| c`通过网络缓冲区的启动器。

**接收者行动：**

1. 从网络缓冲区读取_exactly_ 50个字节。
2. 将读取消息（`m`）解析为`v`，`re`和`c`：
    * 其中`v`是`m`的_first_字节，`re`是下一个33
      `m`的字节和`c`是`m`的最后16个字节。
3. 如果`v`是一个无法识别的握手版本，那么响应者必须
    中止连接尝试。
4. `h = SHA-256（h || re.serializeCompressed（））`
5. `ee = ECDH（e.priv，re）`
    * 其中`re`是响应者的短暂公钥
    * 远程方的短暂公钥（`re`）的原始字节是
      使用仿射坐标作为编码反序列化到曲线上的点
      通过密钥的序列化组合格式。
6. `ck，temp_k2 = HKDF（ck，ee）`
     * 生成新的临时加密密钥，即
       用于生成身份验证MAC。
7. `p = decryptWithAD（temp_k2,0，h，c）`
    * 如果此操作中的MAC检查失败，则启动器必须
      终止连接而不再有任何消息。
8. `h = SHA-256（h || c）`
     * 收到的密文被混合到握手摘要中。这一步有用
       确保MITM不修改有效载荷。

#### 第三幕

```
    - > s，se
```

第三幕是经过认证的密钥协议的最后阶段
这个部分。该行为从发起者发送到响应者作为
结束步骤。如果第二幕成功，第三幕将被执行。
在第三幕中，发起者将其静态公钥传输到
响应者用_strong_前保密加密，使用累积的`HKDF`
在握手的这一点导出的秘密密钥。

握手是_exactly_ 66字节：握手版本为1字节，33
用`ChaCha20`流加密的静态公钥的字节数
密码，16个字节用于通过AEAD生成的加密公钥标记
构造，以及16个字节的最终验证标记。

**发件人行动：**

1. `c = encryptWithAD（temp_k2,1，h，s.pub.serializeCompressed（））`
    * 其中`s`是发起者的静态公钥
2. `h = SHA-256（h || c）`
3. `se = ECDH（s.priv，re）`
    * 其中`re`是响应者的短暂公钥
4. `ck，temp_k3 = HKDF（ck，se）`
    * 最终的中间共享密钥被混合到正在运行的链接密钥中。
5. `t = encryptWithAD（temp_k3,0，h，zero）`
     * 其中`zero`是零长度的明文
6. `sk，rk = HKDF（ck，零）`
     * 其中`zero`是零长度明文，
       `sk`是发起者用来加密消息的密钥
       响应者，
       和`rk`是发起者用来解密发送的消息的密钥
       响应者
     * 最终加密密钥，用于发送和
       生成在会话期间接收消息。
7. `rn = 0，sn = 0`
     * 发送和接收的随机数被初始化为0。
8. 发送`m = 0 || c || t`通过网络缓冲区。

**接收者行动：**

1. 从网络缓冲区读取_exactly_ 66个字节。
2. 将读取消息（`m`）解析为`v`，`c`和`t`：
    * 其中`v`是`m`的_first_字节，`c`是下一个49
      `m`的字节和`t`是`m`的最后16个字节
3. 如果`v`是一个无法识别的握手版本，那么响应者必须
    中止连接尝试。
4. `rs = decryptWithAD（temp_k2,1，h，c）`
     * 此时，响应者已经恢复了静态公钥
       引发剂。
5. `h = SHA-256（h || c）`
6. `se = ECDH（e.priv，rs）`
     * 其中`e`是响应者的原始短暂密钥
7. `ck，temp_k3 = HKDF（ck，se）`
8. `p = decryptWithAD（temp_k3,0，h，t）`
     * 如果此操作中的MAC检查失败，则响应者必须
       终止连接而不再有任何消息。
9. `rk，sk = HKDF（ck，零）`
     * 其中`zero`是零长度明文，
       `rk`是响应者用来解密发送消息的密钥
       由发起人，
       和`sk`是响应者用来加密消息的密钥
       发起者
     * 最终加密密钥，用于发送和
       生成在会话期间接收消息。
10. `rn = 0，sn = 0`
     * 发送和接收的随机数被初始化为0。

## 闪电消息规范

在第三幕结束时，双方都派出了加密密钥
将用于加密和解密其余部分的消息
会话。

实际的Lightning协议消息封装在AEAD密文中。
每条消息都以另一个AEAD密文为前缀，该密文对总数进行编码
以下Lightning消息的长度（不包括其MAC）。

_any_ Lightning消息的* maximum *大小不得超过`65535`字节。一个
最大尺寸`65535`简化了测试，实现了内存管理
更容易，并有助于缓解内存耗尽攻击。

为了使流量分析更加困难，长度前缀为
所有加密的Lightning消息也都是加密的。另外一个
16字节`Poly-1305`标签被添加到加密长度前缀以确保
数据包长度在飞行中没有被修改，也是为了避免
创建一个解密oracle。

线上数据包的结构类似于以下内容：

```
+ -------------------------------
| 2字节加密消息长度|
+ -------------------------------
|加密的16字节MAC |
|消息长度|
+ -------------------------------
| |
| |
|加密闪电|
|消息|
| |
+ -------------------------------
| |的16字节MAC
|闪电消息|
+ -------------------------------
```

带前缀的消息长度编码为2字节的大端整数，
总最大包长度为“2 + 16 + 65535 + 16”=“65569”字节。

### 加密和发送消息

为了加密并向网络流发送Lightning消息（`m`），
给定一个发送密钥（`sk`）和一个nonce（`sn`），完成以下步骤：

1. 设`l = len（m）`。
    * 其中`len`获取Lightning消息的字节长度
2. 将`l`序列化为2个字节，编码为big-endian整数。
3. 加密`l`（使用`ChaChaPoly-1305`，`sn`和`sk`），获得`lc`
    （18个字节）
    * nonce`sn`编码为96位little-endian数字。作为
      解码的随机数是64位，96位随机数被编码为：32位
      前导0后跟64位值。
        * 在此步骤之后，nonce`sn`必须递增。
    * 将零长度字节片作为AD（关联数据）传递。
4. 最后，使用与之相同的过程加密消息本身（`m`）
    加密长度前缀。让加密的密文称为`c`。
    * 在此步骤之后，nonce`sn`必须递增。
5. 发送`lc || c`通过网络缓冲区。

### 接收和解密消息

为了解密网络流中的_next_消息，请执行以下操作
步骤完成：

1. 从网络缓冲区读取_exactly_ 18个字节。
2. 让加密长度前缀称为`lc`。
3. 解密`lc`（使用`ChaCha20-Poly1305`，`rn`和`rk`），以获得大小
    加密数据包`l`。
    * 将零长度字节片作为AD（关联数据）传递。
    * 在此步骤之后，必须递增nonce`rn`。
4. 从网络缓冲区中读取_exactly_` l + 16`字节，并将字节设为
    被称为`c`。
5. 解密`c`（使用`ChaCha20-Poly1305`，`rn`和`rk`）来获得解密
    明文包`p`。
    * 在此步骤之后，必须递增nonce`rn`。

## 闪电消息密钥轮换

定期更改密钥并忘记以前的密钥非常有用
在以后密钥泄漏的情况下防止旧消息的解密（即
向后保密）。

对_each_键（`sk`和`rk`）_individually_执行键旋转。关键
在一方用它加密或解密1000次后（即每500条消息）旋转。
一旦nonce专用，通过旋转键可以恰当地解决这个问题
它超过1000。

键“k”的键旋转按照以下步骤执行：

1. 让`ck`成为第三幕结束时获得的链接密钥。
2. `ck'，k'= HKDF（ck，k）`
3. 将键的nonce重置为“n = 0”。
4. `k = k'`
5. `ck = ck'`

# 安全考虑因素

强烈建议现有的，常用的，经过验证的
库用于加密和解密，以避免许多可能的
实施陷阱。

# 附录A：运输测试向量

要进行可重复的测试握手，下面指定`generateKey（）`将是什么
每一方都返回（即“e.priv”的值）。请注意这一点
是违反规范，这需要随机性。

## 发起人测试

当输入此输入时，启动器应该产生给定的输出。
出于调试目的，注释反映了内部状态。

```
    name：transport-initiator成功握手
    rs.pub:0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv：0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub:0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv:0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub:0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    ＃第一幕
    #h = 0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c
    #ss = 0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3
    ＃HKDF（0x2640f52eebcd9e882958951c794250eedb28002c05d7dc2ea0f195406042caf1,0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3）
    ＃ck，temp_k1 = 0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f，0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f
    #encryptWithAD（0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f，0x000000000000000000000000,0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c，<empty>）
    #c = 0df6086551151f58b8afe6c195782c6a
    #h = 0x9d1ffbb639e7e20021d9259491dc7b160aab270fb1339ef135053f6f2cebe9ce
    输出：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输入：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    ＃re = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    #h = 0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf
    #ss = 0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47
    ＃HKDF（0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f，0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47）
    ＃ck，temp_k2 = 0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba，0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc
    ＃decryptWithAD（0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc，0x000000000000000000000000,0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf，0x6e2470b93aac583c9ef6eafca3f730ae）
    #h = 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72
    ＃第三幕
    ＃encryptWithAD（0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc，0x000000000100000000000000，0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72，0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa）
    #c = 0xb9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c3822
    #h = 0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660
    #ss = 0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766
    ＃HKDF（0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba，0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766）
    ＃ck，temp_k3 = 0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520
    #encryptWithAD（0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520,0x000000000000000000000000,0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660,<empty>）
    #t = 0x8dc68b1c466263b47fdf31e560e139ba
    输出：0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    ＃HKDF（0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01，零）
    输出：sk，rk = 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9,0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442

    name：transport-initiator act2短读取测试
    rs.pub:0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv：0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub:0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv:0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub:0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    输出：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输入：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730
    输出：错误（ACT2_READ_FAILED）

    name：transport-initiator act2 bad version test
    rs.pub:0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv：0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub:0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv:0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub:0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    输出：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输入：0x0102466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    输出：错误（ACT2_BAD_VERSION 1）

    name：transport-initiator act2坏键序列化测试
    rs.pub:0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv：0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub:0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv:0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub:0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    输出：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输入：0x0004466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    输出：错误（ACT2_BAD_PUBKEY）

    name：transport-initiator act2坏MAC测试
    rs.pub:0x028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    ls.priv：0x1111111111111111111111111111111111111111111111111111111111111111
    ls.pub:0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    e.priv:0x1212121212121212121212121212121212121212121212121212121212121212
    e.pub:0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    输出：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输入：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730af
    输出：错误（ACT2_BAD_TAG）
```

## 响应者测试

当输入该输入时，响应者应该产生给定的输出。

```
    name：transport-responder成功握手
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃re = 0x036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f7
    #h = 0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c
    #ss = 0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3
    ＃HKDF（0x2640f52eebcd9e882958951c794250eedb28002c05d7dc2ea0f195406042caf1,0x1e2fb3c8fe8fb9f262f649f64d26ecf0f2c0a805a767cf02dc2d77a6ef1fdcc3）
    ＃ck，temp_k1 = 0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f，0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f
    #decryptWithAD（0xe68f69b7f096d7917245f5e5cf8ae1595febe4d4644333c99f9c4a1282031c9f，0x000000000000000000000000,0x9e0e7de8bb75554f21db034633de04be41a2b8a18da7a319a03c803bf02b396c，0x0df6086551151f58b8afe6c195782c6a）
    #h = 0x9d1ffbb639e7e20021d9259491dc7b160aab270fb1339ef135053f6f2cebe9ce
    ＃第二幕
    ＃e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27 e.priv = 0x222222222222222222222222222222222222222222222222222222222222222222
    #h = 0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf
    #ss = 0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47
    ＃HKDF（0xb61ec1191326fa240decc9564369dbb3ae2b34341d1e11ad64ed89f89180582f，0xc06363d6cc549bcb7913dbb9ac1c33fc1158680c89e972000ecd06b36c472e47）
    ＃ck，temp_k2 = 0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba，0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc
    #encryptWithAD（0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc，0x000000000000000000000000,0x38122f669819f906000621a14071802f93f2ef97df100097bcac3ae76c6dc0bf,<empty>）
    #c = 0x6e2470b93aac583c9ef6eafca3f730ae
    #h = 0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72
    输出：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    ＃第三幕
    输入：0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    ＃decryptWithAD（0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc，0x000000000100000000000000，0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72，0xb9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c3822）
    #rs = 0x034f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    #h = 0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660
    #ss = 0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766
    ＃HKDF（0xe89d31033a1b6bf68c07d22e08ea4d7884646c4b60a9528598ccb4ee2c8f56ba，0xb36b6d195982c5be874d6d542dc268234379e1ae4ff1709402135b7de5cf0766）
    ＃ck，temp_k3 = 0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01,0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520
    #decryptWithAD（0x981a46c820fb7a241bc8184ba4bb1f01bcdfafb00dde80098cb8c38db9141520,0x000000000000000000000000,0x5dcb5ea9b4ccc755e0e3456af3990641276e1d5dc9afd82f974d90a47c918660,0x8dc68b1c466263b47fdf31e560e139ba）
    ＃HKDF（0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01，零）
    输出：rk，sk = 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9,0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442

    name：transport-responder act1短读取测试
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c
    输出：错误（ACT1_READ_FAILED）

    name：transport-responder act1 bad version test
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x01036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    输出：错误（ACT1_BAD_VERSION）

    name：transport-responder act1坏键序列化测试
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00046360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    输出：错误（ACT1_BAD_PUBKEY）

    name：transport-responder act1坏MAC测试
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6b
    输出：错误（ACT1_BAD_TAG）

    名称：transport-responder act3坏版本测试
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输出：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    ＃第三幕
    输入：0x01b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    输出：错误（ACT3_BAD_VERSION 1）

    name：transport-responder act3短读取测试
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输出：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    ＃第三幕
    输入：0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139
    输出：错误（ACT3_READ_FAILED）

    name：transport-responder act3用于密文测试的错误MAC
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输出：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    ＃第三幕
    输入：0x00c9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139ba
    输出：错误（ACT3_BAD_CIPHERTEXT）

    名称：transport-responder act3 bad rs test
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输出：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    ＃第三幕
    输入：0x00bfe3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa2235536ad09a8ee351870c2bb7f78b754a26c6cef79a98d25139c856d7efd252c2ae73c
    ＃decryptWithAD（0x908b166535c01a935cf1e130a5fe895ab4e6f3ef8855d87e9b7581c4ab663ddc，0x000000000000000000000001，0x90578e247e98674e661013da3c5c1ca6a8c8f48c90b485c0dfa1494e23d56d72，0xd7fedc211450dd9602b41081c9bd05328b8bf8c0238880f7b7cb8a34bb6d8354081e8d4b81887fae47a74fe8aab3008653）
    #rs = 0x044f355bdcb7cc0af728ef3cceb9615d90684bb5b2ca5f859ab0f0b704075871aa
    输出：错误（ACT3_BAD_PUBKEY）

    name：transport-responder act3坏MAC测试
    ls.priv = 2121212121212121212121212121212121212121212121212121212121212121
    ls.pub = 028d7500dd4c12685d1f568b4c2b5048e8534b873319f3a8daa612b469132ec7f7
    e.priv = 0x2222222222222222222222222222222222222222222222222222222222222222
    e.pub = 0x02466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f27
    ＃第一幕
    输入：0x00036360e856310ce5d294e8be33fc807077dc56ac80d95d9cd4ddbd21325eff73f70df6086551151f58b8afe6c195782c6a
    ＃第二幕
    输出：0x0002466d7fcae563e5cb09a0d1870bb580344804617879a14949cf22285f1bae3f276e2470b93aac583c9ef6eafca3f730ae
    ＃第三幕
    输入：0x00b9e3a702e93e3a9948c2ed6e5fd7590a6e1c3a0344cfc9d5b57357049aa22355361aa02e55a8fc28fef5bd6d71ad0c38228dc68b1c466263b47fdf31e560e139bb
    输出：错误（ACT3_BAD_TAG）
```

## 邮件加密测试

在此测试中，启动器发送包含“hello”的长度为5的消息
1001次为简洁起见，仅显示了六个示例输出
两个关键轮换：

    名称：传输消息测试
    CK = 0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01
    SK = 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9
    RK = 0xbb9020b8965f4df047e07f955f3c4b88418984aadc5cdb35096b9ea8fa5c3442
    #encrypt l：cleartext = 0x0005，AD = NULL，sn = 0x000000000000000000000000，sk = 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0xcf2b30ddf0cf3f80e7c35a6e6730b59fe802
    #encrypt m：cleartext = 0x68656c6c6f，AD = NULL，sn = 0x000000000100000000000000，sk = 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x473180f396d88a8fb0db8cbcf25d2f214cf9ea1d95
    输出0：0xcf2b30ddf0cf3f80e7c35a6e6730b59fe802473180f396d88a8fb0db8cbcf25d2f214cf9ea1d95
    #encrypt l：cleartext = 0x0005，AD = NULL，sn = 0x000000000200000000000000，sk = 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x72887022101f0b6753e0c7de21657d35a4cb
    #encrypt m：cleartext = 0x68656c6c6f，AD = NULL，sn = 0x000000000300000000000000，sk = 0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9 => 0x2a1f5cde2650528bbc8f837d0f0d7ad833b1a256a1
    输出1：0x72887022101f0b6753e0c7de21657d35a4cb2a1f5cde2650528bbc8f837d0f0d7ad833b1a256a1
    ＃0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31，0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8 =香港民主促进会（0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01，0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9）
    ＃0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31，0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8 =香港民主促进会（0x919219dbb2920afa8db80f9a51787a840bcf111ed8d588caf9ab4be716e42b01，0x969ab31b4d288cedf6218839b27a3e2140827047f2c0f01bf5c04435d43511a9）
    输出500：0x178cb9d7387190fa34db9c2d50027d21793c9bc2d40b1e14dcf30ebeeeb220f48364f7a4c68bf8
    输出501：0x1b186c57d44eb6de4c057c49940d79bb838a145cb528d6e8fd26dbe50a60ca2c104b56b60e45bd
    ＃0x728366ed68565dc17cf6dd97330a859a6a56e87e2beef3bd828a4c4a54d8df06，0x9e0477f9850dca41e42db0e4d154e3a098e5a000d995e421849fcd5df27882bd =香港民主促进会（0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31，0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8）
    ＃0x728366ed68565dc17cf6dd97330a859a6a56e87e2beef3bd828a4c4a54d8df06，0x9e0477f9850dca41e42db0e4d154e3a098e5a000d995e421849fcd5df27882bd =香港民主促进会（0xcc2c6e467efc8067720c2d09c139d1f77731893aad1defa14f9bf3c48d3f1d31，0x3fbdc101abd1132ca3a0ae34a669d8d9ba69a587e0bb4ddd59524541cf4813d8）
    输出1000：0x4a2f3cc3b5e78ddb83dcb426d9863d9d9a723b0337c89dd0b005d89f8d3c05c52b76b29b740f09
    输出1001：0x2ecd8c8a5629d0d02ab457a0fdd0f7b90a192cd46be5ecb6ca570bfc5e268338b1a16cf4ef2d36

# 致谢

TODO（roasbeef）;鳍

# 参考
1.  <a id="reference-1"> https://tools.ietf.org/html/rfc7539 </a>
2.  <a id="reference-2"> http://noiseprotocol.org/noise.html </a>
3.  <a id="reference-3"> https://tools.ietf.org/html/rfc5869 </a>

# 作者

整我

！[知识共享许可]（https://i.creativecommons.org/l/by/4.0/88x31.png“许可证CC-BY”）
 <br>
本作品采用[知识共享署名4.0国际许可]（http://creativecommons.org/licenses/by/4.0/）许可。
