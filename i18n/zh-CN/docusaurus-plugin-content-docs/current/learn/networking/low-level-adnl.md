# 低层级 ADNL

抽象数据报网络层（ADNL）是 TON 的核心协议，它帮助网络节点相互通信。

## 节点身份

每个节点至少应该有一个身份，可以但不必要使用多个。每个身份是一对密钥，用于执行节点间的 Diffie-Hellman。抽象网络地址是从公钥中派生出的，方法如下：`address = SHA-256(type_id || public_key)`。注意，type_id 必须以小端格式的 uint32 序列化。

## 公钥加密系统列表

| type_id | 加密系统                |
| ---------------------------- | ------------------- |
| 0x4813b4c6                   | ed25519<sup>1</sup> |

*1. 要执行 x25519，必须以 x25519 格式生成密钥对。然而，公钥通过网络是以 ed25519 格式传输的，因此你必须将公钥从 x25519 转换为 ed25519，可在[此处](https://github.com/tonstack/adnl-rs/blob/master/src/integrations/dalek.rs#L10)找到 Rust 的转换示例，以及在[此处](https://github.com/andreypfau/curve25519-kotlin/blob/f008dbc2c0ebc3ed6ca5d3251ffb7cf48edc91e2/src/commonMain/kotlin/curve25519/MontgomeryPoint.kt#L39)找到 Kotlin 的转换示例。*

## 客户端-服务器协议（ADNL over TCP）

客户端通过 TCP 连接到服务器，并发送一个 ADNL 握手包，其中包含服务器抽象地址、客户端公钥和加密的 AES-CTR 会话参数，这些参数由客户端确定。

### 握手

首先，客户端必须使用其私钥和服务器公钥执行密钥协议（例如，x25519），同时要考虑服务器密钥的 `type_id`。然后，客户端将获得 `secret`，用于在后续步骤中加密会话密钥。

再然后，客户端必须生成 AES-CTR 会话参数，一个 16 字节的 nonce 和 32 字节的密钥，分别用于 TX（客户端->服务器）和 RX（服务器->客户端）方向，并将其序列化为一个 160 字节的缓冲区，如下所示：

| 参数                            | 大小    |
| ----------------------------- | ----- |
| rx_key   | 32 字节 |
| tx_key   | 32 字节 |
| rx_nonce | 16 字节 |
| tx_nonce | 16 字节 |
| padding                       | 64 字节 |

填充(padding)的目的未知，服务器实现并不使用。建议用随机字节填充整个 160 字节缓冲区，否则攻击者可能利用被篡改的 AES-CTR 会话参数进行主动中间人攻击。

接下来的步骤是使用上述密钥协议中的 `secret` 对会话参数进行加密。为此，必须使用 AES-256 在 CTR 模式下初始化，使用 128 位大端计数器和(key, nonce)对，计算方法如下（`aes_params` 是之前构建的 160 字节缓冲区）：

```cpp
hash = SHA-256(aes_params)
key = secret[0..16] || hash[16..32]
nonce = hash[0..4] || secret[20..32]
```

加密 `aes_params` 后，记为 `E(aes_params)`，因为不再需要 AES，所以应该移除 AES。

现在我们准备将所有信息序列化到 256 字节的握手包中，并发送给服务器：

| 参数                                                          | 大小     | 说明              |
| ----------------------------------------------------------- | ------ | --------------- |
| receiver_address                       | 32 字节  | 服务器节点身份，如相应部分所述 |
| sender_public                          | 32 字节  | 客户端公钥           |
| SHA-256(aes_params) | 32 字节  | 会话参数的完整性证明      |
| E(aes_params)       | 160 字节 | 加密的会话参数         |

服务器必须使用从密钥协议中以与客户端相同的方式派生的secret解密会话参数。然后，服务器必须执行以下检查以确认协议的安全性：

1. 服务器必须拥有 `receiver_address` 对应的私钥，否则无法执行密钥协议。
2. `SHA-256(aes_params) == SHA-256(D(E(aes_params)))`，否则密钥协议失败，双方的 `secret` 不相等。

如果这些检查中的任何一个失败，服务器将立即断开连接，不对客户端作出响应。如果所有检查通过，服务器必须向客户端发送一个空数据报（见数据报部分），以证明其拥有指定 `receiver_address` 的私钥。

### 数据报

客户端和服务器都必须分别为 TX 和 RX 方向初始化两个 AES-CTR 实例。必须使用 AES-256 在 CTR 模式下使用 128 位大端计数器。每个 AES 实例使用属于它的(key, nonce)对进行初始化，可以从握手中的 `aes_params` 中获取。

发送数据报时，节点（客户端或服务器）必须构建以下结构，加密它并发送给另一方：

| 参数     | 大小               | 说明                                      |
| ------ | ---------------- | --------------------------------------- |
| length | 4 字节（LE）         | 整个数据报的长度，不包括 `length` 字段                |
| nonce  | 32 字节            | 随机值                                     |
| buffer | `length - 64` 字节 | 实际要发送给另一方的数据                            |
| hash   | 32 字节            | \`SHA-256(nonce \\ |

整个结构必须使用相应的 AES 实例加密（客户端 -> 服务器的 TX，服务器 -> 客户端的 RX）。

接收方必须获取前 4 字节，将其解密为 `length` 字段，并准确读取 `length` 字节以获得完整的数据报。接收方可以提前开始解密和处理 `buffer`，但必须考虑到它可能被故意或偶然损坏。必须检查数据报的 `hash` 以确保 `buffer` 的完整性。如果失败，则不能发出新的数据报，连接必须断开。

会话中的第一个数据报始终由服务器在成功接受握手包后发送给客户端，其实际缓冲区为空。客户端应该对其进行解密，如果失败，则应与服务器断开连接，因为这意味着服务器没有正确遵循协议，服务器和客户端的实际会话的密钥不同。

### 通信细节

如果你想深入了解通信细节，可以阅读文章 [ADNL TCP - Liteserver](/develop/network/adnl-tcp) 查看一些示例。

### 安全考虑

#### 握手填充

TON 初始团队为什么决定将此字段包含在握手中尚不清楚。`aes_params` 的完整性受 SHA-256 哈希保护，其保密性受从 `secret` 参数派生的密钥保护。可能，他们打算在某个时点迁移到 AES-CTR。为此，规范可能会扩展以包含 `aes_params` 中的特殊值，来表示节点准备使用的更新的原语。对这样的握手的响应可能会用新旧方案解密两次，以确定另一方实际使用的方案。

#### 会话参数加密密钥派生过程

如果仅从 `secret` 参数派生加密密钥，它将是静态的，因为secret是静态的。为了为每个会话派生新的加密密钥，开发人员还使用了 `SHA-256(aes_params)`，如果 `aes_params` 是随机的，则它也是随机的。然而，使用不同子数组拼接的实际密钥派生算法是被认为有问题的。

#### 数据报 nonce

数据报中 `nonce` 字段的存在原因不明确，因为即使没有它，任何两个密文也会由于 AES 的会话绑定密钥和在 CTR 模式下的加密而不同。然而，如果 nonce 不存在或可预测，则可以执行以下攻击。CTR 加密模式将区块密码（如 AES）转换为流密码，以便执行位翻转攻击(bit-flipping attack)。如果攻击者知道属于加密数据报的明文，他们可以获得纯密钥流，将其与自己的明文进行XOR，并有效地替换节点发送的消息。缓冲区的完整性受 SHA-256 哈希保护，但攻击者也可以替换它，因为完整的明文信息意味着拥有其哈希的信息。nonce 字段存在是为了防止这种攻击，所以没有 nonce 的信息，攻击者无法替换 SHA-256。

## P2P 协议（ADNL over UDP）

详细描述可以在文章 [ADNL UDP - Internode](/develop/network/adnl-udp) 中找到。

## 参考

- [开放网络，第 80 页](https://ton.org/ton.pdf)
- [TON 中的 ADNL 实现](https://github.com/ton-blockchain/ton/tree/master/adnl)

*感谢 [hacker-volodya](https://github.com/hacker-volodya) 为社区做出的贡献！*\
*此处是 GitHub 上的[原文链接](https://github.com/tonstack/ton-docs/tree/main/ADNL)。*
