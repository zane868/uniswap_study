# x402 V2 使用示例

x402 将支付接入 HTTP 请求流程。客户端访问收费资源时，服务端返回 `402 Payment Required` 和付款要求；客户端签署支付授权，携带支付信息重新请求；服务端验证、结算成功后返回资源。

下面以一个收费天气接口为例，每次调用收取 **0.01 USDC**，支付网络为 **Base Sepolia 测试网**。

## 1. 请求天气接口

```http
GET /weather HTTP/1.1
Host: api.example.com
Accept: application/json
```

未携带有效支付信息时，服务端返回：

```http
HTTP/1.1 402 Payment Required
Content-Type: application/json
Content-Length: 2
PAYMENT-REQUIRED: eyJ4NDAyVmVyc2lvbiI6MiwicmVzb3VyY2UiOnsidXJsIjoiaHR0cHM6Ly9hcGkuZXhhbXBsZS5jb20vd2VhdGhlciJ9LCJhY2NlcHRzIjpbeyJzY2hlbWUiOiJleGFjdCIsIm5ldHdvcmsiOiJlaXAxNTU6ODQ1MzIiLCJhbW91bnQiOiIxMDAwMCIsImFzc2V0IjoiMHgwMzZDYkQ1Mzg0MmM1NDI2NjM0ZTc5Mjk1NDFlQzIzMThmM2RDRjdlIiwicGF5VG8iOiIweDExMTExMTExMTExMTExMTExMTExMTExMTExMTExMTExMTExMTExMTEiLCJtYXhUaW1lb3V0U2Vjb25kcyI6NjAsImV4dHJhIjp7Im5hbWUiOiJVU0RDIiwidmVyc2lvbiI6IjIifX1dfQ==

{}
```

`PAYMENT-REQUIRED` 是响应头，值为经过 Base64 编码的 JSON。Base64 是编码，不是加密。响应头和响应体之间用空行分隔，这里的响应体是 `{}`。

解码后得到：

```json
{
  "x402Version": 2,
  "resource": {
    "url": "https://api.example.com/weather"
  },
  "accepts": [
    {
      "scheme": "exact",
      "network": "eip155:84532",
      "amount": "10000",
      "asset": "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      "payTo": "0x1111111111111111111111111111111111111111",
      "maxTimeoutSeconds": 60,
      "extra": {
        "name": "USDC",
        "version": "2"
      }
    }
  ]
}
```

收款地址 `0x1111...1111` 是演示占位地址，不用于实际付款。

## 2. 字段说明

| 字段 | 含义 |
| --- | --- |
| `x402Version` | x402 协议版本，这里为 V2。 |
| `resource.url` | 本次请求的收费资源地址。 |
| `accepts` | 可接受的支付方案列表，客户端从中选择一种。 |
| `scheme` | 支付方案。`exact` 表示按指定金额支付。 |
| `network` | CAIP-2 格式的网络标识。`eip155:84532` 表示 Base Sepolia。 |
| `amount` | 支付金额，以代币最小单位表示，使用字符串避免整数精度问题。USDC 有 6 位小数，`10000` 即 `0.01 USDC`。 |
| `asset` | 支付代币的合约地址。这里使用 Base Sepolia 上的 USDC 地址。 |
| `payTo` | 收款地址。 |
| `maxTimeoutSeconds` | 支付允许的最长时间，单位为秒。 |
| `extra` | 支付方案所需的额外信息。这里的 `name` 和 `version` 用于代币的 EIP-712 签名域；`version: "2"` 与 `x402Version: 2` 含义不同。 |

## 3. 可运行的 Node.js 服务端

将以下代码保存为 `server.cjs`，使用 Node.js 运行，无需安装依赖。

该示例演示如何生成并返回付款要求，不执行真实扣款，也不验证支付签名。即使客户端携带 `PAYMENT-SIGNATURE`，此示例仍返回 402。接入完整支付流程时，应使用 x402 SDK 完成验证和结算。

