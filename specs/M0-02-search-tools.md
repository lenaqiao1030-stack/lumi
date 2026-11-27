# M0-02 搜索工具:web_search 与 web_fetch

- 状态:草稿(待你冻结 G3)
- 风险等级:R2(web_fetch 涉及出站网络与 SSRF 防护;合并前由你逐项审阅安全相关测试)
- 对应设计章节:docs/design/lumi.md 第十章(Tool System、防提示注入)、第九章(研究员工具集)
- 依赖的前置 spec:M0-01

## 1. 背景与目标

为领域研究员提供两个只读工具:`web_search`(找候选)与 `web_fetch`(读正文)。两者都属于 `read` 类、并发安全,返回内容一律标记为 `untrusted`。

本 spec 只定义搜索提供方的接口并用假实现测试,**不选定具体搜索服务**;真实适配器留给后续 spec(需要你先在 SOP 的 A3 中定搜索 API 选型)。

## 2. 范围

做:

- `SearchProvider` 接口与 `FakeSearchProvider`(测试用)
- `web_search` 工具
- `web_fetch` 工具:协议与端口限制、SSRF 防护、重定向校验、大小与类型限制、HTML 转文本、超时

不做:

- 真实搜索服务适配器、缓存、并行调度、命令行入口
- 浏览器渲染、登录态抓取、PDF 解析
- 引入第三方 HTML 解析库(先用简单实现,质量不足再由后续 spec 决定)

## 3. 接口与数据结构

```ts
interface SearchHit { title: string; url: string; snippet: string; publishedAt?: string; source?: string }
interface SearchProvider {
  search(query: string, opts: { maxResults: number; signal: AbortSignal }): Promise<SearchHit[]>;
}

// web_search
//   input:  { query: string (1..200), maxResults?: int (1..10, 默认 5) }
//   sideEffect: 'read',concurrencySafe: true,timeoutMs: 15000

// web_fetch
//   input:  { url: string }
//   sideEffect: 'read',concurrencySafe: true,timeoutMs: 15000

interface FetchDeps {
  fetchImpl: typeof fetch;
  resolveHost(host: string): Promise<string[]>;   // 返回解析出的 IP 列表,测试中注入
  now(): Date;
}
```

固定参数:最多 3 次重定向;响应体最多 1 MB;输出正文最多 20000 字符;允许的 content-type:`text/html`、`text/plain`、`application/xhtml+xml`、`application/json`。

已知限制(写入代码注释与回执):Node 的 fetch 无法把连接固定到已校验的 IP,存在 DNS 重绑定窗口。MVP 阶段接受该限制,生产环境依赖网络出口策略兜底。

## 4. 验收标准

web_search:

- AC-1:工具定义为 `sideEffect: 'read'`、`concurrencySafe: true`;`inputSchema` 拒绝空 query、超长 query、`maxResults` 越界
- AC-2:调用 provider,把结果格式化为带编号的文本(title、url、snippet),`trust` 为 `untrusted`
- AC-3:provider 抛错:返回 `isError: true`,错误信息不含堆栈与任何密钥
- AC-4:provider 返回 0 条:不是错误,内容明确说明「无结果」

web_fetch:

- AC-5:拒绝非 http/https 协议、带用户名密码的 URL、80 与 443 以外的端口
- AC-6:拒绝解析到私有、回环、链路本地地址的主机,至少覆盖 127.0.0.0/8、10.0.0.0/8、172.16.0.0/12、192.168.0.0/16、169.254.0.0/16(含 169.254.169.254)、::1、fc00::/7、fe80::/10、IPv4 映射的 IPv6 地址;只要解析结果中有一个落入这些范围就拒绝
- AC-7:重定向的每一跳都重新做 AC-5、AC-6 的校验;超过 3 跳报错
- AC-8:响应体超过 1 MB:中止读取并返回 `isError: true`
- AC-9:content-type 不在允许列表:拒绝并返回 `isError: true`
- AC-10:HTML 转为纯文本:去除 script、style 与标签,保留 title;超过 20000 字符截断并明确标注已截断
- AC-11:超过 `timeoutMs` 返回 `isError: true`,并通过 `AbortSignal` 真正中止请求
- AC-12:输出包含最终 URL、抓取时间(取自注入的 `now`)和固定的不可信内容标注头;`trust` 为 `untrusted`

## 5. 测试要求

- 全部使用注入的假 `fetchImpl`、`resolveHost`、`now` 与 `FakeSearchProvider`,禁止真实网络
- 每条 AC 至少一个测试,测试名以 `AC-n:` 开头
- AC-6 使用表驱动测试覆盖所列全部地址段与 IPv4 映射写法
- AC-7 用「第一跳合法、第二跳指向内网」的用例证明每跳都重新校验

## 6. 涉及文件与依赖

允许新增或修改:

- `src/tools/web-search.ts`、`src/tools/web-fetch.ts`、`src/tools/net-guard.ts`、`src/tools/html-to-text.ts`、`src/tools/search-provider.ts`、`src/tools/index.ts`
- `tests/tools/*.test.ts`、`tests/helpers/fake-search-provider.ts`

允许新增的依赖:无。

明确禁止修改:`.github/`、`.cursor/`、`AGENTS.md`、`src/governance/`、`src/agent/`(如需改动 M0-01 的类型,在回执「偏离」中提出,不要自行修改)。

## 7. 非功能要求

- 错误信息对模型友好但不泄露内部细节
- `net-guard` 为纯函数模块,便于单独测试,不依赖网络

## 8. 发布、回滚与线上核实判据

不适用(无线上环境)。

## 9. 完成定义

- 所有 AC 有对应测试且通过;`pnpm lint`、`pnpm typecheck`、`pnpm test` 通过
- 变更约 400 行以内;若超出,拆为 M0-02a(web_search:AC-1 到 AC-4)与 M0-02b(web_fetch:AC-5 到 AC-12)两个 PR
- 回执完整,并明确写出 SSRF 防护的已知限制

## 10. 开放问题

- 真实搜索服务选型(SOP A3):在 M0-05 之前由你决定
