# 基本概念

## 区块模型

区块链（Blockchain）本身是一种数据结构。区块大致由区块头和区块主体两部分组成。因为每个区块（头）里都存有上一个区块的 hash 值(参见`区块头`部分中的`PrevHash`)，从而形成了一种链式结构。

### 区块头

区块头的数据结构见下：

| 字节数 | 字段          | 名称                   | 类型    | 描述                                            |
| ------ | ------------- | ---------------------- | ------- | ----------------------------------------------- |
| 4      | Version       | 区块版本               | uint    | 区块版本号，目前为 `0`                          |
| 32     | PrevHash      | 上一个区块 Hash        | UInt256 | 上一个区块的 hash 值                            |
| 32     | MerkleRoot    | Merkle 树              | Uint256 | 该区块中所有交易的 Merkle 树的根                |
| 8      | Timestamp     | 时间戳                 | ulong   | 该区块生成的大致时间                            |
| 8      | Nonce         | 随机数                 | ulong   | 该区块的随机数                                  |
| 4      | Index         | 区块高度               | uint    | 创世块的高度为 0                                |
| 1      | PrimaryIndex  | 议长索引               | byte    | 本次共识过程中提案验证人的索引                  |
| 20     | NextConsensus | 下一轮共识验证人的地址 | UInt160 | 下一轮出块的三分之二以上的验证人的签名脚本 hash |
| ?      | Witness       | 见证人                 | Witness | 可执行验证脚本                                  |

区块头包含了一个区块的基本信息，用以保证这个区块能正确地连入区块链。

区块 hash 值和区块高度 Index 都可以用作区块的标识。区块 hash 值是对区块头的前 7 个数据拼合在一起进行两次 SHA256 运算而得。正常运行时，Neo 只会有一条链，且每个区块均由共识节点三分之二以上的节点确认之后才被加入到区块链中，所以区块链上的每一个块的高度唯一。区块高度必须等于前一个区块高度加 1。因为任何区块内的信息变化都会造成区块 hash 值改变，所以区块 hash 值是区块的唯一标识。区块高度不受区块内信息影响，安全性稍低，无法成为唯一标识。

`Timestamp`是每个区块的时间戳，必须晚于其前一个区块的时间戳。两个块确认的时间间隔在 15 秒左右，由系统配置文件`config.json`中，变量`MillisecondsPerBlock`所设定。

`NextConsensus`是一个多方签名脚本的 hash 值，脚本验证时，需要三分之二以上的共识节点的签名作为参数。示例脚本如下图。每一个区块，都会带有`NextConsensus`字段，锁定了参与下一轮共识的节点。在上一轮共识中，当时的议长节点根据当时的投票结果计算出了下一轮的共识节点，再生成多方签名合约，并将合约脚本 hash 值存入到提案块的`NextConsensus`字段中。若提案块最后达成共识，成为被确认的块，则本轮的共识验证人是上一个被确认的块里的多方签名合约地址之一。

`Witness`是这个区块的验证脚本。它的结构是执行脚本（`InvocationScript`）加上验证脚本(`VerificationScript`)。执行脚本就包含进行验证所需要的参数，而验证脚本就是具体验证要使用的脚本。