```javascript
const http = require("node:http");

const paymentRequired = {
  x402Version: 2,
  resource: {
    url: "http://localhost:3000/weather",
  },
  accepts: [
    {
      scheme: "exact",
      network: "eip155:84532",
      amount: "10000",
      asset: "0x036CbD53842c5426634e7929541eC2318f3dCF7e",
      payTo: "0x1111111111111111111111111111111111111111",
      maxTimeoutSeconds: 60,
      extra: {
        name: "USDC",
        version: "2",
      },
    },
  ],
};

const server = http.createServer((req, res) => {
  if (req.method !== "GET" || req.url !== "/weather") {
    res.writeHead(404, { "Content-Type": "application/json" });
    res.end(JSON.stringify({ error: "Not found" }));
    return;
  }

  const encoded = Buffer.from(
    JSON.stringify(paymentRequired),
    "utf8",
  ).toString("base64");

  res.writeHead(402, {
    "Content-Type": "application/json",
    "Content-Length": "2",
    "PAYMENT-REQUIRED": encoded,
  });
  res.end("{}");
});

server.listen(3000, "127.0.0.1", () => {
  console.log("服务已启动：http://localhost:3000/weather");
});
```

启动服务：

```bash
node server.cjs
```

在另一个终端请求：

```bash
curl -i http://localhost:3000/weather
```

可以看到 `402 Payment Required`、`PAYMENT-REQUIRED` 响应头和 `{}` 响应体。运行示例使用本地 URL，因此生成的 Base64 内容与前面的线上域名示例不同。

## 4. 客户端读取付款要求

以下代码可在支持内置 `fetch` 的 Node.js 环境中运行。它只读取并展示报价，不签名，也不付款。

```javascript
async function main() {
  const response = await fetch("http://localhost:3000/weather");

  if (response.status !== 402) {
    throw new Error(`预期返回 402，实际为 ${response.status}`);
  }

  const encoded = response.headers.get("payment-required");
  if (!encoded) {
    throw new Error("缺少 PAYMENT-REQUIRED 响应头");
  }

  const requirements = JSON.parse(
    Buffer.from(encoded, "base64").toString("utf8"),
  );

  console.log(JSON.stringify(requirements, null, 2));
}

main().catch((error) => {
  console.error(error);
  process.exitCode = 1;
});
```

如果在浏览器中跨域读取这些响应头，服务端还需要配置相应的 CORS 规则，并通过 `Access-Control-Expose-Headers` 暴露 `PAYMENT-REQUIRED` 和 `PAYMENT-RESPONSE`。

## 5. 完整支付流程中的后续报文

客户端选择付款方案，检查金额、网络和收款地址，再按对应方案签署支付授权。支付数据经过 Base64 编码后，放入 `PAYMENT-SIGNATURE` 请求头，重新请求原资源。

以下 `<...>` 为占位说明，不是可直接提交的真实签名：

```http
GET /weather HTTP/1.1
Host: api.example.com
PAYMENT-SIGNATURE: <支付数据 JSON 的 Base64 编码>
```

支付数据通常包含：

- `x402Version`：协议版本。
- `accepted`：从 `accepts` 中选定的支付要求。
- `payload`：该支付方案要求的数据，例如签名及授权参数。
- `resource`：可选的资源信息。

例如，采用 EVM `exact` 方案的 EIP-3009 授权方式时，`payload` 包含签名和 `from`、`to`、`value`、`validAfter`、`validBefore`、`nonce` 等授权参数。此处的 `nonce` 是用于防重放的 32 字节随机值，不是 EOA 交易的递增 nonce。

服务端可以自行处理支付，也可以使用 Facilitator 服务验证授权并提交结算。成功后，响应示意如下：

```http
HTTP/1.1 200 OK
Content-Type: application/json
PAYMENT-RESPONSE: <结算结果 JSON 的 Base64 编码>

{
  "city": "Shanghai",
  "temperature": 23
}
```

结算结果通常包含 `success`、`transaction`、`network` 和可选的 `payer`。客户端只提供签名不代表支付已经完成，服务端需要按所选方案完成验证和结算。

| 响应头／请求头 | 方向 | 内容 |
| --- | --- | --- |
| `PAYMENT-REQUIRED` | 服务端 → 客户端 | 付款要求。 |
| `PAYMENT-SIGNATURE` | 客户端 → 服务端 | 所选方案及支付授权数据。 |
| `PAYMENT-RESPONSE` | 服务端 → 客户端 | 结算结果。 |

以上三个头的值都使用 Base64 编码的 JSON。HTTP 响应体由业务服务自行定义。

## 参考文档

- [x402 V2 核心规范](https://github.com/coinbase/x402/blob/main/specs/x402-specification-v2.md)
- [x402 V2 HTTP 传输规范](https://github.com/coinbase/x402/blob/main/specs/transports-v2/http.md)
- [Coinbase x402 文档](https://docs.cdp.coinbase.com/x402/welcome)
