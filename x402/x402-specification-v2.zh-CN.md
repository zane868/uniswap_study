# X402 协议规范

**协议版本**： 2

**文档范围**

本规范定义用于互联网原生支付的 x402 核心协议，涵盖以下内容：

- **协议基础**：付款要求的格式、支付载荷结构和核心消息结构。
- **Facilitator 接口**：支付验证和结算的标准 API。
- **支付方案**：可扩展的支付方式，包括 `exact`、`upto` 和 `batch-settlement`，参见 `specs/schemes/`。
- **安全注意事项**：防止重放攻击，并尽量减少对其他参与方的信任依赖。

**不在范围内的内容**：本规范不包含：

- 特定传输方式的实现，详见传输规范。
- 具体实现模式，详见应用说明。
- 特定框架的集成。
- 客户端预算管理。
- 会话处理机制。

**架构**

x402 由三个核心部分组成：

1. **类型**：核心数据结构，如 `PaymentRequirements`、`PaymentPayload` 和 `SettlementResponse`，独立于传输机制和支付方案。
2. **逻辑**：支付构造和验证逻辑，取决于支付方案（如 exact、upto、batch-settlement）和网络（如 EVM、Solana）。
3. **表示方式**：如何传输支付数据和发出支付信号，取决于 HTTP、MCP、A2A 等传输机制。

**1. 概述**

x402 是一种开放支付标准，允许客户端为外部资源付费。协议统一了消息格式和支付流程，可在不同传输层上实现，为不同支付方案、网络和传输方式提供一致的支付机制。