![](../images/BlockchainBasics//nextconsensus_script.jpeg)

### 区块

区块的数据结构见下：

| 字节数 | 字段         | 名称     | 类型          | 描述         |
| ------ | ------------ | -------- | ------------- | ------------ |
| ？     | Header       | 区块头   | Header        | 区块头       |
| ?\*?   | Transactions | 交易列表 | Transaction[] | 区块的主数据 |

除去区块头，剩下的便是由一个交易列表组成的区块主体。严格讲，区块主体以交易列表长度开始，后面罗列各条交易。此轮共识的议长将从其内存池队列中挑出通过验证的一系列交易，将其交易哈希放入一个共识消息（`PrePareRequest`）中再发送到网络里。共识过程相对复杂，请参见 [共识机制](./03-consensus-alogrithm.md) 章节。

目前，每一个块中最多有 512 笔交易。

## 哈希算法

哈希函数，又称散列算法，是一种从任何一种数据中创建数字 “指纹” 的方法。散列函数把消息或数据压缩成摘要，使得数据量变小，将数据的格式固定下来。该函数将数据打乱混合，重新创建一个叫做散列值（或哈希值）的指纹。散列值通常用一个短的随机字母和数字组成的字符串来代表。

NEO 系统中主要会用到 2 种不同的散列函数：SHA256 和 RIPEMD160。前者用于生成较长的散列值 (32 字节)，而后者用于生成较短的散列值 (20 字节)。通常生成一个对象的散列值时，会运用两次散列函数，例如要生成区块或交易的散列时，会计算两次 SHA256；生成合约地址时，会先计算脚本的 SHA256 散列，然后再计算上一个散列的 RIPEMD160 散列。

此外，区块中还会用到一种散列树的结构 (Merkle Tree)，它将每一笔交易的散列两两相接后再计算一次散列，并重复以上过程直到只剩下一个根散列 (Merkle Root)。

### RIPEMD160

RIPEMD 是一种加密哈希函数，由鲁汶大学 Hans Dobbertin, Antoon Bosselaers 和 Bart Prenee 组成的 COSIC 研究小组发布于 1996 年。

RIPEMD160 是基于 RIPEMD 改进的 160 位元版本，会产生一个 160bit 长的哈希值(可用 16 进制字符串表示)。其能表现出理想的雪崩效应(例如将 d 改成 c，即微小的变化就能产生一个完全不同的哈希值)。

NEO 使用 RIPEMD160 来生成合约脚本 160bit 的哈希值。

Example:

| 字符串      | 哈希值                                   |
| ----------- | ---------------------------------------- |
| hello world | 98c615784ccb5fe5936fbc0cbe9dfdb408d92f0f |

应用场景：

生成合约的哈希

### SHA256

SHA256 是 SHA-2 下细分出的一种算法。SHA-2：一种密码散列函数算法标准，由美国国家安全局研发，属于 SHA 算法之一，是 SHA-1 的后继者。SHA-2 下又可再分为六个不同的算法标准，包括了：SHA-224、SHA-256、SHA-384、SHA-512、SHA-512/224、SHA-512/256。

对于任意长度的消息，SHA256 都会产生一个 256bit 长的哈希值(可用 16 进制字符串表示)。

Example:

| 字符串      | 哈希值                                                           |
| ----------- | ---------------------------------------------------------------- |
| Hello World | a591a6d40bf420404a011733cfb7b190d62c65bf0bcda32b57b277d9ad9f146e |

应用场景：

- 计算合约的哈希

- 签名和确认签名

- Base58Check 编解码

- db3、NEP6 钱包的密钥的存储、导出、验证

## 钱包

钱包是 Neo 的基础组件，是用户接入 Neo 网络的载体，负责完成与之相关一系列的工作和任务。

Neo 的钱包可以自行设计和修改，但需要满足一定的规则。

### 账户

Neo 中，账户即合约，地址代表的为一段合约代码，从私钥到公钥，再到地址的流程如下图。

![](../images/BlockchainBasics/privatekey-2-publickey-address.png)

#### 私钥

私钥是一个随机生成的位于 1 和 n 之间的任意数字（n 是⼀个常数，略小于 2 的 256 次方），一般用一个 256bit (32 字节) 数表示。

在 Neo 中私钥主要采用两种编码格式：

- hexstring 格式

  hexstring 格式是将 byte[]数据使用 16 进制字符表示的字符串。

- WIF 格式

  wif 格式是在原有 32 字节数据前后添加前缀 0x80 和后缀 0x01,并做 Base58Check 编码的字符串

![](../images/BlockchainBasics/wif_format.png)

Example:

| 格式      | 数值                                                                                                                                                                      |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| byte[]    | [0xc7,0x13,0x4d,0x6f,0xd8,0xe7,0x3d,0x81,0x9e,0x82,0x75,<br>0x5c,0x64,0xc9,0x37,0x88,0xd8,0xdb,0x09,0x61,0x92,0x9e,<br>0x02,0x5a,0x53,0x36,0x3c,0x4c,0xc0,0x2a,0x69,0x62] |
| hexstring | c7134d6fd8e73d819e82755c64c93788d8db0961929e025a53363c4cc02a6962                                                                                                          |
| WIF       | L3tgppXLgdaeqSGSFw1Go3skBiy8vQAM7YMXvTHsKQtE16PBncSU                                                                                                                      |

#### 公钥

公钥是通过 ECC 算法将私钥运算得到的一个点（X, Y）。该点的 X、Y 坐标都可以用 32 字节数据表示。Neo 与比特币稍有不同，Neo 选取了 secp256r1 曲线作为其 ECC 算法的参数。在 Neo 中公钥有两种编码格式：

- 非压缩型公钥

  0x04 + X 坐标（32 字节）+ Y 坐标（32 字节）

- 压缩型公钥

  0x02/0x03+X 坐标（32 字节）

示例:

| 格式             |                                                                数值                                                                |
| ---------------- | :--------------------------------------------------------------------------------------------------------------------------------: |
| 私钥             |                                  c7134d6fd8e73d819e82755c64c93788d8db0961929e025a53363c4cc02a6962                                  |
| 公钥（压缩型）   |                                 035a928f201639204e06b4368b1a93365462a8ebbff0b8818151b74faab3a2b61a                                 |
| 公钥（非压缩型） | 045a928f201639204e06b4368b1a93365462a8ebbff0b8818151b74faab3a2b61a35dfabcb79ac492a2a88588d2f2e73f045cd8af58059282e09d693dc340e113f |

#### 地址

地址是由公钥经过一系列转换得到的一串由数字和字母构成的字符串。本节将介绍 Neo 中的公钥到地址的转换步骤。

> [!Note]
>
> Neo N3 中的地址脚本发生了变动，不再使用 Opcode.CheckSig, OpCode.CheckMultiSig 指令， 换成使用互操作服务调用，即`SysCall "Neo.Crypto.ECDsaVerify".hash2uint`, `SysCall "Neo.Crypto.ECDsaCheckMultiSig".hash2unit` 方式。

##### 普通地址

1. 通过公钥，构建一个 CheckSig 地址脚本，脚本格式，如下图

   ```
   0x0C + 0x21 + 公钥(压缩型 33字节) + 0x41 + 0x56e7b327
   ```

   ![](../images/BlockchainBasicsimage.png/account_address_script_checksign.png)

2. 计算地址脚本合约哈希 (20 字节，由地址脚本合约先做一次 SHA256 再做一次 RIPEMD160 得到)

3. 在地址脚本合约哈希前添加版本号（目前 Neo 所使用的协议版本是 53 所以对应字节为`0x35`）

4. 对字节数据做 Base58Check 编码

示例：

| 格式       |                                       数值                                       |
| ---------- | :------------------------------------------------------------------------------: |
| 私钥       |         087780053c374394a48d685aacf021804fa9fab19537d16194ee215e825942a0         |
| 压缩型公钥 |        03cdb067d930fd5adaa6c68545016044aaddec64ba39e548250eaea551172e535c        |
| 地址脚本   | 0c2103cdb067d930fd5adaa6c68545016044aaddec64ba39e548250eaea551172e535c4156e7b327 |
| 地址       |                        NNLi44dJNXtDNSBkofB48aTVYtb1zZrNEs                        |

##### 多方签名地址

1. 通过多个地址，构建一个 N-of-M CheckMultiSig 多方签名的地址脚本，脚本格式如下：

   ```
   emitPush(N) + 0x0C + 0x21 + 公钥1(压缩型 33字节)  + .... + 0x0C + 0x21 + 公钥m(压缩型 33字节)  + emitPush(M) + 0x41 + 0x9ed0dc3a
   ```

   ![](../images/BlockchainBasics/account_address_script_multi_checksign.png)

2. 计算地址脚本合约哈希(20 字节，地址脚本合约做一次 sha256 和 riplemd160 得到)

3. 在地址脚本合约哈希前添加版本号（ 目前 Neo 所使用的协议版本是 53 所以对应字节为`0x35`）

4. 对字节数据做 Base58Check 编码

示例:

| 名称       | 值                                                                                                                                                         |
| ---------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 私钥       | 087780053c374394a48d685aacf021804fa9fab19537d16194ee215e825942a0<br>9a973a470b5fd7a2c12753a1ef55db5a8c8dde42421406a28c2a994e1a1dcc8a                       |
| 压缩性公钥 | 03cdb067d930fd5adaa6c68545016044aaddec64ba39e548250eaea551172e535c<br/>036c8431cc78b33177a60b4bcc02baf60d05fee5038e7339d3a688e394c2cbd843                  |
| 地址脚本   | 110c21036c8431cc78b33177a60b4bcc02baf60d05fee5038e7339d3a688e394c2cbd8430c2103cdb067d930fd5adaa6c68545016044aaddec64ba39e548250eaea551172e535c12419ed0dc3a |
| 地址       | NZ3pqnc1hMN8EHW55ZnCnu8B2wooXJHCyr                                                                                                                         |

emitPush(number) 注意其取值范围， number 的类型为 BigInteger 时，data = number.ToByteArray()：

| Number 值          | 放入指令                         | 值                        |
| ------------------ | -------------------------------- | ------------------------- |
| -1 <= number <= 16 | OpCode.PUSH0 + (byte)(int)number | 0x10 + number             |
| data.Length == 1   | OpCode.PUSHINT8 + data           | 0x00 + data               |
| data.Length == 2   | OpCode.PUSHINT16 + data          | 0x01 + data               |
| data.Length <= 4   | OpCode.PUSHINT32 + data          | 0x02 + PadRight(data, 4)  |
| data.Length <= 8   | OpCode.PUSHINT64 + data          | 0x03 + PadRight(data, 8)  |
| data.Length <= 16  | OpCode.PUSHINT128 + data         | 0x04 + PadRight(data, 16) |
| data.Length <= 32  | OpCode.PUSHINT256 + data         | 0x05 + PadRight(data, 32) |

#### 地址的脚本哈希 (ScriptHash)

在 Neo 上创建钱包时会生成私钥、公钥、钱包地址以及对应的 ScriptHash，在不同情况下系统会根据情况处理不同类型的 Scripthash。以下是一个钱包的标准地址和 ScriptHash 的大小端序示例：

| 格式              |                    数值                    |
| ----------------- | :----------------------------------------: |
| 地址              |     NUnLWXALK2G6gYa7RadPLRiQYunZHnncxg     |
| 大端序 Scripthash | 0xed7cc6f5f2dd842d384f254bc0c2d58fb69a4761 |
| 小端序 Scripthash |  61479ab68fd5c2c04b254f382d84ddf2f5c67ced  |
| Base64 Scripthash |        YUeato/VwsBLJU84LYTd8vXGfO0=        |

要进行钱包地址与 ScriptHash 的互转，以及不同类型的 ScriptHash 之间的互转，可以使用工具 [Data Convertor](https://neo.org/converter)。

### 钱包文件

#### db3 钱包文件

db3 钱包文件是 neo 采用 sqlite 技术存储数据所使用存储文件，文件尾缀名：`.db3`。存储时，分别包含如下四张表：

- 账户表 Account

  | 字段          | 类型        | 是否必填 | 备注                             |
  | ------------- | ----------- | -------- | -------------------------------- |
  | PublicKeyHash | Binary(20)  | 是       | 主键                             |
  | Nep2key       | VarChar(58) | 是       | 按照 NEP2 标准加密的密钥 nep2Key |

- 地址表 Address

  | 字段       | 类型       | 是否必填 | 备注 |
  | ---------- | ---------- | -------- | ---- |
  | ScriptHash | Binary(20) | 是       | 主键 |

- 合约表 Contract

  | 字段          | 类型       | 是否必填 | 备注                        |
  | ------------- | ---------- | -------- | --------------------------- |
  | RawData       | VarBinary  | 是       |                             |
  | ScriptHash    | Binary(20) | 是       | 主键，外键，关联 Address 表 |
  | PublicKeyHash | Binary(20) | 是       | 索引，外键，关联 Account 表 |

- 属性表 Key

  | 字段  | 类型        | 是否必填 | 备注 |
  | ----- | ----------- | -------- | ---- |
  | Name  | VarChar(20) | 是       | 主键 |
  | Value | VarBinary   | 是       |      |

其中 Key-Value 表中，主要存储了 AES256 加密用到的 4 个属性：

- `PasswordHash`：密码的哈希，由密码做 sha256 得到

- `IV`：AES 的初始向量，随机生成

- `MasterKey`：加密密文，由 PasswordHash、 IV 对私钥做 AES256 加密得到

- `Version`：版本

> [!Note]
>
> db3 钱包采用对称加密 AES 相关技术作为钱包的加密和解密方法。db3 钱包常用在交易所钱包，方便大量的账户信息存储与检索查询。

#### NEP6 钱包文件

目前推荐使用 NEP6-JSON 钱包，安全性更高，具有跨平台特性。NEP6 钱包文件是 neo 满足 NEP6 标准的钱包存储数据所使用存储文件，文件尾缀名：`.json`。 json 文件格式如下：

```json
{
  "name": null,
  "version": "3.0",
  "scrypt": {
    "n": 16384,
    "r": 8,
    "p": 8
  },
  "accounts": [
    {
      "address": "Nf8iN8CABre87oDaDrHSnMAyVoU9jYa2FR",
      "label": null,
      "isdefault": false,
      "lock": false,
      "key": "6PYM9DxRY8RMhKHp512xExRVLeB9DSkW2cCKCe65oXgL4tD2kaJX2yb9vD",
      "contract": {
        "script": "DCEDYgBftumtbwC64LbngHbZPDVrSMrEuHXNP0tJzPlOdL5BdHR2qg==",
        "parameters": [
          {
            "name": "signature",
            "type": "Signature"
          }
        ],
        "deployed": false
      },
      "extra": null
    }
  ],
  "extra": null
}
```

> 本例中的密码为 `1`
> | 字段 | 描述 |
> | ------------------------------- | ------------------------------- |
> | name | 名称 |
> | version | 版本，目前为 3.0 |
> | scrypt（n/r/p） | scrypt 算法设置 CPU 性能的三个参数 |
> | accounts | 钱包所包含的账户的集合 |
> | account.address | 账户地址 |
> | account.label | 标题，默认为 null |
> | account.isDefault | 是否默认账户 |
> | account.lock | 是否打开 |
> | account.key | 按照 NEP2 标准加密的密钥 nep2Key |
> | account.contract | 地址脚本合约的详细内容 |
> | account.contract.script | 地址脚本合约的字节 |
> | account.contract.parameters | 地址脚本合约的参数表 |
> | account.contract.parameter.name | 地址脚本合约参数的名称 |
> | account.contract.parameter.type | 地址脚本合约参数的类型 |
> | account.contract.deployed | 是否部署 |
> | account.extra | 账户其他拓展属性 |
> | extra | 钱包其他拓展属性 |

NEP6 钱包采用了以 scrypt 为核心算法的相关技术作为钱包私钥的加密和解密方法，即 Nep2Key:

##### 加密方法

![](../images/BlockchainBasics/nep2key.png)

1. 由公钥计算地址，并获取`SHA256(SHA256(Address))`的前四个字节作为地址哈希

2. 使用 Scrypt 算法算出一个`derivedkey`，并将其 64 个字节数据分成 2 半，作为`derivedhalf1`和`derivedhalf2`。Scrypt 所使用参数如下：

   - 明文：输入的密码（UTF-8 格式）
   - 盐：地址哈希
   - n：16384
   - r：8
   - p：8
   - length：64

3. 把私钥和 derivedhalf1 做异或，然后用 derivedhalf2 对其进行 AES256 加密得到 encryptedkey

4. 按照以下格式拼接数据，并对其进行 Base58Check 编码得到 NEP2Key：

   ```
   0x01 + 0x42 + 0xe0 + addressHash + encryptedKey
   ```

##### 解密方法

1. 对 NEP2key 进行 Base58Check 解码

2. 验证解码后数据长度为 39，以及前 3 个字节（data[0-2]是否为 0x01、0x42、0xe0）

3. 取 data[3-6]作为`addresshash`

4. 把密码、addresshash 代入 Scrypt 算法，指定结果长度为 64，求出`derivedkey`

5. 把`derivedkey`前 32 字节作为`derivedhalf1`，后 32 字节作为`derivedhalf2`

6. 取 data[7-38]作为`encryptedkey`（32 字节），并用 derivedhalf2 作为初始向量对其进行 AES256 解密

7. 把解密结果与 Derivedhalf1 做异或处理求得私钥

8. 把该私钥做 ECC 求出公钥，并生成地址，对该地址做 2 次 Sha256 然后取结果的前四字节判断其是否与 addresshash 相同，相同则是正确的私钥（参考 NEP2）

相关详细技术请参照 neo 文档中的 NEP2 和 NEP6 提案。

NEP2 提案：<https://github.com/neo-project/proposals/blob/master/nep-2.mediawiki>

NEP6 提案：<https://github.com/neo-project/proposals/blob/master/nep-6.mediawiki>

### 签名

在使用钱包对交易进行签名时，Neo 采用 ECDSA 签名方法，ECC 曲线为 nistP256(或 Secp256r1), 摘要算法为 SHA256。

C# 示例代码：

```c#
        public static byte[] Sign(byte[] message, byte[] prikey, byte[] pubkey)
        {
            using (var ecdsa = ECDsa.Create(new ECParameters
            {
                Curve = ECCurve.NamedCurves.nistP256,
                D = prikey,
                Q = new ECPoint
                {
                    X = pubkey[..32],
                    Y = pubkey[32..]
                }
            }))
            {
                return ecdsa.SignData(message, HashAlgorithmName.SHA256);
            }
        }
```

示例：

| 格式      | 值                                                                                                                               |
| --------- | -------------------------------------------------------------------------------------------------------------------------------- |
| 原文      | hello world                                                                                                                      |
| 私钥      | f72b8fab85fdcc1bdd20b107e5da1ab4713487bc88fc53b5b134f5eddeaa1a19                                                                 |
| 公钥      | 031f64da8a38e6c1e5423a72ddd6d4fc4a777abe537e5cb5aa0425685cda8e063b                                                               |
| signature | b1855cec16b6ebb372895d44c7be3832b81334394d80bec7c4f00a9c1d9c3237541834638d11ad9c62792ed548c9602c1d8cd0ca92fdd5e68ceea40e7bcfbeb2 |

### 钱包功能

| 功能名称       | 描述                                                        |
| -------------- | ----------------------------------------------------------- |
| 导入钱包文件   | 从指定钱包文件导入账户信息                                  |
| 导出钱包文件   | 将账户信息导出到指定的钱包文件，如 db3 文件、NEP6 json 文件 |
| 解锁钱包       | 验证用户密码来防止账户信息泄露                              |
| 修改密码       | 修改用户密码                                                |
| 生成私钥       | 推荐使用安全的随机数发生器                                  |
| 导入私钥       | 从 WIF 字符串或者数字证书导入私钥到钱包中                   |
| 导出私钥       | 导出账户的私钥                                              |
| 生成公钥       | 使用 ECC 算法从私钥得到公钥                                 |
| 生成地址       | 从私钥生成地址                                              |
| 导入地址       | 添加新的地址到钱包中                                        |
| 导出地址       | 导出账户地址                                                |
| 导入离线数据包 | 从`chain.acc`文件加载区块数据来减少同步时间                 |
| 导出离线数据包 | 导出区块数据到`chain.acc`文件                               |
| 同步区块数据   |                                                             |
| 转账           | 转账资产到其他地址                                          |
| 签名           | 对数据签名，比如交易                                        |
| 提取 gas       | 提取通过持有 neo 新分配的 gas                               |
| 获取余额       | 显示该钱包中所有账户的资产余额                              |
| 获取交易       | 显示该钱包中产生的交易历史                                  |
| 构造多签合约   | 构造多签合约                                                |
| 扩展           |                                                             |
| 部署智能合约   | 部署智能合约                                                |
| 测试智能合约   | 测试智能合约                                                |

### 钱包软件

#### 全节点钱包

全节点钱包包含所有区块数据的备份，保存了所有链上数据，并且参与 p2p 网络通信，所以占用存储空间较大。

Neo-CLI 和 Neo-GUI 都是全节点钱包，相关信息请参考 [Neo 节点](https://docs.neo.org/docs/zh-cn/node/introduction.html)

### SPV 钱包

SPV (Simplified Payment Verification, 简单支付验证)钱包不同于全节点钱包，它不会存储所有的区块数据，仅存储区块头数据，并使用布隆过滤器和梅克尔树算法。它主要在移动端 App 或者轻节点中使用，因为它能有效得节省存储空间。

如果要开发 SPV 钱包，请参考 Neo 网络协议接口。

使用：

    1. SPV钱包发送一个布隆过滤器到全节点，全节点加载该布隆过滤器。

    2. SPV钱包发送布隆过滤器的参数到全节点，全节点加载该布隆过滤器的参数。（可选）

    3. SPV钱包从全节点查询交易，全节点在使用布隆过滤器过滤后返回交易数据和构造的梅克尔树路径。

    4. SPV钱包使用梅尔克树路径来验证交易数据。

    5. SPV钱包发送`clear the bloom filter`指令给全节点，全节点清除该布隆过滤器。

## 交易

Neo 区块去掉区块头部分就是一串交易构成的区块主体，因而交易是整个 Neo 系统的基础部件。钱包、智能合约、账户和交易相互作用但最终都转化成交易被记入区块链中。在 Neo 的 P2P 网络传输中，信息被打包成`InvPayload`信息包来传送（Inv 即 Inventory）。不同信息包有自己需要的特定数据，`InventoryType.Tx` 用于标识网络中的 InvPayload 信息包内装的是交易数据。

### 数据结构

在 Neo 网络中，交易的数据结构如下所示：

| 字段              | 类型                   | 说明                         |
| ----------------- | ---------------------- | ---------------------------- |
| `version`         | byte                   | 交易版本号，目前为 0         |
| `nonce`           | uint                   | 随机数                       |
| `sysfee`          | long                   | 支付给网络的资源费用         |
| `netfee`          | long                   | 支付给验证人打包交易的费用   |
| `validUntilBlock` | uint                   | 交易的有效期                 |
| `signers`         | Signer[]               | 发送方以及限制签名的作用范围 |
| `attributes`      | TransactionAttribute[] | 交易所具备的额外特性         |
| `script`          | byte[]                 | 交易的合约脚本               |
| `witnesses`       | Witness[]              | 用于验证交易的脚本列表       |

#### version

version 属性允许对交易结构进行更新，使其具有向后兼容性。 目前版本为 0。

#### signers

signers 中的第一个字段为交易发起账户的地址哈希。由于 Neo N3 弃用了 UTXO 模型，仅保留有账户余额模型，原生资产 NEO 和 GAS 的转账交易统一为 NEP-17 资产操作方式，因此交易结构中不再记录 inputs 和 outputs 字段，通过 signers 字段来跟踪交易的发送方。

signers 中余下的字段定义了签名的作用范围。当 checkwitness 用于交易验证时，除交易发送者 sender 外，其他的 signers 都需要定义其签名的作用范围。详情请参见 [签名作用域](#signature-scope)。

| 字段               | 说明                                                  | 类型             |
| ------------------ | ----------------------------------------------------- | ---------------- |
| `Account`          | 账户脚本哈希                                          | `UInt160`        |
| `Scopes`           | 指定签名的作用范围                                    | `WitnessScope`   |
| `AllowedContracts` | 签名可验证的合约脚本数组                              | `UInt160[]`      |
| `AllowedGroups`    | 签名可验证的合约组公钥                                | `ECPoint[]`      |
| `Rules`            | 当 `scopes` 设置为`WitnessRules` 时，填写签名规则数组 | `WitnessRules[]` |

### sysfee

系统费用取决于交易脚本的大小，数量和 NeoVM 指令类型。每一个指令所对应的费用，请参考[opcode 费用](../虚拟机#费用)。Neo N3 取消了每笔交易 10 GAS 的免费额度，系统费用总额受合约脚本的指令数量和指令类型影响。计算公式如下所示：

![](../images/transaction/system_fee.png)

其中，_OpcodeSet_ 为指令集，𝑂𝑝𝑐𝑜𝑑𝑒𝑃𝑟𝑖𝑐𝑒<sub>𝑖</sub>为第 _i_ 种指令的费用，𝑛<sub>𝑖</sub>为第 _i_ 种指令在合约脚本中的执行次数。

#### netfee

网络费是用户向 Neo 网络提交交易时支付的费用，作为共识节点的出块奖励。每笔交易的网络费存在一个基础值，用户支付的网络费需要大于或等于此基础值，否则交易无法通过验证。基础网络费计算公式如下所示：

![network fee](../images/transaction/network_fee.png)

其中，*VerificationCost*为虚拟机验证交易签名执行的指令相对应的费用，*tx.Length*为交易数据的字节长度，*FeePerByte*为交易每字节的费用，目前为 0.00001GAS。

#### attributes

根据具体的交易类型允许向交易添加额外的属性。 对于每个属性，必须指定使用类型，以及外部数据和外部数据的大小。

每个交易最多可以添加 16 个属性。

#### script

在 NeoVM 上执行的脚本，决定了交易的效果。

#### witnesses

witnesses 属性用于验证交易的有效性和完整性。Witness 即“见证人”， 包含两个属性。

| 字段                 | 说明                         |
| -------------------- | ---------------------------- |
| `InvocationScript`   | 执行脚本，向验证脚本传递参数 |
| `VerificationScript` | 验证脚本                     |

可以为每个交易添加多个见证人，也可以使用具有多方签名的见证人。

##### 执行脚本

执行脚本可以通过

`0x0C(PUSHDATA1) + 0x40(数据长度为64字节) + 签名`

添加签名。重复此步，可以为多方签名合约推送多个签名。

##### 验证脚本

验证脚本，常见为地址脚本，包括普通地址脚本和多签地址脚本，该地址脚本可以从钱包账户中直接获取，其构造方式，请参考[钱包-地址](wallets.md#地址)；

也可以为自定义的鉴权合约脚本。

### 交易序列化

除 IP 地址和端口号外，Neo 中所有变长的整数类型都使用小端序存储。交易序列化时将按以下字段顺序执行序列化操作：

| 字段              | 说明                                                                  |
| ----------------- | --------------------------------------------------------------------- |
| `version`         | -                                                                     |
| `nonce`           | -                                                                     |
| `systemFee`       | -                                                                     |
| `networkFee`      | -                                                                     |
| `validUntilBlock` | -                                                                     |
| `signers`         | 需先序列化数组长度`WriteVarInt(length)`，之后再分别序列化数组各个元素 |
| `attributes`      | 需先序列化数组长度`WriteVarInt(length)`，之后再分别序列化数组各个元素 |
| `script`          | 需先序列化数组长度`WriteVarInt(length)`，之后再序列化字节数组         |
| `witnesses`       | 需先序列化数组长度`WriteVarInt(length)`之后再分别序列化数组各个元素   |

> [!Note]
>
> WriteVarInt(value) 是根据 value 的值，存储非定长类型, 根据取值范围决定存储大小。
> | Value 值范围 | 存储类型 |
> |--------------------|--------------|
> | value < 0xFD | byte(value) |
> | value <= 0xFFFF | 0xFD + ushort(value) |
> | value <= 0xFFFFFFFF | 0xFE + uint(value) |
> | value > 0xFFFFFFFF | 0xFF + value |

### 交易签名

交易签名是对交易本身的数据（不包含签名数据，即 witnesses 部分）进行 ECDSA 方法签名，然后填入交易体中的`witnesses`。

交易 JSON 格式示例，其中 script 与 witnesses 字段使用 Base64 替代原有的 Hexstring 编码：

```Json
{
  "hash": "0xd2b24b57ea05821766877241a51e17eae06ed66a6c72adb5727f8ba701d995be",
  "size": 265,
  "version": 0,
  "nonce": 739807055,
  "sender": "NMDf1XCbioM7ZrPZAdQKQt8nnx3fWr1wdr",
  "sys_fee": "9007810",
  "net_fee": "1264390",
  "valid_until_block": 2102402,
  "signers": [{
    "account": "0xdf93ea5a0283c01e8cdfae891ff700faad70500e",
    "scopes": "FeeOnly"
  },
  {
    "account": "0xdf93ea5a0283c01e8cdfae891ff700faad70500e",
    "scopes": "CalledByEntry"
  }],
  "attributes": [],
  "script": "EQwUDlBwrfoA9x+Jrt+MHsCDAlrqk98MFA5QcK36APcfia7fjB7AgwJa6pPfE8AMCHRyYW5zZmVyDBSJdyDYzXb08Aq/o3wO3YicII/em0FifVtSOA==",
  "witnesses": [{
    "invocation": "DEDy/g4Lt+FTMBHHF84TSVXG9aSNODOjj0aPaJq8uOc6eMzqr8rARqpB4gWGXNfzLyh9qKvE++6f6XoZeaEoUPeH",
    "verification": "DCECCJr46zTvjDE0jA5v5jrry4Wi8Wm7Agjf6zGH/7/1EVELQQqQatQ="
  }]
}
```

### 签名作用域

在 Neo Legacy 中，交易签名是全局有效的，所有合约均可使用用户签名。为了让用户能更细粒度地控制签名的作用范围，Neo N3 新增了签名作用域（WitnessScope），对交易结构中的 signers 字段进行了变更，可实现签名只限于验证指定合约的功能，防止未经授权的合约随意使用用户签名。

#### Scopes

在构造交易时，需要指定 `signers` 参数中的 `scopes` 字段，其定义了签名的作用范围，包含的类型如下表所示。

| 值   | 名称              | 说明                                                                                                       |
| ---- | ----------------- | ---------------------------------------------------------------------------------------------------------- |
| 0x00 | `None`            | 仅对交易签名，不允许任何合约使用该签名                                                                     |
| 0x01 | `CalledByEntry`   | 签名只限于由 Entry 脚本调用的合约脚本，建议作为钱包的默认签名作用。                                        |
| 0x10 | `CustomContracts` | 自定义合约，在指定的合约中可以使用该签名。可与 CalledByEntry 配合使用。                                    |
| 0x20 | `CustomGroups`    | 自定义合约组，在指定的合约组中可以使用该签名。可与 CalledByEntry 配合使用。                                |
| 0x80 | `Global`          | 签名全局有效，所有合约均可使用签名。风险极高，合约有可能会转移地址中的所有资产，仅在极其信任该合约时选择。 |
| 0x40 | `WitnessRules`    | 用户自己指定验签规则和范围，参见下节说明。                                                                 |

为了更好地说明各类型，我们假设一条合约调用链为： **[entry]->[合约 A]->[合约 B]->[合约 C]...->[Target]**

Target 合约调用 CheckWitness 验签，且用户赋予了签名。当 scopes 分别设置为以下值时，验签情况如下：

- `None` - **Target** 合约位于任何位置时都不允许验签通过
- `Global` - **Target** 合约位于任何位置时都允许验签通过
- `CallByEntry` - **Target** 合约位于 **entry** 或 **合约 A** 时才允许验签通过
- `CustomContracts` - 此时用户需要额外指定一个合约列表 **CustomContracts**，仅当 **Target** 合约属于 **CustomContracts** 中时允许验签通过
- `CustomGroups` - 此时用户需要额外指定一个 Group 公钥列表 **CustomGroups**，仅当 **Target** 合约带有 **CustomGroups** 中任意一个公钥认证时，允许验签通过

#### WitnessRule

Action(Allow|Deny) 和 Condition (判断条件)

执行逻辑：先执行判断条件，如果符合，则返回 Action，返回 Allow 代表验签成功，Deny 代表验签失败。

##### WitnessCondition 判断条件

- Boolean：true|false

  “expression” = <bool>

  ```
  // 等价于 WitnessScope.Global
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "Boolean",
                  "expression": true
              }
          }
      ]
  }
  ```

- Not: 逻辑非，对其它条件求反

  “expression”=<Condition>

  ```
  // 只有当前合约不是 0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5 才允许使用签名
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "Not",
                  "expression": {
                      "type": "ScriptHash",
                      "hash": "0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5"
                  }
              }
          }
      ]
  }

  ```

- And：逻辑与，连接其它条件求与

  “expressions”=<Condition[]>

  ```
  // 只有当前合约是 0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5 且在入口处调用时才允许使用签名
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "And",
                  "expressions": [{
                          "type": "ScriptHash",
                          "hash": "0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5"
                      }, {
                          "type": "CalledByEntry"
                      }
                  ]
              }
          }
      ]
  }
  ```

- Or：逻辑或，连接其它条件求或

  “expressions”=<Condition[]>

  ```
  // 只有当前合约是 0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5 或在入口处调用时才允许使用签名
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "Or",
                  "expressions": [{
                          "type": "ScriptHash",
                          "hash": "0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5"
                      }, {
                          "type": "CalledByEntry"
                      }
                  ]
              }
          }
      ]
  }
  ```

- ScriptHash：验证当前合约 hash 是否匹配，相当于 CustomContracts

  “hash”= <UInt160>

  ```
  // 只允许合约 0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5 使用签名
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "ScriptHash",
                  "hash": "0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5"
              }
          }
      ]
  }
  ```

- Group：验证当前合约的公钥是否匹配，相当于 CustomGroups

  “group”=<ECPoint>

  ```
  // 只允许经过公钥 021821807f923a3da004fb73871509d7635bcc05f41edef2a3ca5c941d8bbc1231认证的合约使用签名
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "Group",
                  "group": "021821807f923a3da004fb73871509d7635bcc05f41edef2a3ca5c941d8bbc1231"
              }
          }
      ]
  }
  ```

- CalledByEntry：验证当前合约是否为 entry 调用，相当于 CallByEntry

  ```
  // 等价于 WitnessScope.CallByEntry
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "CalledByEntry"
              }
          }
      ]
  }
  ```

- CalledByContract：验证当前合约的上一级合约 hash 是否匹配

  “hash”=<UInt160>

  ```
  // 只允许上级合约是 0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5 时使用签名
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "CalledByContract",
                  "hash": "0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5"
              }
          }
      ]
  }
  ```

- CalledByGroup：验证当前合约的上一级合约公钥是否匹配

  “group”=<UInt160>

  ```
  // 只允许上级合约是公钥 021821807f923a3da004fb73871509d7635bcc05f41edef2a3ca5c941d8bbc1231 认证过的合约时使用签名
  {
      "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
      "scopes": "WitnessRules",
      "rules": [{
              "action": "Allow",
              "condition": {
                  "type": "CalledByGroup",
                  "group": "021821807f923a3da004fb73871509d7635bcc05f41edef2a3ca5c941d8bbc1231"
              }
          }
      ]
  }
  ```

#### 示例

该字段目前只能通过 SDK 构造交易时定义。为了帮助理解，可参考 JSON 格式的代码示例：

```json
{
  "account": "NdUL5oDPD159KeFpD5A9zw5xNF1xLX6nLT",
  "scopes": "WitnessRules",
  "rules": [
    {
      "action": "Allow",
      "condition": {
        "type": "Not",
        "expression": {
          "type": "And",
          "expressions": [
            {
              "type": "ScriptHash",
              "hash": "0xef4073a0f2b305a38ec4050e4d3d28bc40ea63f5"
            },
            {
              "type": "CalledByEntry"
            }
          ]
        }
      }
    }
  ]
}
```
