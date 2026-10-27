# X402 协议规范

**协议版本**： 1

**文档范围**

本规范定义用于互联网原生支付的 x402 核心协议，涵盖以下内容：

- **协议基础**：付款要求的格式、支付载荷结构和核心消息结构。
- **Facilitator 接口**：支付验证和结算的标准 API。
- **支付方案**：可扩展的支付方式，目前支持 `exact` 方案。
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
2. **逻辑**：支付构造和验证逻辑，取决于支付方案（如 exact、deferred）和网络（如 EVM、Solana）。
3. **表示方式**：如何传输支付数据和发出支付信号，取决于 HTTP、MCP、A2A 等传输机制。

**1. 概述**

x402 是一种开放支付标准，允许客户端为外部资源付费。协议统一了消息格式和支付流程，可在不同传输层上实现，为不同支付方案、网络和传输方式提供一致的支付机制。

本规范依据 [Coinbase x402 仓库](https://github.com/x402-foundation/x402)中的实现和文档编写，旨在完整描述 x402 协议，且不依赖特定实现。

**2. 核心支付流程**

x402 将支付接入标准的请求—响应流程：

1. **客户端请求**：客户端向资源服务器发起请求。
2. **要求付款**：如果请求未附带有效的支付信息，服务端返回需要付款的信号及付款要求。
3. **提交支付授权**：客户端在后续请求中提交已签名的支付授权。
4. **结算响应**：服务端验证支付授权，并发起区块链结算。

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

**5.1 PaymentRequirementsResponse 数据结构**

**5.1.1 JSON 载荷**

资源服务器要求付款时，会返回需要付款的信号，以及包含付款要求的 JSON 载荷。例如：

```json
{
  "x402Version": 1,
  "error": "X-PAYMENT header is required",
  "accepts": [
    {
      "scheme": "exact",
      "network": "base-sepolia",
      "maxAmountRequired": "10000",
      "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
      "resource": "https://api.example.com/premium-data",
      "description": "Access to premium market data",
      "mimeType": "application/json",
      "outputSchema": null,
      "maxTimeoutSeconds": 60,
      "extra": {
        "name": "USDC",
        "version": "2"
      }
    }
  ]
}
```

**5.1.2 字段说明**

`PaymentRequirementsResponse` 包含以下字段：

**所有字段均为必填。**

| 字段名 | 类型 | 说明 |
| ---------- | ---- | ----------- |
| `x402Version` | `number` | 协议版本标识 |
| `error` | `string` | 说明为什么需要付款的错误信息，供人阅读 |
| `accepts` | `array` | 付款要求对象数组，列出可接受的支付方式 |

`accepts` 数组中的每个 `PaymentRequirements` 对象包含：

| 字段名 | 类型 | 必填 | 说明 |
| ------------------- | -------- | -------- | ------------------------------------------------------------------------ |
| `scheme` | `string` | 必填 | 支付方案标识，例如 `exact` |
| `network` | `string` | 必填 | 区块链网络标识，例如 `base-sepolia`、`ethereum-mainnet` |
| `maxAmountRequired` | `string` | 必填 | 要求支付的金额，以代币最小单位表示 |
| `asset` | `string` | 必填 | 代币合约地址 |
| `payTo` | `string` | 必填 | 收款钱包地址 |
| `resource` | `string` | 必填 | 受保护资源的 URL |
| `description` | `string` | 必填 | 供人阅读的资源说明 |
| `mimeType` | `string` | 可选 | 预期响应的 MIME 类型 |
| `outputSchema` | `object` | 可选 | 描述响应格式的 JSON Schema |
| `maxTimeoutSeconds` | `number` | 必填 | 允许完成支付的最长时间 |
| `extra` | `object` | 可选 | 特定于支付方案的附加信息 |

**5.2 PaymentPayload 数据结构**

**5.2.1 JSON 结构**

客户端以 JSON 形式在支付载荷字段中提供支付授权：

```json
{
  "x402Version": 1,
  "scheme": "exact",
  "network": "base-sepolia",
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
  }
}
```

**5.2.2 字段说明**

`PaymentPayload` 包含以下字段：

**所有字段均为必填。**

| 字段名 | 类型 | 说明 |
| ------------- | -------- | ------------------------------------------------------------------------ |
| `x402Version` | `number` | 协议版本标识，必须为 1 |
| `scheme` | `string` | 支付方案标识，例如 `exact` |
| `network` | `string` | 区块链网络标识，例如 `base-sepolia`、`ethereum-mainnet` |
| `payload` | `object` | 支付数据对象 |

`payload` 字段包含一个 `SchemePayload` 对象，存放特定支付方案的数据：

**所有字段均为必填。**

| 字段名 | 类型 | 说明 |
| --------------- | -------- | ----------------------------------- |
| `signature` | `string` | 授权的 EIP-712 签名 |
| `authorization` | `object` | EIP-3009 授权参数 |

`Authorization` 对象包含以下字段：

**所有字段均为必填。**

| 字段名 | 类型 | 说明 |
| ------------- | -------- | ----------------------------------------------- |
| `from` | `string` | 付款方钱包地址 |
| `to` | `string` | 收款方钱包地址 |
| `value` | `string` | 支付金额，以最小单位表示 |
| `validAfter` | `string` | 授权生效时间，Unix 时间戳 |
| `validBefore` | `string` | 授权到期时间，Unix 时间戳 |
| `nonce` | `string` | 用于防止重放攻击的 32 字节随机 nonce |

**5.3 SettlementResponse 数据结构**

**5.3.1 JSON 结构**

支付结算后，服务端以 JSON 形式在支付响应字段中返回交易详情：

```json
{
  "success": true,
  "transaction": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "network": "base-sepolia",
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66"
}
```

**5.3.2 字段说明**

`SettlementResponse` 包含以下字段：

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | --------- | -------- | --------------------------------------------------------------- |
| `success` | `boolean` | 必填 | 支付结算是否成功 |
| `errorReason` | `string` | 可选 | 结算失败的原因，成功时省略 |
| `transaction` | `string` | 必填 | 区块链交易哈希，结算失败时为空字符串 |
| `network` | `string` | 必填 | 区块链网络标识 |
| `payer` | `string` | 必填 | 付款方钱包地址 |

**6. 支付方案（逻辑层）**

本节介绍 x402 支持的支付方案。支付方案定义在特定支付网络上如何构造、验证和结算支付，独立于底层传输机制。

每种方案定义：

- 如何构造 `PaymentPayload` 中的 `payload` 字段。
- 结算和验证流程。
- `PaymentRequirements` 的 `extra` 字段中特定于该方案的要求。

**6.1 Exact 方案（EVM 概述）**

`exact` 方案使用 EIP-3009（授权转账），安全地转移指定数量的 ERC-20 代币，付款方无需自行支付 Gas。

**6.1.1 EIP-3009 授权**

授权遵循 EIP-3009 对 `transferWithAuthorization` 的定义：

```javascript
const authorizationTypes = {
  TransferWithAuthorization: [
    { name: "from", type: "address" },
    { name: "to", type: "address" },
    { name: "value", type: "uint256" },
    { name: "validAfter", type: "uint256" },
    { name: "validBefore", type: "uint256" },
    { name: "nonce", type: "bytes32" },
  ],
};
```

**6.1.2 验证步骤**

Facilitator 执行以下验证：

1. **签名验证**：验证 EIP-712 签名有效，且确实由付款方签署。
2. **余额检查**：确认付款方有足够的代币余额完成转账。
3. **金额检查**：确保支付金额达到或超过要求的金额。
4. **有效期检查**：确认授权处于有效时间范围内。
5. **参数匹配**：确认授权参数与原始付款要求一致。
6. **交易模拟**：模拟 `transferWithAuthorization` 交易，确认其能够成功执行。

**6.1.3 结算**

结算时，使用支付载荷中的签名和授权参数，调用 ERC-20 合约的 `transferWithAuthorization` 函数。

**6.2 Exact 方案（SVM 概述）**

在 Solana（SVM）上，`exact` 方案通过 SPL 代币的 `TransferChecked` 实现。关键验证要求包括：

- 严格限制指令排列：计算单元上限、计算单元价格、可选的 ATA 创建指令，然后是 TransferChecked。
- 确保 Facilitator 的手续费支付账户不出现在任何指令的账户列表中，也不是转账的 `authority` 或 `source`。
- 限制计算单元价格，防止滥用手续费。
- 验证目标 ATA 是否符合由 `payTo`／`asset` 确定的 PDA 及账户存在性规则。
- 要求转账 `amount` 与 `maxAmountRequired` 完全相等。

完整的 SVM 规则见 `specs/schemes/exact/scheme_exact_svm.md`。

**7. Facilitator 接口**

Facilitator 提供用于支付验证和结算的 HTTP REST API。资源服务器可以将区块链操作交给受信任的第三方，也可以自行部署这些接口。x402 核心协议不依赖特定传输方式，但目前 Facilitator API 统一使用 HTTP 接口。

**7.1 POST /verify**

验证支付授权，但不在区块链上执行交易。

**请求（Exact 方案）：**

```json
{
  "x402Version": 1,
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
  "x402Version": 1,
  "paymentPayload": {
    "x402Version": 1,
    "scheme": "exact",
    "network": "base-sepolia",
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
    "network": "base-sepolia",
    "maxAmountRequired": "10000",
    "resource": "https://api.example.com/premium-data",
    "description": "Access to premium market data",
    "mimeType": "application/json",
    "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
    "maxTimeoutSeconds": 60,
    "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
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

向区块链广播交易，执行已经验证的支付。

**请求**：与 `/verify` 接口相同。

**成功响应：**

```json
{
  "success": true,
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "transaction": "0x1234567890abcdef1234567890abcdef1234567890abcdef1234567890abcdef",
  "network": "base-sepolia"
}
```

**错误响应：**

```json
{
  "success": false,
  "errorReason": "insufficient_funds",
  "payer": "0x857b06519E91e3A54538791bDbb0E22373e36b66",
  "transaction": "",
  "network": "base-sepolia"
}
```

**7.3 GET /supported**

返回 Facilitator 支持的支付方案和网络列表。

**响应：**

```json
{
  "kinds": [
    {
      "x402Version": 1,
      "scheme": "exact",
      "network": "base-sepolia"
    },
    {
      "x402Version": 1,
      "scheme": "exact",
      "network": "base"
    },
    {
      "x402Version": 1,
      "scheme": "exact",
      "network": "avalanche-fuji"
    },
    {
      "x402Version": 1,
      "scheme": "exact",
      "network": "avalanche"
    },
    {
      "x402Version": 1,
      "scheme": "exact",
      "network": "iotex"
    }
  ]
}
```

**8. 发现 API**

x402 提供资源发现机制，让客户端查找支持 x402 的资源。基于这一机制，可以建立称为 Bazaar（集市）的市场，供用户发现和访问收费 API 及数字服务。

发现功能目前通过 HTTP REST API 实现，但查到的资源可以使用 x402 支持的任意传输方式。

8.1 GET /discovery/resources

列出 Bazaar 中可发现的 x402 资源。

**请求参数：**

| 参数 | 类型 | 必填 | 说明 | 默认值 |
| --------- | -------- | -------- | ------------------------------------------- | ------- |
| `type` | `string` | 可选 | 按资源类型筛选，例如 `http` | - |
| `limit` | `number` | 可选 | 最多返回多少条结果，范围为 1–100 | 20 |
| `offset` | `number` | 可选 | 分页时跳过的结果数量 | 0 |

**响应：**

```json
{
  "x402Version": 1,
  "items": [
    {
      "resource": "https://api.example.com/premium-data",
      "type": "http",
      "x402Version": 1,
      "accepts": [
        {
          "scheme": "exact",
          "network": "base-sepolia",
          "maxAmountRequired": "10000",
          "resource": "https://api.example.com/premium-data",
          "description": "Access to premium market data",
          "mimeType": "application/json",
          "payTo": "0x209693Bc6afc0C5328bA36FaF03C514EF312287C",
          "maxTimeoutSeconds": 60,
          "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
          "extra": {
            "name": "USDC",
            "version": "2"
          }
        }
      ],
      "lastUpdated": 1703123456,
      "metadata": {
        "category": "finance",
        "provider": "Example Corp"
      }
    }
  ],
  "pagination": {
    "limit": 10,
    "offset": 0,
    "total": 1
  }
}
```

**8.2 发现结果中的资源字段**

| 字段名 | 类型 | 必填 | 说明 |
| ------------- | -------- | -------- | --------------------------------------------------------------- |
| `resource` | `string` | 必填 | 收费资源的 URL 或标识符 |
| `type` | `string` | 必填 | 资源类型，目前 HTTP 接口使用 `http` |
| `x402Version` | `number` | 必填 | 资源支持的协议版本 |
| `accepts` | `array` | 必填 | 指定支付方式的 PaymentRequirements 对象数组 |
| `lastUpdated` | `number` | 必填 | 资源最后更新时间，Unix 时间戳 |
| `metadata` | `object` | 可选 | 额外元数据，例如类别、提供方 |

**8.3 Bazaar 概念**

Bazaar 是供用户发现和访问 x402 资源的市场生态，主要功能包括：

- **资源发现**：按类别、服务提供方或付款要求查找 API 和服务。
- **支付透明**：使用前即可查看价格和支付方式。
- **提供方信息**：了解服务提供方及其提供的服务。
- **动态更新**：可随时添加、更新或移除资源。

**8.4 使用示例**

```bash
# 发现金融数据 API
GET /discovery/resources?type=http&limit=10

# 搜索指定的服务提供方
GET /discovery/resources?metadata[provider]=Coinbase
```

<a id="9-error-handling"></a>

**9. 错误处理**

x402 定义了一组标准错误码，Facilitator 或资源服务器可在响应中返回，帮助客户端判断支付失败的原因并采取相应措施。

- **`insufficient_funds`**: 客户端的代币余额不足，无法完成支付。
- **`invalid_exact_evm_payload_authorization_valid_after`**: 支付授权尚未生效，当前时间早于 validAfter。
- **`invalid_exact_evm_payload_authorization_valid_before`**: 支付授权已过期，当前时间晚于 validBefore。
- **`invalid_exact_evm_payload_authorization_value`**: 支付金额不足。
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

**11.1 支持的网络**

参考实现目前支持以下区块链网络：

- **`base-sepolia`**: Base Sepolia 测试网（Chain ID：84532）。
- **`base`**: Base 主网（Chain ID：8453）。
- **`avalanche-fuji`**: Avalanche Fuji 测试网（Chain ID：43113）。
- **`avalanche`**: Avalanche 主网（Chain ID：43114）。

**11.2 支持的资产**

协议目前支持以下代币类型：

- **`USDC`**: USD Coin，符合 EIP-3009 的 ERC-20 代币。
- **其他 ERC-20 代币**：实现 EIP-3009（授权转账）的代币也可以获得支持。

是否支持某种代币取决于：

- 是否满足 `exact` 方案的 EIP-3009 要求。
- Facilitator 的服务能力。
- 该代币是否已在对应网络部署。

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

### 12.3 服务端框架

x402 可以接入常见框架：

- **Express.js**：`require_payment()` 中间件。
- **FastAPI／Flask**：对应框架的中间件。
- **Hono**：支持边缘运行环境。
- **Next.js**：全栈集成。
- **ai/agents**：AI 智能体和 MCP 框架。

### 12.4 客户端库

不同传输方式的客户端都可以接入 x402 支付：

- **HTTP 客户端**：浏览器中的 axios／fetch、Python 中的 httpx／requests，以及命令行中的 curl。
- **MCP 客户端**：ai/agents MCP 客户端。
- **A2A**：x402_a2a（Python）。
- **自定义集成**：应用自行实现的支付处理。

### 12.5 进阶模式

协议可以支持更复杂的收费策略：

- 根据用户认证信息或使用情况**动态定价**。
- 通过**会话管理**按时间控制访问权限。
- 访问多个资源时进行**批量支付**。
- 基于小额支付构建**订阅模式**。

_注：预算管理、关联追踪、会话处理等具体模式的实现细节见应用说明和实现指南。特定传输方式的实现细节见相应传输规范。_

---

## 版本历史

| 版本 | 日期 | 变更 | 作者 |
| ------- | --------- | --------------------------- | ------------------------- |
| v0.2 | 2025-10-3 | 重新设计为不依赖特定传输方式的架构 | Ethan Niser |
| v0.1 | 2025-8-29 | 初始草稿 | [根据仓库内容整理] |