本规范依据 [x402 仓库](https://github.com/x402-foundation/x402)中的实现和文档编写，旨在完整描述 x402 协议，且不依赖特定实现。

**2. 核心支付流程**

x402 将支付接入标准的请求—响应流程：

1. **客户端请求**：客户端向资源服务器发起请求。
2. **要求付款**：如果请求未附带有效的支付信息，服务端返回需要付款的信号及付款要求。
3. **提交支付授权**：客户端在后续请求中提交已签名的支付授权。
4. **结算响应**：服务端验证支付授权，并发起区块链结算。

上述流程描述默认的 `authorization` 支付流程：执行资源请求前先验证支付，执行成功后再结算。支付方案也可以定义先结算再执行的流程，详见第 6.1 节“资产转移方式与支付流程模型”。

**3. 协议参与方**

x402 协议涉及三个主要参与方：

- **资源服务器（Resource Server）**：提供受保护资源的收费服务，例如 API、内容和数据。
- **客户端（Client）**：请求访问受保护资源的应用或智能体。
- **支付处理服务（Facilitator）**：负责支付验证和区块链结算的服务。

**4. 响应类型**

x402 定义了以下标准响应类型：

- **成功（Success）**：请求成功，支付已验证并完成结算。
- **需要付款（Payment Required）**：访问资源需要付款。
- **无效请求（Invalid Request）**：支付载荷或付款要求无效。
- **服务端错误（Server Error）**：处理支付时发生服务端错误。

各传输方式的实现会将这些响应映射为对应的信号，例如 HTTP 状态码或 JSON-RPC 错误码。

**5. 数据类型**

本节定义 x402 使用的核心数据结构。这些结构完全独立于传输机制和支付方案，所有传输方式和方案都使用相同结构，区别只在于数据如何表示（传输层），以及采用何种验证和结算逻辑（方案层）。

**5.1 PaymentRequired 数据结构**

**5.1.1 JSON 载荷**

资源服务器要求付款时，会返回包含 `PaymentRequired` 对象的付款信号。该对象放在何处，由传输规范决定。在 HTTP 中，标准位置是 `PAYMENT-REQUIRED` 响应头，其值为 Base64 编码的对象，参见 [HTTP 付款要求信号](https://github.com/x402-foundation/x402/blob/main/specs/transports-v2/http.md#payment-required-signaling)。

`PaymentRequired` 对象示例：

```json
{
  "x402Version": 2,
  "error": "PAYMENT-SIGNATURE header is required",
  "resource": {
    "url": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json",
    "serviceName": "Example Market Data",
    "tags": ["market-data", "finance"],
    "iconUrl": "https://api.example.com/icon.png"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:84532",
      "amount": "10000",
      "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60,
      "extra": {
        "name": "USDC",
        "version": "2"
      }
    }
  ],
  "extensions": {}
}
```

**5.1.2 字段说明**

`PaymentRequired` 包含以下字段：

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | -------- | -------- | ---------------------------------------------------------------------- |
| `x402Version` | `number` | 必填 | 协议版本标识，必须为 2 |
| `error` | `string` | 可选 | 说明为什么需要付款的错误信息，供人阅读 |
| `resource` | `object` | 必填 | 描述受保护资源的 ResourceInfo 对象 |
| `accepts` | `array` | 必填 | 付款要求对象数组，列出可接受的支付方式 |
| `extensions` | `object` | 可选 | 协议扩展数据 |

`accepts` 数组中的每个 `PaymentRequirements` 对象包含：

| 字段名 | 类型 | 必填 | 说明 |
| ------------------- | -------- | -------- | --------------------------------------------------------------------------------------------------------------------------- |
| `scheme` | `string` | 必填 | 支付方案标识，例如 `exact` |
| `network` | `string` | 必填 | CAIP-2 格式的区块链网络标识，例如 `eip155:84532` |
| `amount` | `string` | 必填 | 要求支付的金额，以代币最小单位表示 |
| `asset` | `string` | 必填 | 代币合约地址，或法币的 ISO 4217 货币代码 |
| `payTo` | `string` | 必填 | 收款钱包地址或角色常量，例如 `merchant` |
| `maxTimeoutSeconds` | `number` | 必填 | 允许完成支付的最长时间 |
| `extra` | `object` | 可选 | 额外信息。协议保留键为 `assetTransferMethod` 和 `paymentFlow`，详见第 6.1 节；其他键由各支付方案定义 |

`ResourceInfo` 对象包含：

| 字段名 | 类型 | 必填 | 说明 |
| --------------- | --------------- | -------- | -------------------------------------------------------------------------------------------------------------------- |
| `url` | `string` | 必填 | 受保护资源的 URL |
| `description` | `string` | 可选 | 供人阅读的资源说明 |
| `mimeType` | `string` | 可选 | 预期响应的 MIME 类型 |
| `serviceName` | `string` | 可选 | 提供资源的服务名称，供人阅读；使用可打印 ASCII 字符，最多 32 个字符 |
| `tags` | `array[string]` | 可选 | 服务主题标签，用于发现时筛选；最多 5 项，每项使用可打印 ASCII 字符，最多 32 个字符 |
| `iconUrl` | `string` | 可选 | 服务图标的绝对 `https`／`http` URL，最多 2048 个字符 |

`Extensions` 对象是一个键值映射，键为扩展标识符，值采用以下统一结构：

| 字段名 | 类型 | 必填 | 说明 |
| ---------- | -------- | -------- | -------------------------------------------------------- |
| `info` | `object` | 必填 | 服务端提供的扩展专属数据 |
| `schema` | `object` | 必填 | 定义 `info` 预期结构的 JSON Schema |

扩展用于在核心支付机制之外提供模块化的可选功能。服务端在 `PaymentRequired` 中声明支持的扩展，客户端在 `PaymentPayload` 中回传。客户端必须至少保留收到的信息，可以追加信息，但不得删除或覆盖已有信息。

**5.2 PaymentPayload 数据结构**

**5.2.1 JSON 结构**

客户端以 JSON 形式在支付载荷字段中提供支付授权：

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json"
  },
  "accepted": {
    "scheme": "exact",
    "network": "eip155:84532",
    "amount": "10000",
    "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
    "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
    "maxTimeoutSeconds": 60,
    "extra": {
      "name": "USDC",
      "version": "2"
    }
  },
  "payload": {
    "signature": "0x2d6a7588d6acca505cbf0d9a4a227e0c52c6c34008c8e8986a1283259764173608a2ce6496642e377d6da8dbbf5836e9bd15092f9ecab05ded3d6293af148b571c",
    "authorization": {
      "from": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
      "to": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "value": "10000",
      "validAfter": "1740672089",
      "validBefore": "1740672154",
      "nonce": "0xf3746613c2d920b5fdabc0856f2aeb2d4f88ee6037b8cc5d04a71a4462f13480"
    }
  },
  "extensions": {}
}
```

**5.2.2 字段说明**

`PaymentPayload` 包含以下字段：

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | -------- | -------- | ------------------------------------------------------------------- |
| `x402Version` | `number` | 必填 | 协议版本标识 |
| `resource` | `object` | 可选 | 描述当前访问资源的 ResourceInfo 对象 |
| `accepted` | `object` | 必填 | 表示所选支付方式的 PaymentRequirements 对象 |
| `payload` | `object` | 必填 | 特定于支付方案的支付数据 |
| `extensions` | `object` | 可选 | 协议扩展数据 |

`accepted` 字段包含一个 `PaymentRequirements` 对象，详见第 5.1.2 节。

`payload` 字段包含特定支付方案的数据。例如，在 EVM exact 方案中，它包含：

| 字段名 | 类型 | 必填 | 说明 |
| --------------- | -------- | -------- | ----------------------------------- |
| `signature` | `string` | 必填 | 授权的 EIP-712 签名 |
| `authorization` | `object` | 必填 | EIP-3009 授权参数 |

`Authorization` 对象包含以下字段：

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | -------- | -------- | ----------------------------------------------- |
| `from` | `string` | 必填 | 付款方钱包地址 |
| `to` | `string` | 必填 | 收款方钱包地址 |
| `value` | `string` | 必填 | 支付金额，以最小单位表示 |
| `validAfter` | `string` | 必填 | 授权生效时间，Unix 时间戳 |
| `validBefore` | `string` | 必填 | 授权到期时间，Unix 时间戳 |
| `nonce` | `string` | 必填 | 用于防止重放攻击的 32 字节随机 nonce |

**5.3 SettlementResponse 数据结构**

**5.3.1 JSON 结构**

支付结算后，服务端以 JSON 形式在支付响应字段中返回交易详情：

```json
{
  "success": true,
  "transaction": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "network": "eip155:84532",
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66"
}
```

**5.3.2 字段说明**

`SettleResponse` 包含以下字段：

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | --------- | -------- | --------------------------------------------------------------------- |
| `success` | `boolean` | 必填 | 支付结算是否成功 |
| `errorReason` | `string` | 可选 | 结算失败的原因，成功时省略 |
| `payer` | `string` | 可选 | 付款方钱包地址 |
| `transaction` | `string` | 必填 | 区块链交易哈希；未广播交易时为空字符串。当 `errorReason` 为 `settlement_pending` 时，必须（MUST）非空，详见[第 9 节“错误处理”](#9-error-handling) |
| `network` | `string` | 必填 | CAIP-2 格式的区块链网络标识 |
| `amount` | `string` | 可选 | 实际结算金额，以最小单位表示；不适用时省略 |
| `extensions` | `object` | 可选 | 协议扩展数据 |

**5.4 VerifyResponse 数据结构**


**5.4.2 字段说明**

`VerifyResponse` 包含以下字段：

| 字段名 | 类型 | 必填 | 说明 |
| --------------- | --------- | -------- | ------------------------------------------------------- |
| `isValid` | `boolean` | 必填 | 支付授权是否有效 |
| `invalidReason` | `string` | 可选 | 无效原因，授权有效时省略 |
| `payer` | `string` | 可选 | 付款方钱包地址 |
| `extensions` | `object` | 可选 | 协议扩展数据 |
| `extra` | `object` | 可选 | 特定于支付方案的附加数据 |

Facilitator 可以（MAY）通过独立于 `extensions` 的 `extensionResponses` 提供扩展处理结果。该字段由传输侧通道填充，不会序列化后发送给购买方。详见第 7.2.1 节。

**6. 支付方案（逻辑层）**

本节介绍 x402 支持的支付方案。支付方案定义在特定支付网络上如何构造、验证和结算支付，独立于底层传输机制。

每种方案定义：

- 如何构造 `PaymentPayload` 中的 `payload` 字段。
- 结算和验证流程。
- `PaymentRequirements` 的 `extra` 字段要求：协议保留键见第 6.1 节，其余键由各支付方案定义。

各支付方案及其在不同网络上的具体规则，包括 `exact`、`upto`、`batch-settlement` 和 `auth-capture`，见 [`specs/schemes/`](https://github.com/x402-foundation/x402/tree/main/specs/schemes/)。

**6.1 资产转移方式与支付流程模型**

`assetTransferMethod` 标识某个机制（即特定网络上的支付方案）**如何授权或转移资金**。例如，EVM `exact` 可使用 `eip3009` 或 `permit2`，XRPL `exact` 可使用 `sequence` 或 `ticketSequence`。允许的字符串值由各机制定义；本协议只保留键名，不统一规定全部资产转移方式（ATM）的取值。不同网络上的机制在语义一致时，可以（MAY）复用同一 ATM 字符串。`extra.assetTransferMethod` 和 `extra.paymentFlow` 是 `PaymentRequirements.extra` 中的协议保留键，客户端和服务端必须（MUST）按本节规定解释，不能将其视为由方案私有定义、无需解析的普通字段。

支付方案的区别不仅在于如何构造和验证支付，还在于结算发生在资源请求执行**之前还是之后**。每个机制都要**针对各个 `assetTransferMethod`** 声明支持的支付流程及默认流程，同时定义方案级默认 `assetTransferMethod`，用于省略 `extra.assetTransferMethod` 的情况。解析出的流程决定：围绕受保护请求的执行，应调用 Facilitator 的只读 `/verify`（第 7.1 节）、提交支付状态的 `/settle`（第 7.2 节）中的哪些接口，以及调用顺序。流程可以（MAY）省略 `/verify`，例如下文的 `upfront` 和 `escrow`。

省略 `extra.assetTransferMethod` 或 `extra.paymentFlow` 时，解析时采用该机制的默认值。如果解析出的支付流程不是 `authorization`，`PaymentRequired` 中就必须（MUST）提供 `accepts[].extra.paymentFlow`，让客户端无需了解特定方案的细节，也能判断是否会在业务处理前承诺资金，例如区分 SVM upto 默认采用的 `escrow` 与 EVM upto 默认采用的 `authorization`。`authorization` 可以（MAY）省略，也可以显式指定。资源服务器必须（MUST）拒绝不支持的 `assetTransferMethod` 与支付流程组合。客户端不得（MUST NOT）为无法识别的 `paymentFlow` 构造支付，选择方案时应当（SHOULD）跳过对应的 `accepts[]` 条目。同一请求同时提供 `authorization`（业务处理后结算）和 `upfront` 或 `escrow`（业务处理前结算）时，客户端应当（SHOULD）优先选择 `authorization`。

定义的流程如下：

| 流程 | 执行顺序 | 说明 |
| --------------------- | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| `authorization`（默认） | verify → resource → settle → respond | 资源请求执行前进行只读验证；仅在请求成功执行后才转移资金。 |
| `upfront` | settle → resource → respond | 资源请求执行前先持久化提交支付，使服务端先获得支付确定性。该流程不包含 Facilitator `/verify`，由 settle 确认支付有效性。没有拉取式结算原语的网络需要采用此流程。 |
| `escrow` | settle → resource → settle → respond | 第一次 settle 提交押金或资金上限，然后执行资源请求，第二次 settle 记录最终费用。该流程不包含 Facilitator `/verify`，第一次 settle 即为执行资源请求前的检查。 |

不变条件：执行资源请求之前，必须（MUST）至少进行一次验证或结算检查。不得在未进行任何检查的情况下执行资源请求。

**7. Facilitator 接口**

Facilitator 提供用于支付验证和结算的 HTTP REST API。资源服务器可以将区块链操作交给受信任的第三方，也可以自行部署这些接口。x402 核心协议不依赖特定传输方式，但目前 Facilitator API 统一使用 HTTP 接口。

**7.1 POST /verify**

验证支付授权，但不在区块链上执行交易。`/verify` 是**只读接口**，用于验证支付状态，不得（MUST NOT）提交支付状态或写入链上状态。只有解析出的支付流程包含 `/verify` 时，资源服务器才调用它，详见第 6.1 节；`upfront` 和 `escrow` 不包含此步骤。

**请求（Exact 方案）：**

```json
{
  "x402Version": 2,
  "paymentPayload": {
    /* PaymentPayload 数据结构 */
  },
  "paymentRequirements": {
    /* PaymentRequirements 数据结构 */
  }
}
```

具体数据示例：

```json
{
  "x402Version": 2,
  "paymentPayload": {
    "x402Version": 2,
    "resource": {
      "url": "https://api.example.com/premium-data",
      "description": "Access to premium market data",
      "mimeType": "application/json"
    },
    "accepted": {
      "scheme": "exact",
      "network": "eip155:84532",
      "amount": "10000",
      "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "maxTimeoutSeconds": 60,
      "extra": {
        "name": "USDC",
        "version": "2"
      }
    },
    "payload": {
      "signature": "0x...",
      "authorization": {
        "from": "0x...",
        "to": "0x...",
        "value": "10000",
        "validAfter": "1740672089",
        "validBefore": "1740672154",
        "nonce": "0x..."
      }
    }
  },
  "paymentRequirements": {
    "scheme": "exact",
    "network": "eip155:84532",
    "amount": "10000",
    "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
    "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
    "maxTimeoutSeconds": 60,
    "extra": {
      "name": "USDC",
      "version": "2"
    }
  }
}
```

**成功响应：**

```json
{
  "isValid": true,
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66"
}
```

**错误响应：**

```json
{
  "isValid": false,
  "invalidReason": "insufficient_funds",
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66"
}
```

**7.2 POST /settle**

持久化提交该请求的支付状态，使资源服务器能够确认支付已落实。通常通过更新网络账本完成，例如广播交易，但不一定需要写入链上状态。对于客户端预付方式，settle 可以（MAY）在只读检查账本或后端状态后，将支付证明绑定到请求，例如消耗一次挑战或将某笔交易标记为已使用。一次 settle 也不一定意味着最终扣款；根据支付方案和流程，它可以（MAY）建立托管、记录费用或转移资金，详见第 6.1 节。

**请求**：与 `/verify` 接口结构相同，包含 `paymentPayload` 和 `paymentRequirements`。

> **注**：虽然请求结构相同，某些支付方案在验证和结算阶段对同一字段的解释不同。例如，`upto` 方案中，`paymentRequirements.amount` 在验证时表示授权金额上限，在结算时表示实际结算金额。具体规则见各方案规范。

> **注**：同一笔支付可以（MAY）多次调用 `/settle`。例如，`escrow` 流程在资源请求执行前结算押金，执行后结算最终费用。定义多次结算的方案必须（MUST）说明 Facilitator 如何根据载荷区分各次结算。由于客户端通常只签署一份载荷，这种区分一般由服务端决定，例如使用方案专属的 `step` 字段；Facilitator 也可以（MAY）根据网络账本状态推断当前步骤。

**成功响应：**

```json
{
  "success": true,
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "transaction": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "network": "eip155:84532"
}
```

**错误响应：**

```json
{
  "success": false,
  "errorReason": "insufficient_funds",
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "transaction": "",
  "network": "eip155:84532"
}
```

**7.2.1 扩展响应侧通道**

Facilitator 可以（MAY）在验证和结算响应中，通过特定传输方式的侧通道返回扩展处理结果。这些结果**不属于** JSON 响应体，也**不会**转发给购买方。

在 HTTP 中，该侧通道使用 `EXTENSION-RESPONSES` 响应头：

| 属性 | 值 |
| -------- | ----- |
| 响应头名称 | `EXTENSION-RESPONSES` |
| 响应头内容 | 以扩展名为键的 JSON 对象，经 Base64 编码 |

每个键对应一个扩展的处理结果对象。具体载荷结构由扩展规范定义，例如 `specs/extensions/bazaar.md` 中的 `bazaar`。

**7.3 GET /supported**

返回 Facilitator 支持的支付方案、网络和扩展列表。

**响应：**

```json
{
  "kinds": [
    {
      "x402Version": 2,
      "scheme": "exact",
      "network": "eip155:84532"
    },
    {
      "x402Version": 2,
      "scheme": "exact",
      "network": "eip155:8453"
    },
    {
      "x402Version": 2,
      "scheme": "exact",
      "network": "eip155:43113"
    },
    {
      "x402Version": 2,
      "scheme": "exact",
      "network": "eip155:43114"
    }
  ],
  "extensions": [],
  "signers": {
    "eip155:*": ["0x1234567890abcdef1234567890abcdef12345678"],
    "solana:*": ["CKPKJWNdJEqa81x7CkZ14BVPiY6y16Sxs7owznqtWYp5"]
  }
}
```

**7.3.1 SupportedResponse 字段**

| 字段名 | 类型 | 必填 | 说明 |
| ------------ | -------- | -------- | ------------------------------------------------------------------------ |
| `kinds` | `array` | 必填 | 支持的支付类型对象数组 |
| `extensions` | `array` | 必填 | Facilitator 已实现的扩展标识符数组 |
| `signers` | `object` | 必填 | CAIP-2 模式（如 `eip155:*`）到公开签名者地址的映射 |

`kinds` 数组中的每个 `SupportedKind` 对象包含：

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | -------- | -------- | ---------------------------------------------------------- |
| `x402Version` | `number` | 必填 | 支持的协议版本，V2 使用 2 |
| `scheme` | `string` | 必填 | 支付方案标识，例如 `exact` |
| `network` | `string` | 必填 | CAIP-2 格式的区块链网络标识 |
| `extra` | `object` | 可选 | 特定于支付方案的额外配置 |

**8. 发现 API**

x402 提供资源发现机制，让客户端查找支持 x402 的资源。基于这一机制，可以建立称为 Bazaar（集市）的市场，供用户发现和访问收费 API 及数字服务。

发现功能目前通过 HTTP REST API 实现，但查到的资源可以使用 x402 支持的任意传输方式。

8.1 GET /discovery/resources

列出 Bazaar 中可发现的 x402 资源。

**请求参数：**

| 参数 | 类型 | 必填 | 说明 | 默认值 |
| --------- | -------- | -------- | ------------------------------------------- | ------- |
| `type` | `string` | 可选 | 按资源类型筛选，例如 `http` | - |
| `payTo` | `string` | 可选 | 按收款地址筛选 | - |
| `scheme` | `string` | 可选 | 按支付方案筛选，例如 `exact` | - |
| `network` | `string` | 可选 | 按支付网络筛选，例如 `eip155:8453` | - |
| `extensions` | `string` | 可选 | 按资源包含的扩展键筛选 | - |
| `limit` | `number` | 可选 | 最多返回多少条结果，范围为 1–100 | 20 |
| `offset` | `number` | 可选 | 分页时跳过的结果数量 | 0 |

**响应：**

```json
{
  "x402Version": 2,
  "items": [
    {
      "resource": "https://api.example.com/premium-data",
      "type": "http",
      "x402Version": 2,
      "accepts": [
        {
          "scheme": "exact",
          "network": "eip155:84532",
          "amount": "10000",
          "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
          "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
          "maxTimeoutSeconds": 60,
          "extra": {
            "name": "USDC",
            "version": "2"
          }
        }
      ],
      "lastUpdated": "2025-08-09T01:07:04.005Z"
    }
  ],
  "pagination": {
    "limit": 10,
    "offset": 0,
    "total": 1
  }
}
```

**8.2 GET /discovery/search**

搜索语义和响应结构由以下 Bazaar 扩展规范定义：
`specs/extensions/bazaar.md`。该接口的行为属于扩展功能。

**8.3 发现结果中的资源字段**

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | -------- | -------- | --------------------------------------------------------------- |
| `resource` | `string` | 必填 | 收费资源的 URL 或标识符 |
| `type` | `string` | 必填 | 资源类型，目前 HTTP 接口使用 `http` |
| `x402Version` | `number` | 必填 | 资源支持的协议版本 |
| `accepts` | `array` | 必填 | 指定支付方式的 PaymentRequirements 对象数组 |
| `lastUpdated` | `string` | 必填 | 资源最后更新时间，ISO 8601 格式时间戳 |
| `extensions` | `object` | 可选 | 与发现的资源相关的额外扩展载荷 |

**8.4 Bazaar 概念**

Bazaar 是供用户发现和访问 x402 资源的市场生态，主要功能包括：

- **资源发现**：按类别、服务提供方或付款要求查找 API 和服务。
- **支付透明**：使用前即可查看价格和支付方式。
- **提供方信息**：了解服务提供方及其提供的服务。
- **动态更新**：可随时添加、更新或移除资源。

**8.5 使用示例**

```bash
# 列出金融数据 API
GET /discovery/resources?type=http&limit=10

