---
name: cloudflare-connect-domain
description: 将用户已拥有的域名接入 Cloudflare 权威 DNS，并绑定到已经部署的 Cloudflare Worker 网站。适用于网站已能通过 workers.dev 访问、用户也已注册域名的情况；不用于购买域名或首次部署网站。
---

# 将域名接入 Cloudflare

先稳妥迁移 DNS 权威，再把确认过的主机名绑定到已有 Worker。始终区分域名注册商、DNS 托管商和网站托管平台：本流程通常只改变权威 DNS，不转移域名注册商。

## 明确范围与前置条件

1. 确认用户拥有已注册域名，并识别注册商、当前 nameservers、当前 DNS 服务商、目标 Cloudflare 账号、已部署 Worker 和可用的 `workers.dev` 地址。
2. 确定正式主域名：根域名 `example.com`、`www.example.com` 或 `app.example.com` 等子域名；同时确定另一个常见入口应如何处理。
3. 检查应用源配置中依赖网址的项目，包括公开基础网址、canonical URL、cookie domain、trusted origins、CORS、OAuth callback、webhook、sitemap 和 feed。
4. 修改 zone、nameservers、DNS 记录或 Worker routes 前，阅读 [references/domain-workflow.md](references/domain-workflow.md)。

如果 Worker 本身还不能稳定访问、用户无法控制注册商，或现有 DNS 记录清单不完整，不要执行正式切换。

## 切换前保全 DNS

- 盘点现有 A、AAAA、CNAME、MX、TXT、CAA、SRV、验证记录和邮件安全记录。Cloudflare 快速扫描只能作为起点，不能当成完整证明。
- 除非对应服务商明确要求修改，否则保持邮件和第三方验证记录不变。只有新的 Worker Custom Domain 确实替代旧网站记录时，才移除冲突的 Web 记录。
- 如果旧 DNS 或注册商启用了 DNSSEC，按 Cloudflare 当前迁移指南处理。普通 full setup 通常需要先撤销旧 DS/DNSSEC 委派，再更换 nameservers；zone 激活后再在 Cloudflare 重新启用 DNSSEC。
- 如果域名直接购买于 Cloudflare Registrar，不要虚构 nameserver 切换步骤；它已经使用 Cloudflare 权威 DNS。

## 把域名加入 Cloudflare

在正确账号中添加根域名，检查导入记录，选择用户认可的计划，并获得 Cloudflare 分配的两个 nameservers。注册商处的 nameserver 修改可能需要用户本人登录或确认。

修改后，等待 Cloudflare 显示 zone 为 Active，并独立检查权威 NS。若在 Pending 状态继续绑定会造成不清晰的半切换状态，应先等待激活。

## 绑定 Worker Custom Domain

- 再次确认目标主机名没有冲突 CNAME，且属于已激活的 Cloudflare zone。
- 对没有独立源站的 Worker 网站，优先使用 Worker Custom Domain。在 Wrangler 源配置中写入精确主机名并设置 `custom_domain: true`；Custom Domain 精确匹配主机名，不支持通配符替代。
- 更新应用的生产公开网址、基础网址、canonical URL，以及受主机名影响的 trusted origin、cookie、OAuth、webhook 或 CORS 配置。修改第三方 OAuth/webhook 控制台属于额外外部操作，需要相应授权和凭据。
- 使用项目锁定的 Wrangler 和现有部署脚本重新部署。Cloudflare 会为 Custom Domain 创建 Worker DNS 记录和证书，不要再添加冲突的手工 CNAME。
- 显式处理另一个 `www` 或根域名入口：如果两个都要直接访问应用，就声明两个精确 Custom Domains；如果只保留一个主网址，就按当前 Cloudflare 指南设置代理 DNS 记录和重定向。

除非用户明确要求关闭，并且自定义域名已经通过完整验证，否则保留 `workers.dev` 作为诊断备用入口。

## 验证与交付

验证 Cloudflare zone 状态、权威 NS、DNS 解析、HTTPS 证书、canonical 重定向、代表性页面与资源、登录和 cookie、API、feed/sitemap，以及依赖 callback 的集成。主域名和备用域名都要测试，并检查是否出现重定向循环。

最终交付 zone/账号、注册商、正式主机名、Worker、nameservers、创建的 Custom Domains 和重定向、更新过的应用设置、DNSSEC 状态、验证结果、传播等待说明和剩余人工步骤。不能因为已经提交 nameserver 表单就宣称完成。
