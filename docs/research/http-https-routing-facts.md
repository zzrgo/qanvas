# 固定入口下 HTTP/HTTPS 与集成机构标识的协议边界

核对日期：2026-10-26  
范围：一级规范与 Node.js 24 系列官方 API 文档的桌面研究。未安装依赖、运行监听实验、调用真实机构、修改代理/DNS/hosts/证书或验证 Electron 打包。仓库技术栈中的版本是选型基线，本研究不证明整套版本已发布、可安装或兼容。文档的 `latest-v24.x` 路径会更新；下列 API 的具体可用性最终应按 Electron 内置 Node 版本复核。

## 请求中有哪些机构信号

HTTP/1.1 的常见直连请求行只有路径与 query，域名/端口由 Host 承载；将所有机构的 URL 直接改成同一个本机 URL 后，默认 Host 也会变为该本机地址。服务端不会自动获得调用方原来的机构 URL。HTTP 正向代理的 absolute-form 请求包含完整目标 URI，是另一种接入方式。[RFC 9112 §3.2](https://www.rfc-editor.org/rfc/rfc9112.html#section-3.2)

Host 的用途之一就是同一服务端按不同目标主机区分资源。HTTP/2 在有 `:authority` 时以它确定目标，不能忽略它只看 Host；二者不应互相冲突。这些都是请求级信息，不是 TLS 证书选择机制。[RFC 9110 §7.2](https://www.rfc-editor.org/rfc/rfc9110.html#section-7.2)、[RFC 9113 §8.3.1](https://www.rfc-editor.org/rfc/rfc9113.html#section-8.3.1)

自定义机构 header 可以由可配置的调用方、其网关/代理或 qanvas 的前置接入规则添加。Node HTTP API 可读取收到的 headers，也提供出站请求 header 设置接口；协议没有“集成机构名”的内建字段，名称和取值是应用契约。[Node HTTP：IncomingMessage.headers](https://nodejs.org/docs/latest-v24.x/api/http.html#messageheaders)、[ClientRequest.setHeader](https://nodejs.org/docs/latest-v24.x/api/http.html#requestsetheadername-value)

**推论**：如果 qanvas 要自己写入机构 header，必须先从 Host、已有字段、路径前缀、明确的接入规则或其他有效信号确定机构。若两个请求所有可观察信息与接入上下文都相同，无法仅凭用户希望它们属于不同机构来消歧。内部补写 header 只能保存已经得到的判断，不能创造缺失的信息。此为应用设计推论，不是新增协议规定。

## HTTPS：证书选择与业务路由分属不同阶段

常规 HTTPS 连接先进行 TLS 握手，再发送 HTTP 请求；HTTP header 在 TLS 内传输。SNI 的 server_name 来自 ClientHello，可用于握手时选择证书；它传递 DNS 名称，不允许 IP 字面量。[RFC 9112 §9.7](https://www.rfc-editor.org/rfc/rfc9112.html#section-9.7)、[RFC 6066 §3](https://www.rfc-editor.org/rfc/rfc6066.html#section-3)

**推论**：不能等读到自定义机构 header 或 Host 后，再用它挽救同一连接上已经发生的初次证书选择。先以统一入口身份建立 TLS、再按 HTTP header 分机构，与按机构域名/SNI 选择证书，是两种可行但前提不同的设计。TLS 1.3 存在恢复连接的 0-RTT 优化，本结论并不要求所有 TLS 会话消息都严格等到完整握手结束；应用 header 仍不是初次握手的 SNI 输入。[RFC 8446 §2、§2.3](https://www.rfc-editor.org/rfc/rfc8446.html#section-2)

| 条件方案 | 客户端与服务端的必要前提 |
| --- | --- |
| 统一入口域名，机构 header 分流 | 客户端能添加机构 header，并信任且验证入口域名的证书；机构不必各自拥有入站证书。 |
| 多机构域名保留，SNI/Host 分流 | 域名的连接目标能导向 qanvas；客户端发送合适 SNI；qanvas 持有可覆盖这些名称的证书/私钥，客户端认可证书链与名称。 |
| 多域名共用一张证书 | 证书需覆盖所有被验证的名称；不能仅因它是“同一应用”就认定任意机构域名都受覆盖。 |
| URL 直接使用本机/LAN IP | 不能假设客户端发送机构 SNI；按 IP 验证时需要匹配的 iPAddress SAN。机构需另有请求级信号。 |

表格中的证书名称/IP 匹配条件来自 [RFC 9525 §4、§6、§7.4](https://www.rfc-editor.org/rfc/rfc9525.html#section-6)；Node HTTPS 在目标为 IP 时默认不发送 servername，TLS 服务器提供 SNICallback / 默认 secure context。[Node HTTPS Agent](https://nodejs.org/docs/latest-v24.x/api/https.html#new-agentoptions)、[Node TLS createServer](https://nodejs.org/docs/latest-v24.x/api/tls.html#tlscreateserveroptions-secureconnectionlistener)

## “固定端口”有几种不同含义

Node 常规入口是 `http.createServer` 和 `https.createServer`，分别处理明文 HTTP 与 TLS 上的 HTTP。**从这些 API 的支持面推论**：不能把“创建两个 server 并监听同一地址端口”当作协议自动分流；常见重复绑定会得到 EADDRINUSE。若同一 TCP 入口必须兼容明文与 TLS，需要额外统一接入/协议识别与分派机制，并另行验证实现。这里没有断言同端口在任何实现中都不可能。[Node HTTP createServer](https://nodejs.org/docs/latest-v24.x/api/http.html#httpcreateserveroptions-requestlistener)、[HTTPS createServer](https://nodejs.org/docs/latest-v24.x/api/https.html#httpscreateserveroptions-requestlistener)、[Net listen](https://nodejs.org/docs/latest-v24.x/api/net.html#serverlisten)

分别设置固定 HTTP 与 HTTPS 端口，可按上述常规服务器分别监听；这与“所有机构共用同一套入口”不冲突。HTTP/2 secure server 的 `allowHTTP1` 是 TLS 连接上的 HTTP/1.x / HTTP/2 兼容，不是额外允许明文 HTTP 的开关。[Node HTTP/2 Compatibility API](https://nodejs.org/docs/latest-v24.x/api/http2.html#compatibility-api)

本机/LAN 可达性取决于绑定地址和网络环境；Node 在未指定 host 时可能监听 `::` 或 `0.0.0.0`，不能把省略 host 等同于仅本机。固定端口与随机分配端口也不同，`port: 0` 会由系统分配。[Node Net listen](https://nodejs.org/docs/latest-v24.x/api/net.html#serverlistenport-host-backlog-callback)

## 不必默认修改系统代理或安装根证书

只改 URL、添加 header、保留 Host、保留 SNI、改变连接目标，是不同能力，需逐个核对真实调用方是否可配置。例如 curl 的 `--connect-to` 只改变连接目标，不改变用于 TLS/SNI 和应用协议的原主机信息；这种客户端能力可以避免依赖全局 hosts 改动，但不能假定每个 SDK 都支持。[curl --connect-to](https://curl.se/docs/manpage.html#--connect-to)

改变 DNS/hosts 指向只会提供连接目标映射这一条件，不能自动提供匹配的 TLS 身份；若 URL 端口也不同，还需要另行配置。客户端必须认可所用证书的信任链和名称，信任可以由客户端单独配置；“安装系统根证书”“关闭校验”都不是协议强制的唯一方案。[RFC 9525 §6](https://www.rfc-editor.org/rfc/rfc9525.html#section-6)、[Node TLS connect / ca / checkServerIdentity](https://nodejs.org/docs/latest-v24.x/api/tls.html#tlsconnectoptions-callback)

## 真实转发是第二段连接

qanvas 终止入站 HTTPS 并读取/转换业务请求后，再向真实机构发起 HTTPS 请求，是两段连接。qanvas 在前一段是 TLS 服务端，在后一段是 TLS 客户端；两段证书、信任、SNI 和客户端身份配置不能混为一套。该分段是对已确认“读取业务字段后转发”的架构推论，区别于不解密内容的纯 CONNECT 隧道。[RFC 9110 §3.7、§9.3.6](https://www.rfc-editor.org/rfc/rfc9110.html#section-9.3.6)、[Node HTTPS request](https://nodejs.org/docs/latest-v24.x/api/https.html#httpsrequestoptions-callback)

入站如需客户端证书认证，可用 requestCert 与相应 CA/验证策略；出站如真实机构要求 mTLS，需要为 qanvas 的客户端连接配置 cert/key/pfx，并验证真实机构证书。Node 的默认验证开关和名称校验有明确 API；研究不据此决定首版必须提供 mTLS，也不把关闭验证作为默认。[Node TLS createServer](https://nodejs.org/docs/latest-v24.x/api/tls.html#tlscreateserveroptions-secureconnectionlistener)、[TLS connect](https://nodejs.org/docs/latest-v24.x/api/tls.html#tlsconnectoptions-callback)、[HTTPS request](https://nodejs.org/docs/latest-v24.x/api/https.html#httpsrequestoptions-callback)

## 留给决策票的产品选择

本研究解除了协议事实的不确定性，以下仍需用户决定：

- 调用方可以修改哪些配置；机构选择优先采用哪一种信号，由谁写机构 header，冲突和缺失时如何反馈。
- “固定端口”是否意味着一对固定端口，还是必须混合使用单端口。
- 入站使用统一入口身份还是机构域名身份；谁提供证书、私钥与调用方信任配置。
- 哪些机构确实需要 mTLS、特殊 Host/SNI 或证书固定；这些都需要具体接入样例。
- 内部机构 header 在真实转发时是否去除；出站目标、Host 与 SNI 如何与机构定义保持一致。

未选择库、端口号、header 名称、监听地址、证书签发方式或产品默认失败策略。