# 搜索天气 API
GET /discovery/search?query=weather+APIs&type=http&limit=5

# 继续分页搜索（需要服务端支持）
GET /discovery/search?query=financial+data&limit=10&cursor=eyJwYWdlIjoyfQ==
```

<a id="9-error-handling"></a>

**9. 错误处理**

x402 定义了一组标准错误码，Facilitator 或资源服务器可在响应中返回，帮助客户端判断支付失败的原因并采取相应措施。

- **`insufficient_funds`**: 客户端的代币余额不足，无法完成支付。
- **`invalid_exact_evm_payload_authorization_valid_after`**: 支付授权尚未生效，当前时间早于 validAfter。
- **`invalid_exact_evm_payload_authorization_valid_before`**: 支付授权已过期，当前时间晚于 validBefore。
- **`invalid_exact_evm_payload_authorization_value_mismatch`**: 支付金额与要求的金额不完全一致。
- **`invalid_exact_evm_payload_signature`**: 支付授权签名无效或签署不正确。
- **`invalid_exact_evm_payload_recipient_mismatch`**: 收款地址与付款要求不一致。
- **`invalid_network`**: 不支持指定的区块链网络。
- **`invalid_payload`**: 支付载荷格式错误或包含无效数据。
- **`invalid_payment_requirements`**: 付款要求对象无效或格式错误。
- **`invalid_scheme`**: 不支持指定的支付方案。
- **`unsupported_scheme`**: Facilitator 不支持该支付方案。
- **`invalid_x402_version`**: 不支持该协议版本。
- **`invalid_transaction_state`**: 区块链交易失败或被拒绝。
- **`unexpected_verify_error`**: 验证支付时发生意外错误。
- **`unexpected_settle_error`**: 支付结算时发生意外错误。
- **`settlement_pending`**: 结算交易已广播，但尚无法确认交易结果，例如节点／RPC 错误，或等待交易回执超时。Facilitator 可以（MAY）返回这一**非终态**错误码，因为交易仍可能在链上确认。`SettleResponse` 使用此 `errorReason` 时，必须（MUST）携带非空的 `transaction`（已广播交易的哈希）和 `network`，便于调用者先核对链上结果，再决定是否重试。

**10. 安全注意事项**

**10.1 防止重放攻击**

x402 通过多层机制防止重放攻击：

- **EIP-3009 Nonce**：每份授权包含一个唯一的 32 字节 nonce，防止重复使用。
- **链上保护**：EIP-3009 合约在智能合约层面阻止重复使用 nonce。
- **时间限制**：授权具有明确的有效时间范围。
- **签名验证**：所有授权都由付款方通过密码学签名确认。

**10.2 身份认证集成**

协议支持接入身份认证系统，例如 Sign-In with Ethereum（SIWE），以便根据认证身份定价，为已验证用户提供折扣或特殊访问条件。

**11. 实现说明**

**11.1 网络标识符**

x402 V2 的网络标识使用 CAIP-2（链无关改进提案）格式：`namespace:reference`。

**格式**：`{namespace}:{reference}`，例如 Base 主网为 `eip155:8453`。

非区块链网络也建议遵循 CAIP-2 格式，例如 `ach:us`、`sepa:eu`。

参考实现同时支持 EVM 和 Solana 网络，例如：

- **`eip155:84532`**: Base Sepolia 测试网
- **`eip155:8453`**: Base 主网
- **`eip155:43113`**: Avalanche Fuji 测试网
- **`eip155:43114`**: Avalanche 主网
- **`solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`**: Solana 主网
- **`solana:EtWTRABZaYq6iMfeYKouRu166VU2xqa1`**: Solana 开发网

**11.2 支持的资产**

支持的代币因网络而异：

**EVM 网络：**
- 实现 EIP-3009（授权转账）的 ERC-20 代币。
- 例如 USDC。

**Solana:**
- 任意 SPL 代币。
- Token2022 程序发行的代币。

具体支持哪些代币，取决于 Facilitator 的服务能力及相应网络上的部署情况。

**12. 用例与应用**

x402 可用于多种互联网收费场景。核心协议基于 HTTP 原生机制设计，且不绑定特定区块链；具体实现可根据场景需要调整。

### 12.1 AI 智能体集成

AI 智能体可以使用 x402 自主购买资源和服务，协议支持：

- 访问资源时**自动处理支付**。
- 通过 Facilitator 服务**发现资源**。
- **预算管理**和支出限制，由具体实现提供。
- 对操作进行分组的**关联追踪**，由具体实现提供。
- **多传输方式支持**，使智能体可通过 HTTP API、MCP 工具及其他协议层交互。

### 12.2 面向用户的应用

应用可以使用 x402 实现：

- **基于会话的访问**，例如限时订阅。
- **按次付费内容**，例如文章、视频、下载和工具。
- 按调用次数收费的**资源变现**。
- **基于身份认证的定价**，例如向已验证用户提供折扣。
- 支持 Web、桌面和 AI 应用的**跨协议支付**。

### 12.3 传输方式支持

x402 可以接入多个传输层：

- **HTTP**：Web API、REST 服务和服务端框架，例如 Express.js、FastAPI、Next.js。
- **MCP（模型上下文协议）**：AI 智能体工具和资源。
- **A2A（智能体间协议）**：智能体之间的直接支付。
- **自定义协议**：任何基于请求—响应的系统都可以实现 x402 支付流程。

### 12.4 服务端框架

x402 可以接入常见框架：

- **Express.js**：`require_payment()` 中间件。
- **FastAPI／Flask**：对应框架的中间件。
- **Hono**：支持边缘运行环境。
- **Next.js**：全栈集成。
- **ai/agents**：AI 智能体和 MCP 框架。

### 12.5 客户端库

不同传输方式的客户端都可以接入 x402 支付：

- **HTTP 客户端**：浏览器中的 axios／fetch、Python 中的 httpx／requests，以及命令行中的 curl。
- **MCP 客户端**：ai/agents MCP 客户端。
- **A2A**：x402_a2a（Python）。
- **自定义集成**：应用自行实现的支付处理。

### 12.6 进阶模式

协议可以支持更复杂的收费策略：

- 根据用户认证信息或使用情况**动态定价**。
- 通过**会话管理**按时间控制访问权限。
- 访问多个资源时进行**批量支付**。
- 基于小额支付构建**订阅模式**。

_注：预算管理、关联追踪、会话处理等具体模式的实现细节见应用说明和实现指南。特定传输方式的实现细节见相应传输规范。_

---

## 版本历史

| 版本 | 日期 | 变更 | 作者 |
| ------- | ----------- | ----------------------------------------------------------------- | ------------------------- |
| v2.0 | 2025-12-9 | 协议 V2：采用 CAIP-2 网络标识，重构 PaymentPayload／Required，分离 ResourceInfo，支持扩展 | x402 团队 |
| v0.2 | 2025-10-3 | 重新设计为不依赖特定传输方式的架构 | Ethan Niser |
| v0.1 | 2025-8-29 | 初始草稿 | [根据仓库内容整理] |
